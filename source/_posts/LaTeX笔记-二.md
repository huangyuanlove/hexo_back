---
title: LaTeX笔记-二
date: 2018-01-27 22:59:42
tags: [LaTex]
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
>该定理的严格表述和证明则见于欧几里得
\footnote{欧几里得，约公元前330--275年。}

在这里，`\footnote`后面花括号内的部分是命令的参数，也就是脚注的内容。
注意一个细节，在表示起止年份时，用两个减号(--)，通常用来表示数字的范围。
文中还使用`\emph`命令改变字体的形状，表示强调的内容：
>的整数称为\emph{勾股数}

一个LaTeX命令(宏)的格式为：
无参数：`\command`
有n个参数:`\command<arg1><arg2><arg3>`
有可选参数:`\command<arg——opt><arg2><arg3>`
命令都一反斜线`\`开头，后接命令名，可以带一些参数，必选参数使用花括号括起来，可选参数使用方括号括起来。
引用的内容这是在正文中使用`quote`环境得到的。
>我国《周髀算经》载商高（约公元前12世纪）答周公问：
\begin{quote}
勾广三，股修四，经隅五。
\end{quote}

quote环境即以`\begin{quote}`和`\end{quote}`为起止位置的部分，突出引用部分。但是quote环境不能改变引用内容的字体，因此还需要再使用改变字体的命令
>\begin{quote}
\kaishu\zihao{-5} 引用的内容
\end{quote}

这里`\zihao`是一个有参数的命令，选择字号(-5就是小五号)，而`\kaishu`则是没有参数的命令，把字体切换为楷书，注意用空格把命令和后面的文字分开。
文章的摘要也是在`\maketitle`之后用`abstract`环境生成的：
>\begin{abstract}
刚开始学\LaTeX,如果学习不是为了装逼，那一切将毫无意义。
\end{abstract}

当然我们也可以自定义环境，比如上面的突出引用的环境：
在导言区`\maketitle`之后
>\newenvironment{myquote}{\begin{quote}\kaishu\zihao{-5}}{\end{quote}}

之后我们在突出引用的时候就可以这么写：
>我国《周髀算经》载商高（约公元前12世纪）答周公问：
\begin{myquote}
勾广三，股修四，经隅五。
\end{myquote}

文章第二节的定理，是用一类定理环境输出的，定理环境是一类环境，在使用前需要先在导言区做定义：
>\newtheorem{thm}{定理}

这就定义了一个`thm`环境，定理环境可以有一个可选参数，就是定理的名字，所以文章中的勾股定义就可以由新定义的`thm`环境得到：
>   \begin{thm}[勾股定理]
直角三角形斜边的平方等于两腰的平方和。
\end{thm}

###### 数学公式
