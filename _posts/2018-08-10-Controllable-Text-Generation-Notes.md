---
title: "Controllable Text Generation Notes"
date: 2018-08-10
categories:
  - blog
tags: []
---

## <a href="#Controllable-Text-Generation" class="headerlink" title="Controllable Text Generation"></a>Controllable Text Generation

文本生成，包括机器翻译、文档摘要、Image Caption 在内，是一个非常广泛的问题，而这篇文章所讲述的文本生成，并不是类似机器翻译一样有具体的目的，有 x-y pair 的监督学习任务，而是侧重于无监督的生成模型。而这生成模型是**可控**的，即并不是仅仅根据 corpus 进行学习语言模型进行分布的模拟（SeqGAN），同时我们也希望能够对于生成的文本具备一定的控制能力，例如，我们可以控制生成的文本和某个主题相关，又或者是具有某种情感极性，在 charbot 这样的场景下，能够做到这到这一点是非常重要的。本文从三种模型架构出发，介绍关于可控文本生成的一些工作。  
<span id="more"></span>

## <a href="#GAN-based——SentiGAN" class="headerlink" title="GAN-based——SentiGAN"></a>GAN-based——SentiGAN

基于 SeqGAN 框架的文本生成工作有很多，但之前的工作主要关注点在于解决 SeqGAN 生成文本过短、mode collapse 问题等问题，而 SentiGAN 则是基于 GAN 框架少有的可控文本生成的工作。

很有幸邮件和 SentiGAN 的作者<a href="https://nrgeup.github.io/resume/" target="_blank" rel="noopener">王科</a>学长交流过，并且还面基蹭了一顿饭。<a href="https://www.ijcai.org/proceedings/2018/0618.pdf" target="_blank" rel="noopener">SentiGAN</a> 这篇文章也被评为了 IJCAI 的 Distinguished Paper，这篇文章的 motivation 是基于 GAN 框架来进行带有情感极性的文本生成，模型结构如下：

![SentiGAN](/img/sentigan.png)

模型的一大亮点是在于使用多个 Generator 来负责各种情感极性的生成，也即我们可以选用不同的 generator 来生成不同极性的文本，达到控制的目的。以及相应地利用判别器来判别生成的文本是否具有对应的情感极性，这和原先 SeqGAN 中只负责判断文本的真假与否相比，对生成的文本提出了更高的要求。另外一大亮点是 Generator 的 reward 的设计上，采用了 **Penalty based objective function** ：

![Penalty based Objective Function](/img/pbof.png)

原先 SeqGAN 中，Generator 希望使判别器认为自己生成文本为真的概率最大化，并且使用的是 log probability，这就是直接将 MLE 的梯度根据 reward 值进行加权；而在 SentiGAN 中，首先去掉了 log，并且是希望将判别器认为自己为假的概率最小化。作者认为这样能够缓解 mode collapse 问题：去掉 $\log$，这和 WGAN 中 的设置类似，作者也在论述中提到了这一点，不过比较奇怪的是，作者开源的<a href="https://github.com/Nrgeup/SentiGAN" target="_blank" rel="noopener">代码</a>中并没有梯度的限制（WGAN 的一个实现 Lipschitz 条件的手段），不过问了学长，说是代码还会再更新，日后再看。接着没有让 Generator 来最大化判别器判断其为真的概率，而是最小化判别器判断其为假的概率，称之为 Penalt-based Obejctive Function，作者给出了简单的一个证明：

$ G(X|S;\theta\_g) V(X) = G(X|S;\theta\_g)(1-D(X;\theta\_d) )=G(X|S;\theta\_g) - G(X|S;\theta\_g)D(X;\theta\_g)$

并认为这样会让 generator 偏向于一个更小的 $G(X|S;\theta\_g) $ ，即概率比较小的句子，从而避免生成很多重复（概率比较大）的句子。**就我个人的实验效果来看，确实对 mode collapse 有很大缓解，但究竟是去 $\log$ 还是 Penalty 带来的效果有待细考**。

## <a href="#C-VAE" class="headerlink" title="C-VAE"></a>C-VAE

