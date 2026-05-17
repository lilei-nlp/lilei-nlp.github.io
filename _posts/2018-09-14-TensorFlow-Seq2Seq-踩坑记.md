---
title: "TensorFlow——Seq2Seq 踩坑记"
date: 2018-09-14
categories:
  - blog
tags: []
---

## <a href="#缘起" class="headerlink" title="缘起"></a>缘起

Seq2Seq 的 TensorFlow 实现有很多，而 TensorFlow 之前也推出了一套新的 <a href="https://www.tensorflow.org/api_docs/python/tf/contrib/seq2seq" target="_blank" rel="noopener">API</a>，文档依旧是令人蛋疼地杂乱。最近在用新的 API 写一个 Many2Many 结构的 Seq2Seq，踩到了一个坑，记录之，也进一步地提醒我应该把主力框架迁移到 PyTorch 提上日程了。

## <a href="#问题描述" class="headerlink" title="问题描述"></a>问题描述

根据示例，只要把 encoder 和 decoder 拼接起来，并且使用 TrainingHelper + BasicDecoder：

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
10</code></pre></td>
<td class="code"><pre><code># 省略参数
helper_pt = tf.contrib.seq2seq.TrainingHelper()
decoder_pt = tf.contrib.seq2seq.BasicDecoder()
outputs_pt, _final_state, sequence_lengths_pt = tf.contrib.seq2seq.dynamic_decode()
# logits
logits = outputs_pt.rnn_output 
# loss
loss = tf.contrib.seq2seq.sequence_loss(logits, target, target_mask,
                    average_across_timesteps=True,
                    average_across_batch=False)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

其中 `logits` 是 vocab\_size 上的分布，shape 为 `[batch_size, ?, vocab_size]`

而 `target` 则是正确的输出，这里我将其做了 padding，补齐到最大长度，shape 为 `[batch_size, max_len]`，`target_mask` 是对应的一个权重，非补齐的部分才会参与到 loss 的计算。

如果不 Padding，则会在 feed\_dict 这一步报错：

> ValueError: setting an array element with a sequence.

**因为 target\_input 的长度是变长的** ，NumPy 无法将其视作一个 array，导致错误。

然后问题就来了，`sequence_loss` 函数报错，说其内部调用的 `sparse_soft_max` 函数的 `labels` 和 `logits` 第一维不匹配，其中：logits 的形状为 `[?, vocab_size]` ；label 的形状为 `[batch_size x max_len]`

为什么会这样呢？来进一步的分析分析

## <a href="#Source-Code" class="headerlink" title="Source Code"></a>Source Code

首先是看一下 `sequence_loss` 的源代码：

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
7</code></pre></td>
<td class="code"><pre><code>with ops.name_scope(name, &quot;sequence_loss&quot;, [logits, targets, weights]):
   num_classes = array_ops.shape(logits)[2]
   logits_flat = array_ops.reshape(logits, [-1, num_classes])
   targets = array_ops.reshape(targets, [-1])
   if softmax_loss_function is None:
     crossent = nn_ops.sparse_softmax_cross_entropy_with_logits(
         labels=targets, logits=logits_flat)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

先是对 logits 和 labels 做了 reshape 操作，之前的形状也能对上，也就是说 logits reshape 之后的 `? = ? x batch_size` ，那么这个 `?` 究竟是是什么呢，不出意外，应该是生成的序列长度，但为什么是不定长的呢？

来看看 `TrainingHelper` 的核心源代码:

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
11</code></pre></td>
<td class="code"><pre><code># TrainingHelper
def next_inputs(self, time, outputs, state, name=None, **unused_kwargs):
      next_time = time + 1
      finished = (next_time &gt;= self._sequence_length)
      all_finished = math_ops.reduce_all(finished)
      def read_from_ta(inp):
        return inp.read(next_time)
      next_inputs = control_flow_ops.cond(
          all_finished, lambda: self._zero_inputs,
          lambda: nest.map_structure(read_from_ta, self._input_tas))
      return (finished, next_inputs, state)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

