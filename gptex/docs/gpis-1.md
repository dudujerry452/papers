---
title: GPIS详解之（一）
date: 2026-09-09 16:24:00
tags:
- gpis
- 高斯过程
- 图形学
- CG
---

> 2026-08-01 20:43:51 发布
>
> 2026-09-09 16:24:00 修订

# 基础

## 定义

**高斯过程**(gaussian process, GP), 是无限维的高斯分布. 记为
$$
f(x)\sim GP(\mu, \kappa)
$$
其中, $\mu$ 是均值, $\kappa$是**协方差核**(Covariance Kernel)或称协方差函数. 相比于一般的有限维高斯分布: 
$$
[X_1, X_2, ..., X_n]\sim \mathcal{N}(\mu, \Sigma)
$$
GP包含了无限个随机变量, 即$n\rarr \infty$, 以至于构成了一条连续的函数曲线$f(x)$. 

-  $f(x)$是一个随机函数, 即: 对确定的$x$, $f(x)$是一个随机变量. 对样本空间$\Omega$及$\omega \in \Omega$, $f(x)$也可以写成$f(x, \omega)$. 
- $f(x)$的定义域被称为**索引集**(Index Set), 记为$\mathcal{X}$, 可以设定为多种空间, 比如$\mathbb{R}^3$. 其定义为$f:\mathcal{X}\times \Omega\rarr \mathbb{R}$, 其中$\Omega$为样本空间. 
-  协方差核$\kappa$也是一个函数, 对标多维分布的协方差矩阵$\Sigma$, 只不过是连续的而非离散的. 其定义为$\kappa : \mathcal{X} \times \mathcal{X} \rarr \mathbb{R}$. 

如果说二维高斯分布是摇一次骰子, 吐出两个值, 那么高斯过程就是摇动一次骰子吐出一条完整的函数曲线(无数个值). 

---

那么, 无穷维的分布是如何被我们认识的呢? 

我们在$\mathcal{X}$上任取有限的$n$个确定点$x_1, x_2, ..., x_n$构成向量$X$(如未特殊说明, 向量指列向量). 将每个分量代入$f$中, 得到的$n$个结果形成的向量记为$f(X)$. 我们发现, $f(X)$形成了一个$n$维高斯分布: 
$$
f(X) \sim \mathcal{N}(\mu, \kappa(X,X))
$$
这里, 稍微滥用记号, 令$\kappa(X, X)$是一个$n\times n$的矩阵, $\kappa(X, X)_{i,j} = \kappa(x_i, x_j)$. 

这就是说, 尽管我们不能一次性观察无穷个点上随机变量的取值, 但我们可以选取其中有限个点来观察GP在这些点上的取值规律, 也就是服从$n$维高斯分布. 

---

除此之外, GP还拥有一个最重要的性质: 它可以仅通过一阶矩(均值)和二阶矩(方差)来完整描述(Isserlis定理, Wick定理). 因此, 我们对GP(以及有限维高斯变量)的一切线性变换都等价于对这两个矩的操作, 这一点将在后面详述. 

## 复习: 联合分布

首先, 一般的, 对两个服从多维高斯分布的随机向量$A \sim \mathcal{N}(\mu_A, \Sigma_{AA}), B \sim \mathcal{N}(\mu_B, \Sigma_{BB})$, 其联合分布为
$$
\begin{bmatrix} 
A \\ 
B
\end{bmatrix}
\sim 
\mathcal{N}(
\begin{bmatrix}
\mu_A \\ 
\mu_B 
\end{bmatrix}, 
\begin{bmatrix}
\Sigma_{AA}, \Sigma_{AB} \\ 
\Sigma_{BA}, \Sigma_{BB}  
\end{bmatrix})
$$
其中$\Sigma_{AB}$为互协方差矩阵: 
$$
\Sigma_{AB} = \mathbb{E}[(A-\mu_A)(B-\mu_B)^T] = \Sigma_{BA}^T
$$
若从GP视角观看, 并将$A$看作$f(X)$, $B$看作$f(Y)$, 则
$$
\begin{bmatrix} 
f(X) \\ 
f(Y)
\end{bmatrix}
\sim 
\mathcal{N}(
\begin{bmatrix}
\mu_X \\ 
\mu_Y 
\end{bmatrix}, 
\begin{bmatrix}
\kappa(X, X), \kappa(X, Y) \\ 
\kappa(Y, X), \kappa(Y,Y) 
\end{bmatrix})
$$

