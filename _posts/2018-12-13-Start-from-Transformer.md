---
title: "从 Transformer 说起"
date: 2018-12-13
categories:
  - blog
tags: []
---

从我开始学习 NLP 以来，带有轰动效应的 Paper 大概就是 Transformer 和 BERT，前者是在机器翻译领域全面超过原先的 Seq2Seq，而后者则是继 Word Embedding 作为 representation 之后的又一突破。而实际上 BERT 的提出也是建立在 Transformer 的基础上。这篇文章就是围绕这两篇 Paper 的笔记以及一些思考。就从 Transformer 说起吧！

<span id="more"></span>

## <a href="#Transformer-的细节" class="headerlink" title="Transformer 的细节"></a>Transformer 的细节

Transformer 的 model，主要就在于 Multi-Head Attention：

![Multi-head Attention](/img/mh-attn.png)

而在这之前文章先向我们介绍了 attention weight 的两种计算方法：

1.  最初的注意力机制中，利用一个 feed-forward network，也即 Bahdanau 提出的 alignment function：$a(s\_{i-1}, h\_j)$
2.  点乘式计算：通过 Query 和 Key 计算一个 Dot-product（可以认为是计算关联度），然后 Softmax 得到权重

二者方式对比：

1.  二者复杂度接近，点乘方法可以通过矩阵计算的优化来实现快的速度
2.  点乘可能因为某个位置的元素过大而在 softmax 之后进入梯度的死区，即没有梯度，因此作者通过一个 $\sqrt{d\_k} $ 做了 scale 来避免这样的情况

回到正主 Multi-head attention，先是 single head 的计算方式：

$ Attention(Q, K ,V) = softmax( \frac{QK^T}{\sqrt{d\_k}} )V$

Multi-head 就是重复以上过程多次，但是每次先对 $Q$，$K$ 和 $V$ 做一个线性变换，也就是乘个矩阵：

![Multi-Head Attention Formula](/img/mh-attn2.png)

美名其曰：

> Multi-head attention allows the model to jointly attend to information from different representation subspaces at different positions.

在我看来，上面的解释有点牵强，但是也勉强说服我了，反正你算的快多算几次总是有好处的…

而在应用过程中，有下面一些细节：

- Decoder 可以对 Encoder 的所有输出进行 Attention，和 Seq2Seq 中的做法一样
- Encoder 内部有 Self-Attention，即能够 Attend 到之前 layer 所有的输出，并且没有**位置的限制**，即某个位置可以 attend 所有位置
- Decoder 内部也有 Self-Attention，即能够 Attend 到之前 layer的输出，但是只能是当前位置以及之前的输出，这很好理解，翻译的时候后面的还没翻译出来呢。怎么实现，在 softmax 的输入上做一个 mask，把不能 attend 位置都 mask 成 $-\infty$ 就 ok

另外一个比较有趣和难理解的点就是 Positional Encoding，有趣是在于因为之前的 Attention 机制都没有考虑到序列的时序信息，因此作者在 Embedding 上额外增加了两个 sin 和 cos 信号：

$PE\_{(pos, 2i)} = sin(pos / 10000^{2i / d\_{model}})$

$PE\_{(pos, 2i+1)} = cos(pos / 10000^{2i / d\_{model}})$

我第一遍读的时候也是很困惑，为什么这样能够增加时序的信息？在学习过信号与系统之后，尝试解释一下：首先，PE 的变量有两个，分别是 pos 以及维度 i，对于某一个位置的某一个维度，其三角函数信号的 $\omega$ 是唯一的，而利用傅里叶变换将频率转到时域，其就是一个冲击信号，冲击信号的位置也因 $\omega$ 的唯一而是唯一的，所以每个位置每个维度都有自己的独特的值，这样就能够引入时序的信息（不同pos冲击信号的位置不同，加的不一样）。

机器之心上也有对此的讨论，摘取一个理解以供参考：

