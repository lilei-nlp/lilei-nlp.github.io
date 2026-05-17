---
title: "RaSGAN Notes"
date: 2018-07-05
categories:
  - blog
tags: []
---

在机器之心上看到说 Goodfellow 给一篇刚上 arXiv 的 <a href="https://arxiv.org/abs/1807.00734" target="_blank" rel="noopener">The relativistic discriminator: a key element missing from standard GAN</a> 文章点了赞，就去看了眼，发现确实很有意思。

## <a href="#Intuition" class="headerlink" title="Intuition"></a>Intuition

文章说标准的 GAN（SGAN） 在 Generator（下文用 G 代替） 生成的样本越来越逼真的时候 Discriminator 缺了个东西。什么呢？**相对**的概念，先来看下面这张图片：

![Corgis and Bread](/img/corgi_and_bread.png)

可以看到，我们的 real data 是面包，fake data 是柯基（好萌啊2333)，$ C(x) $ 越大则说明是面包的概率越大。图中列出了三种情况：

1.  真的面包，真的柯基：这种情况二者的区别很明显，因而 $ P(bread |\bar C) = 1 $
2.  真的面包，柯基屁股（很像面包）：这种情况区别就没有第一种那么明显了，因而 $ P $ 有所下降
3.  像狗的面包，真的柯基：和第二种情况类似，**相对的区别度**降低了， $ P $ 同样有所下降

有了一个模糊的印象之后，我们展开说说这个相对究竟是什么。

## <a href="#Arguments-of-RaGAN" class="headerlink" title="Arguments of RaGAN"></a>Arguments of RaGAN

**相对**，一言以蔽之：Discriminator（下文中用 D 代替）衡量样本真实性的时候，应该要**同时**利用 real data 和 fake data，衡量的由绝对的真假变成**相对的为真或为假**的概率。 作者从三个方面论述了其观点：

### <a href="#Priori-Knowledge" class="headerlink" title="Priori Knowledge"></a>Priori Knowledge

**先验知识**的利用，即每次我们喂给 Discriminator（下文中用 D 代替） 的样本中，基本上是一半 real data，一半 fake（Generator generated）。也就是说，**不知道这个前提的话**，那么如果 G 生成的样本（比如说图片）能够以假乱真的话，那么 D 会认为所有的样本都是 real 的，而如果知道这个前提，那么当 fake 比 real 更 real 的时候，discrinimator 应该给 real samples 打低分（认为他们是 fake） 而不是认为所有的 samples 都是 real。因为在看到了更 real 的 fake samples 之后，相对地，利用先验知识，我们会认为不那么 real 的 samples（比如狗面包）是 fake 的。而 SGAN 的训练中并没有利用到这一部分先验知识。

### <a href="#Divergence-Optimization" class="headerlink" title="Divergence Optimization"></a>Divergence Optimization

我们知道，SGAN 在训练 D 的时候事实上是在 minimize 生成器分布和真实数据分布的 JS-Divergence，而 JS-Divergence 在两个分布相同时达到最小值，其表现是 D 无法区分 real data 和 fake data，认为其为真的概率均为 0.5；JS-Divergence 在两个分布差异较大的时候较大，即 D 认为 real data 为真的概率为 1，而认为 fake data 为真的概率为 0。理想的训练过程是如下 (C) 图所示：

![Divergence Optimization Process](/img/divergence_argument.png)

但事实上，我们在训练的时候**一味地希望生成的图片足够逼真**，即一直在将 `D(fake_data)` 向 `1` 推，而不管 `D(real_data)`。这有做是达不到最小值的，这样的 optimization 过程是存在问题的，WGAN 论文中似乎也有提到这一点。

### <a href="#Gradient" class="headerlink" title="Gradient"></a>Gradient

WGAN 等一系列对 GAN 做了改进的 GAN 被称为 IPM-based GAN，作者将其梯度和标准的 GAN 进行了对比：

![SGAN Gradient](/img/gradient_sgan.png)

下面是 IPM-based GAN 的梯度：

![IPM-based GAN Gradient](/img/gradient_ipm.png)

当下面这些条件满足的时候，两式相等：

1.  在训练 D 的时候，$ D(x\_r) = 0, D(x\_f) = 1 $
2.  训练 G 的时候，$ D(x\_f) = 0 $
3.  $C(x) \in \mathit{F}$，其中 $F$ 是一类实值函数（这个一般都能满足）

