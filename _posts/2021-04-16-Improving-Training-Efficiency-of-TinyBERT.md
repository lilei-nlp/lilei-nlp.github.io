---
title: "TinyBERT 蒸馏速度实现加速小记"
date: 2021-04-16
categories:
  - blog
tags: []
---

最近做的一个 project 需要复现 EMNLP 2020 Findings 的 <a href="https://github.com/huawei-noah/Pretrained-Language-Model/tree/master/TinyBERT" target="_blank" rel="noopener">TinyBERT</a>，这篇文章就是在复现过程对踩到坑，以及对应的解决方案和实现加速的一个记录。

## <a href="#Overview-of-TinyBERT" class="headerlink" title="Overview of TinyBERT"></a>Overview of TinyBERT

BERT 效果虽好，其较大内存消耗和较长的推理延时会对其上线部署造成一定挑战。内存消耗方面，一系列知识蒸馏的工作，例如 <a href="https://arxiv.org/abs/1910.01108" target="_blank" rel="noopener">DistilBERT</a>、<a href="https://arxiv.org/abs/1908.09355" target="_blank" rel="noopener">BERT-PKD</a> 和 <a href="https://github.com/huawei-noah/Pretrained-Language-Model/tree/master/TinyBERT" target="_blank" rel="noopener">TinyBERT</a> 被提出来来降低模型的参数（主要是层数）以及相应地减少时间；推理加速方面，也有例如 <a href="https://arxiv.org/abs/2004.12993" target="_blank" rel="noopener">DeeBERT</a>、<a href="https://arxiv.org/abs/2004.02178" target="_blank" rel="noopener">FastBERT</a> 以及 <a href="https://arxiv.org/abs/2012.14682" target="_blank" rel="noopener">CascadeBERT</a> 等方案来动态地根据样本难度进行模型的执行从而提升推理效率。其中比较具备代表性便是 TinyBERT，其核心框架如下：

![TinyBERT](/img/tinybert.png)

分为两个阶段：

- General Distillation：在通用的语料，例如 BookCorpus， EnglishWiki 上进行知识蒸馏，目标函数包括 Transformer Layer Attention 矩阵以及 Layer Hidden States 的对齐；
- Task Distillation：在具体的任务数据集上进行蒸馏，又被进一步分成两个步骤：
  - Task Transformer Disitllation: 在任务数据集上对齐 Student 和已经 fine-tuned Teacher model 的 attention map 和 hidden states；
  - Task Prediction Distillation：在任务数据集上对 student model 和 teacher model 的 output distritbuion 利用 KL loss / MSE loss 进行对齐。

TinyBERT 提供了经过 General Distillation 阶段的 checkpoint，可以认为是一个小的 BERT，包括了 6L786H 版本以及 4L312H 版本。而我们后续的复现就是基于 4L312H v2 版本的。值得注意的是，TinyBERT 对任务数据集进行了数据增强操作，通过基于 Glove 的 Embedding Distance 的相近词替换以及 BERT MLM 预测替换，会将原本的数据集扩增到 20 倍。而我们遇到的第一个 bug 就是在数据增强阶段。

## <a href="#Bug-in-Data-Augmentation" class="headerlink" title="Bug in Data Augmentation"></a>Bug in Data Augmentation

我们可以按照官方给出的代码对数据进行增强操作，但是在 QNLI 上会报错：

> Index Error: index 514 is out of dimension 1 with size 512

造成数据增强到一半程序就崩溃了，为什么呢？

很简单，因为数据增强代码 BERT MLM 换词模块对于超长（&gt; 512）的句子没有特殊处理，造成下标越界，具体可以参考 <a href="https://github.com/huawei-noah/Pretrained-Language-Model/issues/50" target="_blank" rel="noopener">#Issue50</a>。