> 这个 position embedding 设置得很有意思，我尝试解释一下：每两个维度构成一个二维的单位向量，总共有 d\_model / 2 组。每一组单位向量会随着 pos 的增大而旋转，但是旋转周期不同，按照论文里面的设置，最小的旋转周期是 2pi，最大的旋转周期是 10000 x 2pi。至于为什么说相邻 k 步的 position embedding 可以用一个线性变换对应上，是因为上述每组单位向量的旋转操作可以用表示为乘以一个 2 x 2 的旋转矩阵。

## <a href="#Transformer-的能与不能" class="headerlink" title="Transformer 的能与不能"></a>Transformer 的能与不能

前面讨论了 Transformer 的模型上的一些，事实上还有很多训练的 trick，就不再一一赘述了。进一步地，来看看 Transformer 给我们带来了什么，首先是计算效率的大大提高：

![Comparison](/img/comparison_transformer.png)

上面这张表给出了几种 Model 的对比，序列操作数量 RNN 毫无疑问是 O(n)，再来看看剩下两种：

1.  Maximum Path Length：这一点是序列中两个元素交互（interaction）的路径最长长度，对于 RNN 来说，句首的信息要传递到句尾，需要经过 n 次 RNN 的计算；而 Self-Attention 可以直接连接任意两个节点；CNN 的交互则类似一颗 k 叉树（卷积核大小 k）。下面这张图更为形象：

    ![Comparison2](/img/comparison2.png)

2.  每层的复杂度：为什么 Self-Attention 是 $O(n^2*d)$ 呢，借助上面的图：每个位置的 token 都需要对其他位置的 token 计算一次 d 维的 representation，而为什么这里的 RNN 是$O(n*d^2)$ 呢，因为这里考虑的是多层 RNN 堆叠的情况，那么对于一层中的一个 token，其接收到本层的某个位置信息的操作是 $O(n\*d)$，而对于其之前层产生的信息则相当于再进行了一次 hidden state 的计算，需要额外乘上 d，因此是 $d^2$

这张表的告诉我们：Transformer 因为其可以并行，并且在限制 attention 的范围之后计算效率很高。除了计算效率高以外，Transformer 因其 node 之间交互的路径较短，长距离的依赖信息丢失的问题相比 RNN 会好很多，因此可以通过很多手段把 Transformer 做的很**深**，从而带来在 NMT 问题上 state-of-the-art 的效果。网络深了带来的好处就是模型容量变大，可以捕获到更多的细节，文本的表示也能够因此变得丰富，这一点也就是后续 BERT 工作的基础；但是一个很深的网络的 training 除了计算量以外，也是很难调的，所以文章也有非常复杂的 learning rate 调整公式。

总结一下：Transformer 可以提供高效率的计算，更好地捕获文本的表示。

