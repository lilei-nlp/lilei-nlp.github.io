---
title: "LaTeX Learning Notes"
date: 2018-01-22
categories:
  - blog
tags: []
---

LaTeX 是论文写作中非常重要的一个排版工具，用这篇文章记录下学习过程中的一些语法、注意事项，以便日后查阅。

## <a href="#Installation" class="headerlink" title="Installation"></a>Installation

配置 MacOS 上的 LaTeX 环境还是比较轻松愉快的

1.  安装 <a href="https://www.sublimetext.com/" target="_blank" rel="noopener">Sublime Text</a>

2.  下载 <a href="http://www.tug.org/mactex/" target="_blank" rel="noopener">MacTeX</a>，选择最全的版本 3.1 G

3.  安装 <a href="https://skim-app.sourceforge.io/" target="_blank" rel="noopener">Skim</a>，用来预览，并且在`选项-同步`中选择 `Sublime Text` ，如下所示：

    ![Skim Setting](/img/skim.png)

4.  在 Sublime Text 中安装 LaTexTools 和配置编译路径，首先安装 Sublime Text 的包管理器，点击 `View>Show Console` ，在控制台中输入下列 Python 代码安装：

    <figure class="highlight python">
    <table>
    <colgroup>
    <col style="width: 50%" />
    <col style="width: 50%" />
    </colgroup>
    <tbody>
    <tr>
    <td class="gutter"><pre><code>1</code></pre></td>
    <td class="code"><pre><code>import urllib.request,os,hashlib; h = &#39;6f4c264a24d933ce70df5dedcf1dcaee&#39; + &#39;ebe013ee18cced0ef93d5f746d80ef60&#39;; pf = &#39;Package Control.sublime-package&#39;; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( &#39;http://packagecontrol.io/&#39; + pf.replace(&#39; &#39;, &#39;%20&#39;)).read(); dh = hashlib.sha256(by).hexdigest(); print(&#39;Error validating download (got %s instead of %s), please try manual install&#39; % (dh, h)) if dh != h else open(os.path.join( ipp, pf), &#39;wb&#39; ).write(by)</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    安装成功之后，在 Sublime 中使用`Command+Shift+P`，输入 `install` ，按下`enter`，加载仓库后，再输入LatexTools，回车确认后等待安装完成。

5.  但因为路径的锅，如果我们现在打开 Sublime Text 复制进去一段 LaTeX 代码：

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
    4
    5
    6
    7
    8</code></pre></td>
    <td class="code"><pre><code>%!TEX program = xelatex
    \documentclass{article}
    \usepackage{fontspec, xunicode, xltxtra}
    \setmainfont{Hiragino Sans GB}
    \title{Test}
    \begin{document}
    Hello world!
    \end{document}</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    会发现报错：`CANNOT COMPILE`

    **解决方案**：通过 Package Control 安装一下 `Fix Mac Path` 这个包就能解决，安装步骤同 LaTex Tools。

安装完毕之后，我们就可以使用 `Command + B` 来对写好的 LaTeX 进行渲染，Skim 会弹出渲染后的窗口，并且在 Skim 窗口按住 `Command + Shift` 再点击，会自动跳转到 Sublime 中相应的部分。  
<span id="more"></span>

## <a href="#Structure" class="headerlink" title="Structure"></a>Structure

LaTeX 文章的结构由以下一些元素组成:

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
4
5
6</code></pre></td>
<td class="code"><pre><code>\section{}
\subsection{}
\subsubsection{}

\paragraph{}
\subparagraph{}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

在 `{}` 内书写章节或段落的题目。LaTeX 会自动为章节和段落编号。

### <a href="#Contents" class="headerlink" title="Contents"></a>Contents

LaTeX 可以通过 `\tableofcontents` 来根据文章整体的结构自动生成目录：

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
<td class="code"><pre><code>\begin{document}
\tableofcontents
\newpage
\maketitle{}
  \pagenumbering{gobble}
  \maketitle{}
    \newpage % for a summry
   \pagenumbering{arabic}
  \section{Restatement of the Problem}
    It&#39;s a problem restatements.
 \subsection{ problem 1}
 It&#39;s problem 1
   \subsection{ problem 2}
 It&#39;s problem 2 
  \section{Assumptions}
   \section {The Model}
    \section {Results}
  \section {Strengths and Weaknesses}
\end{document}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

编译之后产生的结果如下：

![Contens](/img/contents.png)

类似的，使用 `\listoftables` 和 `listoffigures` 可以产生关于表格和图片的一张目录。

目录的深度可以通过 `\setcounter{tocdepth}{depth}` 来设置：

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
4
5</code></pre></td>
<td class="code"><pre><code>\setcounter{tocdepth}{1} % Show sections
%\setcounter{tocdepth}{2} % + subsections
%\setcounter{tocdepth}{3} % + subsubsections
%\setcounter{tocdepth}{4} % + paragraphs
%\setcounter{tocdepth}{5} % + subparagraphs</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#Pagenumber" class="headerlink" title="Pagenumber"></a>Pagenumber