<a href="http://arxiv.org/abs/1703.00955" target="_blank" rel="noopener">Towards Controlled Generation of Text</a>，是我见到的第一篇把 Controlled 写进题目中的文本生成文章。作者基于 VAE 进行可控的文本生成，模型架构如下：

![C-VAE](/img/cvae.png)

VAE 做文本生成的步骤简要说明如下：

- 将原句 $x$ 通过 Encoder 进行编码，得到 latent variable 表示 $z$ （通过 mean value 和 std 生成）
- 将 $ z $ 交给一个 Generator 进行文本生成，得到 $\hat x$
- 根据 $x$ 和 $\hat x$ 计算重构 loss，来更新 Generator 和 Encoder

这和 Seq2Seq 结构的 Encoder-Decoder 结构是非常类似的，但 VAE 一个比较好的性质就在于我们可以通过操作 $z$ 来变化输出的结果，我个人的理解就是讲文本映射到一个高维的语义空间中，而在这个空间中文本的分布是近似连续的：比如我们可以通过 $z$ 的线性变化实现文本极性的转变。而这种连续性这就为控制文本提供了空间，于是作者在隐变量 $z$ 表示上挖了一个空叫做 strucutred code $c$，并且认为 $c$ 这一部分：

> without entangling with other attributes, especially those not explictly modeled

换句话说，句子的某些属性（例如情感极性、时态）只和 $c$ 有关，$c$ 一样，则这些属性就是一样的。而为了让生成器能够生成具有相应属性的文本，就需要在原有的 Loss 上加上和属性相关的项。作者从两方面施加了限制：

1.  生成的文本具有特定的属性，这很简单，再训练一个分类器来判断是否具有这样的属性即可，这里的分类器就称之为 Discriminator，定义的 attribute loss $L\_{attr, c}$ 如下：

    ![Attribute Loss-c](/img/attr_c.png)

2.  $z$ 和 $c$ 的功能尽可能分离，即 $c$ 负责文本的属性，而 $z$ 则需要建模文本的其他方面，比如我们不感兴趣的属性以及语义的通顺性等。我们希望根据生成的句子能够恢复出 $ z$ 来，定义的 attribute loss $L\_{attr, z} $ 如下：

    ![Attribute Loss-z](/img/attr_z.png)

结果的 loss 就是三者之和：

$ L\_{G} = L\_{VAE} + \lambda\_c L\_{attr, c} + \lambda\_z L\_{attr, z} $

通过 minimize 这个 loss，达到上述的目的。

## <a href="#Seq2Seq" class="headerlink" title="Seq2Seq"></a>Seq2Seq

Seq2Seq 也是一种广泛应用的生成模型，在机器翻译上有很多作品，框架也基本上是 `Encoder-Decoder + Attention` 。在这介绍两个问题：根据属性生成评论以及根据主题生成回复。

### <a href="#Learning-to-Generate-Product-Reviews-from-Attributes" class="headerlink" title="Learning to Generate Product Reviews from Attributes"></a>Learning to Generate Product Reviews from Attributes

作者提出了一个比较有趣的问题，过去我们关注的是从评论中挖掘情感，比如商品的质量的好坏，但反过来，根据商品的属性生成对应的评论，很少被大家所关注（可能也没什么用，（逃 ）。模型的架构如下：

![Attributes To Review](/img/a2s.png)

- Encoder：作者利用一个全连接层，对评论的属性（用户 id、产品、评分）进行编码，将编码后的 vector 拼接后经过 tanh 层作为 Decoder 的 initial state。
- Decoder：作者使用了 MultiLayer-LSTM 作为 Decoder，并将最后一层 LSTM 的输出作为最终的输出用于 review token 的生成
- Attention：考虑到 Encoder 编码的属性信息在 LSTM 传递过程中可能逐渐衰减，为了提高信息的利用率，作者引入了 Attention 机制，在生成 token $y\_t$ 的时候，将之前的属性的向量经过 attention 得到 $c\_t$，并将 $c\_t$ 和 $h\_t$ 做一个加权之后经过 tanh 层得到最终的 hidden state $h\_t^{attr}$，通过 softmax 层得到词表分布，采样得到词表分布

