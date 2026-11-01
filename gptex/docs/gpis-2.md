---
title: GPIS详解之（二）
date: 2026-09-16 23:24:00
tags:
- gpis
- 高斯过程
- 图形学
- CG
---

# 前情提要

前一篇讲了如何通过函数空间采样和稀疏卷积采样GP. 接下来, 我们进入GPIS的"S"部分, 关注光线打到表面时的法线, 也就是梯度. 

如何采样梯度以及如何提供NEE的反射采样空间? 这就是我们将要讨论的问题. 

# 目标

设GP为$f(x) \sim GP(\mu, \kappa)$, 我们的目标是求出表面处($f(\bold{s}) = 0$)的梯度: 
$$
\nabla f(\bold{s}) = [\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z}]^T
$$
最直观的方法是我们可以在该点及其邻域确定一个实例, 即确定一个硬表面, 在该实例上对$x,y,z$三个方向(或者别的基底向量)分别计算微分. 一共需要采样6次函数值来做差分, 最终得到一个确定性法线. 

# 函数空间

对于函数空间采样方法, 为确定实例, 我们可以把历史点信息和上一个维度的偏微分作为条件, 采样出下一个维度的偏微分. 

**引理1**: 对GP应用任何线性算子后得到的仍然是GP.  

**引理2**: 对GP应用微分算子后得到的新的GP与原GP之间具有联合高斯分布. 

**引理3**: 具有联合高斯分布的随机变量组成的向量拥有稳定的协方差矩阵(称为交叉协方差). 

根据引理2,3,4, 导数可以与函数值同等地作为条件, 计算出包含这两种类型条件的协方差矩阵, 并用该矩阵条件化采样出新的函数值或导数. 实际上, 该矩阵的形式为
$$
K_{total} = \begin{bmatrix}
K_{ff} & K_{f\nabla} \\ 
K_{\nabla f} & K_{\nabla \nabla} 
\end{bmatrix}
$$
其中, $K_{ff}$为所有的历史点和当前点函数值协方差; $K_{f\nabla}$ 描述的是交叉协方差, $K_{\nabla \nabla}$是导数之间的协方差. 求出该矩阵后, 直接对其对应的分布抽样得到梯度. 

**引理4**: GP中, 若核函数是各向同性的, 那么在任意一点, 互相正交方向上的导数之协方差一定为0. 

根据引理4, 我们发现, 若核是各向同性的, 那么将射线方向$\bold{d}$作为一个基底, 求导时另外两个方向的偏导就与之无关, 因此条件化时还能省下一个梯度条件[^1]. 不过, 该优化的效果有限, 因为其余两个方向的偏导依然需要历史点的条件化来采样. 

[^1]: Dario Seyb, Eugene d'Eon, Benedikt Bitterli, and Wojciech Jarosz. 2024. From microfacets to participating media: A unified theory of light transport with stochastic geometry. *ACM Trans. Graph.* 43, 4, Article 112 (July 2024), 17 pages. doi.org
[^2]: Kehan Xu, Benedikt Bitterli, Eugene d'Eon, and Wojciech Jarosz. 2025. Practical Gaussian Process Implicit Surfaces with Sparse Convolutions. *ACM Trans. Graph.* 44, 6, Article 188 (December 2025), 18 pages. doi.org

# 稀疏卷积

对稀疏卷积而言, 我们通过条件化求出并固定了权重$w_i$, 从而直接确定了一个可以$O(1)$查询的硬表面. 此时我们直接通过差分求解梯度即可. 

但是这样只能获得一个确定性法线, 无法得到NDF, 也就不能进行NEE, 也就不支持specular材质. Xu意识到[^2]虽然稀疏卷积的特性能让我们快速获得一个条件化的实例, 但这不代表这个实例就是唯一的. 满足条件的有无数个, 若干个实例, 稀疏卷积只是抽出了其中一种! 

于是我们回到函数空间, 并采用Renewal模型, 根据引理4, 将梯度$\bold{g} = (g_x, g_y, g_z)^T$分解为沿着光线的$\bold{g}_z$和垂直于光线的$\bold{g}_{xy} = (g_x, g_y)^T$两个分量. 

**引理5**: GP中, 若核函数是各向同性的, 那么在任意一点, 其函数值与任意方向上的导数协方差一定为0. 

因为我们采用了Renewal模型, 此时条件只有一个沿着光线方向上的导数约束$\bold{g}_z$以及函数值约束$f(x)=0$, 而根据引理4, 5, 这两者恰好在数学上都不会影响$\bold{g}_{xy}$! 而根据引理1, 导数作为GP的一个线性变换, 那么$\bold{g}_{xy}$服从一个多元高斯分布. 并且, $\bold{g}_{xy}$不受条件影响, 这是一个先验高斯分布, 即: 
$$
p(\bold{g} | \zeta) = p(g_z | \zeta)p(\bold{g}_{xy}) \\
p(\bold{g}_{xy}) = \mathcal{N}(0, -\kappa''(0)I_2)
$$
其中, $\zeta$是$f(x)=0$以及$z ~of~\bold{g} = g_z$两个条件, $\kappa ''(0)$是各向同性核在距离为0时的二阶导, 同时也是梯度场的方差. 

于是, 我们在$xy$两个方向上有了采样的自由度. 在稀疏卷积求出的确定性的$g_z$的基础上, 我们通过调整$\bold{g}_{xy}$从而控制$\bold{g}$的方向. 对于光源与当前点的连线所确定的法线$\bold{g}$, 我们设好$g_z$后求出对应的$\bold{g}_{xy}$, $\bold{g}_{xy}$的PDF值作为NDF值, 从而实现了NEE. 
