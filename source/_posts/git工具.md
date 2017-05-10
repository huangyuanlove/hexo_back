---
title: git工具
date: 2017-05-10 16:51:11
tags: [git,git工具]
---
公司代码版本管理系统由svn迁到了git，对于AndroidStudio来讲，内置的GUI工具足以应付日常开发，但在请求失败的情况下，对失败原因的提示不够清晰。个人习惯上用命令行，但是对于命令行中比较两个文件差异以及合并来说，个人还是不大习惯，于是就配置成了使用其他软件进行合并。可以使用`$ git difftool --tool-help`查看对比文件差异支持的软件，用`$ git mergetool --tool-help`查看合并代码支持的软件，个人只试过两种:`codecompare`和`beyond compare`。不习惯`bc`的界面，最后决定使用`codecompare`。
<!--more-->
#### 配置codecompare为diff和merge工具
1. 安装codecompare软件
2. 配置codecompare为diff工具
	`git config --global diff.tool codecompare`
3. 配置codecompare的路径`path 后面是软件的安装路径`
	`git config --global difftool.codecompare.path D://CodeCompare//CodeCompare.exe`
4. 配置codecompare为merge工具
	`git config --global merge.tool codecompare`
5. 配置codecompare的路径`path 后面是软件的安装路径`
	`git config --global difftool.codecompare.path D://CodeCompare//CodeMerge.exe`
	
在比较本地修改后的文件与本地仓库中的文件差异时，执行 `git difftool <filename>`即可。
![codecompare实例](/image/git/git_diff_tool_codecompare.png)
当更新代码自动合并失败的时候，执行 `git mergetool`即可。
![codecompare实例](/image/git/git_merge_tool_codecompare.png)

#### 配置beyond compare为diff和merge工具
配置方式和codecompare一样，需要注意的是：
1. 如果`beyond compare`软件是4.X
  1) 如果git的版本低于2.2.0,配置的时候用`bc3`
  2) 如果git的版本大于等于2.2.0,配置的时候用`bc`
这个如何配置<a href="http://www.scootersoftware.com/support.php?zz=kb_vcs#gitwindows">官网</a>有说明，就不再赘述
在比较本地修改后的文件与本地仓库中的文件差异时，执行 `git difftool <filename>`即可。
![bc实例](/image/git/git_diff_tool_bc.png)
当更新代码自动合并失败的时候，执行 `git mergetool`即可。
![bc实例](/image/git/git_merge_tool_bc.png)
有人不习惯git自动merge成功后填写merge信息的编辑器，说明一下，默认的编辑器是`vim`,不习惯用的话可以使用 `git config --global core.edit <软件路径>`来修改。需要注意的是，git命令行似乎读不到windows系统的path，需要写软件的绝对路径。
----
以上