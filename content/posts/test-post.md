+++
date = '2024-03-22T00:13:22+08:00'
title = 'test'
categories = ["test"]
description = "样式测试"
+++

假设用户的预算服从的分布的概率密度函数为$b(x_1)$，流量需求服从的分布的概率密度函数为$t(x_2)$，节点质量需求服从的概率密度函数为$q(x_3)$

假设用户对以上 3 种因素侧重比例的概率密度函数为$a(y_1,y_2)$，定义为：

+ $y_1\in(0,1)$,$y_2\in(0,1)$，$y_1+y_2<1$，否则$a(y_1,y_2)=0$
+ $y_1$为预算的权重，$y_2$为流量的权重，$1-y_1-y_2$为质量的权重

每种套餐都有价格、流量、质量 3 个属性。假设当用户思考买哪一种套餐时，会按照层次分析法决策。上述 $b$, $t$, $q$ 的参数需要参数估计。

令 $P=(p_{r1},p_{r2},p_{r3},\cdots,p_{rn})'$, $T=(t_1,t_2,t_3,\cdots,t_n)'$, $Q=(q_1,q_2,q_3,\cdots,q_n)'$ 为产品数组向量，分别为产品的价格、流量、质量。产品数组向量由产品性质决定，不需要参数估计。
```python
import torch

# 定义套餐的价格、流量、质量
packageNumber = 3
packagePrice = [1, 2, 3]
packageTrafficLimit = [1, 2, 3]
packageQuality = [1, 2, 3]
```
## 层次分析法量化

预算满意度，可以定义为$f(p_r,x_1)=10^{\displaystyle-\frac{p_r}{x_1}}$，其中$p_r$为套餐价格，$x_1$为预算

流量满意度，可以定义为$g(t,x_2)=\sqrt{\displaystyle\frac{4t}{x_2}-\frac{3}{4}}$（当根号下为 0 以下时定义为 0），其中$t$为套餐流量，$x_2$为套餐需求

节点质量满意度，可以定义为$h(q,x_3)=\sqrt[3]{\displaystyle\frac{q}{x_3}}$，其中$q$为节点质量量化指标，$x_3$为节点质量需求。

用户会选择满意度

$$
y_1\cdot10^{\displaystyle-\frac{p_r}{x_1}}+y_2\sqrt{\displaystyle\frac{4t}{x_2}-\frac{3}{4}} + y_3\sqrt[3]{\displaystyle\frac{q}{x_3}}
$$


$$
\text{SATISFY} = [f(P,x_1),g(T,x_2),h(Q,x_3)]'Y
$$

$$
SELECT=\begin{cases}
\max_{\text{SATISFY}} \text{PACKAGE},\ \text{SATISFY}>0.1\\
\varnothing,\ \text{SATISFY}\leq 0.1
\end{cases}
$$

```python
def user_satisfaction(vector_x, vector_y, pr, t, q):
    vector_x = torch.tensor(vector_x, dtype=torch.float32)
    vector_y = torch.tensor(vector_y, dtype=torch.float32)
    
    term1 = vector_y[:, 0] * 10 ** (-pr / vector_x[:, 0])
    term2 = vector_y[:, 1] * torch.sqrt(4 * t / vector_x[:, 1] - 0.75)
    term3 = vector_y[:, 2] * (q / vector_x[:, 2]) ** (1 / 3)
    
    return term1 + term2 + term3
```

### 用户预算分布

预算可以认为大致是服从帕累托分布的，假设初始参数为$x_1\sim Par(1,0.6)$，即

$$
b(x_1) = \frac{0.6}{x_1^{1.6}} (x_1>1)
$$

其中 0.6 需要参数估计以获得准确值。

帕累托分布通常用来描述不平等的分布，例如财富或收入，以及对应的预算。在现实世界中，财富分布往往是高度不平等的，少数人拥有大部分财富。因此，为了反映用户预算，使用帕累托分布应该是合适的。

此外，帕累托分布与所谓的“80/20 规则”密切相关，即 80%的效果（如消费）来自 20%的原因（如消费者）。在许多经济模型中，这种现象是普遍存在的，比如少数消费者贡献了大部分的消费支出。

帕累托分布的两个参数中，第一个参数为截止值，当$x_1<1$时，$b(x_1)=0$。截止值无法参数估计，但假设为 1 是合适的。

### 流量需求分布

流量需求可以认为近似服从正态分布，参数为$x_2\sim N(130,30^2)$，即

$$
t(x_2)=\frac{1}{40\sqrt{2\pi}}e^{-\displaystyle\frac{1}{2}\left(\frac{x_2-120}{40}\right)^2}
$$

