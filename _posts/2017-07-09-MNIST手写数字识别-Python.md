---
title: "MNIST手写数字识别-Python"
date: 2017-07-09
categories:
  - blog
tags: []
---

​上篇blog讲了神经网络中BP反向传播算法的推导，并且在Andrew Ng的课程中用Matlab实现了MNIST手写数字数据集的识别。这次决定用Python的sk-learn库来实现（调包）一次。

## <a href="#数据获取以及处理" class="headerlink" title="数据获取以及处理"></a>数据获取以及处理

​Google一下，就能找到<a href="http://yann.lecun.com/exdb/mnist/" target="_blank" rel="noopener">MNIST</a>的网站，下载四个数据集。分别是:

> `train-images-idx3-ubyte.gz: training set images (9912422 bytes)`  
> `train-labels-idx1-ubyte.gz: training set labels (28881 bytes)`  
> `t10k-images-idx3-ubyte.gz: test set images (1648877 bytes)`  
> `t10k-labels-idx1-ubyte.gz: test set labels (4542 bytes)`

​解压之后发现Window自作主张的把第二个`-`变成了`.`， 文件后缀也变成了`idx1-ubyte`。不过这不是什么大问题，问题是，我们怎么把这个格式的文件变成我们想要的一组特征向量以及labels。

> ### <a href="#TRAINING-SET-LABEL-FILE-train-labels-idx1-ubyte" class="headerlink" title="TRAINING SET LABEL FILE (train-labels-idx1-ubyte):"></a>TRAINING SET LABEL FILE (train-labels-idx1-ubyte):
>
> offset type value description\]\[description\]
>
> 0000 32 bit integer 0x00000801(2049) magic number (MSB first)
>
> 0004 32 bit integer 60000 number of items
>
> 0008 unsigned byte ?? label
>
> 0009 unsigned byte ?? label
>
> ……..
>
> xxxx unsigned byte ?? label
>
> `The labels values are 0 to 9.`
>
> ### <a href="#TRAINING-SET-IMAGE-FILE-train-images-idx3-ubyte" class="headerlink" title="TRAINING SET IMAGE FILE (train-images-idx3-ubyte):"></a>TRAINING SET IMAGE FILE (train-images-idx3-ubyte):
>
> offset type value description
>
> 0000 32 bit integer 0x00000803(2051) magic number
>
> 0004 32 bit integer 60000 number of images
>
> 0008 32 bit integer 28 number of rows
>
> 0012 32 bit integer 28 number of columns
>
> 0016 unsigned byte ?? pixel
>
> 0017 unsigned byte ?? pixel
>
> ……..
>
> xxxx unsigned byte ?? pixel
>
> Pixels are organized row-wise. Pixel values are 0 to 255. 0 means background (white), 255 means foreground (black).

​网站也很贴心的给出了文件格式的描述，train\_set和label都是有一个文件头的。然而还是拿这个ubyte文件没有办法（PS:我Python真的菜)，最后在网上找到了利用`struct`模块来处理二进制的方法:

