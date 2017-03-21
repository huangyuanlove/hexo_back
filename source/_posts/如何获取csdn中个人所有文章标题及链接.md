---
title: 如何获取csdn中个人所有文章标题及链接
date: 2016-10-19 22:32:42
tags: csdn
---
小伙伴问如何把自己csdn上的文章标题和超链接都扒下来,问我是不是一个个点开之后抄过去的...当然不是,做为一个爱(sha)好(dou)广(dong)泛(dian)的程序员,怎么会用这么麻烦的方法.
<!-- more -->
本来打算写java模拟登录之后获取网页源码,然后再用jsoup去解析,得到自己需要的数据.原来就这么干过,好像是写学校的绩点计算器来着吧,就是输入帐号密码就能查到自己的成绩和绩点的那种.
翻出来代码看了看太麻烦了,想到自己最近在看python,就想着用python来解决,但是问题又来了,python不熟啊,就算写出来了也得看网页源码,找到规律才行,讲道理的说挺烦这东西的.
最后,还是用js来解决,毕竟这种事也干过,也不是多麻烦.
登录自己的帐号,找到文章列表,打开控制台窗口,忘了说一下自己的环境了`ubuntu`,`chrom`,找到文章标题和超链接的部分,如下图:
![csdn个人博客列表](/image/csdn_home.png)
发现所有的文章标题和超链接结构如下:
```HTML
 <span class="link_title">
 	<a href="/huangyuan_xuan/article/details/51935666">
        初步编写IDEA\AndroidStudio翻译插件            
        </a>
</span>
```
另外,界面中还引入了jQuery这个三方库,这就更简单了:
找到开发者工具的控制台(console),写入下面两行代码
```JavaScript
var aTags = $(".link_title > a")
for(var i = 0;i<50;i++)
{
	console.log("[" + aTags[i].text.trim() + "](" + aTags[i].href +") "+ aTags[i].href) +"<br/>"
	}
```
ok,执行结果如下
![csdn个人博客标题和超链接](/image/csdn_blog_title.png)
在控制台写的代码第一行是使用jQuery库找到所有文章的超链接集合,第二行是按照`markdown`超链接的语法打印出来文章标题和超链接,至于循环中的`50`这个数字,一页最多只有50篇文章,我偷懒了,建议使用`aTags.length`
<hr>
以上
