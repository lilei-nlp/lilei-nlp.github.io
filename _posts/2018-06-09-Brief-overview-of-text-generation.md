---
title: "Brief Overview of Text Generation"
date: 2018-06-09
categories:
  - blog
tags: []
---

最近走马观花地读了关于一些 Text Generation 的文章，梳理一下脉络。

## <a href="#RNN" class="headerlink" title="RNN"></a>RNN

记得在 Coursera 以及各大深度学习框架都有 RNN 做 char 级别的莎士比亚风格的文本生成 Demo，也算是一个非常入门的模型了。<a href="https://arxiv.org/pdf/1308.0850.pdf" target="_blank" rel="noopener">Generating Sequences With Recurrent Neural Networks</a> 提出了用 RNN 来做 Text 的生成任务，因为其 Intuition 很符合我们的认知，**后面的文字基于已经生成的文字而产生**。

事实上，我们在用 RNN 建模一个语言模型：

$ P(s) = P(x\_1) P(x\_2|x\_1) … P(x\_t|x\_1, .. , x\_{t-1}) = \Pi\_{t=1}^{T} P(x\_t|x\_1, .., x\_{t-1})$

取 log 之后就变成相加：

$ log P(s) = \sum\_{t=1}^{T} log P(x\_t|x\_1,…,x\_{t-1}) $

之前的一些 n-gram 模型则会假定当前的词 $$x\_t$$ 只和其附近一定范围的词语有关，通过条件概率进行一定的简化，这里就不展开，因为现在的 RNN 以及其变种能较好的解决的长距离的依赖问题，所以效果自然比简化的版本要好。

在训练这个模型的时候，我们会去 maximize 这个句子在 corpus 中出现的概率，也就是 minimize 整个句子的负 log：

$L(x) = -\sum\_{t=1}^{T} log P(x\_t|x\_1,…,x\_{t-1}) $

一般地，我们会随机从预料中取一个句子的开头作为第一个 token，以这个句子的每个 token 作为 groud-truth，来进行 loss 的计算。但是这样会出现一个问题，就是如果我们一个词生成的不是很好，会导致后面生成的词误入歧途，整个句子就报废了，降低了某些本来是正确的词出现的概率，从而导致训练非常不稳定，类似**一颗老鼠屎坏了一锅粥**。所以，Teacher Forcing 被提出，也就是，每个词的生成会基于 groud-truth 生成，而不是根据模型自己之前生成的词，这样子，即使中途跑偏也能被“老师“纠正回来。

但是呢但是，Teacher Force 也有一个问题，就是 Train 和 Inference 阶段的不同：**Train 阶段是有 ground-truth 给你纠偏，但是 Inference 的时候没有**，这下就歇菜了。就好比平时考试都有老师站在边上看你做题，一做错就告诉你正确答案，到高考了结果就凉透了，因为你可能发现**有一些题你从来没有见过**，导致一败涂地。这种依赖叫做 **Exposure Bias**，随后呢，有人提出了解决方案来：既然没有正确答案不行，那就抛硬币决定要不要给你正确答案，并且越到后面给你正确答案的概率越小，即 <a href="http://papers.nips.cc/paper/5956-scheduled-sampling-for-sequence-prediction-with-recurrent-neural-networks.pdf" target="_blank" rel="noopener">Scheduled Sampling</a>。

你以为到这里就完了吗，没有。<a href="https://arxiv.org/pdf/1511.05101.pdf" target="_blank" rel="noopener">How (not) to Train your Generative Model: Scheduled Sampling, Likelihood, Adversary?</a> 指出 Schedule Sampling 是一个不一致的方法，简而言之：Maximize Likelihood 可以视作是 minimize 两个分布 P 和 Q 的 KL-Divergence，而前提是两个分布相同时 KL-Divergence最小，但是 Schedule Sampling 的 KL-Divergence 的最小值并不是 P = Q。而为什么它能起作用呢：

> Scheduled sampling works by pushling models towards a trivial solution of memorising distribution of symbols conditioned on their position in the sequence.

说人话就是：治标不治本。

另外，除了 exposure bias，还有个比较 trival 的问题：逐字的 loss 是不是不太符合我们的习惯，一般都是以**句子**这个粒度来评估生成的质量，有个别错别字可能无伤大雅呢？有待以后再讨论。

<span id="more"></span>

## <a href="#Seq2Seq-Architexture" class="headerlink" title="Seq2Seq Architexture"></a>Seq2Seq Architexture