这两个参数都需要参数估计。

### 质量需求分布

如果量化质量为$1\sim10$，其中 1 为完全灵车机场，10 为全 IPLC 多入口智能解析机场（性能接近游戏加速器），那么质量需求也可以认为是服从帕累托分布的，参数为$x_3\sim Par(1,2)$，即

$$
q(x_3)=\frac{2}{x_3^2}
$$

这个帕累托分布同样只有第二个参数需要参数估计。

### 权重分布

注意到$y_1$, $y_2$, $y_3$的分布实际上被均匀地放在了一个等边三角形上。这个等边三角形是一个三棱锥的底面。这个三棱锥的 3 个侧面都是直角边边上为 1 的直角三角形。

等边三角形的 3 个顶点分别表示其中一个权重为 1 而另外两个权重为 0 的情况。

在使用$(y_1,y_2)$表示时，3 种情况分别对应$(0,0)$, $(1,0)$, $(0,1)$。而为了让$y_1,y_2,y_3$变成更加均匀的等边三角形，需要将这个区域线性变换为$(-\sqrt{3}+1,0)$, $(\sqrt{3}-1,1)$, $(\sqrt{3}-1,-1)$。这个线性变换很容易求得。

首先把$(-\sqrt{3}+1,0)$和$(0,0)$对齐，那么其他两个点是$(\sqrt{3},1)$, $(\sqrt{3},-1)$。于是，变换矩阵为

$$
\begin{bmatrix}
\sqrt{3} & \sqrt{3} \\
1 & -1
\end{bmatrix}
\begin{bmatrix}
1 & 0 \\
0 & 1
\end{bmatrix}^{-1}=
\begin{bmatrix}
\sqrt{3} & \sqrt{3} \\
1 & -1
\end{bmatrix}
$$

平移向量为$(-1,0)$，即

$$
\begin{bmatrix}
y_1' \\
y_2'
\end{bmatrix}=
\begin{bmatrix}
\sqrt{3} & \sqrt{3} \\
1 & -1
\end{bmatrix}
\begin{bmatrix}
y_1 \\
y_2
\end{bmatrix}+
\begin{bmatrix}
-1 \\
0
\end{bmatrix}
$$

不过，对于 3 种因素的侧重，可以认为一般人根本不在乎节点质量，而只考虑节点的价格和流量。所以，可以选择截半边的二元正态分布。截取之前的参数为

$$
(y_1,y_2)\sim \mathcal{N}\left(
\begin{bmatrix}
1/2 \\
1/2
\end{bmatrix},
\begin{bmatrix}
1/9 && 0 \\
0 && 1/9
\end{bmatrix}
\right)
$$

即概率密度函数为

$$
a(y_1,y_2)= \frac{9}{\pi}e^{\displaystyle -\frac{9}{2}\left[(x_1-1/2)^2+(x_2-1/2)^2\right]}
$$

这两个参数应该也是不需要参数估计的。

```python
from torch.distributions import Pareto, Normal
from torch.distributions.multivariate_normal import MultivariateNormal

# 定义用户预算分布
user_budget_distribution = Pareto(1, 0.6)
# 定义用户流量需求分布
user_traffic_demand_distribution = Normal(130, 30)
# 定义用户质量需求分布
user_quality_demand_distribution = Pareto(1, 2)
# 定义用户侧重分布
user_weight_mean = torch.tensor([0.5, 0.5])
user_weight_covariance = torch.tensor([[1 / 9, 0], [0, 1 / 9]])
user_weight_distribution = MultivariateNormal(user_weight_mean, user_weight_covariance)

# 生成样本
sample_size = 10000     # 样本数量
user_budget_sample = user_budget_distribution.sample((sample_size,))
user_traffic_demand_sample = user_traffic_demand_distribution.sample((sample_size,))
user_quality_demand_sample = user_quality_demand_distribution.sample((sample_size,))
user_weight_sample = user_weight_distribution.sample((sample_size,))

# 模拟用户选择套餐
y_1 = user_weight_sample[:, 0]
y_2 = user_weight_sample[:, 1]
y_3 = 1 - y_1 - y_2
user_sample = torch.stack([user_budget_sample, user_traffic_demand_sample, user_quality_demand_sample], dim=1)
user_weight_sample = torch.stack([y_1, y_2, y_3], dim=1)
user_satisfaction_of_packages = []
for i in range(packageNumber):
    pr = packagePrice[i]
    t = packageTrafficLimit[i]
    q = packageQuality[i]
    satisfactions = user_satisfaction(user_sample, user_weight_sample, pr, t, q)
    user_satisfaction_of_packages.append(satisfactions)
    
user_satisfaction_of_packages = torch.stack(user_satisfaction_of_packages, dim=1)
max_values, max_indices = torch.max(user_satisfaction_of_packages, dim=1)
indices = torch.where(max_values < 0.1, torch.tensor(-1), max_indices)
```

