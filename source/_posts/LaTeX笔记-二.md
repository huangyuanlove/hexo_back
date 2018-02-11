---
title: LaTeX笔记(二)
date: 2018-01-27 22:59:42
tags: [LaTeX]
---
接上篇，写好的tex文件在这 https://github.com/huangyuanlove/latex_practice/blob/master/latex(1)/chapter_one.tex
##### 写个框架
既然有目标了，我们先写个大概的框架，然后往里面填充内容
``` tex 
% coding:UTF-8


\documentclass[UTF8]{ctexart}

\title{杂谈勾股定理}
\author{xuan}
\date{\today}

\bibliographystyle{plain}

\begin{document}

\maketitle
\tableofcontents
\section{勾股定理在古代}
\section{勾股定理的近代形式}
\bibliography{math}

\end{document}
```
<!--more-->
* 以 `%`开头的行是注释，源文件一行中百分号后面的内容都会被忽略。
* 第**4**行是文档类，因为是中文的短文，所以用的是`ctexart`，并用[UTF8]选项说明编码方式。
* 第**6-8**行声明了文章的标题、作者、日期。其中的`\today`是当前日期。但是这些信息是通过**14**行的`\maketitle`排版的。
* 第**10**行的`\bibliographystyle`声明参考文献的格式。
以上在`\begin{document}`之前的部分称为`导言区(preamble)`，导言区通常用来对文档的性质做一些设置，或自定义一些命令。
* 第**12-20**行以`\begin{document}`和`\end{socument}`声明了一个document环境，里面是论文的正文部分，也就是直接输出的部分。
* 第**14**行的`\maketitle`输出论文标题。
* 第**15**行的`\tableofcontents`输出目录。
* 第**16-17**行两个`\section`开始新的一节。
* 第**18**行的`\bibliography{math}`则是从文献数据库math(这里的math文献数据库是我们自己编辑的一个文件)中获取文献信息，打印参考文献列表。
> 为了格式清晰，源文件使用了一些空行作为分隔，在正文外的部分，空行不代表任何意义

##### 填写正文
自己把大段的文字先写上再说，自己试试查看排版之后格式。
* 使用空行分段，
* 段落的第一行缩进不需要自己打空格
##### 命令与环境
###### 脚注
脚注是在正文**欧几里得**的后面用脚注命令`\footnote`得到的。
``` tex
该定理的严格表述和证明则见于欧几里得
\footnote{欧几里得，约公元前330--275年。}
```
在这里，`\footnote`后面花括号内的部分是命令的参数，也就是脚注的内容。
注意一个细节，在表示起止年份时，用两个减号(--)，通常用来表示数字的范围。
文中还使用`\emph`命令改变字体的形状，表示强调的内容：
``` tex
的整数称为\emph{勾股数}
```
一个LaTeX命令(宏)的格式为：
无参数：`\command`
有n个参数:`\command<arg1><arg2><arg3>`
有可选参数:`\command<arg——opt><arg2><arg3>`
命令都一反斜线`\`开头，后接命令名，可以带一些参数，必选参数使用花括号括起来，可选参数使用方括号括起来。
引用的内容这是在正文中使用`quote`环境得到的。
``` tex
我国《周髀算经》载商高（约公元前12世纪）答周公问：
\begin{quote}
勾广三，股修四，经隅五。
\end{quote}
```
quote环境即以`\begin{quote}`和`\end{quote}`为起止位置的部分，突出引用部分。但是quote环境不能改变引用内容的字体，因此还需要再使用改变字体的命令
``` tex
\begin{quote}
\kaishu\zihao{-5} 引用的内容
\end{quote}
```
这里`\zihao`是一个有参数的命令，选择字号(-5就是小五号)，而`\kaishu`则是没有参数的命令，把字体切换为楷书，注意用空格把命令和后面的文字分开。
文章的摘要也是在`\maketitle`之后用`abstract`环境生成的：
``` tex
\begin{abstract}
刚开始学\LaTeX,如果学习不是为了装逼，那一切将毫无意义。
\end{abstract}
```
当然我们也可以自定义环境，比如上面的突出引用的环境：
在导言区`\maketitle`之后
``` tex
\newenvironment{myquote}{\begin{quote}\kaishu\zihao{-5}}{\end{quote}}
```
之后我们在突出引用的时候就可以这么写：
``` tex
我国《周髀算经》载商高（约公元前12世纪）答周公问：
\begin{myquote}
勾广三，股修四，经隅五。
\end{myquote}
```
文章第二节的定理，是用一类定理环境输出的，定理环境是一类环境，在使用前需要先在导言区做定义：
``` tex
\newtheorem{thm}{定理}
```
这就定义了一个`thm`环境，定理环境可以有一个可选参数，就是定理的名字，所以文章中的勾股定义就可以由新定义的`thm`环境得到：
``` tex
\begin{thm}[勾股定理]
直角三角形斜边的平方等于两腰的平方和。
\end{thm}
```
###### 数学公式
最简单的方式就是把公式用`$`符号括起来，比如`$a+b$`就可以得到斜体的*a+b*，这种夹在行文中的公式称为"正文公式(in-text formula)"或"行内公式(inline formula)"。为了方便引用，经常会给公式编号，这种公式被称为"显式公式"或"列表公式(display formula)",使用`equation`环境就可以方便地输入这种公式。
``` tex
\begin{equation}
a(a+b) = ab + ac
\end{equation}
```
 键盘上没有的符号，就需要使用命令来输入，"角"符号"∠",就可以用`\angle`输入(虽然有的输入法也提供了数学符号的选项，但是不建议这么做，可以自己试试有什么区别)。
 数学公式还有上线标、分式、根式等，在勾股定理的表达中，就用到了上标表示乘方：
``` tex
 \begin{equation}
 AB^2 = BC^2 + AC^2
 \end{equation}