如果把机器翻译也看作是一种文本生成任务的话，那么 Seq2Seq 也是非常经典的生成架构。一般会用两个网络，一个用于编码（Encoder），将一些信息，比如待翻译的文本，经过编码之后形成一个 semantic representation 交给另一个解码网络（Decoder），解码网络根据编码网路结果进行目标文本的生成，比如生成对应的翻译文本。

如果没记错的话 Seq2Seq 首先是在 Machine Translation 提出来，就是那篇 Attention 的大作<a href="https://arxiv.org/pdf/1409.0473.pdf" target="_blank" rel="noopener">《Neural Machine Translation By Jointly Learning To Align and Translate》</a>，之后被生成任务所采用。Attention 之所以会在机器翻译任务中提出来，也是很直觉的：**经过编码之后的语义表示向量并不足够支撑较长文本的翻译**，并且和人类翻译的过程中类似，我们在翻译的某句话的个别词汇的时候，会**特别**关注其周围的一些词，来辅助我们的翻译，这种关注也就是一种注意力机制，更多的可以参考我之前的一篇 [Blog](http://tobiaslee.top/2018/05/13/Attention-Notes/)。

回到 Seq2Seq 对于文本生成任务，事实上可以把上面的语言模型少做修改，也即我们建模的是一个条件概率：$$ P(s | c)$$，这里的 $c$ 在机器翻译任务中可以视作是待翻译的文本，在 Image Caption 任务中就可以看做是 Image 所携带的信息。和单纯的 RNN 模型相比，我们可以把 Decoder 就看做是先前的 RNN，而 Encoder 则是 **Seq2Seq 提供了的给 model 先验信息的窗口**，从而能够在特定的任务上有很出色的表现。摘掉 Encoder，就退化成 RNN Generator，所以 Seq2Seq 并没有解决 RNN 模型遇到的问题。

Seq2Seq 有很多基于模型架构微调的工作，但我看到基本都是类似在生成文本之后再进行修改的作品：

1.  推敲网络：<a href="https://papers.nips.cc/paper/6775-deliberation-networks-sequence-generation-beyond-one-pass-decoding.pdf" target="_blank" rel="noopener">Deliberation Networks: Sequence Generation Beyond One-Pass Decoding</a>，模仿人类写作，在初稿完成之后再进行润色。想起那经典的”僧推月下门“和”僧敲月下门“的故事。
2.  编辑网络： <a href="https://arxiv.org/pdf/1805.06064.pdf" target="_blank" rel="noopener">Paper Abstract Writing through Editing Mechanism</a>，思路和上面类似。

## <a href="#GAN-and-VAE" class="headerlink" title="GAN and VAE"></a>GAN and VAE

前面提到，RNN 和 Seq2Seq 的 maximize likelihood 的训练方式存在问题，并没有得到很好的解决。现在我的觉得，很多机器学习问题，归结下来都是一个模拟概率分布的问题。想通过某些手段来 recover 一个已知的分布 P，除了最大似然以外，GAN 通过 minimize 两个分布的 JS-divergence，VAE 则是 KL-divergence。GAN 和 VAE 各有优缺点，Eric P. Xing 组有一篇 <a href="https://arxiv.org/pdf/1706.00550.pdf" target="_blank" rel="noopener">On Unifying Deep Generative Models</a> 从 Adversarial Domain Adaption 的角度来将 GAN 和 VAE 统一起来的文章，当初学的时候就觉得两个模型 minimzie 的东西挺像的，虽然一个是生成器和判别器打架，另一个是 reconstruction loss（如果我没记错的话）。

GAN 用在文本领域，在我之前的笔记里也都有提到，主要是通过强化学习来解决梯度传递的问题；

VAE 用于文本生成的思路是这样的：将真实的文本交给一个 Encoder，编码成 latent vector $z$，然后通过这个 $z$ 来经过 Decoder 重构输入的文本，感觉和 Seq2Seq 也有点类似，鉴于 GAN 似乎已经大量地取代 VAE，所以也就略读了几篇主要的 Paper。

1.  [Generating Sentences from a Continuous Space]()：提出用 VAE 做文本生成
2.  <a href="https://arxiv.org/pdf/1702.02390.pdf" target="_blank" rel="noopener">A Hybrid Convolutional Variational Autoencoder for Text Generation</a>：对 Encoder 和 Decoder 的网络结构进行了修改，采用 CNN 和 RNN 的混合。
3.  <a href="http://proceedings.mlr.press/v70/hu17e/hu17e.pdf" target="_blank" rel="noopener">Toward Controlled Generation of Text</a> ：生成属性能够控制的文本（比如情感极性、句子的时态），通过在 $z$ 中挖出一块来编码语义要求信息，增加相应的 loss 来引导 model 生成我们想要的文本。
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">GAN</a>
