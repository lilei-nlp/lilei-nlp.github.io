---
title: "我们与 GPT-4V 的距离"
date: 2024-03-09
categories:
  - blog
tags: []
---

在 ChatGPT 引爆 AI 圈之后，很多人预言 2024 年将会是多模态的元年。的确，我们在 23 年的最后一季度见证了 GPT-4V 的发布，前不久Google 家的 Gemini 和 Anthropic 的 Claude 3 也同样支持多模态 （Multimodal to Text），并且 Gemini 1.5 中能够从两小时的视频中准确“捞针”出其中一帧包含的画面。国内这方面的工作以 <a href="https://github.com/QwenLM/Qwen-VL" target="_blank" rel="noopener">Qwen-VL</a> 为代表，也同样取得了非常不错的效果。我们最近也在大视觉语言模型（LMM）做了一些尝试，发布了 <a href="https://reka.ai/reka-flash-an-efficient-and-capable-multimodal-language-model/" target="_blank" rel="noopener">Reka Flash</a>，能够接受图片、音频和视频的输入，在 MMMU 上也靠着相对较小的基础语言模型（21B）也排名能够排名靠前（截至 2024 年 3 月 9 日，这各领域变化太快了谁知道明天会是什么样呢哈哈），且 vibe test 下来感觉还行）。

![MMMU Results](/img/mmmu-benchmark-2024-03-09.png)

但是我们真的距离 GPT-4V 很近了吗？ <a href="https://arxiv.org/abs/2309.17421" target="_blank" rel="noopener">The Dawn of LMMs</a> 展现了很多目前无法被 benchmark 分数所涵盖的能力，似乎还在提醒着我们，前面的路还很长。这篇 blog，我将尝试结合自己的经历和公开的资料，分享一下对未来视觉语言模型发展的一些想法。

## <a href="#Why-LMMs" class="headerlink" title="Why LMMs?"></a>Why LMMs?

为什么大家都会预测视觉语言模型会在 2024 年爆发？我觉得原因主要有两点：

- 视觉的基础模型众多 + 数据充足：CV 的自监督学习随着 BERT 开始就已经有一系列工作，CLIP、MAE 、DINO 等能够很好地编码图片，很好地起到了 visual tokenizer 的效果。此外，应对上下文的限制，QFormer、Perceiever 也已经被广泛地验证了其有效性。除了纯文本以外，图文对也是少数我们能够轻易获取到的大量的数据 （e.g，Laion5B）， image captioning 本质也是一种 next token prediction。
- 应用场景广泛：这个也很直接，日常生活中大多数数据的呈现方式就是，图片 + 文本 -&gt; 文本的范式能够极大扩充模型处理任务的范围。另外，随着大语言模型发展催生出的一系列 Agent 研究，在浏览网页的时候会依赖 html 作为输入。如果能够直接让 Agent 看到屏幕，输出对应的操作坐标，更加简洁优雅。进一步地，Deepmind 的 <a href="https://arxiv.org/abs/2307.15818" target="_blank" rel="noopener">RT 2</a> 也验证了视觉语言模型能够很快地迁移到诸如 robotic 场景，在 embodied 环境中发挥重要的作用。

这两个条件可谓是大视觉语言模型发展的天时和地利，我们也同样可以用这一条路径来进一步验**压缩即智能这一想法，看看这一框架是否能够在具备了更丰富模态信息后，背后世界模型的学习速率是否会进一步加快**。关于这一点，之前我们的一个工作 <a href="https://aclanthology.org/2023.emnlp-main.726/" target="_blank" rel="noopener">VEC</a> 就发现即使基于纯文本 NTP 训练的 LLMs 也能够学习到视觉世界的一些基础概念，但更 embodied 的一些知识则很难（或者以相当低的速率）被学习到，需要借助视觉语言模型来辅助学习。

## <a href="#模型架构" class="headerlink" title="模型架构"></a>模型架构

目前主流的 LMM 架构基本上是以大语言模型 LLM 为核心骨架，然后将图片视觉信息整合到 LLM 的预测过程中，因而这个框架里一般有以下几个组件：

- 基座语言模型：负责处理多模态的 embedding，并且执行预测推理的功能。一般选择能够获取到的最强、大小最合适的语言模型即可；

- 视觉编码器：负责将图片信息编码成一组向量，常用的选择是 CLIP-style 的各个模型（e.g., CLIP-ViT-L/14），最新也有<a href="https://arxiv.org/abs/2402.07865" target="_blank" rel="noopener">工作</a>表明，CLIP 得到的图片表示缺少细粒度的信息，可以通过和另外的视觉编码器结合来提升在 grounding 等任务上的性能。

