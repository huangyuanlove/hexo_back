---
title: LaTeX笔记-六(数学公式)
date: 2018-02-27 16:17:51
tags: [LaTeX]
mathjax: true
keywords: LaTeX数学公式
---
TeX有两种数学公式，一种是夹杂在行文段落中的公式，如$\int f(x) \text{d}x=1$,一般称为**行内(inline)**数学公式，或**正文(in-text)**数学公式;另一种就是像下面这样单独占据郑航居中展示出来，称为**显示**数学公式
$$\sum_{i=0}^N \int_a^bg(t,i)\text{d}$$

在TeX中，行内公式一般在前后单个美元符号\$...\$表示，显示公式用连续的两个美元符号\$\$...\$\$表示
<!--more-->
#### 基础
在数学模式下，符号会使用单独的字体，字母通常是倾斜的意大利体，数字和符号则是自立体，仔细看的话，数学符号之间的距离也与一般的水平模式不同：
$a + b = b + a$,如$ 1 + 2 = 2 + 1$
正常模式
a + b = b + a，如 1 + 2 = 2 + 1
因此，在排版数学公式时，即使是没有任何特殊符号的算式$1+1$也要进入数学模式，使用\$1+1\$而不是普通文字的1+1
c除了使用单个美元符号，在LaTeX中还额外定义了命令格式与环境格式的方式输入行内公式，即使用命令`\(`和`\)`环境括起来一个行内数学公式，如`$a+b$`也可以写成`\(a+b\)`或是`\begin{math}a+b\end{math}`。这两种形式提供了更好的错误检查，并且可以更明确地看出公式的开始于结束，也不容易混淆。
同样的，LaTeX中也定义了命令形式和环境形式的输入方法，即使用`\[`和`\]`命令或是`displaymath`环境括起一个显示数学公式，例如：`\[a+b=b+a\]`,虽然并非必须，但最好在源代码中就把单独占据一行的显示公式放在单独的行内，使代码更清晰，推荐使用的方式是`\[...\]`，`$$...$$`会产生不良的间距，缺少错误检查，并且不能正确处理fleqn等文档选项，应该避免使用，而`displaymath`环境可能显得冗长。
LaTeX还提供了带自动编号的数学公式，可以用`equation`环境表示，公式后还可以带引用的标签，例如：
``` tex 
\begin{equation}
  a+b=b+a \label{eq:commutative}
\end{equation}
```
#### 上标与下标
上标和下标是两种最常见的数学结构，它们的形式也很朴素：上标一般在原符号的右上方，下标一般在原符号的右下方，有时也在正上方和正下方，例如：

$$\sum_{i=1}^{n}\max_a10^na_i\int_Da^2_i$$

