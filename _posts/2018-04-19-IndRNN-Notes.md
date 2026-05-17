---
title: "从 IndRNN 来回顾 RNN 模型"
date: 2018-04-19
categories:
  - blog
tags: []
---

<a href="https://link.zhihu.com/?target=https%3A//arxiv.org/abs/1803.04831" target="_blank" rel="noopener">IndRNN</a> 是最近新提出的一种 RNN 架构，作者认为其能够更好地解决 gradient vanishing / exploding 以及具有更好的解释性。

## <a href="#从-RNN-到-IndRNN" class="headerlink" title="从 RNN 到 IndRNN"></a>从 RNN 到 IndRNN

首先来复习一下 RNN t 时刻的 hidden state 的计算公式：

![Vanilla RNN Formula](/img/vanilla_rnn_formula.png)

这个公式相信都是了然于心，如果画成图的话，就是下面这个样子：

![RNN rolled](/img/RNN-rolled.png)

*图片引用自 <a href="http://colah.github.io/posts/2015-08-Understanding-LSTMs/" target="_blank" rel="noopener">colah’blog</a>*

但是，请问一下自己，这个 A 里面是怎么样的一个结构？如果不能一下子回答出来，那么你可能和我一样，从来都不曾真正的理解 RNN。

在这个 RNN cell 的内部单元，其连接的状态是这样的：

![Vanilla RNN](/img/vanilla_rnn.png)

注意：**$h\_{t-1}$ 和 $h\_t$ 之间是一个全连接的关系**。

而 IndRNN 的计算公式呢，做了一个小小的修改：

![IndRNN Formula](/img/ind_rnn_formula.png)

从矩阵乘变成了 haramard dot，也就是 element wise 的乘法，能给 RNN 带来什么样的变化呢？

![IndRNN](/img/ind_rnn.jpg)

全连接变成了单个神经元自身之间的传递，这带来两个变化：

1.  更好地解决梯度爆炸 / 消失
2.  更好的解释性

接下来就从这两个方面入手来深入了解一下 IndRNN

<span id="more"></span>

## <a href="#梯度爆炸-消失" class="headerlink" title="梯度爆炸/消失"></a>梯度爆炸/消失

首先来复习一下 RNN 的梯度怎么计算：

假设目标函数为 $J$，我们对第 $T$ 个 hidden state $h\_T$求导，结果为：

$$\frac{\partial{J}}{\partial{h\_T}} \Pi\_{k=t}^{T-1}diag(\sigma’(h\_{k+1}))\mathbf{U^T}$$

其中 `diag` 是指对角矩阵（其中对角的元素是激活函数对 $h\_{k+1}$ 的导数），其实就是后一 hidden state 通过链式法则展开到最前的一个 hiddent state，求导的结果连乘。

矩阵的连乘可以通过对角化来简便计算，即化为 $PAP^{-1}​$，$A​$ 是一个对角阵，主对角线上的元素就是其特征值。而如果其特征值小于 1，那么在连续的乘积之后其值就会接近 0，梯度消失；如果大于 1，那么就会接近变成 NaN，梯度爆炸。解决的手段分别就是梯度裁剪（gradient clipping）和 合适的初始化 + 更换为 ReLU acitivation（目的是为了选择合适的特征值？）

LSTM 解决这一问题的思路是增加 gates，来控制信息的流动，从而较好的解决梯度消失问题（因为梯度爆炸用 clipping 能够比较粗暴地解决，而 vanishing 并不行）。LSTM 的求导比较繁琐，可以参考一下 <a href="http://arunmallya.github.io/writeups/nn/lstm/#/" target="_blank" rel="noopener">LSTM Forward and Backward</a> 。从繁琐的公式中比较难看出 LSTM 解决梯度消失和爆炸，我们可以通过下面这张图来直观的感受一下 LSTM 的作用：

![Comparision Between RNN and LSTM](/img/compare.png)

左边是 RNN，右边是 LSTM；颜色的深浅表示了梯度影响的程度。可以看到，RNN 第一个时刻的受最后一个时刻的影响微乎其微，这种情况下就可以认为是出现了梯度消失，无法较好地更新我们的参数；右边的 LSTM，为了简化起见，我们将将 input gate 设为 0（图中的 `-` 符号），forget gate 始终记忆前一状态的信息（图中的 `o` 符号），我们可以将第一个时刻的信息一直传递至我们想要的 4, 6，并且其梯度的也能够通过这一条路径成功的回传。所以，通过**控制输入以及先前状态的流动方式**，LSTM 能够较好地解决梯度消失的问题。

GRU 则在 LSTM 基础之上做了简化：

![GRU](/img/gru.png)

1.  将 Forget Gate 和 Input Gate 合并成一个 Update Gate $z\_t$，不像 LSTM 是由两个**独立**的门来控制
2.  使用一个 Reset Gate $r\_t$ 来直接控制 $h\_{t-1}$ 对 $h\_t$ 的贡献，而 LSTM 在计算 $\hat c\_t$ 的时候是没有一个 $r\_t$ 来控制 $h\_{t-1}$ 的

