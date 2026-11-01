热核纹理：不溅射的测地高斯

Simone Foti*，Caner Korkmaz*，Stefanos Zafeiriou，和 Tolga Birdal
伦敦帝国理工学院

图 1：HKTEx：一种使用热核的内在无 UV 纹理表示。左图：黎曼框架将核的运动限制在表面上，随后由流形感知控制器管理核的形状和密度。评估在可微分光线追踪管线中进行。右图：最终基于物理的渲染结果。

摘要。3D 高斯溅射最近彻底改变了新视角合成以及许多其他 3D 视觉方法和应用。受此表示方法的启发，我们重新思考纹理处理以克服 UV 映射的主要问题，同时显著降低其内存占用。热核纹理（HKTEx）消除了 UV 展开及其长期存在的 UV 空间浪费、接缝、畸变、顶点重复和分辨率变化等问题。基于离散黎曼几何并在任何流形表面（离散化为三角网格）上固有定义，HKTEx 使用各向异性热核作为高斯的测地线等价物。与我们的核类似，其位置优化和自适应致密化策略也被重新定义以在待纹理化对象的表面上运行。我们的新表示法还完全集成了基于物理的渲染器，并可以从现有纹理或多视图图像进行优化。我们的项目页面和代码可在 circl egroup.github.io/research/HeatKernelTextures 获取。

关键词：高斯溅射·热核·纹理·测地线

1 引言
过去几十年中渲染和表示技术呈指数级发展，但物体外观的表示仍主要锚定在传统且存在根本缺陷的 UV 映射中。通过这种表示，网格表面上的点被映射到图像平面，其中
* 同等贡献

![第 1 页](tmp/single_893159bdaee7576c5879caeed57c816c/1.png)

存储外观属性不仅会导致接缝、分辨率变化和失真等视觉伪影，还会产生额外的存储成本，包括保存与网格每个顶点关联的UV坐标以及重复接缝线上的顶点。空的UV映射区域进一步加剧了内存浪费。为克服这些限制，我们主张采用一种在物体表面固有定义的新表示方法，该方法能够降低内存占用，并有望惠及游戏、虚拟现实、增强现实和视觉效果领域。

多项研究尝试缓解UV相关问题，最显著的是通过隐式神经表示[19,26]或彩色点云[12,45,46]。尽管前景广阔，但这些方法分别存在显著的内存占用、呈现拓扑依赖结构或需要大量基元的问题。最近，三维高斯泼溅（3DGS）[18]通过一种紧密耦合几何与外观的表示方法革新了新视角合成技术。然而，标准3DGS完全在环境欧几里得空间中运行。当应用于表面纹理时，这一假设成为根本缺陷：欧几里得优化自然会使高斯函数脱离几何结构，而刚性3D高斯函数本身无法弯曲以贴合底层流形。尽管近期方法[8,14]尝试通过引入额外正则化损失将高斯函数对齐到表面，但根本性的脱节仍然存在。

相比之下，我们引入了一种真正的内在测地表示，其中纹理基元原生贴合网格的现有几何结构。我们不在环境3D空间中操作，而是用各向异性热核替代欧几里得高斯函数，直接在流形表面定义所有操作。具体而言，核位置通过黎曼测地线优化进行更新，我们的自适应密度控制也通过核分裂期间的指数映射尊重流形特性。为持续高效地评估表面核，我们开发了新技术来参数化核各向异性，同时桥接连续和离散热扩散公式。我们的公式实现了与基于物理的渲染管线的无缝集成，其中纹理可进行可微分光线追踪。需要注意的是，我们的热核并非泼溅生成，而是在渲染过程中光线相交时动态评估。不失一般性，本研究聚焦于反照率纹理属性，从而模拟光线反射方式。我们证明热核纹理（HKTEx）可以压缩现有UV纹理，并可用于从2D照片进行逆向渲染，从而实现无失真和无缝的纹理获取。

总结而言，我们的主要贡献如下：
1. 一种基于离散黎曼几何的新型内在纹理表示，无需UV映射即可高效表示纹理。
2. 一种黎曼优化框架，严格约束核运动和动量到网格表面。
3. 专为沿物理渲染和连续纹理评估操作而设计的流形感知密度控制器。
4. 将 HKTEx 集成到可微分的 PBR 流程中，实现从多视角图像进行无失真纹理优化。

2 相关工作

神经纹理表示。为规避传统 UV 映射的持续伪影，大量研究致力于采用基于坐标的神经网络来表示表面外观。早期隐式方法直接在 3D 欧几里得坐标上评估多层感知机（MLP）[26]。为提高空间表达能力，后续方法引入了分层或多分辨率的逐顶点特征编码，这些编码经插值后输入 MLP [21]。其他混合方法（如 NeuTex [42]）尝试在神经评估前将 3D 点映射到展开的 2D 空间。虽然这些方法减少了对手动 UV 展开的依赖，但其本质上仍锚定于环境 3D 空间或中间 2D 映射。与我们的数学基础密切相关的是本征神经场 [19]，该方法通过重心插值拉普拉斯 - 贝尔特拉米算子（LBO）的特征向量来条件化 MLP，从而绕过欧几里得坐标。尽管数学形式优雅，基于 MLP 的评估仍存在计算量大且缺乏局部自适应表达能力的缺点。

高斯泼溅与显式表面。基于点的图形和表面泼溅技术长期用于渲染 [51]，但在 3D 高斯泼溅（3DGS）[18] 提出后迎来大规模复兴。3DGS 前所未有的速度和视觉保真度激发了将高斯函数适配到 2D 领域的尝试，例如用显式 2D 高斯函数替代图像像素 [48-50]。为将这种表达能力应用于 3D 网格，近期研究尝试将高斯泼溅与纹理映射结合。然而，这些方法通常回归到使用 UV 参数化，要么将球体的 UV 图像内存储高斯函数 [33]，要么在展开的纹理空间内进行泼溅 [43]。其他高级混合表示（如神经壳纹理 [47]）使用 2D 高斯函数表示局部几何，同时在多分辨率哈希网格中存储颜色 [24]。关键在于，所有这些基于高斯泼溅的方法要么依赖显式 UV 映射，要么在环境欧几里得空间中操作，导致其基元无法内在地弯曲或适应一般拓扑流形。

与基于物理的渲染的集成。虽然 3DGS 设计用于快速光栅化，但其与光线追踪的基于物理的渲染（PBR）的集成仍是新兴前沿。近期研究已将高斯函数纳入 PBR 框架以处理粒子渲染等现象 [10]，并扩展至复杂逆向渲染和重照明任务 [13,44]。然而，这些方法仍依赖于欧几里得基元的体积云。我们的工作与此方向一致，但从根本上改变了范式：我们并非对环境高斯函数进行光栅化，而是将内蕴的、流形约束的热核作为连续可微纹理原生集成，直接在 PBR 引擎的光线交点处进行评估。
3 背景与符号

令 $$ \mathcal{M} = \{\mathbf{X}, \mathbf{F}\} $$ 为一个流形网格，其中 $$ \mathbf{X} \in \mathbb{R}^{N \times 3} = [\mathbf{x}_1 \dots \mathbf{x}_N]^\top $$ 表示用于离散化物体连续表面的 $$ N $$ 个顶点的位置，$$ \mathbf{F} \in \mathbb{N}^{F \times 3} $$ 为面矩阵，索引顶点如何连接成面。我们也将边分组为矩阵 $$ \mathbf{E} \in \mathbb{N}^{E \times 2} $$。$$ \mathcal{M} $$ 表面上的点由对 $$ \mathbf{p} = (\mathrm{f}, \mathbf{b}) $$ 定义，其中 $$ \mathrm{f} \in [1, F] $$ 索引点所在的面 $$ \mathbf{f}_\mathrm{f} \in \mathbf{F} $$，且 $$ \mathbf{b} = [\mathrm{b}_u, \mathrm{b}_v, \mathrm{b}_w]^T $$ 是其在 $$ \mathbf{f}_\mathrm{f} $$ 上的重心坐标。$$ \mathbf{p} $$ 的坐标由 $$ \mathbf{f}_\mathrm{f} $$ 的顶点坐标 $$ \mathbf{x}_{\mathrm{f},0}, \mathbf{x}_{\mathrm{f},1}, \mathbf{x}_{\mathrm{f},2} $$ 重心插值得到：$$ \mathbf{p} = b^{\mathbf{p}}(\mathbf{X}) = \mathrm{b}_u \mathbf{x}_{\mathrm{f},0} + \mathrm{b}_v \mathbf{x}_{\mathrm{f},1} + \mathrm{b}_w \mathbf{x}_{\mathrm{f},2} $$。类似地，重心插值可用于插值定义在网格顶点上的任何其他属性 $$ \mathbf{A} \in \mathbb{R}^{N \times A} $$。多个点 $$ \mathbf{P} = \{\mathbf{p}_j\}_{j=1}^J $$ 上的同步重心插值，下文表示为：$$ b^{\mathbf{P}}(\mathbf{A}) = \mathbf{A}^{\mathbf{P}} \in \mathbb{R}^{P \times A} $$。因此，$$ b^{\mathbf{P}}(\mathbf{X}) = \mathbf{X}^{\mathbf{P}} $$，为简便起见，我们也将其表示为 $$ \mathbf{P} $$。

**Laplace Beltrami 算子。** 我们现在简要介绍 Laplace Beltrami 算子 (LBO) 的各向同性和各向异性 [2, 22, 40] 版本。我们通常将 Laplacian 定义为 $$ \mathbf{L}(\eta, \theta) = -\mathbf{M}\mathbf{W}(\eta, \theta) $$，其中 $$ \mathbf{M} $$ 是对角质量矩阵，其对角线元素与围绕每个顶点的面的总面积成正比 [12, 35]，且 $$ \mathbf{W}(\eta, \theta) $$ 是由权重组成的刚度矩阵：

$$
\mathrm{w}_{ij} = \begin{cases} -\frac{1}{2} \left[ \frac{\langle \hat{\mathbf{e}}_{kj}, \hat{\mathbf{e}}_{ki} \rangle_{\mathbf{H}_{\eta, \theta}}}{\sin(\alpha_{ij})} + \frac{\langle \hat{\mathbf{e}}_{hj}, \hat{\mathbf{e}}_{hi} \rangle_{\mathbf{H}_{\eta, \theta}}}{\sin(\beta_{ij})} \right] & (i, j) \in \mathbf{E}, \\ -\sum_{k \neq i} \mathrm{w}_{ik} & i = j, \\ 0 & \text{else.} \end{cases} \tag{1}
$$

