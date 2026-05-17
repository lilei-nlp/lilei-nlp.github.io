---
title: "AdapterPatternLearning"
date: 2017-04-04
categories:
  - blog
tags: []
---

适配器模式：将一个类的接口转换成另一个接口，以符合需求。让接口不兼容的类可以合作无间。

​更重要的是，适配器将**客户和被适配者解耦**，当接口改变时，利用适配器来封装改变的部分，就可以在不改动客户调用方式的情况下实现更新。

​打个最简单的例子，插座转换器，比如买的港版手机不能插国内的插头，中间加一个插头转换器，就可以了。这个转换器，起到的就是适配器（Adapter）的作用。

![Adapter](/img/adapter.png)

## <a href="#Exmaple" class="headerlink" title="Exmaple"></a>Exmaple

​Java早期的集合（Collection）类型（e.g. Vector,Stack,Hashtable）中都实现了一个名为elements（）的方法，这个方法会返回一个Enumeration（枚举），这个接口有两个方法`hasMoreElemnts()`和`nextElement()`，从而能够在不知道集合的情况下，遍历集合中的元素。

​Sun之后更新了集合类，开始使用Iterator（迭代器）接口，这个接口和枚举接口很类似，有三个方法，分别是`hasNext()`、`next()`、`remove()`，这个remove()就是二者主要的区别。

​那么问题来了，如何适配新老版本的枚举和迭代器接口？适配器就是这个时候派上用场的。  
<span id="more"></span>

#### <a href="#处理remove-方法" class="headerlink" title="处理remove()方法"></a>处理remove()方法

​枚举不支持删除，因为枚举是一个“只读”接口。适配器无法实现一个有实际功能的remove()方法，最多只能抛出一个运行时异常。迭代器的设计者事先考虑到了这一种情况（真是机智啊），所以我们可以将remove()方法定义为会抛出UnsupportedOperationException。

​**适配器并不完美**，客户必须小心潜在的异常（比如不支持），但配合文档的说明和足够小心，这个方案也算合理。

#### <a href="#Code" class="headerlink" title="Code"></a>Code

​接下来就是用代码来实现简单的Enumeration适配器。

<figure class="highlight java">
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
19</code></pre></td>
<td class="code"><pre><code>public class EnumerationIterator implements Iterator{ // 适配器实现被适配者
  Enumeration enum; 
  
  public EnumerationIterator(Enumeration enum) { // 利用组合，将枚举结合进适配器
    this.enum = enum;
  }
  
  public boolean hasNext() {  // 适配器的hasNext()方法实际是委托给enum的hasMoreElements()方法
    return enum.hasMoreElements();
  }
  
  public Object next() { // 同样的 交给enum的nextElement()方法
    return enum.nextElement();
  }
  
  public void remove() {
    throw new UnsupportedOperationException() ; // 不支持该操作，抛出异常
  }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#装饰者模式和适配者模式的区别" class="headerlink" title="装饰者模式和适配者模式的区别"></a>装饰者模式和适配者模式的区别

​还是插头的例子，有些比较智能的插头，不仅能转换插头的样式来适配插口，还有可能更多的功能，比如：涓流充电、指示灯、警报声等等。在需要实现这些新特性的时候，适配者模式就显得有些无力，就要用装饰者模式。

​装饰者模式，简单来说就是给对象包装上一层有一层的外套，每添加一层就可以增加新的行为。它的主要目的是**扩展对象的行为和责任**，但是使用起来真的是…个人觉得很是繁琐。比如JavaIO里对Stream的一系列类都是装饰者类，使用起来就会有很恐怖的构造序列：

<figure class="highlight java">
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
8</code></pre></td>
<td class="code"><pre><code>File file = new File(&quot;Path&quot;);
try {
 DataInputStream din = new DataInputStream(
       new BufferedInputStream(
         new FileInputStream(file)));
} catch (FileNotFoundException e) {
 e.printStackTrace();
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​恐怖的原因其实就在于我们对于这个流的处理有很多需求，比如要利用缓冲机制，所以就套了一层BufferedInputStream，又因为是要从文件来读取输入流，内层就是FileInputStream。

​而适配者模式，其核心就是**传送需求**，把所有的需求都委托给其内部持有的对象去实现。用户只要调用接口，适配者把脏活累活全做了，从而实现了用户和被适配者的解耦。

​**设计模式没有好坏之分，一切都是需求主导**

​
