---
title: git版本控制
tags: [git,运维]
date: 2019-01-27 04:28:32
keywords: git,版本控制
---

版本库也也是仓库，表现为一个目录或者是一个文件夹，这个文件夹里面的所有文件都可以被Git管理起来，文件修改删除也都能被Git记录下来，方便版本控制。

 git相关概念简介：

   + 工作区：就是存放文件的文件夹。
   + 版本库：可以粗略的理解为 `.git` 文件夹
   + 版本库中包含一个`暂存区` 和 多个`分支`，当我们执行完`git init` 的时候，系统自动为我们创建了一个`master` 分支

<!--more-->
#### 1 本地版本库
##### 1.1 创建本地版本库 

   在适当的位置创建一个文件夹，名字随便但最好要有意义，同时不建议文件夹路径中有中文。打开git shell，切换到创建的文件夹中，输入`git init`，将当前文件夹初始化为git仓库，会发现当前文件夹下多出了一个 `.get` 文件夹。

##### 1.2 将文件添加到仓库和文件修改

在当前目录下新建一个文本文档，名字随便，当然要有意义，我新建的是`TestGit.txt`,新建完之后，我们在 `git shell`中输入`git status`来查看工作区的状态

``` shell
$ git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        Test.txt

nothing added to commit but untracked files present (use "git add" to track)

```

注意，**一定要看git返回的提示是否成功**、**一定要看git返回的提示是否成功**、**一定要看git返回的提示是否成功**。

 第一行提示我们现在在 `master` 这个分支上，下面提示我们有一个文件没有被追踪，可以使用`git add fileName` 方式添加，我们输入 `git add Test.txt`,然后再用`git status`来查看工作区间

``` shell
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   Test.txt


huangyuan@xuan MINGW64 ~/Desktop/test (master)

```

系统提示已经将该文件添加到暂存区，可以使用 `git rm --cached FileName`将文件从暂存区删除（这并不会删除你创建的文件，只是删除git版本库中的暂存），接下来可以使用`git commit -m “注释”` 将暂存区内容提交到到`master`分支，然后我们再查看工作区状态，发现工作区是干净的:

``` $ git status
On branch master
nothing to commit, working tree clean
```

我们已经创建并且添加了一个文件到版本仓库，现在修改`Test.txt`文件，随便写点东西，然后保存，回到git shell，使用`git status` 查看工作区状态，提示文件已经被修改，可以使用`git add fileName` 将修改后的文件添加到暂存区，或者使用`git checkout -- fileName` 将文件返回到修改前的状态（这个修改前的状态是指的上一次提交之后版本库的文件状态），或者使用`git diff fileName`命令来查看修改了哪些东西：

``` shell
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   Test.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

使用 `git add`更新将要被commit的文件

使用 `git checkout -- Test.txt` 命令还原文件状态，会发现文件变回上次提交之后的状态了。

####1.3 版本回退
如果我们新建了很多文件，并且都提交到了版本库，现在想要回退到某一个版本，可以使用`git log`或者`git log --graph` （`git reflog` 会记录每一次使用的命令）来查看每一次提交的`commitid`、`Author`、`Date`，我们可以通过`git reset --hard commitid` 或者 `git reset --hard HEAD`来将版本回退到某一个特定的版本。

其中 `HEAD` 表示的是当前版本库中的最新版本，`HEAD^`表示的是上一个版本，`HEAD^^`或者`HEAD~2` 表示的是上上个版本，依此类推。（ps：当使用 `commitid` 来回退版本的时候，不需要将id全部写出来，只写出前边一部分即可）：

这样的话，工作区的内容也回退到了指定版本的状态。

注意，git reset 参数可以是一下几种

``` shell
git reset [--mixed | --soft | --hard | --merge | --keep] [-q] [<commit>]
   or: git reset [-q] [<tree-ish>] [--] <paths>...
   or: EXPERIMENTAL: git reset [-q] [--stdin [-z]] [<tree-ish>]
   or: git reset --patch [<tree-ish>] [--] [<paths>...]

    -q, --quiet           be quiet, only report errors
    --mixed               reset HEAD and index
    --soft                reset only HEAD
    --hard                reset HEAD, index and working tree
    --merge               reset HEAD, index and working tree
    --keep                reset HEAD but keep local changes
    --recurse-submodules[=<reset>]
                          control recursive updating of submodules
    -p, --patch           select hunks interactively
    -N, --intent-to-add   record only the fact that removed paths will be added later
    -z                    EXPERIMENTAL: paths are separated with NUL character
    --stdin               EXPERIMENTAL: read paths from <stdin>
