---
title: LaTeX笔记-五(自动化工具)
date: 2018-02-27 11:09:28
tags: [LaTeX]
mathjax: true
math: true
keywords: LaTeX
---
记录一下某些自动化的工具，比如添加超链接(hyperref)、索引(makeindex)等
<!--more-->
#### 电子文档与超链接
`hyperref`宏包可算是LaTeX最为复杂的宏包之一，它提供了大量的选项和命令，完成各种设置和功能，这里主要记录以PDF格式输出时，`hyperref`有关标签和超链接的一些最基本的功能和设置。
`hyperref`最基本的用法非常简单，就是直接调用此宏包：
``` tex
\usepackage{hyperef}
```
如果使用ctex宏包或文档类，则可以加`hyperef`选项，这样啊ctex宏包会自动根据编码和编译方式选择合适的选项，避免出现乱码：
``` tex
\documentclass[hyperref,UTF8]{ctexart}
```
引入hyperref后再编译文档时，会根据章节结构，自动生成目录结构的PDF文档标签。同时，正文中的目录和所有交叉引用，都会自动成为超链接，可以用鼠标点击跳转到引用位置。要得到正确的PDF标签，也应至少表一两遍文档。
除了直接加在`\usepackage`，也可以使用`\hypersetup`命令单独设置。hyperref的选项大多使用<选项>=<值>的方式设置，如果是布尔类型的真假值选项(true或false),通常可以省略为真的值，常见选项如下：

|选项|类型|默认值|说明|
|:---|:---:|:---:|:---:|
|colorlinks|布尔|false|超链接用彩色显示|
|bookmarks|布尔|true|生成PDF目录书签|
|bookmarksopen|布尔|false|在PDF阅读器中自动打开书签|
|bookmarksnumbered|布尔|false|目录书签带编号|
|pdfborder|数 数 数|0 0 1|当colorlink为假时，超链接由彩色边框包围(不会被打印)。默认值表示1pt宽的边框，可以设置为0 0 0 表示没有边框。
|pdfpagemode|文本||在PDF阅读器中的页面显示方式，常用值是FullScreen，表示全屏显示|
|pdfstartview|文本|Fit|在PDF阅读器中的页面缩放大小，默认值Fit表示“适合页面”；常用取值有适合宽度FitH，适合高度FitV|
|pdftitle|文本||文档标题，会在PDF文档属性中显示|
|pdfauthor|文本||文档作者，会在PDF文档属性中显示|
|pdfsubject|文本||文档主题，会在PDF文档属性中显示|
|pdfkeywords|文本||文档关键字，会在PDF文档属性中显示|
除此之外，可以用`\url`命令输出URL地址，同时也具有超链接的功能。与排版纯文本不同，找`\url`命令的参数中，网址允许使用合法符号直接输入，并且默认以打字机字体输出，如果URL地址不需要超链接的效果，可以改用`\nolinkurl`命令。
`\href{<URL>}{文字}`命令可以用来使文字产生指向URL地址的超链接效果。
`\hyperref[<标签>]{<文字>}`命令可以用来产生使文字执行标签的超链接效果，这里方括号中的标签与`\ref`使用的标签相同，不能省略。
`\hypertarget{<名称>}{<文字>}`用来给文字定义带有名称的链接点，在文档的其他地方，则可以使用命令`\hyperlink{<名称>}{<文字>}`让另一段文字链接到指定名称的连接点。

#### 制作索引
在LaTeX中制作索引，需要.tex源文件和外部索引程序的共同协作。在.tex源文件中，我们需要做一下几件事：
1. 在导言区使用`\makeindex`命令，开启索引文件输出
2. 在导言区调用`makeidx`宏包，开启索引列表排版功能
3. 在正文中需要索引的关键字处使用`\index`命令，生成索引项
4. 在需要生成索引的地方(通常是文档的末尾)，使用`\printindex`命令，实际输出处理好的索引列表

例子如下：
``` tex
\documentclass[hyperref,UTF8]{ctexart}
\usepackage{makeidx}
\makeindex
% ...
西方称勾股定理为毕达哥拉斯定理，
\index{毕达哥拉斯定理}
将勾股定理的发现归功于公元前 6 
%...
勾股定理可以用现代语言表述如下\index{勾股定理}：
%...
\printindex
\end{document}
```
与参考文献类似，要生成索引需要多次编译和外部工具`Makeindex`的配合，编译带索引的文档需要使用如下命令：
``` tex
pdflatex ***
makeindex ***
pdflatex ***
```
以上命令可以在IDE的编译选项中找到
在`\index`的参数中可以使用符号`!`来分隔不同层次的索引项，这样将得到分级的索引项。默认支持三级列表，例如：
```tex
\index{language}
\index{language!Chinese}
\index{language!Chinese!dialect}
\index{language!English}
```
在`\index`的参数后面使用符号`|`则可以使用几个特殊的功能，基本的用法是使用`|see{<条目>}`与`|seealse{条目}`表示参考条目。例如：
``` tex
\index{beta}
\index{Beta|see{beta}}
\index{gamma|seealso{beta}}
```
----
以上