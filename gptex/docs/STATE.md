# mupsi2 — 实现进度

> 设计见 `docs/DESIGN.md`,测试方法论/避坑见 `docs/TEST.md`。本文件记录 8 阶段落地状态、验证结果、复刻论文场景的路线图与下一步。

## 当前状态概览

**8 阶段全部落地;CPU/CUDA 后端均编译通过并渲染出 GPIS 曲面。**
已支持:SphereMean + MeshMean(spot/bunny/dragon)GP 场景、CUDA 后端、eps 统一管理、场景 scale 感知 marching、开网格/翻面稳健的 SDF(角度加权伪法向)、framebuffer 左下角原点的正确输出。

**稀疏卷积性能已对齐 Xu 2025 论文原实现**(见下节):紧凑支撑剔除 + 每 cell RNG + **1D 噪声**(Renewal/Renewal+)+ **稀疏窄带体素网格 μ**(OpenVDB 风格,O(1) 查)。dragon CUDA 256 从 ~37s 降到 1.7s。

**本轮(实验驱动)新增**:各向异性 SE 核(逐轴 L)、非平稳 σ 场(`--sigmafield/--sigmarange/--sigmaclamp`)、逐轴 AABB margin(`--margin`)、轨道相机(`--yaw/--pitch` 绕物体)、四色方案 vivid 化(voxel/surface/kernel/heatkernel)、`spheregp` 纯 GP 球场景、`--voxelres/--meshmean`。1D 噪声/NEE 门控由编译期改为**运行时「平稳 + 各向同性」**判据。结果见 `~/mupsi2exp/`。

## 阶段完成情况(全部 ✅)

| # | 阶段 | 落点 |
|---|---|---|
| 1 | 数学工具 | `core/math/*` |
| 2 | 几何体定义 | `core/geometry/*`、`core/primitive/*` |
| 3 | 光线求交 | `core/bvh/*`、`core/tracer/intersect.*`、`core/scene/*` |
| 4 | 图片工具 | `host/ppm_util.*` |
| 5 | 光线发射 | `core/tracer/camera.h` |
| 6 | 求交更多信息(uv/texture) | `HitRecord.uv`、`core/tracer/texture.h`(Constant) |
| 7 | 介质定义 | `core/medium/*` |
| 8 | GP 介质 | `core/gp/*`、`core/medium/gpmedium.*` |

## 关键实现

- **光源**(本阶段):`trace.cpp` 改为 **NEE + MIS**(直接光采样 + BSDF 采样,power heuristic)+ 环境光 + 渐变天空(待补 IBL)。参照 Tungsten `TraceBase::estimateDirect/chooseLight`(球形面光源 + 按近似辐射度重要性选光 + 阴影射线);`Specular` 镜面 BSDF + 俄罗斯轮盘已加入。
- **场景**:`SceneConfig`(host)→ `toDevice` 拍平 SoA(含 `vertexNormals`)→ `RenderContext` 传 `trace`。入口 `buildMeshGpScene` 通用化,`./build/mupsi <sphere|spot|bunny|dragon> <w> <h>`。
- **MeshMean 有符号距离**:距离 = BVH 最近点;符号用**角度加权伪法向**(Bærentzen & Aanæs 2005)在最近点判号,对开网格 + 局部翻面稳健。
- **eps 统一 + scale 感知**:`core/tracer/constant.h` 分「固定量」(EPS_SURFACE/EPS_NEAR)与「随 sceneScale 缩放的比例量」(marching 步长/符号阈值/远裁剪);`Medium.sceneScale` 由 builder 设。
- **CUDA 后端**:`gpu.cu` 完整实现(静态几何 7 数组上传、每像素一线程 kernel、RDC 链接、download)。
- **图片输出**:`save_ppm` 按「framebuffer (0,0)=左下角」写 PPM(逐行倒序输出),修正了竖直翻转。

## 验证结果

环境:WSL2 + RTX 4080 Laptop(cc 8.9)。图像经系统 `image_viewer`(deepseek-flash)逐张检查。