页码的设置也很简单: `\pagenumberring{property}`

property 有以下三种：

- gobble - no numbers
- *arabic* - arabic numbers 阿拉伯数字
- *roman* - roman numbers 罗马数字

## <a href="#Formula" class="headerlink" title="Formula"></a>Formula

公式也是很重要的一部分，LaTeX 对数学公式的支持真的是非常给力。

最简单的 `$$ F = ma $$`，就能够打出 $$ F = ma $$ ，独占一行。

行内公式只要使用单个美元符号即可 `$ formula in line $`。

常用希腊字母:`\alpha`, `\beta`,`\gamma` 等

使用 `\begin{equation}` + `\end{equation}` 在之间打公式，并且**会有序号标注**。

多个等式需要对齐，在导言部分通过 `usepackage{amsmath}` 导入 `amsmath` 包，通过该包中的 `align*` 可以实现：

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
4
5
6</code></pre></td>
<td class="code"><pre><code>\begin{align*}
 1 + 2 &amp;=3 \\ % 换行
 1 &amp;= 3 -2 \\ % 等式对齐 &amp; 控制需要对齐的符号
   f(x) &amp;= \frac{1}{x} \\ % frac{分子}{分母}
 g(x) &amp;= \int^a_b \frac{1}{4}x^4 % int^上限_下限
\end{align*}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

矩阵：

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
4
5
6</code></pre></td>
<td class="code"><pre><code>\left[
\begin{matrix}
    1 &amp; 0 \\
   8 &amp; 1
\end{matrix}
\right]</code></pre></td>
</tr>
</tbody>
</table>
</figure>

另外，如上面代码里的 `\left[` 和 `\right\`，可以显式的声明括号需要包括的内容，LaTeX 会对括号进行自动的放缩以包含所需的内容。

## <a href="#Images" class="headerlink" title="Images"></a>Images

插入图片:

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
4
5</code></pre></td>
<td class="code"><pre><code>\begin{figure}[h!] % [h!] 强制图片出现在代码所在位置 否则 LaTeX 会自动调整 
   \includegraphics[width=\linewidth] {boat.jpg}
   \caption{ A boat}
   \label {fig:boat1}
\end{figure}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

用 `\includegraphics[width=\linewidth]{IMAGE.jpg}` 来设置图片 `[width=\linewidth]` 控制高度

- h (here) - same location
- t (top) - top of page
- b (bottom) - bottom of page
- p (page) - on an extra page
- ! (override) - will force the specified location

### <a href="#Multi-Images" class="headerlink" title="Multi Images"></a>Multi Images

使用 Subfigure package 来设置多张图片

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
4
5
6</code></pre></td>
<td class="code"><pre><code>\usepackage{graphicx, subcaption}
\begin{figure}[h!]
    \begin{subfigure}[b]{0.2\linewidth}
   \includegraphics[width=\linewidth] [image.jpg]
    \end{subfigure}
\end{figure}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#Reference" class="headerlink" title="Reference"></a>Reference

Citation 是论文的重要组成部分，LaTeX 配合 Google Scholar 可以非常便捷的进行文献的引用，主要步骤如下：

1.  在 `.tex` 同一目录下建立同名 `.bib` 文件
2.  在 <a href="https://scholar.google.com" target="_blank" rel="noopener">Google Scholar</a> 搜索需要引用的文献，然后点击对应文献的 `"` 按钮，选择 `BibTex` 格式，将内容复制到 `.bib` 文件中。 e.g.

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
4
5
6
7</code></pre></td>
<td class="code"><pre><code>@inproceedings{mikolov2013distributed,
  title={Distributed representations of words and phrases and their compositionality},
  author={Mikolov, Tomas and Sutskever, Ilya and Chen, Kai and Corrado, Greg S and Dean, Jeff},
  booktitle={Advances in neural information processing systems},
  pages={3111--3119},
  year={2013}
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

1.  在 `.tex` 文件中 `\end{document}` 前输入：

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
<td class="code"><pre><code>% .... 
\bibliography{bib_name} % 输入 .bib 文件名
\bibliographystyle{ieeetr}  % referene style  引用格式
\end{document}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

1.  在需要引用的地方，使用 `\cite{reference_name}` 来进行引用 e.g. `\cite{mikolov2013distributed}`

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

LaTeX 用熟练了之后还是很方便的，写出来的文章也很漂亮。但这篇文章并不能覆盖所有 LaTeX 的内容，比如很重要的使用 `tikz` 包来进行绘图。很多东西记不住，也都是即查即用，学会 Google 才是上策。