```

#### 2 远程仓库

##### 2.1 创建新仓库

我们已经在本地创建好了仓库，现在想在远程github上创建一个仓库，是本地仓库和远程仓库可以进行同步，大家也可以协同工作。

打开我们自己的github主页，点击界面上的绿色按钮`New repository`

![New repository](/image/git/NewRepository.PNG)

打开创建仓库的界面,如下图所示

![Create new repository](/image/git/CreatNewRepository.PNG)

 填入仓库名字（不能和已有仓库名字相同）、描述、权限（私有的貌似要收费）、选择初始化的时候是否创建`README`文件（Markdown格式）、是否添加`gitignore`文件、是否添加版权（一般是选 GNU General Public License  或者 Apache License，关于这两个协议，大家可以去网上搜一下），然后点击下面的 `Create repository`就完成了远程仓库创建（如果这里选择了添加协议，就不由有下面的2.2步骤）

##### 2.2 推送本地文件到远程仓库

如果我们在创建远程仓库的时候没有选择添加任何文件，那么创建成功的仓库将会是一个空仓库如下图所示：

![empty repository](/image/git/emptyRepository.PNG)

如果选择了添加文件不会有上图所示的提示，而是显示仓库里面包含的文件，下面我们将本地仓库和远程仓库关联起来（其实空仓库都有详细的提示，我也只是负责翻译一下）。

```shell
echo # repository name>> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin git@github.com:your name/repository name.git
git push -u origin master
```

第一条命令，创建 `README.md`文件，并将`# repository name` 写入到文件
2,3,4前面介绍过了，关键是5,6,：
第五条命令，将本地仓库和你自己的远程仓库关联起来，`origin`是本地别名，后面的是远程仓库的地址
第六条命令，将本地仓库的文件推送到远程仓库
如果在本地有建立好的仓库，可以切换到仓库目录，执行第五第六条命令。
如果选择了添加文件，但是本地没有仓库，可以点击仓库页面右下角的 `Download ZIP`按钮，下载远程仓库到本地，然后切换到仓库目录，这时就不需要关联仓库了。不想点击按钮下载的话可以使用 `git clone`命令，格式如下：
`git clone git@github.com:your name/repository name.git`

#### 3 github分支

在git中，每当我们`commit`的时候，git将会把提交的时间串成一条线，这条线就是一个分支，也就是主分支，也叫`master`分支（默认情况下）。严格来说，`HEAD`并没有指向提交，而是指向了`master`，`master`指向的提交。每次提交的时候，`master`就会将当前提交时间串在时间线上，并且指向当前时间点，随着不断的提交，	`master`分支也越来越长，当我们使用`git log`命令的时候，查看到的就是这条时间线。

##### 3.1 分支管理

当我们创建一个新的分支，比如叫 `dev`，现在`dev`指向的提交就是`master`指向的提交，但是`HEAD`现在指向`dev`，现在我们再提交的时候就是提交在`dev`分支了，而`master`分支不变。当我们在`dev`分支上完成开发工作，检查无误后，就可以将`dev`分支合并(merge)`master`分支上了。
应用场景：当我们的软件开发完成1.0版本后，现在有了新的需求，要开发2.0版本。但不幸的是，在开发到一半的时候，1.0版本发现了几个bug，没办法，改吧，但是我们显然不能再现在的工程基础上修改，如果在现在工程上修改的话，1.0版本的软件将会带有部分2.0版本的特性或者功能，成了1.5版本。这时候，我们在完成1.0版本后，就可以新建一个分支，在新的分支上进行新的开发，在确认完成后，再合并到主分支。
操作如下：

```git checkout -b dev```

`-b`参数表示创建并切换到新的分支，相当于下面两条命令

```git
git branch dev
git checkout dev
```

然后使用`git branch`命令查看所有的分支，在当前所在的分支上会有`*`提示.

现在我们处于`dev`分支下，`Test.txt`文件里面没有任何内容，对`Test.txt`进行修改并提交。

```git
git add Test.txt
git commit -m "branch test"
```

##### 3.2 合并分支

然后切换到`master`分支，再去查看`Test.txt`文件中的内容，并不是和`dev`分子相同。这是因为我们提交的内容在`dev`分支上，`master`分支并没有做什么改变。

然后对两个分支进行合并,合并完成后删除`dev`分支
```git
git merge dev
git branch -d dev
```
合并完成后发现`Test.txt`文件内容与`dev`分支下的完全相同。

##### 3.3 合并时的冲突