## 参数估计

综上，需要估计的参数有：

+ 用户预算的帕累托分布$k$参数：$k_b$
+ 流量需求分布的均值和方差：$\mu_t$,$\sigma^2_t$
+ 质量需求的帕累托分布$k$参数：$k_q$

似乎是可以通过神经网络估计出这几个参数，不过我没去研究，初始参数又不是不能用。

## 最小化成本

### 参数定义

假设用户的节点偏好为

$$
R =
\begin{bmatrix}
r_1 \\
r_2 \\
\vdots \\
r_n
\end{bmatrix}
$$

且规定$\sum_{i=1}^n r_i = 1$

用户在各节点实际使用的流量为

$$
U =
\begin{bmatrix}
u_1 \\
u_2 \\
\vdots \\
u_n
\end{bmatrix}
=uR
$$

其中，$u$为用户实际使用的总流量。

假设用户不会因为差点用超流量而改用不喜欢的节点

不考虑每个节点能够承载的用户数量限制，假设用户会先用完节点的流量而不是因为请求导致节点超载。于是，假设各个节点的边际成本为

$$
C=\begin{bmatrix}
c_1 \\
c_2 \\
\vdots \\
c_n
\end{bmatrix}
$$

则成本为$C'U=uC'R$

这里假设了边际成本为常数，但一般机场是规模经济（边际成本随用量降低）的。

若定价为$p_r$，则边际利润为$p_r-uC'R$

于是，得到利润数据，平均每个套餐的利润为

$$
\bar{r} = p_r - \bar{u}C'R
$$

### 超售问题

为了确定节点的流量边际成本，需要计算超售比，因为$c_n=c_{tn}/R_{os}$，其中$c_{tn}$为实际流量边际成本，$R_{os}$为超售比。

为了确保不会因为超售太多而导致可用性下降，定义$Ra_{max}$为超售可用性，意义为：有$Ra_{max}$的概率不会因为超售而不可用。

这里定义超参数$Ra_{max}=1-10^{-4}=99.99\%$

对于套餐$g_1,g_2,\cdots,g_n$，设用户实际使用总流量的概率密度函数为$f_{n}(u)$，则对于某一个节点，使用的流量的概率密度函数为$r_nf_n(ur_n)$

设节点实际的流量为$t_r$，则为满足可用性，需要

$$
Pr\{u<t_r\} \geq Ra_{max}
$$

即

$$
\sum_n\int_0^{t_r} r_nf_n(ur_n) dur_n=\sum_n\int_0^{t_r/r_n}f_n(u)du\geq Ra_{max}
$$

例如，假设只有1个套餐，这个套餐的限额为150G，对于日本节点，倍率为1，用户的偏好程度为0.5，使用的流量服从正态分布$u\sim N(80,100)$，那么，$t_r$应该为$58.595$，超售比为$2.5423$

> 用户使用的流量在与之前参数估计得到的用户需求流量大致相同，但有一定误差。这是因为，用户往往会高估自己需要使用的流量。不过，假设用户实际使用的流量等于用户流量需求可能是可行的，因为留一点流量上的冗余是问题不大的。

因为

$$
R_{os}=\frac{t_r}{\sum_n r_nt}
$$

而$t_r$可以求解方程得到，

所以$R_{os}$可以由数据分析得到。

### 最大期望利润问题

最大期望利润问题，也就是使得

$$
\sum_n f_n\bar{r_n} =f_n\left( p_{r,n} - \bar{u_n}C'R\right)
$$

最大的问题。（其中$f_n$为选择率）

由于$C$和$R$都是基本不变的量，而且在参数估计出用户的需求分布后，$f_n$只取决于套餐的属性，$\bar{u_n}$也是稳定的，所以最大利润问题就是确定套餐属性的问题。

如果假设套餐的质量属性是常数，那么变量就只有各个套餐的价格和流量了。

所以，计算最大期望利润的方法为：

+ 统计得到$C'R$
+ 确定各个套餐的质量
+ 确定各个套餐的价格或者流量，把另一个设置为变量。
+ 对变量进行梯度下降，使上面那个和式取最大值
+ + $f_n$可以在有用户的数学模型之后，蒙特卡罗方法得到
+ + $p_r$为价格，是输入变量
+ + $u_n$与用户的建模相关