- 模态桥接（Modality Bridge）：负责将视觉编码器得到的图片表示进行变换映射到新的空间方便 LLM 进行处理。这里的做法有一些不同的方案：

  - Image as Word Embedding：一种经典的尝试是将视觉编码器得到的图片向量通过简单的 MLP 映射到对应的 word embedding 维度，随后就可以将图片作为多个 word embeddings 进行统一的处理。这一方面的局限是**视觉编码器的分辨率往往是固定且比较小的(224 和 336)**。而在很多场景下这样的分辨率完全不够用（OCR 识别、Agent浏览等），可以通过 post-training 来提升图片的分辨率也可以 bypass 掉 image encoder（没有了预训练分辨率的限制），直接将图片切成小块，随后映射到 word embedding 空间，Fuyu-8B 就是这样一个思路，在高分辨率的场景下展现出了非常出色的性能。分辨率提升带来的图片向量数量平方级增长带来的计算开销，可以通过利用 QFormer 或者是 Perceiver 来映射到固定数量来解决。
  - Cross Attention to Visual Embedding: Deepmind 最早搞的 <a href="https://arxiv.org/abs/2204.14198" target="_blank" rel="noopener">Flamingo</a> 就是通过在 LLM 中引入额外的 Gated Cross-Attention Layer，来在文本生成的过程中整合视觉端的信息：

  ![Flamingo](/img/flamingo-arch.png)

​这种方案对区分不同模态有着更加强的先验，但后续看到的一些开源实现和改进，都很难超越前一种方案。**如果训练量足够大，那么在前一种方案中 LLM 也能够自适应地学习到这种先验**，因而我个人觉得这个方案或许在 2 年前是有道理，但在今天 scaling law 的暴力美学下，可能更少先验，更多数据会是朴实且有效的方案。

**GPT-4V 是什么架构？** 虽然 tech report 里啥也没说，但是我们从 <a href="https://openai.com/pricing" target="_blank" rel="noopener">GPT-4V</a> 的收费计算的方式以及 <a href="https://platform.openai.com/docs/guides/vision/calculating-costs" target="_blank" rel="noopener">API Doc</a>，可能可以猜测一下背后视觉模块的方案。收费模式分两种：

- Low Resolution：这种模式下图片会被算作 85 input token 进行收费；
- High Resolution: 图片首先在保持长宽比会被缩放 2K x 2K 的方块内（花费 85 个 token）然后图片的短边将会被缩放到 768px，并且计算缩放后的图片需要多少个 512 x 512 的 grid 来覆盖。

官方给出的示例:

<figure class="highlight plain">
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
4</code></pre></td>
<td class="code"><pre><code>A 2048 x 4096 image in detail: high mode costs 1105 tokens
1. We scale down the image to 1024 x 2048 to fit within the 2048 square.
2. The shortest side is 1024, so we further scale down to 768 x 1536.
3. 6 512px tiles are needed, so the final token cost is 170 * 6 + 85 = 1105.</code></pre></td>
</tr>
</tbody>
</table>
</figure>

由此我们可以看到，一个 512 x 512 的 image tile 被 170 个 token 所表示。假设背后也是 VIT，那我们可以推测：

- 如果使用 QFormer 对输出的 visual token 做降采样，那原先的 visual tokens 在 300 - 1000 左右（参考 Qwen-VL 的 report，1024 个 patch 被压缩到 256 个的效果相对最好），则意味着 VIT 的 patch size 最大可能是 28，最小可能是 16 的样子；
- 如果没有使用 QFormer 进行压缩，那么以为着 512 x 512 的图片可能被用了一个 40 的 patch size $\sqrt{170} = 13 \approx 512 / 40$ 。如果是用了这样的 patch size，那么我们可以进一步推测 low resolution 原始的图片可能会被统一放缩到 384 x 384 ，因此我们可以用差不多 85 个 token 来表示每个图片。

最近开源的 <a href="https://llava-vl.github.io/blog/2024-01-30-llava-next/" target="_blank" rel="noopener">LLaVA-Next</a> 也采用了类似的方案，并且在一种 benchmark 上都取得了出色的性能，侧面验证了这种方法的有效性。还有一种是 adaptive 的搜索式的方案 <a href="https://vstar-seal.github.io/" target="_blank" rel="noopener">V*</a>，根据需要来切分图片里的小块重新交给模型，类似起到 re-attention 的效果，在小物体的检测问题上面有很大的潜力。总的来说，这些方案都是为了解决**输入图片分辨率不够的问题**。

## <a href="#数据" class="headerlink" title="数据"></a>数据

数据一直是这波大语言模型发展的重中之重，从训练和测评的角度，目前我个人的感受是：

- LMM 依旧能够通过构建高质量的训练数据获取性能跃迁的阶段；
- LMM 测评基准有了不少进展，但是依旧无法比较全面的 cover 多模态的能力。多模态下的 language modeling loss 也许依旧是一个金指标。

