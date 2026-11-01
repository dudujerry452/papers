# mupsi2 — 基础设施设计

> 目标：按用户给定的 8 个阶段顺序，重建原 `mupsi` 的 GPIS 渲染基础设施，最终得到与 `mupsi` 相近的效果。
> 本文档是**已定设计**；实现进度见 `docs/STATE.md`。

## 0. 目标与范围

- 一个可更换 CPU/CUDA 后端的 GPIS 软渲染器（继承 `docs/plan.md`、`docs/prompt.md`、`docs/framwork.md` 的既定决策）。
- 数值内核用 `MPHD`（`__host__ __device__`）标记，g++ 与 nvcc 各编译一次；后端壳（omp vs `<<<>>>`）分开编译。
- 本阶段交付 8 项基础设施（见 §5），最终闭环是：**发射光线 → 求交硬表面 → 进入包裹介质 → ray marching 找到 GP 隐式曲面 → 纯色 Lambertian 着色 → 输出 PPM**。

## 1. 关键决策（回应任务里的 7 点）

1. **BVH 自研，不用 Embree/OptiX。**
   - 理由：`plan.md`/`prompt.md` 已定「基础设施自己手写」；BVH 遍历用显式栈（device 友好）；自研 BVH 在 CPU/CUDA 上**位一致**，可逐像素回归对拍。Embree（CPU）/OptiX（GPU）作为未来的可选加速后端，接口隔离在 `core/bvh` 之后，可无缝替换。
2. **几何体只实现 `primitive`**：`Sphere`（解析求交）与 `Mesh`（三角形 + 每 mesh 一棵 BVH）。Skydrome 留到光照阶段。
3. **图片工具已实现**（`host/ppm_util.*`），本阶段只修复其头文件签名错误（声明与定义不一致）。
4. **光线发射 = `Camera::generateRay`**，支持 ray tracing（主光线 + 反弹光线均出自同一个 trace 循环）。
5. **求交携带表面 `uv`，但不分配材质**：`HitRecord` 含 `uv/texId/bsdfId/lightId/mediumId`；BSDF 只写「纯色 Lambertian + 透明 Null」，材质/纹理后续再挂。
6. **介质是「包裹在 primitive 里」的东西**：`PrimitiveInfo::mediumId` 标记该 primitive 内部裹有介质；trace 循环里介质**单独走 ray marching 分支**（`sampleDistance`/`transmittance`），硬表面走 ray tracing 分支。
7. **效果对齐**：噪声/核函数/条件化不要按原 `mupsi` 逐字移植, 原来的写法太繁琐不直观. 而是应该阅读 @~/dudublog/docs/source/_posts/gpis*.md , 利用写好的math工具(已经写好了)写一版更可读更对应数学概念的

## 2. 命名空间与依赖

沿用现有代码：基础类型/数学在 `mps`，算法在 `mps::core`（嵌套命名空间，可无前缀访问 `mps` 符号）。

```
mps         :: Vec2f/Vec3f/Vec4f/RGB/FrameBuffer、math_util、sample(切空间/采样)
mps::core   :: Ray/AABB/Primitive/BVH/Intersect/Scene/Camera/Texture/Bsdf/Medium/GP/Trace
mps::host   :: SceneBuilder、DataManager、ppm_util
mps::kernels:: 后端壳(Entrance<CPU>/<CUDA>)
```

依赖单向：`core ← kernels ← host`。

## 3. 数据模型

### 3.1 图元（SoA，device 可拷贝）

```cpp
enum class PrimitiveType : u16 { Sphere, Mesh };

struct PrimitiveInfo {           // 公共头，alignas(16)
  PrimitiveType type;
  u16 bsdfId;    // BsdfType
  u16 texId;     // 纹理 id（本阶段只有 ConstantTexture）
  u16 mediumId;  // 0xFFFF = 无；否则为包裹在内部的介质
  u16 lightId;   // 0xFFFF = 非光源
  vec3f wpos;    // 平移/球心
  vec4f quater;  // 旋转（四元数）
  f32  scale;    // 均匀缩放（球 = 半径）
  AABB bounds;   // 世界空间包围盒（host 预计算）
};

struct Sphere { PrimitiveInfo info; };   // 半径 = info.scale
struct Mesh   { PrimitiveInfo info;
                u32 vertexOffset, vertexCount;   // 指向 SceneDevice::vertices
                u32 faceOffset, faceCount;       // 指向 SceneDevice::faces (3×u32/三角形)
                u32 bvhRoot, bvhNodeOffset, bvhNodeCount, bvhTriOffset, bvhTriCount; };
```

