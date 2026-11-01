Practical Gaussian Process Implicit Surfaces with Sparse Convolutions
基于稀疏卷积的实用高斯过程隐式曲面

KEHAN XU, Dartmouth College, USA
BENEDIKT BITTERLI, NVIDIA, USA
EUGENE D'EON, NVIDIA, Switzerland
WOJCIECH JAROSZ, Dartmouth College, USA

图 1. 我们用于高斯过程隐式曲面 (GPISes) 的新型稀疏卷积表示首次实现了单一非平稳实现（分屏，中右）与其系综平均光传输（右）的可视化。单一实现允许检查导致聚合粗糙表面或体积外观的随机几何的底层结构。此外，与 Seyb 等人 [2024] 之前的工作（左）相比，我们的算法（中左）在相同时间（40 分钟）下显著提高了系综光传输的渲染效率。最右列展示了比较这两种方法的放大插图，加速比通过均方误差 (MSE) 的比率进行量化。这种改进源于我们高效的稀疏卷积表示与即使在镜面微表面上也能执行下一事件估计的能力的结合。

渲染中的一个根本挑战一直是表面模型和体积模型之间的二分法。高斯过程隐式曲面 (GPISes) 最近为表面、体积及其之间的光谱提供了一种统一的方法。然而，由于高昂的计算成本和数学复杂性，这种表示仍然不切实际。我们通过将 GPISes 重新表述为程序噪声来解决这些限制，消除了昂贵的线性系统求解，同时保持了对空间相关性的控制。我们的方法实现了随机实现的高效采样，并支持通过路径更新灵活地调节值和导数。为了进一步实现实用渲染，我们推导了表面法线的解析分布，允许通过下一事件估计和多重重要性采样进行方差减少的光传输。我们的框架在 CPU 和 GPU 上实现了显著简化的实现，同时保持了原始 GPIS 表示的通用性，从而实现了随机表面和体积的高效、高质量渲染。

CCS 概念：• 计算方法 → 渲染；光线追踪。

其他关键词和短语：随机过程，隐式曲面

ACM 参考格式：
Kehan Xu, Benedikt Bitterli, Eugene d'Eon, and Wojciech Jarosz. 2025. Practical Gaussian Process Implicit Surfaces with Sparse Convolutions. ACM Trans. Graph. 44, 6, Article 188 (December 2025), 18 pages. https://doi.org/10.1145/3763329

1 引言
传统的光传输模拟将场景组件分类为固体表面或参与介质，每种都由不同的渲染方程控制。这种二元区分不足以捕捉现实世界中发现的多样外观，特别是在考虑细节层次技术时，这些技术可能需要将硬表面转换为具有表面相关性的体积效果。
Seyb 等人 [2024] 展示了一个使用通过随机几何的系综平均光传输的统一框架，涵盖了表面、体积及其之间的光谱。一个关键贡献是建立了系综平均传输的严格数学形式体系。尽管其具有潜力，但该方法仍然不切实际，渲染速度比传统路径追踪慢至少两个数量级。这种性能差距产生于

作者地址：Kehan Xu, 达特茅斯学院，美国，kehan.xu.gr@dartmouth.edu; Benedikt Bitterli, NVIDIA, 美国，bbitterli@nvidia.com; Eugene d'Eon, NVIDIA, 瑞士，edeon@nvidia.com; Wojciech Jarosz, 达特茅斯学院，美国，wojciech.k.jarosz@dartmouth.edu.

© 2025 版权归所有者/作者所有。
这是工作的作者版本。发布在此供您个人使用。不得重新分发。最终版本发表于 ACM Transactions on Graphics, https://doi.org/10.1145/3763329.

ACM Trans. Graph., Vol. 44, No. 6, Article 188. Publication date: December 2025.

![第 1 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/1.png)

因为与高斯过程隐式表面（GPIS）相交的射线需要求解大型线性系统，这些系统的复杂度随射线长度呈立方增长。该方法还为创作空间相关性设置了不直观的约束，因为除非特别小心以确保协方差保持正半定函数，否则线性系统会变得奇异。此外，GPIS 背后的数学机制目前对图形学从业者来说不太熟悉，造成了可用性和采用的实质性障碍。

在本文中，我们在缓解这些担忧的每一个方面都取得了显著进展。我们首先展示（第 4 节）高斯过程隐式表面可以由某些类别的程序噪声等效表示。这使我们能够从以图形为中心的视角看待高斯过程，并利用几十年来关于程序噪声高效点评估的工作。通过将高斯过程显式链接到稀疏卷积噪声，我们消除了求解大型线性系统的计算瓶颈，确保了恒定时间的评估和保证的协方差有效性。这提高了性能，使其更直观地创作和编辑这种新表示。

然而，GPIS 先前公式的一个关键优势是其固有的条件化支持。这对于高效计算集成平均传输至关重要，其中递归射线与先前路径段上的观测保持一致。相比之下，程序噪声表示通常缺乏对此类条件化的直接支持。为了克服这一限制，我们引入（第 5 节）一种基于“路径级条件化（pathwise conditioning）”的高效解决方案，它将程序噪声实现的生成与条件化步骤解耦。我们通过支持沿射线的值和导数的联合条件化进一步扩展了这种方法，显著拓宽了基于程序噪声的 GPIS 的实际能力和外观建模潜力。

GPIS 的另一个显著局限性出现在渲染具有高度镜面反射材质的随机几何时。如果不了解射线与 GPIS 相交时遇到的表面法线的条件分布，像 NEE 和多重重要性采样（MIS）[Veach and Guibas 1995] 这样的技术——这对于减少高频照明场景中的方差至关重要——就无法应用。我们通过解析推导所需的法线分布来解决这一限制（第 6 节），这在渲染此类场景时显著减少了方差（图 1）。这极大地扩展了 GPIS 的实际范围，并为未来的双向光传输打开了大门。

最终结果是一个用于非平稳、可控随机表面的快速且高效的框架。这对渲染之外的含义产生了影响，并开启了 GPIS 在创作管道和逆渲染中作为通用、可控表示的潜在用途，涵盖了从表面到体积的范围。

