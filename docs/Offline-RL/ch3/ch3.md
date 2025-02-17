# 基于正则化的方法与CQL

正则化是一个很好调节模型的工具。对于策略的正则化，策略梯度的目标函数可为式(1)。

$$
\begin{equation}
J(\theta)=\mathbb{E}_{s,a\sim\mathcal{D}}[\mathcal{Q}^{\pi_{\theta}}(s,a)]+\mathcal{R}(\theta)\tag{1}
\end{equation}
$$
对于值函数的正则化，目标函数可为式(2)。
$$
\begin{equation}
J(\phi)=\mathbb{E}_{s,a,{s}'\sim\mathcal{D}}[(r(s,a)+\gamma\mathbb{E}_{{a}'\sim\pi_{off}(.\vert s)}[\mathcal{Q}_{\phi}^{\pi}({s}',{a}')]-\mathcal{Q}^{\pi}_{\phi}(s,a))^2]+\mathcal{R}(\phi)\tag{2}
\end{equation}
$$
式(1)和(2)中$\mathcal{R}(\theta)$表示的是正则化项。

正则化可以调节模型的形状或参数，但是不能限制策略$\pi_{\theta}$与$\pi_{\beta}$的距离。因此，基于正则化的离线强化学习算法，需要其它方法限制策略，例如：保守型模型、策略约束。



## Conservative Q-Learning

