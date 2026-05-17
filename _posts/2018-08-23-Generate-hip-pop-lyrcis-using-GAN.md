---
title: "DeeCamp 2018 - AI 有嘻哈，惊掉你下巴"
date: 2018-08-23
categories:
  - blog
tags: []
---

## <a href="#前言" class="headerlink" title="前言"></a>前言

非常有幸参与了 2018 Dee Camp 人工智能夏令营，在这一个月的时间里面，我们享受了来自学界、业界各种大咖的分享，同时也和导师一起参与到一个项目中，将所学知识进行落地。我们组的题目是 AI 有嘻哈，利用相关技术进行嘻哈歌词的生成。

先来看看模型的效果，给定第一句，生成接下来的几句，猜猜下面哪句是模型生成的哪句是原作？

> 不是乐理专修 做点儿曲式研究 我们的力量来自宇宙 自己的节奏
>
> 不是乐理专修 所有听的观众 打破他们传统 进到环球 继续让你感受

再来一个：

> 自己就带上了有色眼镜 金钱摧毁多少事情 瓦解你的中枢神经
>
> 自己就带上了有色眼镜 我只想把世界分的更清 却发现自己却模糊了心

正确答案是：**第一行都是模型生成的，第二行是原作**，你答对了吗？有没有被惊艳到，我们组内的测试结果显示，大家的正确率不超过 30%，值得一提的是：**输入都是来自于测试集中**，没有参与到模型的训练中。

**因为某些原因，代码和数据暂时无法开源，以后有机会会整理出来**。

<s>更多的例子和项目的代码已经上传到 <a href="https://github.com/TobiasLee/Chinese_Hip-pop_Generation" target="_blank" rel="noopener">Github</a>，欢迎大家 Star。</s>

## <a href="#数据" class="headerlink" title="数据"></a>数据

创新工场的导师为我们提供了 10 w 条嘻哈歌词，并且已经将一些不符合社会主义核心价值观的句子标注了出来。数据的预处理主要步骤如下：

- 在对句子进行筛选之后，我们利用 <a href="https://github.com/fxsjy/jieba" target="_blank" rel="noopener">Jieba</a> 进行分词，观察到单句长度集中在 8~10 左右；
- 在利用 Tensorflow 中的 Tokenizer 进行 tokenize 并构建 word2idex 字典后，词表大小在 11000 左右，考虑到这个大小还可以接受，没有做限制词表大小的操作；
- 利用 `pad_sequence` 将句子 padding 到 20（和 SeqGAN 中相同）；
- 构建 x-y pair，利用上一句预测下一句（导师后来建议可以借鉴用 Skip-gram 的思路，同时预测上一句和下一句，但没有时间去尝试了），分割数据集

<s>知道大家都不喜欢洗数据，所以我们的训练数据也已经放到了 <a href="https://drive.google.com/drive/folders/1QrO0JAti3A3vlZlUemouOW7jC3K5dFZr" target="_blank" rel="noopener">Google Drive</a>，大家自取即可。</s>

## <a href="#模型" class="headerlink" title="模型"></a>模型

我们的生成模型的整体基于 SeqGAN，并对其做了一些修改，模型架构如下：

![AI-Hippop](/img/AI-hippop.jpg)

主要改动有两点：

1.  增加输入语句的编码：这一点类似 Seq2Seq 的 Encoder，SeqGAN 原本的 initial state 是全 0 的，为了将上文的信息传递给生成器，我们采用了一个简单的全连接层（Fully Connected Layer），将输入句子的 Word Embedding 经过一个线性变化之后作为生成器的 LSTM。事实上也可以尝试使用 RNN（LSTM）来作为 Encoder，不过这样模型的速度可能会比较慢。

2.  将原先 Generator 的 Loss Function 改为 Penalty-based Objective：在训练模型的过程中我们发现，模型在 Adversarial Training 多轮之后出现了严重的 mode collapse 问题，比如：

    > 别质疑自己  
    > 遮罩错的消息不要过得消极  
    > 世间人都笑我太疯癫  
    > 世间人都笑我太疯癫
    >
    > ​
    >
    > 守护地狱每座坟墓  
    > 世间人都笑我太疯癫  
    > 你不知道rapper付出多少才配纸醉金迷  
    > 世间人都笑我太疯癫
    >
    > ​
    >
    > 但却从来没有心狠过  
    > 如果你再想听  
    > 你不知道rapper付出多少才配纸醉金迷  
    > 你不知道rapper付出多少才配纸醉金迷

    可以看到 `世间人都笑我太疯癫` 和 `你不知道rapper付出多少才配纸醉金迷` 占据了我们生成的结果。mode collapse，简单来说就是输入的改变不会影响生成的结果。为此我们调研了一些 Paper，最终采用了<a href="https://www.ijcai.org/proceedings/2018/0618.pdf" target="_blank" rel="noopener">SentiGAN</a> 中提出的 Penalty-based Objective Function：

    ![Penalty based Objective Function](/img/pbof.png)

    SeqGAN 是将原先 Discriminator 的认为 Generator 生成句子为真的概率作为奖赏，Generator 通过最大化奖赏（最小化其相反数）来更新自己的梯度；而 SentiGAN 则恰好相反，将 Discriminator 判断为假的概率作为惩罚，Generator 通过最小化惩罚来更新自己，并且去掉了概率上的 log 函数。作者分析其起作用的效果在于：一、去掉 log，可以视作 WGAN 中的 generator loss（这一点我持怀疑态度，因为代码中并没有对梯度进行裁剪来满足 WGAN 所需的 Lipschitz 条件）；二、惩罚和奖赏存在一个 $ 1- V = D$ 的关系，根据 $ G(X|S;\theta\_g) V(X) = G(X|S;\theta\_g)(1-D(X;\theta\_d) )=G(X|S;\theta\_g) - G(X|S;\theta\_g)D(X;\theta\_g)$ ，这样会让 generator 偏向于一个更小的 $G(X|S;\theta\_g) $ ，即概率比较小的句子，从而避免生成很多重复（概率比较大）的句子。Anyway，在我们的模型上，SentiGAN 的 loss 立竿见影，更多的细节请大家参考原作。