2 相关工作
2.1 随机几何与介质
外观和传输的统计模型是计算机图形学和视觉的基础。典型示例包括微面元模型和参与介质，用于高效渲染复杂表面和体积，而无需显式采样微几何，或用于新视图合成的体积/神经 [Mildenhall et al. 2020] 场景表示。Seyb et al. [2024] 和 Miller et al. [2024] 最近表明，随机几何视角可以统一此类专用模型。例如，Seyb et al. 的 GPIS 框架可以表达 Beckmann [Beckmann and Spizzichino 1963] 微面元表面以及表现出指数或非指数散射行为的体积。至关重要的是，GPIS 支持更长程的记忆和更一般的空间相关性，允许它们捕捉超越标准模型假设的传输效应。这允许提升 Smith 表面理论 [Heitz et al. 2016] 的独立性假设，以及超越通常用于使随机介质中传输可处理的更新 [Bitterli et al. 2018; d'Eon 2018; Jarabo et al. 2018] 和马尔可夫 [Pomraning 1991] 近似。

取决于图像中解析随机微几何的尺度，显式渲染个体实现而不是对它们进行平均可能很重要。我们的方法自然支持这种用例，能够以最小的努力高效渲染详细的非平稳微结构（图 1）。这种能力与之前关于渲染颗粒介质的工作 [Meng et al. 2015; Moon et al. 2007; Müller et al. 2016; Zhang and Zhao 2020] 以及关于闪光和闪烁表面的工作 [Jakob et al. 2014; Yan et al. 2014] 相一致，其中高频表面细节产生视觉上显著、依赖于实现的效果。我们渲染个体实现和集成平均传输的能力是迈向这些极端之间细节水平的一个重要步骤。

2.2 高斯过程
GPIS 在随机几何中建模长程相关性的能力源于高斯过程（GPs）的基础属性——一种灵活且广泛使用的用于建模函数分布的框架。GPs 已在机器学习 [Rasmussen and Williams 2006]、地球统计学 [Murakami et al. 2020]、天体物理学 [Hoffman and Ribak 1991]、控制理论 [Kocijan 2016]、信号处理 [Ricciardi and Sato 1986] 以及计算机图形学和机器人学中的几何建模 [Martens et al. 2017; Sellán and Jacobson 2022, 2023] 中发现了广泛的应用。

GPs 通常有两种解释方式 [Rasmussen and Williams 2006]。函数空间视角直接在一组评估点上定义多元高斯分布，在指定相关性方面提供了极大的灵活性，但代价是显著的复杂性。另一种权重空间视角假设每个实现是基函数的线性组合，提供了一种更具建设性且通常更高效的视角。

Seyb et al. [2024] 采用函数空间视角来推导光通过随机几何传输的形式主义。虽然这使得一般协方差结构和路径依赖条件化成为可能，但它在计算和实现中都引入了复杂性。全局实现是使用带有随机傅里叶特征的权重空间方法可视化的 [Rahimi and Recht 2007]，但这种方法仅限于平稳过程。

相比之下，我们展示了当权重空间视角被框架化为稀疏卷积噪声时，自然地支持平稳和非平稳过程。
非平稳高斯过程。这一视角为 GPIS 实现提供了一条实用且对图形学友好的路径，同时保留了原始模型的表达能力。

2.3 程序噪声
程序噪声长期以来作为图形学中纹理合成和自然现象建模的基石，从分形到地形生成 [Ebert 等人 2003; Lagae 等人 2010; Perlin 1985]。虽然很少用相同语言描述，但许多程序噪声函数近似高斯过程。例如，多个八度的 Perlin 噪声之和近似分数布朗运动。然而，大多数传统噪声函数在空间相关性上提供有限的控制，并且缺乏与高斯过程核心协方差结构的正式联系。

稀疏卷积噪声（SC 噪声）是个例外。由 Lewis [1986, 1989] 引入并由 Lagae 等人 [2009] 扩展，SC 噪声通过用紧凑核卷积白噪声脉冲构建随机场。我们证明这自然产生了高斯过程的权重空间近似，其中协方差函数直接由卷积核的选择控制。进一步的扩展，包括多尺度 [Lagae 等人 2011] 和滤波 [Lagae 和 Drettakis 2011] 变体，实现了空间变化（非平稳）协方差和沿射线的高效评估——这些特性对高效且直观地渲染 GPIS 实现至关重要。

与 Seyb 等人 [2024] 使用的函数空间视图形成对比，后者需要精心设计以保持正半定性，SC 噪声在温和的核条件下保证了有效的高斯过程 [Higdon 等人 1999; Paciorek 和 Schervish 2006]。这极大地简化了构建有效空间相关性的任务。

虽然传统程序噪声不支持条件化，但将 SC 噪声视为高斯过程开启了可能性。朴素地对这种表示进行条件化由于核重叠仍然计算昂贵，但我们采用路径式条件化——一种最初在地统计学 [Journel 和 Huijbregts 1978] 中开发并在天体物理学 [Hoffman 和 Ribak 1991] 中重新发现的技术，之后被机器学习领域重新发现 [Wilson 等人 2020, 2021]。路径式条件化将先验采样与条件化步骤分离。这与 SC 噪声自然契合，因为两者都由核放置定义。我们扩展此方法以允许不仅对值而且对 GP 的导数进行条件化。

3 背景
为了奠定我们对 GPIS 的稀疏卷积公式的基础，我们首先回顾与隐式曲面和高斯过程相关的关键概念和符号。

3.1 隐式曲面
隐式曲面定义为标量函数 f 的等值面。不失一般性，我们将考虑零水平集，即满足 f(p) = 0 的点 p ∈ ℝ³。当 f 可微时，隐式曲面的法向量由归一化梯度给出：n(p) := ∇f(p)/‖∇f(p)‖。

对于一般的 f，计算与隐式曲面的射线交点需要数值求根。尽管存在许多求根方法，但它们难以应用于我们考虑的曲面，因此为简化起见我们将使用光线步进。

3.2 高斯过程
高斯过程 f(p) ~ GP(μ(p), κ(p, p')) 是函数 f 上的分布，其特征由均值 μ(p) = 𝔼_f[f(p)] 和协方差函数 κ(p, p') 刻画，后者编码了任意位置对 p, p' 上 f(p) 和 f(p') 的共变关系。我们在附录 A 中提供了高斯过程的更深入概述。

要成为有效协方差，函数 κ 必须是正半定的。即，对于任意点集 p₁,...,pₙ，条目为 κᵢⱼ = κ(pᵢ, pⱼ) 的协方差矩阵必须是正半定的，即 qᵀκq ≥ 0 对任意 q ∈ ℝ³成立。

3.3 高斯过程隐式曲面上的光传输
高斯过程隐式曲面 (GPIS)  simply 是高斯过程的零水平集。由于高斯过程是函数上的分布，隐式曲面是随机的：我们从高斯过程中抽取的每个实现 f 都会得到不同的曲面。我们可以等效地将实现 f 视为确定性隐式函数 μ(p) 和随机函数 ψ(p) ~ GP(0, κ(p, p')) 的和 f(p) = μ(p) + ψ(p)，其中 ψ(p) 是从均值为零的高斯过程中抽取的。要计算曲面法向量，我们需要协方差函数 κ 二次可微。

在每个实现内，光传输由经典渲染方程给出 [Immel 等人 1986; Kajiya 1986]
$$
L^f(\boldsymbol{p}, \boldsymbol{\omega}) = \int_{\mathcal{S}^2} \rho^f(\boldsymbol{p}_s^f) L^f(\boldsymbol{p}_s^f, \boldsymbol{\omega}_s) \, d\boldsymbol{\omega}_s,
$$
其中 L^f(p, ω) 是从方向 ω 到达点 p 的辐射度，用射线与曲面 f 诱导的最近交点 p_s^f := p + sω 离开的辐射度表示。这通过计算到达 p_s^f 并反射到方向 ω 的光的球面积分得到，后者由余弦加权 BRDF ρ^f(p_s^f) := ρ(p_s^f, ω, ω_s, n_s^f)|n_s^f · ω_s| 给出。我们使用上标 f 表示依赖于实现的量。为简化表述，我们在本文所有方程中省略了发射辐射度 L_e，其包含是直接的。

系综平均传输。L^f 表示特定实现 f 内的传输。总接收辐射度然后是所有可能实现的系综平均：
$$
\langle L^f(\boldsymbol{p}, \boldsymbol{\omega}) \rangle_\zeta = \int_{\text{GP}} L^f(\boldsymbol{p}, \boldsymbol{\omega}) \, d\gamma(f \mid \zeta).
$$
这里，⟨·⟩_ζ = ∫_GP · dγ(f | ζ) 表示对满足约束集 ζ 的实现（例如用户控制编辑施加的约束）的系综平均，γ(f | ζ) 表示经典维纳测度——采样 f ~ GP|ζ 的概率密度。

3.4 内存模型
虽然方程 (2) 可以通过蒙特卡洛估计器轻松近似，该估计器为每条光路径绘制随机实现 f 并执行标准路径采样，但如此朴素地操作——通过预先生成完整实现——使用 Seyb 等人 [2024] 的函数空间采样方法在计算上是不可行的。为了使
在这种实际应用中，Seyb 等人利用了路径追踪仅需观察光线遍历路径上的值这一事实。因此，高斯过程可以通过对路径段上所有先验观测值进行顺序条件化来“即时”（on the fly）采样，其结果等同于路径追踪的完整实现。

虽然这将渲染 GPIS 的成本从 $$O(n^9)$$ 降低到了 $$O(m^3)$$（$$n$$ 为离散化分辨率，$$m$$ 为沿光线的步数），但对于长路径而言，随条件点数量呈立方增长的成本仍然可能无界。为了解决这个问题，Seyb 等人引入了记忆模型（memory models），以便在路径段之间有选择地丢弃条件化（即过去观测到的 $$f$$ 值的“记忆”）。在数学上，这首先是通过解耦后续路径段的集合平均值来实现的：

$$
\langle L^f(\boldsymbol{p}, \boldsymbol{\omega}) \rangle_\zeta = \left\langle \int_{\mathcal{S}^2} \rho^f(\boldsymbol{p}'_s) \langle L^{f'}(\boldsymbol{p}'_s, \boldsymbol{\omega}_s) \rangle_{\zeta'} d\boldsymbol{\omega}_s \right\rangle_\zeta.
\tag{3}
$$

从仅包含先验约束开始，通过对实现 $$f$$ 的平均公式 (1) 获得辐射度，如前所述。然而，上述方程中的递归项 $$L^{f'}$$ 是通过对实现 $$f'$$ 的嵌套平均值（nested average）计算的。该嵌套平均值使用了一组增强的条件 $$\zeta'$$，这将 GP 实现 $$f'$$ 限制为那些与沿射线段 $$[\boldsymbol{p}, \boldsymbol{p}_s]$$ 的 $$f$$ 的观测值相匹配的实现。

此过程等同于前面描述的“即时”采样方法；两者都允许通过对先前观测到的路径信息进行条件化，从一致的实现中采样值。然而在概念上，这种公式化截然不同：它不是预先选择一个实现 $$f$$ 并逐步对其进行采样，而是相当于追踪一条穿过具有不确定性的隐式表面的路径，随着路径段观察到更多值，该路径逐渐被“锁定”。

公式 (3) 对应于 Seyb 等人的全局记忆模型，其中保留了所有路径历史，导致条件约束随路径长度无界增长。具体而言，单次反弹期间的增强条件集为 $$\zeta' = \zeta \wedge \zeta(\boldsymbol{p}, \boldsymbol{p}_s) \wedge \zeta_\delta$$。这包括 $$\zeta(\boldsymbol{p}, \boldsymbol{p}_s)$$，即沿射线段 $$(\boldsymbol{p}, \boldsymbol{p}_s)$$ 在 $$f$$ 中观测到的值，以及 $$\zeta_\delta$$，即 $$\boldsymbol{p}_s$$ 处的值和梯度（即 $$\zeta_\delta = \{ f'(\boldsymbol{p}_t) = 0 \wedge \nabla f'(\boldsymbol{p}_s) = \nabla f(\boldsymbol{p}_s) \}$$）。

为了管理计算成本，Seyb 等人提出了一种 Renewal+ 记忆模型，它将递归平均值 $$\langle L^{f'} \rangle_{\zeta_\delta}$$ 限制为在交点 $$\boldsymbol{p}_s$$ 处具有表面（即满足 $$\zeta_\delta$$）的实现，从而排除任何先验条件 $$\zeta$$ 和沿射线段 $$\zeta(\boldsymbol{p}, \boldsymbol{p}_s)$$ 观测到的值。Seyb 等人表明，在实践中，这种近似只会导致微小的视觉差异，同时通过仅在路径段之间携带固定数量的条件化来保持计算成本有界。

## 4 通过（稀疏）卷积的高斯过程

尽管先前的工作取得了进展，但使用 GPIS 进行渲染仍然昂贵：评估一致实现 $$f$$ 的成本随 $$f$$ 的先验评估数量呈立方增长。Seyb 等人的记忆模型方法为每个路径段“重置”了这个数字，但在每个段内，评估成本仍然随段长度呈立方增长。

在本节中，我们展示了如何以恒定成本评估 GPIS，无论先前提取了多少观测点。我们开始（第 4.1 节）通过展示高斯过程可以写为与核卷积的白噪声。然后我们展示（第 4.2 节）用有限数量的脉冲近似白噪声给出了高斯过程的一致近似，这最终允许在恒定时间内通过求和有限数量的核评估来进行简单且高效的评估方案（第 4.2.2 节）。

### 4.1 来自卷积白噪声的高斯过程

设 $$W$$ 为独立高斯白噪声的一个实现，方差为 $$\sigma^2$$，即对于任何 $$\boldsymbol{p}$$，$$W(\boldsymbol{p}) \propto \mathcal{N}(0, \sigma^2)$$。我们可以将 $$W$$ 写为属于高斯过程 $$W \sim \text{GP}(0, \sigma^2\delta(\boldsymbol{p} - \boldsymbol{p}'))$$，具有 Dirac delta $$\delta$$ 协方差。

现在考虑 $$W$$ 与核 $$h$$ 的卷积（图 2 顶部）：

$$
\psi_{\text{dense}}(\boldsymbol{p}) := \int_{\mathbb{R}^d} h(\boldsymbol{s}, \boldsymbol{p}) W(\boldsymbol{s}) \, d\boldsymbol{s}.
\tag{4}
$$

因为高斯过程上的线性运算会产生另一个高斯过程，且 $$W$$ 属于零均值过程，我们知道卷积白噪声也是一个高斯过程 $$\psi_{\text{dense}}(\boldsymbol{p}) \sim \text{GP}(0, \kappa_{\text{dense}})$$，其协方差为：

$$
\begin{aligned}
\kappa_{\text{dense}}(\boldsymbol{p}, \boldsymbol{p}') &= \mathbb{E}[\psi_{\text{dense}}(\boldsymbol{p}) \psi_{\text{dense}}(\boldsymbol{p}')] \\
&= \iint h(\boldsymbol{s}, \boldsymbol{p}) h(\boldsymbol{u}, \boldsymbol{p}') \mathbb{E}[W(\boldsymbol{s}) W(\boldsymbol{u})] \, d\boldsymbol{s} \, d\boldsymbol{u} \\
&= \underbrace{\mathbb{E}[W^2]}_{\sigma^2} \int h(\boldsymbol{s}, \boldsymbol{p}) h(\boldsymbol{s}, \boldsymbol{p}') \, d\boldsymbol{s}.
\end{aligned}
\tag{5}
$$

**图 2.** 我们展示了 (a) 一个白噪声过程 $$W(\boldsymbol{p})$$ 和 (c) 几个密度递减的稀疏脉冲过程 $$W_\lambda(\boldsymbol{p})$$（橙色、绿色、紫色）。在 (b) 和 (d) 中，我们展示了将这些过程与 (e) 中所示的平稳核 $$h(\boldsymbol{s}, \boldsymbol{p}) = h(\boldsymbol{p} - \boldsymbol{s})$$ 进行卷积的结果。卷积白噪声 $$\psi_{\text{dense}}(\boldsymbol{p})$$ 是一个高斯过程，稀疏卷积噪声 $$\psi_\lambda(\boldsymbol{p})$$ 随着密度增加在视觉上接近 $$\psi_{\text{dense}}(\boldsymbol{p})$$。(f) 显示了关于距离的平均协方差统计量。虽然所有过程共享相同的协方差，但仅凭此信息并不能完全表征它们。参见图 3 进行进一步分析。

![第 4 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/4.png)

这意味着我们可以通过用核卷积白噪声来创建具有非平凡协方差的高斯过程。根据卷积核 $$h$$，所得的协方差函数$^1$ $$\kappa$$ 可能是解析已知的（参见表 1 示例）。然而更重要的是，对于几乎任何 $$h$$，协方差被*保证*是正半定的。

*证明*：将公式 (5) 代入正半定性的定义（3.2 节），我们得到 [Paciorek and Schervish 2006]：
$$
\boldsymbol{q}^\top \boldsymbol{\kappa} \boldsymbol{q} = \sum_{i=1}^n \sum_{j=1}^n q_i q_j \kappa(\boldsymbol{p}_i, \boldsymbol{p}_j)
$$
(6)
$$
= \sigma^2 \sum_{i=1}^n \sum_{j=1}^n q_i q_j \int h(\boldsymbol{s}, \boldsymbol{p}_i) h(\boldsymbol{s}, \boldsymbol{p}_j) \, \mathrm{d}\boldsymbol{s}
$$
$$
= \sigma^2 \int \Bigl( \sum_{i=1}^n q_i h(\boldsymbol{s}, \boldsymbol{p}_i) \Bigr) \Bigl( \sum_{j=1}^n q_j h(\boldsymbol{s}, \boldsymbol{p}_j) \Bigr) \, \mathrm{d}\boldsymbol{s}
$$
$$
= \sigma^2 \int \Bigl( \sum_{i=1}^n q_i h(\boldsymbol{s}, \boldsymbol{p}_i) \Bigr)^2 \, \mathrm{d}\boldsymbol{s} \ge 0.
$$
(7)
只要 $$h$$ 是平方可积的，所得高斯过程的协方差函数就是正半定的。参见附录 B 了解使用频域论据的第二个证明。
这给了我们在控制高斯过程协方差方面很大的自由度，无需仔细分析过程是否仍然有效：我们只需将白噪声与我们选择的（几乎）任何核进行卷积。

**4.2 稀疏卷积噪声作为 GP 近似**
虽然直接评估公式 (4) 不切实际，但 Lewis [1986, 1989] 展示了“稠密”白噪声过程如何通过由随机脉冲过程形成的“稀疏”版本来近似：
$$
W(\boldsymbol{s}) \approx W_\lambda(\boldsymbol{s}) := \sum_i w_i \, \delta(\boldsymbol{s} - \boldsymbol{s}_i).
$$
(8)
脉冲位置 $$\{\boldsymbol{s}_i\}$$ 是不相关的，并且是从密度为 $$\lambda$$ 的泊松点过程中抽取的，以及独立的随机权重 $$\{w_i\}$$，其均值 $$\mathbb{E}[W_\lambda] := 0$$ 且方差 $$\mathbb{E}[W_\lambda^2] := \sigma^2/\lambda$$。
将 $$W_\lambda(\boldsymbol{s})$$ 代入 (4) 得到稀疏卷积噪声 $$\psi_\lambda(\boldsymbol{p})$$（图 2 (c) 和 (d)）：
$$
\psi_{\text{dense}}(\boldsymbol{p}) \approx \psi_\lambda(\boldsymbol{p}) := \sum_i w_i \, h(\boldsymbol{s}_i, \boldsymbol{p}).
$$
(9)
Lagae 等人 [2009] 推导了公式 (9) 的方差，并表明噪声的（单点）分布可以很好地由正态分布 $$\mathcal{N}(0, \sigma^2)$$ 近似。此外，我们可以证明 $$\psi_\lambda$$ 的协方差与其稠密版本 $$\psi_{\text{dense}}$$ 的协方差相匹配（参见附录 C）：
$$
\kappa_\lambda(\boldsymbol{p}, \boldsymbol{p}') = \underbrace{\lambda \mathbb{E}[W_\lambda^2]}_{\sigma^2} \int h(\boldsymbol{s}, \boldsymbol{p}) h(\boldsymbol{s}, \boldsymbol{p}') \, \mathrm{d}\boldsymbol{s}.
$$
(10)

表 1. 卷积核 - 协方差函数对的归一化版本 [Matérn 1960]。我们使用 $$\boldsymbol{s} := \boldsymbol{p} - \boldsymbol{p}'$$ 且 $$s := |\boldsymbol{s}|$$ 以及维度 $$d$$（对于渲染 GPIses，$$d=3$$）。Gamma 函数是 $$\Gamma(n)$$ 且 $$\Lambda_v(s) := v! \, (\frac{s}{2})^v J_v(s)$$，其中 $$J_v(s)$$ 是第一类贝塞尔函数，$$K_v(s)$$ 是第二类修正贝塞尔函数。参数如下：$$v, l$$ 是标量，卷积核和协方差函数的归一化因子 $$C_h^i, C_\kappa^i$$ 分别为：$$C_h^1 = (\sqrt{2\pi}l)^{-d}$$, $$C_\kappa^1 = (2\sqrt{\pi}l)^{-d}$$; $$C_h^2 = 2^d \pi^{-\frac{1+d}{2}} \Gamma(\frac{1+d}{2}) l^{-d}$$, $$C_\kappa^2 = \pi^{-\frac{1+d}{2}} \Gamma(\frac{1+d}{2}) l^{-d}$$; $$C_h^3 = (2\sqrt{\pi}l)^{-d} \Gamma(\frac{1+v}{2}) \Gamma(\frac{1+d+v}{2})^{-1}$$, $$C_\kappa^3 = (2\sqrt{\pi}l)^{-d} \Gamma(v) \Gamma(\frac{d+2v}{2})^{-1}$$; $$C_h^4 = 2\pi^{-\frac{d}{2}} (2l)^{-\frac{3d+2v}{4}} \Gamma(\frac{d+2v}{4})^{-1}$$, $$C_\kappa^4 = 2\pi^{-\frac{d}{2}} (2l)^{-d-v} \Gamma(\frac{d+2v}{2})^{-1}$$。

[表格内容]
名称 | 卷积核 $$h$$ | 协方差函数 $$\kappa$$
--- | --- | ---
Squared exp. | $$C_h^1 \exp(-s^2/2l^2)$$ | $$C_\kappa^1 \exp(-s^2/4l^2)$$
Cauchy | $$C_h^2 (1 + 4s^2/l^2)^{-\frac{d+1}{2}}$$ | $$C_\kappa^2 (1 + s^2/l^2)^{-\frac{d+1}{2}}$$
Bessel | $$C_h^3 \Lambda_{\frac{d+v-1}{2}}(s/l)$$ | $$C_\kappa^3 \Lambda_{\frac{d+2v-2}{2}}(s/l)$$
Matérn | $$C_h^4 s^{\frac{2v-d}{4}} K_{\frac{2v-d}{4}}(s/l)$$ | $$C_\kappa^4 s^v K_v(s/l)$$

图 3. 来自图 2 (b) 和 (d) 的四个过程在特定距离下的单点幅度分布（顶行）和两点联合分布（底行）。图 2 (f) 仅显示了这些分布的均值。随着低脉冲密度 $$\lambda$$，稀疏卷积噪声 $$\psi_\lambda$$ 的分布明显不同于卷积白噪声 $$\psi_{\text{dense}}$$ 的分布。随着密度增加，$$\psi_\lambda$$ 的统计特性趋近于 $$\psi_{\text{dense}}$$，即高斯过程。

虽然稀疏卷积噪声*不是*高斯过程，但它依*分布*收敛于高斯过程：
$$
\psi_\lambda(\boldsymbol{p}) \xrightarrow{d} \mathbf{GP}(0, \kappa_{\text{dense}}(\boldsymbol{p}, \boldsymbol{p}')) \quad \text{as} \quad \lambda \to \infty,
$$
(11)
这是随机测度中心极限定理的结果 [Kallenberg and Kallenberg 1997]。这意味着 $$\psi_\lambda$$ 是高斯过程的一致近似，随着脉冲密度 $$\lambda$$ 的增加变得越来越准确。图 3 可视化了 $$\psi_\lambda$$ 在增加密度下的统计特性，证实了其向高斯过程的收敛。

**4.2.1 解释为具有局部基的权重空间评估。**
有趣的是，稀疏卷积噪声可以被解释为具有无限数量基函数 $$\phi_i(\boldsymbol{p}) = h(\boldsymbol{s}_i, \boldsymbol{p})$$ 和权重 $$w_i$$ 的高斯过程的权重空间近似 (30)。与随机傅里叶特征 [Rahimi and Recht 2007] 相比，如果卷积核具有紧支撑，则只有有限个点位于任何 $$\boldsymbol{p}$$ 周围的 $$h$$ 的支撑内，并且展开变得稀疏。

$^1$一些 GP 文献将协方差函数 $$\kappa$$ 称为协方差*核*。我们将 $$\kappa$$ 称为协方差*函数*以避免与卷积*核* $$h$$ 混淆。

![第 5 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/5.png)

188:6 • Kehan Xu, Benedikt Bitterli, Eugene d'Eon, and Wojciech Jarosz

随机傅里叶特征 (平稳) | 稀疏卷积噪声 (平稳) | (非平稳)
[图 4 图像区域]
图 4. 在左侧两列中，我们通过随机傅里叶特征和稀疏卷积噪声近似相同的 2D 平稳 GP，从上到下基函数（正弦波或重叠核泼溅）的数量增加。稀疏卷积噪声在视觉上更快地收敛于真实 GP，并且可以合成非平稳 GP（右列）。

图 4 的前两列比较了使用稀疏卷积噪声（增加脉冲密度）和随机傅里叶特征（增加特征计数）的 2D GP 实现，计算成本相等。对于稀疏卷积噪声，成本取决于重叠每个查询点的核数量。具有 100 个特征（中间行），稀疏卷积噪声紧密近似真实 GP，而随机傅里叶特征仍然表现出重复的伪影，并且需要更多特征才能达到可比的精度。此外，稀疏卷积噪声可以合成非平稳 GP 实现（右列），而随机傅里叶特征不能。

4.2.2 高效评估。在稀疏卷积视图中工作的一个主要好处是它允许更高效地评估高斯过程。如果 h 具有紧凑支撑（例如，截断高斯），那么只有 O(1) 个脉冲位于任何查询位置 p 附近，每次评估产生 O(1) 成本。这可以即时高效计算 [Lagae et al. 2009]（见算法 1）：我们形成一个网格，其单元格宽度等于 h 的支撑半径，并在每个单元格内生成一组随机脉冲位置 s_i，其中脉冲数量是从密度为 λ 的泊松分布中抽取的。由于核的紧凑支撑，唯一对公式 (9) 有贡献的脉冲位于包含 p 的单元格及其直接相邻单元格中，总共 3^d 个单元格（其中 d 是维度）。脉冲可以使用由单元格索引种子的随机数生成器即时生成，从而在任何地方对高斯过程进行恒定时间评估而无需预计算。通过全局置换每单元格种子，采样过程的不同实现也是微不足道的。

与经典的权值空间近似不同，稀疏卷积噪声还支持使用空间变化卷积核的非平稳过程（图 4 右）。可以选择任何核分布（例如 Gabor 卷积噪声 [Charpenay et al. 2014; Lagae et al. 2009]），尽管核支撑中的极端可变性使得评估效率较低，因为单元格大小基于最坏情况核支撑，这反过来影响了紧密 GP 近似所需的脉冲密度。我们使用多尺度稀疏卷积噪声 [Lagae et al. 2011] – 它可以有效地混合不同长度尺度的噪声独立实现。我们同样可以从稀疏卷积噪声的许多其他改进中受益 [Lagae and Drettakis 2011; Tavernier et al. 2019]。

算法 1 评估稀疏卷积噪声
1: function SCNOISE(p, seed)
2:   f ← 0
3:   λ ← numKernelsPerCell/cellRadius³
4:   cellIndex ← ⌊p/cellRadius⌋
5:   for xyz ∈ {-1, 0, 1}³ do
6:     rng ← RNG(HASH(seed, cellIndex + xyz))
7:     cellPosition ← cellIndex * cellRadius
8:     for i = 1 to numKernelsPerCell do
9:       s_i ← cellPosition + rng.nextFloat3()
10:      l_i, σ_i ← QUERYKERNELPARAMETERS(s_i)
11:      w_i ← (σ_i/√λ) SAMPLESTANDARDNORMAL(rng)
12:      f ← f + w_i h(p, s_i, l_i)
13:  return f

算法 2 渲染全局 GPIS 实现
1: function L(p, ω, seed)
2:   for 每个光线步进步骤 s do
3:     p_s^f ← p + sω
4:     f ← μ(p_s^f) + SCNOISE(p_s^f, seed)
5:     if f ≤ 0 then
6:       n_s^f ← NORMALIZE(∇f)
7:       ω_s, p_ω_s ← SAMPLESCATTEREDDIRECTION(n_s^f, ω)
8:       ρ ← EVALBSDFCOSINE(p_s^f, ω, ω_s, n_s^f)
9:       return L_e(p_s^f) + (ρ/p_ω_s) L(p_s^f, ω_s, seed)
10:  return 0

4.3 渲染全局 GPIS 实现
我们可以通过简单地对随机部分 ψ 进行稀疏卷积噪声 ψ_λ 建模（见算法 2），有效地生成和渲染全局 GPIS 实现 f(p) = μ(p) + ψ(p)。图 7 的左行和中行比较了使用随机傅里叶特征与稀疏卷积噪声来渲染平稳 GPIS。与 2D 情况（图 4）类似，稀疏卷积噪声在 3D 中也是一种更高效的表示，在低特征数量下紧密近似真实 GP（图 7 中上）。利用稀疏卷积噪声，我们可以首次渲染非平稳高斯过程的单个实现（右下），允许长度尺度和各向异性的变化。我们可以通过简单地改变算法 2 中每个追踪路径的随机种子来计算系综平均光传输（图 7 右上）。

5 基于路径更新的调节
现在我们可以有效地采样 GP 实现，我们将注意力转向调节。我们要支持的调节有两种形式：1) 建模时调节以塑造...

![第 6 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/6.png)

图 5. 路径值条件化过程的可视化。我们通过稀疏卷积噪声采样一个先验实现（左），并将其与（中）一个“路径更新”结合，该更新构建为以每个条件点为中心的协方差函数的加权和。权重通过求解一个线性系统计算——其大小等于条件点的数量——确保先验和更新的总和精确插值约束（右）。每个先验实现（左图中的 faded curves）产生一组独特的核权重，生成严格满足值约束的后验实现（右图中的 faded curves）。

图 6. 给定相同的先验分布和约束，函数空间条件化（橙色）和路径更新（蓝色）产生等价的后验分布。我们在值条件化（左）以及值和梯度条件化的混合（右）中评估这种等价性。对于每种情况，我们绘制先验和后验均值（虚线）以及两个采样实现（实线）。值约束由黑点指示，而梯度约束由黑色箭头表示。关键在于，两种方法都满足所有指定的约束，并在每种情况下产生相同的后验均值。

图 7. 随机傅里叶特征（RFF）和我们的稀疏卷积噪声方法都可以渲染平稳 GPIS 的单个实现。与 RFF 相比，我们的方法在特征数量较少的情况下（左上图 vs 中上图）更接近真实 GP。我们的方法还支持渲染具有通用非平稳性（右下图）的 GPIS，包括空间变化的长度尺度和各向异性。集成光传输（右上图）可以通过对每个样本的不同实现进行平均来轻松实现。

...GPIS 在渲染前的属性，以及 2) 渲染时条件化，作为一种例如启用 Seyb 等人提出的连续路径段之间记忆模型的方法。我们可以承担更大的前期成本用于建模时条件化，但希望渲染时条件化尽可能轻量。

不幸的是，传统的条件化方法在我们的上下文中效率低下，并且不容易区分这两个阶段的性能需求。相反，我们展示了如何使用“路径更新”[Wilson et al. 2020, 2021] 来有效地对值进行条件化，然后再推导这种方法对值和梯度联合条件化的扩展。

5.1 值条件化
给定一个包含 $|C|$ 个位置 - 值对的集合 $(C, \boldsymbol{v})$，我们要采样通过这些观察点的实现。与其在采样实现之前进行条件化，不如我们先采样一个先验 GP 实现 $\psi(\cdot) \sim \mathbf{GP}(\mathbf{0}, \kappa)$（通过稀疏卷积噪声），然后添加一个“路径”更新：
$$ \psi_{|C}(\cdot) = \psi(\cdot) + \sum_{i=1}^{|C|} \kappa(\cdot, \boldsymbol{c}_i) u_i \text{ .} \quad (12) $$
这采用了一个先验实现（图 5，左），并添加了 $|C|$ 个以条件点为中心的加权协方差函数 $\kappa(\cdot, \boldsymbol{c}_i)u_i$（图 5，中）。为了强制 $\psi_{|C}(\cdot)$ 通过条件值，权重 $u_i$ 必须满足 $|C| \times |C|$ 线性系统：
$$ \boldsymbol{\kappa}(C, C)\boldsymbol{u} = \boldsymbol{v} - \psi(C), \quad \text{i.e.} \quad \boldsymbol{u} = \boldsymbol{\kappa}(C, C)^{-1}(\boldsymbol{v} - \psi(C)), \quad (13) $$
它是稀疏的（由于 $\kappa$ 的紧支撑）。方程 (13) 可以看作是求解一个 RBF 散乱插值 [Anjyo et al. 2014] 问题（协方差函数作为 RBF 核），以“修正”先验实现 $\psi(\cdot)$ 以匹配条件约束（图 5，右）。
先前的工作 [Wilson et al. 2020, 2021] 已经证明，通过公式 (12) 和 (13) 计算的 GP 实现是从正确的后验分布中抽取的。图 6（左）通过比较使用函数空间和路径条件化从后验 GP 分布中抽取的实现，经验性地证明了这一点。

5.2 梯度条件化
我们也希望支持对 GPIS 的导数进行条件化（例如在建模期间影响局部表面方向，或在渲染期间利用 Renewal+ 记忆模型）。

ACM Trans. Graph., Vol. 44, No. 6, Article 188. Publication date: December 2025.

![第 7 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/7.png)

为了推广路径更新框架以强制导数条件，我们利用这样一个事实：任何高斯过程 $$ \psi \sim \mathbf{GP}(0, \kappa) $$ 的导数是另一个高斯过程 $$ \nabla \psi \sim \mathbf{GP}(0, \nabla_1 \nabla_2^\intercal \kappa) $$，其中 $$ \nabla_1 $$ 和 $$ \nabla_2 $$ 表示分别相对于函数的第一或第二参数的所有元素的梯度算子，而 $$ \nabla_1 \nabla_2^\intercal $$（条目为 $$ \frac{\partial^2}{\partial p_i \partial q_j} $$）是相对于两个参数的所有元素的混合导数算子。
我们可以类比公式 (12) 执行路径梯度更新，但在每个条件点使用一阶导数核的加权和：
$$
\psi_{|C'}(\cdot) = \psi(\cdot) + \sum_{i=1}^{|C'|} \nabla_2^\intercal \kappa(\cdot, \mathbf{c}'_i) \mathbf{u}'_i. \quad (14)
$$
为了看出这产生了正确的梯度条件化实现，我们取其导数
$$
\nabla \psi_{|C'}(\cdot) = \nabla \psi(\cdot) + \sum_{i=1}^{|C'|} \nabla_1 \nabla_2^\intercal \kappa(\cdot, \mathbf{c}'_i) \mathbf{u}'_i, \quad (15)
$$
这正是值路径更新公式 (12)，但使用了梯度过程的先验和协方差。从公式 (13) 和 (15) 中，我们看到权重必须满足：
$$
\nabla_1 \nabla_2^\intercal \boldsymbol{\kappa}(C', C') \mathbf{u}' = \boldsymbol{v}' - \nabla \psi(C'), \quad (16)
$$
即 $$ \mathbf{u}' = [\nabla_1 \nabla_2^\intercal \boldsymbol{\kappa}(C', C')]^{-1} (\boldsymbol{v}' - \nabla \psi(C')) $$，其中 $$ \nabla_1 \nabla_2^\intercal \boldsymbol{\kappa}(C', C') $$ 是一个 $$ (3|C'|) \times (3|C'|) $$ 矩阵，$$ \mathbf{u}' $$ 和 $$ \boldsymbol{v}' $$ 是 $$ (3|C'|) $$-向量（每个条件点 3 个元素），分别表示权重和目标梯度。

5.3 Value-Gradient Conditioning (值 - 梯度条件化)
结合公式 (12) 和 (14) 允许我们要同时条件化值和梯度
$$
\psi_{|C, C'}(\cdot) = \psi(\cdot) + \sum_{i=1}^{|C|} \kappa(\cdot, \mathbf{c}_i) u_i + \sum_{i=1}^{|C'|} \nabla_2^\intercal \kappa(\cdot, \mathbf{c}'_j) \mathbf{u}'_i, \quad (17)
$$
其中权重 $$ \mathbf{u} $$ 和 $$ \mathbf{u}' $$ 满足耦合的 $$ (|C| + 3|C'|) \times (|C| + 3|C'|) $$ 系统：
$$
\begin{bmatrix} \boldsymbol{\kappa}(C, C) & \nabla_2^\intercal \boldsymbol{\kappa}(C, C') \\ \nabla_1 \boldsymbol{\kappa}(C', C) & \nabla_1 \nabla_2^\intercal \boldsymbol{\kappa}(C', C') \end{bmatrix} \begin{bmatrix} \mathbf{u} \\ \mathbf{u}' \end{bmatrix} = \begin{bmatrix} \boldsymbol{v} - \psi(C) \\ \boldsymbol{v}' - \nabla \psi(C') \end{bmatrix}. \quad (18)
$$
图 6（右）说明了 1D 中的这种情况。

5.4 Unidirectional rendering of conditioned GPIses (条件化 GPI 的单向渲染)
路径条件化允许我们在渲染集成平均光传输时有效地结合模型时条件化和 Seyb 等人 的内存模型。
Model-time conditioning (模型时条件化)。当渲染单个实现时，我们可以通过求解公式 (18) 一次来有效地预计算任何模型时条件化权重，然后通过公式 (1) 的蒙特卡洛估计渲染条件化实现。路径条件化 (17) 本质上将空间变化（依赖于实现的）加性项溅射（splats）到正在渲染的 GPIS 上，并且评估它与条件化约束的数量呈线性关系。图 8 说明了使用路径更新来条件化单个 3D GPIS 实现的想法，提供对表面局部高度和方向的控制。

图 8. 值和梯度路径更新允许直观地条件化一个用稀疏卷积噪声创建的 GPIS 实现（左），使其穿过某个位置（中）或采取特定方向（右）。

算法 3 Renewal+ ensemble-average, unidirectional (Renewal+ 集成平均，单向)
1: 函数 $$ L(\boldsymbol{p}, \boldsymbol{\omega}, \boldsymbol{v}', d, \text{seed}) $$
2:     $$ f \leftarrow \mu(\boldsymbol{p}) + \text{SCNOISE}(\boldsymbol{p}, \text{seed}) $$
3:     $$ l_{0, \_} \leftarrow \text{QUERYKERNELPARAMETERS}(\boldsymbol{p}) $$
4:     如果 $$ d = 0 $$ 则 $$ \triangleright $$ 深度 = 0, 相机光线
5:         $$ u, \mathbf{u}' \leftarrow 0, 0 $$
6:     否则
7:         $$ u, \mathbf{u}' \leftarrow \text{GETWEIGHTS}(\boldsymbol{p}, \underbrace{f, 0}_{\text{Values (值)}}, \boldsymbol{\omega}, l_0, \underbrace{\nabla f, \boldsymbol{v}'}_{\text{Gradients (梯度)}}) \triangleright $$ 公式 (18)
8:     对每个光线步进步骤 $$ s $$ 执行
9:         $$ \boldsymbol{p}_s^f \leftarrow \boldsymbol{p} + s \boldsymbol{\omega} $$
10:        $$ f \leftarrow \underbrace{\mu(\boldsymbol{p}_s^f) + \text{SCNOISE}(\boldsymbol{p}_s^f, \text{seed})}_{\text{Unconditioned GP (无条件 GP)}} $$
           $$ + \underbrace{\kappa(\boldsymbol{p}_s^f, \boldsymbol{p}, l_0) u + \nabla_2^\intercal \boldsymbol{\kappa}(\boldsymbol{p}_s^f, \boldsymbol{p}, l_0) \mathbf{u}'}_{\text{Pathwise update (路径更新)}} $$
11:        $$ \triangleright $$ 算法 2 的第 5-8 行
12:        返回 $$ L_e(\boldsymbol{p}_s^f) + \frac{\rho}{p_{\omega_s}} L(\boldsymbol{p}_s^f, \boldsymbol{\omega}_s, \nabla f, d+1, \text{seed}) $$
13: 返回 0

由于路径更新依赖于实现，我们无法在模拟集成平均传输之前预计算权重。然而，我们可以在渲染之前预计算公式 (18) 中系统矩阵的 LU 分解，这将使评估变为二次方。此外，我们可以每个像素样本求解一次权重，然后在估计所有像素的公式 (2) 时重用相同的条件化实现。
Rendering with memory models (使用内存模型渲染)。如果我们采用 Seyb 等人 的 Renewal+ 模型，我们获得了特别简单且高效的算法 3。在这种情况下，渲染时条件化需要仅在散射光线的原点处约束值和梯度。因此，$$ |C| = |C'| = 1 $$ 且公式 (17) 为三个维度中的每一个添加一个加权值条件化溅射和一个加权导数条件化溅射。这使得公式 (18) 成为一个小的 $$ 4 \times 4 $$ 系统，必须在路径追踪期间每个顶点求解一次。评估公式 (17) 则是常数时间。

6 NEXT-EVENT ESTIMATION (下一事件估计)
第 4 节介绍了一种渲染高斯过程隐式表面 (GPIS) 全局实现的有效方法。虽然这使得单个实现的忠实可视化成为可能，但它完全“锁定”了

![第 8 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/8.png)

沿每条光路的几何结构。因此，一旦实现固定，法线变得确定性，消除了方差缩减技术所依赖的基本自由度。当GPIS表面BRDF ρ为镜面反射时，这个问题尤为突出。散射方向ω_s由入射光线方向ω和法线n_s唯一确定，使方程(1)中的积分坍缩，导致传统的下一事件估计(NEE)无法进行。这与经典微面元模型形成鲜明对比，在微面元模型中，微表面法线在找到交点后才被采样。此时渲染算法在着色点维护微面元法线的分布，允许重要性采样策略（如NEE）有效引导光线朝向光源。

虽然需要特定实现来定位GPIS上的交点，但沿路径收集的数据仅形成随机几何的一维切片。实际上，在积分方程(2)时，存在无限多个与这些部分观测一致的实现——所有实现共享相同的交点位置和沿路径段的一致值。因此，我们可以将着色点p_s^f处方向ω_o = -ω的出射辐射度L_o表示为积分：
$$
\langle L_o^f(\boldsymbol{p}_s^f, \omega_o) \rangle_\zeta = \int_{S^2} \rho(\boldsymbol{p}_s^f) \langle L^{f'}(\boldsymbol{p}_s^f, \omega_s) \rangle_\zeta, p_n(\mathbf{n}_s \mid \zeta) \, d\mathbf{n}_s,
$$
其中我们使用简写ρ(p_s^f) := ρ(p_s^f, ω_o, ω_s)表示BRDF，p_n(n | ζ)是这些兼容实现处法线的条件分布，ζ封装了迄今为止关于GPIS的所有观测。从概念上讲，这种法线分布意味着具有完全镜面BRDF的GPIS仍可能呈现非镜面外观，因为p_n通常会产生出射方向的分布。

我们有一种方法可以通过第5节从p_n(n | ζ)采样，即通过在先前所有光线步进步骤中添加条件溅射。然而，这a)成本高昂且b)无法提供评估n的PDF的方法。这是将我们的渲染方法与改进的采样技术（如NEE）结合所缺失的部分。

6.1 从法线到梯度
由于n是归一化梯度∇f，两个分布通过涉及∇f幅值（长度）的变量变换相联系。具体而言，法线分布p_n是梯度分布p_∇在所有梯度幅值t = ||∇f||上的边缘分布：
$$
p_n(\mathbf{n} \mid \zeta) = \int_0^\infty p_\nabla(t\mathbf{n} \mid \zeta) \, t^2 \, dt,
$$
其中雅可比因子t²考虑了球坐标系中的体积变化。

通过这种联系，方程(19)可以重新表述为梯度分布的形式：
$$
\begin{aligned}
\langle L_o^f(\boldsymbol{p}_s^f, \omega_o) \rangle_\zeta &= \int_{S^2} \int_0^\infty \rho(\boldsymbol{p}_s^f) \langle L^{f'}(\boldsymbol{p}_s^f, \omega_s) \rangle_\zeta, p_\nabla(t\mathbf{n}_s \mid \zeta) \, t^2 \, dt d\mathbf{n}_s \\
&= \int_{\mathbb{R}^3} \rho(\boldsymbol{p}_s^f) \langle L^{f'}(\boldsymbol{p}_s^f, \omega_s) \rangle_\zeta, p_\nabla(\nabla f \mid \zeta) \, d(\nabla f) \\
&= \int_{\mathbb{R}^3} \rho(\boldsymbol{p}_s^f) \langle L^{f'}(\boldsymbol{p}_s^f, \omega_s) \rangle_\zeta, p(\mathbf{g} \mid \zeta) \, d\mathbf{g},
\end{aligned}
$$
其中在最后一步我们使用简写g := ∇ψ(p_s^f)，p(g | ζ) := p_∇(∇μ + g | ζ)表示零均值稀疏卷积噪声的梯度。

由于高斯过程ψ的梯度本身是高斯过程，我们知道条件分布p(g | ζ) = N(m, K)确实是ℝ³中具有均值m和协方差K（依赖于条件ζ）的多元高斯分布。如果我们能找到这个均值和协方差，就可以 trivially 使用任意采样PDF的蒙特卡洛积分评估方程(21)。不幸的是，我们没有有效的方法做到这一点：对射线上所有观测进行条件处理，无论是通过函数空间条件(28)还是路径更新（第5节），计算成本都过高。我们将在下一小节展示如何克服这个问题。

6.2 条件梯度分布
不失一般性，我们在局部z轴与光线方向对齐的坐标系中工作，并将该空间中梯度的分量表示为g := (g_x, g_y, g_z)。

我们首先考虑相机光线击中点p_s^f处g(p_s^f)的分布。当从相机行进时，我们在s个等间距位置p_i = (0, 0, z_i)处观测ψ(p)，其中p_s^f是首次穿过(f(p_s^f) = μ(p_s^f) + ψ(p_s^f) = 0)。这给出了长度-s观测向量ζ := (ψ(p_1), ..., ψ(p_s))^T。

ζ和g的联合分布是多元高斯分布：
$$
\begin{bmatrix}
\boldsymbol{\zeta} \\
\mathbf{g}
\end{bmatrix}
\sim \mathcal{N}\left(
\mathbf{0},
\begin{bmatrix}
\mathbf{K}_{\zeta\zeta} & \mathbf{K}_{\zeta\mathbf{g}} \\
\mathbf{K}_{\mathbf{g}\zeta} & \mathbf{K}_{\mathbf{g}\mathbf{g}}
\end{bmatrix}
\right),
\quad \text{其中块}
$$
• [K_ζζ]_ij = Cov(ψ(p_i), ψ(p_j))：ζ中所有条目间的s×s协方差，
• [K_gg]_ab = Cov(∂_aψ(p_s^f), ∂_bψ(p_s^f))：g(p_s^f)梯度分量间的3×3协方差，以及
• [K_gζ] = K_ζg^T]_ai = Cov(∂_aψ(p_s^f), ψ(p_i))：梯度分量与观测值间的3×s互协方差矩阵。

给定ζ的g的条件分布因此为：
$$
p(\mathbf{g} \mid \boldsymbol{\zeta}) = \mathcal{N}(\mathbf{m}, \mathbf{K}), \quad \mathbf{m} = \mathbf{K}_{\mathbf{g}\zeta}\mathbf{K}_{\zeta\zeta}^{-1}\boldsymbol{\zeta}, \quad \mathbf{K} = \mathbf{K}_{\mathbf{g}\mathbf{g}} - \mathbf{K}_{\mathbf{g}\zeta}\mathbf{K}_{\zeta\zeta}^{-1}\mathbf{K}_{\zeta\mathbf{g}}.
$$
6.2.1 平稳各向同性情况。我们现在假设平稳各向同性协方差κ(p_i, p_j) := k(r = |p_j - p_i|)，并讨论如何在6.4节中提升这一假设。在这种情况下，K_gg = k''(0) I_3。此外，沿z轴的所有观测p_i使K_gζ也显著简化：通常[K_gζ]_zi ≠ 0，但由于p_s^f和p_i在x和y坐标上的差异为零，[K_gζ]_xi = [K_gζ]_yi = 0（唯一特殊考虑是i = s，因为此时r = 0，但只要k'(0) = 0，这种简化仍然成立）。这意味着K_gζ在前两行（对应g_x和g_y）包含零，最后一行（对应g_z）可能包含非零元素。因此，g_xy := (g_x, g_y)^T的均值和协方差变得与ζ无关，允许我们将梯度分布分解为：
$$
p(\mathbf{g} \mid \boldsymbol{\zeta}) = p(g_z \mid \boldsymbol{\zeta}) p(\mathbf{g}_{xy}) \quad \text{其中} \quad p(\mathbf{g}_{xy}) = \mathcal{N}(\mathbf{0}, \, k''(0) \, \mathbf{I}_2).
$$
图 9. 我们可视化了各向同性平稳 GP 的梯度分布 $$\mathbf{g}$$，包括无条件分布（c，红色）和沿射线方向导数 $$g_z$$ 的条件分布（c，绿色）。为了清晰起见，我们仅显示 2D 分布 $$g_{xz}$$，$$g_y$$ 垂直于纸面向外。无条件 $$g_{xz}$$ 分布是一个 2D 各向同性高斯分布。当射线在 $$p_s$$ (a) 处与 GPIS 实现 $$f$$ 相交时，$$g_z$$ 可以根据沿射线的值观测值计算得出 (b)。这种条件作用使 $$g_{xz}$$ 减少为遵循 1D 高斯分布。对于从外部与 GPIS 相交的射线，$$g_z$$ 将为负值。GPIS 法线分布 $$p(\mathbf{n} \mid \zeta)$$ 是梯度分布的半球投影（公式 (20)）；固定 $$g_z$$ 后，GPIS 法线分布是 Beckmann NDF 的推广，其本身可以表示为斜率空间中的 2D 高斯分布。

图 10. 我们从自上而下的视角可视化了法线的半球分布，以验证我们要推导的解析分布（公式 (24)，上半部分）与从稀疏卷积噪声实现中采样的梯度（下半部分）相匹配。(a), (b), (c) 显示了无条件和条件分布，对应于图 9 (c) 中描绘的梯度分布。$$g_z$$ 的绝对值越大，法线分布越集中（c 对比 b），而较小的协方差长度尺度（e 对比 b）导致更宽的分布。有趣的是，这两个变量都可以解释为调整 Beckmann 表面粗糙度。我们的交互系统还允许我们验证各向异性协方差核的正确性 (d)，并允许从任意观察方向观察分布 (f)。演示视频在补充材料中提供。

图 9 展示了 $$\mathbf{g}$$ 的分布，无论是无条件分布还是基于采样 $$g_z$$ 的条件分布。图 10 验证并分析了公式 (24) 中推导出的概率分布。

这让我们将公式 (21) 重写为关于 $$g_z$$ 和 $$\mathbf{g}_{xy}$$ 的嵌套积分：
$$
\langle L_o(\mathbf{p}_s^f, \boldsymbol{\omega}_o) \rangle_\zeta = \int_{\mathbb{R}} p(g_z \mid \zeta) \left( \int_{\mathbb{R}^2} \rho(\mathbf{p}_s^f) \langle L^f(\mathbf{p}_s^f, \boldsymbol{\omega}_s) \rangle_\zeta, p(\mathbf{g}_{xy}) \, \mathrm{d}\mathbf{g}_{xy} \right) \mathrm{d}g_z. \quad (25)
$$
所有前面的公式专门解决零均值分量 $$\mathbf{g} = \nabla \psi$$ 的梯度。为了获得 $$\nabla f$$，我们必须简单地将推导出的梯度偏移均值梯度 $$\nabla \mu$$。

6.3 蒙特卡洛估计

我们现在有了使用 NEE 执行 MC 估计的路径。虽然我们不能轻易评估 $$p(g_z \mid \zeta)$$，但我们可以通过生成随机实现 $$f$$（通过稀疏卷积噪声）并找到交点以获得 $$\mathbf{p}_s^f$$ 来从中采样。方向导数 $$g_z(\mathbf{p}_s^f)$$ 的分布完全符合 $$p(g_z \mid \zeta)$$。用这个采样 PDF 估计外部积分会抵消这个因子，留下一个包含我们要现在闭式形式 $$p(\mathbf{g}_{xy})$$ 的 2D 积分。因此我们可以形成蒙特卡洛估计量：
$$
\widehat{L}_o(\mathbf{p}_s^f, \boldsymbol{\omega}_o) = \frac{p(g_z \mid \zeta) \rho(\mathbf{p}_s^f, \boldsymbol{\omega}_o, \boldsymbol{\omega}_s) \langle L^f(\mathbf{p}_s^f, \boldsymbol{\omega}_s) \rangle_\zeta, p(\mathbf{g}_{xy})}{p(g_z \mid \zeta) p_{\mathrm{mc}}(\mathbf{g}_{xy})}, \quad (26)
$$
其中我们使用我们选择的 MC 采样 PDF $$p_{\mathrm{mc}}(\mathbf{g}_{xy})$$。

选择通过查找命中点 $$\mathbf{p}_s^f$$ 处的方向导数 $$g_x(\mathbf{p}_s^f)$$ 和 $$g_y(\mathbf{p}_s^f)$$ 来采样 $$\mathbf{g}_{xy}$$ 将设置 $$p_{\mathrm{mc}}(\mathbf{g}_{xy}) = p(\mathbf{g}_{xy})$$，恢复到我们之前的纯单向策略。为了执行下一事件估计，我们可以改为设计 $$p_{\mathrm{mc}}(\mathbf{g}_{xy})$$ 使得反射方向 $$\boldsymbol{\omega}_s$$ 指向光源。我们还可以使用算法 4 中概述的 MIS 结合这两种策略。这需要 (a) 从 $$\boldsymbol{\omega}_s$$ 和 $$g_z$$ 计算 $$\mathbf{g}_{xy}$$，如图 11 所示，(b) 将光采样 PDF $$p^\omega(\boldsymbol{\omega}_s)$$ 从立体角测度转换为梯度测度 $$p(\mathbf{g}_{xy})$$。我们在图 12 中展示了我们 MIS 策略的有效性，重现了 Veach 的经典 MIS 场景，其中不同粗糙度的金属板通过平稳各向同性 GPIS 建模。

图 11. 一旦 $$g_z$$ 被采样，可能的 3D 梯度 $$\mathbf{g}$$ 集合跨越 (a) 一个以 $$(0, 0, g_z)$$ 为中心平行于 $$X, Y$$ 轴的无限平面。阴影表示 2D 概率密度 $$p(\mathbf{g}_{xy})$$。GPIS 梯度分布 $$\nabla f$$ 是这个平面高斯分布平移均值梯度 $$\nabla \mu$$ (b)。发射器采样需要计算梯度，归一化后等于给定的 $$\mathbf{n} = \frac{\boldsymbol{\omega}_o + \boldsymbol{\omega}_s}{\|\boldsymbol{\omega}_o + \boldsymbol{\omega}_s\|}$$。该梯度位于沿 $$\mathbf{n}$$ 的射线与 $$\nabla f$$ 平面的交点处。

![第 10 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/10.png)

图 12. 我们展示了由 GPIS 表示的平面组成的 Veach MIS 场景，具有完美的镜面微表面和变化的协方差函数长度尺度。较小的长度尺度产生更粗糙的聚合外观。我们的等时（1 分钟）渲染比较评估了三种方法：(a) 单向采样 (UNI)，(b) 我们的下一事件估计 (NEE) 策略，以及 (c) 结合两种策略的多重重要性采样 (MIS)。

算法 4 Renewal+ 集成平均，delta BSDF，MIS
1: function L(p, ω, v', d, seed)
2:   ▷ 算法 3 的第 2-10 行
3:   if f ≤ 0 then
4:     p_s^f ← p + sω
5:     L_o ← L_e(p_s^f)
6:     g^gp ← ∇SCNOISE(p_s^f, seed)
7:     g_z ← g_z^gp                     ▷ 由 UNI 和 NEE 共享
8:   ▷ 下一事件估计
9:     ω_s^nee, p_ω^nee, L_e^nee ← SAMPLELIGHTDIR(p_s^f)
10:    g_xy^nee ← TOGXY(p_s^f, ω_s^nee, g_z)           ▷ 图 11
11:    g^nee ← (g_xy^nee, g_z)
12:    p_xy^nee ← TOGXYMEASURE(p_ω^nee, ω_s^nee, g^nee)
13:    p_xy^gp ← EVALGXYPDF(p_s^f, g_xy^nee)
14:    n_s^nee ← NORMALIZE(∇μ(p_s^f) + g^nee)
15:    ρ^nee ← EVALBSDFCOSINE(p_s^f, ω, ω_s^nee, n_s^nee)
16:    L_o ← L_o + (p_xy^nee / (p_xy^nee + p_xy^gp)) ρ^nee L_e^nee
17:  ▷ 单向采样
18:    n_s^gp ← NORMALIZE(∇μ(p_s^f) + g^gp)
19:    ω_s^gp, _ ← SAMPLESCATTEREDDIRECTION(n_s^gp, ω)
20:    p_ω^gp ← EVALLIGHTDIRPDF(ω_s^gp)
21:    p_xy^nee ← TOGXYMEASURE(p_ω^nee, ω_s^gp, g^gp)
22:    p_xy^gp ← EVALGXYPDF(p_s^f, g_xy^gp)
23:    ρ^gp ← EVALBSDFCOSINE(p_s^f, ω, ω_s^gp, n_s^gp)
24:    L_o ← L_o + (p_xy^gp / (p_xy^nee + p_xy^gp)) ρ^gp L(p_s^f, ω_s^uni, ∇f(p_s^f), d + 1, seed)
     return L_o
25: function TOGXYMEASURE(p^ω(ω_s), ω_s, g)
26:   n_s ← NORMALIZE(g)
27:   p^n(n_s) ← 4(n_s · ω_s)p^ω(ω_s)
28:   return p^n(n_s)|n_s.z|/LENGTH(g)^2

6.4 推广与优化
到目前为止，Eq. (26) 仅对相机射线和平稳各向同性协方差函数有效，但我们可以将其扩展到 Renewal 和 Renewal+ 模型，以及更一般的协方差形式。

内存模型。在 Eq. (26) 中，ζ' 定义了通过内存模型传递给递归光线追踪调用的条件。Renewal 模型（仅记住值 f(p_s^f) = 0 而不记住梯度 ∇f(p_s^f)）在下一条光线的开始处对值 ψ(p_1) 进行条件化。由于 Eq. (24) 已经对沿光线的 g 的所有值观测进行了条件化，它在 Renewal 模型下的次级射线也是有效的。采样 g_z ~ p(g_z | ζ) 只需要生成一个随机实现，并在 p_1 处放置一个单路径条件化 splat。

Renewal+ 内存模型 additionally 在光线开始处的梯度（称为 g'）上进行条件化。由于 g' 在所有三个维度上变化，它会将我们之前的 3D 高斯分布 g（它是各向同性且 x-y 零均值的）转变为更一般的各向异性 3D 高斯分布。尽管如此，仍然可以将分布分解为 p(g) = p(g_z | ζ, g')p(g_xy | g_z, ζ, g') 并采样 g_z ~ p(g_z | ζ, g') 用于 Eq. (26)：我们只需要与一个实现相交，该实现额外放置了一个单梯度条件化 splat 以在 p_1 处进行约束。这在 Eq. (26) 的分子中留下了 p(g_xy | g_z, ζ, g')。现在这将是一个更一般的各向异性 2D 正态分布，但由于它仅依赖于一个额外的梯度观测，我们仍然可以有效地以闭式计算该分布的均值和协方差。

在实践中，我们发现从 2D 各向同性高斯分布采样 g_xy，即 p(g_xy | g_z, ζ) 而不是 p(g_xy | g_z, ζ, g')，可以从我们的 NEE 和 MIS 策略中受益更多。忽略 g_xy 和 g' 之间的相关性定义了一个新的内存模型，我们称之为 Renewal Half+。图 13 的前两行表明 Renewal Half+（右）和 Renewal+（左）的渲染在视觉上相似。最后一行显示，虽然两种内存模型在单向采样下渲染出相似的噪声水平，但 Renewal+ 比 NEE 受益明显更少，因此也比 MIS 受益更少。这是因为对 g' 进行条件化缩小了 g_xy 的分布，使得光线采样策略效果降低。在 Sec. 7.3 中，我们对所有集成平均的光传输结果使用 Renewal Half+。

非平稳性与各向异性。如果 GP 是平稳的但是全局各向异性的，那么其随机分量可以写为 ψ_ani(p) = ψ_iso(Mp) 对于某个 3 × 3 矩阵 M。因此我们可以通过 M 变换光线以进入各向同性空间，然后在评估 Eq. (26) 时分别通过 M⁻¹ 和 Mᵀ 变换 p_s^f 和 g 回来。
我们使用多分辨率稀疏卷积噪声 [Lagae et al. 2011] 来有效地评估具有非平稳长度尺度的 GP。

![第 11 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/11.png)

这使得 $$ \psi(\boldsymbol{p}) $$ 成为两个不同尺度的独立平稳噪声实现的线性组合，$$ \psi_{\text{multi}}(\boldsymbol{p}) = w_1(\boldsymbol{p})\psi_1(\boldsymbol{p}) + w_2(\boldsymbol{p})\psi_2(\boldsymbol{p}) $$，通过空间变化的权重函数 $$ w_1, w_2 $$ 进行混合。Eq. (23) 的梯度协方差和价值 - 梯度交叉方差块仍然可以通过简单应用链式法则以各向同性平稳情况的形式精确写出。在这种情况下，分布 $$ p(\mathbf{g} \mid \boldsymbol{\zeta}) $$ 变为两个 3D 正态分布的组合。

在一般非平稳各向异性的情况下，Eq. (23) 的梯度协方差和价值 - 梯度交叉方差块不会完全简化为 Eq. (24)。然而，如果各向异性变化缓慢，则 Eq. (24) 应保持为一个良好的近似。更准确地说，如果各向异性在来自交点 $$ \boldsymbol{p}_s^f $$ 的长度尺度量级上保持近似恒定，那么（类似于上面的全局各向异性情况）我们可以转换到 $$ \boldsymbol{p}_s^f $$ 处的局部各向同性空间，其中 Eq. (24) 的假设将保持近似成立。

从 3D 到 1D 噪声。我们可以在 Renewal, Renewal Half+, 或 Renewal+ 内存模型下为 Eq. (26) 启用一个关键优化：由于沿光线采样只需要在一个维度上评估噪声，我们可以用 1D 等效物替换 3D 稀疏卷积噪声。这降低了噪声评估的成本，其随维度 $$ d $$ 按 $$ 3^d $$ 缩放。我们在结果部分展示并评估了其对性能的影响。

7 IMPLEMENTATION AND RESULTS
7.1 Implementation
我们在 Tungsten 渲染器 [Bitterli 2018] 中实现了我们的稀疏卷积 GPIS 方法，以便与 Seyb et al. [2024] 提供的实现进行对比。我们在 Intel Xeon Platinum 8488C CPU 上运行了所有等时实验，拥有 48 核和 96 线程。此外，我们在 Shadertoy 中开发了一个简化的 GPU 实现。补充材料包括论文图表中呈现的所有渲染图以及几个展示我们要 GPU 实现的视频。

我们使用均方误差 (MSE)，由于其与渲染时间呈线性缩放，我们通过取方法间 MSE 值的比率来估算加速比。所有评估的方法——我们的方法和 Seyb et al. [2024] 的方法——由于其各自的近似而引入偏差，详见 Sec. 7.3。尽管如此，Fig. 14 以高 SPP 计数渲染了这些方法，经验地展示了它们的视觉相似性。为了计算误差指标，我们使用我们的 MIS 方法渲染了地面真实图像，因为其他技术需要更多的样本才能在设计的场景上收敛。使用此参考计算的误差在所有方法之间提供了公平的比较，特别是对于 Seyb et al. [2024] 的方法，因为误差主要源于蒙特卡洛采样方差而非根本的图像差异。

我们实验中的所有 GPIS 都使用平方指数卷积核。为了渲染具有非平稳协方差函数的 GPIS，我们采用多分辨率稀疏卷积噪声 [Lagae et al. 2011] 以消除对高脉冲密度的需求。

图 13. 中间一行显示了 Renewal+（左）和 Renewal Half+（右）内存模型的收敛渲染图。它们在视觉上是相同的，正如近乎黑色的差异图像（第一行，左）所证实的那样。为了清晰起见，我们还展示了放大了 10 倍的差异（第一行，右）。第三行比较了两种内存模型在 UNI（左）、NEE（中）和 MIS（右）下的性能。虽然 UNI 为两种模型产生了可比的噪声水平，但我们的 NEE 策略（以及因此我们的 MIS）在 Renewal Half+ 下效率显著提高。这种效率的提升是由于前向采样 $$ g_{xy} $$ 分布更宽且与从光照采样导出的梯度分布更好地对齐。

图 14. 我们渲染了具有非平稳长度尺度 GPIS 的 DRAGON 场景。我们在 Renewal+ 内存模型下比较了三种方法：(1) 来自 Seyb et al. [2024] 的函数空间采样，(2) 我们的带有单向路径采样的稀疏卷积噪声方法，以及 (3) 我们结合多重重要性采样 (MIS) 的方法。尽管这三种方法各有近似，但产生的渲染图在视觉上是相似的。这表明稀疏卷积噪声公式和下一事件估计引入的近似误差都很小。所有三个渲染图每像素使用 4096 个样本。

2 代码见于 https://github.com/dartmouth-vcl/sparse-conv-gpis-tungsten.
3 代码见于 https://github.com/dartmouth-vcl/sparse-conv-gpis-shadertoy.

![第 12 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/12.png)

图 15. 我们在相同时间（20 分钟）下，评估了 Seyb 等人 [2024] 的函数空间方法和我们的稀疏卷积方法，场景由具有完美镜面微表面材料的非平稳 GPIS 组成。我们展示了我们方法的几种变体（中间三列插图），包括 3D 噪声实现、具有单向采样的 1D 噪声以及具有 MIS 的 1D 噪声。这些技术是逐步添加的，以展示每种技术带来的改进。此外，在最右列中，我们可视化了 GPIS 的单一实现，提供了对构成整体外观的底层结构的洞察。SPP 和整个图像的加速比显示在底部和顶部插图中心。每个图像块的加速比显示在角落。

内核尺度变化较大的区域。遵循 Tavernier 等人 [2019] 的方法，我们在每个单元中飞溅固定数量的内核（10 个），而不是从泊松分布中采样这个数字。我们在 7.3 节中提供了关于这个数字的分析。对于所有系综平均的光传输结果，我们使用 Renewal Half+ 内存模型。

7.2 场景表示与内存使用

在一点评估 GPIS 需要能够 (1) 查询均值和协方差场，以及 (2) 使用稀疏卷积噪声合成零均值随机分量。随机分量是即时过程生成的，不需要存储样本。

因此，持久内存使用量取决于均值和协方差场的表示方式。这可以是过程式的（无存储）或查表式的（例如体素网格或八叉树）。例如，我们可以通过直接计算查询点到网格三角形的最近距离，以过程方式表达从网格导出的均值函数——对于 $N$ 个三角形，通过 BVH 需要 $$O(\log N)$$ 时间。或者，我们可以通过在体素网格上预计算最近距离来实现 $$O(1)$$ 查找复杂度，但这以增加内存使用为代价。我们在结果中使用了后一种方法，并将场存储在 OpenVDB 网格中。协方差参数同样可以是过程式的或存储在 OpenVDB 网格中。这些存储要求与 Seyb 等人 [2024] 的相同。

![第 13 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/13.png)

7.3 结果

稀疏卷积噪声与随机傅里叶特征。我们在图4中比较了两种用于2D噪声生成的权重空间方法，并在图7（左和中）中比较了用于渲染3D平稳GPIS的方法。两种技术都能高效采样全局实现，尽管随机傅里叶特征（RFF）本质上受限于平稳协方差函数。此外，两幅图都表明，在同等计算成本下，稀疏卷积噪声比RFF更接近真实高斯过程。

稀疏卷积噪声与函数空间采样。Seyb等人[2024]主要提出了使用函数空间方法渲染GPIS。我们将我们的稀疏卷积方法与他们在Renewal+内存模型下渲染集成光传输的方法进行比较，因为他们的方法对采样单次实现不切实际（而我们的方法可以，如图15最右列所示）。

如图14（左和中）所示，两种函数空间采样方法和我们的方法都能产生视觉上可比的渲染结果。虽然两种方法都引入了实用性近似（Seyb等人仅在相邻批次中评估GPIS并传播相关性，而我们的方法利用多分辨率网格），但我们并未发现这些近似在实践中导致显著的视觉差异。

在图15中，我们展示了我们的方法变体与Seyb等人方法的等时比较。我们方法的每个后续变体逐步整合了额外技术，展示了它们对性能提升的个体贡献。对于所有场景，我们以8射线行进点的批次大小评估了函数空间方法。

值得注意的是，仅通过将函数空间采样（图15插图第一列）替换为3D稀疏卷积噪声采样（插图第二列），我们的方法平均比Seyb等人快2.4倍。

从3D到1D噪声。在Renewal、Renewal Half+或Renewal+内存模型下，当通过射线行进采样命中点$p_s^f$和$g_z \sim p(g_z | \zeta)$时，我们只需要沿射线查找值（及其方向导数）。这意味着我们可以用合成1D而非3D噪声来替代，使噪声评估成本降低9倍。剩余的两个梯度分量$g_{xy}$随后可以从正态分布$p(g_{xy})$中显式采样，如公式(24)所示。

图15插图第三列展示了从3D到1D稀疏卷积噪声过渡的性能提升。在计入其他开销（如VDB查询、发射器采样）后，这产生了平均2.3倍的渲染加速。

下一事件估计。我们在第6节推导的技术允许我们应用NEE，即使在微表面使用完美镜面BRDF的情况下。图12比较了单向采样、我们的NEE及其MIS组合在经典Veach场景中的有效性，其中每个平面表示为GPIS。

我们的方法无论是否使用MIS，都会收敛到相同结果，即使对于非平稳GPIS（只要仅尺度而非协方差的各向异性在空间上变化）。我们在非平稳DRAGON场景（图14中和右）中 visually 确认了这一点。

图16。我们的稀疏卷积GPIS框架解锁了可视化和交互式探索微几何个体实现的能力，当集成平均时（b），重现Beckmann微面外观（c）。这对于先前函数空间方法昂贵的线性系统求解是不可行的，但现在即使在简单的基于GPU的Shadertoy实现中也能交互式完成。

图15包含另外两个更复杂的场景，均具有空间变化长度尺度的GPIS、镜面微表面GPIS和有限面积光源。执行NEE的能力显著提高了渲染效率（插图第四列），加速比在不同场景设置中差异显著。

GPU加速原型。GPIS的实现通过稀疏卷积噪声显著简化，实现了跨不同平台的广泛兼容性。作为概念验证，我们演示了在Shadertoy中简化光传输的实时GPU渲染GPIS。这是可能的，因为我们避免了函数空间采样所需的昂贵线性系统求解。图16展示了我们GPU加速原型的截图。演示视频可在补充材料中获取。

空间平均与集成平均。在此，我们探索GPIS空间平均和集成平均之间的联系。当协方差尺度较小时，单次实现的空间平均与跨样本的集成平均非常相似。这种相似性是Seyb等人最初探索集成光传输的基本原理；然而，他们缺乏直接验证，部分原因是他们的函数空间方法不支持渲染个体GPIS实现。

在图17中，我们使用咖啡杯和烟雾验证了这一理论，两者均由具有非平稳协方差函数的单一GPIS表示。(b)和(c)中的空间和集成平均结果显示出强烈的视觉相似性。此外，放大空间平均使我们能够在(a)中可视化GPIS模拟的微观结构，这些结构产生了表面和体积的宏观外观。体积由小粒子组成，而表面呈现凹凸结构，两者均与经典外观模型一致。

虽然空间平均产生与集成平均相似的外观，并 uniquely 实现了微观结构的可视化。

![第 14 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/14.png)

图 17. 我们使用咖啡杯和烟雾验证了空间平均与系综平均之间的联系，二者均使用单个非平稳 GPIS 建模。在亚像素协方差尺度下，空间平均在 (b) 和 (c) 中产生了与系综平均相似的宏观外观。我们的稀疏卷积噪声方法允许我们通过放大空间平均来可视化复杂的表面和体积微观结构 (a)。相反，系综光传输的灵活性使我们能够采用下一事件估计以改进重要性采样。(d) 定性和定量地说明了这种改进，比较了单向路径追踪和 MIS 在相同时间（20 分钟）下的表现。

其作为单一实现的性质防止了在微表面为 delta BSDF 时应用重要性采样技术。另一方面，系综平均中的几何随机性提供了执行下一事件估计的灵活性。图 17 (d) 展示了与单向路径追踪相比，使用 MIS 在相同时间下的性能增益。

每个单元的核数量。稀疏卷积噪声中每个单元的核数量关键影响其逼近高斯过程的准确性。不足的密度可能导致有效透射率、NDF 和最终渲染中的偏差。幸运的是，我们发现平稳 GPIS（即使是各向异性的）可以用非常少的核实现准确的结果。如图 7（中上）所示，即使每个单元只有一个核（27 个重叠核或“特征”），产生的实现也类似于更高密度的对应物。同样，当使用多分辨率噪声实现时，非平稳各向同性缩放不会增加所需的核密度。

由于过程网格单元大小基于保守的最大核半径，当各向异性在空间中快速变化时，核填充空间更为稀疏。在这些情况下，我们发现需要更高的脉冲密度以获得视觉上无偏差的结果。图 18 展示了一个具有变化核密度的非平稳各向异性 GPIS，演示了随着每个单元核数量的增加，偏差如何减小。

8 局限性与未来工作

卷积核 - 协方差函数对。不存在一种通用方法来为任意正定协方差函数推导解析卷积核，反之亦然。例如，我们无法为有理二次协方差函数推导相应的卷积核。尽管如此，我们要强调的是，稀疏卷积 GPIS 仅需要了解卷积核本身。这提供了显著的灵活性，因为我们可以选择任何提供所需空间相关属性的一次可微核。

高效的方向非平稳性。虽然多尺度稀疏卷积噪声有效地处理了长度尺度变化，但表示极端的方向非平稳性仍然具有挑战性。我们当前的方法仅通过低效地增加脉冲密度来实现近似逼近，这需要针对局部变化进行全局过采样。未来的工作可以专注于自适应策略，以高效处理复杂的空间变化各向异性，而无需这种均匀过采样。

光线步进加速。尽管我们取得了进展，但 GPIS 渲染速度仍然受到光线相交成本的约束，由于我们要使用暴力光线步进，该成本随光线长度缩放。最近关于次线性光线步进或隐式表面加速结构的工作 [Moinet and Neyret 2025; Tokuyoshi and Harada 2019] 可能会提供实质性的进一步改进。

改进下一事件估计。我们当前的下一事件估计策略是为 3D GPIS 制定的。对于 2D GPIS（例如高度场），在相交处的梯度分布，以光线观测为条件，只有一个自由度而不是两个。因此，可能的出射方向形成一维分布，所以随机采样二维面光源提供贡献的几率为零。因此，这种情况需要专门的采样技术。

我们针对完美反射材料的 NEE 推导很容易扩展到折射材料。虽然我们当前的策略对非 delta BSDF（图 15 的最后一行）没有产生改进，但解析梯度分布为进一步增强其重要性采样技术提供了机会。例如，如果微 BSDF（即 GPIS 表面上的 BSDF）遵循高斯分布，它可以与梯度分布（另一个高斯分布）进行卷积，以产生解析宏观 BSDF。否则，其中的项...

![第 15 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/15.png)

被积函数可以单独进行重要性采样，不同的策略通过 MIS（多重重要性采样）结合。

GPIS 编辑工具。作为一种新颖的几何表示，GPIS 提供了直观的编辑能力，正如我们在图 8 中概念性地展示的那样。开发涵盖建模和纹理处理的 GPIS 艺术工具是一个理想的未来方向。这一过程不仅提出了工程挑战，也带来了有趣的研究问题，例如建立 GPIS 参数与传统外观模型之间的映射，探索不确定性操作的机制，以及将成熟的 3D 噪声纹理技术应用于 GPIS。

互易性。为了启用双向渲染算法，如双向路径追踪或光子映射，我们必须严格证明整体光传输积分的互易性。假设确定性的相机和光源（其中只有中间路径顶点位于 GPIS 上），我们预见整体光传输互易性没有根本障碍。然而，更具挑战性的场景将涉及随机光源，即发射性 GPIS。

9 结论

我们在高斯过程隐式曲面（Gaussian Process Implicit Surfaces, GPIS）的实际应用中提出了重大进展。我们的核心贡献是一种新颖的稀疏卷积噪声公式，它提供了比先前函数空间方法更直观的表示。这种新表示使得全局非平稳 GPIS 实现的实际采样成为可能，并提高了整体光传输的渲染效率。我们进一步引入了路径更新，以便在建模和渲染期间对灵活的值和梯度进行条件处理。至关重要的是，通过将解析法线分布推导为高斯分布，我们实现了对镜面微表面的下一事件估计。这进一步提高了渲染效率，弥合了 GPIS 渲染与传统外观模型重要性采样之间的差距。由此产生的基于噪声的方法易于在 CPU 或 GPU 平台上实现。

致谢

我们感谢 Dario Seyb 就 GPIS 进行的富有洞察力的讨论，感谢 Jeffrey Liu 提供的有益反馈。这项工作得到了 NSF 奖项 1844538 和 2440472 的部分支持。我们在本文的许多图中使用了根据 CC0 许可的资产，并感谢原始创作者提供它们：着色器球 [Takikawa 等人 2022]，棋盘（polyhaven.com，由 Riley Queen 制作），龙（OpenVDB 示例模型），两座雕像（threedscans.com），以及咖啡杯（Sketchfab）。我们与 LLM 结对编程，以完善几个程序生成图的代码和布局：图 2、3、5、6 和 19。

参考文献

Ken Anjyo, J. P. Lewis, 和 Frédéric Pighin. 2014. 计算机图形学的散乱数据插值。载于 ACM SIGGRAPH 课程。ACM 出版社，美国纽约，1–69。https://doi.org/10/gt2jcm
Petr Beckmann 和 André Spizzichino. 1963. 粗糙表面对电磁波的散射。Pergamon 出版社，纽约。
Benedikt Bitterli. 2018. Tungsten 渲染器。https://github.com/tunabrain/tungsten
Benedikt Bitterli, Srinath Ravichandran, Thomas Müller, Magnus Wrenninge, Jan Novák, Steve Marschner, 和 Wojciech Jarosz. 2018. 非指数介质的辐射传输框架。ACM 图形学汇刊（SIGGRAPH Asia 会议论文集）37, 6 (2018 年 11 月), 225:1–225:17。https://doi.org/10/gf2cm
Victor Charpenay, Bernhard Steiner, 和 Przemysław Musialski. 2014. 空间域中的 Gabor 噪声采样。载于第 30 届计算机图形学春季会议论文集 (SCCG '14)。计算机协会，美国纽约，79–82。https://doi.org/10/f3s4cw
Eugene d’Eon. 2018. 非指数辐射传输的互易公式。1：草图与动机。arXiv:1803.03259 [nucl-th, physics:physics] (2018 年 3 月)。arXiv:1803.03259 [nucl-th, physics:physics]
David S. Ebert, F. Kenton Musgrave, Darwyn Peachey, Kenneth Perlin, 和 Steven Worley. 2003. 纹理与建模：程序化方法（第 3 版）。Morgan Kaufmann，美国旧金山。
Dennis Gabor. 1946. 通信理论。第一部分：信息分析。电气工程师学会期刊 - 第三部分：无线电与通信工程 93, 26 (1946), 429–441。
Eric Heitz, Johannes Hanika, Eugene d’Eon, 和 Carsten Dachsbacher. 2016. 基于 Smith 模型的多散射微表面 BSDF。ACM 图形学汇刊（SIGGRAPH 会议论文集）35, 4 (2016 年 7 月)。https://doi.org/10/f89kmn
David Higdon, Jenise Swall, 和 John Kern. 1999. 非平稳空间建模。载于贝叶斯统计 6：第六届瓦伦西亚国际会议论文集，J M Bernardo, J O Berger, A P Dawid, 和 A F M Smith (编)。牛津大学出版社。https://doi.org/10.1093/oso/9780198504856.003.0036
Yehuda Hoffman 和 Erez Ribak. 1991. 高斯场的约束实现：一种简单算法。天体物理学期刊 380 (1991 年 10 月), L5。https://doi.org/10/dngzjw
David S. Immel, Michael F. Cohen, 和 Donald P. Greenberg. 1986. 非漫反射环境的辐射度方法。计算机图形学（SIGGRAPH 会议论文集）20, 4 (1986 年 8 月), 133–142。https://doi.org/10/dmjn9t
Wenzel Jakob, Miloš Hašan, Ling-Qi Yan, Jason Lawrence, Ravi Ramamoorthi, 和 Steve Marschner. 2014. 离散随机微表面模型。ACM 图形学汇刊（SIGGRAPH 会议论文集）33, 4 (2014 年 7 月), 115:1–115:10。https://doi.org/10/f6cnzx
Adrian Jarabo, Carlos Aliaga, 和 Diego Gutierrez. 2018. 空间相关材料的辐射传输框架。ACM 图形学汇刊（SIGGRAPH 会议论文集）37, 4 (2018 年 7 月), 83:1–83:13。https://doi.org/10/gd52qq
A. G. Journel 和 Ch J. Huijbregts. 1978. 矿业地质统计学。Academic 出版社。
James T. Kajiya. 1986. 渲染方程。计算机图形学（SIGGRAPH 会议论文集）20, 4 (1986 年 8 月), 143–150。https://doi.org/10/cvf53j
Olav Kallenberg 和 Olav Kallenberg. 1997. 现代概率论基础。第 2 卷。Springer。
Juš Kocijan. 2016. 使用高斯过程模型对动态系统进行建模与控制。Springer 国际出版社，查姆。https://doi.org/10.1007/978-3-319-21021-6
Ares Lagae 和 George Drettakis. 2011. 滤波实体 Gabor 噪声。ACM 图形学汇刊 30, 4 (2011 年 7 月), 51:1–51:6。https://doi.org/10/bdm39b
Ares Lagae, S. Lefebvre, Rob Cook, Tony DeRose, George Drettakis, David S. Ebert, J. P. Lewis, Ken Perlin, 和 Matthias Zwicker. 2010. 程序化噪声函数的现状。载于 Eurographics 2010 - 现状报告。欧洲图形学协会。https://doi.org/10/gtxh9f
Ares Lagae, Sylvain Lefebvre, George Drettakis, 和 Philip Dutré. 2009. 使用稀疏 Gabor 卷积的程序化噪声。ACM 图形学汇刊（SIGGRAPH 会议论文集）28, 3 (2009 年 7 月), 1–10。https://doi.org/10/cvv5s7
Ares Lagae, Sylvain Lefebvre, 和 Philip Dutre. 2011. 改进 Gabor 噪声。IEEE 可视化与计算机图形学汇刊 17, 8 (2011 年 8 月), 1096–1107。https://doi.org/10/cr3qwx
J. P. Lewis. 1986. 随机频谱合成方法。载于图形界面与视觉界面 '86 会议论文集 (Gi '86)。加拿大人机通信协会，加拿大多伦多，173–179。http://graphicsinterface.org/wp-content/uploads/gi1986-30.pdf
J. P. Lewis. 1989. 实体噪声合成算法。载于第 16 届计算机图形学与交互技术年度会议论文集 (SIGGRAPH '89)。计算机协会，美国纽约，263–270。https://doi.org/10/cw7wzj
Wolfram Martens, Yannick Poffet, Pablo Ramon Soria, Robert Fitch, 和 Salah Sukkarieh. 2017. 高斯过程隐式曲面的几何先验。IEEE 机器人与自动化快报 2, 2 (2017 年 4 月), 373–380。https://doi.org/10/ghk78w
Bertil Matérn. 1960. 空间变异。博士论文。斯德哥尔摩大学，示例城市，CA。
Johannes Meng, Marios Papas, Ralf Habel, Carsten Dachsbacher, Steve Marschner, Markus Gross, 和 Wojciech Jarosz. 2015. 颗粒材料的多尺度建模与渲染。ACM 图形学汇刊（SIGGRAPH 会议论文集）34, 4 (2015 年 7 月), 49:1–49:13。https://doi.org/10/gfzndr
Ben Mildenhall, Pratul P. Srinivasan, Matthew Tancik, Jonathan T. Barron, Ravi Ramamoorthi, 和 Ren Ng. 2020. NeRF：将场景表示为用于视图合成的神经辐射场。载于欧洲计算机视觉会议 (ECCV)。https://doi.org/10/gn9vj9
Bailey Miller, Hanyu Chen, Alice Lai, and Ioannis Gkioulekas. 2024. Objects as Volumes: A Stochastic Geometry View of Opaque Solids. In IEEE/CVF International Conference on Computer Vision and Pattern Recognition (CVPR). 87–97. https://doi.org/10/gt258p
Mathéo Moinet and Fabrice Neyret. 2025. Fast sphere tracing of procedural volumetric noise for very large and detailed scenes. In Computer Graphics Forum. Wiley Online Library, e70072.
Jonathan T. Moon, Bruce Walter, and Stephen R. Marschner. 2007. Rendering Discrete Random Media Using Precomputed Scattering Solutions. In Rendering Techniques (Proceedings of the Eurographics Symposium on Rendering). Eurographics Association, 231–242. https://doi.org/10/gfzp5n
Thomas Müller, Marios Papas, Markus Gross, Wojciech Jarosz, and Jan Novák. 2016. Efficient Rendering of Heterogeneous Polydisperse Granular Media. ACM Transactions on Graphics (Proceedings of SIGGRAPH Asia) 35, 6 (Nov. 2016), 168:1–168:14. https://doi.org/10/f9cm65
Daisuke Murakami, Yoshiki Yamagata, and Toshihiro Hirano. 2020. Geostatistics and Gaussian Process Models. In Spatial Analysis Using Big Data, Yoshiki Yamagata and Hajime Seya (Eds.). Academic Press, 57–112. https://doi.org/10.1016/B978-0-12-813127-5.00004-7
Christopher J. Paciorek and Mark J. Schervish. 2006. Spatial Modelling Using a New Class of Nonstationary Covariance Functions. Environmetrics 17, 5 (Aug. 2006), 483–506. https://doi.org/10/bt6vvk
Ken Perlin. 1985. An Image Synthesizer. Computer Graphics (Proceedings of SIGGRAPH) 19, 3 (July 1985), 287–296. https://doi.org/10/bbsdjx
Gerald C Pomraning. 1991. Linear Kinetic Theory and Particle Transport in Stochastic Mixtures. Vol. 7. World Scientific Press, Singapore. https://doi.org/10/mr49
Ali Rahimi and Benjamin Recht. 2007. Random Features for Large-Scale Kernel Machines. In Advances in Neural Information Processing Systems (NIPS). Curran Associates Inc., Red Hook, NY, USA, 1177–1184.
Carl Edward Rasmussen and Christopher K. I. Williams. 2006. Gaussian Processes for Machine Learning. MIT Press, Cambridge, Mass. https://gaussianprocess.org/gpml/
Luigi M. Ricciardi and Shunusuke Sato. 1986. On the Evaluation of First Passage Time Densities for Gaussian Processes. Signal Processing 11, 4 (Dec. 1986). https://doi.org/10/cqq2hg
Silvia Sellán and Alec Jacobson. 2022. Stochastic Poisson Surface Reconstruction. ACM Transactions on Graphics (Proceedings of SIGGRAPH) 41, 6 (Dec. 2022). https://doi.org/10/mr5b
Silvia Sellán and Alec Jacobson. 2023. Neural Stochastic Screened Poisson Reconstruction. ACM SIGGRAPH Asia Conference Papers (2023). https://doi.org/10/mr5c
Dario Seyb, Eugene d’Eon, Benedikt Bitterli, and Wojciech Jarosz. 2024. From Microfacets to Participating Media: A Unified Theory of Light Transport with Stochastic Geometry. ACM Transactions on Graphics (Proceedings of SIGGRAPH) 43, 4 (July 2024). https://doi.org/10/gt5nh9
Towaki Takikawa, Andrew Glassner, and Morgan McGuire. 2022. A Dataset and Explorer for 3D Signed Distance Functions. Journal of Computer Graphics Techniques (JCGT) 11, 2 (April 2022), 1–29. http://jcgt.org/published/0011/02/01/
Vincent Tavernier, Fabrice Neyret, Romain Vergne, and Joëlle Thollot. 2019. Making Gabor Noise Fast and Normalized. (2019). https://doi.org/10/g825mg
Yusuke Tokuyoshi and Takahiro Harada. 2019. Hierarchical russian roulette for vertex connections. ACM Transactions on Graphics (TOG) 38, 4 (2019), 1–12.
Eric Veach and Leonidas J. Guibas. 1995. Optimally Combining Sampling Techniques for Monte Carlo Rendering. In Annual Conference Series (Proceedings of SIGGRAPH), Vol. 29. ACM Press, 419–428. https://doi.org/10/d7b6n4
James Wilson, Viacheslav Borovitskiy, Alexander Terenin, Peter Mostowsky, and Marc Deisenroth. 2020. Efficiently Sampling Functions from Gaussian Process Posteriors. In Proceedings of the 37th International Conference on Machine Learning. PMLR, 10292–10302. https://doi.org/10/gt2hjn
James T. Wilson, Viacheslav Borovitskiy, Alexander Terenin, Peter Mostowsky, and Marc Peter Deisenroth. 2021. Pathwise Conditioning of Gaussian Processes. Journal of Machine Learning Research 22, 5 (July 2021). https://doi.org/10/gt2hjr
Ling-Qi Yan, Miloš Hašan, Wenzel Jakob, Jason Lawrence, Steve Marschner, and Ravi Ramamoorthi. 2014. Rendering Glints on High-Resolution Normal-Mapped Specular Surfaces. ACM Transactions on Graphics (Proceedings of SIGGRAPH) 33, 4 (July 2014), 116:1–116:9. https://doi.org/10/f6cpr
Cheng Zhang and Shuang Zhao. 2020. Multi-Scale Appearance Modeling of Granular Materials with Continuously Varying Grain Properties. In Eurographics Symposium on Rendering – DL-only Track. The Eurographics Association, 13 pages. https://doi.org/10/ghcp26

A GAUSSIAN PROCESSES

高斯过程 $f(\boldsymbol{p}) \sim \text{GP}(\mu(\boldsymbol{p}), \kappa(\boldsymbol{p}, \boldsymbol{p}'))$ 是定义在 $d$ 维输入空间上的随机函数，使得其在任意有限数量 $n$ 个位置 $P \coloneqq \{\boldsymbol{p}_1, \dots, \boldsymbol{p}_n\}$ 处的评估值遵循 $n$ 维高斯分布：

$$
\begin{bmatrix} f(\boldsymbol{p}_1) \\ \vdots \\ f(\boldsymbol{p}_n) \end{bmatrix}_{f(P) \in \mathbb{R}^n} \sim \mathcal{N} \left( \begin{bmatrix} \mu(\boldsymbol{p}_1) \\ \vdots \\ \mu(\boldsymbol{p}_n) \end{bmatrix}_{\mu(P) \in \mathbb{R}^n}, \begin{bmatrix} \kappa(\boldsymbol{p}_1, \boldsymbol{p}_1) & \cdots & \kappa(\boldsymbol{p}_1, \boldsymbol{p}_n) \\ \vdots & \ddots & \vdots \\ \kappa(\boldsymbol{p}_n, \boldsymbol{p}_1) & \cdots & \kappa(\boldsymbol{p}_n, \boldsymbol{p}_n) \end{bmatrix}_{\kappa(P, P) \in \mathbb{R}^{n \times n}} \right), \quad (27)
$$

其中 $\mu(\boldsymbol{p}) \coloneqq \mathbb{E}[f(\boldsymbol{p})]$ 是确定性均值函数，$\kappa(\boldsymbol{p}, \boldsymbol{p}') \coloneqq \mathbb{E}[(f(\boldsymbol{p}) - \mu(\boldsymbol{p}))(f(\boldsymbol{p}') - \mu(\boldsymbol{p}'))]$ 是协方差函数（核）。高斯过程是平稳的（stationary），当且仅当 $\kappa(\boldsymbol{p}, \boldsymbol{p}') = \kappa(\boldsymbol{p} - \boldsymbol{p}')$；若是各向同性的（isotropic），则协方差仅取决于距离 $\kappa(\boldsymbol{p}, \boldsymbol{p}') = \kappa(\|\boldsymbol{p} - \boldsymbol{p}'\|)$。我们用 $\|\boldsymbol{p}\|$ 表示 $d$ 维向量的 $L_2$ 范数，用 $|P|$ 表示集合的基数。我们主要关注输入维度 $d = 1, 2, 3$ 的高斯过程。机器学习中常见的协方差函数包括平方指数、有理二次和马特恩核。

由于导数算子的线性性质，高斯过程 $f$ 的梯度 $\nabla f(\boldsymbol{p})$ 仍然是一个高斯过程，它与值过程联合高斯分布：

$$
\begin{bmatrix} f(\boldsymbol{p}) \\ \nabla f(\boldsymbol{p}) \end{bmatrix} \sim \mathcal{N} \left( \begin{bmatrix} \mu(\boldsymbol{p}) \\ \nabla \mu(\boldsymbol{p}) \end{bmatrix}, \begin{bmatrix} \kappa(\boldsymbol{p}, \boldsymbol{q}) & \nabla_{\boldsymbol{q}}^\intercal \kappa(\boldsymbol{p}, \boldsymbol{q}) \\ \nabla_{\boldsymbol{p}} \kappa(\boldsymbol{p}, \boldsymbol{q}) & \nabla_{\boldsymbol{p}} \nabla_{\boldsymbol{q}}^\intercal \kappa(\boldsymbol{p}, \boldsymbol{q}) \end{bmatrix} \right), \quad (28)
$$

其中 $\nabla_{\boldsymbol{p}} \coloneqq [\partial/\partial p_x, \partial/\partial p_y, \partial/\partial p_z]^\intercal$ 表示关于 $\boldsymbol{p}$ 的梯度算子。因此，梯度过程的协方差由协方差的二阶导数定义。

A.1 Sampling GP priors

我们可以在“函数空间”或“权重空间”视图中创建 GP 实现（“样本 GP 先验”）。

函数空间（Function-space）。给定一组 $n$ 个期望的评估位置，我们可以从联合分布 (27) 中对它们的值进行采样：

$$
f(P) = \mu(P) + \kappa^{\frac{1}{2}}(P, P) \xi, \quad \text{with} \quad \xi \sim \mathcal{N}(\mathbf{0}, \mathbf{I}). \quad (29)
$$

这需要一个矩阵“平方根”$\kappa^{\frac{1}{2}}$，如乔列斯基分解（Cholesky decomposition），使其复杂度为 $O(|P|^3)$。

权重空间（Weight-space）。权重空间近似首先假设实现 $f$ 可以表示为 $b$ 个基函数 $\Phi(\boldsymbol{p}) \coloneqq (\phi_1(\boldsymbol{p}) \dots \phi_b(\boldsymbol{p}))^\intercal$ 的线性组合：

$$
f(\cdot) = \mu(\cdot) + \sum_{i=1}^b \phi_i(\cdot) w_i = \mu(\cdot) + \Phi(\cdot)^\intercal \mathbf{w}, \quad (30)
$$

其中 $\mathbf{w}$ 是权重向量。对随机权重向量 $\mathbf{w} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$ 进行采样会产生一个可以在任何点进行评估的实现，并且现在的评估复杂度为 $O(b|P|)$——与评估点数量 $|P|$ 呈线性关系。

通过公式 (30) 采样的实现的协方差是 $\kappa(P, P) = \Phi(P) \Phi^\intercal(P)$，其中 $\Phi(P)$ 是 $|P| \times b$ 矩阵，其行对于每个 $\boldsymbol{p} \in P$ 为 $\Phi^\intercal(\boldsymbol{p})$。基函数的选择通常是为了获得所需的协方差。平稳核的一个常见选择是随机傅里叶特征 [Rahimi and Recht 2007]。Seyb et al. [2024] 认为权重空间方法在很大程度上仅限于平稳过程。我们表明这并非如此，并且图形学中的一些程序噪声技术可以被视为权重空间 GP。
A.2 条件化
高斯过程的一个强大特性是它们可以基于一组观测值 (C, v) 进行条件化，其中 C 是输入域上的一组点，v 是这些位置处的观测值，每个观测值带有高斯不确定性参数 σ 以量化模型预测中的不确定性。在高斯过程隐式表面的应用中，不涉及预测任务，因此 σ 通常设为 0。

函数空间。在观测 (C, v) 后，评估点集 P 处的“后验”分布也是高斯分布，其均值和协方差为：
$$
\mu_{|C}(P) = \mu(P) + \kappa(P, C)[\kappa(C, C) + \sigma^2\mathbf{I}]^{-1}(v - \mu(C)),
$$
$$
\kappa_{|C}(P, P) = \kappa(P, P) - \kappa(P, C)\kappa(C, C)^{-1}\kappa(C, P). \tag{31}
$$
形成此后验均值和协方差需要 $O(|C|^3 + |C||P| + |P|^2)$，然后通过代入公式 (29) 进行采样需要 $O(|P|^3)$。

权重空间。在权重空间中，我们可以通过修改公式 (30) 中的权重分布进行条件化，得到 $\mathbf{w}_{|C} \sim \mathcal{N}(\mathbf{m}_{|C}, \Sigma_{|C})$，其中
$$
\mathbf{m}_{|C} = (\Phi^\top(C)\Phi(C) + \sigma^2\mathbf{I})^{-1}\Phi^\top(C)v \tag{32}
$$
$$
\Sigma_{|C} = (\Phi^\top(C)\Phi(C) + \sigma^2\mathbf{I})^{-1}\sigma^2.
$$
其复杂度为 $O(b^3 + b|C|)$。

B 频域正定性证明
为简化起见，假设核函数 $h(s, p) = h(p - s)$ 是平移不变/平稳的，
$$
\psi_{\text{dense}}(p) := \int_{\mathbb{R}^d} h(p - s) W(s) \, ds, \quad \text{且} \tag{33}
$$
$$
\kappa(p, p') = \sigma^2 \int h(p - s) h(p' - s) \, ds, \tag{34}
$$
则可通过傅里叶变换轻松证明公式 (34) 是有效协方差。根据 Bochner 定理，函数 $k$ 是正半定的当且仅当其傅里叶变换 $\mathcal{F}\{k\}$ 是非负函数。我们认识到公式 (34)（忽略比例因子 $\sigma^2$）是 $h$ 的自相关：$(h \star h)(p - p')$。根据卷积 - 乘积定理，有
$$
\mathcal{F}\{\kappa\} = \sigma^2 \mathcal{F}\{h \star h\} = \sigma^2 \mathcal{F}\{h\} \overline{\mathcal{F}\{h\}} = \sigma^2 |\mathcal{F}\{h\}|^2 \geq 0, \tag{35}
$$
其中 $\overline{\mathcal{F}}$ 表示复共轭。公式 (35) 表明 $h$ 的功率谱（必须非负）给出了协方差 $\kappa$ 的傅里叶变换，因此 $\kappa$ 必须是正半定的。图 19 展示了二维 Gabor 核的讨论量可视化。可通过使用局部中心傅里叶变换将此证明扩展到非平稳版本 (4)。

C 稀疏卷积噪声的协方差
将公式 (9) 代入协方差定义并展开得
$$
\kappa_\lambda(p, p') = \mathbb{E}[\psi_\lambda(p) \psi_\lambda(p')] \tag{36a}
$$
$$
= \mathbb{E}\left[\left(\sum_{i=1}^N w_i h(s_i, p)\right)\left(\sum_{i=1}^N w_i h(s_i, p')\right)\right] \tag{36b}
$$
$$
= \mathbb{E}[N]\mathbb{E}[W_\lambda^2]\mathbb{E}[h(s, p)h(s, p')] \tag{36c}
$$
$$
= \underbrace{\lambda \mathbb{E}[W_\lambda^2]}_{\sigma^2} \int h(s, p) h(s, p') \, ds. \tag{36d}
$$
从公式 (36b) 到 (36c)，交叉项在期望中消失（由 Campbell 定理），并通过假设域体积为 1（不失一般性）得到公式 (36d)。

![第 18 页](tmp/single_19c94692a9e3d2dde6b8c76c9177474d/18.png)