这样做的好处很直观地一点就是减少参数的数量，能够加快训练速度；另外 $r\_t$ 对于 $h\_{t-1}$ 的控制能够让 cell 更好的理清过去时刻状态对现在状态的影响程度，但 Update 门的不独立性又使得它的效用有所下降。我猜测如果直接在 LSTM 的基础之上加一个 Reset Gate，可能效果会更好，但参数的数量就上去了，所以这里必然存在一个性能和速度的 tradeoff。

回到 IndRNN，其对于 $h\_{n,t}$ 的梯度计算如下（t 时刻 hidden state 的第 n 个单元）：

![Gradient Of IndRNN](/img/grad_ind.png)

最大的差别就是**矩阵连乘变成了一个数的幂次**，这样我们就可以通过控制这个矩阵中元素的大小来避免梯度消失和爆炸。当然，我们也可以选择一个合适的数值范围来让让梯度更好地流动，加快训练的速度。

实验的结果也证明，IndRNN 的长期记忆（也就是梯度能够传递的时间步数）效果远好于 RNN 和 LSTM（5000 vs 500~1000）

## <a href="#解释性" class="headerlink" title="解释性"></a>解释性

因为 $h\_{t-1}$ 和 $h\_t$ 各个单元之间是相互独立的，那么就可以提供了看待 IndRNN 的两个权重矩阵的新的视角：

1.  $W$：负责提取输入的空间特征
2.  $ U$: 提取时间上的特征

而且，在这之后加一层全连接层，IndRNN 在一些条件的限制下（ two constrains : 1. linear activation; diagonalizable weight matrix）能够变成一个普通的 RNN 模型。

不过关于这一点，我觉得还需要更好地可视化的手段来帮助解释，否则这样的解释依旧停留在 Intuition 上，不足以说明问题。

## <a href="#更深的-RNN" class="headerlink" title="更深的 RNN"></a>更深的 RNN

写这篇文章的时候突然想到一个问题，为什么 RNN 里面都使用 tanh /sigmoid 而不是 ReLU，Leaky ReLU 这种现在的爆款标配 activation 呢？事实上是有的，Hinton 在 <a href="https://arxiv.org/pdf/1504.00941.pdf" target="_blank" rel="noopener">IRNN</a> 这篇文章里就尝试使用一些初始化的 Trick 和 ReLU activation 来解决梯度消失的问题，并且能够取得近似 LSTM 的效果。

对于 LSTM 来说，其中有三个门，因为门的值需要在 \[0, 1\] 之间，所以选择 sigmoid 函数，没有问题；那么 Cell State 的计算和最后的 Hidden State 为什么不尝试使用 ReLU 呢，应该也没问题，求导还快，计算也方便，但是同时也就有两个缺点，一个是**不是以 0 为中心，这个在 CS231n 中对于各种激活函数中有讲到，ReLU 虽然是 Non-saturated activation，但他的值域是大于零的** ；另外一点，就是如果使用 ReLU，那么 LSTM 的输出可能会很大。所以，这里同样存在着一个 trade off，考虑到 LSTM 提出的时间以及在各个任务上性能都还不错，替换的需求不大，所以可能也就这么用下来了。所以，使用 **ReLU 或者 Leaky ReLU** 是可行的。

既然如此，那么 IndRNN 用上 ReLU 更加没什么问题，所以作者也提出能够借鉴 CNN 中的一些方法，使用类似 ResNet 进行堆叠，得到更深的 RNN：

![Deep RNN](/img/deep_rnn.png)

对于 ResNet 的了解不多，所以这里先暂时搁置，有待后面补上。但看到 Batch Normalization 以及 ResNet，也是在提醒我们 CNN 和 RNN 相互借鉴非常重要。

## <a href="#Acknowledgement" class="headerlink" title="Acknowledgement"></a>Acknowledgement

非常感谢来自一位资深 NLP 算法工程师泼的冷水和指导，让我意识到自己的局限性。希望以后自己能够在以下三个方面继续努力：

1.  独立思考：现在养成了读二手再到原版资料（先论文解读，再看论文本身）这种习惯，前人的理解会禁锢我们的看法，并且每个人都会有自己的盲点。应该是一手资料现行，二手资料对比补充。
2.  关注新进展的同时，注重基础的沉淀：IndRNN 就是最好的例子，同时还应该更多横向地对比相关的文章来获得更全面的理解；还需要补一些语言学的知识来形成更完整的体系。
3.  代码的优化和重构：复现阶段运行优先没错，但完成之后应该回过头来重新优化和重构代码，这应该是一个计算机科班学生应有的素质，同时，重写代码也能够进一步提升 Coding 的能力。

## <a href="#References" class="headerlink" title="References"></a>References

<a href="http://www.cs.toronto.edu/~tingwuwang/rnn_tutorial.pdf" target="_blank" rel="noopener">RNN Tutorial-Toronto University</a>

<a href="http://colah.github.io/posts/2015-08-Understanding-LSTMs/" target="_blank" rel="noopener">Understanding-LSTM</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">NLP</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color5">论文笔记</a>