```
 符号`^`用来引入一个上标，`_`医用一个下标，如果上下标识多个字符，则需要使用花括号分组：`2^{10}`。
 所以90°怎么输入，在latex默认的字体中，并没有专用于表示角度的符号，输入角度的时候是通过上标输入的:`^\circ`，其中`\circ`通常用来表示函数符号的二元运算符`○`，我们把它的上标借用来表示角度。

 ###### 插入图片
 使用图片有两种途径：一个是插入事先准备好的图片，二是使用latex代码直接在文档中画图。
 插图功能不是由latex内核直接提供，而是有**graphicx**宏包提供的，要使用**graphicx**宏包的插图功能呢个，需要在导言区使用`\usepackage`命令引入宏包：
 ``` tex
 \documentclass{ctexart}
 \usepackage{graphicx}
 ```
 引入**graphicx**宏包后，就可以使用`\includegraphics`命令插图了：
 ``` tex
 \includegraphic[width=3cm]{gougu.png}
 ```

这里`\includegraphic`有两个参数，方括号中可选参数`width=3cm`设置图形在文档中显示的宽度为3cm，第二个参数则是图片的文件名(和源文件同级)，还有一些类似的参数如scale(缩放)、height等。
除了一些很小的图标，我们很少进行图文混排，而是使用单独的环境列出，而且很大的图形如果位置是固定，会给分页造成困难，因此，通常都把图像放在一个可以变动相对位置的环境中，称为浮动体。在浮动体中还可以给图片加入说明性标题。
``` tex
\begin{figure}[ht]
  \centering
  \includegraphics[width=3cm]{gougu.png}
  \caption {\kaishu\zihao{-5} 宋赵爽在《周髀算经》注中做的弦图（仿制），该图给出了勾股定理的一个极具对称美德证明。}
  \label{fig:gougu}
\end{figure}
```
在上面的代码中，第1行和第6行使用了`figure`环境，就是插图使用的浮动体环境，`figure`环境有可选参数`[ht]`，表示浮动体可以出现在环境周围的文本所在处和一页的顶部。`figure`环境内部相当于普通的段落(默认没有缩进)；第二行用生命`\centering`表示后面的内容居中；第3行插入图片；第4行使用`\caption`命令给插图加上自动编号和标题；第5行的`\lable`命令则给图形定义一个标签，这个标签就可以在文章的其他地方引用`\caption`产生的编号。
###### 使用表格
制作表格，需要确定是表格的行、列、对齐模式和表格线，这是由`tabular`环境完成的：
``` tex
\begin{table}[h]
\begin{tabular}{|lcr|}
\hline
直角边 $a$ & 直角边 $b$ & 斜边 $c$\\
\hline
3 &  4 & 5\\
5 &  12&  13\\
\hline
\end{tabular}%
\qquad
($a^2 + b^2 = c^2$)
\end{table}
```
表格和插图一样，一般也放在浮动环境中，即`table`环境中，参数大致和`figure`差不多；
`tabular`环境有一个参数，里面声明了表格中列的模式，在上面的表格中`|lcr|`表示第一列的内容左对齐，第二类居中对齐，第三列居中对齐，在第一列前面和第三列后面各有一条垂直的表格线。
在`tabular`环境中，行与行之间用命令`\\`隔开，每一行内的表项使用符号`$`隔开，表格中的横线是用命令`\hline`生成。
这里并没有给表格加标题，也没有把内容居中，而是把表格个一个公式并排排开，中间使用一个`\qquad`分割，这个命令产生长为2em(大约两个'M'的宽度)的空白。因为已经使用`\qquad`生成足够长度的空格了，所以再用`\end{tabular}`后的注释符取消换行产生的一个多余空格。又因为表格是和正文连在一起的，不允许再浮动了，所以在`table`环境中的表示位置参数处使用了`[H]`，但是这个参数是由**float**宏包提供的，所以还要在导言区使用`\usepackage{float}`。

###### 文献引用
上面提到了引用文献，是在`math.bib`这个文件中指定的，下面是`math.bib`文件的内容
``` bib
% Encoding: UTF-8

