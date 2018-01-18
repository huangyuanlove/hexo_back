---
title: LaTeX笔记(一)
date: 2018-01-18 21:21:41
tags: [LaTex]
---
如果学习不是为了装逼，那一切都将毫无意义。
没错，我又想学**LaTeX**了，作为世界上最好的排版系统(没有之一)，我也只是在写简历的时候用过(装上编译器，改模板而已)，没有怎么了解过。最近妹子有提到过说些毕业论文的时候可能要用，这就需要系统的学习一下了。
#### 准备阶段
本着实用至上的原则，LaTeX的历史以及光辉事迹就不在提了，想看的自己去搜
1. **TexLive2017套装**：下载地址http://mirrors.tuna.tsinghua.edu.cn/CTAN/systems/texlive/Images/texlive2017-20170524.iso 这个是清华镜像站点。http://mirror.ctan.org/systems/texlive/Images/texlive2017.iso 这个是官方的镜像下载地址。
不推荐使用**CTex**
2. **TexWorks**：Tex文件编辑器，你也可以使用其他编辑器+插件：比如 VisualStudioCode，Sublime，Notepad++等。
3. **JabRef**：下载地址 https://www.fosshub.com/JabRef.html 大部分用在写论文的引用文献上，只有使用LaTeX撰写科技论文的研究人员才能完全领略到JabRef的妙不可言。
4. **学习资料**：LaTeX实在太庞大了，加上各种package，网上各种博客资料不是很全面，只照顾到一部分，推荐*刘海洋*的*LaTeX入门*。
5. **建议**：多练习，就像学编程一样，把书上的例子都自己敲一遍。

<!--more-->
#### 安装软件
将`TexLive2017套装`加载到光驱，
* Windows 用户双击 install-tl-advanced.bat；
* \*nix 用户执行 install-tl。
然后按照提示来安装就好了，最后将`texlive\2017\bin\win32`加入环境变量，*nix用户自己找执行文件所在的路径加到环境变量里面。
其他的东西可以参考这个网站 http://www.latexstudio.net/archives/10208

#### Hello World
老规矩，先跑个`Hello World`。
1. 打开`TexWork`,在编辑区输入以下代码：
``` Tex
\documentclass{article}

\begin{document}

Hello World!

\LaTeX

\end{document}
```
然后保存一下文件，注意一定要使用UTF-8编码(默认)，Tex文件的后缀名为`.tex`，编译过程会多出好多临时文件。
2. 在下图红框处(界面左上角)选择`pafLaTeX`,然后点击绿色按钮进行编译，编译过程中绿色按钮会变成红叉，编译成功后又变成绿色按钮。你会在弹出的窗口看到"Hello World LaTeX"，并且在`tex`文件同级的文件夹下看到生成的`pdf`文件。
如果没有编译成功，可以尝试选择不同的编译类型，也可以搜索控制台报错信息来解决。
![TexWorks界面](/image/latex/latex_note_one_1.png)
3. 如果你觉得编辑窗口的字体看着不舒服，可以在`编辑`-->`首选项`-->`编辑器`或者`格式`-->`字体`里面调整。
4. 多说一句，`TexWorks`支持代码补全

#### 说明
`\documentclass[UTF-8]{article}` 声明文档类型是一篇文章。
`\begin{document}` 和 `\end{document}` 标识出正文的范围
`\LaTeX` 看结果就知道是表示结果中高低不平的LaTeX
但是，当你把内容替换为汉字的时候，发现汉字并不能在pdf文件中展示，这是因为LaTeX原本是面向西文写作的，默认没有加载中文字体。我们可以通过替换文档类型来显示中文，将 `\documentclass{article}` 替换为 `\documentclass[UTF-8]{ctexart}` 就可以显示中文了。

#### 下一个目标
如下图，接下来我们来慢慢写出来图片所示的样式。
![目标](/image/latex/latex_note_one_2.png)

----
以上。