为了解决由智能体学到的策略$\pi$与数据产生的行为策略$\pi_{\beta}$之间分布偏移产生的价值高估问题，[CQL](https://arxiv.org/abs/2006.04779 "CQL")算法学习出保守型Q函数，使Q函数的期望值为策略$\pi$下价值函数$V(s)$真实值的下界。在理论上，证明了CQL的确产生了价值函数$V(s)$真实值的下界，且该算法可应用到策略学习迭代步骤中。

为了公式的简洁性，利用$\mathcal{B}^*Q(s,a)$表示贝尔曼最优运算，可见式(3)
$$
\mathcal{B}^*Q(s,a)=r(s,a)+\gamma\mathbb{E}_{{s}'\sim P({s}'|s,a)}[max_{{a}'}Q({s}',{a}')]\tag{3}
$$
利用$\mathcal{B}^{\pi}Q$表示策略$\pi$下的贝尔曼运算，可见式(4)
$$
\mathcal{B}^{\pi}Q=r + \gamma \mathbb{E}_{{s}'\sim T({s}'\vert s,a),{a}'\sim\pi({a}'\vert{s}')}Q({s}',{a}')\tag{4}
$$

同时，在CQL中行为策略$\pi_{\beta}$为数据统计的结果，计算公式为$\hat{\pi}_{\beta}:=\frac{\sum_{s,a\in\mathcal{D}}1[s=s,a=a]}{\sum_{s\in\mathcal{D}}1[s=s]}$

### Conservative Off-Policy Evaluation

CQL主要考虑的是防止策略$\pi$下价值函数$V^{\pi}(s,a)$的高估问题。因此，在最小化贝尔曼误差的同时，也最小化$Q(s,a)$的值，可见式(5)。
$$
\begin{equation}
\hat{Q}^{k+1}\leftarrow \underset{Q}{argmin}\quad\alpha\mathbb{E}_{s\sim\mathcal{D},a\sim{\mu(s,a)}}[Q(s,a)]+\frac{1}{2}\mathbb{E}_{s,a\sim\mathcal{D}}[(Q(s,a)-\hat{\mathcal{B}}^{\pi}\hat{Q}^k(s,a))^2]\tag{5}
\end{equation}
$$
由于标准的$Q(s,a)$函数的训练，不会查询未观测过的状态，但会查询未观测过的动作。所以，限制$\mu$与数据集中状态边缘分布匹配，以便于$\mu(s,a)=d^{\pi_{\beta}}(s)\mu(a\vert s)$成立。其中，$d^{\pi_{\beta}}(s)$为策略$\pi_{\beta}$下的状态$s$的折扣边缘分布。具体的限制方法，可见下文。

文献[2]表明，对于$\forall(s,a)$，$\hat{Q}^{\pi}:=lim_{k\to\infty}\hat{Q}(k)$为$Q^{\pi}$的下界。然而，CQL更关注的是价值函数$V^{\pi}(s)$的估计，那么可进一步加紧下界。为了使策略$\pi(a|s)$下$\hat{Q}(\pi)$的期望值为$V^{\pi}$的下界，文献[2]引入数据集的策略$\pi_{\beta}(a\vert s)$下的$Q$函数最大化项，可见式(6)。

$$
\begin{aligned}
\hat{Q}^{k+1}\leftarrow \underset{Q}{argmin}\quad \alpha(\mathbb{E}_{s\sim\mathcal{D},a\sim\mu(a\vert s)}[Q(s,a)]-\mathbb{E}_{s\sim\mathcal{D},a\sim\hat{\pi}_{\beta}(s,a)}[Q(s,a)]) \\
+\frac{1}{2}\mathbb{E}_{s,a,{s}'\sim\mathcal{D}}[(Q(s,a)-\hat{\mathcal{B}}^{\pi}\hat{Q}^k(s,a))^2]
\end{aligned}\tag{6}
$$

根据文献[2]中理论，可知，由式(6)产生的$\hat{Q}^{\pi}$不一定能够对于$\forall(s,a)$均为$Q^{\pi}$的下界。但是，$\hat{Q}^{\pi}$的均值一定为价值函数$V^{\pi}$的下界，可以理解为：若$\mu(a\vert s)=\pi(a\vert s)$成立，则$\mathbb{E}_{\pi(a\vert s)}[\hat{Q}^{\pi}(s,a)]\le V^{\pi}(s)$。

直觉上来说，式(6)中最大化策略$\hat{\pi}_{\beta}$下的$Q$值，那么策略$\hat{\pi}_{\beta}$下动作的$Q$值最可能被高估，即数据集中出现过的动作才有可能会被高估，因此$\hat{Q}^{\pi}$处处小于$Q^{\pi}$未必成立。




### Conservative Q-Learning算法

根据文献[2]，可知，求解式(6)可得策略$\pi$价值的下界，那么就会降低策略高估的风险。因此，若利用式(6)作为策略的评估可得$\hat{Q}^{\pi}$，然后再进行一步的策略提升，那么就可得$V^{\pi}$的下界。然而，这种计算成本很高。另一种方法是使$u(\mathbf{a}\vert\mathbf{s})$近似策略$\hat{\pi}^k$，该策略$\hat{\pi}^k$是通过最大化当前迭代$Q$函数获得的，这种方式更接近线上强化学习，可见式(7)。这类优化问题的实例可被称为$CQL(\mathcal{R})$。

$$
\begin{aligned}
\underset{Q}{min}\underset{\mu}{max}\quad\alpha(\mathbb{E}_{s\sim\mathcal{D},a\sim\mu(a\vert s)}[Q(s,a)]-\mathbb{E}_{s\sim\mathcal{D},a\sim\hat{\pi}_{\beta}(a\vert s)}[Q(s,a)]) \\
+\frac{1}{2}\mathbb{E}_{s,a,{s}'\sim\mathcal{D}}[(Q(s,a)-\hat{\mathcal{B}}^{\pi_k}\hat{Q}^k(s,a))^2]+\mathcal{R}(\mu)
\end{aligned}\tag{7}
$$

式(7)中$\mathcal{R}(\mu)$为正则化器。

**CQL的变体：** 若利用先验分布为均匀分布的$\rho(a\vert s)$与策略$\mu$之间KL-Divergence距离作为正则化器$\mathcal{R}(\mu)$，那么$CQL(\mathcal{R})$可被实例化为式(8)。其中，$\mu(\mathbf{a}\vert\mathbf{s})$正比于$\rho(\mathbf{a}\vert\mathbf{s})\cdot exp(Q(\mathbf{s},\mathbf{a}))$
$$
\begin{equation}
\underset{Q}{min}\quad\alpha\mathbb{E}_{s\sim\mathcal{D}}[log\sum_a exp(Q(s,a))-\mathbb{E}_{a\sim\hat{\pi}_{\beta}(a\vert s)}[Q(s,a)]]+\frac{1}{2}\mathbb{E}_{s,a,{s}'\sim\mathcal{D}}[(Q-\hat{\mathcal{B}}^{\pi_k}\hat{Q}^k)^2]\tag{8}
\end{equation}
$$

若先验分布为之前策略$\hat{\pi}^{k-1}$，那么式(8)中第一项应该被替换为策略$\hat{\pi}^{k-1}(a\vert s)$下Q值的指数权重平均。


<div align="center">
  <img src="https://www.robotech.ink/usr/uploads/2024/02/1057244385.png" width=600 />
</div>
<div align="center">
  图1 CQL算法两种变体的伪代码
</div>

图1中式(4)为本文中式(8)。

## 参考文献

[1] Prudencio R F, Maximo M R O A, Colombini E L. A survey on offline reinforcement learning: Taxonomy, review, and open problems[J]. IEEE Transactions on Neural Networks and Learning Systems, 2023.

[2] Kumar A, Zhou A, Tucker G, et al. Conservative q-learning for offline reinforcement learning[J]. Advances in Neural Information Processing Systems, 2020, 33: 1179-1191.