@Book{Shiye,
  title  = {几何的有名定理},
  year   = {1986},
  author = {矢野健太郎},
}

@Book{Kline,
  title  = {古今数学思想},
  year   = {2002},
  author = {克莱因},
}

@Book{quanjing,
  title   = {商高、赵爽与刘徽关于勾股定理的证明},
  year    = {1998},
  author  = {曲安京},
  volume  = {20},
  number  = {3},
  journal = {数学传播},
}

@Comment{jabref-meta: databaseType:bibtex;}
```
一个文献数据文件的格式并不复杂，每一个条目包括类型、引用标签、标题、年限、作者等信息，可以手工输入，也可以通过jabref制作。
在实际应用中，BIBTEX数据并不需要我们自己录入，可以从相关的学科网站直接现在或是从其他类型的文献数据库转换得到。
BIBTEX是一个专门用于处理LATEX文档文献列表的程序，使用BIBTEX处理文献时，编译源文件需要增加为四次运行程序(在TexWorks中点击四次按钮)
`pdflatex ***.tex`
`bibtex ***.aux`
`pdflatex ***.tex`
`pdflatex ***.tex`
第一次运行为BIBTEX准备好辅助文件，确定数据库中的哪些文献将被列出来，然后bibtex处理辅助文件aux，从文献数据库中选取文件，按指定的格式生成文献列表的latex代码，后面两次再读入文献列表代码并生成正确的引用信息。
latex只选择被引用的文献，引用文献的方法是在正文中使用`\cite`命令，如：
``` tex
将勾股定理的发现归功于公元前 6 世纪的毕达哥拉斯学派\cite{Kline}。该学派得到了一个法则...
...是我国古代对勾股定理的一种证明\cite{quanjing}。 
```
`\cite`命令的参数`Kline`和`quanjing`分别是其中两篇的引用标签，也就是在`math.bib`中每个条目第一行出现的星系，使用`\cite`命令会在引用的文字显示文献在列表中的标号(它在第3次pdflatex编译后才能确定)，同时在辅助文件中说明某文献将被引用。如果要在列表中显示并不直接引用的文献，可以使用`\nocite`命令，一般是把它放在`\bibliography`之前：
``` tex
\nocite{Shiye}
\bibliography{math}
```
目录也是自动从章节命令中提取并写入目录文件中的，在文章中我们就使用了`\tableofcontents`命令，它将在第二次编译时生效。
引用并不仅限于参考文献，图表、公式的编号，只要事先设定了标签，同样可以引用。基本的交叉引用命令是`\ref`，它以标签问参数，得到被引用标号。比如在这篇文章中，在插入图片时是引用`\label`命令为弦图定义了标签`\label{fig:gougu}`，这样在正文中就可以使用
``` tex
图\ref{fig:gougu}是我国古代对勾股定理的一种证明\cite{quanjing}。
```
公式编号的引用也可照此做法，不过需要先在公式中定义标签：
``` tex
\begin{equation} \label{eq:gougu}
AB^2 = BC^2 + AC^2
\end{equation}
```
引用数学公式的时候一般使用数学宏包`amsmath`就定义了`\eqref`命令，专门用于公式的引用，并能产生括号：
导言区引入包
```tex
\usepackage{amsmath}
```
正文中引用公式：
``` tex
满足式\eqref{eq:gougu}的整数称为\emph{勾股数}。
```

----
以上。