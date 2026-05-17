---
title: "MaskGAN 学习笔记"
date: 2018-04-01
categories:
  - blog
tags: []
---

MaskGAN 是 Goodfellow 组的新作，已经被 ICLR 2018 接收，标题很是风骚，<a href="https://openreview.net/pdf?id=ByOExmWAb" target="_blank" rel="noopener">MaskGAN: Better Text Generation via Filling in the <strong><strong>____</strong></strong></a>，这个下划线的操作真是… astounding。代码也已<a href="https://github.com/tensorflow/models/tree/master/research/maskgan" target="_blank" rel="noopener">开源</a>。这篇文章依旧是熟悉的套路，从模型 + 代码来解读论文，走起！

## <a href="#SeqGAN-To-MaskGAN" class="headerlink" title="SeqGAN To MaskGAN"></a>SeqGAN To MaskGAN

### <a href="#SeqGAN-的缺点" class="headerlink" title="SeqGAN 的缺点"></a>SeqGAN 的缺点

上一篇讲 SeqGAN 的时候我们提到，SeqGAN 开创了 GAN 在 Text Generation 的先河，但是，实验结果证明，其 Idea 是能 Work（通过强化学习解决 GAN 无法在离散文本上梯度回传），合成数据中的 loss 确实有下降，但是在真实的古诗数据集上，其**生成的文本质量不如人意**。我利用全唐诗做了实验，不过囿于设备和时间原因，并没有充分的训练和调优，摘录部分生成结果如下：

> 霞畅拍起妇 已煦肃兢恼 鶋仝棚愕迷 啼肃次念云
>
> 岂阳孤任帐 因伊牧掩牢 人原马槎问 弥章斗天钓
>
> 鸡行肩始昏 晨刺重云千 指瘼山月堂 一似蕃德率
>
> 有足偶有欲 威飏欢浩潋 戏鸟靓簪粘 性负觉狄至

有没有一种狗屁不通的感觉… 反正我是很绝望。

也就是说 SeqGAN 效果不是很好（也有实验室做过实验，其中生成质量较好的古诗基本都是训练集中的），而 MaskGAN 可能为提升生成文本的质量指出了一个方向，其和 SeqGAN 有两点主要的区别：

1.  增加额外的 Information，Masked Sequence $m(x)$，这也导致了其使用的模型架构变成 Seq2Seq，而非 SeqGAN 中 LSTM（Generator）和 CNN（Discriminator）
2.  使用 Actor-Critic 来进行强化学习，而非 SeqGAN 中的 Policy Gradient + Monte Carlo

接下来我们就从这两点不同入手，来讲解 MaskGAN。

### <a href="#Masked-Token" class="headerlink" title="Masked Token"></a>Masked Token

MaskGAN 在文中指出了 GAN 的两个问题，一是 Mode Collapse，即可能出现少数的生成样本种类占据了整个生成集，缺乏多样性；二是训练不稳定，GAN 难调试是出了名的。文章解决这两个问题的方案是：不再让生成器来生成的完整的文本，而是做“完形填空”，不过关于为什么能解决，他们是这么说的：

> We believe the in-filling may mitigate the problem of severe mode-collapse.

一个`believe` 再来 `may` 加上一个 `mitigate`，这就是论文的表述的艺术啊。解决训练不稳定的方法呢就是从 Policy Gradient 转换为 Actor-Critic，后面再说。

“完形填空”相信大家高中都做过，就是把文章挖空然后让你选一个正确的单词填进去，MaskGAN 就是这么干的，对于一个输入序列 $x = （x\_1,…, x\_T)$，经过一个 mask： $m=(m\_1,…m\_T)$，其中 $m\_i$ 的取值为 0 或者 1，0 就代表挖掉，1 就意味着保留。经过挖空的操作之后呢，我们就得到了 Masked Token $m(x)$，并将它交给我们以 Seq2Seq 为架构的 Generator 来进行生成，模型见下：

![MaskGAN](/img/maskgan_model.png)

需要注意一点就是：生成的 token 不一定会作为下一个生成的 pre-token，而是取决于是否被挖空，如有原；这也是一个重要的细节，因为一个错误的答案可能会导致一整篇文章都是错误的，所以，如果有参考答案还是用参考答案。

<span id="more"></span>

Discriminator 的架构也是采用的 Seq2Seq，只不过是 many2one，即最后生成的每个 token 为真的概率。除了有填好的句子做为输入以外，$m(x)$ 也作为 Discriminator 的输入，文章是这么解释这么做的原因的：对于一个生成的句子 `the director director guided the series` ，如果没有 $m(x)$ 的话，那么判别器无法分别到底前一个 `director` 是原文呢还是后一个是，因为句子有可能是 `the *associate* director guided the series` 或者是 `the director *expertly* guided the series`，因此是有必要给判别器关于原文的信息，从而做出更好的判断。生成器和判别器的公式如下：