边单位向量之间的内积由剪切矩阵 $$ \mathbf{H}_{\eta, \theta} = \mathbf{R}_\theta \mathbf{U}_{ijk} \mathbf{D}_\eta \mathbf{U}_{ijk}^T \mathbf{R}_\theta^T $$ 加权。$$ \mathbf{U}_{ijk} = [\hat{\mathbf{u}}, \hat{\mathbf{v}}, \hat{\mathbf{n}}] $$ 是包含定义在面上的局部参考系的矩阵，其中 $$ \hat{\mathbf{n}} $$ 是表面的法线，$$ \hat{\mathbf{u}}, \hat{\mathbf{v}} $$ 是定义在面上的正交分量——常见的选择涉及使用主曲率并设置 $$ \hat{\mathbf{u}} = \hat{\mathbf{v}}_M $$ 和 $$ \hat{\mathbf{v}} = \hat{\mathbf{v}}_m $$。$$ \mathbf{R}_\theta $$ 是旋转参考系的旋转矩阵，且 $$ \mathbf{D}_\eta = \text{diag}(\frac{1}{1+\eta}, 1, 1) $$ 是一个控制各向异性的 $$ 3 \times 3 $$ 对角矩阵。当 $$ \eta = \theta = 0 $$ 时，我们有 $$ \mathbf{H}_{\eta, \theta} = \mathbf{I} $$，因此 $$ \langle \hat{\mathbf{e}}_{kj}, \hat{\mathbf{e}}_{ki} \rangle_{\mathbf{H}_{\eta, \theta}} = \hat{\mathbf{e}}_{kj}^T \mathbf{H}_{\eta, \theta} \hat{\mathbf{e}}_{ki} = \cos(\alpha_{ij}) $$ 且 $$ \langle \hat{\mathbf{e}}_{hj}, \hat{\mathbf{e}}_{hi} \rangle_{\mathbf{H}_{\eta, \theta}} = \cos(\beta_{ij}) $$。这将公式 (1) 简化为传统的各向同性 LBO 公式，我们表示为 $$ \mathbf{L} = \mathbf{L}(0, 0) $$。对于 $$ \eta, \theta \neq 0 $$，我们有各向异性 LBO (ALBO)，我们表示为 $$ \mathbf{L}_{\eta, \theta} = \mathbf{L}(\eta, \theta) $$。

LBO 的广义特征分解可以写为 $$ \mathbf{W}\mathbf{\Phi} = \mathbf{M}\mathbf{\Phi}\mathbf{\Lambda} $$，其中 $$ \mathbf{\Phi} \in \mathbb{R}^{N \times K} = [\mathbf{\phi}_1 \dots \mathbf{\phi}_K]^\top $$ 表示定义在网格顶点的前 $$ K $$ 个特征向量，且 $$ \mathbf{\Lambda} = \text{diag}(\lambda_1, \dots, \lambda_K) $$ 是包含特征值的对角矩阵。ALBO 的特征分解遵循相同的过程来确定依赖于 $$ \eta $$ 和 $$ \theta $$ 的 $$ \mathbf{\Phi}_{\eta, \theta} $$ 和 $$ \mathbf{\Lambda}_{\eta, \theta} $$。

![第 4 页](tmp/single_893159bdaee7576c5879caeed57c816c/4.png)

Heat Kernel Textures 5

**热扩散与核 (Heat Diffusion and Kernels)。** 网格上的热扩散过程在数学上描述为：$$ \mathbf{L} \ u(\mathbf{x}_i, t) = \frac{\partial u(\mathbf{x}_i, t)}{\partial t} $$，其中顶点值的标量函数 $$ u : \mathcal{M} \times \mathbb{R}^+ \to \mathbb{R} $$ 表示初始热分布在 $$ \mathcal{M} $$ 上如何随时间 $$ t > 0 $$ 演化。用 $$ \boldsymbol{u}_t $$ 表示通过对所有顶点评估 $$ u(\mathbf{x}_i, t) $$ 获得的向量，热方程的解具有形式 [9,12,34,37]：

$$
\boldsymbol{u}_t = e^{-t\mathbf{L}}\boldsymbol{u}_0 \approx \mathbf{\Phi} e^{-t\mathbf{\Lambda}} \mathbf{\Phi}^T \mathbf{M} \boldsymbol{u}_0 \tag{2}
$$

当初始分布为 delta 函数时，$$ u(\mathbf{x}_i, 0) = \delta(\mathbf{x}_i) = \boldsymbol{\delta}_{\mathbf{x}_i} $$，方程 (2) 被称为*热核 (heat kernel)*，它表示由单点源产生的热量如何在表面扩散。在欧几里得空间中，它具有高斯函数的显式表达式 [36]。注意，虽然可以通过求解稀疏线性系统来模拟扩散，但我们采用的谱方法被证明快一个数量级，并且能够扩展到高分辨率网格 [34]。

**4 热核纹理 (Heat Kernel Textures)**

我们现在详细介绍我们的 HKTEx 模型，该模型使用任意定位的热核直接在流形网格表面上内在地表示纹理。

**核参数 (Kernel parameters)。** 我们将 HKTex 定义为 S 个可优化热核的集合，$$ \mathcal{K} = \{\boldsymbol{\kappa}_s\}_{s=1}^S $$。每个核被参数化为 $$ \boldsymbol{\kappa} = (\boldsymbol{p}^*, \theta, \eta, \tau, \varsigma, \mathbf{c}) $$，其中 $$ \boldsymbol{p}^* $$ 是热源的位置，$$ \theta $$ 和 $$ \eta $$ 是控制扩散过程的角度和各向异性，$$ \tau $$ 是控制核尺度的阈值，$$ \varsigma $$ 是控制其边界锐度的参数，$$ \mathbf{c} $$ 是 RGB 颜色。所有 $$ \mathcal{K} $$ 的参数被联合优化。

**归一化热核 (Normalised heat kernels)。** 在第 3 节中，我们介绍了热核和扩散的离散公式。虽然这是 3D 网格上的黄金标准，但我们寻求摆脱网格离散化并表示超越顶点的纹理。因此，我们利用热核的连续流形公式，并定义如何将离散量转换为连续量。对各向异性地热扩散一个以 $$ \boldsymbol{p}^* $$ 为中心、持续时间为 $$ t $$ 的 Dirac delta 函数，我们得到截断的连续各向异性热核：

$$
h_t(\boldsymbol{p}, \boldsymbol{p}^* \mid \theta, \eta) = \sum_{k=1}^K e^{-t\lambda_k^{\eta, \theta}} \ \varphi_k^{\eta, \theta}(\boldsymbol{p}) \ \varphi_k^{\eta, \theta}(\boldsymbol{p}^*) \ m^*, \tag{3}
$$

其中 $$ \boldsymbol{p} $$ 是 $$ \mathcal{M} $$ 表面上的任意点，$$ \lambda_k^{\eta, \theta} $$ 是第 $$ k $$ 个特征值，其对应的特征函数 $$ \varphi_k^{\eta, \theta} $$ 在 $$ \boldsymbol{p} $$ 或 $$ \boldsymbol{p}^* $$ 处进行评估，$$ m^* $$ 是源处的质量。

由于物理热扩散过程守恒能量，随着热量扩散，源温度降低。因此，核的大小和形状会影响它们的整体温度。为了稳定优化并使滤波

![第 5 页](tmp/single_893159bdaee7576c5879caeed57c816c/5.png)

可预测，我们通过对扩散后的源温度进行归一化，保证核中心的最高温度为1.0：

$$
\bar{h}_t(\boldsymbol{p} \mid \boldsymbol{p}^\star, \theta, \eta) = \frac{h_t(\boldsymbol{p}, \boldsymbol{p}^\star \mid \theta, \eta)}{h_t(\boldsymbol{p}^\star, \boldsymbol{p}^\star \mid \theta, \eta)}.
$$

双调和距离加权。热核在整个物体表面具有全局支撑。虽然它们应随距热源的距离衰减，但由于我们采用对K个最重要频率的谱截断，核会产生吉布斯类现象[30,31]，在远端区域产生频谱振铃伪影（见补充材料）。为抑制这些不需要的伪影并强制局部支撑，我们引入双调和距离高斯加权，以衰减远端核的贡献。我们使用各向同性特征属性计算双调和距离[20]：$d_{\text{BH}}(\boldsymbol{p}, \boldsymbol{p}^\star)^2 = \sum_{k=1}^K \frac{(\varphi_k^{\text{iso}}(\boldsymbol{p}) - \varphi_k^{\text{iso}}(\boldsymbol{p}^\star))^2}{(\lambda_k^{\text{iso}})^2}$。随后，我们对归一化核进行加权：

$$
\tilde{h}_t(\boldsymbol{p} \mid \boldsymbol{p}^\star, \theta, \eta) = \bar{h}_t(\boldsymbol{p} \mid \boldsymbol{p}^\star, \theta, \eta) \cdot \exp\left(-\frac{d_{\text{BH}}(\boldsymbol{p}, \boldsymbol{p}^\star)^2}{2\sigma_{\text{BH}}^2}\right).
$$

核滤波。$\tilde{h}_t(\boldsymbol{p} \mid \boldsymbol{p}^\star, \theta, \eta)$ 生成一个高斯型测地核，值域为$[0,1]$，从源位置逐渐衰减。我们固定$t$并通过重缩放软阶跃滤波函数调节核尺度（$\tau$）和锐度（$\varsigma$）：

$$
\alpha(\boldsymbol{p} \mid \boldsymbol{p}^\star, \theta, \eta, \tau, \varsigma) = \frac{\xi(\tilde{h}_t(\boldsymbol{p} \mid \boldsymbol{p}^\star, \theta, \eta)) - \xi(0)}{\xi(1) - \xi(0)}, \quad \text{其中} \quad \xi(x) = \sigma(\varsigma \cdot (x - \tau)),
$$

其中$\sigma(\cdot)$是sigmoid函数。该滤波保留$[0,1]$值域，并提供对核外观的更多控制，现在可轻松缩放到点状维度，同时控制信号衰减速率。滤波后的核可保留其特征衰减外观，或转换为网格表面定义的清晰测地椭球。

颜色形成。对于给定核$\boldsymbol{\kappa}_s \in \mathcal{K}$，我们将其评估的空间足迹表示为$\alpha_s(\boldsymbol{p}) \equiv \alpha(\boldsymbol{p} \mid \boldsymbol{p}_s^\star, \theta_s, \eta_s, \tau_s, \varsigma_s)$。为确定表面点$\boldsymbol{p}$处的最终纹理颜色$C(\boldsymbol{p})$，需要聚合所有核的贡献。受[49]启发，我们选择对$\boldsymbol{p}$贡献最大的$K_c$个核。在我们的公式中，这可通过动态识别具有最高$\alpha_s(\boldsymbol{p})$的核来实现。局部纹理随后通过对这些选定核（$\mathcal{K}(\boldsymbol{p}) \subset \mathcal{K}$）的alpha加权单位分解计算，并作为残差添加到可优化的全局基色$\mathbf{c}_{\text{base}}$：

$$
C(\boldsymbol{p}) = \mathbf{c}_{\text{base}} + \frac{\sum_{s \in \mathcal{K}(\boldsymbol{p})} \alpha_s(\boldsymbol{p}) \, \mathbf{c}_s}{\sum_{s \in \mathcal{K}(\boldsymbol{p})} \alpha_s(\boldsymbol{p})},
$$

该颜色形成范式保证了局部重叠核之间的可控混合，而空白区域则无缝默认为底层基色。
4.1 谱基的参数插值

ALBO 特征属性的插值。评估方程 (3) 中的连续各向异性热核需要与核的特定形状参数 $$ (\eta, \theta) $$ 相关的谱属性。由于在优化流程中为每个核构建并特征分解一个唯一的 ALBO 在计算上是不可行的，我们设计了一种简化且易于微分的替代方案。因此，我们将角度和各向异性的参数空间离散化为一个半极坐标式的网格，其中各向异性采用对数缩放，以使空间在感知上更加均匀，并防止高各向异性占据主导地位。我们将每个网格节点 $$ (i, j) $$（位于离散角度 $$ \theta_i $$ 和对数缩放的各向异性 $$ s_j = \ln(1 + \eta_j) $$ 处）与其预计算的 $$ \mathbf{\Phi}^\ddagger_{i,j} $$ 和 $$ \mathbf{\Lambda}^\ddagger_{i,j} $$ 关联起来。为了评估具有参数 $$ (\eta, \theta) $$ 的连续核查询，我们首先将查询的各向异性投影到相同的对数空间中，$$ s = \ln(1+\eta) $$。然后我们执行二分搜索以确定边界网格单元索引 $$ i $$ 和 $$ j $$，使得 $$ \theta_i \le \theta < \theta_{i+1} $$ 且 $$ s_j \le s < s_{j+1} $$。在此单元内，我们计算归一化局部坐标：$$ \Delta\theta = \frac{\theta - \theta_i}{\theta_{i+1} - \theta_i} $$ 和 $$ \Delta s = \frac{s - s_j}{s_{j+1} - s_j} $$。然后通过四个单元角点 $$ \mathcal{C} = \{(a, b) \mid a \in \{j, j+1\}, b \in \{i, i+1\}\} $$ 上的双线性插值有效地近似查询的特征属性：

