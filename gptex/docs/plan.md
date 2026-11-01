# mupsi2 实现计划

重写 GPIS 软渲染器(mupsi)为**可更换 CPU/CUDA 后端**的 C++20 CMake 项目。

## 1. 目标与约束

- **完全独立跨平台编译**的 C++ CMake 项目。
- **允许依赖**高知名度跨平台包(如 Eigen 于 host、nlohmann_json、fmt、GLFW/ImGui)。
- **基础设施自己手写**:BVH、RNG、GP 核函数、BSDF、medium 步进、path tracer、以及后端 dispatch 层。
- **可更换后端**:CPU / CUDA,按功能粒度可插拔(某个功能可以加新后端,也可以不加)。

## 2. 核心架构决策

### 2.1 可更换后端 = 编译期模板 / 宏,而非运行时多态

- 后端切换是**编译期**的,零虚函数开销。
- 数值逻辑用 `MUPSI_HOST_DEVICE` 宏标记,同一份 `.inl` 被 g++(CPU)和 nvcc(CUDA)各编译一次。
- 资源/launch 层面的差异用**模板特化**:`template<Backend B> struct Dispatch`,CPU 特化是 `#pragma omp parallel for`,CUDA 特化是 `<<<grid,block>>>`。
- 一个功能没有某后端实现 = 主模板未特化 → 编译期报错(或显式回退),不静默退化。

### 2.2 函数式内核

- 内核是**纯函数**:输入 query → 输出结果,无状态、无副作用、无全局可变状态。
- 参数打包成 POD struct,不收类引用/`shared_ptr`。
- 全局状态(g_settings、medium 指针、default_bsdf)全部显式传参,`RenderContext` 作为不可变快照。

### 2.3 文件分层:内核 / 后端壳 / host

```
src/
├── core/           # 纯头:types.h(类型别名层)、platform.h(宏)、rng.inl
├── kernels/        # ★ device 闭包:渲染核心实现,全是 .inl
├── backends/       # ★ 后端壳:每个后端一组 .cpp/.cu,include 上面的 .inl
│   ├── cpu/
│   └── cuda/
├── host/           # ★ host-only:照常 .cpp
├── device.h        # 抽象 Device 接口
├── renderer.cpp    # 编排
├── editor/
└── main.cpp
```

- **内核实现进 `.inl` 头文件**(不是 `.cpp`),因为 `.cpp` 只能被一个编译器编一次,而内核要同时被 g++ 和 nvcc 各编一次。
- 普通 host 逻辑(IO、构建、线程、editor)照常放 `.cpp`。
- 划分标准不是"内核 vs 非内核",而是"**device 执行闭包内的代码 vs host-only 的代码**"。

### 2.4 数据搬运调度 = 生命周期分桶

| 桶 | 数据 | 搬几次 | 时机 |
|---|---|---|---|
| 静态资源 | 场景几何(BVH/mesh)、GP 参数 | 1 次 | 初始化 upload |
| 帧级 | 相机、settings、seed | 每帧 1 次 | 帧开始 |
| 流式中间数据 | query 点、噪声结果、conditioning | 0 次 | 同侧闭环 |
| 输出 | framebuffer | 1 次 | 帧末 download |

- 搬运只在**数据域边界**发生一次,不在函数边界。
- 数据域整体同侧(噪声/BVH/medium 步进同后端),避免跨 PCIe 高频往返。
- 异步流 + 双缓冲 + pinned memory 隐藏搬运延迟。

## 3. 为 CUDA 预留的约束(现在写 CPU 也要遵守)

1. **类型层隔离**:内核绝不直接写 `Eigen::Vector3f`,统一用 `Vec3f`/`Vec4f` 别名(CPU=Eigen,CUDA=float3)。
2. **文件分层**:device 内核进 `.inl`,host 逻辑进 `.cpp`。
3. **内核禁 `std` 容器/`shared_ptr`/虚函数/递归/`new`**。
4. **SoA 布局**:BVH 节点、mesh 顶点、参数用扁平数组,别 `vector<大struct>`。
5. **`MUPSI_HOST_DEVICE` 宏现在就标**(CPU 展开为空,标记"将来上 device")。
6. **RNG 纯整数**(PCG/xxhash),保证 CPU/GPU 位一致。
7. **内核统一 float,禁 double**。
8. **参数打包 POD struct**。
9. **控制流**:BVH 遍历用显式栈(不递归,device 栈小);medium 步进的变长 while 将来 wavefront 化。