在对应的函数中进行边界的判断即可：

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
<td class="code"><pre><code>def _masked_language_model(self, sent, word_pieces, mask_id):

    if mask_id &gt; 511: # if mask id is longer than max length 
        return [] 
    tokenized_text = self.tokenizer.tokenize(sent)
    tokenized_text = [&#39;[CLS]&#39;] + tokenized_text
    tokenized_len = len(tokenized_text)
    tokenized_text = word_pieces + [&#39;[SEP]&#39;] + tokenized_text[1:] + [&#39;[SEP]&#39;]
    segments_ids = [0] * (tokenized_len + 1) + [1] * (len(tokenized_text) - tokenized_len - 1)
    if len(tokenized_text) &gt; 512: #  truncation 
        tokenized_text = tokenized_text[:512]
        segments_ids = segments_ids[:512]  
    token_ids = self.tokenizer.convert_tokens_to_ids(tokenized_text)
    tokens_tensor = torch.tensor([token_ids]).to(device)
    segments_tensor = torch.tensor([segments_ids]).to(device)
    self.model.to(device)
    predictions = self.model(tokens_tensor, segments_tensor)
    word_candidates = torch.argsort(predictions[0, mask_id], descending=True)[:self.M].tolist()
    word_candidates = self.tokenizer.convert_ids_to_tokens(word_candidates)

    return list(filter(lambda x: x.find(&quot;##&quot;), word_candidates))</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#Acceleration-of-Data-Parallel" class="headerlink" title="Acceleration of Data Parallel"></a>Acceleration of Data Parallel

当我们<s>费劲</s>愉快地完成数据增强之后，下一步就是要进行 Task Specific 蒸馏里的 Step 1，General Distillation 了。对于一些小数据集像 MRPC，增广 20 倍之后的数据量依旧是 80k 不到，因此训练速度还是很快的，20 轮单卡大概半天也能跑完。但是对于像 MNLI 这样 GLUE 中最大的数据集（390k），20 倍增广后的数据集（增广就花费了大约 2 天时间），如果用单卡训练个 10 轮那可能得跑上半个月了，到时候怕不是黄花菜都凉咯。遂打算用多卡训练，一看，官方的实现就通过 `nn.DataParallel` 支持了多卡。好嘛，直接 `CUDA_VISIBLE_DEVICES="0,1,2,3"` 来上 4 块卡。不跑不知道，加载数据（tokenize, padding ）花费 1小时，好不容易跑起来了，一开 `nvidia-smi` 吓一跳，GPU 的利用率都在 50% 左右，再一看预估时间，大约 21h 一轮，10 epoch 那四舍五入就是一个半礼拜。好家伙，这我还做不做实验了？这时候就去翻看 PyTorch 文档，发现 PyTorch 现在都不再推荐使用 `nn.DataParallel` 了，为什么呢？主要原因在于 DataParallel 的实现是单进程的，每次都是有一块主卡读入数据再发给其他卡，这一部分不进带来了额外的计算开销，而且会造成主卡的 GPU 显存占用会显著高于其他卡，进而造成潜在的 batch size 限制；此外，这种模式下，其他 GPU 算完之后要传回主卡进行同步，这一步又会受限于 Python 的线程之间的 GIL（global interpreter lock），进一步降低了效率。此外，还有多机以及模型切片等 DataParallel 不支持，但是另一个 DistributedDataParallel 模块支持的功能。所以，废话少说，得把原先 TinyBERT DataParallel（DP）改成 DistributedDataParallel(DDP)。那么，请问，把 DP 改成 DDP 需要几步？答：大概，就那么多步吧，实现可以参考这个<a href="https://zhuanlan.zhihu.com/p/98535650" target="_blank" rel="noopener">知乎-当代研究生需要掌握的并行训练技巧</a>。核心的代码就是做一下初始化，以及用 DDP 替换掉 DP：

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
32</code></pre></td>
<td class="code"><pre><code>from torch.nn.parallel import DistributedDataParallel as DDP
import torch.distributed as dist 

# 给 parser 增加一个 local rank 参数来在启动的时候传入 rank 
parser.add_argument(&#39;--local_rank&#39;,
                        type=int,
                        default=-1)
# ...

# 初始化
logger.info(&quot;Initializing Distributed Environment&quot;)
torch.cuda.set_device(args.local_rank)
dist.init_process_group(backend=&quot;nccl&quot;)

# 设置 devicec
local_rank = args.local_rank
torch.cuda.set_device(local_rank)

# ...

# 初始化模型 并且 放到 device 上
student_model = TinyBertForSequenceClassification.from_pretrained(args.student_model, num_labels=num_labels).to(device)    
teacher_model = TinyBertForSequenceClassification.from_pretrained(args.teacher_model, num_labels=num_labels).to(device)

# 用 DDP 包裹模型
student_model = DDP(student_model, device_ids=[local_rank], output_device=local_rank)
teacher_model = DDP(teacher_model, device_ids=[local_rank], output_device=local_rank)

# ..

# 用 DistributedSampler 替换原来的 Random Sampler
train_sampler = torch.utils.data.DistributedSampler(train_data)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

然后，大功告成，一键启动:

> GPU=”0,1,2,3”
>
> CUDA\_VISIBLE\_DEVICEES=$GPU python -m torch.distributed.launch –n\_proc\_per\_node 4 task\_disti.py

启动成功了吗？模型又开始处理数据….

One hours later，机器突然卡住，程序的 log 也停了，打开 htop 一看，好家伙，256G 的内存都满了，程序都是 D 状态，咋回事？

## <a href="#Acceleration-of-Data-Loading" class="headerlink" title="Acceleration of Data Loading"></a>Acceleration of Data Loading

我先试了少量数据，降采样到 10k，程序运行没问题， DDP 速度很快；我再尝试了单卡加载，虽然又 load 了一个小时，但是 ok，程序还是能跑起来，那么，问题是如何发生的呢？单卡的时候我看了一眼加载全量数据完毕之后的内存占用，大约在 60G 左右，考虑到 DDP 是多进程的，因此，每个进程都要独立地加载数据，4 块卡 4个进程，大约就是 250 G 的内存，因此内存爆炸，到后面数据的 io 就卡住了（没法从磁盘 load 到内存），所以造成了程序 D 状态。看了下组里的机器，最大的也就是 250 G 内存，也就是说，如果我只用 3 块卡，那么是能够跑的，但是万一有别的同学上来开程序吃了一部分内存，那么就很可能爆内存，然后就是大家的程序都同归于尽的局面，不太妙。一种不太优雅的解决方案就是，把数据切块，然后读完一小块训练完，再读下一块，再训练，再读。咨询了一下组里资深的师兄，还有一种办法就是实现一种**把数据存在磁盘上，每次要用的时候才 load 到内存的**数据读取方案，这样就能够避免爆内存的问题。行吧，那就干吧，但是总不能从头造轮子吧？脸折师兄提到 huggingface(yyds) 的 <a href="https://huggingface.co/datasets" target="_blank" rel="noopener">datasets</a> 能够支持这个功能，check 了一下文档，发现他是基于 pyarrow 的实现了一个 memory map 的数据<a href="https://github.com/huggingface/datasets/blob/fcd3c3c8e3b1d9a2f3686a496082e21f06591380/src/datasets/table.py#L400" target="_blank" rel="noopener">读取</a>，以我的 huggingface transformers 的经验，似乎是能够实现这个功能的，所以摩拳擦掌，准备动手。

首先，要把增广的数据 load 进来，datasets 提供的 `load_dataset` 函数最接近的就是 `load_dataset('csv', data_file)`，然后我们就可以逐个 column 的拿到数据并且进行预处理了。写了一会，发现总是报读取一部分数据后 columns 数目不对的错误，猜测可能原始 MNLI 数据集就不太能保证每个列都是在的，检查了一下 `MnliProcessor` 里处理的代码，发现其写死了 `line[8]` 和 `line[9]` 作为 sentence\_a 和 sentence\_b。无奈之下，只能采取最粗暴地方式，用 `text` mode 读进来，每一行是一个数据，再 split：

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
56</code></pre></td>
<td class="code"><pre><code>from datasets import 

processor = processors[task_name]()
output_mode = output_modes[task_name]
label_list = processor.get_labels()
num_labels = len(label_list)

tokenizer = BertTokenizer.from_pretrained(args.student_model, do_lower_case=args.do_lower_case)
# 用 text
mnli_datasets = load_dataset(&quot;text&quot;, data_files=os.path.join(args.data_dir, &quot;train_aug.tsv&quot;))
label_classes = processor.get_labels()
label_map = {label: i for i, label in enumerate(label_classes)}
        def preprocess_func(examples, max_seq_length=args.max_seq_length):
            splits = [e.split(&#39;\t&#39;) for e in examples[&#39;text&#39;]] # split
            # tokenize for sent1 &amp; sent2
            tokens_s1 = [tokenizer.tokenize(e[8]) for e in splits] 
            tokens_s2 = [tokenizer.tokenize(e[9]) for e in splits]
            for t1, t2 in zip(tokens_s1, tokens_s2):
                truncate_seq_pair(t1, t2, max_length=max_seq_length - 3)
            input_ids_list = []
            input_mask_list = []
            segment_ids_list = []
            seq_length_list = []
            labels_list = []
            labels = [e[-1] for e in splits] # last column is label column 
            for token_a, token_b, l in zip(tokens_s1, tokens_s2, labels):  # zip(tokens_as, tokens_bs):
                tokens = [&quot;[CLS]&quot;] + token_a + [&quot;[SEP]&quot;]
                segment_ids = [0] * len(tokens)
                tokens += token_b + [&quot;[SEP]&quot;]
                segment_ids += [1] * (len(token_b) + 1)
                input_ids = tokenizer.convert_tokens_to_ids(tokens) # tokenize to id 
                input_mask = [1] * len(input_ids)
                seq_length = len(input_ids)
                padding = [0] * (max_seq_length - len(input_ids))
                input_ids += padding
                input_mask += padding
                segment_ids += padding
                assert len(input_ids) == max_seq_length
                assert len(input_mask) == max_seq_length
                assert len(segment_ids) == max_seq_length
                input_ids_list.append(input_ids)
                input_mask_list.append(input_mask)
                segment_ids_list.append(segment_ids)
                seq_length_list.append(seq_length)
                labels_list.append(label_map[l])

            results = {&quot;input_ids&quot;: input_ids_list,
                       &quot;input_mask&quot;: input_mask_list,
                       &quot;segment_ids&quot;: segment_ids_list,
                       &quot;seq_length&quot;: seq_length_list,
                       &quot;label_ids&quot;: labels_list}
            return results
# map datasets
mnli_datasets = mnli_datasets.map(preprocess_func, batched=True)
# remove column
train_data = mnli_datasets[&#39;train&#39;].remove_columns(&#39;text&#39;)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

写完这个 `preprocess_func` ，我觉得胜利在望，但还有几个小坑需要解决s：

- map 完之后，返回的还是一个 DatasetDict，得手动取一下 `train` set；

- 对于原先存在的列，map 函数并不会去除掉，所以如果不用的列，需要手动 `.remove_columns()`

- 在配合 DDP 使用的时候，因为 DistributedSample 取数据的维度是在第一维取的，所以取到的数据可能是个 seq\_len 长的列表，里面的 tensor 是 \[bsz\] 形状的，需要在交给 model 之前 stack 一下：

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
  <td class="code"><pre><code>inputs = {}
  for k, v in batch.items():
      if isinstance(v, torch.Tensor):
          inputs[k] = v.to(device)
      elif isinstance(v, List):
          inputs[k] = torch.stack(v, dim=1).to(device)</code></pre></td>
  </tr>
  </tbody>
  </table>
  </figure>

至此，只要把之前代码的 train\_data 都换成现在的版本即可。

此外，为了进一步加速，我还把混合精度也整合了进来，现在 Pytorch 以及自带对混合精度的支持，代码量也很少，但是有个坑就是**loss 的计算必须被 auto() 包裹住**，同时，所有模型的输出都要参与到 loss 的计算，这对于只做 prediction 或者是 hidden state 对齐的 loss 很不友好，所以只能手动再额外计算一项为系数为 0 的 loss 项（这样他参与到训练但是不会影响梯度）。

## <a href="#Finally" class="headerlink" title="Finally"></a>Finally

最后，改版过的代码在我的 GitHub <a href="https://github.com/TobiasLee/Pretrained-Language-Model/blob/master/TinyBERT/fast_td.py" target="_blank" rel="noopener">fork</a> 版本中，我不要脸地起名为 `fast_td` 。实际上，改版后的有点有一下几个：

- 数据加载方面，第一次加载/处理 780w 大约耗时 50m，但是不会多卡都消耗内存，实际占用不到 2G；同时，得益于 datasets 的支持，后续加载不会重复处理数据而是直接读取之前的 cache；
- 模型训练方面，得益于 DDP 和 混合精度，在 MNLI 上训增强数据 10 轮，3 块卡花费的时间大约在 20h 左右，提速了 10 倍。

这次修改代码大概花了 2 天时间来实现和 debug，不过感觉收益还是挺大的，此处需要感谢任大佬 & 脸折师兄的建议，以及 andy 提供的知乎文章，撒花~
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">PyTorch</a>