$$ 
\mathbf{\Phi}^{\mathbf{q}}_{\eta, \theta} = \sum_{(a, b) \in \mathcal{C}} w_{a, b} \mathbf{\Phi}^\ddagger_{a, b} \quad \text{and} \quad \mathbf{\Lambda}^{\mathbf{q}}_{\eta, \theta} = \sum_{(a, b) \in \mathcal{C}} w_{a, b} \mathbf{\Lambda}^\ddagger_{a, b}, \quad (8) 
$$

其中插值权重 $$ w_{a,b} $$ 源自分数距离 $$ \Delta\theta $$ 和 $$ \Delta s $$（即 $$ w_{i,j} = (1 - \Delta\theta)(1 - \Delta s) $$，$$ w_{i+1,j} = \Delta\theta(1 - \Delta s) $$ 等）。

预计算 ALBO 特征属性的对齐。在对预计算的 ALBO 进行特征分解时，我们观察到了形状匹配问题中广泛观察到的相同的排列、旋转和符号歧义（例如，[27]）。由于这种行为会阻碍方程 (8) 中双线性插值的正确性，我们在网格构建期间引入了特征属性对齐过程，从而确保所有 $$ \mathbf{\Phi}^\ddagger_{i,j} $$ 和 $$ \mathbf{\Lambda}^\ddagger_{i,j} $$ 与各向同性 LBO 的 $$ \mathbf{\Phi} $$ 和 $$ \mathbf{\Lambda} $$ 一致。首先，我们通过最大化两组特征向量 $$ \mathbf{\Phi}_{\text{ref}} $$ 和 $$ \mathbf{\Phi}^\ddagger_{i,j} $$ 之间的质量加权互相关来解决排列交换问题。我们计算相关矩阵 $$ \mathbf{C} = \mathbf{\Phi}_{\text{ref}}^\top \mathbf{M} \mathbf{\Phi}^\ddagger_{i,j} $$ 并应用匈牙利算法找到最优分配矩阵 $$ \mathbf{\Pi} $$，从而得到置换后的特征向量 $$ \mathbf{\Phi}_{\text{perm}} = \mathbf{\Phi}^\ddagger_{i,j} \mathbf{\Pi} $$。接下来，我们通过 $$ \mathbf{\Phi}_{\text{perm}}^\top \mathbf{M} \mathbf{\Phi}_{\text{ref}} $$ 的奇异值分解 (SVD) 求解正交普鲁克问题，最小化质量加权 Frobenius 距离 $$ \|\mathbf{\Phi}_{\text{ref}} - \mathbf{\Phi}_{\text{perm}} \mathbf{R}\|_\mathbf{M}^2 $$。由此产生的旋转确定了 $$ \mathbf{\Phi}_{\text{rot}} = \mathbf{\Phi}_{\text{perm}} \mathbf{R} $$。最后，我们通过构建一个对角矩阵 $$ \mathbf{S} $$ 来校正符号翻转，其中 $$ \mathbf{S}_{k,k} = \text{sgn}((\mathbf{\Phi}_{\text{ref}}^\top \mathbf{M} \mathbf{\Phi}_{\text{rot}})_{k,k}) $$。因此，存储在每个网格节点处的完全对齐的谱基获得为 $$ \mathbf{\Phi}_{\text{aligned}} = \mathbf{\Phi}^\ddagger_{i,j} \mathbf{\Pi} \mathbf{R} \mathbf{S} $$。特征值仅根据 $$ \mathbf{\Pi} $$ 重新排序。初始参考特征向量是 LBO 的特征向量，然后我们使用最后一组

![第 7 页](tmp/single_893159bdaee7576c5879caeed57c816c/7.png)

在网格构建过程中对齐。此后，为简化符号表示，我们默认所有$\Phi_{i,j}^\dagger$和$\Lambda_{i,j}^\dagger$已隐式对齐。

4.2 谱性质与质量的空域评估

连续空域评估。所有预计算的谱性质严格定义在流形$\mathcal{M}$的顶点上。将核函数的放置与评估仅限于这些位置会将模型束缚于表面离散化。为突破此限制并在任意连续位置$\boldsymbol{p}$评估基于顶点的谱量$\mathbf{A}$，我们利用谱基的固有平滑性。受文献[12]启发，通过重心插值$b^p(\mathbf{A})$实现该连续空域评估。

双调和KNN搜索。使用公式(3)对所有S核函数和P目标点进行热方程评估会产生$\mathcal{O}(P \times S)$的不可行复杂度。由于双调和距离加权（公式(5)）迫使核函数在局部半径外衰减至零，可安全剪枝远距离源点。定义顶点处的谱嵌入为$\mathbf{Z} = \Phi^{\text{iso}}(\Lambda^{\text{iso}})^{-1}$。在此空间中，欧氏距离对应流形上的双调和距离$d_{\text{BH}}$。采用连续评估策略，计算核源点与目标点的嵌入$\mathbf{z}(\boldsymbol{p}) \approx b^p(\mathbf{Z})$和$\mathbf{z}(\boldsymbol{p}^\star) \approx b^{p^\star}(\mathbf{Z})$。随后在该嵌入空间构建K近邻（KNN）图，动态限制每个点$\boldsymbol{p}$仅评估其$K_s$个最近邻源点$\mathcal{N}(\boldsymbol{p})$。

局部各向异性评估。确定局部源点子集$\mathcal{N}(\boldsymbol{p})$后，需评估这些源点所需的连续各向异性特征函数。不同于全局计算所有核函数的基，我们检索仅对应$K_s$个邻近源点特定形状参数$(\theta_s, \eta_s)$的离散特征向量$\Phi_{\eta,\theta}^q$。应用连续评估原理，在源点位置$\boldsymbol{p}^\star$和目标位置$\boldsymbol{p}$近似这些源特定特征函数：$[\varphi_1^{\eta,\theta}(\boldsymbol{p}^\star), \dots, \varphi_K^{\eta,\theta}(\boldsymbol{p}^\star)] \approx b^{p^\star}(\Phi_{\eta,\theta}^q)$和$[\varphi_1^{\eta,\theta}(\boldsymbol{p}), \dots, \varphi_K^{\eta,\theta}(\boldsymbol{p})] \approx b^p(\Phi_{\eta,\theta}^q)$。这种双重评估允许在$\boldsymbol{p}^\star$与$\boldsymbol{p}$间构建热核（公式(3)），同时将计算复杂度严格限制在$\mathcal{O}(P \times K_s)$。

流形$\mathcal{M}$上任意点的质量定义。最终，评估连续热核（公式(3)）需要连续源点$\boldsymbol{p}^\star$处的质量$m^\star$。由于离散质量矩阵$\mathbf{M}$与顶点面积成比例，即局部采样密度，无法像特征向量那样直接使用重心插值。因此提出核密度估计（KDE）方法，基于周围点密度准确估计各点质量（见补充材料）。虽然该方法近似真实质量，但经验表明对所有源点赋予统一单位质量（$m^\star = 1$）显著更快且不降低性能。由于逐点评估仅依赖源质量作为线性标量（公式(3)），优化过程中其他核属性可学习补偿缺失的局部积分权重。
4.3 优化与密度控制

测地线位置优化。标准欧几里得优化会直接将梯度更新应用于 3D 坐标，这不可避免地会将热核源拉离网格表面。由于源 $$p^*$$ 是在 $$\mathcal{M}$$ 上内蕴定义的，我们必须将运动限制在表面上。因此，我们构建了一个黎曼梯度下降优化器：每个梯度 $$\nabla_{p^*}\mathcal{L}$$ 被投影到局部切平面 $$\mathcal{T}_{p^*}\mathcal{M}$$ 并缩放以定义初始速度 $$\mathbf{v}^*$$。然后我们计算指数映射 $$\text{Exp}(p^*, \mathbf{v}^*)$$，追踪由该向量定义的测地线以确定更新后的核位置。为了利用动量加速收敛，我们将累积的动量平行移动到新核位置。指数映射和平行移动均通过最直测地线算法 [32] 计算，利用了 [38] 引入的对优化友好且 CUDA 并行化的实现。

核剪枝策略。像大多数高斯泼溅（Gaussian Splatting）技术一样，我们在整个优化过程中自适应地控制热核的密度。为了减少计算开销和存储成本，我们的目标是删除对最终纹理贡献很小的核。在一个累积窗口 $$\Delta_{\text{prune}}$$ 期间，我们记录每个核 $$\kappa_s \in \mathcal{K}$$ 的两个指标：命中计数 $$\nu(\kappa_s)$$ 表示其在局部单位分解 $$\mathcal{K}(p)$$（公式 (7)）中被选中的频率，以及能量分数 $$\omega(\kappa_s)$$ 聚合其过滤后的权重 $$\alpha_s(p)$$。定期地，我们剪枝那些要么很少被命中（$$\nu(\kappa_s) < \gamma_{\text{hit}} \cdot \max(\{\nu(\kappa_i)\}_{i=1}^S)$$）要么能量较弱（$$\omega(\kappa_s) < \gamma_{\text{energy}}$$）的核。

核致密化策略。为了捕捉缺失的细节，我们基于局部重建误差动态地致密化核。在一个累积窗口 $$\Delta_{\text{error}}$$ 内，我们计算空间误差 $$e(p) = \|\mathbf{c}(p) - \mathbf{c}_{\text{GT}}(p)\|_1$$ 并将其按归一化单位分解加权分配给贡献核，从而得到每个核的分数 $$E(\kappa_s)$$。超过误差阈值 $$\gamma_{\text{error}}$$ 的核根据其空间范围 $$(1 - \tau_s)$$ 进行致密化。小核 $$(1 - \tau_s \le \gamma_{\text{size}})$$ 被原地克隆，允许优化器自然地分离它们。大核 $$(1 - \tau_s > \gamma_{\text{size}})$$ 被分裂成两个较小的核。关键在于，为了在分裂过程中尊重流形约束，我们使用 [38] 提供的指数映射沿其主轴位移新核。最大增长率 $$\gamma_{\text{growth}}$$ 限制每步的添加量以确保稳定性。

4.4 渲染与应用

基于物理的渲染。由于我们的 HKTex 公式在 $$\mathcal{M}$$ 上连续求值，它可以无缝集成到现代基于物理的渲染（PBR）管线中。我们还在 Mitsuba3 渲染器 [16] 中将 HKTex 实现为可微纹理。在光线追踪期间，当光线与网格在任意连续点 $$p$$ 相交时，渲染器查询相应的局部核子集 $$\mathcal{K}(p)$$ 并按照公式 (7) 计算 $$C(p)$$。通过将我们基于 PyTorch 的评估封装在 Dr.Jit [15] 中，我们允许梯度
直接从渲染图像像素反向传播到我们的内在核参数。

