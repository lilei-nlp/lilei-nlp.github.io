---
title: "Adversarial Papers 脉络梳理"
date: 2017-10-26
categories:
  - blog
tags: []
---

Adversarial Training 是如今深度学习中非常重要的一种训练方法，这段时间读了这方面的一些 Paper，在此做些梳理，帮助理清思路。  
<span id="more"></span>

## <a href="#Intriguing-properties-of-neural-networks" class="headerlink" title="Intriguing properties of neural networks"></a>Intriguing properties of neural networks

这篇 Szegedy 在 2013 年的 Paper，如题目所说，阐述了神经网络的两个 intrigue 的属性：神经网络的空间含有语义信息（It suggests that it is the space, rather than the individual units, that contains the semantic information in the high layers of neural networks）；另一个则就是我们提到的 Adversarial Training，其第一次（存疑）提到了神经网络对于输入的扰动（perturbation）非常敏感，可以通过对输入增加扰动来形成对抗样本（adversarial examples），导致模型做出错误的分类。

## <a href="#Explaining-and-harnessing-adversarial-examples" class="headerlink" title="Explaining and harnessing adversarial examples"></a>Explaining and harnessing adversarial examples

Goodfellow 在这篇 Paper 中对 Adversarial Examples 的原理以及危害做了进一步的阐释，并且提出了一种计算扰动和利用对抗样本进行训练，从而提升模型鲁棒性的方法。并且，Goodfellow 认为 Adversarial Training 实际上是一种 Regularization 的手段，并且性能优于 Dropout。

为什么小小的扰动会产生巨大的影响：Feature 的精度有限，如果扰动 η 小于精度的话，模型的预测不应该出现误差。但因为我们的模型存在一个 W （NN 习得的权重），而 W *x\_perturbation = W* x + W \* η ，那么扰动就会被放大，而且空间维度越高，其产生的影响就越大。

如何计算扰动： fast gradient sign method，计算 loss 函数对 x 的导数取 sign（为了数值上的 smooth），再乘上一个我们设定的扰动大小 norm 即可。

## <a href="#The-limitations-of-deep-learning-in-adversarial-settings" class="headerlink" title="The limitations of deep learning in adversarial settings"></a>The limitations of deep learning in adversarial settings

Paper 提出了 Adversarial Samples（Examples） 所需要达到目标，并进行分级。

Adversarial Goals:

1.  Confidence Reduction：降低 model 对输出的信心，导致分类 Ambiguity
2.  Misclassification：错误分类，model 无法正确输出 input 的 label
3.  Targeted Misclassification：指定错误分类，产生样本使得模型能够输出指定的错误 label
4.  Source/target misclassification：使得一类输入都产生同一种输出

也对产生 Adversarial Samples 方法，根据所需知识进行了分级:

1.  Training data and network architecture：这种方法知道 model 的输入、模型的架构和参数以及 loss 的设置，这是最强的一种 adversarial
2.  Network architecture：能够知道 model 的架构和参数，足以来模拟模型
3.  Training Data：能够从 input data 中采样一部分数据来生成对抗样本，但是不知道模型的架构
4.  Oracle：仅仅能够通过一个模型的代理来获知模型输入和输出，能通过改变输入来观察所引起的输出的改变（作者打了个比方：在密码学里，能够看到明文和解密之后的文本，以及对应的变化）
5.  Samples：只能看到输入和输出这样的 pair 数据，不能和 Oracle 一样来尝试改变 input 获取 output 的变化

还提出了一种和 Goodfellow 不一样的 Adversarial Training 方法，通过 forward derivative（Jacobian，directly providing gradients of the output components with respect to each input component），来产生 Adversarial Examples。

## <a href="#Crafting-Adversarial-Input-Sequences-for-Recurrent-Neural-Networks" class="headerlink" title="Crafting Adversarial Input Sequences for Recurrent Neural Networks"></a>Crafting Adversarial Input Sequences for Recurrent Neural Networks

在上一篇文章的基础上，将计算 Adversarial Examples 的方法由图像领域迁移到文本领域，对 RNN （LSTM）产生 Adversarial Sequence。

通过替换输入句子的第 i 个词，替换的要求有：

1.  替换词来自我们的字典 D
2.  替换词和原词的不同 （通过 sgn 函数来衡量）和 sgn（ 模型输出 logits 对 原词的偏导数），尽可能接近

上面的偏导数的含义：

> This gives us a precise mapping between changes made to the word embeddings and variations of the output of the pooling layer

结果: 在 Sentiment Classification 中，对于平均长度为 71.06 的句子，平均替换 9.18 个词语能够达到使原 model 完全无法正确预测（error 100 %）

## <a href="#Distillation-as-a-defense-to-adversarial-perturbations-against-deep-neural-networks" class="headerlink" title="Distillation as a defense to adversarial perturbations against deep neural networks"></a>Distillation as a defense to adversarial perturbations against deep neural networks

提出了一种防御机制“Defense Distillation”来降低 Adversarial Examples 对 DNN 的影响。

该机制核心思想是先通过一个 NN 来产生一个 Probablity Vector 作为第二个 NN 的输入 Label，其中包含的概率信息能够起到以下作用：

> Training a network with this explicit relative information about classes prevents models from fitting too tightly to the data, and contributes to a better generalization around training points.

作者举了 MNIST 的例子：如果数字是 7，那么对应的 label 就是 \[0, 0, 0, 0 ,0 , 0, 0, 1,0,0\]，而我们的 probabilibty vector 则可能是 在 1 上是 0.4 在 7 上是 0.6，这就给我们的第二个 NN 一种提示，1 和 7 间有某种联系，他会尝试学到一些 1 和 7 共同的结构（比如那一竖），从而避免过于相信数字是 7，而在一些写的比较潦草的数字上做出错误的判断。

这样，第二个模型得到的 F\_star 就能够对 Adversarial Samples 更加 Robust，但是在一部分模型上会带来一部分 accuracy 的损失。

作者还提出了产生 Adversarial Samples 的 Framework:

1.  Direction Sensitivity Estimation: evaluate the sensitivity of class change to each input feature 能够找到一个 data mainfold 上导致 class change 的最为敏感的方向
2.  Perturbation Selection : use the sensitivity information to select a perturbation delta-X among the input dimensions

定义了 DNN Robustness :

1.  Display good accuracy inside and outside of its training dataset
2.  model a smooth clssifier function F which would intuitively classify inputs relatively consistently in the neighborhood

衡量 Robustness 的一个 metric：使分类器产生不同分类结果所需进行扰动的平均大小。
