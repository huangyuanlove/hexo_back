---
title: LaTeX笔记(四)--文档结构层次
date: 2018-02-11 17:59:21
tags: [LaTeX]
mathjax: true
math: true
keywords: LaTeX
---
下面是介绍文档的层次结构
<!-- more -->
#### 标题和标题页
在LaTeX中，使用标题通常分为两个部分：声明标题内容和实际输出标题。每个标题则由标题、作者、日期等部分组成。
声明标题、作者和日期分别使用`\title`，`\author`，`\date`命令。它们都带有一个参数，里面可以使用`\\`进行换行。标题的声明通常放在导言区，也可以放在标题输出之前的任何位置：
``` tex
\documentclass[UTF8,titlepage]{ctexart}
\title {语言\thanks{本文由XX基金会赞助}}
\author{huangyuan\thanks{sdut}\\sdut \and xuan\thanks{sdut}\\sdut}
\date{戊戌初春}
```
`\author`定义的参数可以分行，一般第一行是作者姓名，后面是作者的单位、联系方式等，如果文档有多个作者，则多个作者之间用`\and`分隔。
在声明标题和作者时，可以使用`\thanks`命令产生一种特殊的脚注，它默认使用特殊符号和编号，通常用来表示文章的致谢、文档的版本、作者的详细信息等。
使用`\maketitle`命令可以输出前面声明的标题，通常`\maketitle`是文档中document环境后面的第一个命令。整个标题的格式是预设好的，在`article`或`ctexart`中，标题不单独成页，可以使用文档类的选项`titlepage`和`notitlepage`来设置标题是否单独成页。
#### 划分章节
LaTeX的标准文档类可以划分多层章节，可以使用6到7个层次的章节。

|层次|名称|命令|说明|
|:---:|:---:|:---:|:---:|
|-1|part(部分)|`\part`|可选的最高层|
|0|chapter(章)|`\chapter`|report,book或ctexrep,ctexbook文档类的最高层|
|1|section(节)|`\section`|article或ctexart类的最高层|
|2|subsection(小节)|`\subsection`||
|3|subsubsection(小小节)|`\subsubsection`|report,book或ctexrep,ctexbook文档类默认不编号，不编目录|
|4|paragraph(段)|`\paragraph`|默认不编号，不编目录|
|5|subparagraph(小段)|`\subparagraph`|默认不编号，不编目录|
一个文档的最高层章节可以是`\part`，也可以不用`\part`直接使用`\chapter`(对book和report等)或`\section`(对article)。除`\part`外，只有在上一层章节存在时才能使用下一章节，否则编号会出现错误。在`\part`下面，`\chapter`或`\section`是连续编号的；在其他情况下，下一级的章节随上一节的编号增加会清零重新编号。可以使用带星号的章节命令(如`\chapter*`)来表示不编号、不编目录的章节。
#### 多文件编译
对于一篇只有几页的文章，把所有的内容都放进一个tex源文件就足够了，但是如果要排版更长的内容，单一文件编译的方式就不那么方便可，可以按照文档的逻辑层次，把整个文档分成多个tex源文件，这样文档的内容更便于检索和管理，也适合大型文档的多人协同编写。
LaTeX提供`\include{<文件名>}`命令可以用来导入两一个文件的内容作为一个章节，文件名不用带`.tex`扩展名，`\include`命令会在之前和之后使用`\clearpage`或`\cleardoublepage`另起新页，同时将这个文件的内容贴到`\include`命令所在的文字。所以我们可以这样来组织一篇较长的文章：
``` tex
\documentclass[UTF8,titlepage]{ctexart}
\title {语言}
\author{huangyuan\thanks{sdut}\\SDUT \and xuan\thanks{sdut}\\SDUT}
\date{戊戌初春}
\begin{document}
\maketitle
\tableofcontents
\include{lang-natural}
\include{lang-computer}
\end{document}
```

``` tex
%lang-natural.tex 不能单独编译
\chapter{自然语言}
这是自然语言章节
```

``` tex
% lang-computer.tex 不能单独编译
\chapter{计算机语言}
这是计算机语言章节
```
划分文档后，可以通过主文件来控制编译整个文档的一章或者某几章。当然可以把不要的章节注释掉，更好的办法是通过`\includeonly{<文件列表>}`命令，其中<文件列表>是用英文都好隔开的若干文件名。在导言区使用`\includeonly`命令以后，只有在文件列表中的文件才会被实际的引入主文件。更好的是，如果以前曾经完整的编译过整个文档，那么在使用`\includeonly`选择编译时，原来的章节编号、页码、交叉引用等仍然会保留为前一次编译的效果：
``` tex
\documentclass[UTF8,titlepage]{ctexart}
\title {语言}
\author{huangyuan\thanks{sdut}\\SDUT \and xuan\thanks{sdut}\\SDUT}
\includeonly{lang-natural}
\date{戊戌初春}
\begin{document}
\maketitle
\tableofcontents
\include{lang-natural}
\include{lang-computer}
\end{document}
```
值得注意的是，在使用`\include`命令时，最好不要在子文件中新定义计数器、声明新字体，否则在使用`\includeonly`时，会因为找不到出现在辅助文件中而在源文件中缺失的计数器而出错。
比`\include`命令更一般的是`\input`命令，它直接把文件的内容复制到`\input`命令所在的文字，不做其他多余的操作。`\input`命令接受一个文件名参数，文件名可以带扩展名，也可以不带扩展名(此时认为扩展名是.tex)。
一般可以把导言区、复杂图标代码放在一个单独文件中，然后在主文件中使用`\input`插入。在**被引入的文件**末尾，可以使用`\endinput`命令显式的结束文件的读入，在`\endinput`命令的后面，就可以直接写一些注释性的文字，而不必再加入注释符号。

----
以上