**优化现有 UV 纹理。** 当存在真实值 UV 纹理时，我们可以直接优化我们的核。在每一步优化中，我们在 $$ \mathcal{M} $$ 的面上直接均匀采样一组密集的点 $$ \boldsymbol{p} $$，并用 $$ C(\boldsymbol{p}) $$ 评估它们的颜色。然后通过最小化相对于 $$ C_{\text{GT}}(\boldsymbol{p}) $$（即从同一位置的 UV 纹理查询到的颜色）的光度误差来更新我们的 HKTEx 模型：$$ \mathcal{L} = \|C(\boldsymbol{p}) - C_{\text{GT}}(\boldsymbol{p})\|_2^2 $$。

**优化多视图图像。** 我们的 HKTEx 模型也可以通过可微分 PBR 渲染器从多视图图像进行优化。在这种情况下，我们假设目标网格与一组相机位姿一起给出。在每个训练步骤中，我们采样 V 个相机视图，识别所有击中网格的光线，并选择一个随机子集以提供围绕物体的均匀覆盖。最小化光线交点处相对于真实值像素的渲染误差驱动了 HKTex 优化。

## 5 实验

### 5.1 数据

我们在 Objaverse [11] 的一个精选子集上进行实验。由于完整数据集包含抓取的在线资产，网格质量无法保证。因此，我们筛选具有 LVIS 类别标签、有效纹理以及至少一个社区点赞和视图的对象，同时排除点云和动画模型。为了保持合理的计算开销，我们将最大顶点数限制为 $$ N = 60,000 $$。为了确保与我们的 HKTEx 表示兼容，我们还定义了严格的几何过滤：我们排除具有多个不连通分量、非流形几何体以及复杂内部结构的网格（即从定义在物体边界框上的规则网格投射的任何光线呈现超过 20 个交点的模型）。最后，由于我们的方法严重依赖于 LBO 的特征函数，我们拒绝那些特征分解失败的模型。过滤后，我们保留了 $$ \sim 3,000 $$ 个高质量模型。请注意，虽然拥有可特征分解的 LBO 至关重要，但我们的过滤标准可以放宽，代价是整体数据集质量降低或单独处理非流形和复杂网格的部分。当拟合多视图图像时，网格以其反照率作为唯一的纹理贴图进行渲染。

### 5.2 实现细节

我们的 HKTEx 流程是使用 PyTorch [29] 构建的，利用 FAISS [17] 的 GPU 版本进行双调和距离 KNN 搜索，并使用带有 Dr.Jit [15] 的 Mitsuba3 [16] 进行可微分光迹追踪。我们没有构建专门的 CUDA 内核，而是使用 PyTorch 的 Triton 代码生成 [3]。对于所有网格，我们预先计算各向同性 LBO 的 64 维特征分解以用于计算
双调和距离，以及针对网格中所有ALBOs的256维特征分解，涉及7个角度和7个各向异性的叉积。网络参数按以下段落初始化，我们使用投影梯度下降（PGD）算法确保参数范围正确。除核位置外的所有参数，我们采用带有阶梯学习率调度器的Adam优化器来拟合UV纹理和多视角图像。为优化核位置，我们使用文献[38]中的库在GPU上运行测地黎曼-SGD算法。我们在两个设置中对5个排除在评估数据集外的网格进行简短的贝叶斯超参数调优，使用Optuna [1]与树结构帕森估计器[4]，并在补充材料中共享精确超参数。我们采用重要性采样策略（详见补充材料）来拟合UV纹理。为渲染输出图像并执行可微分光线追踪进行逆向渲染，我们使用Mitsuba3 [16]与Dr.Jit [15]。采用路径回放反向传播（PRB）积分器，训练时射线深度为3，并通过光线追踪过程微分以优化纹理。推理时仅构建KNN数据库并预计算自参考热核，显著减少计算量。我们使用调优后的超参数设置两种配置的剪枝和致密化超参数。

初始化。我们采用基于网格面积的均匀采样策略，首先在网格表面过采样大量点，然后使用最远点采样获取核位置和面索引。核颜色从-1到1的均匀分布中采样，减去均值后设置为全局基色。角度和各向异性分别从0到π和1到100的均匀分布中采样。对于锐度和阈值初始化，我们分别从β分布$\mathcal{B}(2,1)$和$\mathcal{B}(10/7,1)$采样。随后，锐度缩放并偏移至10-50范围，阈值调整至0.9-1.0（多视角渲染案例为0.7-1.0）。参数随后使用PGD进行优化，详见前文及补充材料。

5.3 评估指标
为定量评估纹理与真实值的对比，我们对每个对象渲染五个不同视角并测量图像指标。具体计算均方误差（MSE）和峰值信噪比（PSNR）。为评估结构细节和对比度的保持程度，我们使用结构相似性指数（SSIM）及其多尺度变体（MS-SSIM）。此外，我们报告学习感知图像块相似度（LPIPS）以衡量感知质量。最后，为验证内存效率的核心主张，我们以千字节（KB）为单位报告总存储占用，严格计入所有本征参数、网络权重及任何拓扑开销（如UV坐标）。存储格式与基线方法相同，未使用方法特定的量化。所有方法均应用相同的ZIP压缩，为比较本征信息密度提供公平基线。
12 Foti et al.

Original GT UV Textures | LR GT UV Textures | MLP Positional Encoding | InstantNGP | Intrinsic Neural Fields | ImageGS | HKTex (ours)

图 2：与最先进（SotA）方法的定性比较。除了 GT UV 纹理外，所有其他方法都有相似或更大的存储大小。为了视觉清晰，我们仅在恒定环境光照下渲染反照率纹理。

5.4 拟合 UV 纹理

我们首先展示我们的方法以显著降低的存储成本表示现有 UV 纹理的能力。我们将我们的 HKTex 针对从我们精选的数据集中任意选取的 313 个网格的真实反照率 UV 纹理进行优化。然后，我们在严格匹配的内存预算下，与领先的先进内在纹理表示和基于图像的基线进行定量和定性比较。

对于神经基线，我们实现了三个多层感知机（MLP），以射线交点的不同位置编码作为输入。具体来说，我们评估了一个具有传统傅里叶位置编码 [23] 的标准 MLP，一个利用哈希网格编码的 InstantNGP [24] 的改编版本，以及内在神经场 [19]，后者对 LBO 的特征向量进行重心插值。为了确保公平比较，我们对网络宽度、深度和哈希网格大小进行网格搜索，以完全匹配我们优化的 HKTex 的存储成本。

此外，我们与两个显式 2D 基线进行比较：下采样的真实图像和 ImageGS [49]（一种 2D 高斯泼溅方法）。至关重要的是，这两种基于图像的方法从根本上都依赖于 UV 映射。甚至

![第 12 页](tmp/single_893159bdaee7576c5879caeed57c816c/12.png)

当提供 UV 贴图时，逐顶点 UV 坐标的存储会产生固定的内存开销，这必须计入总纹理预算中。因此，对于具有高顶点数的网格，这种开销占据了预算的主导地位，严重限制了可用于实际纹理或 2D 高斯元的内存。如表 1 所示，对于这些基线而言，通常无法实现精确的存储匹配；即使将下采样纹理限制为最小分辨率 $4 \times 4$ 像素，并将 ImageGS 限制为仅 10 个高斯元，对于密集采样的网格，其性能惩罚也是灾难性的（图 2）。

最后，我们将直接与在原生网格分辨率下从 GT UV 纹理采样的顶点颜色（GT VTex）以及通过迭代细分网格以匹配我们存储预算的方法（HR GT VTex）进行比较。

如表 1 所示，HKTex 在保持所有指标上具有竞争力的重建质量的同时，实现了除 GT VTex 之外最低的存储占用。虽然 LR UV 纹理由于直接存储图像样本而实现了最高的 PSNR，但我们的表示产生了第二好的感知质量（LPIPS）以及最好的 SSIM、MS-SSIM 和 MSE，表明尽管使用了更少的参数，但结构细节的保留得到了改善。神经基线和 HR GT VTex 需要显著更大的存储预算才能实现几乎相当的精度，这凸显了我们测地热核表示的效率。

图 2 中的定性比较进一步突出了不同表示的行为。神经场基线倾向于过度平滑精细的纹理细节，特别是在具有锐利颜色过渡或高频模式的区域（例如，青蛙和树莓示例）。ImageGS 偶尔会产生明显的伪影，这是由于在存储限制下可用的 splat 数量有限，有时无法捕捉到正确的颜色分布。顶点颜色，即使经过细分（图 3），也无法正确表示纹理，因为顶点位置是由几何形状而非纹理内容决定的。相比之下，HKTex 在保持相同或更小的存储预算的同时，保留了更清晰的纹理边界和物体间更忠实的颜色分布。

**图 3：** 与顶点颜色的比较。

**拟合图像。** 为了测试 HKTex 在粗糙网格上表示高分辨率纹理的能力，我们将图像拟合到由两个三角形组成的矩形上。在这种 UV 纹理拟合的特殊情况下，我们发现在过滤之前放大加权和归一化核 $(\hat{h}_t)^a$ 是有益的（其中 $a=30$）。尽管如此，$a=1$ 对于大多数网格来说已经足够了。

**图 4：** 两个三角形上的图像。

**5.5 拟合多视图图像**

为了展示我们方法优化多视图图像的能力，我们创建了一个合成训练场景，其中不使用 UV 纹理，而是使用真实 UV 纹理的渲染图像，遵循 UV 中相同的数据设置

![第 13 页](tmp/single_893159bdaee7576c5879caeed57c816c/13.png)

14 Foti et al.

表 1：我们在 Objaverse [11] 策划子集中的 313 个网格上进行的 UV 纹理拟合的定量比较。HR 和 LR 分别表示高分辨率和低分辨率。

| | PSNR ($\uparrow$) | LPIPS ($\times 10^{-2}, \downarrow$) | SSIM ($\times 10^{-2}, \uparrow$) | MS-SSIM ($\times 10^{-2}, \uparrow$) | MSE ($\times 10^{-3}, \downarrow$) | Render (ms, $\downarrow$) | Storage (KB, $\downarrow$) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| LR GT UVTextures | $\mathbf{48.4}_{\pm 15.2}$ | $2.3_{\pm 4.2}$ | $98.2_{\pm 3.4}$ | $97.7_{\pm 5.6}$ | $0.89_{\pm 3.61}$ | $33.7_{\pm 3.2}$ | $115.8_{\pm 103.2}$ |
| GT VTex (N $\approx$ 7.7k) | $37.9_{\pm 8.5}$ | $4.0_{\pm 5.0}$ | $96.8_{\pm 4.2}$ | $96.1_{\pm 6.6}$ | $0.94_{\pm 2.33}$ | $\mathbf{25.4}_{\pm 3.1}$ | $\mathbf{17.3}_{\pm 32.6}$ |
| HR GT VTex (N $\approx$ 210k) | $45.3_{\pm 5.7}$ | $\mathbf{1.1}_{\pm 1.7}$ | $98.8_{\pm 2.1}$ | $99.4_{\pm 1.4}$ | $\mathbf{0.08}_{\pm 0.20}$ | $50.3_{\pm 49.4}$ | $179.6_{\pm 89.5}$ |
| MLP Pos. Enc. | $42.9_{\pm 6.6}$ | $2.3_{\pm 2.5}$ | $98.2_{\pm 2.2}$ | $99.0_{\pm 1.3}$ | $0.15_{\pm 0.30}$ | $90.9_{\pm 35.2}$ | $551.3_{\pm 759.6}$ |
| InstantNGP [24] | $41.3_{\pm 7.4}$ | $3.2_{\pm 3.5}$ | $97.5_{\pm 3.1}$ | $98.2_{\pm 2.7}$ | $0.26_{\pm 0.52}$ | $90.8_{\pm 35.3}$ | $473.2_{\pm 714.9}$ |
| INFs [19] | $42.5_{\pm 6.8}$ | $2.7_{\pm 3.4}$ | $98.0_{\pm 2.7}$ | $98.8_{\pm 1.9}$ | $0.19_{\pm 0.44}$ | $91.4_{\pm 35.3}$ | $928.2_{\pm 1,439.8}$ |
| ImageGS [49] | $44.6_{\pm 14.6}$ | $2.9_{\pm 5.4}$ | $97.3_{\pm 5.6}$ | $95.9_{\pm 9.6}$ | $3.65_{\pm 11.95}$ | NA | $115.4_{\pm 93.9}$ |
| HKTex (S $\approx$ 4.8k) | $44.8_{\pm 5.8}$ | $\underline{1.3}_{\pm 1.6}$ | $\mathbf{98.9}_{\pm 1.5}$ | $\mathbf{99.5}_{\pm 0.7}$ | $\mathbf{0.08}_{\pm 0.20}$ | $784.7_{\pm 474.3}$ | $\underline{96.6}_{\pm 12.2}$ |

