---
title: "Text Classification 代码学习笔记"
date: 2017-08-07
categories:
  - blog
tags: []
---

找到了 TensorFlow 官方的 Text Classification 的代码来学习研究，<a href="https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/learn/text_classification.py" target="_blank" rel="noopener">GitHub地址</a>。官方有给了四个代码，用 **CNN** 和 **RNN** 分别对**字符**和**词语**做文本分类。这里的数据主要是 `dbpedia` 的数据，数据的描述是这样的:

> The DBpedia ontology classification dataset is constructed by picking 14 non-overlapping classes from DBpedia 2014. They are listed in classes.txt. From each of thse 14 ontology classes, we randomly choose 40,000 training samples and 5,000 testing samples. Therefore, the total size of the training dataset is 560,000 and testing dataset 70,000.

也就是说我们有 14 个类别的文本数据，然后我们要做的就是给每一段文本进行分类。

<span id="more"></span>

马士兵老师教我：看代码从 main 函数看起，于是乎：

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
<td class="code"><pre><code>def main(unused_argv):
    
    # Prepare training and testing data
    # ...
    
    
    # Process vocabulary
    # ...
    

    # Build model
    # ...

    # Train
    # ...
    
    # Predict.
    # ...
    
    # Socre
    # ...</code></pre></td>
</tr>
</tbody>
</table>
</figure>

把细节隐去，我们能够看到 main 函数内部的主题框架，就是一个典型的 ML 的流程:

1.  准备数据
2.  处理数据
3.  建模
4.  训练模型
5.  预测结果
6.  评价模型

## <a href="#1、数据准备" class="headerlink" title="1、数据准备"></a>1、数据准备

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
<td class="code"><pre><code>import ... # 省略导包

# 官方的下载数据，以及通过pandas读取数据的代码
dbpedia = tf.contrib.learn.datasets.load_dataset(
    &#39;dbpedia&#39;,
    test_with_fake_data=FLAGS.test_with_fake_data)

x_train = pd.Series(dbpedia.train.data[:, 1])
y_train = pd.Series(dbpedia.train.target)
x_test = pd.Series(dbpedia.test.data[:, 1])
y_test = pd.Series(dbpedia.test.target)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

但是可能因为墙的缘故，我实际运行的时候等了两个小时才下好这个数据。不能每次都让他去在下载一遍吧，不然我就疯了。于是掏出《利用 Python 进行数据分析》这本书，学了一下 `pandas` 的基本用法，改成从本地读取 csv 文件：

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
<td class="code"><pre><code>names = [&quot;class&quot;, &quot;title&quot;, &quot;content&quot;] # 描述中 数据有三项 类别 标题 和内容

# 读取 csv 文件 并且利用我们制定的 names 来分组
train_csv = pd.read_csv(&quot;./dbpedia_data/dbpedia_csv/train.csv&quot;, names=names)
test_csv = pd.read_csv(&quot;./dbpedia_data/dbpedia_csv/test.csv&quot;, names=names)

# 通过指定的 name 拿到我们要的 x:content 和 y:class 
x_train = pd.Series(train_csv[&quot;content&quot;])
y_train = pd.Series(train_csv[&quot;class&quot;])
x_test = pd.Series(test_csv[&quot;content&quot;])
y_test = pd.Series(test_csv[&quot;class&quot;])</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#2、数据处理" class="headerlink" title="2、数据处理"></a>2、数据处理

这一步其实应该是相当繁琐并且麻烦的，因为我们要把文本变成机器能够读懂的东西。

而官方的代码只有寥寥几行：

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
12</code></pre></td>
<td class="code"><pre><code>vocab_processor = tf.contrib.learn.preprocessing.VocabularyProcessor(
    MAX_DOCUMENT_LENGTH)

x_transform_train = vocab_processor.fit_transform(x_train)
x_transform_test = vocab_processor.transform(x_test)

x_train = np.array(list(x_transform_train))
x_test = np.array(list(x_transform_test))

