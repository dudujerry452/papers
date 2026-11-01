# mupsi2 — 测试方法论与记录

> 方法论 + 调试过程。结果见 `docs/STATE.md`(以 STATE 为准)。设计见 `DESIGN.md`,数学见 `gpis-1.md`/`gpis-2.md`。

## 0. 环境

- WSL2(`nixos`),`.envrc = use flake .#cuda`(nvcc 12.9)。
- GPU:NVIDIA GeForce RTX 4080 Laptop,cc 8.9(经 `cudaGetDeviceCount` 确认;`nvidia-smi` 不在 PATH)。
- CPU:32 核。两个构建目录:`build-cpu`(g++,CPU)、`build`(nvcc,CUDA)。
- 图像查看:系统全局 agent `image_viewer`(`~/.pi/agent/agents/image_viewer.md`,自带 deepseek-flash 视觉模型)。

## 1. 构建 / 运行

```bash
cmake -S . -B build-cpu -G Ninja -DMUPSI_ENABLE_CUDA=OFF && cmake --build build-cpu -j$(nproc)
cmake --build build -j$(nproc)     # build/ 已配 CUDA
./build/mupsi <scene> <w> <h>      # scene ∈ sphere|spot|bunny|dragon
# PPM → PNG 后交给 image_viewer 读图
```

## 2. 场景(带 GP + ~/models 几何体)

`buildMeshGpScene(objPath, w, h, dist, targetRadius, mode, sigma, kernelRadius)`:
自动居中 + 缩放 mesh 到 `targetRadius`(默认 150),放在 (0,0,-dist),包进 Null 包裹球,
相机原点正视。`buildSpotGpScene` / `buildBunnyGpScene` / `buildDragonGpScene` 是薄封装。
dragon 模型:`~/models/dragon.obj`(从 alecjacobson/common-3d-test-models 下载的 `xyzrgb_dragon.obj`,25 万三角形)。

## 3. 已发现并修复的问题

1. `random.cpp: u32_to_f` 在 `__CUDA_ARCH__` 下误用 `cuda::std::bit_cast`(未链接 libcu++)→ 改 union 位重解释。
2. `trace.cpp` 首跳 `inMedium=ctx.hasMedium` 错误(没进包裹球就 marching)→ 改 `false`。
3. **bunny 渲染出现细长"假面"**(先是一条到左边缘的横条,后是一条竖直竖条)。
   根因:`~/models/bunny/bunny.obj` 是**开网格**(42 条边界边)+ **局部翻面**(如 tri 1700 法向指向内部)。
   - 奇偶射线判内外(inside)要求封闭 → 对开网格失效(横条)。
   - 最近三角面法向判号 → 对局部翻面失效(竖条)。
   修法:改用 **角度加权伪法向**(Bærentzen & Aanæs 2005)在最近点判号——对开网格 + 局部翻面都稳健。
   实现:`SceneHost/SceneDevice` 增加 `vertexNormals`;host 预计算;`MeshMean::evalGrad` 在最近点按重心坐标插值伪法向。
4. **CUDA 后端 `gpu.cu` 原是占位壳** → 完整实现(静态几何 upload、每像素一线程 kernel、download)。
5. **`SceneDevice` 缺共享数组元素计数** → 加 `numVertices/numFaces/numBvhNodes/numBvhTriIndices`(upload 需要)。

## 4. 代码组织(用户要求)

- **eps 集中到 `core/tracer/constant.h`**:
  - 固定量(不随场景缩放):`EPS_SURFACE=1e-3`、`EPS_NEAR=1e-2`(防重复撞击的绝对偏移)。
  - 随场景缩放的比例量(使用处乘 `medium.sceneScale`):`MARCH_MAX_DT_FRAC=1/300`、`MARCH_FAR_CLAMP_FRAC=20`、`MARCH_SIGN_EPS_FRAC=1e-8`、`SDF_MIN_STEP_FRAC=1e-5`。
  - 无量纲:`MARCH_MIN_DEPTH=100`、`SDF_MAX_ITER=512`、`SDF_BISECT=16`。
- **场景 scale 特殊处理**:`Medium.sceneScale`(≈物体半径)由 scene builder 设置;marching 步长/符号阈值/远裁剪随 `sceneScale` 缩放,而 `EPS_SURFACE/EPS_NEAR` 保持绝对。

## 5. 时间限制