纹理拟合，但使用较小的数据子集。为了生成训练数据，我们在训练期间从随机视角采样射线，并在每个 epoch 选择一个随机子集来形成小批量（minibatches）。

类似于 UV 拟合设置，我们实现了两个 MLP 架构，基于不同的位置编码进行条件化，对应于空间变化的反照率图（albedo map）。为了公平比较，这些表示也使用自动微分以及 Mitsuba3 和 PyTorch 之间的互操作性进行训练。第一个基线利用传统的位置编码。第二个基线是 NVDiffRec [25] 的改编，网格和环境图固定为真值（ground-truth）网格和环境图，使用哈希编码，且仅输出反照率图。类似于 uv 拟合设置，我们也对宽度、深度和哈希网格大小进行网格搜索，以尝试匹配我们模型的存储成本。

如表 2 所示，HKTex 在所有评估指标上一致地优于高分辨率顶点颜色和神经基线，同时需要显著更少的存储。特别是，LPIPS 和 SSIM 的改进表明我们的内在公式在逆渲染过程中更有效地保留了感知细节和结构一致性。这些结果表明，将表示直接约束到表面流形（surface manifold）为从稀疏多视图观测中恢复外观提供了强大的归纳偏置。

虽然如表 1 所示，由于核聚合和测地邻域查询（geodesic neighbourhood queries），我们的表示引入了额外的评估开销，但在多视图设置中，这种成本被部分摊销了，因为在表 2 中可以看到，光线追踪（具有增加的深度）主导了渲染时间。

6 结论

在这项工作中，我们提出了 HKTex，一种内在纹理表示，它用直接定义在表面流形上的各向异性热核（anisotropic heat kernels）替换了传统的 UV 映射纹理。通过完全在网格的内在几何中操作，我们的公式消除了与 UV 参数化相关的接缝、失真和冗余存储。通过使用测地热

![第 14 页](tmp/single_893159bdaee7576c5879caeed57c816c/14.png)

Heat Kernel Textures 15

表 2：来自 Objaverse [11] 的 162 个网格的精选子集上的多视图渲染定量评估。NVDiffRec [25] 是我们的改编版本，其中真实网格（ground truth mesh）和环境贴图是固定的而非学习得到的，并且仅像我们的方法一样输出反照率（albedo）贴图。

方法 | PSNR (↑) | LPIPS (×10⁻², ↓) | SSIM (×10², ↑) | MS-SSIM (×10⁻², ↑) | MSE (×10⁻³, ↓) | 渲染 (s, ↓) | 存储 (KB, ↓)
---|---|---|---|---|---|---|---
MLP Pos. Enc. | 35.75 ± 7.76 | 3.7 ± 3.8 | 97.2 ± 3.1 | 96.8 ± 4.9 | 1.55 ± 4.83 | 0.68 ± 0.12 | 602.71 ± 755.89
NVDiffRec* [25] | 36.40 ± 6.83 | 3.4 ± 3.6 | 97.4 ± 2.9 | 97.4 ± 3.5 | 0.79 ± 2.04 | 0.66 ± 0.11 | 516.12 ± 722.66
HR VTex | 37.16 ± 5.69 | 3.1 ± 3.4 | 97.5 ± 2.8 | 97.9 ± 2.6 | 0.44 ± 0.85 | 0.58 ± 0.08 | 81.06 ± 131.16
HKTex ($$ S \approx 3.8k $$) | 37.61 ± 4.71 | 2.1 ± 1.9 | 98.3 ± 1.6 | 98.6 ± 1.4 | 0.30 ± 0.53 | 1.2 ± 0.3 | 78.73 ± 23.12

核与黎曼优化，我们的方法在保持与底层表面严格对齐的同时，能够在可微分基于物理的渲染器内实现连续的外观评估。这种表述既实现了现有 UV 纹理的压缩，又实现了从多视图图像直接重建表面外观，同时完全内在于网格且独立于任何全局表面参数化。HKTex 的数学连续性本质上充当了表面上的扩散曲线，为流形矢量图形奠定了基础步骤。

局限性与未来工作。未来的工作可能会将表示从反照率扩展到额外的外观属性，例如空间变化的 BSDF 参数。虽然我们目前的评估集中在反照率纹理上，但每个核都可以用额外的材料参数（例如，粗糙度或镜面反射分量）进行增强，并通过 Mitsuba3 的可微分渲染进行优化。

致谢
T. Birdal 得到了 UKRI Future Leaders Fellowship [grant number MR/Y018818/1] 的支持。S. Foti, S. Zafeiriou, 和 T. Birdal 得到了 EPSRC Project GNOMON (EP/X011364/1) 的支持。S. Foti 和 S. Zafeiriou 还得到了 Turing AI Fellowship MAGAL (EP/Z534699/1) 的支持。

参考文献
1. Akiba, T., Sano, S., Yanase, T., Ohta, T., Koyama, M.: Optuna: A next-generation hyperparameter optimization framework (2019), https://arxiv.org/abs/1907.10902
2. Andreux, M., Rodola, E., Aubry, M., Cremers, D.: Anisotropic laplace-beltrami operators for shape analysis. In: European conference on computer vision. pp. 299-312. Springer (2014)
3. Ansel, J., Yang, E., He, H., Gimelshein, N., Jain, A., Voznesensky, M., Bao, B., Bell, P., Berard, D., Burovski, E., Chauhan, G., Chourdia, A., Constable, W., Desmaison, A., DeVito, Z., Ellison, E., Feng, W., Gong, J., Gschwind, M., Hirsh, B.,

![第 15 页](tmp/single_893159bdaee7576c5879caeed57c816c/15.png)

16 Foti 等人

Huang, S., Kalamarkar, K., Kirsch, L., Lazos, M., Lezcano, M., Liang, Y., Liang, J., Lu, Y., Luk, C.K., Maher, B., Pan, Y., Puhrsch, C., Reso, M., Saroufim, M., Siraichi, M.Y., Suk, H., Zhang, S., Suo, M., Tillet, P., Zhao, X., Wang, E., Zhou, K., Zou, R., Wang, X., Mathews, A., Wen, W., Chanan, G., Wu, P., Chintala, S.: Pytorch 2：通过动态 Python 字节码转换和图编译实现更快的机器学习。载于：第 29 届 ACM 编程语言与操作系统架构支持国际会议论文集，第 2 卷。第 929–947 页。ASPLOS '24，计算机协会，美国纽约州纽约市 (2024)。https://doi.org/10.1145/3620665.3640366, https://doi.org/10.1145/3620665.3640366

4. Bergstra, J., Bardenet, R., Bengio, Y., Kégl, B.: 超参数优化算法。载于：Shawe-Taylor, J., Zemel, R., Bartlett, P., Pereira, F., Weinberger, K. (编) 神经信息处理系统进展。第 24 卷。Curran Associates, Inc. (2011), https://proceedings.neurips.cc/paper_files/paper/2011/file/86e8f7ab32cfd12577bc2619bc635690-Paper.pdf

5. Burley, B.: 迪士尼的物理着色。SIGGRAPH 2012 课程笔记 (2012), https://www.disneyanimation.com/publications/physically-based-shading-at-disney/

6. Burley, B.: 将迪士尼 BRDF 扩展为集成次表面散射的 BSDF。SIGGRAPH 2015 课程笔记 (2015), https://blog.selfshadow.com/publications/s2015-shading-course/#course_content

7. Chaitanya, C.R.A., Kaplanyan, A.S., Schied, C., Salvi, M., Lefohn, A., Nowrouzezahrai, D., Aila, T.: 使用循环去噪自编码器交互式重建蒙特卡洛图像序列。ACM 图形汇刊 36(4) (2017 年 7 月)。https://doi.org/10.1145/3072959.3073601, https://doi.org/10.1145/3072959.3073601

8. Choi, J., Lee, Y., Lee, H., Kwon, H., Manocha, D.: Meshgs：用于高质量渲染的自适应网格对齐高斯泼溅。载于：亚洲计算机视觉会议论文集。第 3310–3326 页 (2024)

9. Chung, M.K., Qiu, A., Seo, S., Vorperian, H.K.: 流形上扩散、核平滑和小波的统一热核回归及其在 CT 图像下颌骨生长建模中的应用。医学图像分析 22(1), 63–76 (2015)

10. Condor, J., Speierer, S., Bode, L., Bozic, A., Green, S., Didyk, P., Jarabo, A.: 不要泼溅你的高斯：用于建模和渲染散射及发光介质的体积光线追踪图元。ACM 图形汇刊 44(1), 1–17 (2025)

11. Deitke, M., Schwenk, D., Salvador, J., Weihs, L., Michel, O., VanderBilt, E., Schmidt, L., Ehsani, K., Kembhavi, A., Farhadi, A.: Objaverse：一个注释 3D 对象的宇宙。载于：IEEE/CVF 计算机视觉与模式识别会议论文集。第 13142–13153 页 (2023)

12. Foti, S., Zafeiriou, S., Birdal, T.: 具有去噪和测地热扩散的无 UV 纹理生成。神经信息处理系统进展 37, 128053–128081 (2024)

13. Gao, J., Gu, C., Lin, Y., Li, Z., Zhu, H., Cao, X., Zhang, L., Yao, Y.: 可重光照 3D 高斯：通过 BRDF 分解和光线追踪实现逼真的点云重光照。载于：欧洲计算机视觉会议。第 73–89 页。Springer (2024)