<span id="more"></span>

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
53</code></pre></td>
<td class="code"><pre><code>def decode_idx3_ubyte(idx3_ubyte_file):
    &quot;&quot;&quot;
    解析idx3文件的通用函数
    :param idx3_ubyte_file: idx3文件路径
    :return: 数据集
    &quot;&quot;&quot;
    # 读取二进制数据
    bin_data = open(idx3_ubyte_file, &#39;rb&#39;).read()

    # 解析文件头信息，依次为魔数、图片数量、每张图片高、每张图片宽
    offset = 0
    fmt_header = &#39;&gt;iiii&#39;
    magic_number, num_images, num_rows, num_cols = struct.unpack_from(fmt_header, bin_data, offset)
    print(&#39;魔数:%d, 图片数量: %d张, 图片大小: %d*%d&#39; % (magic_number, num_images, num_rows, num_cols))

    # 解析数据集
    image_size = num_rows * num_cols
    offset += struct.calcsize(fmt_header)
    fmt_image = &#39;&gt;&#39; + str(image_size) + &#39;B&#39;
    images = np.empty((num_images, num_rows, num_cols))
    for i in range(num_images):
        if (i + 1) % 10000 == 0:
            print(&#39;已解析 %d&#39; % (i + 1) + &#39;张&#39;)
        images[i] = np.array(struct.unpack_from(fmt_image, bin_data, offset)).reshape((num_rows, num_cols))
        offset += struct.calcsize(fmt_image)
    return images

def decode_idx1_ubyte(idx1_ubyte_file):
    &quot;&quot;&quot;
    解析idx1文件的通用函数
    :param idx1_ubyte_file: idx1文件路径
    :return: 数据集
    &quot;&quot;&quot;
    # 读取二进制数据
    bin_data = open(idx1_ubyte_file, &#39;rb&#39;).read()

    # 解析文件头信息，依次为魔数和标签数
    offset = 0
    fmt_header = &#39;&gt;ii&#39;
    magic_number, num_images = struct.unpack_from(fmt_header, bin_data, offset)
    print(&#39;魔数:%d, 图片数量: %d张&#39; % (magic_number, num_images))

    # 解析数据集
    offset += struct.calcsize(fmt_header)
    fmt_image = &#39;&gt;B&#39;
    labels = np.zeros((num_images, 10), dtype=&#39;int8&#39;)
    for i in range(num_images):
        if (i + 1) % 10000 == 0:
            print(&#39;已解析 %d&#39; % (i + 1) + &#39;张&#39;)
        digit = (int)(struct.unpack_from(fmt_image, bin_data, offset)[0])
        labels[i][digit] = 1
        offset += struct.calcsize(fmt_image)
    return labels</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​从二进制流中先读出文件头，确定图片数量，宽、高，然后通过`struct.unpacked_from(fmt, binfile, offset)`这个方法来读出每一个图片的相关信息，这里把整个数据集储存成了一个三维的ndarray。而label这边我对原有的代码进行了一些修改，把label的值（0~9），展开成了一个`1x10`的数组，方便后面使用神经网络来训练模型。

## <a href="#利用sk-learn库来训练神经网络" class="headerlink" title="利用sk-learn库来训练神经网络"></a>利用sk-learn库来训练神经网络

​数字识别实际上就是个多分类问题，用神经网络可以很好的解决。在Andrew Ng的课里是手搓了一部分Matlab代码来实现数字识别的，今天用Python，做一次调包侠，就可以很简单（偷懒）的完成了。

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
<td class="code"><pre><code>def mnistUsingNN(train_dataSet, train_labels, test_dataSet, test_labels):
    # 初始化一个分类器，传入设定的参数
   clf = MLPClassifier(hidden_layer_sizes=(100, 50, 25),
                        activation=&#39;logistic&#39;, solver=&#39;adam&#39;,
                        learning_rate_init=0.001, max_iter=2000)
    print(&#39;开始训练模型&#39;)
    start = time.time()
    clf.fit(train_dataSet, train_labels)
    print(&#39;训练完毕, 时间:&#39; + str(time.time() - start))

    res = clf.predict(test_dataSet)  # 对测试集进行预测
    error_num = 0  # 统计预测错误的数目
    num = len(test_dataSet)  # 测试集的数目
    for i in range(num):  # 遍历预测结果
        # 比较长度为10的数组，返回包含01的数组，0为不同，1为相同
        # 若预测结果与真实结果相同，则10个数字全为1，否则不全为1
        if np.sum(res[i] == test_labels[i]) &lt; 10:
            error_num += 1
    print(&quot;Total num:&quot;, num, &quot; Wrong num:&quot;,
          error_num, &quot;  CorrectRate:&quot;, (1 - error_num / float(num)) * 100, &#39;%&#39;)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​这里我参数设了hidden layers为3， 每层的个数分别为(100, 50, 25)， 而默认值是一层，100个`神经元`。最大迭代次数设置成了2000，基本能够保证converge了，learning\_rate设置成了0.0001。

​参数的设置比如hidden layer的层数，一般来说是层数越多效果越好，一层的时候准确率在92%， 2层就到了93%，三层达到了95%，当然训练的时间也是在不断地增加，三层用mbp跑，花了213s，用神舟跑大概是280s；learning\_rate对于最后的结果也是有比较大的影响，这里的learning\_rate在0.001的时候准确率只有91%，调到0.0001之后达到了95%。

​有一点需要注意，在传参数的时候，我们读进来的train\_dataSet是三维的，需要reshape一下，变成`60000x784`，也就是每一张图片对应一个列向量。

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
<td class="code"><pre><code>train_dataSet = train_images.reshape(60000, 784)
test_dataSet = test_images.reshape(10000, 784)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#总结" class="headerlink" title="总结"></a>总结

​调包的过程确实是愉快而且轻松的，但仅仅是调包、调参，可能在这种比较简单的场景之下能够达到比较高的精确度，在复杂的情况下，精确度达不到要求的时候就需要对过程进行详细分析、找出问题所在（比如过拟合或者欠拟合）。这时候就要借助一些可视化的工具（比如learing-curve)来帮助分析。这些能力是更加重要，也是仅仅调包学不到的。

## <a href="#完整代码" class="headerlink" title="完整代码"></a>完整代码

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
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
125
126
127
128
129
130
131
132
133
134
135
136
137
138
139
140
141
142
143
144
145
146
147
148
149
150
151
152
153
154
155
156
157
158
159
160
161
162
163
164
165
166
167
168
169
170
171
172
173
174
175
176
177
178
179
180
181
182
183</code></pre></td>
<td class="code"><pre><code># encoding: utf-8

import numpy as np
import struct
import matplotlib.pyplot as plt
from sklearn.neural_network import MLPClassifier
from sklearn import neighbors
import time

# 训练集文件
train_images_idx3_ubyte_file = &#39;./train-images-idx3-ubyte&#39;
# 训练集标签文件
train_labels_idx1_ubyte_file = &#39;./train-labels-idx1-ubyte&#39;

# 测试集文件
test_images_idx3_ubyte_file = &#39;./t10k-images-idx3-ubyte&#39;
# 测试集标签文件
test_labels_idx1_ubyte_file = &#39;./t10k-labels-idx1-ubyte&#39;


def decode_idx3_ubyte(idx3_ubyte_file):
    &quot;&quot;&quot;
    解析idx3文件的通用函数
    :param idx3_ubyte_file: idx3文件路径
    :return: 数据集
    &quot;&quot;&quot;
    # 读取二进制数据
    bin_data = open(idx3_ubyte_file, &#39;rb&#39;).read()

    # 解析文件头信息，依次为魔数、图片数量、每张图片高、每张图片宽
    offset = 0
    fmt_header = &#39;&gt;iiii&#39;
    magic_number, num_images, num_rows, num_cols = struct.unpack_from(fmt_header, bin_data, offset)
    print(&#39;魔数:%d, 图片数量: %d张, 图片大小: %d*%d&#39; % (magic_number, num_images, num_rows, num_cols))

    # 解析数据集
    image_size = num_rows * num_cols
    offset += struct.calcsize(fmt_header)
    fmt_image = &#39;&gt;&#39; + str(image_size) + &#39;B&#39;
    images = np.empty((num_images, num_rows, num_cols))
    for i in range(num_images):
        if (i + 1) % 10000 == 0:
            print(&#39;已解析 %d&#39; % (i + 1) + &#39;张&#39;)
        images[i] = np.array(struct.unpack_from(fmt_image, bin_data, offset)).reshape((num_rows, num_cols))
        offset += struct.calcsize(fmt_image)
    return images


def decode_idx1_ubyte(idx1_ubyte_file):
    &quot;&quot;&quot;
    解析idx1文件的通用函数
    :param idx1_ubyte_file: idx1文件路径
    :return: 数据集
    &quot;&quot;&quot;
    # 读取二进制数据
    bin_data = open(idx1_ubyte_file, &#39;rb&#39;).read()

    # 解析文件头信息，依次为魔数和标签数
    offset = 0
    fmt_header = &#39;&gt;ii&#39;
    magic_number, num_images = struct.unpack_from(fmt_header, bin_data, offset)
    print(&#39;魔数:%d, 图片数量: %d张&#39; % (magic_number, num_images))

    # 解析数据集
    offset += struct.calcsize(fmt_header)
    fmt_image = &#39;&gt;B&#39;
    labels = np.zeros((num_images, 10), dtype=&#39;int8&#39;)
    for i in range(num_images):
        if (i + 1) % 10000 == 0:
            print(&#39;已解析 %d&#39; % (i + 1) + &#39;张&#39;)
        digit = (int)(struct.unpack_from(fmt_image, bin_data, offset)[0])
        labels[i][digit] = 1
        offset += struct.calcsize(fmt_image)
    return labels


def load_train_images(idx_ubyte_file=train_images_idx3_ubyte_file):
    
    return decode_idx3_ubyte(idx_ubyte_file)


def load_train_labels(idx_ubyte_file=train_labels_idx1_ubyte_file):

    return decode_idx1_ubyte(idx_ubyte_file)


def load_test_images(idx_ubyte_file=test_images_idx3_ubyte_file):
    &quot;&quot;&quot;
    TEST SET IMAGE FILE (t10k-images-idx3-ubyte):
    [offset] [type]          [value]          [description]
    0000     32 bit integer  0x00000803(2051) magic number
    0004     32 bit integer  10000            number of images
    0008     32 bit integer  28               number of rows
    0012     32 bit integer  28               number of columns
    0016     unsigned byte   ??               pixel
    0017     unsigned byte   ??               pixel
    ........
    xxxx     unsigned byte   ??               pixel
    Pixels are organized row-wise. Pixel values are 0 to 255. 0 means background (white), 255 means foreground (black).

    :param idx_ubyte_file: idx文件路径
    :return: n*row*col维np.array对象，n为图片数量
    &quot;&quot;&quot;
    return decode_idx3_ubyte(idx_ubyte_file)


def load_test_labels(idx_ubyte_file=test_labels_idx1_ubyte_file):
    &quot;&quot;&quot;
    TEST SET LABEL FILE (t10k-labels-idx1-ubyte):
    [offset] [type]          [value]          [description]
    0000     32 bit integer  0x00000801(2049) magic number (MSB first)
    0004     32 bit integer  10000            number of items
    0008     unsigned byte   ??               label
    0009     unsigned byte   ??               label
    ........
    xxxx     unsigned byte   ??               label
    The labels values are 0 to 9.

    :param idx_ubyte_file: idx文件路径
    :return: n*1维np.array对象，n为图片数量
    &quot;&quot;&quot;
    return decode_idx1_ubyte(idx_ubyte_file)

# 用KNN来实现
def mnistUsingKNN(train_dataSet, train_labels, test_dataSet, test_labels):

    knn = neighbors.KNeighborsClassifier(algorithm=&#39;kd_tree&#39;, n_neighbors=10)
    print(&#39;开始训练模型&#39;)
    start = time.time()
    knn.fit(train_dataSet, train_labels)
    print(&#39;训练完毕, 时间:&#39; + str(time.time() - start))

    res = knn.predict(test_dataSet)  # 对测试集进行预测
    error_num = 0  # 统计预测错误的数目
    num = len(test_dataSet)  # 测试集的数目
    print(num)
    for i in range(num):  # 遍历预测结果
        # 比较长度为10的数组，返回包含01的数组，0为不同，1为相同
        # 若预测结果与真实结果相同，则10个数字全为1，否则不全为1
        if np.sum(res[i] == test_labels[i]) &lt; 10:
            error_num += 1
    print(&quot;Total num:&quot;, num, &quot; Wrong num:&quot;, \
          error_num, &quot;  CorrectRate:&quot;, (1-error_num / float(num)) * 100, &quot;%&quot;)




def mnistUsingNN(train_dataSet, train_labels, test_dataSet, test_labels):

    clf = MLPClassifier(hidden_layer_sizes=(100, 50, 25),
                        activation=&#39;logistic&#39;, solver=&#39;adam&#39;,
                        learning_rate_init=0.0001, max_iter=2000)
    print(&#39;开始训练模型&#39;)
    start = time.time()
    clf.fit(train_dataSet, train_labels)
    print(&#39;训练完毕, 时间:&#39; + str(time.time() - start))

    res = clf.predict(test_dataSet)  # 对测试集进行预测

    error_num = 0  # 统计预测错误的数目
    num = len(test_dataSet)  # 测试集的数目

    for i in range(num):  # 遍历预测结果
        # 比较长度为10的数组，返回包含01的数组，0为不同，1为相同
        # 若预测结果与真实结果相同，则10个数字全为1，否则不全为1
        if np.sum(res[i] == test_labels[i]) &lt; 10:
            error_num += 1
    print(&quot;Total num:&quot;, num, &quot; Wrong num:&quot;,
          error_num, &quot;  CorrectRate:&quot;, (1 - error_num / float(num)) * 100, &#39;%&#39;)

if __name__ == &#39;__main__&#39;:

    train_images = load_train_images()
    train_labels = load_train_labels()
    test_images = load_test_images()
    test_labels = load_test_labels()

    train_dataSet = train_images.reshape(60000, 784)
    test_dataSet = test_images.reshape(10000, 784)


    mnistUsingNN(train_dataSet, train_labels, test_dataSet, test_labels)
    # mnistUsingKNN(train_dataSet, train_labels, test_dataSet,test_labels)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​

​

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color2">Machine Learning</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color2">Python</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/07/09/MNIST手写数字识别-Python/)