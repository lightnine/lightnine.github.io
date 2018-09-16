---
title: Gaussian Discriminant Analysis and Logistic Regression
date: 2018-09-16 11:22:36
categories:
- 机器学习
tags:
- Logistic Regression
- Gaussian Discriminant Analysis
- Discriminative model
- Machine Learning
- Generative model
- Statistics
copyright:
top:
type:
mathjax: true
---
# 高斯判别分析和逻辑回归算法的关系

最近一直在看Andrew Ng的cs229那门课的讲义，看到高斯判别分析模型和逻辑回归算法的关系那一部分，自己采用贝叶斯后验概率也证明了两者之间的关系，证明不难，本来打算记录一下的。在网上看到有个外国人写的更好，那我就把他写的直接翻译过来了。当然文章中也加入了在网上看到的其他资料和自己的一些思考。
[原文](https://duphan.wordpress.com/2016/10/27/gaussian-discriminant-analysis-and-logistic-regression/)

## 判别模型和生成模型

有很多方式可以对机器学习算法进行分类，比如：监督/非监督，回归/分类,等等。还有一种方式用判别模型(**Discriminative model**)和生成模型(**Generative model**)进行区分.本篇文章，我们将会讨论高斯判别分析(Gaussian Discriminant Analysis )模型和逻辑回归(Logistic Regression)之间的关系,这也体现了判别模型和生成模型之间的关系。
判别模型是求$p(y|x)$.在分类问题中，判别模型是直接寻找能够将不同类的数据分开的超平面或者称之为决策边界。有很多常用的机器学习算法属于判别模型，比如：逻辑回归，SVM，神经网络等等。另一方面，生成模型是求取$p(x|y)$和$p(y)$.这意味着，在分类问题中，生成模型给出了每个类的概率分布，给出了数据是如何生成的。也即生成(generative)这个词的意思.生成模型依赖贝叶斯公式计算后验概率$p(y|x)$.像朴素贝叶斯，高斯判别分析等都属于生成模型。
{% asset_img discriminative_vs_generative.png Discriminative vs Generative%}
在实际使用中，判别模型要比生成模型用的更多，原因很多。比如：判别模型更加灵活，更加鲁棒，当对模型做出错误的假设时，模型表现的也不是很敏感。而生成模型需要我们定义数据的先验概率，即数据分布，在很多时候这是非常有困难和挑战的。然后，生成模型也有它的优势，比如：相比于判别模型，生成模型对于数据有更多的先验信息。这导致如果我们对于数据的假设是正确的，那么生成模型在少量数据下也能够表现的更好。

## GDA和逻辑回归

接下来我将要证明，GDA(Gaussian Discriminant Analysis)如何最终推导出逻辑回归，从这个推导中也可以看出采用逻辑回归的道理。
对于二分类问题，GDA假设类别服从伯努利分布，数据服从多元高斯分布，如下：
{% asset_img 先验分布.PNG 先验分布%}
具体的数学公式如下：
{% asset_img 分布的数学公式.PNG 分布的数学公式%}
而采用GDA就是求取后验概率，我们可以证明如下等式成立：
$$p(y=1|x) = \frac{1}{1+exp(-\theta^Tx)}$$
在公式中可以看到，GDA关于类别1的后验概率就是逻辑回归的sigmoid函数，这里$\theta$就是关于参数$\phi,\mu_0,\mu_1,\sum$的函数
接下来进行证明：
$$
p(y=1|x)
=\frac{p(x|y=1)\times{p(y=1)}}{p(x)}
=\frac{p(x|y=1)\times{p(y=1)}}{p(x|y=1)\times{p(y=1)} + p(x|y=1)\times{p(y=0)}}
=\frac{1}{1 + \frac{p(x|y=0)\times{p(y=0)}}{p(x|y=1)\times{p(y=1)}}}
$$
我们接下来带入概率公式，计算$\frac{1}{1 + \frac{p(x|y=0)\times{p(y=0)}}{p(x|y=1)\times{p(y=1)}}}$,计算过程如下，主要是涉及一些log和exp的转换：
{% asset_img 证明.PNG 证明%}
在公式的最后，我们令$x_0=1$,至此我们得到了$\theta^Tx$,即如下式：
{% asset_img 证明结果.PNG 证明结果%}
$\theta$向量的值如下：
$$\theta=\left[
 \begin{matrix}
   log(\frac{1-\phi}{\phi}) - \frac{\mu_0^2 + \mu_1^2}{2\sum} \\
   \frac{\mu_0 - \mu_1}{\sum}
  \end{matrix}
  \right]
$$

## 关系说明

可以看出，从GDA可以推导出逻辑回归，但是反过来是不成立的。即如果$p(y|x)$是一个逻辑回归函数并不能推导出$p(x|y)$是一个多元的高斯分布。这说明GDA模型比逻辑回归模型具有更强的假设。事实上，如果$p(x|y)$是指数分布族(**Exponential Family**，比如高斯分布，泊松分布)中的一种，那么其后验概率就是逻辑回归。由此我们也得到一个为什么逻辑回归使用的更加广泛的原因，因为逻辑回归是一种非常通用，健壮的算法，适用于许多基本假设。另一方面，GDA以及一般的生成模型做出了更强的假设，因此对于非高斯或某些未定义分布数据来说并不理想。

假设有一个一维训练集，包含一些正样本和负样本，如下图x轴的叉和圈，设叉为0，圈为1，用GDA对两类样本分别拟合高斯概率密度函数p(x|y=0)和p(x|y=1)，如下图的两个钟形曲线。沿x轴遍历样本，在x轴上方画出相应的p(y=1|x)。如选x轴靠左的点，那么它属于1的概率几乎为0，p(y=1|x)=0，两条钟形曲线交点处，属于0或1的概率相同，p(y=1|x)=0.5，x轴靠右的点，输出1的概率几乎为1，p(y=1|x)=1。最终发现，得到的曲线和sigmoid函数曲线很相似。
{% asset_img 高斯分布和逻辑回归.png 高斯分布和逻辑回归%}
也就是说，当使用GDA模型时，p(x|y)属于高斯分布，计算p(y|x)时，几乎能得到和logistic回归中使用的sigmiod函数一样的函数，但实际上还是存在本质区别。

在Andrew Ng的cs229采用两种方法可以推导出逻辑回归

1. 通过指数分布族来推导。
2. 通过生成学习假设先验概率分布的方式进行推导。

## 如何选择两种模型

由上面的分析可以知道，GDA比逻辑回归有更多的前置假设。当数据服从或大致服从正态分布时，GDA会具有更高的拟合度，因为GDA利用了更多的信息构建模型。但是当数据不服从正态分布时，那么逻辑回归更有效，因为它做出更少的假设，构建的模型更加强壮，更加具有鲁棒性。生成学习还有另外的一个好处，就是可以使用比判别学习模型使用更少的数据构建出强壮的模型。

## 总结

通过推导，我们可以看到很多算法内在都是存在某种联系的，理解算法之间的关系对于我们理解算法以及选择什么算法都是有很多的帮助的。

## 参考文章

1. [生成学习算法.高斯判别分析(GDA).GDA与Logistic模型](https://blog.csdn.net/v1_vivian/article/details/52190572)
2. [Gaussian Discriminant Analysis and Logistic Regression](https://duphan.wordpress.com/2016/10/27/gaussian-discriminant-analysis-and-logistic-regression/)