## 复习: 条件分布

首先, 一般的, 对两个服从多维高斯分布的随机向量$A \sim \mathcal{N}(\mu_A, \Sigma_{AA}), B \sim \mathcal{N}(\mu_B, \Sigma_{BB})$, 以及确定向量$a$, 我们推导条件分布$\mathbb{P}(B|A = a)$. 

我们先提取出$B$中与$A$无关的部分$Z$: 
$$
Z = B - XA \\ 
$$
令$Cov(Z, A) = 0$, 继续推导: 
$$
Cov(B-XA, A) = \bf{0} \\
Cov(B, A) - Cov(XA, A) = \bf{0}\\ 
\Sigma_{BA} - X\Sigma_{AA} = 0 \\ 
X = \Sigma_{BA}\Sigma_{AA}^{-1}
$$
这样, $B$就被分解为$Z+XA$. 进而, $\mathbb{P}(B|A=a)=\mathbb{P}(Z+Xa)$. 因为$A, B$是高斯的, 故$Z+Xa$也是高斯的, 我们只需要求出$Z+Xa$的均值和方差即可. 我们来计算$Z$的均值和方差: 
$$
\mu_Z = \mathbb{E}(B-XA) = \mu_B - X\mu_A \\ 
$$

$$
\begin{align} 
\Sigma_Z &= Cov(B-XA, B-XA) \\ 
&= Cov(B, B-XA) - Cov(XA, B-XA) \\ 
&= Cov(B, B) - Cov(B, A)X^T - XCov(A, B) + XCov(A, A)X^T \\ 
\end{align}
$$



代入$X$发现后三项相等, 得
$$
\Sigma_Z = \Sigma_{BB} - \Sigma_{BA}\Sigma_{AA}^{-1}\Sigma_{AB}
$$
将求出的值代入, 求出$Z+Xa$的均值和方差: 
$$
\mu = \mu_B + X(a-\mu_A) \\ 
\Sigma = \Sigma_Z 
$$

# 性质

## 子域不变性