| 场景 | 后端 | 分辨率/spp | 耗时 | 结果 |
|---|---|---|---|---|
| 默认 GP 球 + Lambertian 球 | CPU | 160×120/1 | 5.1 s | ✅ |
| 默认 GP 球 | CUDA | 160×120/1 | 2.3 s | ✅ |
| spot 网格 GP | CPU / CUDA | 512×512/1 | 86 s / 17.7 s | ✅ 居中、双耳+身体+腿 |
| bunny 网格 GP(开网格+翻面) | CUDA | 512×512/1 | 31–41 s | ✅ 修复后连贯 blob,无假面 |
| dragon 网格 GP(25 万三角) | CUDA | 512×512/1 | 147–154 s | ✅ 居中,无伪影 |

> surface 细密散斑、弱方向光照是 GPIS 预期(σ=1、L=(1,1,1) 噪声与均值梯度同量级);1spp 加重走样。

## 稀疏卷积性能优化(对齐 Xu 2025 官方 Tungsten 实现)

官方仓库(不是 mitsuba2,是 Tungsten):`github.com/dartmouth-vcl/sparse-conv-gpis-tungsten`。对照它找到我们慢的三个根因,并逐一修复:

| 官方实现 | 我们(改造前) | 状态 |
|---|---|---|
| `cell3D` 里 `if(to_point.lengthSq()<1) continue` —— 支撑半径外不碰 exp | 81 脉冲每个都 exp | ✅ `gpnoise.cpp` 紧凑支撑剔除 |
| `noise1D/cell1D` —— marching 沿光线只遍历 3 cell(不是 27) | 只写 3D | ✅ `eval1D`(仅 Renewal/Renewal+) |
| 每 cell `set_state(xxhash32(ijk)+1)` 顺序抽取,无 warmup | 每脉冲 new Random + 2 warmup | ✅ 每 cell 一 RNG |

**1D 噪声**(论文 477/520 行):只在 **平稳各向同性 + Renewal/Renewal Half+/Renewal+** 下可用(SingleRealization 需要全图一致的一个 3D 实现,不能沿线抽 1D;非平稳 σ/L 场也破坏子域不变性,必须回退 3D)。命中点梯度 = 沿线导数 g_z·d + 垂直分量 g_xy ~ N(0, σ²/L²)(论文 §6.2.1:g_xy ⊥ g_z,各向同性 2D 高斯)。判据(**运行时**):`isIsotropic()` + `kernelField.empty()` + `gpMode != SingleRealization`。踩坑:σ 场 + Renewal+ 仍走 1D 会算出错误的零等值面(硬水平切断 bunny 耳),已修。

**体素网格 μ**(论文 §7,OpenVDB 风格):稀疏窄带网格替代每步最近点 BVH。
- `SparseSDFGrid`:密集 1 字节符号网格(res³)+ 排序窄带数组(bandIndex/bandValue,仅 |sdf|<band 存全精度);查 = 二分搜窄带 + 三线性值/梯度;带外节点回退 ±band。
- host `buildSparseSDFGrid`(OpenMP 并行):对每个格点算一次 mesh SDF(最近点 + 角度加权伪法向判号),保留窄带。res=256、band=4 cells。
- 上传/重绑定与 meshMean 同一条路(SceneDevice/SceneHost 增加三个体素数组)。**窄带宽度按峰值 σ 缩放** `band = 4σ_peak/cellSize + 2`,否则大 σ 下 ±band 回退被噪声淹没、在包围盒内产生伪曲面(σ=3/15 时 bunny 曾渲染成「平盒子」)。

**实测**(512×512 为旧的 mesh/BVH 数字):

| 场景 | 后端/分辨率 | 优化前 | 优化后 | 提速 |
|---|---|---|---|---|
| sphere(Renewal+) | CPU 160×120 | 5.1 s | 0.40 s | ~13× |
| sphere(Renewal+) | CUDA 512 | ~14 s | 1.8 s | ~7.7× |
| spot(SingleRealization) | CPU 256 | ~21.5 s | 4.9 s | ~4.4× |
| spot(SingleRealization) | CUDA 256 | 14.9 s | 1.33 s | ~11× |
| dragon(SingleRealization) | CPU 256 | ~37 s | 4.0 s | ~9× |
| dragon(SingleRealization) | CUDA 256 | ~37 s | 1.70 s | ~22× |