### <a href="#训练数据" class="headerlink" title="训练数据"></a>训练数据

我们大致的可以将训练分成两个阶段：Modality Alignment Training 和 Supervised fine tuning（SFT），前者为了图片映射到 LLM 的语义空间，而后者则是激发模型的能力来做各种下游任务。

**Alignment Dataset**：这块早先大家会用开源的 Laion400M 和 Laion5B 进行对齐训练，但实际情况可能是这些数据集中的 image-text pair 都过于 noisy，对于学习模态的 alignment 效率并不高。一种解决思路是对 **alignment数据集进行更加细粒度的表述**，进而能够帮助模型更好地学习图片中物体的相关位置等关系，和LLM原先的知识挂上钩。<a href="https://sharegpt4v.github.io/" target="_blank" rel="noopener">ShareGPT4V</a> 就是一个很好的尝试，验证了利用 GPT-4V 重新标注 image captions，就能够带来明显的提升。除了 ShareGPT4V 以外，<a href="https://arxiv.org/abs/2310.20550" target="_blank" rel="noopener">CapsFusion</a> 也展现了用更丰富的 caption 带来的提升。

**SFT Dataset**：

学术界开源的比较好的训练数据目前主要是 <a href="https://github.com/haotian-liu/LLaVA" target="_blank" rel="noopener">LLaVA</a> 系列，其**利用 bounding box 等辅助信息将图片文本化后，利用 ChatGPT/GPT-4 来生成了大量的 pseudo multimodal pair (detailed captioning, reasoning and conversation)**。这个范式非常有效，也是为什么 LLaVA 系列一出来效果很惊艳的原因。但他依旧存在着一些问题：

- 既然是 pseudo multimodal，那必然会引发 hallucination 问题（因为 ChatGPT 并没有真正的 see the image）。这一点也是目前大家关注的重点。解决的方案有 <a href="https://llava-rlhf.github.io/" target="_blank" rel="noopener">LLaVA-RLHF</a> ，通过额外引入一个 Factual reward model 来提升 hallucination； <a href="https://arxiv.org/abs/2311.07362" target="_blank" rel="noopener">Volcano</a> 则是用 self-feedback 来 revise 输出。或者更直接一点，我们用早先人工标注的数据做一下统一格式，在保真度方面就会有很大的提升，这方面我们做了 <a href="https://m3-it.github.io/" target="_blank" rel="noopener">M3IT</a> 来方便大家重新利用之前的数据集来做 SFT 。
- 任务的覆盖面不够广，在重要的 OCR、Chart 场景下能力都有所欠缺。这点我们对比 Qwen、LLaVA 1.5 以及 LLaVA-Next 的性能就能看出来，使用了更多更丰富的多模态数据集，基本上都能对下游如 MMMU、MathVista 等测评数据集有所提升。

通过这些研究我们可以猜测，GPT-4V 背后一定是大量的数据工程，具体地可能体现在：

- Alignment 端：相比于开源模型利用 CLIP 等作为 vision encoder，OpenAI 应该采用了强化版的 CLIP 模型（毕竟现在的 CLIP 还都是他们 2021 年的成果）。之前的 CLIP 不够好的很大原因就在于**图片和文本的信息量不对等**，caption 大多是简单的几个词来描述物体，而图片中则有丰富的颜色、位置等时空信息。不妨可以想象一下，我们用现在的 GPT-4V 标注整个 web images（~ 10B level ?），提升文本端的丰富度并对 hallucination 做控制。在此数据集基础上我们训练一个 vision encoder，再迭代地更新 GPT-4V，相信会有一个明显的提升；
- SFT 端：我认为在足够好的对齐 + 在基模型足够强大这两个条件下，可能只需要足够多样的（质量 &gt;&gt; 数量）的 prompting 数据就能够在现在的 VQA、Captioning benchmark 上表现出色。因为客观来说，现在的测评数据集也都集中在这两个任务形式上，因此少量的 prompt 就能够泛化到下游的数据集上。

### <a href="#测评基准" class="headerlink" title="测评基准"></a>测评基准

目前关注 LMM 测评的工作有很多，大致归类一下：

- 综合性 Benchmark：融合了各种多模态任务，综合地评估 LMM 各个方面的能力，主要形式是 VQA，给定问题和图片让模型回答 Yes/No 或者是给出选项，代表的工作有 [MME]() 还有 <a href="https://github.com/yuweihao/MM-Vet" target="_blank" rel="noopener">MM-Vet</a>。这里有一些有意思的事情是 MME 采用 Yes/No parsing 来评估，而 MM-Vet 则会采用 ChatGPT 打分的方式评估，前者其实对 GPT-4V 喜欢给出带理由的回答的模型并不友好，模型可能回答正确但没有被正确解析；而后者则容易倾向于 prefer ChatGPT style 的模型，偏好使用了接近数据的模型。