`next_inputs` 函数是 RNN 不断获取下一步的迭代函数，其中 `finished` 相当于指示了当前的时间步是否已经超出最大长度，即 `self._sequence_length` 一个记录目标输出长度 int32 向量。`reduce_all()` 是对某一个维度求逻辑与，如果没有指定 axis 参数，则是对所有元素进行与运算。由此，我们可以得知：当未运行至 target 目标时间步时，会用 Groud-Truth 作为下一个的输入，否则则为 0。但这里并没有给出什么时候停止，所以进一步地，看看 decoder 在哪里调用这个的函数：

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
13</code></pre></td>
<td class="code"><pre><code>def step(self, time, inputs, state, name=None):
    cell_outputs, cell_state = self._cell(inputs, state)
    if self._output_layer is not None:
        cell_outputs = self._output_layer(cell_outputs)
    sample_ids = self._helper.sample(
    time=time, outputs=cell_outputs, state=cell_state)
    (finished, next_inputs, next_state) = self._helper.next_inputs(
              time=time,
              outputs=cell_outputs,
              state=cell_state,
              sample_ids=sample_ids)
    outputs = BasicDecoderOutput(cell_outputs, sample_ids)
    return (outputs, next_state, next_inputs, finished)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这是 decoder 的 `step` 函数，相当于对 `helper` 进一步地封装，以便使用一些功能（TrainingHelper 是训练时 feed groud-truth，在 inference 阶段会使用 GreedyEmbeddingHelper 在无 Groud-Truth 帮助下进行生成等）。再向上，我们去看 `dynamic_decode` 的函数：

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
35</code></pre></td>
<td class="code"><pre><code># dynamic_decode 为了节约篇幅，仅保留重要的代码
def dynamic_decode():

    initial_finished, initial_inputs, initial_state = decoder.initialize()

    def condition(unused_time, unused_outputs_ta, unused_state, unused_inputs,
                  finished, unused_sequence_lengths):
      return math_ops.logical_not(math_ops.reduce_all(finished))

    def body(time, outputs_ta, state, inputs, finished, sequence_lengths):
      (next_outputs, decoder_state, 
       next_inputs, decoder_finished) = decoder.step(time, inputs, state)
      # next_finished 是 decoder_finished 和 finished 的 or
      next_finished = math_ops.logical_or(decoder_finished, finished)
      next_sequence_lengths = array_ops.where(
          math_ops.logical_not(finished),
          array_ops.fill(array_ops.shape(sequence_lengths), time + 1),
          sequence_lengths)
      return (time + 1, outputs_ta, next_state, next_inputs, next_finished,
              next_sequence_lengths)

    res = control_flow_ops.while_loop(
        condition,
        body,
        loop_vars=(
            initial_time,
            initial_outputs_ta,
            initial_state,
            initial_inputs,
            initial_finished,
            initial_sequence_lengths,
        ),
        parallel_iterations=parallel_iterations,
        maximum_iterations=maximum_iterations,
        swap_memory=swap_memory)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里，总算看到了我们要的 while loop，循环的控制变量是 `finished`，而其又 是 decoder\_finished 和 finished 的逻辑或所得到，所以可以得出：当 decoder 解码到 sequence\_length 时，其才会停止；另一方面，因为一个 batch 中的长度不都相同，所以得到的 dynamic\_length 应该是某个 batch 中最长的一句的长度，到了这里，问题总算是知道根源所在了，那么，怎么解决呢？

## <a href="#Solution" class="headerlink" title="Solution"></a>Solution

GitHub 上有人提出过这个问题 <a href="https://github.com/tensorflow/nmt/issues/2" target="_blank" rel="noopener">Issue</a> ，并且有很长的讨论，有一个比较粗暴的解决方案，在无法喂给它 padding 之前的情况下，对 target 做一个截取，因为之前的研究能够让我们确信生成的序列的长度是一定小于等于 batch 中 target 最长的长度的，所以：

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
6</code></pre></td>
<td class="code"><pre><code># 获取当前的长度，max_len 和 logits 的较小者，事实上，我们可以认为就是 logits 的长度
current_ts = tf.to_int32(tf.minimum(tf.shape(self.target_input)[1], tf.shape(logits)[1]))
# 对 target 进行截取
target_sequence = tf.slice(self.target_input, begin=[0, 0], size=[-1, current_ts])
mask_ = tf.sequence_mask(lengths=self.target_len, maxlen=current_ts, dtype=logits.dtype)
logits = tf.slice(self.logits_pt, begin=[0, 0, 0], size=[-1, current_ts, -1])</code></pre></td>
</tr>
</tbody>
</table>
</figure>

截取之后，就可以保证二者的长度一致，再使用`sequence_loss()` 计算就可以了。

另外有一个疑问就是，有些代码是可以运行不会报错（网上 Seq2Seq 的教程都是这么写的），猜测是输入数据的格式问题，日后碰到了再提。
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">Seq2Seq</a>