> dragon 体素构建一次 ~30s(OpenMP 32 核,16.7M 次 BVH 查),渲染不计入;spot ~4s。
> 所有体素图像经 image_viewer 验证 PASS(spot 与 mesh 参考图视觉一致,dragon 形状正确、无符号错误/带边界伪影)。
> **体素薄尖局限**:薄于 1 cell 的特征(如 bunny 耳尖)零等值面在体素上被水平截断;`--meshmean`(精确 mesh SDF)可完整还原,但 ~20× 慢。

## 论文场景复刻(进行中)

已能复刻论文的核心光照路径(面光源 NEE+MIS、完美镜面微表面 GPIS、envmap IBL),导出到 `~/mupsi2test/`:

| 论文图 | 场景 | 命令 | 状态 |
|---|---|---|---|
| Fig 14 | dragon(平稳核近似)+ envmap + Renewal+ + MIS | `dragon 512 512 renewal lambertian 64` | ✅ `dragon_lambertian_512_s64.png` |
| Fig 15 | dragon 镜面微表面 + envmap + Renewal+ | `dragon 512 512 renewal specular 64` | ✅ `dragon_specular_512_s64.png` |
| Fig 1/15 | sphere 镜面微表面 + envmap | `sphere 512 512 renewal specular 64` | ✅ `sphere_specular_512_s64.png` |

CLI 已扩展:`mupsi <scene> <w> <h> [single|renewal] [lambertian|specular] [spp] [sigma]`。

**诚实局限**(已核实):
1. **✅ 镜面 GPIS 已升级到 §6 解析梯度分布 NEE+MIS**(`core/gp/gradient_nee.h`):`toGxy`(图 11 反解反射方向)、`evalGxyPdf`(N(0,σ²/L²) NDF)、`toGxyMeasure`(立体角→梯度测度 Jacobian)、`phaseEval`(方向空间 NDF)。**运行时**门控 `mediumIsIsotropic()` + `kernelField.empty()`(平稳)+ `gpMode != SingleRealization`。踩坑:NDF 与光 pdf 同时 ~0 时 power heuristic 权重 0/0=NaN 致黑块(已 guard)。仍待:与参考实现逐像素收敛对拍(UNI vs MIS 同像验证无偏)。
2. **非平稳协方差(Eq. 25)已实现(对角形式 + 空间 σ 场)**:逐轴 `lengthScale`(对角 M)+ `KernelField` σ 斜坡/阶跃场(`--sigmafield/--sigmarange/--sigmaclamp`)。龙场景默认仍是平稳 SE 核近似。

**下一步(按用户定)**:① ✅ §6 解析梯度分布 NEE;② 非平稳核 Eq.25(体素网格 σ/Σ + M 变换);③ GPIS 纹理渲染(三方向开分支:体素颜色插值 / 表面映射 / 空间颜色核)。

## 复刻论文场景的路线图(缺口清单)
论文(Seyb et al. 2024)场景更复杂,主要是**光照/材质**而非 GPIS 数学。对照原 mupsi(`../mupsi/src/rendering/trace.cpp`),当前 mupsi2 缺:

1. **✅ NEE + 阴影射线**:`light.h::chooseLight/sampleLightDirect`(球形幅采样,Tungsten `Sphere::sampleDirect`)+ `sceneOccluded`(跳过 Null、排除目标光)+ GP `transmittance` 沿阴影射线。已替代固定太阳。
2. **✅ 发光图元(面光源)**:`AreaLight`(球形,常量发射)+ `SceneHost::addLight`(光既注册发射又作为可遮挡 Lambertian 球)。spot/球/dragon 各两个面光源(key+fill)。
3. **✅ Skydrome / IBL**:`host/hdr_loader.*`(Radiance .hdr 装载)+ `EnvMap`(equirectangular 双线性方向查找)+ `RenderContext.envMap` + `background()` 采样环境图。`~/models/envmap.hdr` 已接入(镜像球可见反射)。
4. **✅ 完整 BSDF + 俄罗斯轮盘 + MIS**:`BsdfType::Specular`(完美镜面, delta 瓣,跳过 NEE 保留 specularness)+ RR(throughput 存活)+ `powerHeuristic` 合并 light-sampled / BSDF-sampled 直接光(论文 Algorithm 4)。
5. **Bitmap 纹理 + 材质绑定**:`texture.h` 当前只有 Constant;需 BitmapTexture(PNG/JPG)+ `texId` 采样(`HitRecord.uv` 已备)。
6. **✅ GP 阴影透射**:`nextEventEstimate` 阴影射线先 `transmittance()` 后 `sceneOccluded()`,GP 面可自遮挡投影。
7. **✅ Renewal+ 逐弹条件化**:`trace` 每 bounce 用 `make_seed(seed, bounce)` 作介质种子(sampleDistance/shadow 透射),`sampleCondition` 用 `bounce+1`;SingleRealization 全图固定一 realization。对齐原 mupsi `make_seed(pixel,spp,seed,bounce_times)` 链。
8. (可选)可渲染 mesh 硬表面 + 材质,以及 OpenVDB/体积。

