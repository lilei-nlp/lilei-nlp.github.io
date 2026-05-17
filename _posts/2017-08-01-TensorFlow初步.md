---
title: "TensorFlow初步"
date: 2017-08-01
categories:
  - blog
tags: []
---

其实这是一篇 Google 官网 TensorFlow Tutorial 的学习笔记，主要侧重记录使用 TensorFlow 框架的一个流程，细节暂且按下不表。

首先是导入 TensorFlow 以及准备数据

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
<td class="code"><pre><code>import tensorflow as tf # 约定的简写
from tensorflow.examples.tutorials.mnist import input_data

# 准备mnist数据集
mnist = input_data.read_data_sets(&#39;MNIST_data&#39;, one_hot=True)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

提一下 `one hot encoding`，就是一个由 0/1 组成的向量来表示的 label，如果说数字是“0”，那么对应的 label 就是\[1,0,0,…,0\]，如果是“1”，那么就是\[0,1,0,…,0\]

这篇教程里使用了两个模型，一个是线性的“Wx+b”，还有一个则是CNN，因为主要侧重使用的结构以及我对CNN不是很熟悉（后者才是主要原因），所以这篇就只介绍第一种模型。  
<span id="more"></span>

然后就是使用 TensorFlow 的主体框架了。

首先是规定我们的输入`X，`以及对应的 label `y`

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
<td class="code"><pre><code># placeholder 占位符 等待外部传进来的变量
x = tf.placeholder(tf.float32, [None, 784])
y_ = tf.placeholder(tf.float32, [None, 10])</code></pre></td>
</tr>
</tbody>
</table>
</figure>

传入的两个参数分别是

`dtype`：数据的类型，多是`tf.float32`

`shape`：数据的形状，因为输入是一张 `28*28`的图片，我们将它展开成一个`1*784`的向量，输出则是经过 `one-hot encoding` 的十维的向量。前面的一个参数 `None`代表可以是任何值，一般这里我们会填入数据集的大小`dataset_size`。因为这里我们并没有预先知道数据集的大小，所以就用了`None`。

那么接下来就是要引入我们的需要学习的两个变量，`Weight` 和 `bias`，这里的大写的

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
<td class="code"><pre><code># 设置变量 tf.Variable
# 权重 这里是一个784*10的矩阵 这样就可以在 W*x之后变成一个1*10的向量
W = tf.Variable(tf.zeros([784, 10]))
# 偏置 相当一个常数项
b = tf.Variable(tf.zeros([10]))</code></pre></td>
</tr>
</tbody>
</table>
</figure>

**注意**：所有 TensorFlow 的变量在使用之前都需要**初始化**

初始化的过程是这样的：

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
<td class="code"><pre><code>sess = tf.Session()
init = tf.global_variables_initializer()
sess.run(init)
# 或者是 sess.run(tf.global_variables_initializer())</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里又出现了一个新的东西：`Session`。TensorFlow 后台的计算是交给C++去实现的（不然难道用Python？），Session就是 TensorFlow 与后台计算的连接。简单来说，我们只要把计算丢到Session里去，让它run就行。

接下来，我们定义一下我们的输出

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
<td class="code"><pre><code># 我们的模型输出 y = Wx + b
y = tf.matmul(x, W) + b</code></pre></td>
</tr>
</tbody>
</table>
</figure>

然后是重要的**loss function**，这里我们用 cross\_entropy （交叉熵）作为误差函数。这里就不赘述原理了，大致就是通过`softmax`函数将我们的输出变成一个概率，再选取0概最大的一项作为输出向量中的“1”，其他为“0”，然后我们再通过一个函数来衡量与标签的误差，最后求和取平均。

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
<td class="code"><pre><code># cross_entropy
cross_entropy = tf.reduce_mean(
    tf.nn.softmax_cross_entropy_with_logits(labels=y_, logits=y))</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里我们使用 GradientDescent（梯度下降）算法来使我们的loss function最小，并且设置`0.05`的学习率。

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
<td class="code"><pre><code># 训练方法
train_step = tf.train.GradientDescentOptimizer(0.05).minimize(cross_entropy)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

终于到了训练模型的时候了：

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
<td class="code"><pre><code>for _ in range(1000):
    # 我们每次训练100个数据 训练1000次
    batch = mnist.train.next_batch(100)
    # 注意后面的 feed_dict 我们传入我们的训练集数据 
    sess.run(train_step, feed_dict={x: batch[0], y_: batch[1]})</code></pre></td>
</tr>
</tbody>
</table>
</figure>

然后就是评估模型准确度的时候了，比较输出的向量和测试集中的label。

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
<td class="code"><pre><code>correct_prediction = tf.equal(tf.argmax(y, 1), tf.arg_max(y_, 1))
accuracy = tf.reduce_mean(tf.cast(correct_prediction, dtype=tf.float32))
# 注意喂给它的数据是mnist中的测试集
print(sess.run(accuracy, feed_dict={x: mnist.test.images, y_: mnist.test.labels}))</code></pre></td>
</tr>
</tbody>
</table>
</figure>

然后看看我们的输出

> 0.9214

其实这个accuracy是很低的，在MNIST上，没到99%都是很差的，当然这也可以理解，因为我们用的是一个最简单Softmax Regression的模型，隔壁CNN一下子就能到达99.2%。

## <a href="#总结" class="headerlink" title="总结"></a>总结

在TF上用最简单的Model实现MNIST的识别，主要流程（其实貌似也是整个ML的流程）如下：

- 准备工作：导包，导数据，数据处理，模型的选择
- 定义：变量、数据等等（**初始化很重要**）
- 训练：丢进`session`去跑，必要时还要利用可视化工具检查训练的情况
- 评估：模型是否 overfitting / underfitting 等等

接下来就得迅速进入 RNN 以及 LSTM 去做 Text Classification了，加油了！
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">TensorFlow</a>