## 4. 实现顺序

自底向上 + 通用光追器优先,GPIS 后置。每阶段**以"与现有 mupsi 输出对齐"收尾**。

### 阶段 0 — 数学地基

- `core/types.h`(Vec3f/Vec4f 别名层)、`core/platform.h`(`MUPSI_HOST_DEVICE`)、`core/rng.inl`(PCG/xxhash/`make_seed`)。
- **验证**:RNG 对固定 seed 输出已知值(锁定位一致性基准)。
- **为 CUDA 预留**:RNG 全整数、宏从第一天就标。

### 阶段 1 — 显式几何 + ray tracing

- `ray`、`intersection`、`primitive`(sphere/mesh)、`bvh`(SoA 扁平节点 + 显式栈)、`scene`(`intersect`/`occluded`)。
- **验证**:单球 + 一个 mesh 交点正确。
- **为 CUDA 预留**:BVH 节点 SoA 扁平数组。

### 阶段 2 — BSDF

- `bsdf` 接口 + lambertian/null/specular,`SurfaceScatterEvent`/`SurfaceScatterSample`。
- 去掉 `default_bsdf_` 静态,显式持有。
- **验证**:每个 BSDF 的 eval/pdf/sample 对拍。

### 阶段 3 — 光追核心:RTE + NEE

- `camera`(generateRay)、`trace`(递归 RTE)、`framebuffer`、`renderer`(并行像素循环)。
- **RTE**:throughput 累积 + emission + 俄罗斯轮盘。
- **NEE**:`chooseLight` + shadow ray + BSDF weight。
- 去掉 settings 全局,走 `RenderContext`。
- **验证**:球 + 面光源 + lambertian 渲染出正确直接光照,与现有 mupsi 对齐。**到此一个完整标准路径追踪器闭环。**

### 阶段 4 — ray marching(medium 抽象)

- `medium` 接口(`sampleDistance`/`transmittance`)+ 一个最简单体积介质(SDF/均匀密度)验证 march 循环。
- **关键结构决策**:交付的是 `Medium` 抽象接口,不是 GP。GPIS 只是接口的一个实现。
- **验证**:一个体积雾/云渲染正确。

### 阶段 5 — GPIS 接入(后置特性)

- `kernel`、`gpnoise`、`mean`、`gpmedium`、conditioning(Renewal+)。
- 把 `GPMedium` 实现成阶段 4 的 `Medium` 接口实现;噪声/kernel 纯函数 + 参数打包。
- **验证**:GP 介质渲染与现有 mupsi 对齐(single_realization / renewal_plus)。

### 阶段 6 — IO + 入口 + editor

- `config`、`controller`、`main`、editor(ImGui/GLFW)最后挂。

## 5. 基础设施(已完成)

- **flake.nix**:两套 devShell —— `.#cpu`(纯 CPU,无 Eigen)、`.#cuda`(nvcc + cuBLAS + CUTLASS);nixpkgs 锁定 ~/nixos 同 rev 复用缓存。
- **CMakeLists.txt**:CPU 核心无条件编译;`check_language(CUDA)` 检测到 nvcc 才引入 CUDA 后端。
- **git**:`github.com:dudujerry452/mupsi2`,main 分支。
- **direnv**:本机 `.envrc` = `use flake .#cpu`,wslw `.envrc` = `use flake .#cuda`,各自本地、不入库。

## 6. 验证基准

- 现有 mupsi 的 `test.png`、`config.json`、`scenes/*.json` 作为逐像素对齐的回归集。
- 阶段 3 之后每个阶段都能渲染对照,不攒到最后一起对。