![MaskGAN Generator](/img/mask_gen.png)

![MaskGAN Discriminator](/img/mask_dis.png)

### <a href="#Actor-Critic" class="headerlink" title="Actor-Critic"></a>Actor-Critic

前面的文章谈到了，AC 的做法相比 Policy Gradient，很大的区别就在是**单步更新**，以及用一个 NN 来拟合 Advantage Function 来指导生成器生成更加逼真的文本。MaskGAN 的单步reward $r\_t$ 设置为了 log probablity，也就是：

$$ r\_t = log D\_\phi(\hat{x\_t}|\hat{x}\_{0:T}, \\ \textbf{m(x)})$$

总的 reward $R\_t​$ 则为这一时刻到句子结束 T 时之和：

$$R\_t = \sum\_{s=t}^T \gamma ^s r\_s$$

我们通过减去一个 critic 产生的 baseline $b\_t$ 来降低 variance，更新梯度的计算就变成：

![Gradient](/img/ac_gradient.png)

其中 $b\_t$ 由一个 NN 来拟合，MaskGAN 选择使用 Discriminator 的前半部分来估计 $b\_t$，详细说明需要结合代码来进行。

## <a href="#Code-Matters" class="headerlink" title="Code Matters"></a>Code Matters

代码永远是一个很好的学习材料，也是检验论文到底是不是糊弄人的试金石。

### <a href="#Generator" class="headerlink" title="Generator"></a>Generator

从代码里我们可以看到，作者实现了很多种 Generator 的架构，有 CNN、RNN 和 Seq2Seq，所以 Seq2Seq 应该是经过对比之后选出来效果比较好的一种。

