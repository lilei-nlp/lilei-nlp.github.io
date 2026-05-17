---
title: "PKU 这三年"
date: 2023-07-04
categories:
  - blog
tags: []
---

两个月之前，我在语言所的工位上坐着，有些无所事事，在博客的目录敲下

`hexo new "ThreeYearsatPKU"`

创建了这篇文章，以为还有很多时间让我来收拾情绪，写下很多点滴。一转眼，我已经收拾好了行李，和舍友们、同学们告别，离开了理科一号楼，<s>在家里</s>在香港住下来了。**我真傻，哪有那么多的时间留给自己，三年不过弹指一挥间**。反过来想想，是得写点什么，在旅途中设下几个锚点，给以后的回忆留一些线索。

## <a href="#My-Research" class="headerlink" title="My Research"></a>My Research

如果要用一根线来串起我的研究生三年，我觉得也许我的毕业论文会是一个很好的roadmap，大体上，我的经历也都和论文中的内容相关。

在入学之前，我加入了孙老师和微信的合作项目，并且在林老师和李老师的指导下学习如何做科研。前几次的讨论中，鹏导的高标准让我一下子感觉有点紧张，很担心自己是否能够完成好这个项目。

我们花了大约三个月的时间来讨论方向，有几个方面是我到现在依旧在选择 topic 时觉得很重要的：(i) 和时代大潮流相关，当时是预训练模型 BERT 的时代，大家都在思考怎么更好地利用预训练模型做各种各样的任务，那么我们的研究方向必然是要以预训练语言模型为基础的，这样的工作才可能会有影响力。(ii) 这个方向的容量要足够大，足以支撑完成一篇 thesis，但又需要能够有机的串联在一起。大体来说，基本上 \*CL 的 track 都可以认为是一个大的 node，然后往下走一层的节点基本上是能够满足容量足够大。当时我选择的 Efficient NLP 这个领域现在也被逐渐认可成为一个 track，这一 track 下像知识蒸馏和推理加速都是比较大的 area，并且有一定的实际应用意义。

确定了大体的方向后便是文献调研和一些 idea 的尝试，以及每周固定的讨论。CascadeBERT 是我在 WeChat AI 做的第一个 idea，一开始 motivation 是希望能够利用类似 NAS 或者 RL 的方法在 BERT 内部进行一些 skip-layer 或者是在多个 BERT model 之间进行 switch。在激情 coding & training 一个多月之后发现，发现怎么都训不出来，进一步分析 RL 训不出来的原因在于 reward 很稀疏，即很少的样本需要大规模的计算，因此没办法获得充足的监督信号。相反，一个简单的，两个模型级联的 baseline 效果很好，稳定地超过当时的所有 baseline。后续和两位老师讨论后决定，我们把 paper 做成一个分析导向的 paper，弱化级联这一方案的贡献。然而第一次 ACL submission，却因为方案比较简单，reviewer 抨击 novelty 而被拒稿；第二次 EMNLP，投稿了新开的 efficient track，review 意见依旧比较 negative，在 rebuttal 之后依旧是均分 &lt; 3 (3, 2.5, 2.5 ?)。但那次的 AC 非常给力，把这篇 paper 捞成了 Findings。

相比之下，Dynamic KD 就顺利的多，在做完推理加速之后，我的兴趣转移到了如何让模型参数减少但是尽可能不降低性能。这一方面，知识蒸馏就是非常经典的框架。然而，当时有着 TinyBERT，纯蒸馏的性能的天花板基本就在那。于是考虑如何利用更少的数据完成同等效果的蒸馏，又发现有个同期的 MixKD 方法，利用 Mixup 数据做知识蒸馏，很难 beat 掉他们的效果。然后和 Andy 讨论之后发现，当时的工作都在聚焦如何设计更多的 supervision objective 来利用好 teacher model，但实际上，学生模型的能力不断变强，应该更有针对性地挑选教师模型、训练数据和训练目标来实现更加高效的知识蒸馏。因此我们设计了一个动态的框架，来挑选这三个方面，从而提升模型的效果和训练的效率。定完这个框架之后，距离 ddl 只有两个礼拜了，当时手上有的实验结果里，挑选数据部分是我们最早做过探究，有着比较丰富的实验的结果；教师模型选择和蒸馏目标选择则是紧赶慢赶，终于在距离截稿还有一周的时候完成了实验的部分。最后一周疯狂的 polish 论文，并且在这一阶段林老师帮忙 rewrite 了 introduciton，我再一次窥见了我写作方面的不足，并且尝试模仿这样一种写作思路：**将所作的事情抽象成一个 research question，然后娓娓道来你是如何探究这些问题的。**这篇文章的 review 很 positive，录用也变得顺理成章，还有幸入选了 oral。因为疫情缘故，开会是 online 的，也对消弭第一次 oral 的紧张有很大的帮助。现在，我已经不记得在 talk 里说了些什么，**只记得等 runxin 的 talk 讲完之后，一起到东南门的雪地里滚了一大圈**。