global n_words
n_words = len(vocab_processor.vocabulary_)
print(&#39;Total words: %d&#39; % n_words)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

代码少是因为调用了已经写好的代码，这里比较麻烦的一点是，有很多代码我不能通过编译器直接看到，还好，Google 把相关的代码都放在 GitHub 上。

第一行代码中的 `VocabularyProcessor` 是词汇处理器，详细的<a href="https://github.com/tensorflow/tensorflow/blob/master/tensorflow/contrib/learn/python/learn/preprocessing/text.py" target="_blank" rel="noopener">代码</a>。`tf.contrib.learn.preprocessing` 这个模块主要包含了一些文本预处理的工具，而词汇处理器的作用呢，就是”Maps documents to sequences of word ids”，翻过来就是“把文档映射成词汇的索引序列”。

首先，需要理解我们是把文本，一段话，变成一个向量，这样机器处理起来就会比较方便。

这里我们在构造处理器的时候传入了 `MAX_DOCUMENT_LENGTH`这个参数，来控制我们的文本向量的最大长度，太长了训练起来比较困难。

那么文本向量是怎么编码的呢？就可以看到下面两代码：

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
<td class="code"><pre><code># 注意这两个函数不一样哟 一个是 fit_transform 一个是 transform
x_transform_train = vocab_processor.fit_transform(x_train)
x_transform_test = vocab_processor.transform(x_test)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

> fit\_transform(): Learn the vocabulary dictionary and return indexies of words，
>
> transform(): Transform documents to word-id matrix

这两个函数的返回是一个可迭代对象:

> x: iterable, \[n\_samples, max\_document\_length\]. Word-id matrix.

上面这几行代码干的就是**把文本变成向量**这样的工作。

首先构造词汇处理器，限制文本向量的最大长度，然后通过 `fit_transform()` 方法，这个函数内部是先 `fit` ，就是整体扫描一遍数据，给每个不同的单词一个编号，然后再 `transform`，也就是把每个对应的单词替换成数字。

看一个小例子：

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
<td class="code"><pre><code>x_text = [&#39;This is a cat&#39;,&#39;This must be boy&#39;, &#39;This is a a dog&#39;]
vocab_pro = VocabularyProcessor(6)
x = np.array(list(vocab_pro.fit_transform(x_text)))
print(x)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

输出结果:

> \[\[1 2 3 4 0 0\]  
> \[1 5 6 7 0 0\]  
> \[1 2 3 3 8 0\]\]

不难理解，数字 1 对应 “This”，数字 2 对应 “is”，以此类推，我们就能够得到一个词汇和数字一一对应的字典。需要特别提一下的是，我们可以给不认识的词汇统统做一个类似 `<UNK>` 的标记，以及没有词的地方就用 0 来表示。

在代码中有提到我们是对训练集进行 `fit` 和 `transform` ，对测试集进行 `transform`，显然，对测试集转换的时候使用的是从训练集上学得的字典。我觉得这里可能就是要求我们训练集尽可能大的一个地方，这样才能保证测试集中的词汇基本都能够出现，否则就会大大影响我们模型的准确性。

所以经过这一步，我们就把文本数据变成了机器能够比较容易处理的一组向量。

## <a href="#建立模型" class="headerlink" title="建立模型"></a>建立模型

代码里给出了两个模型，一个是 **RNN**，另一个是 **bag of words**，词袋模型。我个人理解，模型实质上就是一组**输入到输出的映射**，就相当于是函数`y = f(x)` 就是一个最简单的模型了。

### <a href="#RNN-Model" class="headerlink" title="RNN Model"></a>RNN Model

RNN，循环神经网络，公式和原理主要是在这篇 <a href="http://colah.github.io/posts/2015-08-Understanding-LSTMs/" target="_blank" rel="noopener">Blog</a> 中，这里就不赘述了。

我觉得比较重要的一点就是，RNN 能够捕获数据在时间（多表现为序列输入）上的特征，这和我们的文本的特点是非常的符合的。

和 RNN 经常一起使用的两个技术是 GRU 以及 LSTM，这两个是在 RNN 的单元上进行了一些变化，来避免 RNN 的梯度消失或者是梯度爆炸的缺点。这里代码里使用的是 GRU。

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
<td class="code"><pre><code>def rnn_model(features, labels, mode):
    &quot;&quot;&quot;用RNN模型来预测文本的类别&quot;&quot;&quot;
    
    # 词语索引 -&gt; 词嵌入 embedding 
    word_vectors = tf.contrib.layers.embed_sequence(
        features[WORDS_FEATURE], vocab_size=n_words, embed_dim=EMBEDDING_SIZE)

    # 展平向量  word_list 变成 [batch_size, EMBEDDING_SIZE] 形状的一个 tensor
    word_list = tf.unstack(word_vectors, axis=1)

    # 用嵌入层数量(也就是对应的GRU单元个数) 创建GRU单元
    cell = tf.contrib.rnn.GRUCell(EMBEDDING_SIZE)

    # 创建一个展平的 RNN 网络(其长度等于我们的MAX_DOCUMENT_LENGTH),并且把 word_list 作为输入
    _, encoding = tf.contrib.rnn.static_rnn(cell, word_list, dtype=tf.float32)

    # 得到通过 RNN 编码之后的一组向量(logits)，然后利用 softmax 来变成一个归一化概率的向量，从而预测对应的类别
    logits = tf.layers.dense(encoding, MAX_LABEL, activation=None)
    return estimator_spec_for_softmax_classification(
        logits=logits, labels=labels, mode=mode)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里涉及到一个我也并不是非常清楚的概念，有待日后继续学习：

`word embedding` 词嵌入，是在已经用数学向量表达的文本上再做一次变换，变换后的词向量会更能够代表词的特征（个人见解，存疑），并且比 `one-hot encoding` 编码更加节省空间，计算效率也更高。

### <a href="#Bag-of-Words-词袋模型" class="headerlink" title="Bag of Words 词袋模型"></a>Bag of Words 词袋模型

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
<td class="code"><pre><code>def bag_of_words_model(features, labels, mode):
    &quot;&quot;&quot;词袋模型，不考虑文本的顺序&quot;&quot;&quot;
    bow_column = tf.feature_column.categorical_column_with_identity(
        WORDS_FEATURE, num_buckets=n_words)

    bow_embedding_column = tf.feature_column.embedding_column(
        bow_column, dimension=EMBEDDING_SIZE)

    bow = tf.feature_column.input_layer(
        features,
        feature_columns=[bow_embedding_column])
    logits = tf.layers.dense(bow, MAX_LABEL, activation=None)

    return estimator_spec_for_softmax_classification(
        logits=logits, labels=labels, mode=mode)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

`bag of words` 和 RNN 最大的区别就在于**它不考虑文本的顺序**，只管文本中出现了什么词。这里做的和 RNN 类似，我们再一次把文本向量进行一次转换，只不过这一次不考虑文本的顺序，单纯看这句话里有什么词，然后给它对应的一个整数。（这块也不是很清楚，有待深入理解）

## <a href="#训练模型、预测、评价" class="headerlink" title="训练模型、预测、评价"></a>训练模型、预测、评价

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
<td class="code"><pre><code>def estimator_spec_for_softmax_classification(
        logits, labels, mode):
    &quot;&quot;&quot;Returns EstimatorSpec instance for softmax classification.&quot;&quot;&quot;
    predicted_classes = tf.argmax(logits, 1)
    if mode == tf.estimator.ModeKeys.PREDICT # Predict 时候用         
        return tf.estimator.EstimatorSpec(
            mode=mode,
            predictions={
                &#39;class&#39;: predicted_classes,
                &#39;prob&#39;: tf.nn.softmax(logits)
            })
 
      # 生成一个 one-hot 编码的 label
    onehot_labels = tf.one_hot(labels, MAX_LABEL, 1, 0)
     # loss 函数 交叉熵 
    loss = tf.losses.softmax_cross_entropy(
        onehot_labels=onehot_labels, logits=logits)
    # train 模式下使用
    if mode == tf.estimator.ModeKeys.TRAIN:
        # 设置 学习率为0.1的 AdamOptimizer
        optimizer = tf.train.AdamOptimizer(learning_rate=0.01)
        # minimize loss function
        train_op = optimizer.minimize(loss, global_step=tf.train.get_global_step())
        return tf.estimator.EstimatorSpec(mode, loss=loss, train_op=train_op)
    
    # Score 时候使用
    eval_metric_ops = {
        &#39;accuracy&#39;: tf.metrics.accuracy(
            labels=labels, predictions=predicted_classes)
    }
    return tf.estimator.EstimatorSpec(
        mode=mode, loss=loss, eval_metric_ops=eval_metric_ops)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里 train 的过程和以前写的非常类似。大致就是我们通过文本的类别来 生成对应的 `one-hot encoding` 的 label，然后通过 `cross entropy` 交叉熵函数作为我们的 loss function，然后用 `AdamOptimizer` 来训练这个模型。

预测的部分就是通过我们的模型生成对应的 logits，然后再通过 softmax 函数变成一个概率向量，其中最大的那一个一般就作为我们预测的类别。

评估也很简单，就看看我们预测的是否和测试集中的 label 吻合，计算准确率就行。

## <a href="#总结" class="headerlink" title="总结"></a>总结

完整的数据集还是挺大的，在我 i7 的 rmbp 上跑了大概一个半小时。

其实在学习代码的时候遇到了一些有意思的情况：

1.  把 `GRUCell` 换成 `LSTMCell` 报错，不知道为什么（应该是对应参数的问题吧）
2.  在小数据集上跑的时候，词袋模型比 RNN 的效果好很多。RNN的准确率在 31% ~ 38% 之间浮动，而词袋模型在58%左右；大数据集 RNN 在 87 % 而 词袋模型达到了 97 %！

这篇文章写的还是比较仓促，很多细节部分都没有深究，也只是这个方向上的第一步探索，接下来会对细节的一些地方进行深入的理解。
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color2">Machine Learning</a>
