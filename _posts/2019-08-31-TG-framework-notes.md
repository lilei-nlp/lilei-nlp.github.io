---
title: "文本生成框架体验记录"
date: 2019-08-31
categories:
  - blog
tags: []
---

我之前主要在做的工作像嘻哈歌词、主题写作都是是文本生成任务，而大部分的生成任务都可以归结到 Seq2Seq 框架下来，最近也接触了利用框架跑了一些 Machine Translation 的数据集。在实现 idea 的时候，我喜欢的一种方式是快速实现一个简单的 Baseline，跑通模型看看主要的指标例如 BLEU 的效果怎么样，然后进一步地做模型的设计和修改。要快速迭代，一方面需要对基本框架 TF、PyTorch 的 API 足够熟悉，另一方面也需要有一个趁手的 toolkit，来避免做一些重复的工作（例如，Tokenization, Load Pre-trained Word Embedding）等。**这些 pipeline 在学习初期很有必要自己亲手实现以了解细节，但是在需要快速迭代的时候，就不要重复造轮子了**。这里，我对使用过（大部分是浅尝辄止）的框架写一下主观上的感受，主要考虑几个方面：

- 能否开箱即用
- 定制 Model 的难易程度
- API 是否易于理解

希望对选择困难的朋友们有所帮助。

<span id="more"></span>

## <a href="#AllenNLP" class="headerlink" title="AllenNLP"></a>AllenNLP

<a href="https://allennlp.org/" target="_blank" rel="noopener">AllenNLP</a> 是 Allen 人工智能研究所的产品，这个组最近也产生了一系列的很不错的文章，大牛 Noah A Smith 也在这里，牛组出品的质量一般都还不错。

> GitHub 地址： <a href="https://github.com/allenai/allennlp" target="_blank" rel="noopener">https://github.com/allenai/allennlp</a>
>
> 文档地址：<a href="https://allenai.github.io/allennlp-docs/" target="_blank" rel="noopener">https://allenai.github.io/allennlp-docs/</a>
>
> Tutorial：<a href="https://allennlp.org/tutorials" target="_blank" rel="noopener">https://allennlp.org/tutorials</a>

Tutorial 给的比较齐全，基本照着抄一遍就能明白是怎么玩的了，**源码也很值得学习**。安装方面强烈建议使用虚拟环境 Anaconda，因为在很多服务器上我们并没有 root 权限，虽说可以通过 `pip -u` 来指定只给当前用户安装，但是不同人物需要的库版本可能冲突，**会导致新/旧代码出现不兼容无法运行的情况**，所以，conda 大法好（尖叫

用 AllenNLP 我的遇到的一个坑的就是：它要求 **PyTorch 的版本 &lt; 1.2.0**，并且目前用 conda 安装的版本只支持到 CUDA 9，对于需要使用 FP16 提升速度的 Model 不是很友好。如果卸掉 conda 带的 PyTorch 版本，从官网安装 CUDA 10版本，请注意：**指定 PyTorch 版本 = 1.1.0**。还有一个坑点就是**小版本之间的兼容性**不好，有些 API 在你升级之后会发生路径的移动，所以，建议**固定一个版本进行使用**。

目前使用下来，**AllenNLP 是唯一一个支持在训练过程中利用 BLEU 做评测的框架**，冲着这一点，加分！

定义模型，很轻松，因为整个 AllenNLP 的设计 Model、Trainer 以及 Predictor 这样的一个思路，各个模块划分的很清楚，也能够间接地敦促我们据此来写代码，比面条一样的一坨不知道高到哪里去了。如果需要对于 Metric 定制，也可以通过自己魔改 Trainer 来实现相应的计算和日志功能。

## <a href="#FairSeq" class="headerlink" title="FairSeq"></a>FairSeq

Facebook 团队出品的 Seq2Seq 框架，**机器翻译首选**，配合 CUDA 10 使用，速度和效果都是顶呱呱。

> 文档地址：<a href="https://fairseq.readthedocs.io/" target="_blank" rel="noopener">https://fairseq.readthedocs.io/</a>
>
> GitHub 地址：<a href="https://github.com/pytorch/fairseq" target="_blank" rel="noopener">https://github.com/pytorch/fairseq</a>
>
> Example Translation: <a href="https://github.com/pytorch/fairseq/tree/master/examples/translation" target="_blank" rel="noopener">https://github.com/pytorch/fairseq/tree/master/examples/translation</a>

安装方面就按文档来就可以了，丝滑无坑，基本做到开箱即用，参数的说明文档也给的很明白了，不行的话查看一下源码也就知道一系列的配置是干嘛的。

