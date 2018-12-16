---
title: UNIX/linux的find命令细节
tags: [linux]
date: 2015-2-11 23:40:31
keywords: [linux find]
---

find 命令的工作方式如下:

沿着文件层次结构向下遍历,匹配符合条件的文件,并执行相应的操作.

<!--more-->

#### 根据文件名或者正则表达式匹配搜索

选项-name的参数指定了文件名所必须匹配的字符串,我们可以将通配符作为参数使用."*.txt" 可以匹配所有的以".txt"结尾的文件.选项-print 在终端中打印出符合条件的文件名或者文件路径,这些匹配条件作为find 的参数给出

find /home/huangyuan -name "*.txt" -print

find 有一个选项-iname 可以忽略文件名的大小写

如果想匹配多个条件中的一个,可以使用 OR 条件操作

find . \(-name "*.txt" -o -name "*.pdf"\) -print

上面的代码会打印出所有的 ".txt" 和 ".pdf" 文件,是因为这个find 命令可以匹配所有的这两类文件

其中, "\(" 和 "\)" 用于将 -name "*.txt" -o -name "*.pdf" 视为一个整体

选项 -path 可以使用通配符来匹配文件路径或者文件名

选项 -regex 是基于正则表达式来匹配文件路径货文件.正则表达式是通配符的高级形式,例如,可以使用正则表达式来匹配Email ,一个 Email 的形式通常是 name@host.root

所以可以将其一般化为 [0-9a-zA-A]+@[0-9a-zA-A]+.[0-9a-zA-A] +

 "+" 指明在它之前的字符类中字符可以出现一次或者多次



#### 否定参数 !

find 也可以用 "!" 来否定参数的含义

find . ! -name "*.txt" -print

上面的命令可以用来 匹配所有不以 ".txt" 结尾的文件



#### 基于目录深度的搜索

find 命令在使用时会遍历所有的子目录,我们可以使用 -mindepth 和 -maxdepth 来限制搜索子目录的深度

find . -maxdeoth 1 -print 

这条目录只会打印当前目录中的所有文件,而不会打印子目录中的文件

find . -mindepth 2 -print

这条目录打印目录深度至少为2 的文件



####  根据文件类型搜索

find . -type d -print

上面的命令只会打印出目录

文件类型                类型参数

普通文件                f

符号连接                l

目录                       d

字符设备                c

块设备                    b

套接字                    s

Fifo                        p



#### 根据文件时间进行搜索

unix/linux 文件系统中的每一个文件都有三种时间戳(timestamp) ,如下所示

访问时间 (-atime) : 用户最近一次访问文件的时间

修改时间 (-mtime): 文件内同最后一次被修改的时间

变化时间 (-ctime) : 文件元数据(metadata,例如权限或者所有权)最后一次被修改的时间

在unux/linux中没有创建时间这个概念

-atime,-mtime,-ctime 可以作为 find 的时间参数,例如

打印出 最近七天内被访问过的文件

find . -atime -7 -print

打印出 刚好七天前访问过的文件

find . -atime 7 -print

打印出 访问时间超过7天的文件

find . -atime +7 -print

还有其他一些基于时间的参数 (以分钟为单位)

-amin

-mmin

-cmin

#### 基于文件大小搜索

find . -size +2k

大于2k的文件

find . -size -2k

小于2k的文件

find .-size 2k

等于2k的文件

除了K之外,还有其他单元

b------块  512字节

c------字节

w-----字  2字节

k-----千字节

m----兆字节

G----吉字节

#### 删除匹配文件

-delete 可以删除 find 查找到的匹配文件

find . -name "*.flv" -delete ##(删除所有flv文件)

基于文件权限和所有权的匹配

find . -perm 644 -print

find . -user huangyuan -print

#### 结合 find 执行命令或动作

find 命令可以借助选项 -exec 与其他命令结合

例如  将制定目录中的所有C程序文件拼接起来写入单个文件 all_C_file.txt 

find . -name "*.c" -exec cat {} \;> all_C_file.txt

-exec 后面可以接任何命令.{}表示一个匹配.使用 > 而不使用 >>　的原因是 find 命令的输出只是一个单数据流,而只有当多个数据流被追加到单个文件中的时候才有必要用 >> 

#### 跳过指定目录

find .  \( -name ".get" -prune  \) -o  \( -print \)

这条命令中  \(-name ".git" -prune \) 指明 .git 目录应该排除掉, 而 \( -print \) 指明了执行的动作.这些动作需要放在第二个语句块中.