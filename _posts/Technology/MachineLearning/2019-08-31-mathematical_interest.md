---

layout: post
title: 直觉上理解机器学习
category: 技术
tags: MachineLearning
keywords: 深度学习

---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

* TOC
{:toc}

## 前言

区块链、docker是各种技术的整合，机器学习也是

## 线性回归

很多人工智能问题可以理解为分类问题，比如判断一个邮件是否为垃圾邮件，根据身高体重预测一个人的性别，当我们将目标特征数值化之后，可以将目标映射为n维度空间上的一个点

从几何意义上来说，分类问题就是 在n维空间上将 一系列点 划分成 不同的集合，以二分为例

1. 对于二维空间，分类器就是一条线，直线or 曲线
2. 对于三维空间，分类器就是一个面，平面or 曲面
3. 对于高维空间，xx

**我们要做的就是根据一系列样本点，找到这条“分界线/面”**。线性回归假设特征和结果满足线性关系，线性关系的表达能力非常强大，通过对所有特征的线性变换加一次非线性变换，我们可以划一条“近似的分界线/面”，虽然并不完全准确，但大部分时候已堪大用。

就像数学中的泰特展开式一样，不管多么复杂的函数，泰勒公式可以用这些导数值做系数构建一个多项式来近似函数在这一点的邻域中的值。而不同的函数曲线其实就是这些基础函数的组合，理所当然也可以用多项式去趋近

### 正向传播——计算估计值的过程