GP 已大幅提速(1D 噪声 + 体素网格):512×512 CUDA 约 sphere 1.8s / spot ~5s / bunny 待测 / dragon ~7s;CPU 相应更快。
体素网格构建一次 ~30s(dragon,OpenMP 32 核,16.7M 次 BVH 查;spot ~4s)。
测试时先 128×128 冒烟,再 512×512;`timeout` 设 300s。

## 6. 工作流程

在 main 分支直接迭代(首轮已从 `test/cpu-cuda` 合并回 main);每改一处 → CPU/CUDA 编译 + 运行 + image_viewer 读图验证;结果写 `STATE.md`。

## 7. 规范化测试(路线图阶段)

### 7.0 前置:CLI 模式覆盖

`./build/mupsi <scene> <w> <h> [single|renewal]` —— 第 4 参数覆盖场景默认 GPMode(`single`=SingleRealization,`renewal`=RenewalPlus),缺省用 builder 默认(sphere=renewal,spot/bunny/dragon=single)。

### 7.1 视觉验证矩阵(新功能:光照/材质/BSDF)

每加一个新功能(面光源/NEE/skydrome/纹理/Specular/RR/…)后,跑满矩阵并交 image_viewer 核验:

| 场景 | mode | 分辨率/spp | 输出 |
|---|---|---|---|
| sphere | single | 512×512/1 | `~/mupsi2test/sphere_single_512.png` |
| sphere | renewal | 512×512/1 | `~/mupsi2test/sphere_renewal_512.png` |
| spot   | single | 512×512/1 | `~/mupsi2test/spot_single_512.png` |
| spot   | renewal | 512×512/1 | `~/mupsi2test/spot_renewal_512.png` |
| dragon | single | 512×512/1 | `~/mupsi2test/dragon_single_512.png` |
| dragon | renewal | 512×512/1 | `~/mupsi2test/dragon_renewal_512.png` |

命令(逐条):`./build/mupsi <scene> 512 512 <mode>` 后把 `test.ppm` 转 png 存到上表路径。

image_viewer 核验项(每次一致):
1. 形状/轮廓符合模型(球/spot/龙);
2. 无符号错误(洞、缺部件、内外翻转、游离 blob、假面);
3. 无带边界/体素伪影(硬接缝、台阶、方块);
4. 无噪声爆炸/黑区/条纹;
5. 结论 PASS/FAIL。

### 7.2 逐像素对拍(优化类改动,不改变输出)

**适用范围**:只做「不改数学结果」的优化(合并求值、缓存、换等价数据结构等)。新功能会改变输出,走 7.1 视觉核验。

**方法**:
1. 同一场景 + 同一后端 + 同一 mode,优化前后各渲一张(固定 seed,确定性输出);
2. `python3 tools/ppmdiff.py a.ppm b.ppm` 对比。

**判据**(每通道 max abs diff):
- `0` = 位一致(最优);
- `≤ 2/255` = 浮点重排可接受;
- `> 2/255` = 结果变了 → 改成数学等价,或改走 7.1 视觉核验并说明理由。

**注意**:CPU vs CUDA 因 FMA+`expf` 差异**不强制位一致**(零等值面附近散斑可差 ~95/255),对拍只在**同一后端**内做;跨后端位一致需 `-fmad=false` + 统一 `expf`。

## 8. §6 解析梯度分布 NEE + Eq.25 非平稳核的测试

- **§6 NDF NEE(镜面微表面 NEE)**:
  - 单元:sphere/spot `renewal specular`(粗糙档 σ=0.5、平滑档 σ=0.1)各渲 512×512/64spp;image_viewer 核验「粗糙镜面」(模糊反射可见天空/地面分离,无黑块/飞蝇)与「抛光镜面」两档;
  - 编译期门控:在 `SEKernel::isotropic=false` 或 `GPMode=SingleRealization` 时 `static_assert`/编译期回退到 UNI(镜像递归),不产生未定义行为;
  - 逐像素对拍:镜面微表面在 `single` 模式下应为确定输出(同后端同 seed 位一致)。
- **Eq.25 非平稳核**:
  - dragon + 空间 σ/Σ(如 σ 随到表面距离变化)各渲 512/64,image_viewer 核验「长度尺度空间变化」的视觉差异(远处粗、近处细);
  - 与平稳核同 σ 时位一致/近一致(退化验证)。
- **GPIS 纹理渲染(三方向,各开分支)**:
  - 方向 1(体素颜色插值)、方向 2(表面映射)、方向 3(空间颜色核)各自 512/64 出图到 `~/mupsi2test/`,image_viewer 核验颜色是否正确附着表面。