对应论文图:Fig 22(bunny GPIS)、Fig 19(dragon)、Fig 7/9(SDF 球)需 1–6;Fig 24(树)需 5 + 3。

## 避坑(见 TEST.md 详述)

- bunny 是开网格 + 局部翻面:奇偶射线判号怕开洞,面法向怕翻面 → 用角度加权伪法向。
- `u32_to_f` 别用 `cuda::std::bit_cast`(libcu++ 未链接)。
- 首跳 `inMedium` 必须从 `false` 起,先命中包裹球再进介质。
- CPU/CUDA 仅结构一致(物体区逐通道差可达 ~95/255):GPU FMA + `expf` 差异,GP 零等值面附近放大为不同散斑。
- 图片竖直翻转:save_ppm 必须按左下角原点倒序写行。

## 待统一处理的问题(Step 1/2/3 遗留)
1. **§6 NDF NEE 高方差**(`sphere ... specular`,σ=0.5):功能正确(球实心、glint 落位正确、无 NaN),但粗糙镜面 + 小面光源固有高方差 → 64spp 仍像 TV 雪花。要读得出模糊 envmap 需 σ≤0.15、更大面光源、或更多 spp;待 UNI vs MIS 逐像素收敛对拍验证无偏。
2. **非平稳核(Eq.25)已落地**:(a) 逐轴各向异性 `lengthScale`(§6/§8 对角 M);(b) 空间 σ 场 `--sigmafield x|y|z --sigmarange min,max [--sigmaclamp]`,σ 沿轴在网格 bbox 上线性插值/阶跃,单张图内底部稳定→顶部发散(image_viewer 验证)。`dragonns` 的 L 随高度场仍是早期测试。
3. **纹理三方向已 vivid 化**:`colorfield.h` 的 `fieldColor` 改为 6 段彩虹(全饱和)、`kernelColor` 改为 4 主色锐利高斯核(红/蓝/绿/黄);`buildColorGrid` 现在烤 `fieldColor`(voxel=彩虹,与 surface 同源),`kernel` 分支是 4 主色块。三方向 image_viewer PASS、颜色强烈。
4. **热核纹理(HKTEx)已 vivid 化**:`precompute_heat.py` 存**实际色**(非残差,金色角间距色相)、`HeatKernel::evalColor` 按 `Σα` 直接归一(非 `max(Σα,1)`,稀疏覆盖下不再变灰)+ 色度增强 ×2.6。多核重叠仍会向灰平均(测地扩散内在偏软),但已是「彩虹色相齐全」;voxel/kernel 更高饱和。
5. 全流程均为 8spp 冒烟;最终图需 64–256spp。

6. **地板/墙壁 + 1024 渲染 + CLI**(最新一轮):
   - `scene_builder` 加 `addQuad`/`addFloorWalls`(地板 y=-300、背墙 z=-850,Lambertian);修了两个 bug:(a) `scene.cpp` bvhTriIndices 存 `m.faceOffset+ti`(u32 单位+三角索引,第二个 mesh 起偏 3×→段错误)→ `m.faceOffset/3+ti`;(b) `trace.cpp` 出介质后 `ray.max` 仍钳在包裹球出口→地板/墙看不到→ 退出介质时 `ray.max=1e30`。
   - CLI:命名参数 `--scene/--width/--height/--mode/--bsdf/--spp/--sigma/--color/--heat/--output` + `--help`;`Surface` 改用解析 RGB 梯度(与 `Kernel` 区分)。
   - 四分支 1024×1024 已渲染 `~/mupsi2test/bunny1024_{voxel,surface,kernel,heatkernel}_v2.png`,地板/墙可见。
   - **遗留**:地板/墙过曝(近白),冲淡了兔子本身的颜色;且地板/墙没铺满视野(四角仍见 envmap)。→ 已修(见下 7)。