在TeX中，上标用特殊字符`^`表示，下标用特殊字符`_`表示。在数学模式中，符号`^`和`_`的用法差不多相当于带一个参数的命令，如**\$10^n\$**可以得到 $10^n$ ,而**\$a_i\$**可以得到 **$a_i$** 当上标和下标多余一个字符时，需要使用分组确定上下标范围，如**\$A_{ij}=2^{i+j}\$**得到**$A_{ij}=2^{i+j}$**
上标和下标可以同时使用，也可以嵌套使用。同时使用上标和下标，上下标的先后次序并不重要，二者互不影响，嵌套使用上下标时，则外层一定要使用分组。数学公式中空格是不起实际作用的，适当的空格可以将代码分隔得好看一些。
数学公式中单引号是一种特殊的上标，表示用符号`\prime`(即')做上标，可以与下标混用，也可以连续使用，但不能与上标直接混用，如：
```tex
$a=a'$,$b_0'=b_0''$,
${c'}^2=(c')^2$
```
得到
$a=a'$,$b_0'=b_0''$,${c'}^2=(c')^2$
#### 上下画线与花括号
`\overline` 和 `\underline`命令可用来在公式的上方和下方划横线，`overbrace` 和 `underbrace`命令可以在公式上方和下方带上花括号如：

|代码|结果|
|:----:|:----:|
|`$\overleftarrow{a+b}$`| $\overleftarrow{a+b}$|
|`$\overrightarrow{a+b}$` |$\overrightarrow{a+b}$|
|`$\overleftrightarrow{a+b}$`| $\overleftrightarrow{a+b}$|
|`$\underleftarrow{a-b}$` |$\underleftarrow{a-b}$|
|`$\underrightarrow{a-b}$` |$\underrightarrow{a-b}$|
|`$\underleftrightarrow{a-b}$`|$\underleftrightarrow{a-b}$|
|`$\vec x = \overrightarrow{AB}$`|$\vec x = \overrightarrow{AB}$|
|`$\overbrace{a+b+c} = \underbrace{1+2+3}$`|$\overbrace{a+b+c} = \underbrace{1+2+3}$|

还可以使用上下标在花括号上做标注
``` tex
  $$ (\overbrace{a_0,a_1,...,a_n}^{\text{共 $n+1$ 项}}) = (\underbrace{0,0,...,0}_n,1) $$
```
$$ (\overbrace{a_0,a_1,...,a_n}^{\text{共 $n+1$ 项}}) = (\underbrace{0,0,...,0}_n,1) $$
#### 分式
在LaTeX中分式用`\frace<分子><分母>`得到，如：
`$$ \frac 12 + \frac 1a = \frac{2+a}{2a} $$`  $ \frac 12 + \frac 1a = \frac{2+a}{2a} $
在行内公式和显示公式中，分式的大小是不同的。行内分式中分子分母都用较小的字号排版，以免超出文本行的高度。
连分式是一种特殊的分式，amsmath提供的`\cfrac`专用于输入连分式。这个命令可以带一个可选的参数l、c、r，表示左、中、右，默认是居中，如：
`$$ \cfrac{1}{1+\cfrac{2}{1+\cfrac{3}{1+x}}} = \cfrac[r]{1}{1+\cfrac{2}{1+\cfrac[l]{3}{1+x}}} $$`

得到  $$ \cfrac{1}{1+\cfrac{2}{1+\cfrac{3}{1+x}}} = \cfrac[r]{1}{1+\cfrac{2}{1+\cfrac[l]{3}{1+x}}} $$
 ![结果](/image/latex/latex_note_six_1.png),`markdown`对这个支持不是很好，结果用图片代替了。
还有一些类似分数分成上下两半,如二项式系数$\binom nk$，`amsmath`提供了`\binom`来输入二项式系数，其用法与`\frac`类似:
``` tex
`$$ (a+b)^2 = \binom {20}{02} a^2 + \binom 21 ab + \binom 22 b^2 $$`
```
$$ (a+b)^2 = \binom {20}{02} a^2 + \binom 21 ab + \binom 22 b^2 $$
#### 根式
根式在LaTeX用单参数的命令`\sqrt`得到，同时可以带一个可选参数，表示开方得次数，如：
`$\sqrt 4 = \sqrt[3]{8} = 2$` 得到 $\sqrt 4 = \sqrt[3]{8} = 2$
嵌套使用根式或与其他数学结构结合也很常见：
``` tex
$$ \sqrt[n]{\frac{x^2 + \sqrt 2}{x+y}}$$
```
$$ \sqrt[n]{\frac{x^2 + \sqrt 2}{x+y}}$$
如果开方得次数不是简单的整数，或者被开方得内容过长，通常改用等价的指数形式：
``` tex 
$$ (x^p+y^q)^{\frac{1}{1/p+1/q}}$$
```
$$ (x^p+y^q)^{\frac{1}{1/p+1/q}}$$
有时可能对开方次数的排版位置不满意，可以用`amsmath`提供的`\uproot`和`\leftroot`命令调整，命令参数是整数，移动的单位是很小的一段距离，如：
``` tex
$$ \sqrt[\uproot{16}\leftroot{-2}n] {\frac{x^2 + \sqrt 2}{x+y}} $$
```

#### 矩阵
在基本的LaTeX中，矩阵是用PlainTeX一样的命令`\matrix`和`\pmatrix`,各类矩阵环境的区别在于外面的括号不同：

|环境|代码|结果|
|:----:|:----:|:----:|
|matrix|`$$\begin{matrix} 1&0&0\\ 0&1&0\\ 0&0&1\\ \end{matrix}$$`|$$\begin{matrix} 1&0&0\\\\ 0&1&0\\\\ 0&0&1\\\\ \end{matrix}$$|
|bmatrix|`$$\begin{bmatrix} 1&0&0\\ 0&1&0\\ 0&0&1\\ \end{bmatrix}$$`|$$\begin{bmatrix} 1&0&0\\\\ 0&1&0\\\\ 0&0&1\\\\ \end{bmatrix}$$|
|vmatrix|`$$\begin{vmatrix} 1&0&0\\ 0&1&0\\ 0&0&1\\ \end{vmatrix}$$`|$$\begin{vmatrix} 1&0&0\\\\ 0&1&0\\\\ 0&0&1\\\\ \end{vmatrix}$$|
|pmatrix|`$$\begin{pmatrix} 1&0&0\\ 0&1&0\\ 0&0&1\\ \end{pmatrix}$$`|$$\begin{pmatrix} 1&0&0\\\\ 0&1&0\\\\ 0&0&1\\\\ \end{pmatrix}$$|
|Bmatrix|`$$\begin{Bmatrix} 1&0&0\\ 0&1&0\\ 0&0&1\\ \end{Bmatrix}$$`|$$\begin{Bmatrix} 1&0&0\\\\ 0&1&0\\\\ 0&0&1\\\\ \end{Bmatrix}$$|
|Vmatrix|`$$\begin{Vmatrix} 1&0&0\\ 0&1&0\\ 0&0&1\\ \end{Vmatrix}$$`|$$\begin{Vmatrix} 1&0&0\\\\ 0&1&0\\\\ 0&0&1\\\\ \end{Vmatrix}$$|

在矩阵环境中，不同的列用符号`&`分隔，行用`\\`分隔，矩阵中每列元素居中对齐，例如：

``` tex
$$\begin{Vmatrix} 
 A_1 & A_2 & A_3 \\ 
 B_1 & B_2 & B_3 \\ 
 C_1 & C_2 & C_3 \\ 
 \end{Vmatrix}$$
 ```

$$\begin{Vmatrix}  A_1&A_2&A_3\\\\  B_1&B_2&B_3\\\\  C_1&C_2&C_3\\\\  \end{Vmatrix}$$


在矩阵中经常使用各种省略号即`\dots`、`\vdots`、`\ddots`、`\iddots`,amsmath还提供了可以跨多列的省略号`\hdotsfor{<列数>}`，在行公式中，有时需要使用很小的矩阵，这可以由amsmath提供的`smallmatrix`环境得到.

-----
以上