- Mesh 顶点/面在 host 上**预先变换到世界空间**，device 求交无需再乘变换矩阵。
- 每个 mesh 携带**自己的 BVH**（节点 + 重排三角形索引），与原 mupsi 一致。

### 3.2 场景（SoA）

```cpp
struct SceneDevice {
  u32 numSpheres;  Sphere* spheres;
  u32 numMeshes;   Mesh*   meshes;
  vec3f*  vertices;        // 所有 mesh 顶点（世界空间）
  u32*    faces;           // 所有 mesh 三角形
  BVHNode* bvhNodes;       // 所有 mesh BVH 节点
  u32*    bvhTriIndices;   // 所有 mesh BVH 重排三角形索引
  vec3f*  vertexNormals;   // 角度加权伪法向（平行于 vertices）
  // 体素均值（MeanType::Voxel）：密集符号网格 + 排序窄带数组
  u8*  voxelSignGrid; u32* voxelBandIndex; f32* voxelBandValue;
  u32  voxelRes, voxelBandCount;
};
```

### 3.3 介质（POD，device 可拷贝）

```cpp
enum class MediumType : u16 { SDF, GP };
enum class MeanType   : u16 { Sphere, Mesh, Voxel };  // Voxel = 稀疏窄带 SDF 网格
typedef enum class GPMode     : u16 { SingleRealization, RenewalPlus };

struct Medium {
  MediumType type;  MeanType meanType;
  vec3f meanCenter; f32 meanRadius;       // 球均值
  MeshMean meshMean;                      // 网格均值(MeanType::Mesh)
  SparseSDFGrid voxelMean;                // 体素均值(MeanType::Voxel)
  f32 sigma; vec3f lengthScale; f32 kernelRadius;  // SE 核（lengthScale 逐轴 → 各向异性）
  u32 impulseDensity;                      // points_per_cell
  GPMode gpMode;
  KernelField kernelField;   // 非平稳 {σ,L} 体素场（σ 场 / L 场）
  u16 colorMethod; ColorGrid colorGrid;   // GPIS 纹理（voxel/surface/kernel）
  HeatKernel heatKernel;                  // 热核纹理 HKTEx（heatkernel）
  u16 surfaceBsdf;                        // GP 曲面 BSDF（Lambertian / Specular）
  vec3f boundMin, boundMax; f32 sceneScale;
};
```

介质接口是三个纯函数(无虚函数,device 友好):

```cpp
bool    sampleDistance(const Ray&, const Medium&, const GPConditioningState&, u32 seed, MediumSample&);
RGB_F32 transmittance (const Ray&, const Medium&, const GPConditioningState&, u32 seed);
GPConditioningState sampleCondition(const MediumSample&, const Medium&, u32 seed, const vec3f& rayDir);
```

## 4. 求交 / trace 流程

```
trace(ray, ctx, seed):
  throughput=1, emission=0, conditioning={}, inMedium=false
  loop bounce:
    if inMedium:                          # —— ray marching 分支（介质）
      if medium.sampleDistance(ray,...):  # 命中 GP 隐式曲面
        纯色 Lambertian 着色(法线=∇f 归一化)
        散射出新方向 wi；conditioning = sampleCondition(...)
        ray = 新光线；continue
      else: inMedium=false                # 走出介质，回落到硬表面
    hr = scene.intersect(ray)             # —— ray tracing 分支（硬表面）
    if !hr.hit: emission += throughput·背景(ray.dir)；break
    wi = bsdfSample(hr.bsdfId, -ray.dir, hr.ng, ...)
    if wi·hr.ng < 0 && hr.mediumId != NONE:   # 穿透边界进入包裹介质
      inMedium=true; ray 向内偏移；continue
    if bsdfId == Lambertian: 着色 + throughput *= weight
    ray 向外偏移沿 wi 前进
```

- 包裹球 `bsdf=Null`（透明），`mediumId=介质`；Null BSDF 的 `wi = -wo`，据此用 `wi·ng` 的符号判断「进入/离开」介质，与原 mupsi 的 backside 逻辑等价。
- 本阶段光照为「固定平行光 + 环境光 + 渐变天空背景」（无 NEE/阴影射线），足以显示 GP 曲面；NEE/发光图元/材质留待后续阶段（与用户「先不写材质」一致）。

## 5. 实现顺序（8 阶段）