先来看 Encoder 部分，其作用是把 Masked Token 交给一个 LSTM：

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
66</code></pre></td>
<td class="code"><pre><code>def gen_encoder(hparams, inputs, targets_present, is_training, reuse=None):

  # We will use the same variable from the decoder: get word embedding matrix

  with tf.variable_scope(&#39;encoder&#39;, reuse=reuse):

    def lstm_cell(): 
        return tf.contrib.rnn.BasicLSTMCell() #... 

    attn_cell = lstm_cell
    if is_training and hparams.gen_vd_keep_prob &lt; 1:
      def attn_cell():
        # .. Add variational dropout on the cell

    cell = tf.contrib.rnn.MultiRNNCell(
        [attn_cell() for _ in range(hparams.gen_num_layers)],
        state_is_tuple=True)

    initial_state = cell.zero_state(FLAGS.batch_size, tf.float32)

    # 进行 Mask 操作
    real_inputs = inputs
    masked_inputs = transform_input_with_is_missing_token(
        inputs, targets_present)

    with tf.variable_scope(&#39;rnn&#39;) as scope:
      hidden_states = []

      # Split the embedding into two parts so that we can load the PTB
      # weights into one part of the Variable.
      if not FLAGS.seq2seq_share_embedding:
        embedding = tf.get_variable(&#39;embedding&#39;,
                                    [FLAGS.vocab_size, hparams.gen_rnn_size])
      missing_embedding = tf.get_variable(&#39;missing_embedding&#39;,
                                          [1, hparams.gen_rnn_size])
      embedding = tf.concat([embedding, missing_embedding], axis=0)
      
      real_rnn_inputs = tf.nn.embedding_lookup(embedding, real_inputs)
      masked_rnn_inputs = tf.nn.embedding_lookup(embedding, masked_inputs)

      state = initial_state
 
      def make_mask(keep_prob, units):
        random_tensor = keep_prob
        # 0. if [keep_prob, 1.0) and 1. if [1.0, 1.0 + keep_prob)
        random_tensor += tf.random_uniform(
            tf.stack([FLAGS.batch_size, 1, units]))
        return tf.floor(random_tensor) / keep_prob

      if is_training:
        output_mask = make_mask(hparams.gen_vd_keep_prob, hparams.gen_rnn_size)

      hidden_states, state = tf.nn.dynamic_rnn(
          cell, masked_rnn_inputs, initial_state=state, scope=scope)
      if is_training:
        hidden_states *= output_mask

      final_masked_state = state

      # 在未 mask 的输入上再来一次 encode 操作
      real_state = initial_state
      _, real_state = tf.nn.dynamic_rnn(
          cell, real_rnn_inputs, initial_state=real_state, scope=scope)
      final_state = real_state

  return (hidden_states, final_masked_state), initial_state, final_state</code></pre></td>
</tr>
</tbody>
</table>
</figure>

思路是这样的：输入的 Inputs，根据 targets\_present（一个 bool 向量指示是否 mask）进行 mask 操作，然后丢进 RNN 里面，得到最后的 state 作为输出。

但这个代码里看到了几个 tricks：

1.  在 RNN Cell 外再包了一层 Variational Dropout，每个 Unit 的 Dropout Rate 也是随机产生的，而不再是定值。推测是想要加强 Regularization 的作用，学到了。
2.  在 Masked 的 Inputs 上进行一次 Encode 得到一个 final\_masked\_state 后，又在 Origin Input 上做了一次 Encode 得到 final\_state，还不知道是干嘛用的，稍后再看。

接下来是 Decoder 部分：

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
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101</code></pre></td>
<td class="code"><pre><code>def gen_decoder(hparams,
                inputs,
                targets,
                targets_present,
                encoding_state,
                is_training,
                is_validating,
                reuse=None):
  &quot;&quot;&quot;Define the Decoder graph. The Decoder will now impute tokens that
      have been masked from the input seqeunce.
  &quot;&quot;&quot;
  gen_decoder_rnn_size = hparams.gen_rnn_size

  targets = tf.Print(targets, [targets], message=&#39;targets&#39;, summarize=50)
  if FLAGS.seq2seq_share_embedding:
    with tf.variable_scope(&#39;decoder/rnn&#39;, reuse=True):
      embedding = tf.get_variable(&#39;embedding&#39;,
                                  [FLAGS.vocab_size, hparams.gen_rnn_size])

  with tf.variable_scope(&#39;decoder&#39;, reuse=reuse):

    # 准备工作 定义 lstm_cell / attn_cell
    # 获取 hidden_states 和 final_state
    hidden_vector_encodings = encoding_state[0]
    state_gen = encoding_state[1]

    # Add variational Droppout ...
  # Generate Tokens
    with tf.variable_scope(&#39;rnn&#39;):
      sequence, logits, log_probs = [], [], []
      # 利用 word embedding matrix 作为 Softmax_W 的 Matrix
      softmax_w = tf.matrix_transpose(embedding)
      softmax_b = tf.get_variable(&#39;softmax_b&#39;, [FLAGS.vocab_size])

      rnn_inputs = tf.nn.embedding_lookup(embedding, inputs)
        
      rnn_outs = []

      fake = None
      for t in xrange(FLAGS.sequence_length):
        if t &gt; 0:
          tf.get_variable_scope().reuse_variables()

        # Input to the Decoder.
        if t == 0:
          # Always provide the real input at t = 0.
          rnn_inp = rnn_inputs[:, t]

        
        else:
          real_rnn_inp = rnn_inputs[:, t]
        # MLE 或者是 validating 时
          if is_validating or FLAGS.gen_training_strategy == &#39;cross_entropy&#39;:
            rnn_inp = real_rnn_inp
          else:
            fake_rnn_inp = tf.nn.embedding_lookup(embedding, fake)
            # 如果有 real token 用 real，没用就用先前生成的 token
            rnn_inp = tf.where(targets_present[:, t - 1], real_rnn_inp,
                               fake_rnn_inp)

        # RNN. run one step
        rnn_out, state_gen = cell_gen(rnn_inp, state_gen)

        if FLAGS.attention_option is not None:
          rnn_out = attention_construct_fn(rnn_out, attention_keys,
                                           attention_values)
        if is_training:
          rnn_out *= output_mask
     
        rnn_outs.append(rnn_out)
        if FLAGS.gen_training_strategy != &#39;cross_entropy&#39;:
          logit = tf.nn.bias_add(tf.matmul(rnn_out, softmax_w), softmax_b)

          # Decoder 的输出，如果有 real token 则输出 real token，没有则输出 fake token
          real = targets[:, t]

          categorical = tf.contrib.distributions.Categorical(logits=logit)
          if FLAGS.use_gen_mode:
            fake = categorical.mode()
          else:
            fake = categorical.sample() # sample a token based on the distribution
          log_prob = categorical.log_prob(fake)
          output = tf.where(targets_present[:, t], real, fake)

        else:
          real = targets[:, t]
          logit = tf.zeros(tf.stack([FLAGS.batch_size, FLAGS.vocab_size]))
          log_prob = tf.zeros(tf.stack([FLAGS.batch_size]))
          output = real

        # Add to lists.
        sequence.append(output)
        log_probs.append(log_prob)
        logits.append(logit)

      if FLAGS.gen_training_strategy == &#39;cross_entropy&#39;:
        # Code for MLE pre-training 
      else:
        logits = tf.stack(logits, axis=1)

  return (tf.stack(sequence, axis=1), logits, tf.stack(log_probs, axis=1))</code></pre></td>
</tr>
</tbody>
</table>
</figure>

Decoder 的思路也是很直接，就是用 Encoder 传入的 state tuple，进行 token 的生成。有几点需要注意的是：

1.  和论文中**一样**，如果有 real token，那么 real token 就会作为下一个 token 的 input，而非使用生成的 token
2.  作者在设计的时候考虑到了使用 MLE 进行预训练的情况，这时候就全部使用 real tokens，并基于此生成一句话

### <a href="#Discriminator" class="headerlink" title="Discriminator"></a>Discriminator

文章说 Discriminator 的架构和 Generator 架构是一样的，只是最后输出是一个 scalar:

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
14</code></pre></td>
<td class="code"><pre><code>with tf.variable_scope(&#39;dis&#39;, reuse=reuse):
  encoder_states = dis_encoder(
      hparams,
      masked_inputs,
      is_training=is_training,
      reuse=reuse,
      embedding=embedding)
  predictions = dis_decoder(
      hparams,
      sequence,
      encoder_states,
      is_training=is_training,
      reuse=reuse,
      embedding=embedding)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

确实，`dis_encoder` 部分的代码和 `gen_encoder` 是一致的，实现也是类似的；

而 `dis_decoder` ：

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
15</code></pre></td>
<td class="code"><pre><code>  with tf.variable_scope(&#39;rnn&#39;) as vs:
    predictions = []

    rnn_inputs = tf.nn.embedding_lookup(embedding, sequence)

    for t in xrange(FLAGS.sequence_length):
      if t &gt; 0:
        tf.get_variable_scope().reuse_variables()
# 
      rnn_in = rnn_inputs[:, t]
      rnn_out, state = cell_dis(rnn_in, state)

      # Prediction is linear output for Discriminator.
      pred = tf.contrib.layers.linear(rnn_out, 1, scope=vs)
      predictions.append(pred)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

对于输入 `sequence` ，进行 embedding 后，拿出里面的每一个 token，交给 RNN，输出一个 probability，没问题！

## <a href="#Critic" class="headerlink" title="Critic"></a>Critic

论文中有提到一嘴，就是说 AC 这个算法是后来审稿人提出意见之后再加的。我一开始还担心代码里没有，但 Google 还是做的很不错的：

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
62</code></pre></td>
<td class="code"><pre><code>def critic_seq2seq_vd_derivative(hparams, sequence, is_training, reuse=None):

  sequence = tf.cast(sequence, tf.int32)

  # parameter setting ...

  # reuse decoder&#39;s variables
  with tf.variable_scope(
      &#39;dis/decoder/rnn/multi_rnn_cell&#39;, reuse=True) as dis_scope:

    def lstm_cell():
      return tf.contrib.rnn.BasicLSTMCell(
          hparams.dis_rnn_size,
          forget_bias=0.0,
          state_is_tuple=True,
          reuse=True)

    attn_cell = lstm_cell
    if is_training and hparams.dis_vd_keep_prob &lt; 1:

      def attn_cell():
        return variational_dropout.VariationalDropoutWrapper(
            lstm_cell(), FLAGS.batch_size, hparams.dis_rnn_size,
            hparams.dis_vd_keep_prob, hparams.dis_vd_keep_prob)

    cell_critic = tf.contrib.rnn.MultiRNNCell(
        [attn_cell() for _ in range(hparams.dis_num_layers)],
        state_is_tuple=True)

  with tf.variable_scope(&#39;critic&#39;, reuse=reuse):
    state_dis = cell_critic.zero_state(FLAGS.batch_size, tf.float32)

    def make_mask(keep_prob, units):
      # .. 

    if is_training:
      output_mask = make_mask(hparams.dis_vd_keep_prob, hparams.dis_rnn_size)

    with tf.variable_scope(&#39;rnn&#39;) as vs:
      values = []

      rnn_inputs = tf.nn.embedding_lookup(embedding, sequence)

      for t in xrange(FLAGS.sequence_length):
        if t &gt; 0:
          tf.get_variable_scope().reuse_variables()

        if t == 0:
          rnn_in = tf.zeros_like(rnn_inputs[:, 0])
        else:
          rnn_in = rnn_inputs[:, t - 1]
        rnn_out, state_dis = cell_critic(rnn_in, state_dis, scope=dis_scope)

        if is_training:
          rnn_out *= output_mask

        # Prediction is linear output for Discriminator.
        value = tf.contrib.layers.linear(rnn_out, 1, scope=vs)

        values.append(value)
  values = tf.stack(values, axis=1)
  return tf.squeeze(values, axis=2)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

和文中所说的 `head of discriminator` 一致，代码中 Critic 的实现就是前半部分的 Discriminator，并且复用了 Discriminator 的参数，最后输出也就是一个 scalar，每个 token 的奖励 value；

### <a href="#Objective-Function" class="headerlink" title="Objective Function"></a>Objective Function

我一开始以为公式中的 $r\_t$ 是要计算每个 time step 的，但论文中的注释中说：

> The REINFORCE objective should only be on the tokens that were missing. Specifically, the final Generator reward should be based on the Discriminator predictions on missing tokens.  
> The log probaibilities should be only for missing tokens and the baseline should be calculated only on the missing tokens.

也就是说，只在 missing tokens 上计算相应的 reward。这也很简单，对输出的进行一个 mask 就行：

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
9</code></pre></td>
<td class="code"><pre><code># Generator rewards are log-probabilities.
  eps = tf.constant(1e-7, tf.float32)
  dis_predictions = tf.nn.sigmoid(dis_predictions)
  rewards = tf.log(dis_predictions + eps)

  # Apply only for missing elements.
  zeros = tf.zeros_like(present, dtype=tf.float32)
  log_probs = tf.where(present, zeros, log_probs)
  rewards = tf.where(present, zeros, rewards)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

代码中还实现了很多种 baseline，这里我们只看 Critic 作为 baseline 的情况:

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
29</code></pre></td>
<td class="code"><pre><code>if FLAGS.baseline_method == &#39;critic&#39;:

  # critic loss，只在 missing tokens 上计算
  critic_loss = create_critic_loss(cumulative_rewards, estimated_values,
                                   present)

  # 通过 estimated_values(critic 产生的结果) 来得到 baselines
  baselines = tf.unstack(estimated_values, axis=1)

  ## Calculate the Advantages, A(s,a) = Q(s,a) - \hat{V}(s).
  advantages = []
  for t in xrange(FLAGS.sequence_length):
    log_probability = log_probs_list[t]
    cum_advantage = tf.zeros(shape=[FLAGS.batch_size])

    for s in xrange(t, FLAGS.sequence_length):
      cum_advantage += missing_list[s] * np.power(gamma,
                                                  (s - t)) * rewards_list[s]
    cum_advantage -= baselines[t]
    # Clip advantages.
    cum_advantage = tf.clip_by_value(cum_advantage, -FLAGS.advantage_clipping,
                                     FLAGS.advantage_clipping)
    advantages.append(missing_list[t] * cum_advantage)
    final_gen_objective += tf.multiply(
        log_probability, missing_list[t] * tf.stop_gradient(cum_advantage))

  maintain_averages_op = None
  baselines = tf.stack(baselines, axis=1)
  advantages = tf.stack(advantages, axis=1)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这段代码就对应上面的 $R\_t - b\_t$，只是这里的 $b\_t$ 是由 Critic 产生的；剩下的几种 $b\_t$ 里还有一半 Monte Carlo，一半 Critic 的情况，就不再细说。

### <a href="#Training" class="headerlink" title="Training"></a>Training

训练的套路呢也是类似的，先让预训练 Generator，再是进入 GAN 的一个对抗训练过程之中：

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
<td class="code"><pre><code># pretraining 
fwd_cross_entropy_loss = tf.reduce_mean(fwd_cross_entropy_losses)
    gen_pretrain_op = model_optimization.create_gen_pretrain_op(
        hparams, fwd_cross_entropy_loss, global_step)
    
# Generator Training
gen_loss = (fwd_RL_loss + inv_RL_loss) / 2. # average of forward and backward
[gen_train_op, gen_grads,gen_vars] = model_optimization.create_reinforce_gen_train_op(
         hparams, learning_rate, gen_loss, fwd_averages_op, inv_averages_op,
         global_step)
# Discriminator
dis_train_op, dis_grads, dis_vars = model_optimization.create_dis_train_op(
      hparams, dis_loss, global_step)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

值得一提的是，loss 是 forward 和 backword 的平均值，这点似乎论文中并没有提到，算是作者的一个小心机？2333

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

这篇文章存了有一个礼拜才写完，总算是赶完了；代码部分读的还是很粗糙，接下来会继续把几篇 GAN + NLP 的文章好好读一下写笔记，跑个 Demo，然后试着写一篇 Overview 出来看看能不能忽悠住大家（逃
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">NLP</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color5">论文笔记</a>