考虑到 IPM-based GAN 相对于 SGAN 具有更好的稳定性，可以推断，如果将 SGAN 推向 IPM-based GAN，能够提高其稳定性。怎么才能达到这个转变呢？如果我们认为 D 足够好，即能够在训练 G 时做到认为 $D(x\_r) = 1 \\ and \\ D(x\_f) = 0 ​$（这是一个比较强的假设，但是在训练一开始是能够满足的），而在训练 D 的时候 $D(x\_r) = D(x\_f) = 1 ​$，如开头图片的 (b) 所示那种情况，那么唯一缺少的条件就是 $D(x\_r) = 0 ​$。但是在 SGAN 中，$D(x\_r)​$ 只跟 real data 有关，也即是**绝对的真假**，如果 G 生成的样本能够影响 $D(x\_r)​$，让它变得不那么真实，也即**相对的真假**。这个时候，$D(x\_r)​$ 才可能变成 0。总结起来一句话就是，在 $D(x\_f)​$ 提升的时候 $D(x\_r)​$ 相应地减少（这是一种相对的变化），这样 GAN 的训练才能更加稳定。

## <a href="#Methods" class="headerlink" title="Methods"></a>Methods

### <a href="#Relativistic-GAN" class="headerlink" title="Relativistic GAN"></a>Relativistic GAN

如何把这种相对为真和为假的概率考虑进去？很简单，只要把 logits 做一个简单相减，即：

![Relative Loss](/img/relative_gan.png)

这里用 `sigmoid` 函数将 logits 转化成概率，然后再取对数；我们可以很容易地把这个式子进行泛化，即用更一般的函数（不一定是似然函数）来替换它。

### <a href="#Relativistic-average-GAN" class="headerlink" title="Relativistic average GAN"></a>Relativistic average GAN

不过这个时候，我们发现 GAN 中 D 的功能已经悄然发生的改变，由原来的：**衡量输入数据为真的概率**，变成了输入的数据与其对立类型随机的一个样本（如果输入为 real data，那么衡量其比 fake data 更像真的概率，反之亦然）相比**更**像真的的概率。那么，更一般地，我们利用对立类型的**平均真实度**来作为更可靠的参照对象。

![Relativistic average GAN](/img/ragan.png)

所谓平均真实度，就是对整个（或者一个 mini-batch） real data 和 fake data 求其 $D(x)$ 的数学期望，这样的估计能够更加整体的反映训练在这一时刻生成器生成的 fake data 的真实程度。同样，我们可以轻易地将其泛化到别的函数之上。最后整个算法流程如下：

![Algorithm of RaGAN](/img/algorithm_ragan.png)

## <a href="#Toy-Demo" class="headerlink" title="Toy Demo"></a>Toy Demo

原作给了很多例子来说明 RaGAN 的效果，并且也提供了相应的<a href="https://github.com/AlexiaJM/RelativisticGAN" target="_blank" rel="noopener">代码</a>。这里我就拿 MNIST 做一个小小的 Demo，在原版的 GAN 的 demo 上做了一个很小的改动，就是把 loss 计算中的 logits 进行对应的相减。

### <a href="#Generator" class="headerlink" title="Generator"></a>Generator

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
17</code></pre></td>
<td class="code"><pre><code># Generator Net
Z = tf.placeholder(tf.float32, shape=[None, 100])

G_W1 = tf.Variable(xavier_init([100, 128]), name=&quot;G_W1&quot;)
G_W2 = tf.Variable(xavier_init([128, 784]), name=&quot;G_W2&quot;)

G_b1 = tf.Variable(tf.zeros([128]), name=&quot;G_b1&quot;)
G_b2 = tf.Variable(tf.zeros([784]), name=&quot;G_21&quot;)

theta_G = [G_W1, G_W2, G_b1, G_b2]

def generator(z):
    G_h1 = tf.nn.relu(tf.matmul(z, G_W1) + G_b1)
    G_log_prob = tf.matmul(G_h1, G_W2) + G_b2
    G_prob = tf.nn.sigmoid(G_log_prob)

    return G_prob</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#Discriminator" class="headerlink" title="Discriminator"></a>Discriminator

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
<td class="code"><pre><code># Discriminator Net
D_W1 = tf.Variable(xavier_init([784, 128]), name=&quot;D_W1&quot;)
D_b1 = tf.Variable(tf.zeros(shape=[128]), name=&quot;D_b1&quot;)