设$f(x)\sim GP(\mu, \kappa)$, 其中$\kappa:\mathcal{X}\times \mathcal{X}\rarr \mathbb{R}$. 现取$\mathcal{X'}\subset \mathcal{X}$, 设$f$在子域上的新过程为$f|_\mathcal{X'}$, 那么它仍然是高斯过程: 
$$
f|_\mathcal{X'}(x) \sim GP(\mu|_\mathcal{X'}, \kappa|_\mathcal{X'})
$$
其中, $\kappa': \mathcal{X'}\times \mathcal{X'}\rarr \mathbb{R}$. 这就是说, 对索引集取任意的子集, 其上的过程仍是同分布的高斯过程. 这一点从联合分布可以看出: $A, B$都是同一个高斯过程采样有限点得到的同分布多维随机变量, 组合起来仍然是该高斯过程采样有限点得到的多维高斯, 那么去掉$B$也一样是. 

例如, $\mathcal{X} = \mathbb{R}^3$, 此时索引集为三维空间. 取一条直线$\mathcal{X'} = \{\bold{x}+t\omega | \omega\in \mathbb{R}\}$, $\mathcal{X'}\subset \mathcal{X}$, 那么在直线上任取点仍然满足同样的高斯分布! 

## 二阶矩封闭性

高斯过程的统计特性完全由其一阶矩(均值函数)和二阶矩(协方差函数)唯一确定. 只要确定了这两个矩, GP的概率结构就完全确定了. 

设$f(x)\sim GP(\mu, \kappa)$, 对于$n$阶矩, 取$X = [x_1, x_2, ..., x_n]^T$, 根据Wick's Theorem: 
$$
\begin{align} 
&\mathbb{E}[(f(x_1)-\mu_1)(f(x_2)-\mu_2)...(f(x_n)-\mu_n)] \\

&= \int_{-\infty}^{\infty}...\int_{-\infty}^{\infty}(f(x_1)-\mu_1)(f(x_2)-\mu_2)...(f(x_n)-\mu_n)f(X)dx_1...dx_n \\ 

&= \sum_{p\in P_n}\prod_{(i,j)\in p}\kappa(x_i, x_j)
\end{align}
$$
其中, $P_n$是将$n$个数所有的两两配对之集合. 当$n$为奇数时, 因为不存在合法的两两配对, 因此直接为0. 当$n$为偶数时, $n$阶中心矩也可以由二阶中心距$\kappa$直接导出. 

## 半正定性

概率不能为负, 方差也不能为负. 因此, 对GP采样出的任意一组随机变量进行任意线性组合, 得到的新随机变量的方差都必须是正的. 

设$f(x)\sim GP(\mu, \kappa)$, 任取$X = [x_1, x_2, ..., x_n]^T$, 从GP取一组随机变量为$f(X)$, 其均值为$\mu(X)$. 那么取任意系数$C = [c_1, c_2, ..., c_n]^T$, 进行线性组合: 
$$
x = C^Tf(X) = c_1f(x_1) + c_2f(x_2) + ... + c_nf(x_n)
$$
计算新随机变量的方差: 
$$
\begin{align}
Var(x) &= \mathbb{E}[(x - \mu(x))^2]  \\ 
&= \mathbb{E}[(C^Tf(X) - C^T\mu(X))^2] \\ 
&= \mathbb{E}[(C^T(f(X) - \mu(X))^2] \\ 
&= \mathbb{E}[(C^T(f(X) - \mu(X))(C^T(f(X) - \mu(X))^T] \\ 
&= \mathbb{E}[C^T(f(X) - \mu(X)(f(X) - \mu(X)^TC] \\ 
&= C^T\mathbb{E}[(f(X) - \mu(X)(f(X) - \mu(X)^T]C \\ 
&= C^T\kappa(X, X)C
\end{align}
$$
要让$Var(x)\geq 0$, 就要让$C^T\kappa(X, X)C\geq 0$, 发现这正是半正定矩阵的定义! 

# 计算

## 直接采样

采样GP时, 先选出采样点, 得到$A\sim \mathcal{N}(\mu, \Sigma)$, 问题变为采样多维高斯. 因为计算机只能采样$\eta \sim \mathcal{N}(0, 1)$, 那我们就用$\eta$构造出随机向量$A$. 

首先, 需要满足: 
$$
Cov(A, A) = \Sigma
$$
若对协方差做Chomskey分解, 使得 $LL^T = \Sigma$, 可令 $A = \mu + L\eta$, 有
$$
\begin{align}
Cov(\mu + L\eta, \mu + L\eta) &= Cov(L\eta, L\eta) \\ 
&= LCov(\eta, \eta)L^T \\
&= LL^T \\
&= \Sigma
\end{align}
$$
此时, $A$就是采样出来的多维高斯. 

因为Chomskey分解, **直接采样$n$个点, 时间复杂度为$O(n^3)$**. GPIS中, 我们需要骰出整个空间的GP场, 但不能采用对空间格点采样, 这是因为若每个维度有$m$个格子, 即$m^3$个采样点, 则时间复杂度就是$O((m^3)^3) = O(m^9)$. 

但由[子域不变性](#子域不变性), 假设我们采样GP场时使用ray marching, 那么即使只采样光线上的点, 我们也可以得到与3D空间完全一致的分布. 并且, 由于GP是光滑的, 所以我们可以在一条直线上采样$m$个点, 并在$m$个点之间进行插值, 从而计算0-level. 如果发生了散射, 我们只需要将之前的采样点作为条件, 计算条件分布下新的协方差矩阵即可. 

然而, 我们不可能将每一条光线的所有采样点都记住(那样很显然也会超复杂度), 所以我们有如下策略: 

1. Renewal, 完全不进行任何条件化. 光线之间, 每段光线之间都无记忆. 相当于每段光线对应一个single realization; 最后的结果是ensemble average; 
2. Renewal+, 只记录上一段光线的采样点. 光线之间无记忆, 但每段光线将上一段光线的所有采样点记忆化, 这样相邻的光线段共享一个realization, 更接近每条光线对应realization; 
3. Global-n, 记录整条光线的采样点. 光线之间无记忆, 但同一条光线中新的采样点会条件化该光线中所有之前的采样点, 即每条光线对应一个realization.

这也是论文[^1]的的主要idea. 然而我们注意到, 它无法实现整体的single realization, 只能约束单条光线. 这也是为什么论文[^2]开发了稀疏卷积方法. 

[^1]: Dario Seyb, Eugene d'Eon, Benedikt Bitterli, and Wojciech Jarosz. 2024. From microfacets to participating media: A unified theory of light transport with stochastic geometry. *ACM Trans. Graph.* 43, 4, Article 112 (July 2024), 17 pages. doi.org
[^2]: Kehan Xu, Benedikt Bitterli, Eugene d'Eon, and Wojciech Jarosz. 2025. Practical Gaussian Process Implicit Surfaces with Sparse Convolutions. *ACM Trans. Graph.* 44, 6, Article 188 (December 2025), 18 pages. doi.org

## 稀疏卷积模拟

*接下来, 为方便推导, 符号上我们一般取$\psi(x) \sim GP(0, \kappa)$.* 

### 定义

无论是计算先验分布还是条件分布, 都要计算协方差chomskey分解, 复杂度非常高. 是否可以在给定GP的情况下, 快速计算其某一个实例对应的标量场$\psi(p)$? 答案是肯定的. 

首先, 定义白噪声高斯过程
$$
W(s) \sim GP(0, \kappa_w)
$$
$\kappa_w(p,q) = \sigma^2\delta(p-q)$, $\delta$为狄拉克函数, 满足: 
$$
\delta(x) = 0, x\neq 0 \\ 
\int_{-\infty}^{\infty} \delta(x)dx = 1
$$
$W$的GP空间中两个点互不相关, 每个点都是一个独立的正态分布: 对任意位置$s$, 有$W(s) \sim \mathcal{N}(0, \sigma^2)$. 

接下来我们借助$W$来构造新的GP场. 取$W$的一个实例$W_d$, 给定核函数为$h$,我们希望构造 $\psi \sim GP(0, \kappa)$  的一个实例$\psi_d$ . 做如下尝试: 
$$
\psi_d(p) = \int_{\mathbb{R}^3}h(s, p)W_d(s)ds
$$
则根据定义, 
$$
\begin{align}
\kappa(p, q) &= Cov(\psi_d(p), \psi_d(q)) \\ 
&= \mathbb{E}(\psi_d(p)\psi_d(q)) \\ 
&= \mathbb{E}(\int_{\mathbb{R}^3}h(s, p)W_d(s)ds\int_{\mathbb{R}^3}h(u, q)W_d(u)du) \\
&= \mathbb{E}(\int_{\mathbb{R}^3}\int_{\mathbb{R}^3}h(s, p)h(u, q)W_d(s)W_d(u)dsdu) \\
&=  \int_{\mathbb{R}^3}\int_{\mathbb{R}^3}h(s, p)h(u, q)\mathbb{E}(W_d(s)W_d(u))dsdu \\ 
&= \int_{\mathbb{R}^3}\int_{\mathbb{R}^3}h(s, p)h(u, q)\sigma^2\delta(s-u)dsdu \\ 
&= \sigma^2 \int_{\mathbb{R}^3}\int_{\mathbb{R}^3}h(s, p)h(u, q)\delta(s-u)dsdu  \\ 
&= \sigma^2 \int_{\mathbb{R}^3}h(s, p)h(s, q)ds  \\ 
\end{align}
$$
这也就表明, 两个位置的相关性是以其为中心的核的重叠部分. 

> 例1: 写出平方指数核$h(s,p) = exp(-\frac{\|p-s\|^2}{L^2})$ 导出的GP的对应的$\kappa$. 
>
> 解: 
> $$
> \begin{align}
> \kappa(p, q) &= \sigma^2 \int_{\mathbb{R}^3}h(s, p)h(s, q)ds \\ 
> &= \sigma^2 \int_{\mathbb{R}^3}exp(-\frac{\|p-s\|^2+\|q-s\|^2}{L^2})ds
> \end{align}
> $$
> 其中, $\|p-s\|^2+\|q-s\|^2$可化简为
> $$
> 2\|s-\frac{p+q}{2}\|^2 + \frac{1}{2}\|p-q\|^2
> $$
> 代回原式
> $$
> \begin{align}
> \kappa(p, q) &= \sigma^2\int_{\mathbb{R}^3}exp(-\frac{2\|s-\frac{p+q}{2}\|^2 + \frac{1}{2}\|p-q\|^2}{L^2})ds \\
> &= \sigma^2exp(\frac{\|p-q\|^2}{2L^2})\int_{\mathbb{R}^3}exp(-\frac{2\|s-\frac{p+q}{2}\|^2}{L^2})ds
> \end{align}
> $$
> 作换元$u = s - \frac{p+q}{2}$, 对式子后半部分
> $$
> \begin{align}
> &\int_{\mathbb{R}^3}exp(-\frac{2\|s-\frac{p+q}{2}\|^2}{L^2})ds \\ 
> &=\int_{\mathbb{R}^3}exp(-\frac{2\|u\|^2}{L^2})du \\ 
> &= (\frac{\pi L^2}{2})^{1.5}
> \end{align}
> $$
> 最后一步来自于将三维的积分拆开求解. 最终, 我们得到
> $$
> \kappa(p,q) = \sigma^2(\frac{\pi L^2}{2})^{1.5}exp(\frac{\|p-q\|^2}{2L^2})
> $$
> 这个式子分为两部分, 即常系数和衰减函数. 有时候我们只想保留衰减函数的部分, 于是我们就除去常系数部分(并把$\sigma$设为1). 这一点在[离散化](#离散卷积)部分亦有记载. 

### 半正定性

接下来证明 $\kappa$ 是半正定的: 对空间中任意$n$个点 $\{p_1, p_2, ..., p_n\}$ 构成的协方差矩阵 $K$ , 和空间中任意选取的非零向量$q$, 满足
$$
q^TKq \geq 0
$$
继续推导
$$
\begin{align}
&q^TKq \\
&=\sum_{i=1}^{n}q_i\sum_{j=1}^nq_jK_{j,i} \\
&=\sum_{i=1}^n\sum_{j=1}^nq_iq_jK_{i,j}  \\ 
&=\sum_{i=1}^n\sum_{j=1}^nq_iq_j\kappa(p_i, p_j)  \\
\end{align}
$$
代入 $\kappa(p, q) = \sigma^2 \int_{\mathbb{R}^3}h(s, p)h(s, q)ds$
$$
\begin{align}
&=\sum_{i=1}^n\sum_{j=1}^nq_iq_j\sigma^2 \int_{\mathbb{R}^3}h(s, p_i)h(s, p_j)ds \\
&=\sigma^2 \int_{\mathbb{R}^3}\sum_{i=1}^n\sum_{j=1}^nq_iq_jh(s, p_i)h(s, p_j)ds \\ 
&=\sigma^2 \int_{\mathbb{R}^3}(\sum_{i=1}^nq_ih(s, p_i))(\sum_{j=1}^np_jh(s, p_j))ds \\
&=\sigma^2 \int_{\mathbb{R}^3}(\sum_{i=1}^nq_ih(s, p_i))^2ds \geq 0\\
\end{align}
$$
因此,$q^TKq\geq 0$. 

### 离散卷积

刚才我们推导了连续的卷积分布, 我们下面证明稀疏卷积对应的协方差核与连续版本一致: 
$$
W_\lambda(s) = \sum_iw_i\delta(s - s_i) \\ 
$$
其中, $s_i$采样于密度为$\lambda$的泊松点过程, $w_i\sim \mathcal{N}(0, \sigma^2/\lambda)$. 将其带入$\psi_d(p)$: 
$$
\begin{align}
\psi_\lambda(p) &= \int_{\mathbb{R}^3}W_\lambda(s)h(s,p)ds \\ 
&= \sum_i w_ih(s_i, p)
\end{align}
$$
接下来证明 $\lim_{\lambda\rightarrow\infty}\psi_\lambda$  在协方差函数上与 $\psi_d$ 等价:
$$
\begin{align}
\kappa_\lambda(p, q) &= Cov(\psi_\lambda(p), \psi_\lambda(q)) \\ 
&= \mathbb{E}((\sum_i w_ih(s_i, p))\cdot(\sum_j w_jh(s_j, q))) \\ 
&= \mathbb{E}(\sum_i\sum_jw_iw_jh(s_i, p)h(s_j, q)) \\ 
&= \sum_i\sum_j\mathbb{E}(w_iw_jh(s_i, p)h(s_j, q)) \\ 
&= \sum_i\sum_j\mathbb{E}(w_iw_j)\mathbb{E}(h(s_i, p)h(s_j, q)) \\ 
\end{align}
$$
 接下来分类讨论 $i = j, i \neq j$. 

$i \neq j$ 时, 此时$w_i$, $w_j$独立, 
$$
\mathbb{E}(w_iw_j) = \mathbb{E}(w_i)\mathbb{E}(w_j) = 0\times 0 = 0
$$
因此这些项消掉. 

$i = j$ 时, 退化为单重求和
$$
\begin{align}
\kappa_\lambda(p, q) &= \sum_i\mathbb{E}(w_iw_i)\mathbb{E}(h(s_i,p)h(s_i,q)) \\ 
&=\sum_i(Var(w_i) + \mathbb{E}^2(w_i))\mathbb{E}[h(s_i, p)h(s_i,q)] \\ 
&=\frac{\sigma^2}{\lambda}\sum_i\mathbb{E}[h(s_i, p)h(s_i,q)] \\ 
&=\frac{\sigma^2}{\lambda}\mathbb{E}[\sum_i(h(s_i, p)h(s_i,q)] \\ 
\end{align}
$$
根据Campbell's Theorem, 对任意密度为 $\lambda$ 的泊松点过程的函数 $g$ , 有
$$
\mathbb{E}[\sum_ig(s_i)] = \lambda\int_{\mathbb{R}^3}g(s)ds
$$
应用到 $\kappa_\lambda$ , 有
$$
\begin{align}
\kappa_\lambda(p, q)  &= \frac{\sigma^2}{\lambda}\lambda\int_{\mathbb{R}^3}h(s,p)h(s,q)ds  \\ 
&= \sigma^2\int_{\mathbb{R}^3}h(s, p)h(s,q)ds
\end{align}
$$
代入后, 发现与连续卷积等价. 这也就启示我们, 可以在空间中泊松随机地撒下多个脉冲$s_1, s_2, ...$, 并计算它们对$p$的影响$\sum_i w_ih(s_i, p)$, 从而计算出GP场函数[^2]. 这给了我们计算single relization的可能性, 并且在计算ensemble average时, 只需要在每次采样时再次随机, 更换脉冲位置即可. 

> 实际工程中我们常设$\sigma=1$, 并使用密度为$\lambda$的离散撒点(卷积)方法得到GP场函数$\psi$, 此时$\psi$对应的GP的协方差函数为
> $$
> \kappa(p,q) = \lambda\int_{\mathbb{R}^3}h(s, p)h(s,q)ds
> $$
> 为了处理成更核心的形式, 以例1为例, 我们设:
> $$
> \psi'(p,q) = (\frac{\pi L^2}{2})^{-1.5}\lambda^{-1}\psi(p,q)
> $$
> 这样, 无论$L$和$\lambda$, 我们总有
> $$
> \psi'(p, q) = exp(\frac{\|p-q\|^2}{2L^2})
> $$
> 我们便可以毫无顾忌地设置$L$和点密度$\lambda$, 而不用担心它对GP场造成额外的影响.

### 条件化

给定一系列约束点$C = [c_1, c_2, ..., c_n]$ 以及其对应的值 $\mathbf{v} = [v_1, v_2, ..., v_n]$ 要求对先验分布$\psi$求出后验分布$\psi_p$使得
$$
\forall i \in \{1, 2, ..., n\}, \quad \psi_p|_{C}(c_i) = v_i
$$
那么我们在$\psi$的基础上加上修正项得到后验分布$\psi_p$ : 
$$
\psi_p(x) = \psi(x) + \sum_{i=1}^n\kappa(x, c_i)u_i
$$
要让最初的约束成立, 修正参数$\mathbf{u}$就要满足
$$
\kappa(C, C)\mathbf{u} = \mathbf{v} - \psi(C)
$$
因此, 我们只需要解出$n\times n$的线性系统即可进行条件化. 相比函数空间的chomsky求解$O(n^3)$, 代价很小. 





