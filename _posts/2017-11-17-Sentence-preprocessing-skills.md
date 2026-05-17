---
title: "文本预处理方法小记"
date: 2017-11-17
categories:
  - blog
tags: []
---

最近做 Sentiment Analysis 的问题，用 IMDB，Twitter 等 Dataset，拿到原始的一条条文本，直接喂给 Model 肯定不行，需要进行对文本进行预处理。预处理的精细程度很大程度上也会影响模型的性能。这篇 Blog 就记录一些预处理的方法。

## <a href="#Remove-Stop-Words" class="headerlink" title="Remove Stop Words"></a>Remove Stop Words

​Stop Words，也叫停用词，通常意义上，停用词大致分为两类。一类是人类语言中包含的功能词，这些功能词极其普遍，与其他词相比，功能词没有什么实际含义，比如’the’、’is’、’at’、’which’、’on’等。另一类词包括词汇词，比如’want’等，这些词应用十分广泛，但是对这样的词搜索引擎无法保证能够给出真正相关的搜索结果，难以帮助缩小搜索范围，同时还会降低搜索的效率，所以通常会把这些词从问题中移去，从而提高搜索性能。

<span id="more"></span>

​在对 Twitter 做 Sentiment Analysis 时，因为Twitter 的文本大部分是人们的口头语，所以停用词的使用是非常频繁的，而这些停用词对于情感极性的贡献并不大，所以一般在预处理阶段我们会将它们从文本中去除，以更好地捕获文本的特征和节省空间（Word Embedding）。Remove Stop Words 的方法有很多，Stanford NLP 组有一个工具就能够办到，Python 中也有 nltk 库来做一些常见的预处理，这里就以 nltk 为例来记录去除停用词的操作：

