---
title: Understanding LSTM Networks
date: 2018-07-17 11:22:56
categories:
tags:
copyright:
top:
type:
mathjax: true
---
这是一篇译文,[原文地址](https://colah.github.io/posts/2015-08-Understanding-LSTMs/)

# 循环神经网络(Recurrent Neural Networks)

在我们思考时,我们不会从头开始,肯定会在思考时加入之前的知识.就如同当你在阅读当前的博客时,你读的每个单词都是基于前面的单词.你不会扔掉所有的东西,然后在从头开始.你的想法有持久性.
传统的神经网络不能使用之前信息.假设你使用传统神经网络来将一部电影中的正在发生的事件分类,怎么使用当前事件之前的信息是很困难的.
循环神经网络解决了这个问题,在其中有循环,允许信息能够保持.
{% asset_img RNN-rolled.png Recurrent Neural Networks have loops. %}
这个图看起来有点奇怪,我们可以将其展开,如下图所示
{% asset_img RNN-unrolled.png An unrolled recurrent neural network %}
从图中可以看出,RNN使用了前一时刻的状态来预测当前的状态
RNN在很多领域都取得了成功,比如:语音识别,自然语言处理,翻译,图像字幕等等.这里Andrej Karpathy的[blog](http://karpathy.github.io/2015/05/21/rnn-effectiveness/),里面列举了RNN一些应用.
本文主要介绍LSTM模型,LSTM模型是RNN的一种变种,在很多领域的表现都要比一般的RNN模型更好

# 长期依赖存在的问题(The Problem of Long-Term Dependencies)

加入我们要基于之前的单词预测下一个单词,如"the clouds are in the *sky*",我们要预测最后一个单词sky.在这个例子中,我们不需要更多的上下文,很明显最后一个单词是sky.RNN针对这种情况有很好的表现.
{% asset_img RNN-shorttermdepdencies.png %}
但是在有些情况下,我们需要更多的上下文.例如,我们要预测"I grew up in France… I speak fluent *French*."中的最后一个单词French.从最近的信息,比如speak fluent等,可以推测最后一个单词是一种语言.但是如果要知道具体的语言,我们需要更多的上下文.预测点和其对应的相关信息之间的差距很大是由很大可能的.
但是,不幸的是随着差距增加,RNN不可能学会如何连接这种信息
{% asset_img RNN-longtermdependencies.png %}
从图中可以看出$h_{t+1}$不能很好地利用$X_0$和$X_1$的信息.理论上,RNN能够学习到长期依赖,但是在实践中,要想在RNN中使用这些长期依赖很困难.幸运的是LSTM模型解决了这个问题.

# LSTM网络(LSTM Networks)

LSTM全称Long Short Term Memory,长短期记忆网络.LSTM的设计就是为了避免长依赖问题,记忆长周期的信息是LSTM的默认行为.
所有的递归神经网络都具有重复模块的链式结构.在标准的RNN中,重复模块是一个非常简单的结构,例如单层tanh
{% asset_img LSTM3-SimpleRNN.png The repeating module in a standard RNN contains a single layer. %}
LSTM也有这种链式结构,但是重复的模块有一个不同的结构,在其中一个模块中,包含一个四层的神经网络,并且这四层以一种特别方式进行交互.
{% asset_img LSTM3-chain.png The repeating module in an LSTM contains four interacting layers.%}
图中的示例如下
{% asset_img LSTM2-notation.png %}
接下来我会详细介绍其中涉及的内容.

# LSTMs中的核心思想(The Core Idea Behind LSTMs)

LSTMs中的核心是单元状态,在图中就是最上面的那条水平线.单元状态就像是一条输送带.单元状态沿着整个链流动,对其只有一些线性作用,信息的流动很容易.
{% asset_img LSTM3-C-line.png %}
LSTM能够移除或者添加信息到单元状态,并且由成为门的结构来控制.门可以选择性的让信息通过或不通过,它是由sigmoid神经网络层和点积操作组成.
{% asset_img LSTM3-gate.png %}
sigmoid层的输出范围0-1,描述了可以通过多少的信息.输出为0意味着什么都不通过,输出为1意味着都能通过.从LSTM的结构中,我们可以看到LSTM的一个重复模块包括三个这样的门.

# 一步一步认识LSTM

LSTM的第一步就是决定从单元状态中丢弃哪些信息.这个决定是由一个称为"遗忘门(forget gate layer)"单层sigmoid层组成.它接收$h_{t-1}$和$x_t$作为输入,输出0到1之间的数字.对于前一个单元状态$C_{t-1}$,当遗忘门的输出为1时,表示完全保留$C_{t-1}$;当遗忘门的输出为0时,表示完全舍弃$C_{t-1}$.
我们在回到之前根据之前的内容来预测下一个单词这个问题.在这样的问题中,单元状态可能包括当前对象的性别,所以可以使用正确的代词.但是当遇到一个新的对象时,我们就想要忘记老对象的性别.
{% asset_img LSTM3-focus-f.png %}

>图中的$W_f \cdot [h_{t-1},x_t]$,大家看起来有些不清楚.个人认为比较合适的写法如下
>$\sigma(W_fx_t + U_r h_{t-1} + b_f)$.大家可以参考下[英文博客](http://blog.echen.me/2017/05/30/exploring-lstms/),[中文博客](https://www.jiqizhixin.com/articles/2017-07-24-2)