14. Guédon, A., Lepetit, V.: Sugar：用于高效 3D 网格重建和高质量网格渲染的表面对齐高斯泼溅。载于：IEEE/CVF 计算机视觉与模式识别会议论文集。第 5354–5363 页 (2024)
15. Jakob, W., Speierer, S., Roussel, N., Vicini, D.: Dr.jit：一种用于可微分渲染的即时编译器。图形学汇刊（SIGGRAPH会议论文集）41(4)（2022年7月）。https://doi.org/10.1145/3528223.3530099
16. Jakob, W., Speierer, S., Roussel, N., Nimier-David, M., Vicini, D., Zeltner, T., Nicolet, B., Crespo, M., Leroy, V., Zhang, Z.: Mitsuba 3渲染器（2022），https://mitsuba-renderer.org
17. Johnson, J., Douze, M., Jégou, H.: 基于GPU的十亿级相似度搜索。IEEE大数据汇刊7(3), 535–547（2019）
18. Kerbl, B., Kopanas, G., Leimkühler, T., Drettakis, G.: 用于实时辐射场渲染的3D高斯溅射。ACM图形学汇刊42(4), 139–1（2023）
19. Koestler, L., Grittner, D., Moeller, M., Cremers, D., Lähner, Z.: 流形上的本征神经场：学习函数。载于：欧洲计算机视觉会议。第622–639页。施普林格（2022）
20. Lipman, Y., Rustamov, R.M., Funkhouser, T.A.: 双调和距离。ACM图形学汇刊（TOG）29(3), 1–11（2010）
21. Mahajan, M., Hofherr, F., Cremers, D.: Meshfeat：网格上神经场的多分辨率特征。载于：欧洲计算机视觉会议。第268–285页。施普林格（2024）
22. Melzi, S., Rodolà, E., Castellani, U., Bronstein, M.M.: 各向异性窗口傅里叶变换的形状分析。载于：2016年第四届三维视觉国际会议（3DV）。第470–478页。IEEE（2016）
23. Mildenhall, B., Srinivasan, P.P., Tancik, M., Barron, J.T., Ramamoorthi, R., Ng, R.: Nerf：将场景表示为用于视图合成的神经辐射场。ACM通讯65(1), 99–106（2021）
24. Müller, T., Evans, A., Schied, C., Keller, A.: 具有多分辨率哈希编码的即时神经图形基元。ACM图形学汇刊（TOG）41(4), 1–15（2022）
25. Munkberg, J., Hasselgren, J., Shen, T., Gao, J., Chen, W., Evans, A., Müller, T., Fidler, S.: 从图像中提取三角形三维模型、材质和光照。载于：IEEE/CVF计算机视觉与模式识别会议论文集（CVPR）。第8280–8290页（2022年6月）
26. Oechsle, M., Mescheder, L., Niemeyer, M., Strauss, T., Geiger, A.: 纹理场：在函数空间中学习纹理表示。载于：IEEE/CVF国际计算机视觉会议论文集。第4531–4540页（2019）
27. Ovsjanikov, M., Ben-Chen, M., Solomon, J., Butscher, A., Guibas, L.: 函数映射：形状间映射的灵活表示。ACM图形学汇刊（ToG）31(4), 1–11（2012）
28. Parker, S.G., Bigler, J., Dietrich, A., Friedrich, H., Hoberock, J., Luebke, D., McAllister, D., McGuire, M., Morley, K., Robison, A., Stich, M.: Optix：通用光线追踪引擎。ACM图形学汇刊29(4)（2010年7月）。https://doi.org/10.1145/1778765.1778803, https://doi.org/10.1145/1778765.1778803
29. Paszke, A., Gross, S., Massa, F., Lerer, A., Bradbury, J., Chanan, G., Killeen, T., Lin, Z., Gimelshein, N., Antiga, L., Desmaison, A., Köpf, A., Yang, E., DeVito, Z., Raison, M., Tejani, A., Chilamkurthy, S., Steiner, B., Fang, L., Bai, J., Chintala, S.: Pytorch：一种命令式风格的高性能深度学习库（2019），https://arxiv.org/abs/1912.01703
30. Patané, G.: 用于几何处理和形状分析的星形拉普拉斯谱核与距离。载于：计算机图形学论坛。第35卷，第599–624页。Wiley在线图书馆（2016）
31. Patané, G.: 任意域上信号的谱拉普拉斯变换。科学计算期刊96(3), 65（2023）
18 Foti 等人

32. Polthier, K., Schmies, M.: 多面体表面上的最直测地线。收录于：ACM SIGGRAPH 2006 课程。第 30–38 页。SIGGRAPH '06，美国计算机协会，纽约，纽约州，美国 (2006)。https://doi.org/10.1145/1185657.1185664, https://doi.org/10.1145/1185657.1185664
33. Rai, A., Wang, D., Jain, M., Sarafianos, N., Chen, K., Sridhar, S., Prakash, A.: Uvgs: 利用 uv 映射重构非结构化 3d 高斯泼溅。收录于：计算机视觉与模式识别会议论文集。第 5927–5937 页 (2025)
34. Sharp, N., Attaiki, S., Crane, K., Ovsjanikov, M.: Diffusionnet: 表面上的离散化无关学习。ACM 图形学汇刊 (TOG) 41(3), 1–16 (2022)
35. Sharp, N., Crane, K.: 非流形三角网格的拉普拉斯算子。收录于：计算机图形学论坛。第 39 卷，第 69–80 页。Wiley 在线图书馆 (2020)
36. Sharp, N., Soliman, Y., Crane, K.: 向量热方法。ACM 图形学汇刊 (TOG) 38(3), 1–19 (2019)
37. Sun, J., Ovsjanikov, M., Guibas, L.: 基于热扩散的简洁且可证明信息丰富的多尺度签名。收录于：计算机图形学论坛。第 28 卷，第 1383–1392 页。Wiley 在线图书馆 (2009)
38. Verninas, H., Korkmaz, C., Zafeiriou, S., Birdal, T., Foti, S.: 3d 网格的并行化可微分最直测地线。收录于：IEEE/CVF 计算机视觉与模式识别会议论文集 (2026)
39. Vicini, D., Speierer, S., Jakob, W.: 路径回放反向传播：使用恒定内存和线性时间微分光路径。图形学汇刊 (SIGGRAPH 论文集) 40(4), 108:1–108:14 (2021 年 8 月)。https://doi.org/10.1145/3450626.3459804
40. Wardetzky, M., Mathur, S., Kälberer, F., Grinspun, E.: 离散拉普拉斯算子：没有免费的午餐。收录于：几何处理研讨会。第 33 卷，第 37 页。瑞士 Aire-la-Ville (2007)
41. Wiersma, R., Philip, J., Hasan, M., Mullia, K., Luan, F., Eisemann, E., Deschaintre, V.: 使用频率分析进行 svbrdf 获取的不确定性。收录于：计算机图形学与交互技术特别兴趣小组会议论文集。第 1–12 页 (2025)
42. Xiang, F., Xu, Z., Hasan, M., Hold-Geoffroy, Y., Sunkavalli, K., Su, H.: Neutex: 用于体积神经渲染的神经纹理映射。收录于：IEEE/CVF 计算机视觉与模式识别会议论文集。第 7119–7128 页 (2021)
43. Xu, T.X., Hu, W., Lai, Y.K., Shan, Y., Zhang, S.H.: Texture-gs: 解缠 3d 高斯泼溅编辑的几何和纹理。收录于：欧洲计算机视觉会议。第 37–53 页。Springer (2024)
44. Ye, K., Gao, C., Li, G., Chen, W., Chen, B.: Geosplatting: 迈向用于基于物理的逆渲染的几何引导高斯泼溅。收录于：IEEE/CVF 国际计算机视觉会议论文集。第 28991–29000 页 (2025)
45. Yuksel, C.: 网格颜色纹理。收录于：高性能图形学论文集。HPG '17，美国计算机协会，纽约，纽约州，美国 (2017)。https://doi.org/10.1145/3105762.3105780, https://doi.org/10.1145/3105762.3105780
46. Yuksel, C., Keyser, J., House, D.H.: 网格颜色。ACM 图形学汇刊 (TOG) 29(2), 1–11 (2010)
热核纹理 19

47. Zhang, X., Chen, A., Xiong, J., Dai, P., Shen, Y., Xu, W.: Neural shell texture splatting: More details and fewer primitives. In: Proceedings of the IEEE/CVF International Conference on Computer Vision. pp. 25229–25238 (2025)
48. Zhang, X., Ge, X., Xu, T., He, D., Wang, Y., Qin, H., Lu, G., Geng, J., Zhang, J.: Gaussianimage: 1000 fps image representation and compression by 2d gaussian splatting. In: European Conference on Computer Vision. pp. 327–345. Springer (2024)
49. Zhang, Y., Li, B., Kuznetsov, A., Jindal, A., Diolatzis, S., Chen, K., Sochenov, A., Kaplanyan, A., Sun, Q.: Image-gs: Content-adaptive image representation via 2d gaussians. In: Proceedings of the Special Interest Group on Computer Graphics and Interactive Techniques Conference Conference Papers. pp. 1–11 (2025)
50. Zhu, L., Lin, G., Chen, J., Zhang, X., Jin, Z., Wang, Z., Yu, L.: Large images are gaussians: High-quality large image representation with levels of 2d gaussian splatting. In: Proceedings of the AAAI Conference on Artificial Intelligence. vol. 39, pp. 10977–10985 (2025)
51. Zwicker, M., Pfister, H., Van Baar, J., Gross, M.: Surface splatting. In: Proceedings of the 28th annual conference on Computer graphics and interactive techniques. pp. 371–378 (2001)
补充材料

S.1 附加实现细节

图5：HKTex构造过程中不同阶段的热核值。当查询点仅具有标量值时，我们使用颜色映射来表示其范围。为便于表示，我们将源热表示为$$h_t^* = h_t(\mathbf{p}^*, \mathbf{p}^* \mid \theta, \eta)$$。

热核构造可视化。此处我们展示了第4节所述完整热核构造的各个阶段：截断连续各向异性核及其归一化、双调和距离加权、滤波和颜色形成。有趣的是，在早期阶段可以观察到频谱振铃现象；双调和距离加权成功抑制了这些远距离伪影，确保其振幅安全地低于截止值，从而使后续滤波步骤（α）能够完全消除它们。

投影梯度下降（PGD）。为确保HKTEx模型的物理有效性和数值稳定性，我们在优化过程中对颜色、锐度、阈值、角度和各向异性进行约束。不同于使用可能导致梯度消失的sigmoid类激活函数，我们采用投影梯度下降（PGD）在强制约束条件下优化参数。每次优化器迭代后，我们通过严格截断将参数投影回有效可行集。具体而言，我们将核角度约束在θ ∈ [0, π]，锐度约束在ς ∈ [10, 200]，阈值约束在τ ∈ [0.3, 1.0 − 10⁻⁸]，各向异性约束在η ∈ [1, 200]。这些约束范围是通过观察核行为遍历其值时经验性选择的。此外，为使核能够作为平均基色上的加性和减性高频细节层，我们允许单个核颜色在c ∈ [−1, 1]³范围内优化，而全局平均颜色则严格限制在c_base ∈ [0, 1]³。

核参数初始化。对于多视图图像拟合，位置如第5节所述在表面均匀采样，颜色通过从均匀分布采样随机初始化。在UV纹理拟合中，50%的核位置采用均匀采样初始化，另外50%根据UV纹理内容通过计算图像梯度（如文献[49]所述）并将采样的2D点转移到流形上进行初始化。这种随机初始化与基于图像梯度初始化的1:1比例，平衡了高频细节捕捉与保持足够的基础表面覆盖。

![第 20 页](tmp/single_893159bdaee7576c5879caeed57c816c/20.png)

策略中，颜色最初从UV纹理中对应于核源坐标的位置进行采样。