问题：作者的衡量指标使用的是 BLEU，而 BLEU 事实上并不是一个非常好的衡量指标（不过也没能找到更好的），上次听 William Wang 老师报告时他举了一个例子：

> We had a great time to have a lot of the. They were to be a of the. They were to be in  
> the. The and it were to be the. The, and it were to be the.

这个句子的 BLEU 值可能很高，但是并不 make sense。所以，**文本生成的衡量指标**是一个非常值得深究的话题，也只有找准了这个 target，进一步的长文本生成才能有的放矢。

### <a href="#Topic-Aware-Neural-Response-Generation" class="headerlink" title="Topic Aware Neural Response Generation"></a>Topic Aware Neural Response Generation

这篇文章关注的是 Chatbot 中一个另一个问题：如何从输入中捕获主题信息，并据此进行回复的生成。作者举了一个例子：

> Input: My skin is dry
>
> Output: Me too.
>
> Human: You may need to hydrate and moisturize your skin.

`Me too` 这样的词在之前 Jiwei Li 的文章中也有提到，是一种 trival 的回复，万金油；但如果我们希望微软小冰能够做到和人一样，告诉你需要补水，那么就必须从输入中挖掘出 `skin` 这个主题，并据此产生回复。模型的架构如下：

![TA-Seq2Seq](/img/ta-seq2seq.png)

如何获得输入中的主题呢？通过主题建模 LDA 的方法，抽出主题词来。作者用了 Twitter LDA 模型来实现这一过程。具体到三个组件：

- Encoder: 首先是对输入的句子进行编码， 作者采用的 Bi-GRU 作为 Encoder，并将两个方向的 $h\_t$ 拼接在一起作为 hidden state；拿到抽出的主题词之后，进行 embedding，准备交给 Decoder

- Decoder 和 Attention: 这里的 Attention 作者称之为 Joint Attention，分为两个部分：

  - Message Attention: 就是我们常用的 Attention，得到一个 Context Vector $c\_t$
  - Topic Attention: 作者将 Topic Embedding 做了一个 Attention 之后构成一个 $o\_t$，而这个 Attention 的权重的计算是根据 topic embedding 、前一个输出的词、最后一个 hidden state $h\_T$计算得到。和传统的 Attention 对比，作者认为加入 $h\_T$ ，其代表的是语句的整体信息，能够对主题词权重计算起到帮助作用，比如减少不相关主题的权重和增加相关主题的权重。

  最后，$c\_t$ 和 $o\_t$ 参与到 $s\_i$ 的计算： $ s\_i = f(c\_i, o\_i, y\_{i-1}, s\_{i-1}) $

- 词表分布的计算：作者又在最终的词表分布上根据主题词进行一个类似偏置操作（真是花里胡哨），$p(y\_i) = p\_V(y\_i) + p\_K(y\_i)$，前者就是我们根据 $s\_i$ 和 $y\_{i-1}$ 计算得到的词表分布，而后者则再一次利用主题词的信息，对词表分布进行一个修正，但比较奇怪的一点是，作者再一次利用了 $c\_t$ ，但我认为 $c\_t$ 并没有携带 Topic 的信息。**或许是希望考虑 $s\_t$ 中的主题词信息和 $c\_t$ 的关联？**

问题：作者做了三个实验，但并没有针对其 Claim 设置相关的支撑实验，比较侧重于几个指标提升，我觉得并不是很信服，而且这个 model 在词表分布上的 bias 的设置，让我很怀疑前面一系列设置是否有意义？

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

这篇文章拖了很久，总算是完成了。而在整个阅读过程中，我也感受到读 Paper 的速度变快，以及关注点的改变：以前重理解模型和关注其效果，现在的关注点不在于指标多高，而在于作者为什么这么设置模型，会去看作者的 motivation 和 claim，以及实验的设置能否支撑起他的说明。也愈发的感受到，写出一篇好文章和灌水文章的差距。加油！

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">Deep Learning</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">GAN</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">Text Generation</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2018/08/10/Controllable-Text-Generation-Notes/)