7. **修复「大球包裹兔子」+ 过曝 + 地板/墙不铺满**(用户反馈 4 点,image_viewer 逐轮验证):
   - **根因**:透明(Null)包裹球被当作不透明球体着色,生成一个硬边大圆盘覆盖全帧;四角露 envmap;地板/墙过曝。
   - **修法**:
     - `trace.cpp` 改为**直接对 boundCenter/boundRadius 做 ray-sphere 进入测试**(`sphereEnterT`),**包裹球不再作为场景图元**(不可能被着色);删除 Null 进入分支。
     - `trace.cpp` 首跳及硬表面 `throughput` 改为乘**实际 albedo**(原先恒乘 `lambertianAlbedo()`,导致多弹跳能量不对)。
     - 硬表面(地板/墙)用**暗中性色** `{0.28,0.26,0.28}`;面光降为 `{5,4,3.5}/{2,1.6,1.4}`;地板抬到 y=-150、背墙加高到 y=+400 铺满视野。
     - `main.cpp` envmap 载入时乘 **0.15×** 防止反射过曝。
   - **验证**:`bunny_v5_512.png` image_viewer 判 **PASS**——大圆盘消失,兔子清晰、背景暗墙/地板无 envmap 缺口。
   - **遗留(小)**:MC 噪点(低 spp);墙面淡横纹(欠采样);兔子边缘软光晕(内散射);地板左下角有一小暗斑待查(疑似接触阴影/采样噪点,非 bug)。

## 已知局限 / 下一步

1. **单像素位一致**:如需逐像素对拍,须 `-fmad=false` 并统一 `expf`。
2. **光照/材质**:路线图 1/2/3/4/6/7 已落地;item 5(纹理)已开三分支(见上)。
3. **镜面 GPIS**:§6 解析梯度分布 NEE 已落地(见上问题 1)。
4. **非平稳协方差**:Eq.25 对角形式(逐轴 L)+ 空间 σ 场已落地;旋转/剪切 M、多分辨率稀疏卷积(论文 469 行)未做。
5. **体素网格**:(a) res 可上调(`--voxelres`);(b) 带外回退 ±band 是 C⁰ 截断;(c) 构建 O(res³);(d) **薄于 1 cell 的特征被水平截断**(bunny 耳尖),`--meshmean` 可完整还原但 ~20× 慢——将来可做「体素主体 + 薄特征回退 mesh 查询」混合 mean。
6. 极度破损网格下角度加权伪法向仍可能留少量伪影。
7. `docs/` 仍被 `.gitignore` 忽略(本地文档)。
8. 实验结果(`~/mupsi2exp/`):hero 四色 2048×1024×16spp、σ 扫描、各向异性核对比、24 张交叉矩阵(4 色×3σ×2 核)、σ 场颜色对比——含 `table.csv`(参数+耗时)与 `README.md`。

## 构建 / 运行

```bash
cd ~/mupsi2
cmake -S . -B build-cpu -G Ninja -DMUPSI_ENABLE_CUDA=OFF && cmake --build build-cpu -j$(nproc)   # CPU
cmake --build build -j$(nproc)                                                                      # CUDA
./build/mupsi --scene bunny --width 1024 --height 1024 --spp 4 --noenv --color voxel
# 关键 CLI(命名参数,均可组合):
#   --scene sphere|spheregp|spot|bunny|dragon|dragonns
#   --mode single|renewal   --bsdf lambertian|specular
#   --sigma S   --lscale Lx,Ly,Lz(各向异性)   --margin mx,my,mz(σ 倍数,默认 3,5,3)
#   --sigmafield x|y|z --sigmarange min,max --sigmaclamp(空间中点阶跃)
#   --color constant|voxel|surface|kernel|heatkernel [--heat file.bin]
#   --yaw deg --pitch deg(绕物体轨道)   --voxelres N --meshmean --noenv --output out.ppm
```