## <a href="#押韵" class="headerlink" title="押韵"></a>押韵

嘻哈歌词非常重要的一个特点就是句与句之间的押韵，我们在实现这一功能的时候尝试了两种方案：

1.  Reward based，在 reward 函数上增加额外的押韵奖赏项， $ \\ r\_{rhyme}$：对 Generator 的生成的句子和输入的句子进行押韵的判断，如果押韵，则提供额外的奖赏。
2.  Rule-based，生成时只对押韵的词进行采样：在生成句尾的词的概率分布时候，通过获取和输入句尾押韵的词，只在这些押韵的词进行采样。

方法一，如果能够通过设计 reward function 就能实现押韵的功能，那模型就是完全 end2end，非常 fancy 了。但是理想很丰满，现实很骨感，经过几天的调整押韵奖赏的权重，都没能看到押韵率（我们设置的用于检测押韵奖赏效果的指标，每个 batch 中和 input 押韵的句子的比例）的上升 。我们怀疑是**这种奖赏的结合会让 Generator 产生混淆，并不能明确自己 reward 来自何处**，应该需要更加具体的一些限制才能够实现这一方法。

方法二，一开始我是拒绝这么做的，用基于规则的方法不是我的理想。

![真香](/img/zx.gif)

但是为了做出产品来，我还是屈服了。但还有一个问题摆在面前：怎么知道生成的是句尾呢?导师提醒我们，我们可以把**输入倒过来**。这是 NMT 中常用的一个手段，对于 LSTM，句子是真的还是反的差别不大，即使有差别，也可以通过一个 Bi-LSTM 来捕获不同顺序的信息。而为了知道哪些字词是押韵的，我们实现制作了一张 `vocab_size x vocab_size` 的大表 `rhyme`，如果两个词（index 分别为 i, j）押韵，则 `rhyme[i, j]` 非 0，否则为 0。

![Rhyme Mask](/img/rhyme_module.png)

如上图所示，如果我们的输入为“你真美丽”，句尾词为“美丽”，韵脚为 i；最终采样结果只会在押韵的词中采样，示例的采样结果为“春泥”。

据此，我们就可以对生成过程的第一个词的词表分布进行一个 mask 操作，使得非押韵的词的概率都变成 0，就能够保证押韵了，代码片段如下：

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
<td class="code"><pre><code># 获取 input 的最后一个词
first_token = self.inputs[:, 0]  # (batch_size, 1)
# 控制押韵的概率, 现在设置为 1.0 ，即 100% 押韵
select_sampler = Bernoulli(probs=1.0, dtype=tf.bool)
select_sample = select_sampler.sample(sample_shape=self.batch_size)
# 获取对应的 index 押韵行
token_rhyme = tf.cast(tf.gather(self.table, first_token), tf.float32)
# 进行 mask
prob_masked = tf.where(select_sample, tf.log(tf.multiply(token_rhyme, tf.nn.softmax(o_t))),tf.log(tf.nn.softmax(o_t)))
# 根据 mask 之后的概率分布进行采样
next_token = tf.cast(tf.reshape(tf.multinomial(prob_masked, 1), [self.batch_size])</code></pre></td>
</tr>
</tbody>
</table>
</figure>

不过这个制表的过程比较耗费时间（大约跑了 3 个小时，i7）。另一种思路是可以根据韵脚对字词进行分类，将相同韵脚的词的 index 编到一起，这样我们可以通过获取每个词的韵脚来知道目标词的范围，而不用挨个的去判断是否押韵。

## <a href="#结语" class="headerlink" title="结语"></a>结语

在 DeeCamp 的一个月里，除了讲座以外，更让我觉得收获许多的是认识了很多非常优秀的小伙伴，以及开拓了自己的视野，看到了更大的舞台。我愈发地坚信，**未来的十年，是属于人工智能的，是属于愿意投身于这波浪潮中我们的**。也再次向大家安利一波 <a href="https://challenger.ai/" target="_blank" rel="noopener">DeeCamp</a>，如果你对人工智能感兴趣，明年暑假，千万别错过~ 最后希望 DeeCamp 能够一直办下去，并且越办越好！

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">Deep Learning</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">Text Generation</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color2">SeqGAN</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2018/08/23/Generate-hip-pop-lyrcis-using-GAN/)