| # | 阶段 | 落点 | 状态 |
|---|---|---|---|
| 1 | 数学工具 | `core/math/*`（vec 运算、PCG32/xxhash、切空间/半球采样） | 见 STATE.md |
| 2 | 几何体定义 | `core/geometry/*`、`core/primitive/*` | |
| 3 | 光线求交 | `core/bvh/*`、`core/tracer/intersect.*`、`core/scene/*` | |
| 4 | 图片工具 | `host/ppm_util.*`（修复签名） | |
| 5 | 光线发射 | `core/tracer/camera.h` | |
| 6 | 求交更多信息(uv/texture) | `HitRecord.uv`、`core/tracer/texture.h` | |
| 7 | 介质定义 | `core/medium/*`（SDF + GP 的 POD 抽象） | |
| 8 | GP 介质 | `core/gp/*`、`core/medium/gpmedium.*` | |

## 6. 数值要点（按 gpis-1/2.md 的数学概念，干净实现）

- `make_seed` 使用 `0x9e3779b9`（修正原 mupsi2 误写的 `0x9e3779b0`）；PCG32 与 `normalizedUint` 只求 CPU/GPU 位一致，不再逐字对齐旧 mupsi。
- **SE 核**（各向异性，Eq.25 对角形式）：`lengthScale = (Lx,Ly,Lz)`，
  `h(s,p)=exp(-Σ(rᵢ/Lᵢ)²)`（`r=p-s`）。
  导数：`∇_p h = h·(-2)(r_x/Lx², r_y/Ly², r_z/Lz²)`；
  Hessian-向量积 `H_h·v = h·[-2(v_x/Lx²,...) + 4g(g·v)]`，`g=(r_x/Lx²,...)`。
  条件化 β 由标量 `1/h''(0)=-L²/2` 换成**矩阵解** `solveBeta(t)=-(1/2)diag(Lx²,Ly²,Lz²)·t`（因 `H_h(0)=-2·diag(1/L²)`）。
  `isIsotropic() = (Lx==Ly==Lz)`；各向同性时退化为标量 L，上述公式全部退化为旧形式。
- **稀疏卷积噪声**（gpis-1.md「离散卷积」）：`ψ(p) = Σᵢ wᵢ h(sᵢ,p)`。
  - 取整用 `floor(p/cellSize)`（数学正确，不用旧实现的截断）。
  - 每格固定 `pointsPerCell` 个脉冲，`sᵢ` 在格内伪随机，`wᵢ = ±1`（Bernoulli，单位方差；连续推导中为 `N(0, σ²/λ)`）。
  - 归一化：`Var = λ·(π/2)^{3/2}·Lx·Ly·Lz`（各向同性退化为 `λ(πL²/2)^{3/2}`），其中 `λ = pointsPerCell / cellSize³`；`ψ ← σ·ψ_raw/√Var`，使 `Cov(ψ(p),ψ(q)) = σ² exp(-‖p-q‖²/(2L²))`（各向同性时），与 `L`、`λ` 无关（gpis-1.md「处理成更核心的形式」）。
- **条件化**（gpis-1.md「条件化」+ gpis-2.md 梯度）：命中点 `C`（`f(C)=0`）处加修正 `ψ_cond(x)=ψ(x)+α·h(C,x)+β·∇h(C,x)`。
  - `α = -μ(C) - ψ(C)`（钉住 `f(C)=0`）。
  - `β = solveBeta(n - ∇μ(C) - ∇ψ(C))`（匹配法向；各向同性时 = `(-L²/2)·(...)`）。
  - 梯度修正：`∇ψ_cond = ∇ψ + α·∇h(C,·) + H_h(C,·)·β`。