不幸的是，没有什么是一帆风顺的，合并时遇到了冲突怎么办？系统给出了这么个提示
```git
D:\GitRepositorys [master]> git merge dev
warning: Cannot merge binary files: Test.txt (HEAD vs
Auto-merging Test.txt
CONFLICT (content): Merge conflict in Test.txt
Automatic merge failed; fix conflicts and then commit
D:\GitRepositorys [master +0 ~0 -0 !1 | +0 ~0 -0 !1]>
```

这个冲突貌似只能手动修改(借助一些工具比如beyond compile)，打开冲突的文件，查看冲突提示，手动修改完成后再提交、合并
（其实我也不大懂，只是建议不要在两个分支上同时修改同一个文件）

#### 4 常用分支管理策略

##### 4.1   分支合并模式

通常情况下，git在合并分支的时候会用`Fast forward`模式，但是在这种模式下，删除分支后，会丢掉分支信息，也就是分支提交的信息，如果强制禁用`Fast forward`模式，git在葛冰分支的时候会生成一个新的`commit`，语法格式如下（将dev分支合并到master分支上）：
`git merge --no-ff -m "禁用Fast forward 模式合并分支" dev`
##### 4.2 分支策略

首先，`master`分支是非常稳定的，也就是仅仅用来发布新的版本。
其次，从`master`分支上创建一个新的分支（dev），每个开发人员都有自己的分支（从dev分支上创建的），推送代码时只要将自己的分支和dev分支合并就可以了，
最后，当药发布新版本时，只需要将dev分支合并到master分支就可以了

##### 4.3 bug分支

当你正在进行开发时，突然间发现了以前的一个bug，需要立刻修复，但是你现在的开发工作还没有完成，没有办法提交，git提供了一个`stash`功能，可以把当前场景保存起来，等以后可以恢复现场：`git stash`，现在用`git status`查看工作区，是干净的了，可以创建一个新的分支（首先要确定从哪个分支上修复bug），修复完成后，合并并删除bug分支。
现在我们需要恢复现场，使用`git stash list`命令来查看保存了哪些现场，恢复现场有两种方式，一是用`git stash apply stash@{NUM}` 或者是用`git stash pop`，区别是前者不会删除stash，恢复现场后需要收到执行 `git stash drop`来删除。当然，也可以多次stash然后使用`git stash apply stash@{num}`来恢复指定的现场。

#### 5. 多人协作

##### 5.1 克隆仓库

当你从远程仓库克隆时，实际上Git自动把本地的`master`分支和远程的`master`分支对应起来了，并且，远程仓库的默认名称是`origin`。

要查看远程库的信息，用g`it remote：`

```git
$ git remote
origin
```
或者，用`git remote -v`显示更详细的信息：
```git
$ git remote -v
origin  git@github.com:michaelliao/learngit.git (fetch)
origin  git@github.com:michaelliao/learngit.git (push)
```
上面显示了可以抓取和推送的origin的地址。如果没有推送权限，就看不到push的地址。
##### 5.2 推送分支

推送时，要指定本地分支，这样，Git就会把该分支推送到远程库对应的远程分支上：
`$ git push origin master`
如果要推送其他分支，比如dev，就改成：
`$ git push origin dev`

其中`origin`就是上篇文章中提到的`本地别名`

##### 5.3 抓取分支

多人协作时，大家都会往`master`和`dev`分支上推送各自的修改。当我们从远程仓库克隆的时候，我们只能看到本地`master`分支，如果我们要在其他分支上开发，就必须创建`origin`的其他分支到本地，或者说建立远程分支和本地分支的关联:
`git checkout -b dev origin/dev`,然后，我们就可以在`dev`分支上继续修改，然后将修改后的项目推送到远程

#### 6. 标签管理

##### 6.1 创建标签

首先切换到需要打标签的分支上，`git checkout branch-name`,
然后使用命令`git tag <name>`就可以了，默认标签是打在最新提交上的，如果我们想要打在以前的提交上，可以使用`git log`查看以前提交的commit id，然后使用`git tag <name> <commit id>`就可以了。可以使用命令`git tag`查看标签，但是标签不是按照时间顺序排序的，而是按照字母顺序排序。在打标签的时候可以带有说明，语法格式如下：
`git tag -a <tag-name> -m "comment" <commit id>`
使用命令`git show <tag-name>`可以查看详细信息。

##### 6.2 删除标签

如果标签打错了，可以使用`git tag -d <tag-name>`来删除指定的标签，默认情况下，标签不会自动推送到远程。如果想把某个标签推送到远程，可以使用`git push origin <tag-name>`，或者一次性推送所有本地的标签:`git push origin --tags`,如果标签已经推送的远程仓库，想要删除远程仓库的标签需要如下两步:
首先删除本地标签 `git tag -d  <tag-name>`
然后从远程删除 `git push origin :refs/tags/<tag-name>`



----

以上