- 特定领域的 Benchmark：hallucination 是多模态更容易体现出来的一个问题，造成的潜在后果也挺大，这方面测评的benchmark 像 <a href="https://arxiv.org/abs/2305.10355" target="_blank" rel="noopener">POPE</a> 和 <a href="https://llava-rlhf.github.io/" target="_blank" rel="noopener">MMHal</a>。但是 POPE 有个问题这个数据集依赖于 COCO 的 tag，就我个人的经验而言，那个 tag 的准确率并不高，POPE 上的分数因而会收到一定程度的影响。此外，大家认为 math reasoning 可能是比较有挑战性的任务，因此像 MMMU 和 MathVista 的关注度都比较高，目前 GPT-4V 也距离人类还是有很大差距。这块我们最近的一个工作是意识到 ArXiv 上的很多 paper 天然也是多模态的，并且涵盖了丰富的学科内容，因而我们构建了一个 <a href="https://mm-arxiv.github.io/" target="_blank" rel="noopener">Multimodal ArXiv</a>，提供 captioning 和 QA (GPT-4V generated）的数据集，能够很有效地提升模型数学推理的能力。

这些基准上的分数依旧很难比较全面的反应模型的能力，**模型会做题不代表这个模型可用性高**。能够给用户体验让用户有 wow 感觉的模型，才可能说真的是看到了 GPT-4V 的尾巴，而目前能做到这点的模型，还不多。

## <a href="#Future-Directions" class="headerlink" title="Future Directions"></a>Future Directions

总体来看，我认为我们和 GPT4-V 的差距在于(i) 基模型的指令跟随和理解能力；(ii) 模态对齐的训练质量和数量，以及 (iii)多样的 SFT 数据的构建。

其中 (i) 是国内很多公司和研究组努力的方向，相信在大伙的努力下我们会有一个强大的基模型，现在有的 Qwen 、Deepseek、Skywork 等系列模型都很有机会。(ii) 目前开源出来数据集的量级还不够大，而这件事情的投入（re-annotating the image world）应该也是巨大。但值得注意的是，DALLE 3 和 Sora 也是用了类似的方案来对 image/video的描述进行细节化，因而进一步提升了生成图片和视频的质量。做这件事情的意义可能对于我们去建模一个高分辨率的世界模型是有重大意义的。(iii) 这件事情或许可以交给学术界来搞，定义和标注有意义的多模态任务，进而整合到 SFT 过程中即可。

除去这些看似比较 boring 的搞数据以外，还有什么值得探索的方向呢，这边我也分析一些我个人比较感兴趣的问题（<s>带货环节</s>）：

- LMM Hallucination 形成的原因？在文本领域的 Hallucination 的原因大家也都还在广泛地讨论中，那引入一个额外模态之后，hallucination 的来源会更多了吗？是数据还是模型架构带来的问题？如果我们能够更清晰的看到模型内部的一些信号，或许会对理解这些问题更有帮助。
- LMM 的安全性：ChatGPT 出来之后就有很多 Red Teaming 和 Jailbreaking的尝试，那 GPT-4V 会不会也有这种安全性上的 concern 呢？<a href="https://arxiv.org/abs/2401.12915" target="_blank" rel="noopener">Red Teaming VLM</a> 提供了一个很好的 benchmark 来做这方面的探索；此外，<a href="https://github.com/xijia-tao/ImgTrojan" target="_blank" rel="noopener">ImgTrojan</a> 也发现之前 NLP 广泛存在的后门攻击同样适用于 LMM，并且会成为更为隐蔽的特洛伊木马来规避掉 safe alignment。这里后续的研究又可以进行攻击、防御、消除的探索。
- RLHF/DPO for LMM：前面提到的 alignment 和 sft 更多地还是依赖于人类标注的数据，当人类无法给出 ground-truth 的标注的时候，我们就需要构建一个 reward model 来告诉我们哪些回复是更合适的。RLHF 和 DPO 已经在大语言模型上被验证了有效性，但当存在额外的模态的时候，如何定义哪个回复是更好的（例如会有更多样的偏见），如何更好地协调一致的 reward label 的标注等等都会带来新的问题和挑战。我们的 <a href="https://vlf-silkie.github.io/" target="_blank" rel="noopener">VLFeedback</a> 提供了一个很直给的方案，让 GPT-4V 来标注不同的方面，并且也验证了这个框架的有效性。但最近我们也发现 DPO 在不同基模型上的效果还不太一样，依旧存在很多细节的问题，值得进一步的分析。

总的来说，LMM 在无论是学术界还是工业界，都依旧大有可为。

希望能和这一领域的研究者们一起，接近 GPT-4V，超越 OpenAI！
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">GPT4V</a>
