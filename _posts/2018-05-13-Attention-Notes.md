---
title: "别人家的 Attention"
date: 2018-05-13
categories:
  - blog
tags: []
---

<a href="https://arxiv.org/abs/1706.03762" target="_blank" rel="noopener">Attention Is All You Need</a> 前段时间火了一把，其提出完全用 Attention 替代传统的 CNN 和 RNN 架构来做特征的提取，也在 NMT 上也取得了 state-of-the-art。这两天读了一下这篇 Paper，并且在熟悉的 Text Classification 问题上用其模型做了一下尝试，这篇 Blog 就用来记录过程中的一些想法和感受。

## <a href="#What-is-Attention" class="headerlink" title="What is Attention"></a>What is Attention

注意力机制之前在学的时候就有过一次[梳理](http://tobiaslee.top/2017/08/15/Attention-Mechanism-%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/)，上一次对于什么是注意力机制，我的回答是：

> 聚焦在某个局部的 focus

现在我的回答是：Attention（一般指的是 Self-Attention），是特征提取过程中，信息融合的手段。其目的是能够让模型的信息视野有的放矢，其数学上的表现就是加权和。

NLP community 曾经有过这么一种说法：

> an LSTM with attention will yield state-of-the-art performance on any task

以及这样一张图：

![Joke](/img/joke.jpg)

中心思想就是：Attention + LSTM 是一个非常 Powerful 的 model，基本能在所有的 NLP 任务上 work。就我有限的经验来说，大抵如此了。特别是 Attention，简直是即插即用效果还特别好的万金油。

而论文中对 Attention 的定义是这样的：

> An attention function can be described as mapping a query and a set of key-value pairs to an output, where the query, keys, values, and output are all vectors.

这里的 query、key、value 是理解的重点：

对于机器翻译任务来说，在传统的 Seq2Seq 架构中，假设我们将要输出第 k 个词，那么这个 query 就代表这**第 k 个词**对应的 hidden state，key 和 value 一般是相等的（作者也提出了一种不相等的方式，详见下图），即之前 encode 的所有 hidden state：

![Query、Key and Values](/img/qkv.jpg)

一开始提出 Attention 的使用一个 Alignment Function 来描述，并且提出了几种 score 的计算方式。这里的计算公式就是用最普通的矩阵乘法：

$Attention(Q, K, V) = softmax( \frac{QK^T}{\sqrt{d\_k}})V$

Softmax 项就是权重项，$V$ 是一系列 hidden states，也就是说，attention 最终的表现形式依旧是**加权和**。

<span id="more"></span>

## <a href="#Multihead-Attention" class="headerlink" title="Multihead Attention"></a>Multihead Attention

到了本文最重要的部分， Multi-head Attention。作者的 Motivation 认为是原有的 RNN 和 CNN 并行化不够，太慢了；同时觉得原先的复杂度太高，像 RNN，从头滚到尾关于序列长度是一个 $O(n)$ 的复杂度。所以期望单用一个 Attention 来做特征的提取，因而提出了 Multi-head Attention。

![Multi-head Attention](/img/multi-head.png)

$MultiHead(Q, K, V) = Concat( head\_1, … , head\_h) W^O$

$head\_i = Attention(QW\_i^Q, KW\_i^L, VW\_i^V)$

就是先让 Q，K，V 做一个线性的投影（分别乘上个矩阵），再做 Attention，这样重复多次，将结果拼接起来，得到一个“多头” Attention。

背后的动机是什么呢？文章中这样说：

> Multi-head attention allows the model to jointly attend to information from different representation subspaces at different positions.

一方面，从直觉上多次 Attention 操作就能够捕获更多的信息；另一方面，先进行的投影操作能够把 Q、K、V 映射到不同空间，也许能够发现更多的特征。

然后再给他套上一层全连接：

$FFN(x) = ReLU(xW\_1 + b\_1) W\_2 + b\_2$

这样的 Attention 操作没有考虑到时序信息，但序列位置的信息还是很重要的，因此，作者对位置信息进行了 Encoding:

![Position Encoding](/img/pe.png)

同时文章还仿照 CNN，增加了常用的 Residual Connection 以及 Layer Normalization 操作，这里就不再展开。

## <a href="#Implementation" class="headerlink" title="Implementation"></a>Implementation

该 Paper 有 TensorFlow 的开源<a href="https://github.com/Kyubyong/transformer" target="_blank" rel="noopener">实现</a>，侧重看一下 Multi-head Attention 以及 FFN 的实现：

<figure class="highlight python">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70</code></pre></td>
<td class="code"><pre><code>def multihead_attention(queries,
                        keys,
                        num_units=None,
                        num_heads=8,
                        dropout_rate=0,
                        is_training=True,
                        causality=False,
                        scope=&quot;multihead_attention&quot;,
                        reuse=None):
    with tf.variable_scope(scope, reuse=reuse):
        if num_units is None:  # set default size for attention size C
            num_units = queries.get_shape().as_list()[-1]

        # Linear Projections 线性投影
        Q = tf.layers.dense(queries, num_units, activation=tf.nn.relu)  # [N, T_q, C]
        K = tf.layers.dense(keys, num_units, activation=tf.nn.relu)  # [N, T_k, C]
        V = tf.layers.dense(keys, num_units, activation=tf.nn.relu)  # [N, T_k, C]

        # Split and concat 分割成 head = 8 块，再拼起来
        Q_ = tf.concat(tf.split(Q, num_heads, axis=-1), axis=0)  # [num_heads * N, T_q, C/num_heads]
        K_ = tf.concat(tf.split(K, num_heads, axis=-1), axis=0)  # [num_heads * N, T_k, C/num_heads]
        V_ = tf.concat(tf.split(V, num_heads, axis=-1), axis=0)  # [num_heads * N, T_k, C/num_heads]

        # Attention  根据公式，做 Attention 计算 weight matrix
        outputs = tf.matmul(Q_, tf.transpose(K_, [0, 2, 1])) # (num_heads * N, T_q, T_k)

        # Scale 缩放操作 outputs = outputs / sqrt( d_k)
        outputs = outputs / (K_.get_shape().as_list()[-1] ** 0.5)

        # Key Masking
        key_masks = tf.sign(tf.abs(tf.reduce_sum(keys, axis=-1)))  # (N, T_k)
        key_masks = tf.tile(key_masks, [num_heads, 1])  # (h*N, T_k)
        key_masks = tf.tile(tf.expand_dims(key_masks, 1), [1, tf.shape(queries)[1], 1])  # (h*N, T_q, T_k)

        paddings = tf.ones_like(outputs) * (-2 ** 32 + 1)  # -infinity
        outputs = tf.where(tf.equal(key_masks, 0), paddings, outputs)  # (h*N, T_q, T_k)

        # Causality = Future blinding
        if causality:
            diag_vals = tf.ones_like(outputs[0, :, :])  # (T_q, T_k)
            tril = tf.contrib.linalg.LinearOperatorTriL(diag_vals).to_dense()  # (T_q, T_k)
            masks = tf.tile(tf.expand_dims(tril, 0), [tf.shape(outputs)[0], 1, 1])  # (h*N, T_q, T_k)

            paddings = tf.ones_like(masks) * (-2 ** 32 + 1)
            outputs = tf.where(tf.equal(masks, 0), paddings, outputs)  # (h*N, T_q, T_k)

        # Activation: outputs is a weight matrix
        outputs = tf.nn.softmax(outputs)  # (h*N, T_q, T_k)

        # Query Masking
        query_masks = tf.sign(tf.abs(tf.reduce_sum(queries, axis=-1)))  # (N, T_q)
        query_masks = tf.tile(query_masks, [num_heads, 1])  # (h*N, T_q)
        query_masks = tf.tile(tf.expand_dims(query_masks, -1), [1, 1, tf.shape(keys)[1]])  # (h*N, T_q, T_k)
        outputs *= query_masks  # broadcasting. (N, T_q, C)

        # dropouts
        outputs = tf.layers.dropout(outputs, rate=dropout_rate, training=tf.convert_to_tensor(is_training))

        # weighted sum
        outputs = tf.matmul(outputs, V_) # ( h*N, T_q, C/h)

        # reshape
        outputs = tf.concat(tf.split(outputs, num_heads, axis=0), axis=2)  # (N, T_q, C)

        # residual connection
        outputs += queries

        # layer normaliztion
        outputs = layer_normalization(outputs)
        return outputs</code></pre></td>
</tr>
</tbody>
</table>
</figure>

基本就是按着 Paper 来的，不过一个很让人费解的地方就是其中的 Key Masking 和 Query Masking，Paper 中写是 Optional 的，代码的作者非常细致的实现了这一部分。其目的是考虑到**变长的序列**，比如第一句的长度为 128 而第二句只有 64，对于第二句，其 Encoding 的结果或者说是 Hidden State 的后面 64 个单元是没有意义的，因此将其设置为一个非常小的数，从而对应的权重接近 0；Query 类似。具体内容可以参考这个<a href="https://github.com/Kyubyong/transformer/issues/3" target="_blank" rel="noopener">Issue</a>。

<figure class="highlight python">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19</code></pre></td>
<td class="code"><pre><code># Feed Forward Network
def feedforward(inputs,
                num_units=[2048, 512],
                scope=&quot;multihead_attention&quot;,
                reuse=None):
    with tf.variable_scope(scope, reuse=reuse):
        # Inner layer
        params = {&quot;inputs&quot;: inputs, &quot;filters&quot;: num_units[0], &quot;kernel_size&quot;: 1,
                  &quot;activation&quot;: tf.nn.relu, &quot;use_bias&quot;: True}
        outputs = tf.layers.conv1d(**params)
        # Readout layer
        params = {&quot;inputs&quot;: outputs, &quot;filters&quot;: num_units[1], &quot;kernel_size&quot;: 1,
                  &quot;activation&quot;: None, &quot;use_bias&quot;: True}
        outputs = tf.layers.conv1d(**params)
        # Residual connection
        outputs += inputs
        # Normalize
        outputs = layer_normalization(outputs)
    return outputs</code></pre></td>
</tr>
</tbody>
</table>
</figure>

FFN 的实现就很简单，用两个 conv1d 的卷积，手动写矩阵乘法也可以；另外就是最后两步的 Residual Connection 直接加上输入以及 Layer Normalization。

PS：我拿着这个代码跑了一下 IMDB 的文本分类，只用了 Multi-head 和 FFN，Query 是一个随机初始化的向量，Key 和 Value就是经过 embedding 后的句子。 和 LSTM 对比下来，时间是 LSTM 的 6 倍，效果比 LSTM 还差… 为什么呢？因为没有**并行化**，事实上那些矩阵乘法都是可以用多块 GPU 来并行进行的，论文就说他们用了 **8 块 P100**。流下了没有钱的泪水。

## <a href="#Reference" class="headerlink" title="Reference"></a>Reference

<a href="https://zhuanlan.zhihu.com/p/34781297" target="_blank" rel="noopener">《attention is all you need》解读</a>

<a href="https://github.com/Kyubyong/transformer" target="_blank" rel="noopener">Github-Kyubyong/transformer</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">NLP</a>
