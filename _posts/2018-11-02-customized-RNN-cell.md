---
title: "定制 RNN Cell"
date: 2018-11-02
categories:
  - blog
tags: []
---

之前写过一篇 IndRNN 的文章，第一次照着开源代码实现了自己定制 RNN，其实也很简单，就是把 RNN 实现里的 `__call__` 函数进行修改，每一个时间步的全连接 `matmul` 改成 element-wise 的 `*` 即可，IndRNN 的<a href="https://github.com/batzner/indrnn" target="_blank" rel="noopener">GitHub</a>，是一个很不错的练手参考。最近做文本生成的时候遇到两个经典模型，在复现的时候同样是需要对 RNN 的结构进行改动，这篇文章就记录实现的一些细节。两个模型可以分成两类：在时间步的输入上进行操作和修改 Cell 内部结构。

## <a href="#Add-Extra-Input" class="headerlink" title="Add Extra Input"></a>Add Extra Input

第一个模型是 <a href="http://ir.hit.edu.cn/~xcfeng/xiaocheng%20Feng&#39;s%20Homepage_files/final-topic-essay-generation.pdf" target="_blank" rel="noopener">MTA-LSTM</a>，IJCAI 2018 的一篇关于主题写作的文章，也有开源<a href="https://github.com/hit-computer/MTA-LSTM" target="_blank" rel="noopener">实现</a>，但是因为其代码版本比较老，并且 inference 阶段的代码比较难并行无法和现有的 `seq2seq` 的接口结合使用。So，需求就是实现其代码并且尽可能地需要和`seq2seq` 接口符合以便于后续的使用。模型的结构如下：

![MTA-LSTM](/img/MTA-LSTM.png)

论文的核心思想如下：

1.  根据主题词进行文章写作，利用 LSTM 作为生成器
2.  在每个时间步，将 Topic Word Embedding 和 Inputs 拼接在一起交给 LSTM 进行生成
3.  为了控制主题信息的流动，会维持一个 Coverage Vector，来表明哪些主题已经使用过哪些尚未被表达出来，从未让写出的文章主题信息更加明确

我的实现版本代码已经放在了 <a href="https://github.com/TobiasLee/MTA-LSTM-TensorFlow" target="_blank" rel="noopener">GitHub</a>，下面主要记录一下实现的思路。

首先明确一点，为了和现有的 `seq2seq` 接口匹配，必然是无法去修改 LSTM Cell 的 `__call__` 函数的，因为目前的 `dynamic_decode` 函数对其中的 `cell` 每一个时间步都会调用 `cell(inputs, state)` ，也即我们无法在函数参数列表中增加我们想要的额外的参数。怎么做呢？利用 **Wrapper**！类似装饰者模式，我们在 LSTM Cell 外面包上一层，通过这一层来对 LSTM 的输入进行操作。下面是代码：

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
21</code></pre></td>
<td class="code"><pre><code>class MTAWrapper(RNNCell):
    def __init__(self, cell, topic, v, uf, query_layer, memory_layer, mask=None, max_len=100, attention_size=128, state_is_tuple=True
                 ):
      # 实际的 LSTM 由 cell 完成
        self._cell = cell
        self.topic = topuc # topic embedding
        # initilize coverage vector
        self.coverage_vector = array_ops.ones([self.batch_size, self.num_keywords])
        # get some weight variables from params
        # compute res 
        res1 = tf.sigmoid(
            tf.matmul(tf.reshape(self.topic, [self.batch_size, -1]), self.u_f))  # batch_size x num_keyword
        self.phi_res = self.seq_len * res1  # batch_size x num_keywords

    def __call__(self, inputs, state, scope=None):
        c_t, h_t = state  # h_t batch_size x hidden_size
        with vs.variable_scope(&quot;topic_attention&quot;):
            # Attention mechanism based on coverage vector to compute mt 
            # update coverage vector
            self.coverage_vector = self.coverage_vector - score / self.phi_res
        return self._cell(tf.concat([inputs, mt], axis=1), state)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

为了节约篇幅，省去一些代码，主要看两个部分：