D_W2 = tf.Variable(xavier_init([128, 1]), name=&quot;D_W2&quot;)
D_b2 = tf.Variable(tf.zeros(shape=[1]), name=&quot;D_b2&quot;)

theta_D = [D_W1, D_W2, D_b1, D_b2]

def discriminator(x):
    D_h1 = tf.nn.relu(tf.matmul(x, D_W1) + D_b1)
    D_logit = tf.matmul(D_h1, D_W2) + D_b2
    D_prob = tf.nn.sigmoid(D_logit)

    return D_prob, D_logit</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#RaGAN" class="headerlink" title="RaGAN"></a>RaGAN

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
<td class="code"><pre><code># Ra GAN  simple version 
mnist = input_data.read_data_sets(&#39;MNIST_data&#39;, one_hot=True)
X = tf.placeholder(tf.float32, shape=[None, 784], name=&quot;X&quot;)


G_sample = generator(Z)
D_real, D_logit_real = discriminator(X)
D_fake, D_logit_fake = discriminator(G_sample)
# discriminator 
# log( real_logits - fake_logits)
D_loss = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits=D_logit_real - D_logit_fake, labels=tf.ones_like(D_logit_fake)))

# generator loss
# log(fake_logits - real_logits )
G_loss = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits=D_logit_fake - D_logit_real, labels=tf.ones_like(D_logit_fake)))

# optimizer
D_solver = tf.train.AdamOptimizer().minimize(D_loss, var_list=theta_D)
G_solver = tf.train.AdamOptimizer().minimize(G_loss, var_list=theta_G)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#Train" class="headerlink" title="Train"></a>Train

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
58</code></pre></td>
<td class="code"><pre><code>def sample_Z(m, n):
    return np.random.uniform(-1., 1., size=[m, n])


steps = 1000001
mb_size = 128
Z_dim = 100


def plot(samples):
    fig = plt.figure(figsize=(4, 4))
    gs = gridspec.GridSpec(4, 4)
    gs.update(wspace=0.05, hspace=0.05)

    for i, sample in enumerate(samples):
        ax = plt.subplot(gs[i])
        plt.axis(&#39;off&#39;)
        ax.set_xticklabels([])
        ax.set_yticklabels([])
        ax.set_aspect(&#39;equal&#39;)
        plt.imshow(sample.reshape(28, 28), cmap=&#39;Greys_r&#39;)

    plt.show(block=False)
    return fig


with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    num = 0
    for i in range(steps):
        X_mb, _ = mnist.train.next_batch(mb_size)
        
        _, D_loss_curr = sess.run([D_solver, D_loss], feed_dict={
            X: X_mb,
            Z: sample_Z(mb_size, Z_dim)
        })
        # 注意，这里的 feed_dict 和原来不一样了
        _, G_loss_curr = sess.run([G_solver, G_loss], feed_dict={
            X: X_mb, Z: sample_Z(mb_size, Z_dim)
        })
        # 下面的 vanilla GAN 的 loss
        # _, G_loss_curr = sess.run([G_solver, G_loss], feed_dict={Z: sample_Z(mb_size, Z_dim)})

        if i % 100 == 0:
            print(&quot;Step %d&quot; % i)
            print(&quot;G loss: %f&quot; % G_loss_curr)
            print(&quot;D loss: %f&quot; % D_loss_curr)
            print()

        if i % 1000 == 0:
            samples = sess.run(G_sample, feed_dict={Z: sample_Z(16, Z_dim)})
            fig = plot(samples)
            fname = &#39;{}.png&#39;.format(str(num).zfill(5))
            plt.savefig(fname, bbox_inches=&#39;tight&#39;)
            print(&#39;saved image &#39; + fname)
            num += 1
            # plt.clf()
            plt.close(fig)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

GAN 的难训练是**臭名昭著**了，作者通过考虑引入相对这个概念来使得训练过程变得更加稳定。这类见微知著的工作，还有前不久的 IndRNN 真的非常符合我的胃口了。

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">Deep Learning</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">GAN</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2018/07/05/RaGAN-Notes/)