令$u \sim \mathcal{U}(0,1)$为每个初始$S$核采样的均匀随机变量。对于需要均匀分布的参数，我们简单地将$u$缩放到目标范围。具体而言，核角度初始化为$\theta = u \cdot \pi$，各向异性在范围$[1, 100]$内通过$1.0 + 99.0 \cdot u$均匀采样。相反，我们发现将锐度和阈值参数的初始化偏向更高值更有益。这种几何偏差使核从第一步起就保持锐利且高度局部化，防止重叠核间的梯度纠缠。为实现此目的，我们从Beta分布中采样这些参数。实践中，我们通过在对$u$缩放前应用分数幂偏斜来高效实现。核锐度$\varsigma$在范围$[10, 50]$内使用$\varsigma = 10 + 40 \cdot u^{0.5}$初始化。类似地，评估阈值$\tau$在范围$[\tau_{\min}, 1.0)$内使用$\tau = \tau_{\min} + (1.0 - \tau_{\min} - 10^{-8}) \cdot u^{0.7}$初始化，其中$\tau_{\min}$是可配置的下限（我们的UV纹理和多视图拟合实验分别设为0.7和0.9）。

**重要性采样**。为确保优先优化表面细节更丰富的区域，我们采用重要性采样策略。在启动优化过程前，我们在网格表面均匀预采样$10M$个点池。这些点在整个优化过程中用于按公式(7)评估$C(\boldsymbol{p})$，并针对GT颜色$C_{\mathrm{GT}}(\boldsymbol{p})$评估光度损失$\mathcal{L}(\boldsymbol{p})$。在前250个优化步骤中，样本从池中均匀抽取。在此预热阶段后，与每个点相关的作为采样权重的误差$\varepsilon(\boldsymbol{p})$开始使用指数移动平均更新记录：$\varepsilon_{\text{new}}(\boldsymbol{p}) = \gamma \varepsilon_{\text{old}}(\boldsymbol{p}) + (1 - \gamma)\mathcal{L}(\boldsymbol{p})$，衰减系数$\gamma = 0.99$。这些$\varepsilon(\boldsymbol{p})$用于驱动采样分布朝向具有更高重建误差的点。

**质量核密度估计**。如4.2节所述，公式(3)的评估需要连续源点$\boldsymbol{p}^*$处的质量$m^*$。虽然在应用中将其设为单位值不影响性能，但我们为网格表面任意位置的质量估计制定了更通用的核密度估计（KDE）方法。KDE质量的直观理解是：紧密聚集的点应代表较小的表面积比例，而孤立点应代表较大面积。因此，我们首先评估所有源点之间的成对双调和距离[20]。然后，将高斯核应用于成对距离矩阵$\mathbf{D}$，并对行求和以计算每个源的局部密度$(\varrho_s)$：
$$
\varrho_s = \sum_{i=1}^{S} \exp\bigg(-\frac{\mathrm{D}_{i,s}^2}{\sigma_m^2}\bigg),
$$
其中$\sigma_m^2$是通过取所有非零距离的中位数计算的自适应带宽。质量因此定义为密度的倒数：$m_s^* = \frac{1}{\varrho_s}$。
最后，质量会经历额外的归一化步骤，使其总和等于曲面的总面积。

该估计也可用于将热量扩散到与源点不同的位置。在这种情况下，任意点可视为附加源而不改变估计过程。

热核可视化。为了可视化每个网格表面上的热核分布，我们修改方程(6)中引入的滤波函数，将其替换为箱型函数。这种简单修改允许我们追踪空心测地椭球。为便于可视化，我们还随机化其颜色。尽管在大多数情况下利用高密度核来拟合细微反照率变化，但当存在均匀颜色时，HKTEx可通过增加这些区域的稀疏度自适应调整核的位置和大小（参见图2中的地球示例）。

核分裂的补充细节。由于HKTEx的各向异性热核是相对于主曲率定义的（第3节），在需要核分裂的核致密化策略中使用相同轴是自然的。与迄今为止遇到的许多其他量一样，离散化为网格的曲面主曲率也仅在顶点位置定义。因此，第一步需要在旧源位置对主方向进行重心插值：$$\hat{\mathbf{v}}_M^\star = b^{\mathbf{p}_\text{old}^\star}(\mathbf{V}_M)$$。由于以$$\mathbf{p}_\text{old}^\star$$为中心的核$$\boldsymbol{\kappa}_\text{old}$$根据其可学习的$$\theta_\text{old}$$旋转，我们通过绕面法向量($$\hat{\mathbf{n}}$$)旋转$$\hat{\mathbf{v}}_M^\star$$角度$$\theta_\text{old} + \pi/2$$来定义新的主轴向量：
$$
\hat{\mathbf{d}}^\star = \hat{\mathbf{v}}_M^\star \cos\left(\theta_\text{old} + \frac{\pi}{2}\right) + (\hat{\mathbf{n}} \times \hat{\mathbf{v}}_M^\star) \sin\left(\theta_\text{old} + \frac{\pi}{2}\right).
$$
然后通过沿该主核轴相反方向使用文献[38]计算的指数映射移除旧核并替换为两个新核。具体计算为：
$$
\mathbf{p}_\text{new}^{\star(\pm)} = \text{Exp}\left(\mathbf{p}_\text{old}^\star, \pm \hat{\mathbf{d}}^\star (1 - \tau_\text{old}) r_\text{base}\right),
$$
其中$$r_\text{base}$$是基础缩放半径，$$(1 - \tau_\text{old})$$作为核空间半径的比例代理，用于计算指数映射的符号($$\pm$$)决定两个新位置。

最后，新分裂的核必须收缩。其空间范围按1.6倍缩减，产生新阈值$$\tau_\text{new} = 1.0 - \frac{1.0 - \tau_\text{old}}{1.6}$$。此外，这些新参数的优化器状态（如动量和方差）被严格归零，以防止注入的核继承触发分裂的极端梯度。

![第 22 页](tmp/single_893159bdaee7576c5879caeed57c816c/22.png)

**关于渲染和多视图优化的其他细节。**对于多视图优化，我们将 HKTex 实例化为自定义的可微分 Mitsuba3 纹理 [16] 并将其附加到目标网格上的 Principled BSDF [5, 6] 上，目前作为反照率纹理。在训练期间，我们使用透视相机，其传感器捕获 256×256 像素的图像，使用恒定发射器，以及基于 Mitsuba 的 PRB 积分器 [39] 构建的路径回放反向传播 (PRB) 风格的光线积分器 (`rb_ray`)。我们对该自定义 PRB 积分器的实现基于 [41] 的现有扩展。

多视图优化被组织为一个两级循环：外部迭代采样一组新的相机视图并构建相交光线池，而相应的内部迭代通过连续的光线小批量处理该池并应用参数更新。在每次外部训练迭代中，我们在配置的边界内均匀绘制方位角、仰角和相机距离，从而采样一批随机相机。对于每个采样的相机，我们随后为每个像素生成一条主光线并迭代重采样，直到仅保留实际与物体相交的光线。来自所有视图的结果光线集被连接、随机打乱，并划分为 1024 条光线的小批量。这对应于所报告的多视图设置中总共 3840 个优化步骤，计算所有内部光线小批量更新，而不仅仅是外部相机采样迭代。

在优化期间，每条选定的光线都会使用可配置的每像素梯度样本重新评估（我们的结果中使用 $$N_{\text{spp}} = 8$$）。在对具有现有 UV 纹理的模型执行合成训练时，真实图像使用相同的积分器渲染，然后在相同的样本上进行积分。在计算光度损失之前，预测和目标都被裁剪到 $$[0, 1]$$ 并从线性辐射度转换为伽马空间，以匹配我们实验中使用的图像空间监督。在此步骤中，如果目标图像使用高动态范围 (HDR)，则可以省略裁剪。在报告的多视图设置中，我们优化对应于光线小批量平均 L1 损失的每光线光度损失。形式上，令 $$\mathcal{R} = \{r_i\}_{i=1}^B$$ 为当前相交光线的小批量，其中 $$B = 1024$$。对于每条光线 $$r_i$$，渲染器产生 $$N_{\text{spp}}$$ 个蒙特卡洛辐射度样本 $$\{\mathbf{L}_{i,s}\}_{s=1}^{N_{\text{spp}}}$$，对其进行平均以获得预测的线性 RGB 颜色

$$
\hat{\mathbf{c}}_i = \frac{1}{N_{\text{spp}}} \sum_{s=1}^{N_{\text{spp}}} \mathbf{L}_{i,s}.
$$

用 $$\mathbf{c}'_i$$ 表示相应的目标颜色，我们对预测和目标应用相同的图像空间预处理：

$$
\tilde{\mathbf{c}}_i = \Gamma\big(\text{clip}(\hat{\mathbf{c}}_i, 0, 1)\big), \quad \tilde{\mathbf{c}}'_i = \Gamma\big(\text{clip}(\mathbf{c}'_i, 0, 1)\big),
$$

其中 $$\text{clip}(\cdot, 0, 1)$$ 是逐通道应用的，$$\Gamma(\cdot)$$ 表示线性到伽马的颜色变换。光线训练损失随后为：

$$
\mathcal{L}_{\text{ray}} = \frac{1}{B} \sum_{i=1}^B \|\tilde{\mathbf{c}}_i - \tilde{\mathbf{c}}'_i\|_1 \, ,
$$
这对应于我们在多视图实验中使用的L1配置。为提高效率，热核纹理查询以块形式评估（通常每次处理1024个表面点），最终渲染时，我们选择性地对图像平面进行分块（例如32×32裁剪）以限制GPU内存。我们仅对显示的渲染结果使用OptiX降噪器[7,28]，而不在基于射线的训练损失中使用。

需要注意的是，虽然我们使用合成渲染图像进行训练，但真实目标信号可以自然地替换为来自真实照片的屏幕空间图像像素以执行多视图图像训练。然而，由于当前实现仅参数化了主BSDF的反照率分量，直接针对真实照片优化会导致模型将场景光照、阴影和视角相关反射错误地“烘焙”到漫反射颜色中。充分利用基于物理的渲染（PBR）管线需要解耦这些效果。虽然将我们的HKTex公式扩展到联合优化其他材质属性（例如粗糙度、金属度和法线贴图）是高度可行的，但我们目前隔离反照率拟合，使用合成真实值以确保物理精确的材质重建。

超参数和超参数调优。如第5节所述，为确定UV和多视图实验的最佳超参数，我们使用Optuna[1]在Objaverse[11]精选子集的5个网格上执行了贝叶斯超参数优化。这些网格被排除在所有评估集之外。

我们的超参数搜索被表述为多目标优化问题，采用树结构帕森估计器[4]同时最小化最终模型存储大小和3个调优网格渲染视图的均方误差。由于优化动态在不同任务间存在差异，我们对UV纹理拟合和多视图图像拟合分别执行了独立的调优扫描。

搜索空间包含超过30个参数。连续变量（如核参数的学习率和密度控制器阈值）从对数均匀分布中采样。离散分类变量用于选择最佳架构选择，包括初始化策略、损失函数（例如MSE、平滑L1、L1）和学习率调度器（步进、余弦或无）。最终超参数列表报告在表3中。

关于本征属性预计算的说明。在4.1节中，我们详细说明了在半极坐标网格上ALBO本征属性（$$\Phi_{i,j}^{\dagger}$$和$$\Lambda_{i,j}^{\dagger}$$）的预计算。虽然这些本征属性在7×7网格节点上的存储成本平均每个网格约370MB，但它们是确定性计算的量，不需要存储。我们预计算特征分解并在每次运行时缓存一次，跨运行重复使用，且不将预计算包含在计时中。为7×7半极坐标网格预计算ALBO特征分解需要(97.8±286.9)秒。其中，匈牙利方法仅需(0.05±0.03)秒来对齐49个特征向量$$\Phi_{i,j}^{\dagger}$$。这种（确定性）顺序CPU开销可以通过GPU并行化显著优化，使设备上的重新计算足够快。
6 Foti et al.

