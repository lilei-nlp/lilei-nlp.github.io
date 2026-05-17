---
title: "Pytorch 初体验"
date: 2018-06-30
categories:
  - blog
tags: []
---

深度学习框架很多，TensorFlow 算是比较流行的了，但静态图有时候确实写的很难受，以及一些比较复杂的 model 实现起来比较蛋疼；最近在看的文章的实现没有 TensorFlow 版本，起初我想改写一个 TensorFlow 版本出来，结果实现不了，一方面可能我确实比较菜，另一方面，也只能说 TensorFlow 有些东西在设计上让人为难。所以，那就试试 PyTorch 咯。这篇文章主要是记录我学习 PyTorch 官方 60 分钟<a href="https://pytorch.org/tutorials/beginner/deep_learning_60min_blitz.html" target="_blank" rel="noopener">教程</a>的过程。

## <a href="#Tensor" class="headerlink" title="Tensor"></a>Tensor

逃不开的还是 Tensor 这个对象，PyTorch 创建、操作 Tensor 的方法和 numpy 很类似，据说 Torch 一开始就是 Numpy 的 GPU 加速版，简单列些一些 API：

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
<td class="code"><pre><code>x = torch.empty(5, 3) # 创建一个未初始化的 5x3 tensor
x = torch.rand(5, 3) # 随机生成 5x3 
x = torch.zeros(5, 3, dtype=torch.long)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

比较有意思的是，创建 Tensor 的时候 size 参数不是一个 `tuple` e.g. `(5, 3)` ，而是用 `*args` 的方式传入。

常用的算术操作可以通过 `x+y` 或者是 `torch.add(x, y)` 来实现，同时 PyTorch 也提供了**就地** (in place) 的操作：

<figure class="highlight python">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1</code></pre></td>
<td class="code"><pre><code>y.add_(x) # equals y = y + x</code></pre></td>
</tr>
</tbody>
</table>
</figure>

所有的操作名 + 下划线都是一个就地操作，`x.copy_(y)` 会用 `y` 的值**替代**`x`。

另外，对于单元素的 Tensor，可以通过使用 `.item()` 来获取它；TensorFlow 中的 `reshape()` 对应的是 `view()`，并且，和 numpy 一样，**Tensor** 支持切片操作：

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
<td class="code"><pre><code>x = torch.randn(4, 4) # x -&gt; 4x4
x.view(-1 , 8) # x -&gt; 2 x 8
x[:, 1] # 取第二列的所有元素</code></pre></td>
</tr>
</tbody>
</table>
</figure>

既然之前说 Torch 是 Numpy 的一个 GPU 版本，也就支持 ndarray 和 Tensor 的相互转化（这就比 TensorFlow 用起来舒服很多），可以用 `Tensor.numpy()` 获取到对应的 ndarray，逆操作则是 `torch.from_numpy()`：

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
13</code></pre></td>
<td class="code"><pre><code># example 1 tensor -&gt; numpy
a = torch.ones(3) # tensor([1., 1. , 1.])
b = a.numpy() # 通过 tensor.numpy() 获取对应的 ndarray 对象
a.add_(1)  # 就地 + 1 , a= a+1
print(a)# tensor([2., 2., 2.])
print(b) # [2., 2., 2.]

# example 2   numpy -&gt; tensor
a = np.ones(3)
b = torch.from_numpy(a)
np.add(a, 1, out=a)
print(a)  # [2., 2., 2.]
print(b) # tensor([2., 2., 2.])</code></pre></td>
</tr>
</tbody>
</table>
</figure>

上面这个例子很重要，说明了**ndarray 和 Tensor 对象之间的转换是一种引用关系，而非创建新的对象**。也就是说，**任何一个相关对象的变化都会影响到另一方**，就像例子 1 中，我们对原始的 `a` 进行 `add_(1)`，在此之前得到的 `b` 对象的值也改变了，例子二中同样出现了这样的情况，这一点需要特别注意。

### <a href="#CUDA-Tensors" class="headerlink" title="CUDA Tensors"></a>CUDA Tensors

我们可以把 Tensor 放到 GPU 上来加速计算，而 PyTorch 为此提供了很方便的方式：

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
7</code></pre></td>
<td class="code"><pre><code>if torch.cuda.is_available(): # 如果 CUDA 可用
    device = torch.device(&quot;cuda&quot;)          # CUDA 设备对象
    y = torch.ones_like(x, device=device)  # 直接在 GPU 上创建 Tensor 对象
    x = x.to(device)                       # 也可以用``.to(&quot;cuda&quot;)`` 代替
    z = x + y
    print(z)
    print(z.to(&quot;cpu&quot;, torch.double))       # ``.to`` 同时可以改变数据类型</code></pre></td>
</tr>
</tbody>
</table>
</figure>