- **紧凑支撑剔除 + 每 cell RNG**(论文 Algorithm 1 / 官方 cell3D):每 cell 一个 `Random(make_seed(ijk, seed))` 顺序抽脉冲(无 warmup);按**各向异性距离** `q=Σ(rᵢ/Lᵢ)² ≥ cellSize²` 时 `continue`,不碰 exp;邻域 cell 数按 `ceil(Lᵢ)` **自适应扩展**(各向同性退化为 3×3×3)。
- **1D 噪声**(论文 477/520 行;gpis-1.md「子域不变性」):**平稳各向同性** GP 限制到一条直线上仍是同协方差 `exp(-r²/(2L²))` 的 1D GP。成立条件(均为**运行时**判据):`isIsotropic()`(Lx==Ly==Lz)**且** `kernelField.empty()`(无 σ/L 场=平稳)**且** `gpMode != SingleRealization`。满足时 marching 沿光线只遍历 3 个 cell:`eval1D(t) → {ψ, ∂ψ/∂t}`,归一化 `Var₁ = λ₁(πL²/2)^{1/2}`;命中点全梯度 = `g_z·d + g_xy`,`g_xy ~ N(0, σ²/L²)`(论文 §6.2.1 分解)。条件化修正始终走全 3D 单 splat(廉价,与噪声路径无关)。
- **体素网格均值**(论文 §7,OpenVDB 风格):`SparseSDFGrid` 预计算 mesh SDF——密集 1 字节符号网格 + 排序窄带数组(|sdf|`<band` 存全精度); 查 = 二分搜 + 三线性插值(值/梯度),带外回退 ±band。替换每步 closestPointBVH,O(log N)→O(1)。**窄带宽度按峰值 σ 缩放** `band = 4σ_peak/cellSize + 2`(否则大 σ 下 ±band 回退被噪声淹没、在包围盒内产生伪曲面)。局限:薄于 1 cell 的特征(如 bunny 耳尖)零等值面在体素上会被水平截断;`--meshmean` 退回精确 mesh SDF 可完整还原薄尖(代价 ~20× 慢)。
- **GP ray marching**：固定步长 `dt = min(sceneScale/300, (ft-nt)/100)` + 符号翻转检测 + 交点回退。
- **AABB 介质边界**：直接 `aabbIntersect` 进入测试(无包裹图元);margin 逐轴可配(`--margin mx,my,mz`,σ 倍数,默认 3,5,3——顶部 5σ 给耳朵余量)。
- **空间 σ 场**(Eq.25):`--sigmafield x|y|z` 沿该轴在**网格 bbox** 上把 σ 从 `--sigmarange min,max` 线性插值(或 `--sigmaclamp` 在模型中点处硬阶跃);体素窄带与 AABB margin 按 σ 峰值计算。

## 7. 解析梯度分布 NEE（论文 §6,已实现）

**目标**：对 delta BSDF(镜面微表面)GPIS 也能做 NEE/MIS,消除「原始梯度法线 + 镜像递归」的粗糙翻转伪影。

- **数学**(论文 396–400 行):命中点 `p_s` 处法线不是确定的,而是**条件分布** `p(n|ζ)`。对平稳各向同性核,`g_xy := (g_x,g_y) ⊥ g_z`,`g_z` 由沿线观测确定(1D 噪声导数),`g_xy ~ N(μ_xy, Σ_xy)` 是 2D 各向同性高斯(无条件时零均值、方差 1/(2L²),Renewal+ 下由命中点梯度条件化修正)。
- **NEE**:采样 `g_xy ~ p(g_xy|g_z,ζ)` 使反射方向 `ω_s` 指向光源(图 11:由 `ω_s` 和 `g_z` 反解 `g_xy`);光采样 PDF 从立体角测度换到梯度测度。MIS 把光采样 / 梯度采样两策略用 power heuristic 合并(Algorithm 4)。
- **门控**(已改为运行时,非编译期):此特性要求
  - `mediumIsIsotropic(m)`(各向同性,否则 g_xy⊥g_z 分解不成立);
  - `kernelField.empty()`(平稳,否则 σ/L 随空间变,分解同样不成立);
  - `GPMode != SingleRealization`(SingleRealization 法线确定,无分布可采)。
  不满足时回退到镜像递归 UNI(现有路径)。

## 8. 非平稳协方差（论文 Eq. 25,已实现对角形式）

- **Eq. 25 对角形式**:`ψ_ani(p) = ψ_iso(Mp)`,`M = diag(1/Lx,1/Ly,1/Lz)`——即逐轴 `lengthScale`(§6 已实现)。
- **空间变化的 σ**:`KernelField`(res³ 体素场,存每点 σ 与 L,三线性插值)。`SparseConvNoise::sigmaAt/lengthScale2At` 查询局部核参数;`--sigmafield x|y|z --sigmarange min,max [--sigmaclamp]` 在网格 bbox 上构建 σ 斜坡/阶跃场。
- **非平稳 → 禁用 1D 噪声**:`use1DNoise` 要求 `kernelField.empty()`;任何 σ/L 场一律退回 3D 噪声(27 cell + 自适应邻域)。
- 各向异性 M 与空间 σ/Σ 的完整组合(论文 Eq.25 的一般形式,含旋转/剪切)仍以单分辨率起步、未做多分辨率稀疏卷积。