1.  `__init__` 函数：我们在这里传入了大量参数，其中包括了:

    1.  cell：实际 RNN 的操作是由这个 LSTM cell 完成的，我们的 wrapper 只是夹在中间进行一些**小小的修改**。
    2.  主题词的 embedding：这是每一步计算需要用到的信息，并且是固定。
    3.  一些要使用到的 variable：为什么 variable 从外部传入而不是定义在 `__call__` 函数之内呢？**因为 training 和 inference 阶段，我们会重新 wrap 一下我们的 cell**，至于为什么，先按下不表。如果在内部定义的话，则 training 阶段学到的权重无法在 inference 阶段使用（其 name scope 是不一样的），inference 使用的依旧是初始化得到的权重，等于白学。

2.  `__call__` 函数：在这里我们进行每个时间步的 `mt` 的计算，并且将其和 inputs 进行拼接作为额外的信息，其中计算的细节就是对照公式进行实现的。这里有个坑需要谈一下：**TensorFlow 不支持一个 Tensor 出现在多个 Loop 中**。而如果我们之前不对 inference 阶段的 cell 重新进行包装的话，则 coverage vector 将会出现在 training 和 inference 的循环中，无法构建计算图。为此，我们才需要对 cell 进行重新 wrap：

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
    5</code></pre></td>
    <td class="code"><pre><code>training_cell = MTAWrapper(self.decoder_cell, topic_embedded,  self.v, self.uf, self.query_layer, self.memory_layer, mask=masks)
    decoder_pt = tf.contrib.seq2seq.BasicDecoder( cell=training_cell, helper=helper_pt, initial_state=self.initial_state, output_layer=self.output_layer)
    # 重新 wrap，并且把之前习得的 weight variable 通过构造函数传入
    infer_cell = MTAWrapper(self.decoder_cell, topic_embedded, self.v, self.uf, self.query_layer, self.memory_layer, mask=masks)
    decoder_i = tf.contrib.seq2seq.BasicDecoder( cell=infer_cell, helper=helper_i, initial_state=self.initial_state, output_layer=self.output_layer)</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    所以我们才会需要重新传入 weight variables。最后，计算完 `mt` 之后，直接拼接交给 `cell` 去调用每一时间步就 ok。

## <a href="#Add-Extra-Gate" class="headerlink" title="Add Extra Gate"></a>Add Extra Gate

对 LSTM 的结构进行修改，听着很有挑战性吧，这次不仅是要增加一个额外的输入，并且还需要添加一个额外的门来控制额外输入信息的流动。<a href="http://www.emnlp2015.org/proceedings/EMNLP/pdf/EMNLP199.pdf" target="_blank" rel="noopener">SC-LSTM</a> 是一篇发表在 EMNLP 2015 上的文章，也是做类似的主题生成问题，模型图如下：

![SC-LSTM](/img/SC-LSTM.png)