表 3：超参数配置。我们报告了用于 UV 纹理拟合和多视图图像拟合实验的确切超参数。除了优化步数外，所有参数都是通过我们的多目标 Ray Tune 和 Optuna 优化从搜索空间中动态选择的。

| 类别 | 参数 | UV | 多视图 |
| :--- | :--- | :--- | :--- |
| General (通用) | Optimisation Steps (优化步数) | 5,000 | 3,840 |
| | Batch Size (批次大小) | 4,096 (points) | 1,024 (rays) |
| | Loss Function (损失函数) | MSE | L1 |
| | Random:Gradient Init. (随机:梯度初始化) | 1:1 | NA |
| Architecture (架构) | Initial Sources ($S$) (初始源) | 1,000 | 5,000 |
| | Diffusion Time ($t$) (扩散时间) | $$6.25 \times 10^{-2}$$ | $$1.11 \times 10^{-6}$$ |
| | Colour Formation k-NN ($K_c$) (颜色形成 k-NN) | 30 | 20 |
| | Sources k-NN ($K_s$) (源 k-NN) | 50 | 50 |
| | Init. Threshold ($\tau_{\min}$) (初始化阈值) | 0.3 | 0.7 |
| Learning Rates (学习率) | Mean Colour ($\mathbf{c}_{\text{base}}$) (平均颜色) | $$9.61 \times 10^{-4}$$ | $$1.71 \times 10^{-3}$$ |
| | Kernel Colours ($\mathbf{c}$) (核颜色) | $$6.15 \times 10^{-3}$$ | $$3.33 \times 10^{-2}$$ |
| | Angles ($\theta$) (角度) | $$9.93 \times 10^{-2}$$ | $$1.07 \times 10^{-4}$$ |
| | Anisotropies ($\eta$) (各向异性) | $$4.67 \times 10^{-3}$$ | $$3.66 \times 10^{-2}$$ |
| | Thresholds ($\tau$) (阈值) | $$1.09 \times 10^{-3}$$ | $$1.36 \times 10^{-3}$$ |
| | Sharpnesses ($\varsigma$) (锐度) | $$1.29 \times 10^{-3}$$ | $$1.20 \times 10^{-3}$$ |
| | Locations ($\boldsymbol{p}^{\star}$) (位置) | $$9.91 \times 10^{-2}$$ | $$6.30 \times 10^{-2}$$ |
| | Geodesic Momentum (测地动量) | 0.9 | 0.0 |
| | Adam Scheduler (Adam 调度器) | StepLR | StepLR |
| | Geodesic Scheduler (测地调度器) | StepLR | Cosine Annealing |
| Density Control (密度控制) | Max Kernels Limit (最大核限制) | 5,000 | 10,000 |
| | Pruning Selection Threshold (剪枝选择阈值) | $$1.69 \times 10^{-3}$$ | $$2.49 \times 10^{-3}$$ |
| | Pruning Contrib. Threshold (剪枝贡献阈值) | $$3.42 \times 10^{-2}$$ | $$2.05 \times 10^{-2}$$ |
| | Densify Error Threshold (致密化误差阈值) | $$2.37 \times 10^{-2}$$ | $$2.38 \times 10^{-2}$$ |
| | Densify Size Threshold (致密化尺寸阈值) | 0.174 | 0.181 |

![第 25 页](tmp/single_893159bdaee7576c5879caeed57c816c/25.png)

以完全消除传输需求的同时保留我们的存储优势。我们将此留待未来研究。

关于特征模态和顶点数量依赖性的说明。HKTEx 受特征模态数量影响，但实验中$$K = 256$$已提供足够的控制能力。核尺寸随$$K$$减小而增大，但可学习的$$\tau$$可进行补偿（见插图）。

将 HKTEx 推向极限以拟合由两个三角形组成的矩形图像（图 4），此时$$K = 3$$证明我们的方法不受顶点数量限制，甚至能在比网格更高分辨率下表示纹理。测试数据中的网格平均包含$$N \approx (9.6 \pm 19)k$$个顶点，但我们仅用$$S \approx (4.3 \pm 0.6)k$$个核就超越了替代方法（表 1）。平均 49.8% 的面没有核，13.2% 的面有一个核，但每个面的核中心平均数量为$$4.8 \pm 7.1$$，最大值达到$$70 \pm 123$$。这证明了我们的核可以移动到最有利于纹理质量的位置。

关于 HKTEx 可编辑性的说明。虽然直接编辑具有挑战性（与 GS 方法共有的局限性，其中球谐函数会加剧该问题），但可以通过将目标区域烘焙到密集点云、绘制该区域并运行简要优化来更新 HKTEx 参数。

S.2 补充实验

本节对我们方法的主要组件进行全面消融实验。此外，我们报告了通过将方法拟合到 UV 纹理（图 2）获得的大量定性结果，以及与 SotA 方法在 UV 纹理（图 8 和 9）和多视图图像拟合设置（图 10 和 11）上的额外对比。我们的方法通过生成更清晰、更详细且不受 UV 坐标开销影响的结果，显著优于其他方法。总体而言，HKTEx 在视觉质量上与原始 GT 纹理高度匹配。

表 4：在 Objaverse [11] 的较小子集上进行 UV 纹理拟合的消融结果。最佳值加粗，次佳值加下划线。箭头指示数值越低或越高越好。

模型                PSNR (↑)       LPIPS (↓)        SSIM (↑)       MS-SSIM (↑)    MSE (×10⁻³,↓)   存储 (KB, ↓)   核数 (↓)
完整模型            45.05 ± 4.90   0.011 ± 0.010    0.991 ± 0.008  0.996 ± 0.004   0.058 ± 0.082   96.87 ± 11.90  4811 ± 557
无误差驱动致密化    43.05 ± 5.61   0.020 ± 0.019    0.985 ± 0.015  0.992 ± 0.008   0.104 ± 0.146   22.18 ± 1.01   995 ± 42
无剪枝              45.06 ± 4.91   0.011 ± 0.010    0.991 ± 0.008  0.996 ± 0.004   0.058 ± 0.083   96.89 ± 11.94  4810 ± 558
无密度控制          43.09 ± 5.60   0.020 ± 0.019    0.985 ± 0.015  0.992 ± 0.009   0.102 ± 0.142   22.29 ± 0.54   1000 ± 0
无颜色形成 KNN      45.04 ± 4.92   0.011 ± 0.010    0.991 ± 0.008  0.996 ± 0.004   0.058 ± 0.080   96.75 ± 11.98  4804 ± 561
无双调和加权        43.99 ± 5.71   0.016 ± 0.015    0.988 ± 0.012  0.994 ± 0.006   0.093 ± 0.154   95.59 ± 9.48   4896 ± 427
无重要性采样        45.13 ± 5.86   0.012 ± 0.022    0.988 ± 0.033  0.993 ± 0.027   0.691 ± 6.148   94.39 ± 18.38  4712 ± 836
无 PGD              44.22 ± 5.10   0.014 ± 0.013    0.989 ± 0.010  0.995 ± 0.005   0.071 ± 0.091   96.93 ± 12.84  4853 ± 517

![第 26 页](tmp/single_893159bdaee7576c5879caeed57c816c/26.png)

**消融实验** 在表 4 中，我们提供了通过移除 HKTex 不同组件在 UV 纹理拟合设置下获得的消融结果，评估对象为来自 Objaverse [11] 的 92 个网格的子集，并如前所述进行了过滤。表 4 中的行对应于从完整模型中一次移除一个组件：*No Error-Based Densification*（无基于误差的致密化）禁用了从累积重建误差中插入新核，*No Pruning*（无剪枝）禁用了剪枝步骤，*No Density Control*（无密度控制）移除了致密化和剪枝，*No Colour Formation KNN*（无颜色形成 KNN）禁用了颜色形成-KNN 阶段，*No Biharmonic Weighting*（无双调和加权）禁用了用于强制局部支持的基于距离的衰减，*No Importance Sampling*（无重要性采样）用均匀采样替换了我们的重要性采样，而 *No PGD*（无 PGD）移除了对优化参数的投影范围约束。在这个非 PGD 变体中，有界几何参数改为通过 sigmoid 激活函数强制实施，而核颜色使用 $$ \text{tanh} $$ 激活函数。

总体而言，调整后的完整模型主要由自适应核插入驱动，而不是仅靠剪枝。我们观察到，移除剪枝后最终质量几乎不变，表明基于误差的致密化与流形感知核分裂一起已经提供了足够的适应性。相比之下，移除基于误差的致密化会导致立即退化，而完全移除密度控制并不能挽回这种损失，证实了自适应核插入是主要的密度控制机制。禁用颜色形成-KNN 阶段产生的结果与完整模型的结果实际上是一致的，表明该组件对于准确性并不关键。然而，它仍然可以提供微小的计算优势，因此保留作为实现选择。

其余的消融实验也与相应组件的预期作用一致。移除双调和加权会降低所有图像指标，正如预期其在抑制长程频谱振铃和强制局部支持方面的作用一样。移除重要性采样产生了有竞争力的 PSNR，但 MSE 差得多，表明均匀采样在较容易的区域过拟合，同时未能一致地解决较难的高频区域。移除 PGD 也降低了性能，支持了我们在整个优化过程中显式约束核参数的决定。
热核纹理 9
原始GT UV纹理 HKTEx（我们的方法） HKTEx核
原始GT UV纹理 HKTEx（我们的方法） HKTEx核
图7：我们的HKTEx在拟合UV纹理时的定性结果。我们的纹理所需的存储空间比原始GT纹理大约少一个数量级。

![第 28 页](tmp/single_893159bdaee7576c5879caeed57c816c/28.png)

10 Foti 等人。
原始真实UV纹理 低分辨率真实UV纹理 MLP位置编码 InstantNGP 本征神经场 ImageGS HKTEx（我们的方法）
图8：在拟合UV纹理时与现有最先进方法的额外定性比较。除了真实UV纹理外，所有其他方法的存储大小相似或更大。

![第 29 页](tmp/single_893159bdaee7576c5879caeed57c816c/29.png)

Heat Kernel Textures 11

原始真值 UV 纹理 低分辨率真值 UV 纹理 MLP 位置编码 InstantNGP 内在神经场 ImageGS HKTex ( ours )

图 9：在拟合 UV 纹理时与最先进（SotA）方法的额外定性比较。除了真值（GT）UV 纹理外，所有其他方法都具有相似或更大的存储大小。

![第 30 页](tmp/single_893159bdaee7576c5879caeed57c816c/30.png)

12 Foti et al.

原始 GT UV 纹理	MLP 位置编码	NVDiffRec	HKTex (ours)

图 10：拟合合成多视图图像时与最先进（SotA）方法的定性比较。除 GT UV 纹理外，所有其他方法均具有相似或更大的存储大小。

![第 31 页](tmp/single_893159bdaee7576c5879caeed57c816c/31.png)

Heat Kernel Textures 13

Original GT
UV Textures
MLP Positional
Encoding
NVDiffRec
HKT ex (ours)

图 11：在拟合合成多视图图像时，与最先进（SotA）方法的额外定性比较。除了 GT UV 纹理外，所有其他方法都具有相似或更大的存储大小。

![第 32 页](tmp/single_893159bdaee7576c5879caeed57c816c/32.png)
