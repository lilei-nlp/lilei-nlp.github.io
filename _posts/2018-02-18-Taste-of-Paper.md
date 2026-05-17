---
title: "Taste of Paper"
date: 2018-02-18
categories:
  - blog
tags: []
---

好久没更新 Blog，随便写点这段时间写文章的感想吧。

## <a href="#Paper-的种类" class="headerlink" title="Paper 的种类"></a>Paper 的种类

琢磨 Paper 的 Experiment 怎么写的时候，想起老师说的，Paper 大体上分三种：

1.  Paper for paper：为了写而写，这是我们尽可能需要避免的一类文章。似乎挺多综述文章可以归到这一类了，并不是说总结前人工作没有意义，而是一部分领域已经有较好综述类文章的还写… 就难免刻意之嫌。
2.  Paper for application: 这应该是目前 CS 或者说是 DL 领域最多的文章了。大家把一开始用在 CV 上的技术或者是模型套到 NLP 上看看能不能 work，或者是拿着第三档的 idea 在一些更实际的问题上跑一个 state of art 的结果。有些人把这些文章也归入灌水的行列，我觉得有失偏驳。首先这类文章对落地，工业界出产品很有帮助的，理论/模型提出 -&gt; 在细分问题上做出好的结果 -&gt; 工业现实应用大致是这么个流程，没了第二步直接转化是不显示的；另外一方面，能起到倒逼创新的作用，像 CNN 的几种架构在 ImageNet 上狂刷 Accuracy，大家怎么调参都无法进步的时候自然回想着去探索一些新的思路。
3.  Paper for idea/innovation: 说实在话，这一档的文章基本上都是大牛的作品了，算是挖出一个巨坑然后无数人前仆后继的往里跳，比如 Goodfellow 的 GAN 和 Adversarial 一系列作品。

三种的 Paper 客观存在，不能说好坏吧。毕竟科研工作者也是人，也要混饭吃，谁能保证一个大牛一开始不是靠糊墙支撑到他挖坑呢？但就本科生来讲，我觉得第二种性价比比较高的。第一种 Paper 可能会被你未来的老板看穿然后认为你就是个科研混子而拒绝了你，第三种势必要有对某一领域的足够积累之后才可能，毕竟这么多人盯着呢谁不想做下一个大牛去 FAIR 当主任… 拍脑袋想出的觉得能改变世界的 idea 不是没有，只不过如我这样的普通人可能第二天起来仔细分析一下就给自己判死刑了。第二种要求你对特定领域问题有深入了解，能看懂最新的 Paper，同时要能够实现相关的模型以及具备调优的能力，大约是一个准 PhD 应该具备的简单版技能树吧。这样老师可能觉得：“诶这学生还不错，培养培养可能是个搬砖的能手。”然后就把你收了（逃

## <a href="#Taste-Of-Paper" class="headerlink" title="Taste Of Paper"></a>Taste Of Paper

即使是同一档次的文章，其作者的 taste 也是有着蛮大差别的，拿在读的两篇文章来举个例子。

### <a href="#Adversarial-Training-Methods-for-Semi-Supervised-Text-Classification" class="headerlink" title="Adversarial Training Methods for Semi-Supervised Text Classification"></a>Adversarial Training Methods for Semi-Supervised Text Classification

前面已经写过详细写过这篇文章了，可以算是首次 Adversarial Training 应用在 NLP 领域吧。Adversarial Training 在 CV 领域已经用的蛮多了，自然地想到能不能迁移到 NLP，于是在 Word Embedding 层面搞了事情，做出了Semi-Supervised Text Classification 的 SoA。我个人对这篇文章是很有好感的，能算作第二种里面比较接近第三类的，挖了一个 Adversarial Training 在 NLP 的小坑吧，后面的一篇文章就是用这个技术在相对而言比较小众一点的 Relation Extraction 问题上做了工作。文章的 Taste 不错，体现在以下几个方面：

1.  对于 Adversarial Training（AT） 的定位：在 CV 领域， AT 是作为一种对抗 Adversarial Example 的手段，作为防御机制而提出的，这篇文章认为 AT 是一种 Regularization，通过 Word Embedding 质量的提升来达到避免过拟合的目的，并且通过可视化 good 和 bad 的几个 neighbor 词语来实证；进一步地，对比了 AT 和加 Noise 的效果，有信服力地说明了 AT 是一种更强力的 Regularization
2.  对实验结果的分析：Paper 本身就是对结果在做解释，但实验结果中出现的意料之外的情况还是需要作者的阐述的，文章提出的一个方法在其中一个数据集上性能输于 baseline， 也给出了较合理的解释。
3.  参数的选择：这点应该是很多 SoA 的文章都欠缺的，这篇似乎也没能给出比较详尽的解释参数选择的原因。反正能 Work 嘛，反正是玄学（逃

### <a href="#Adversarial-Training-for-Relation-Extraction" class="headerlink" title="Adversarial Training for Relation Extraction"></a>Adversarial Training for Relation Extraction

这篇就是典型的第二类的文章了，应该是受了前一篇文章的启发，在一个新的问题上评估 AT 的性能。这篇的 Taste 可能就不如前一篇，我觉得一个是很多讨论都过于浅显，没有继续深挖；另一个实验做的也不是很全，可能是囿于篇幅和时间吧。整体来说，灌水的可能性比较大。

惭愧地说，我在写的东西水平和这篇差不多，甚至还要低一点（不然我也能发顶会了不是）。但我相信，我以后肯定能够完成出色的工作的。

## <a href="#Supervised-to-Unsupervised" class="headerlink" title="Supervised to Unsupervised"></a>Supervised to Unsupervised

由监督学习到非监督学习，应该也是一个趋势。

看了 Goodfellow 组最新的 GAN 在文本生成上的一篇文章，《Maskgan: Better Text Generation Via Filling in the **\_\_\_\_**》 标题把下划线进去，一开始我都以为这是漏写或者是 Bug，读过之后发现不然。文章大致是用类似初高中完形填空的方法来让 GAN 学习生成文本，但原先 GAN 在文本领域的尝试失败说是梯度无法回传（？存疑，具体有待研究），这里加入了 Reinforcement Learning （RL，强化学习）的机制，来传递梯度更新 Generator 的参数。

讲上面的例子是想说明一点， RL 是更加接近人类学习过程的一种手段，Reward 函数的设置真的很像一个人活着，为了某一个目标而不断奋斗的这样一个过程。相信未来很多如今是 Supervised 的问题都会与 RL 结合产生新的碰撞。

## <a href="#Future" class="headerlink" title="Future"></a>Future

接下来的半个寒假的计划大概就是：

1.  继续完善手头的工作
2.  细读 Goodfellow 新作，写一篇笔记，尝试跑代码
3.  学一波 RL，尝试写个打游戏的 Demo
4.  刷 PAT
