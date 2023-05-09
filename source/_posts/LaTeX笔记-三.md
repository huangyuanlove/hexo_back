---
title: LaTeX笔记(三)(杂)
date: 2018-02-11 16:25:02
tags: [LaTeX]
mathjax: true
math: true
keywords: LaTeX
---
前两篇笔记记录下了写一篇小文章所需要的东西，这一片就记录下一些零零散散的东西，计算机专业的嘛，只关心中文和英文，像什么德语、法语之类的东西就不在考虑范围内了，用的时候再去查也是可以的。
<!--more-->
##### 标点
* 在LaTeX中遇到单引号与双引号连续出现的情形，在中间使用`\,`命令分开：
``` tex
``\,`A' or `B'\,'' he asked.
``It's Knuth's book。'', he said。
```
这里`\,`命令产生很小的间距，但是LaTex并不会忽略以符号命名的宏前后的空格，所以在它的前后都不要加多余的空格。符号`'`同时也是表示所有格和省字的撇好。
* 除了在数学模式中表示减号，符号`-`在LaTeX正文中也有多种用途：单独使用它是连字符(hyphen);两个连用`--`是en dash,用来表示数字范围;三个连用`---`是 em dash，即破折号。不过在中文书写中，表示数学范围也常使用符号`~`(数学模式的符号$\sim$)。
* 西文的省略号(ellipsis)使用`\ldots`或`\dots`命令产生，相比直接输入三个句号，拉开的间距要合理的多。
#### 水平间距
在正文中可以使用下面的命令表示不可换行的水平间距
不可换行的水平间距

|命令|间距|
|:----:|:----:|
|`\thinspace` 或`\,` | 0.1667em|
|`\negthinspace` 或`\!` | -0.1667em|
|`\enspace` | 0.5em|
|`\nobreakspace` 或`~` | 空格|

可换行的水平间距

|命令|间距|
|:----:|:----:|
|`\quad` | 1em|
|`\qquad` 或`\!` | 2em|
|`\enskip` | 0.5em|
|`\空格` | 空格|

当上面的命令中没有合适的距离时，可以用`\hspace{距离}`命令来产生指定的水平间距(这里的`\,`用来分隔数字和单位)
``` tex
Space\hspace{1cm}1\,cm
```
`\hspace`命令产生的距离是可断行的，但是在某些情况下(强制断行的行首)，改命令产生的距离会被忽略，此时可以用带星号的命令`\hspace*{距离}`阻止距离被忽略。
#### 盒子
盒子(box)是TeX中的基本处理单位。
最简单的命令是`\mbox{内容}`,它产生一个格子，内容以左右模式排列，可以用它表示不允许断行的内容。
`\makebox`与`\mbox`类似，但可以带两个可选参数，指定盒子的宽度和对齐方式：
`\makebox[<宽度>][<位置>]{<内容>}`
对齐参数可以取c(居中)、l、r、s(分散),默认居中。
`\fbox` 和 `\framebox`产生带边框的盒子，语法和`\mbox`、`\makebox`类似
#### 列表环境
列表是常用的文本格式，LaTeX标准文档类提供了三种列表环境：带编号的`enumerate`环境、不编号的`itemize`环境和使用关键字的`description`环境。在列表内部使用`\item`命令开始一个列表项，它可以带一个可选参数表示手动编号或关键字
``` tex
\begin{enumerate}
\item 中文
\item English
\end{enumerate}

\begin{itemize}
\item 中文
\item English
\end{itemize}
```
description环境总是使用`\item`命令的可选参数，把它作为条目的关键字加粗。
``` tex
\begin{description}
\item[中文] 中文
\item[英文] English
\end{description}
```
上面三种列表环境可以嵌套使用(最多四层)
#### 抄录和代码环境
`\verb`命令可用来表示杭温中的抄录，语法格式如下：
`\verb<符号><抄录内容><符号>`
在`\verb`后，两个符号相同，表示起始符号和末尾符号，两个符号之间的内容会原样输出。
使用带星号的命令`\verb*`则可以使输出的空格为可见的。
大段的抄录可以使用`verbatim`环境：
``` tex
\begin{verbatim}
#! user/bin /env perl
$name = ''guy'';
print ''Hello,$name!\n''
\end{verbatim}
```
同样可以使用带星号的`verbatim*`环境输出可见空格。
如果想在程序代码中增加语法高亮功能，可以使用**listings**宏包
``` tex
\begin{lstlisting}[language=java]
class Test{
	public static void main(String ... args){
		System.out.println(''Hello Java'');
	}
}
\end{lstlisting}
```
----

以上
