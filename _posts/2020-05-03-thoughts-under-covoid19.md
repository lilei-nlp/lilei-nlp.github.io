---
title: "三分之一，二零二零"
date: 2020-05-03
categories:
  - blog
tags: []
---

> 2020 年的开端，或许会成为我们十年后为数不多有着清晰记忆的四个月，光怪陆离到让人不敢相信它是真的，而每次口罩下的呼吸却又在真真切切地提醒着我们，发生了什么。

新冠疫情、A 股熔断、美股熔断、原油暴跌，这是一个让巴菲特都感慨“我还是太年轻的”的一年，也是我在 XDU 的最后一年。在学校里，学期内的划分让我对时间的流逝明确的感受，一般半年也就该写篇博（fei）客（hua）来吹吹水了，在家，除了觉得日子过的很快，确实没什么想要写点东西的冲动，所以导致一直没有更新。但说实在话，很久不更新我也有一种愧疚感，博客于我而言，是记录，也是纪念。**输出文字，是很重要的一种能力，也能敦促我反过来系统性地回顾、总结，写的是生活也好，科研也罢，或许这些日后看来非常稚嫩无知的文字，是这一路最忠实的见证者。**

## <a href="#Life" class="headerlink" title="Life"></a>Life

我一直对于老家不是特别感冒，一直在杭州读书，逢年过节回老家赶场式地吃完饭拜年就收拾回杭州了，确实很难说有特殊的情感。今年这个特殊情况，在老家呆了很长一段时间。纠结了很久才买的“买前生产力，买后爱奇艺”的 ipad， 居然也派上了很大的用场，最后甚至惊奇地发现还涨价了，小巴菲特石锤。

在老家的这段时间里，我在田间地头拔过葱（<s>因为想吃葱油拌面</s>），帮爷爷种土豆（<s>被大伯强制要求去的</s>），去鱼塘抓了一堆鱼（<s>我就是站边上看看</s>），也算是体验了一把乡村生活。看着一个一个安放小土豆的爷爷，我眼前又浮现出很多年前夏天他在收割水稻的身影，恍然间，爷爷已经耄耋之年，岁月流逝，陪他劳作的从儿子变成了孙子。**从他身上，我能感受到那一代人与土地绵长不绝的关系，而我的身上流淌着他的血液，亦无法与脚下的土地分割。**所以，即使未来有可能到国外读 PhD，我也会回来的，何处心安是吾乡，大洋彼岸的空气再甜，也填不满一个炎黄子孙的胃口与内心。另外，想到身边不少同学的爷爷奶奶已经去到了另一个美好的地方了，我感到一阵庆幸，希望所有的爷爷奶奶都能健康长寿，能让他们的孙子孙女陪他们钓钓鱼、种种菜。

回到杭州之后，和初中、高中同学都聚了一聚。不少初中同学已经走上了工作岗位，逐渐在向社会人转变。隔阂到说不上，确实能聊的东西不多，毕竟学校和社会之间的鸿沟，慢慢在同学情谊上拉出缝隙，变成平行线，也许只是时间问题。高中同学还好，在继续求学的居多，所以还都有的聊，甚至还能联机打把游戏。

再就是大学同学了，真的，谁能想到呢，想象中每天嗨到昏厥，彻夜把酒言欢的毕业季，就要这样结束了。希望能早日回到学校，毕竟，**我所理解的生活，就是和喜欢的一切在一起**。

## <a href="#Research" class="headerlink" title="Research"></a>Research

最近正在 On-going 的一些 project 还是蛮有意思的，朝着我之前说的更具备 insight 的方向努力。对于做 solid 的研究，我觉得首先要问自己下面三个问题：

- 文献调研足够了吗？用好搜索引擎，对于 NLP 来说，主要是 Google Scholar + ACL Anthology。当然不同会议的侧重也会有所不同，例如，Data Mining 的 paper 还是得去 KDD、ICDM 上找找 related paper。把这些 paper 写成一个 list，尝试提炼出其脉络和框架，然后把自己的想法放到一个尽可能高的位置，**即，能把别人的工作放进你的框架里**。即使是已经确保没有人做过了，也要关注一下 arxiv，因为现在是个百舸争流的时代，虽说同时期的工作可以不作为被拒稿的理由，但是从别人的paper，找到自己没有看到的 point，复盘学习，也是很有价值的。
- 问题定义清楚了吗？A + B 式的研究逐渐成为过去式，并且，随着工具门槛的降低，算法工程师写代码这一侧的能力不再成为其核心竞争力，即，solution 的 toolkit 大家都有，那么，只有能够找到有价值的问题、定义出问题才是不可替代的。这方面我觉得我还是有所欠缺的，没想明白就开始动手，需要向 Senior 的同学老师学习。
- 实验可复现吗？对于 baseline，一方面要选比较强大的模型，比如现在做 NLP，是骡子还是马我觉得起码得和 BERT 比一比；另外，对于 baseline 的性能，也要考虑周到，不是随便跑一组看的过去的就好了，因为 task 不太一样，可能原先的 setting 并不适合新的问题，对于常见的几个超参跑个 grid search 是必需的。此外，注重统计上的显著性，最简单的，多跑几 个seed ，看看 mean 和 std，确保提升是显著的。

另外一点，也是看刘洋老师的<a href="https://www.bilibili.com/video/av94614779/" target="_blank" rel="noopener">研究生论文选题方法</a>以及和师兄交流得到的一个感悟，就是要**尽可能做成体系的工作**。对于一个博士生来说，他的 thesis 应该就是他发表的若干文章，应该落在一个三角形之内，在这个三角形之内的内容，就是他**没有人比我更懂**的领域。或许对于一个硕士生来说，在一个领域开坑的难度角度，但是也应该尽可能把工作 focus 到一个较大的坑里，聚沙成塔，再努力挖出自己的一些小坑。一来，这样在写论文的时候，能够更有体系的组织已经发表的工作；二来，很多问题并不是一篇文章就能够解决的，多篇文章从不同的视角，去解决一个 topic，能够更加全面和完善，也能够在以后的找工作的时候更好地阐述自己研究生期间究竟做了什么。

最后，就上我看刘洋老师视频做的笔记，希望能常常翻看，不忘初心。

> 问题应该是怎么样的？
>
> - 重要性：work on important problem, so you can do important work
> - 创新性：
> - 基础性：做树的根节点的工作，少做叶子节点；
> - 复杂性：多个问题的 子问题
> - 系统性：子问题之间要有关联，e.g. 难表示 低资源 缺知识 不要做 xx 的若干问题研究 可以多向老师沟通 选一个适合自己的、感兴趣的题目
> - 可行性：该问题应该具备在短期内被解决的可能性
> - 承接性：课题组的积累，利用组里的资源优势
> - 适合性：自己感兴趣 能让自己觉得非常 exciting 斗志昂扬的 能发挥自己的优势
>
> 如何选题：
>
> 战略性的思考，顶层设计
>
> 1.  调研：看 paper，和打仗中的侦查差不多；站在领域的前沿，时效和质量
>     1.  经典著作 教材 PRML manning 的 intro， famous scholars 的 thesis
>     2.  Journal 、 Conference paper
>     3.  Social media, twitter
>     4.  Arxiv
> 2.  思索
> 3.  判断
>
> 怎么读论文：
>
> - 80% 只读标题
> - 14%的文章 只读标题和摘要
> - 5% 读标题 摘要 正文
> - 1% 搞懂全部细节

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">随笔</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2020/05/03/thoughts-under-covoid19/)