首先我们导入 `nltk.corpus` 中的 `stopwords` 对象，选取 `english` 的 stopwords，生成一个 set

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
3</code></pre></td>
<td class="code"><pre><code>from nltk.corpus import stopwords
stop = set(stopwords.words(&#39;english&#39;)) # 
print(stop)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

> ‘below’, ‘she’, ‘both’, ‘didn’, ‘his’, ‘we’, ‘they’, ‘from’, ‘themselves’, ‘more’, ‘shan’, ‘which’, ‘whom’, ‘further’, ‘needn’, ‘while’, ‘at’, …

以上是 stop 中的部分 stop words，确实没有什么意义，接下来定义一个函数，将原始的数据集文本中的停用词去除：

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
<td class="code"><pre><code>def remove_stop(data):
   total_words = 0 # 用于计算平均长度 以方便后面截断长度的设定
    after_remove = list()
    length = len(data) # 获取数据集的大小 一般是一个 ndarray
    total_words = 0
    for i in range(length): # 依次处理文本
        clean_sentence = list()
        for word in data[i].split(): # 将文本 spilit 成一个个单词
            if word not in stop:
                clean_sentence.append(word)
        total_words += len(clean_sentence)
        afters.append(&quot; &quot;.join(after))

    print(&quot;Average length:&quot;, total_words/length)
    return np.asarray(after_remove)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

思路很简单，就是判断 word 是否在 stop words 的集合中，如果不在就保留下来，最后通过 `" ".join(list)` 将非停用词的列表生成一个字符串，这个 `.join` 非常有意思；同样，为了统计去掉停用词之后的平均句子长度，在代码中我们每次都计算一下每个句子的长度，最后求一个平均，这是为了一些需要进行截断操作的处理做准备。

## <a href="#To-Word-Index" class="headerlink" title="To Word Index"></a>To Word Index

文本是无法直接交给我们模型进行训练的，我们需要把它们变成数字，在 NLP 领域很常用的一种方法就是 Sentence -&gt; Word ID -&gt; Word Embedding，而Sentence -&gt; Word ID 这一步，就是把每一个词变成一个独立的整数，比如下面的例子：

“I am a student”，“You are a student, too”

两个句子中总共有 7 个不一样的单词，我们按照出现的顺序来编号：

I：1 am: 2 a: 3 student: 4 You: 5 are: 6 too: 7

那么两个句子就会对应的被转换为：

\[1 2 3 4\] 和 \[5 6 3 4 7\]

如果我们遇到了词汇表中没有的词，一般用 0 或者 UNK（unknown）来表示，所以“You are beautiful today”的表示就是 \[5 6 0 0\]

如何利用 TensorFlow 来实现呢，很简单，利用 VocabularyProcessor 这个类：

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
4</code></pre></td>
<td class="code"><pre><code>vocab_processor = tf.contrib.learn.preprocessing.VocabularyProcessor(
    MAX_LENGTH, min_frequency=2)
x_transform_train = vocab_processor.fit_transform(train_text)
x_transform_test = vocab_processor.transform(test_text)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

其主要有两个函数：`fit()` 和 `transform()`，fit 就是形成一个 word: id 的字典，transform 就是根据字典来把句子转换成 id 组成的向量，一般我们通过 fit 训练集，再根据由训练集得到的 vocab dict 来 transform 测试集。你可能会说这样岂不是会有很多 UNK？是的，这也是真实的情况，每时每刻都有新词被造出来，而我们的训练集是不可能 hold 住所有的词语的。

值得一提的是，这里 `VocabularyProcessor` 的构造函数中还有一个 `min_frequency` 参数，可以筛掉出现次数少于这个参数的词，去低频次，也是一种预处理的手段。

**Update**: 随着 TensorFlow 版本的更新，原来 `contrib` 包里的 `VocabularyProcessor` 已经不被推荐使用了，改为使用 `tf.keras.preprocessing` 包中的相关 API 进行操作，其中最重要的一个类就是 `Tokenizer` 这个类：

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
<td class="code"><pre><code># Tokenizer
tokenizer = tf.keras.preprocessing.text.Tokenizer(num_words=20000， oov_token=&#39;&lt;UNK&gt;&#39;)
tokenizer.fit_on_texts(train_text)
train_idxs = tokenizer.text_to_sequences(train_text)
test_idxs = tokenizer.text_to_sequences(test_text)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

`Tokenizer()` 构造函数有几个比较有用的参数：

- num\_words: 保留的 vocab\_size 的个数，即词频最高的 `num_words` 个 word 在词表中，这和之前的 `min_frequency` 的设计是恰好相反的，一个是考虑留下多少，一个是考虑去掉多少出现频率较低的。
- oov\_token: 超出词表（test 中有 train 中未出现的词）时，将其设置为指定的 token，**这个 `<UNK>` 在不会出现在 `word_docs` 和 `word_counts` 中，但是会出现 `word_index` 中，所以在计算 vocab\_size 时使用 `len(word_index)` 比较稳妥**

流程也是一样的，先利用 fit\_on\_texts 进行词表的构建，再利用 `text_to_sequences()` 来将 word 转化为对应的 idx；Tokenizer 有三个非常有用的成员：

- word\_docs：一个 `OrderedDict`，用于记录各个词出现的次数
- word\_count：一个 `dict`，用于记录各个词出现的次数
- word\_index：word2idx 的一个字典，我们可以根据 word 拿到对应的 index，也可以通过简单的一行代码来构建一个 idx2word 的字典用于之后将 indexes 翻译成文本 `idx2word = {idx: word for word, idx in zip(word2idx.keys(), word2idx.values())}`

转换成 index 序列之后，长度的截取和 padding 则需要利用 `tf.keras.preprocessing.sequence` 包中的 `pad_sequences` 函数来进行：

<figure class="highlight python">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2</code></pre></td>
<td class="code"><pre><code>train_padded = tf.keras.preprocessing.sequence.pad_sequences(train_idxs, 
                                    maxlen=MAX_LENGTH, padding=&#39;post&#39;, truncating=&#39;post&#39;)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

`MAX_LENGTH` 和之前一样，用于指定需要截取到或者补齐到的程度，而之后的 `padding` 和 `truncating` 关键字则用于指定补齐和截取是哪个方向（`post` 或者 `pre`），比如对于一个序列 `[1, 2, 3, 4, 5]` 如果我们用 `maxlen = 6` ，而 padding 选择 `pre` 得到的结果就是：

> \[0, 1, 2, 3, 4, 5\]

而如果 padding 选择 `post` ，结果就是：

> \[1, 2, 3, 4, 5, 0\]

`truncating` 同理。

## <a href="#Load-Pre-trained-Word-Embedding" class="headerlink" title="Load Pre-trained Word Embedding"></a>Load Pre-trained Word Embedding

我们可以使用预训练好的 Word Embedding 来加快模型的训练速度，常用的有 Stanford 的 <a href="https://nlp.stanford.edu/projects/glove/" target="_blank" rel="noopener">GloVe</a> 和 Google 家的 <a href="https://code.google.com/archive/p/word2vec" target="_blank" rel="noopener">Word2Vec</a> ，这里就以 GloVe 为例，展示如何加载预训练的 Word Embedding。

GloVe 解压后会得到几个 `.txt` 文件，分别是 50/100/200/300 维的 Word Embedding，其格式是 “词语 向量” ，如下所示

> the -0.038194 -0.24487 0.72812 -0.39961 0.083172 0.043953 -0.39141 0.3344 -0.57545 0.087459 0.28787 -0.06731 0.30906 -0.26384 -0.13231 -0.20757 0.33395 -0.33848 -0.31743 -0.48336 0.1464 -0.37304 0.34577 0.052041 0.44946 -0.46971 0.02628 -0.54155 -0.15518 -0.14107 -0.039722 0.28277 0.14393 0.23464 -0.31021 0.086173 0.20397 0.52624 0.17164 -0.082378 -0.71787 -0.41531 0.20335 -0.12763 0.41367 0.55187 0.57908 -0.33477 -0.36559 -0.54857 -0.062892 0.26584 0.30205 0.99775 -0.80481 -3.0243 0.01254 -0.36942 2.2167 0.72201 -0.24978 0.92136 0.034514 0.46745 1.1079 -0.19358 -0.074575 0.23353 -0.052062 -0.22044 0.057162 -0.15806 -0.30798 -0.41625 0.37972 0.15006 -0.53212 -0.2055 -1.2526 0.071624 0.70565 0.49744 -0.42063 0.26148 -1.538 -0.30223 -0.073438 -0.28312 0.37104 -0.25217 0.016215 -0.017099 -0.38984 0.87424 -0.72569 -0.51058 -0.52028 -0.1459 0.8278 0.27062

那么我们先需要构建一个 vocab 词汇表表存放所有的单词，以及其对应的 Word Embedding：

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
<td class="code"><pre><code>import numpy as np
filename = &#39;glove.6B.50d.txt&#39;

def loadGloVe(filename):
    vocab = []
    embd = []
    file = open(filename, &#39;r&#39;)
    for line in file.readlines(): # 读取 txt 的每一行
        row = line.strip().split(&#39; &#39;)
        vocab.append(row[0])
        embd.append(row[1:])
    print(&#39;Loaded GloVe!&#39;)
    file.close()
    return vocab, embd

vocab, embd = loadGloVe(filename)
vocab_size = len(vocab) # 词表的大小
embedding_dim = len(embd[0]) # embedding 的维度
print(&quot;Vocab size : &quot;, vocab_size)
print(&quot;Embedding dimensions : &quot;, embedding_dim)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

获取了 vocab 和 embedding 之后，我们将就可以利用 VocabularyProcessor 来根据 vocab 将文本转化为对应的 Id，并且进行 Embedding 操作：

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
<td class="code"><pre><code>
print(embedding.shape)

vocab_processor = tf.contrib.learn.preprocessing.VocabularyProcessor(MAX_LENGTH)
pretrain = vocab_processor.fit(vocab) # 根据我们的 vocab 进行 fit
x_transform_train = vocab_processor.transform(x_train) # train set
x_transform_test = vocab_processor.transform(x_test) # test set

vocab = vocab_processor.vocabulary_
vocab_size_after_process = len(vocab) # 注意：这个 size 和前面的不一样了
print(&quot;Vocab size after process:&quot;, vocab_size_after_process)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

值得注意的是，**经过 fit 后的 vocab\_size 会变小**，因为预训练的 Word Embedding 包括 `,` `!` 类似的符号，而在 VocabularyProcessor 的处理代码中，会忽略所有非单词的符号，这就是为什么 vocab\_size 变小的原因。

接下来就是通过 Tensorflow 的 `embedding_look_up` 进行 embedding 操作了：

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
<td class="code"><pre><code>embeddings_var = tf.Variable(tf.constant(0.0, shape=[vocab_size_after_process, embedding_dim]), trainable=False)
# 创建 Embedding 变量 
embedding_placeholder = tf.placeholder(tf.float32, [vocab_size_after_process, embedding_dim]) # 通过 Placeholder 喂给 graph
embedding_init = embeddings_var.assign(embedding_placeholder) # 初始化赋值

embedding = np.asarray(embd) # 将 list 变成 ndarray 便于喂给 graph
with tf.Session() as sess:
  sess.run(embedding_init, feed_dict={embedding_placehoder:embedding})
  # ... 其他代码</code></pre></td>
</tr>
</tbody>
</table>
</figure>

**注意：`embedding_var` trainable 属性设置为`False`（已经预训练好了）； 以及这里的 vocab\_size 建议采用 after process 的较小的值，避免输入中没有符号而 Embedding Weight 中有符号，造成的计算、空间资源的浪费。**

## <a href="#Shuffle" class="headerlink" title="Shuffle"></a>Shuffle

打乱训练集也是我们经常需要做的，避免同种 label 的数据大量的出现，我们处理的数据常常是 ndarray 或者是 pandas 的 Series，这里就介绍两个 shuffle 的函数：

首先是 numpy 的，通过 np.random.shuffle 打乱数组：

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
3</code></pre></td>
<td class="code"><pre><code>import numpy as np
np.random.shuffle(train_set)
np.random.shuffle(test_set)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

再就是 Pandas Series 的：

<figure class="highlight python">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2</code></pre></td>
<td class="code"><pre><code>train = train.sample(frac=0.3)
test = test.sample(frac=1.0)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

`Series.sample(frac)` 可以起到 Shuffle 的作用，并且还能够通过 frac 参数来指定采样的比例，如果我们希望只用 0.3 的数据，就可以指定 `frac = 0.3`，来实现这一目的，这使用起来非常方便。
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">TensorFlow</a>