以上这两点应该就是 Transformer 带来的主要好处了，然后有人就会问，那么 RNN 是不是就可以被取代了呢？有研究者就开始思考 RNN，或者说这种循环结构能够带来什么好处？[\[1\]](#Reference) 中就对比了 RNN 结构和 Transformer 捕获文本层次结构的能力。文章用了两个问题来研究所谓的文本层次结构（Hierarchical 这个东西在文本中到底指什么？有待进一步的补一些语言学的知识）：主谓一致和逻辑推理。主谓一致任务就是我们经常做的语法题，让你选一个词来使其的单复数形式和主语相符；而逻辑推断则是使用简单的一些人工定义语言来进行一些关系的推断，感兴趣的可以参考原 Paper。来看看在主谓一致任务上二者的表现：

![Transformer v.s RNN](/img/fan_rnn.png)

从图中可以看出，在这个任务上 RNN 可以说是完胜 Transformer 的，尤其是当任务变得困难的时候：主语和谓词的距离变长，attractor 的数量增加（可以认为是用于混淆的主语）等都会拉大二者的差距。文章的解释是 LSTM 被强迫压缩句子的信息到一个 vector 中，因此能够更好地捕获结构信息；而 Transformer 能够获取所有的历史信息，所以可能在这方面有所忽视。但是，这里的 head 数目最大为 4，不知道增加 multi-head 的数量会不会对 Transformer 的性能有所帮助。

在逻辑推断上二者的表现也同样是 RNN 优于 Transformer，作者也基于此提出了循环结构可能是捕获层次化信息的一个关键，但是不要笃信这个结论，说不定调调 Transformer 的参数性能就上去了。但这依旧是一个值得探索的方向，探索 RNN 相比 Transformer 更擅长哪些任务。

另外，[\[2\]](##Reference) 中对 Transformer 中用到的很多 trick 做了详尽的 study，把其中的各个部件进行任意组合，弥补了 Transformer 缺少 Ablation 实验的遗憾。文章也提出了三个 Key Point：

1.  Source 端对 Encoder 最下面的几层做 Attention 没什么用：可能因为下层的表示还都太浅，无法捕获高层语义信息
2.  多头 Attention 和 Residual Connection 是 Transformer Work 的关键
3.  相比 Decoder 端，Source 端的 Self-Attention 起到的作用更大

同时文章也提到，RNN 的结构能够从 Residual Network 和 Multiple Souce Attention 获益很多，或许可以在 RNN 上采用一些 Transformer 的组件，也是一个值得探索的方向。

## <a href="#BERT-之前" class="headerlink" title="BERT 之前"></a>BERT 之前

深度学习可以认为由两部分组成：表示学习（如何表示数据）和归纳偏好学习（根据特定的场景利用特定的 feature，引导 model），而 BERT 则就可以说是 NLP 表示的新以里程碑。但在这之前，其实就有类似的工作了，比如 ELMo，NAACL 2018 的 Best Paper，ELMo 和 BERT 差的也就是个 Transformer，所以可以看做是 Transformer 和 BERT 的桥梁，在 讨论 BERT 之前，先来看看 ELMo 究竟做了什么。

![ELMo](/img/elmo.png)

*图来自 [4](##Reference)*

一言以蔽之：利用一个多层 biLSTM 训练 Language Model 得到各个 word token 的 hidden state，将其拼接起来作为表示。

Language Model 的任务就是根据前面的词预测后的词，文章用了一个双向语言模型，即也考虑从后向前的预测，来获取更多的下文信息；这里的多层可以如图中所示，可以认为是考虑不同层面的信息：语义、句法特征；实际使用的时候再结合具体任务进行 scale 以及加权：

![Scale and Weighted Sum](/img/elmo_formula.png)

**不同的任务对于词表示的特征，是有不同的需求的**，这就是我们之前说的归纳偏好（inductive bias），例如，sentiment analysis 可能就会对词的语义层面信息更加关注，这个词是积极的还是消极的；而生成词的时候会考虑一些句法上的信息，这个词的词性是什么？利用这个不同层表示的加权和，可以灵活的调整模型的归纳偏好，从而适应下游的任务。

带来什么好处呢？主要有两点：词的表示携带更多的信息以及解决词的多义问题。前者，可以提升各种 NLP 任务的性能：

![Improvement ELMo](/img/elmo_impv.png)

而后者，正是因为词的表示是 deep contextualized 的，因此能够和具体语境结合，表达词的不同含义：

![Polysemy ELMo](/img/elmo_poly.png)

如此 fancy 的结果，也难怪能够斩获 NAACL 的 Best Paper。如果我们把 biLSTM 换成抽特征能力更强的 Transformer，是不是又会更上一层楼呢？于是，BERT 应运而生。

## <a href="#BERT-的细节" class="headerlink" title="BERT 的细节"></a>BERT 的细节

BERT 对于前辈 ELMo 的 Language Model 有些不满意，认为其只能单向预测 next token 限制了 Model 抽信息的能力，并且没有捕获 sentence level 的关联信息，因此提出了下列两个任务来进行 pre-train：

1.  Masked LM：随机把句子中的一部分（15%） token 替换为 `[MASK]`，让 Model 来预测这个 token 原本到底是什么。在实作上有一些在替换的时候用的trick 来避免之后进行下游任务时 model 完全没有见过 token 的情况：

    1.  80% 的概率，正常替换
    2.  10% 的概率，保持 token 不变
    3.  10% 的概率，把 token 替换成词表随机的 token

    这样的做能够强迫 model 保持对每个 Token 的上下文都有建模，从而产生一个好的表示，否则很难正确预测被 mask 的 token；另外，也因为每次只 mask 了 15%，带来收敛速度的下降。

2.  Next Sentence Prediction（NSP）：NLP 的有些任务需要考虑的是句子间的 relation，例如 QA，NLI 等，这也是一种 inductive bias 的体现。为了建模这一点，BERT 将句对拼接起来，判断他们到底是不是正确的组合，并且通过随机拼接构造负例，用一个二分类任务来让模型学习到句子之间的关系。

最后，把 Mask LM 的 MLE loss 和 NSP 的 MLE loss 结合作为 pre-train 的总 loss，来达到同时获取 token level 以及 sentence level 的表示的目标。

此外，在输入的表示上，BERT 也和之前 ELMo 有所不同：

![BERT-Representation](/img/bert_repr.png)

除了 token 本身的 embedding 信息以外，还有用于句对的分割句子信息以及Position encoding 信息（这里采用的就是 learned position encoding 了，而不是 transformer 的三角函数），其中一个特别的 `CLS` 是用于分类任务的输出：

![BERT Downstream Task](/img/bert_downstream.png)

在前两个分类任务上（前者是句对分类，e.g. NLI，后者是单句分类，e.g. 情感分析），都是将这个标识的经过 BERT 得到的 representation 取出来，再加上全连接层进行分类的 fine-tune；而对于 QA 任务，则是将 paragraph 段对应的输出作为表示进行 answer 的生成，NER 任务也是类似，这两个任务都不再是单个句子的表示，而会利用每个 token 的信息。

至此，BERT 的全貌其实就已经在我们的眼皮底下：

![BERT Model](/img/bert_model.png)

当然，不可能所有的细节都被我 cover 到，最好的老师还是原文 [\[5\]](##Reference)。

## <a href="#展望未来" class="headerlink" title="展望未来"></a>展望未来

整篇文章下来，我们可以得出一个结论：**Transformer 很强**，强大之处在于其能够通过做的很深，并且考虑 token 之间丰富的 interactions 来捕获很多信息，得到一个很好的表示。而有了一个好的表示之后，如何**整合具体的 inductive bias**，例如 QA，NLI 需要的句子级别的 relation，就如 BERT 一样加入句对预测任务来引导模型去做这方面的模式识别，是很值得继续深挖的。从 ELMo 到 BERT，可以说是预训练或者说是迁移学习在 NLP 的一次胜利，相信会有更多的工作会涌现出来。正如 Ruder 在 [\[6\]](##Reference) 中所说，下一波浪潮很有可能就是迁移学习在各个领域的开花结果。

我们能做的，就是努力翻出一颗属于自己的小浪花！

## <a href="#Reference" class="headerlink" title="Reference"></a>Reference

1.  <a href="http://arxiv.org/abs/1803.03585" target="_blank" rel="noopener">The Importance of Being Recurrent for Modeling Hierarchical Structure</a>
2.  <a href="http://aclweb.org/anthology/P18-1167" target="_blank" rel="noopener">How Much Attention Do You Need ? A Granular Analysis of Neural Machine Translation Architectures</a>
3.  <a href="http://arxiv.org/abs/1802.05365" target="_blank" rel="noopener">Deep contextualized word representations</a>
4.  <a href="https://zhuanlan.zhihu.com/p/49271699" target="_blank" rel="noopener">从Word Embedding到Bert模型—自然语言处理中的预训练技术发展史</a>
5.  <a href="http://arxiv.org/abs/1810.04805" target="_blank" rel="noopener">BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding</a>
6.  <a href="http://ruder.io/transfer-learning/" target="_blank" rel="noopener">Transfer Learning - Machine Learning’s Next Frontier</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color5">Attention</a>