如上面的例子所示，我们可以在创建 Tensor 对象时指定 `device` 参数来指明 Tensor 创建的位置，也可以再创建了之后使用 `.to()` 方法来进行双向的迁移：既可以从 CPU 到 GPU（例子中的 `x.to(device)`，也可以从 GPU 到 CPU（`z.to("cpu", torch.double)`）。对比 TensorFlow ：

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
<td class="code"><pre><code>with tf.device(&#39;/cpu:0&#39;):
    # something you want to do on CPU
with tf.device(&#39;/gpu:0&#39;):
    # somethin you want to do on GPU</code></pre></td>
</tr>
</tbody>
</table>
</figure>

就我所知，TensorFlow 只能在创建的时候指明位置，而无法像 PyTorch 这般灵活的迁移。

## <a href="#Autograd" class="headerlink" title="Autograd"></a>Autograd

自动求导已经是深度学习框架的必备了，首先，我们可以通过设置 Tensor 的 `.requires_grad` 参数来指明 Tensor 是否参与梯度运算，这和 TensorFlow 中的 `trainable` 是类似的；接着我们通过对某个对象（比如 loss ）函数进行 `.backward()` 操作，来进行反向梯度的计算，并且能够通过计算链上对象的 `.grad` 属性来获取到对应的梯度。拿官方的例子做一个简单的说明：

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
<td class="code"><pre><code>x = torch.ones(2, 2, requires_grad=True) # 声明 x，指明需要参加梯度的计算 x: 2x2 全 1
y = x + 2
z = y * y * 3 # z = 3 * (x+2)^2
out = z.mean()</code></pre></td>
</tr>
</tbody>
</table>
</figure>

梯度计算的过程如下：

$ o =\frac{1}{4}\sum\_i z\_i$

$ z\_i = 3(x\_i+2)^2 $

$ z\_i\bigr\rvert\_{x\_i=1} = 27 $

$ \frac{\partial o}{\partial x\_i} = \frac{3}{2}(x\_i+2) $

$ \frac{\partial o}{\partial x\_i}\bigr\rvert\_{x\_i=1} = \frac{9}{2} = 4.5 $

因此，`x.grad` 的值在我们对 `out` 进行 `out.backward()` 操作之后，结果为：

> tensor(\[\[ 4.5000, 4.5000\],
>
> ​ \[ 4.5000, 4.5000\]\])

`backward()` 函数的原型如下：

> torch.autograd.backward(*tensors*, *grad\_tensors=None*, *retain\_graph=None*, *create\_graph=False*, *grad\_variables=None*)

其中第一个参数用来声明**梯度的权重**。如果目标函数是一个标量，则可以不传这个参数，比如上面的 `out.backward()` ，则默认权重为 `1` ，如果我们改成 `out.backward(torch.tensor(2,dtype=torch.float))` 则各个 $$x\_i$$ 梯度就会变成原来的 2 倍，利用权重我们就可以来控制各个参数的更新速度。

## <a href="#Build-A-Neural-Network" class="headerlink" title="Build  A Neural Network"></a>Build A Neural Network

学框架不搭一个 NN 来玩 MNIST 或者 CIFAR 怎么能叫入门呢？来来来，走一波~

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
27</code></pre></td>
<td class="code"><pre><code>class Net(nn.Module):
    def __init__(self):
        super(Net, self).__init__()
        self.conv1 = nn.Conv2d(3, 6, 5) # input 3 channels, output 6 channels , filter size = 5
        self.conv2 = nn.Conv2d(6, 16, 5) # input 3 channels, output 16 channels, filter size = 5
        # Wx + b
        self.fc1 = nn.Linear(16 * 5 * 5, 120)
        self.fc2 = nn.Linear(120, 84)
        self.fc3 = nn.Linear(84, 10)

    def forward(self, x):
        # max pooling, window size : 2x2
        x = F.max_pool2d(F.relu(self.conv1(x)), (2, 2))
        # 如果是 方形的输入，window size 可以只用一个参数
        x = F.max_pool2d(F.relu(self.conv2(x)), 2)
        x = x.view(-1, self.num_flat_features(x))
        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))
        x = self.fc3(x)
        return x

    def num_flat_features(self, x): # 展平 x
        size = x.size()[1:]  # 获取各个维度的大小，除了第一维 batch_size
        num_features = 1
        for s in size:
            num_features *= s
        return num_features</code></pre></td>
</tr>
</tbody>
</table>
</figure>

诶，有没有觉得有点像 Keras，不过高级的 API 用起来就是爽，不用写那么多代码2333，接下来就是定义 loss 和进行 backprop 来更新参数：

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
<td class="code"><pre><code>import torch.optim as optim
# 定义 loss
criterion = nn.CrossEntropy()
# 创建 optimizer
optimizer = optim.SGD(net.parameters(), lr=0.01)

# 在循环里不断进行如下操作：
optimizer.zero_grad()   # 清零梯度
output = net(input)
loss = criterion(output, target)
loss.backward()
optimizer.step()    # 梯度更新</code></pre></td>
</tr>
</tbody>
</table>
</figure>

和 TensorFlow 的区别就在于，TensorFlow 的话只要我们用 `session.run(optimizer)` 就可以完成梯度的更新操作，而 PyTorch 则需要：

1.  Optimizer 清零梯度
2.  对 loss 进行手动的 `backward()` 操作
3.  Optimizer 更新 `.step()`

三步走，记住了没有！

还有一点比较方便的是，和 Tensor 类似，我们可以使用 `Net.to()` 来把模型部署到 GPU 上：

<figure class="highlight python">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1</code></pre></td>
<td class="code"><pre><code>net.to(&quot;cuda:0&quot;)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

但是！记得要把你的输入也放到 GPU 上，不然就会报错：

<figure class="highlight python">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1</code></pre></td>
<td class="code"><pre><code>inputs, labels = inputs.to(device), labels.to(device) # 把 input 和 label 同样迁移到 GPU 上</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

初体验如果要给个评价的话，我觉得是比 TensorFlow 好很多（毕竟 TensorFlow 一上来的 Session、静态图会让人有点摸不着头脑）。但用什么框架其实都无所谓，就和语言一样，虽然争来争去，但每种语言都有自己的用武之地，以前我还会和室友争辩 Java 和 Python 谁才是最好的语言，现在就不会了（因为我也觉得 Python 好写一点，逃）。框架更不用说，TensorFlow 有他应用的工业场景（希望能早日接触到），PyTorch 现在看来也许更适合需要快速实现原型的科研人员。而我们能做的，就是多接触，横向比较着来看而不要因为自己擅长而蒙蔽了双眼，**多学一技压身，总是没错的**。

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">Deep Learning</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2018/06/30/Pytorch-Tutorial-Walkthrough/)