FairSeq 的一个很大的优势就是更新速度很快，基本上一有新的模型都能在 repo 里找到，像最新的 ReBERTA，也已经在库里等着你跑了（毕竟是自家的 model）。**大公司出品的好处，就是自家的代码基本都用这个框架，follow 起来比较方便**。

模型定制方面，我最近在机器翻译任务上对 Transformer 进行了一些改进，**小的改动还是比较轻松的，但是要从头定制，不方便也没必要**，适合快速地验证一下 idea。

## <a href="#OpenNMT" class="headerlink" title="OpenNMT"></a>OpenNMT

看名字就知道是做机器翻译的一个框架，它的一个很大的好处就是同时提供了 **PyTorch 和 TensorFlow 两个版本**，工业界很多时候部署都是要用到 TensorFlow 的，所以这是他的一个优势所在。

> GitHub: <a href="https://github.com/OpenNMT/OpenNMT-tf" target="_blank" rel="noopener">https://github.com/OpenNMT/OpenNMT-tf</a>
>
> Documentation: <a href="http://opennmt.net/OpenNMT-tf/installation.html" target="_blank" rel="noopener">http://opennmt.net/OpenNMT-tf/installation.html</a>
>
> QuickStart: <a href="http://opennmt.net/OpenNMT-tf/quickstart.html" target="_blank" rel="noopener">http://opennmt.net/OpenNMT-tf/quickstart.html</a>

相比 fairseq，OpenNMT 的一个缺点就是**更新不够及时**，很多新的东西（例如预训练大礼包），目前来看 OpenNMT 还是没有相应的支持。

开箱即用方面，对于一般的翻译任务都是能够胜任的，但是仔细对比一下可以发现，TF 版本对于训练参数的配置的支持是要比 PyTorch 版本好一些，所以大家可以在阅读文档对两个版本有个基本的了解之后，根据自己需要来进行选择。

定制方面没有尝试，不多说。

## <a href="#Texar" class="headerlink" title="Texar"></a>Texar

Texar 是 CMU 邢波老师组出品的框架，一作 Zhiting Hu 在生成领域也有很多代表作。

> GitHub: <a href="https://github.com/asyml/texar" target="_blank" rel="noopener">https://github.com/asyml/texar</a>
>
> Document: <a href="https://texar.readthedocs.io/en/latest/" target="_blank" rel="noopener">https://texar.readthedocs.io/en/latest/</a>
>
> Examples: <a href="https://github.com/asyml/texar/tree/master/examples" target="_blank" rel="noopener">https://github.com/asyml/texar/tree/master/examples</a>

我个人比较喜欢的是里面一些关于 RL 的例子，因为 RL 做 NLP 大部分时候需要用到 Policy Gradient 这一方法，所以其内置的 `PGAgent` 对于**实现关于 RL 的想法是很方便的**。类似有着 RL 相关 NLP代码的还有上交出品的 <a href="https://github.com/geek-ai/Texygen" target="_blank" rel="noopener">Texygen</a>，但似乎最近已经不再更新，因此如果需要复现上交组 RL 的 Paper 相关代码可以参考一下，使用的话还是推荐 Texar。

API 方面的设计其实官方提供的图就很不错，大家可以参考一下：

![Texar-Stack](/img/texar-stack.png)

## <a href="#HuggingFace" class="headerlink" title="HuggingFace"></a>HuggingFace

HuggingFace 我没用过，但是组里师兄有不少用它的，因为它对**新模型的支持非常及时**。<a href="https://github.com/huggingface/pytorch-transformers" target="_blank" rel="noopener">PyTorch-Transformer</a> 这个 repo 集预训练模型之大成，像 GPT-2、XLNet 和 RoBerta 它都有收录。最重要的一点在于，原先像 Google 官方提供的 BERT 代码只能够跑分类任务，但是 HuggingFace 可以跑生成！

> `run_generation.py`: GPT, GPT-2, Transformer-XL and XLNet for conditional language generation

之前看到 ConvAI 这个对话挑战上的第一名就是 HuggingFace，所以说有刷榜需求的同学们，可以尽早把它添加到你的武器库里，相信能够让你如虎添翼。

## <a href="#Misc" class="headerlink" title="Misc"></a>Misc

最近没怎么更新 Blog，一方面是因为实验室的砖确实比较多，搬不怎么过来，另外一方面，也是这段时间**忙于 coding，没有时间整理**。但还是要保持读文章的习惯，做好笔记！

然后是清华的 NLP 小组收集整理了一波 <a href="https://github.com/THUNLP-MT/TG-Reading-List" target="_blank" rel="noopener">Text Generation</a> 的文章，大家可以配合着这些框架，把一些有意思的文章复现一下，说不定，一篇顶会就出来啦！

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">NLP</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">Text Generation</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2019/08/31/TG-framework-notes/)