[【机器学习】代价函数（cost function）](https://www.cnblogs.com/Belter/p/6653773.html)

给出一个输入数据，我们的算法会通过一系列的过程得到一个估计的函数，这个函数有能力对没有见过的新数据给出一个新的估计，也被称为构建一个模型。

假设有训练样本(x, y)，数据有两个特征x1,x2，模型为h，参数为θ，\\(\theta=(w1,w2,b)\\)，估计函数

$$h(x)=g(θ^T x)=g(w_1x_1+w_2x_2+b)$$

即对x1,x2 施加一次线性变换 + 非线性变换，（\\(θ^T\\)表示θ的转置）。因为x1,x2 都是已知的，所以h(θ) 是一个关于θ的函数

两层神经网络的表示（不同文章的表示用语略微有差异）

![](/public/upload/machine/two_layer_neural_network.jpeg)

### 正向传播为什么需要非线性激活函数

![](/public/upload/machine/activation_function.jpg)

如果要用线性激活函数，或者没有激活函数，那么无论你的神经网络有多少层，神经网络只是把输入线性组合再输出，两个线性函数组合本身就是线性函数，所以不如直接去掉所有隐藏层。唯一可以用线性激活函数的通常就是输出层

常见激活函数： sigmoid,tanh,ReLU,softMax [机器学习中常用激活函数对比与总结](https://zhuanlan.zhihu.com/p/40327813)

### 反向传播——调整θ的过程

从直觉上来说， 如何确定最优的w、b？其他参数都不变，w（或b）的微小变动，记作Δw（或Δb），然后观察输出有什么变化。不断重复这个过程，直至得到对应最精确输出的那组w和b，就是我们要的值。

反向传播的思路：

1. 概况来讲，任何能够衡量模型预测出来的值h(θ)与真实值y之间的差异的函数都可以叫做代价函数C(θ)，如果有多个样本，则可以将所有代价函数的取值求均值，记做J(θ)。因此很容易就可以得出以下关于代价函数的性质：

    * 对于每种算法来说，代价函数不是唯一的；
    * 代价函数是参数θ的函数；
    * 总的代价函数J(θ)可以用来评价模型的好坏，代价函数越小说明模型和参数越符合训练样本(x, y)；
    * J(θ)是一个标量；

2. 当我们确定了模型h，后面做的所有事情就是训练模型的参数θ。那么什么时候模型的训练才能结束呢？这时候也涉及到代价函数，由于代价函数是用来衡量模型好坏的，我们的目标当然是得到最好的模型（也就是最符合训练样本(x, y)的模型）。因此训练参数的过程就是不断改变θ，从而得到更小的J(θ)的过程。理想情况下，当我们取到代价函数J的最小值时，就得到了最优的参数θ.例如，J(θ) = 0，表示我们的模型完美的拟合了观察的数据，没有任何误差。

3. 代价函数衡量的是模型预测值h(θ) 与标准答案y之间的差异，所以**总的代价函数J是h(θ)和y的函数**，即J=f(h(θ), y)。又因为y都是训练样本中给定的，h(θ)由θ决定，所以，最终还是模型参数θ的改变导致了J的改变。

4. 在优化参数θ的过程中，最常用的方法是梯度下降，这里的梯度就是代价函数J(θ)对`θ1, θ2, ..., θn`的偏导数。由于需要求偏导，我们可以得到另一个关于代价函数的性质：选择代价函数时，最好挑选对参数θ可微的函数（全微分存在，偏导数一定存在）


### 梯度下降法 ==> 求使得J极小的(w1,w2,b)

1. 首先对(w1,w2,b)赋值，这个值可以是随机的，也可以让(w1,w2,b)是一个全零的向量
2. 改变(w1,w2,b)的值，使得J(w1,w2,b)按梯度下降的方向进行减少，梯度方向由J(w1,w2,b)对(w1,w2,b)的偏导数确定
3. \\(w1=w1+\sigma d_{w1}\\) 依次得到 w1,w2,b 的新值，\\(\sigma\\)为学习率


### 损失函数如何确定

[机器学习-损失函数](https://www.csuldw.com/2016/03/26/2016-03-26-loss-function/)

**损失函数是h(θ)和y的函数**，本质是关于\\(\theta\\)的函数，分为

1. log对数损失函数，PS：概率意义上，假设样本符合伯努利分布

	$$L(Y,P(Y|X))=-logP(Y|X)$$
2. 平方损失函数 ，PS：概率意义上，假设误差符合高斯分布
	
	$$L(Y,h_\theta(X))=(Y-h\_\theta(x))^2$$

   $$MSE(\theta)=MSE(X,h_\theta)=\frac{1}{m}\sum\_{i=1}^m(\theta^T.x^i-y^i)^2$$
    
    

3. 指数损失函数
4. Hinge损失函数
5. 0-1损失函数
6. 绝对值损失函数

	$$L(Y,h_\theta(X))=|Y-h\_\theta(x)|$$



https://zhuanlan.zhihu.com/p/46928319

[对线性回归，logistic回归和一般回归的认识](https://www.cnblogs.com/jerrylead/archive/2011/03/05/1971867.html)



## 逻辑回归

逻辑回归是线性回归的一种特例，激活函数使用 Sigmoid，损失函数使用 log对数损失函数

正向传播过程

$$f(x)=\theta^Tx=w_1x_1+w_2x_2+b$$


$$g(z)=\frac{1}{1+e^{-z}}$$

### 激活函数为什么使用Sigmoid

1. 它的输入范围是正负无穷，而值刚好为（0，1），正好满足概率分布为（0，1）的要求。我们**用概率去描述分类器**，自然比单纯的某个阈值要方便很多
2. 它是一个单调上升的函数，具有良好的连续性，不存在不连续点

### 损失函数为什么使用对数损失函数

\\(h_\\theta(x)\\)与y 只有0和1两个取值

#### 对数损失函数的直接意义

[logistic回归详解(二）：损失函数（cost function）详解](https://blog.csdn.net/bitcarmanlee/article/details/51165444)

$$
cost(h_\theta(x),y)=
\begin{cases}
-logh\_\theta(x)\ \ \ \ \ \ \ \ \ \ \ if\ y=1 \\\
-log(1-h\_\theta(x))\ \ if\ y=0 \\\
\end{cases}
$$

当y=1时

1. 如果此时\\(h_θ(x)=1\\)，\\(logh\_θ(x)=0\\)，则单对这个样本而言的cost=0，表示这个样本的预测完全准确。
2. 如果此时预测的概率\\(h_θ(x)=0\\)，\\(logh\_θ(x)=-\infty\\)，\\(-logh\_θ(x)=\infty\\)，相当于对cost加一个很大的惩罚项。 

当y=0 时类似，所以，**取对数在直觉意义上可以将01二值与 正负无穷映射起来**

汇总一下就是

|\\(h_\\theta(x)\\)|y|cost|
|---|---|---|
|0|0|0|
|0|1|\\(\infty\\)|
|1|0|\\(\infty\\)|
|1|1|0|

将以上两个表达式合并为一个，则单个样本的损失函数可以描述为： 

$$
cost(h_\theta(x),y)=
-y\_ilogh\_\theta(x)-
(1-y\_i)log(1-h\_\theta(x))
$$

因为\\(h_\\theta(x)\\)与y 只有0和1两个取值，该函数与上述表格 或分段表达式等价

全体样本的损失函数可以表示为： 

$$
cost(h_\theta(x),y)=
\sum\_{i=1}^m
-y\_ilogh\_\theta(x)-
(1-y\_i)log(1-h\_\theta(x))
$$


#### 对数损失函数的概率意义

[逻辑回归为什么使用对数损失函数](https://blog.csdn.net/saltriver/article/details/63683092)

对于逻辑回归模型，假定的概率分布是伯努利分布（p未知）

$$
P(X=n)=
\begin{cases}
1-p   \ \ \ \ n=0\\\
p     \ \ \ \ \ \ n=1\\\
\end{cases}
$$

概率公式可以表示为（ x只能为0或者1）

$$
f(x)=p^x(1-p)^{1-x}
$$

假设我们做了N次实验，得到的结果集合为 `data={x1,x2,x3}`，对应的最大似然估计函数可以写成

$$
P(data|p)=
\prod_{i=1}^Nf(x\_i)=
\prod\_{i=1}^Np^{x\_i}(1-p)^{1-x\_i}
$$

要使上面的式子最大，等价于使加上ln底的式子值最大，我们加上ln的底就可以将连乘转换为加和的形式

$$
lnP(data|p)=
\sum\_{i=1}^Nx\_ilnp+(1-x\_i)(1-p)
$$

对数损失函数与上面的极大似然估计的对数似然函数本质上是等价的，xi 对应样本实际值yi，p即 \\(h_\theta(x)\\)，所以逻辑回归直接采用对数损失函数来求参数，实际上与采用极大似然估计来求参数是一致的

**几何意义和概率意义殊途同归（当然，不是所有的损失函数都可以这么理解）**

## 概率论

![](/public/upload/machine/probability_theory.png)

[为什么我说概率论是大学最不能翘的一门数学课](https://zhuanlan.zhihu.com/p/36920233)

假设你在一个教室里，同时周围有30名学生，游戏的规则是这样的：一开始，大家都是鸡蛋，你们需要和周围同类型的物种进行石头剪子布来升级，鸡蛋找鸡蛋划石头剪子布，胜利者就变成了小鸡。接着小鸡找小鸡划，胜利者就变成了凤凰，然后，凤凰和凤凰划，胜利者就变成了人，变成人的同学就是最后的赢家，同时，每次石头剪子布输的那一方都会降一级，比如从凤凰降级到小鸡，小鸡降级到鸡蛋，游戏限时十分钟，所以，现在问你在这个游戏中如何取胜（此处认真思考二十秒）

如果一个人在这十分钟内只进行一轮石头剪子布（我们现在定义由鸡蛋到人的三次石头剪子布为一轮，如果提前输掉则视作此轮结束）那么这个人变成人的概率为1/8，而如果它进行两轮，那么我们先不管他是在第一轮还是第二轮中变成了人，只要最后变成人了就OK，所以我们只用讨论它两轮都输的概率，就是（1-1/8）^2，而变成人的概率为1-（1-1/8）^2=15/64>1/8，依次类推，可知道如果进行n轮，则他变成人的概率为1-（1-1/8）^n,当n趋于无穷时，这个概率将趋于 1，所以这个游戏取胜的秘诀就是不停的和周围的人进行石头剪子布，频次越高，也就越有机会变成人。

对于大多数玩家而言，是不会仔细思考这个游戏背后的数学模型的。想对上述游戏进行一次完整而严谨的建模也将是一个十分复杂的过程。

[概率：了解不确定性](https://songshuhui.net/archives/93539)在1654年的一天早上，法国数学家布莱兹·帕斯卡收到了他的朋友贡博的一封来信：两位贵族A与B正在进行一场赌局，赌注是每人500法郎，两人轮流掷硬币，得到正面则A得一分，反面则B得一分，每一局两人得分的机会相等，谁先得到6分谁就得到1000法郎。两人激战正酣，比分达到2比4之际，B突然有事需要终止赌局。赌注应该如何分配才最公平。

对于某个非常简单的随机事件，比如说掷硬币，我们知道每种结果出现可能性的大小，这样的事件被称为“基本事件”。我们可以多次重复这些基本事件，假定它们发生的可能性不会改变，而且这些重复没有相互影响。如果我们将这些基本事件以合适的形式组合起来，就能得到一个更为复杂而有趣的系统。许多概率问题实际上就是对这些随机系统的各种性质的研究。

### 概率与自然语言处理

概率的定义就是随机事件发生的可能性的度量，而信息则是减少随机不定性的东西，这两者生而就是有着千丝万缕的联系的，所以我们在研究自然语言处理（本质是通信）这类信息时，也必然要引入概率的知识

### 贝叶斯公式的另类解读

```
P(AB) = P(A|B) * P(B) = P(B|A) * P(A)
```

$$P(A|B) = \frac{P(B|A) * P(A)}{P(B)} = \frac{P(B|A)}{P(B)} * P(A)$$

`P(A|B)`  可以视为 `P(A)` 的增强，也就是后验概率是先验概率的增强。**观测者观测历史数据得出预测假设P(A)，然后新的信息`P(B)`出现，观测者修正预测为`P(A|B)`**。`P(B|A)/P(B)` 被称为调整因子。

定义`P(A|B)`是P`(B|A)`的逆概率。贝叶斯最早的目的就是研究一个概率和他的逆概率之间的关系。

在实际的场景下， A 和 B 通常代表了一个结果和原因， `P(结果|原因)`好算，`P(原因|结果)`知果索因就很麻烦，**不然侦探片就没那么好看了**。

人们已经能够计算”正向概率“，如“一个袋子N个白球M个黑球，你伸手进去摸一把，摸出黑球的概率多大”。而一个自然而然的问题反过来：如果事先不知道袋子中黑白球比例，而是闭着眼睛摸出好几个球，观察这些取出来的球的颜色之后，那么我们可以对袋子中黑白球的比例做出怎样的推测？如果再摸出几个球，是否要对刚才的推测进行校正？

贝叶斯公式看着没什么感觉，但在贝叶斯分类中就很有用武之地了。比如我们将水果的形状、颜色、纹理、重量、握感、口感全部数值化。可以得到一个苹果、橙子是黄颜色的概率分别是多少，基于贝叶斯公式就可以反推一个黄色的、圆形水果是苹果、橙子的概率。前者是训练数据，后者是计算机根据贝叶斯分类算法做出的判断。

[怎么简单理解贝叶斯公式?半瓶晃荡加水中的回答 ](https://www.zhihu.com/question/51448623/answer/306116102)最重要的思维并非是逻辑思维，是直觉式思维。在学习过程中，必须把知识和直觉思维联系起来，才能真正掌握这个知识。直觉性思维的本质，是通过观察少量结果，反向推导出事物的因果联系。

### 概率论与数理统计

||概率论|数理统计|
|---|---|---|
|方法|根据已知的分布来分析随机变量的特征与规律|根据得到的观察结果对原始分布做出推断<br>具体的说是假定样本符合某一分布，估计分布对应概率函数的参数值|
|例子|已知摇奖规律判断一注号码中奖的可能性|根据之前多次中奖不中奖的号码以一定的精确性推断摇奖的规律|
|例子|已知人群身高符合正态分布，预测一个陌生人的身高|已知一个群体的身高， 且假设符合正态分布，推测正态分布的参数值|

不管是概率论，还是数理统计，分布对应的概率函数都是已知的。

对于`P(x|θ)`输入有两个：x表示某一个具体的数据；θ表示模型的参数。

1. 如果θ是已知确定的，x是变量，这个函数叫做概率函数(probability function)，它描述对于不同的样本点x，其出现概率是多少。
2. 如果x是已知确定的，θ是变量，这个函数叫做似然函数(likelihood function), 它描述对于不同的模型参数，出现x这个样本点的概率是多少，也可以用`P(x;θ)` 来表示

[详解最大似然估计（MLE）、最大后验概率估计（MAP），以及贝叶斯公式的理解](https://blog.csdn.net/u011508640/article/details/72815981)

1. 最大似然估计是求参数θ, 使似然函数`P(data|θ)`最大。换个说法，求使似然函数`P(data|θ)`最大的θ
2. 最大后验概率估计则是想求θ使`P(data|θ)P(θ)`最大。换个说法，求使`P(data|θ)P(θ)`最大的θ

假设有一个造币厂生产某种硬币，现在我们拿到了一枚这种硬币，想试试这硬币是不是均匀的。即想知道抛这枚硬币，正反面出现的概率（记为θ）各是多少？于是我们拿这枚硬币抛了10次，得到的数据（data）是：反正正正正反正正正反。我们可以假设抛硬币模型是**二项分布**，正面概率θ是模型参数，如何根据 data 求θ？

### 最大似然估计MLE

那么，出现实验结果data（即反正正正正反正正正反）的似然函数是多少呢？

$$P(data|θ)=(1−θ)×θ×θ×θ×θ×(1−θ)×θ×θ×θ×(1−θ)=θ^7(1−θ)^3=f(θ)$$

注意，这是个只关于θ的函数。而最大似然估计，顾名思义，就是要最大化这个函数。对应使得 f(data,θ) 导数为0 的点， 在θ=0.7时，似然函数取得最大值。

### 最大后验概率估计MAP

一些人可能会说，硬币一般都是均匀的啊！ 就算你做实验发现结果是“反正正正正反正正正反”，我也不信θ=0.7。如果一枚硬币抛10次，10次均为正面，根据最大似然估计，那么这枚硬币的概率应该为1。这也未免太武断了。

MAP其实是在最大化

$$P(θ|data)=\frac{P(data|θ)P(θ)}{P(data)}$$

不过因为data是确定的（即投出的“反正正正正反正正正反”），P(data)是一个已知值，所以去掉了分母P(data)。对于投硬币的例子来看，我们认为（”先验地知道“）θ取0.5的概率很大，取其他值的概率小一些。我们用一个高斯分布来具体描述我们掌握的这个先验知识，例如假设P(θ)为均值0.5，方差0.1的高斯函数

![](/public/upload/machine/gaussian_function.png)

\\(P(data|θ)P(θ)=θ^7(1−θ)^3P(θ)\\)，则`P(data|θ)P(θ)`的函数图像为：

![](/public/upload/machine/gaussian_function_move.png)

在θ=0.558时函数取得了最大值。那要怎样才能说服一个贝叶斯派相信θ=0.7呢？你得多做点实验。如果做了1000次实验，其中700次都是正面向上，则在θ=0.696处，`P(data|θ)P(θ)`取得最大值。

[最大后验估计](https://zhuanlan.zhihu.com/p/32616870)此外再提两点：

1. 如果先验认为这个硬币是概率是**均匀分布**的，被称为无信息先验( non-informative prior )，通俗的说就是“让数据自己说话”，此时贝叶斯方法等同于频率方法。
2. 随着数据的增加，先验的作用越来越弱，数据的作用越来越强，参数的分布会向着最大似然估计靠拢。而且可以证明，最大后验估计的结果是先验和最大似然估计的凸组合。

## 参数和超参数

parameters: \\(W\^{[1]},b^{[1]},W^{[2]},b^{[2]},...\\)

超参数hyperparameters: 每一个参数都能够控制w 和 b

1. learning rate
2. 梯度下降算法循环的数量
3. 隐层数
4. 每个隐层的单元数
5. 激活函数

深度学习的应用领域，很多时候是一个凭经验的过程，选取一个超参数的值，试验，然后再调整。

吴恩达：我通常会从逻辑回归开始，再试试一到两个隐层，把隐藏数量当做参数、超参数一样调试，这样去找比较合适的深度。

## 小结

首先你要知道 机器学习的两个基本过程

1. 正向传播，线性 + 非线性函数（激活函数） 得到一个估计值
2. 反向传播，定义损失函数，通过（链式）求导 更新权重

关键问题

1. 激活函数的选择
2. 损失函数的选择
3. 对损失函数及\\(\theta\\)链式求偏导数，涉及到矩阵的导数，根据偏导 + 学习率 调整\\(\theta\\)
4. 向量化上述过程
4. 使用python 中线程的 矩阵/向量计算的库 将上述过程代码化， 涉及到numpy的学习

吴恩达：深度学习