这里的额外输入就是 action vector，一个 one-hot 的向量来表明主题信息，额外的 gate 就是图中的 $r\_t$，其对输入的 action vector 根据当前的输入以及之前的 hidden state 计算一个值，控制 topic 信息的流动，最后这一信息经过 gate 之后会参与的新的 hidden state 的计算之中。这次，我们必须对 `__call__` 函数的进行修改了，因为不再是简单的输入拼接，**而 LSTM 内部的 hidden state 的计算过程也需要依赖这个 action vector**。先来看代码：

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
38</code></pre></td>
<td class="code"><pre><code>class SCLSTM(BasicLSTMCell):
    def __init__(self, kwd_voc_size, *args, **kwargs):
        BasicLSTMCell.__init__(self, *args, **kwargs)
        self.key_words_voc_size = kwd_voc_size

    def __call__(self, inputs, state, d_act):
        sigmoid = math_ops.sigmoid
        one = constant_op.constant(1, dtype=dtypes.int32)
     # parameters for tanh function
        w_d = vs.get_variable(&#39;w_d&#39;, [self.key_words_voc_size, self._num_units])
        # Parameters of gates are concatenated into one multiply for efficiency.
        if self._state_is_tuple:
            c, h = state
        else:
            c, h = array_ops.split(value=state, num_or_size_splits=2, axis=one)
        gate_inputs = math_ops.matmul(
            array_ops.concat([inputs, h], 1), self._kernel)
        gate_inputs = nn_ops.bias_add(gate_inputs, self._bias)

        # i = input_gate, j = new_input, f = forget_gate, o = output_gate
        i, j, f, o = array_ops.split(
            value=gate_inputs, num_or_size_splits=4, axis=one)

        forget_bias_tensor = constant_op.constant(self._forget_bias, dtype=f.dtype)

        add = math_ops.add
        multiply = math_ops.multiply
        # add extra topic information to candidate calculation 
        new_c = add(add(multiply(c, sigmoid(add(f, forget_bias_tensor))),
                        multiply(sigmoid(i), self._activation(j))),
                    math_ops.tanh(math_ops.matmul(d_act, w_d)))
        new_h = multiply(self._activation(new_c), sigmoid(o))

        if self._state_is_tuple:
            new_state = LSTMStateTuple(new_c, new_h)
        else:
            new_state = array_ops.concat([new_c, new_h], 1)
        return new_h, new_state</code></pre></td>
</tr>
</tbody>
</table>
</figure>

其中的核心就是我们申请了一个额外的变量 `w_d`，并且在 candidate 的计算之中加上了一项额外的 `tanh` 项，这是和文章的公式对应的：

![Candidate Computation](/img/cd_com.png)

但是修改了 LSTM Cell 的 `__call__` 函数之后，就和 seq2seq 接口不匹配了，依旧是借助 Wrapper 来实现接口的匹配，并且额外的 gate 也在 wrapper 中进行实现：

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
20</code></pre></td>
<td class="code"><pre><code>class ActionWrapper(RNNCell):
    def __init__(self, cell, action_vec, wr, hr):
        if not isinstance(cell, SC_DropoutWrapper) and not isinstance(cell, SCLSTM):
            raise TypeError(&quot;The wrapper is only designed for SCLSTM.&quot;)
        self._cell = cell
        self.action_vec = action_vec  # initial one-hot action vector
        self.wr = wr  # [word_embedding_size, topic_size ]
        self.hr = hr  # [hidden_size, topic_size]

    # note: params only inputs and state
    def __call__(self, inputs, state, scope=None):
        ct, ht = state
        # compute sigmoid and update action vec
        # r_t = sigmoid( W_wr x_t + W_hr h_{t-1})  = sigmoid(e1 + e2)
        e1 = math_ops.matmul(inputs, self.wr)  # [batch_size, topic_size]
        e2 = math_ops.matmul(ht, self.hr)
        r_t = math_ops.sigmoid(math_ops.add(e1, e2))  # [batch_size, topic_size]
        # update action vector
        self.action_vec = r_t * self.action_vec
        return self._cell(inputs, state, self.action_vec)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

和之前一样，在构造函数之中我们传入了 one-hot 的 action vector，并且获取一些 variables，理由和之前一样，**action vector 每步都要更新，但是不能出现在两个循环之中**。`__call__` 函数的结构和原先的 LSTM 保持一致，这样就能够和 seq2seq 的接口配合着使用，而在内部，根据文章的公式设置了一个 `r_t` 作为 gate 来控制 topic information：

![r\_t](/img/rt_formula.png)

使用的时候，同样地，我们可以通过使用 `ActionWrapper(SCLSTMCell(hidden_state))` 来进行使用。

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

至此，我们已经展现了如何利用 TensorFlow 的 `Wrapper` 对象来 RNN cell 功能的修改，同时能够通过接口参数的设计来让自定义的 RNN cell 和其他的 API 一起协作起来。这个 `Wrapper` 类，实际上就是装饰者模式的一种应用，一层一层地包裹住来增加功能并且能够屏蔽接口的不同。踩到的坑也就是在文中提到的：

1.  同一个 Tensor 对象不能出现在不同 Loop 中
2.  使用 Wrapper 对象的时候注意内部定义的 variable，可能需要通过外部传入来使学到的参数进入到后续的使用中（training 和 inference）

复现这两篇 Paper 的过程中主要参考了 RNN 的<a href="https://github.com/tensorflow/tensorflow/blob/r1.11/tensorflow/python/ops/rnn_cell_impl.py" target="_blank" rel="noopener">源码</a>（似乎新旧版本的 API 还有不一样？），虽然之前也有读过，但自己写一遍又是不一样体验。希望以后多写写这样的代码~
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">RNN</a>