在做完推理加速和知识蒸馏之后，我看到 CV 方向有不少对模型融合的探究，即将多个教师模型的能力（分割、分类）蒸馏到一个学生模型中，然而这一想法却鲜有 NLP 方面的探索。我理解这个是新的一种维度的高效化，即我们不再需要部署多个 task-specific 模型，而只需要一个 unified model 来完成这一切。不过当时可能过于看重眼前的 story，回过头来看，LLM 也是这样一个 Unified model，instruction tuning 中有足够的数据来教会他所有的事情。两篇 paper 以后，我已经对科研的套路相对熟悉，因此又是一阵设计、实验和分析，最后延续了 Dynamic KD 中的不确定度的思想，来挑选合适的教师模型进行学习。方法效果依旧是非常不错，beat 掉了 CV 那边几乎所有的方案。投稿的时候是第一次 ARR，还想着挺新鲜，review 意见也不错(4, 3.5, 3, ARR 的 3 分当时根据我的理解应该是 softconf 的 3.5），没成想 commit 之后录成了 Findings，感觉很不值当，于是修改之后再次 commit EMNLP，还是 Findings。想了想，算了，也许这就是命，出来混迟早要还的，低分 Findings 和相对高分 Findings，so take it。

这段在 WeChat AI 的实习经历，除了论文的发表以外，对我科研之路的影响非常之大，令我从一个懵懵懂懂的大四毕业生变成一个对科研、发论文有一些概念的研究生。最重要的是，我也意识到，**you own the project**，所有的合作者都是你的助手，你可以向他们寻求各个层面的帮助，但是最终需要你来 make decision, and take responsiblity。感谢给予过我帮助的所有人。

往后的故事，就是到上海 AI Lab 和晶晶师姐合作（顺便 tla (x，在多模态方面开始做一些尝试，并且我也突然发现，预训练模型的时代，以一个更快地速度过去，而等待着我们的，则是大语言模型的波澜壮阔。也许这一部分，等我 PhD 毕业再来回顾，会更有整体的视角，指不定那个时候又出什么幺蛾子了哈哈。

## <a href="#My-Friends-amp-Love" class="headerlink" title="My Friends &amp; Love"></a>My Friends & Love

Research 的道路也因为朋友们的陪伴变得更加丰富多彩，还记得在 [2020 年总结](https://tobiaslee.top/2021/02/04/Goodbye-2020/) 给 20 级的同学们都写了一句话，三年之后，大家的 paper 都挺多，鹤一烤肉也没少吃，也都有着光明的未来。师兄师姐师弟师妹们也都很给力，一起打狼人杀也一起做科研，德艺双馨。还有我的舍友们，毕业旅行一起去到了霓虹，每天早上出去觅食走走看看，晚上喝酒打牌畅谈人生理想的日子，会是很多年回想起来都感觉很珍贵的回忆，再往前，谁能想到一墙之隔的几个人会成为德扑局上根据 raise 尺度就能锁定范围的*知己*呢？

感谢我的 nvpy 一直以来的陪伴，新冠时期的异地恋着实是煎熬，所有的一切都存在着巨大的不确定性。好在，我们彼此是笃定的，虽有波折，却也坚持了下来。相信，风雨过后的彩虹会更加绚烂，以及，会抓住一切机会创造更加美好的经历。

## <a href="#My-Future" class="headerlink" title="My Future"></a>My Future

简单的展望一下未来的 PhD 生涯：

- 做有影响力的研究：影响力是个很抽象的东西，但有一些基本的原则可循，**有用的研究**往往会产生更大的影响力，例如高质量的开源数据集、工具，开源 &gt;&gt; 闭源。此外，研究的 topic 粒度要粗，能够容纳更多的叶子节点的研究，但是落脚在确保可以泛化的情况下可以在具体的问题上开展，以小见大。
- 保持身心的健康：BMI 或许会是一个主要的指标，多多健身并且希望能够跟梦觉老师学会自由泳。心理健康大抵比较好保持，有空的话多出去走走，找找好吃的！
- 构建自己的知识体系：回顾这三年，学到的很多东西都是零散的分布在各式各样平台上（主要是我的脑子里），没有比较好的整理，进而对复用效率造成影响（脑子会变慢的）。在保持输入（广泛阅读）和输出（尝试提升写技术文章的频率）的同时，在个人知识库中沉淀感兴趣的模块的内容和工具。

祝自己能够如愿，也祝愿 PKU 的大家都有美好的未来！

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">随笔</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2023/07/04/ThreeYearsAtPKU/)