# git实践笔记

# 概述
本文记录常用 git 的功能和命令.

<!-- more -->

# Git实践笔记

## Why
一年多前一边工作一边学,做的笔记,后来换了工作,改用SVN,git也就生疏了,最近公司打算换git了,正好重新整理一下笔记.

## What
git是目前最好的版本控制工具,是一种动态异步的版本控制工具,对于版本控制的发展历程,可以参考别的文章.目前各个开源管理平台基本上都是用的git,git是必备的技能.

## 简介

Linus花了两周时间自己用C写了一个分布式版本控制系统，这就是Git!大写的牛逼!一个月之内，Linux系统的源码已经由Git管理了!
起初的git只能在linux和Unix上运行。

## 安装git

-  在Linux上安装Git

命令行下输入,*sudo apt-get install git*,直接安装.

- 在Windows上安装
从http://msysgit.github.io/下载。

在bash下输入以下命令，设置账号和邮箱。是全局的，在之后的所有git操作，都是以这个账号.

```shell
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
```
** 注意：**
git config命令的--global参数，用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址。

```shell
$ git config --global --list查看当前的所有设置清单列表。
```
## 创建版本库（repository）
如果你使用Windows系统，为了避免遇到各种莫名其妙的问题，请确保目录名（包括父目录）不包含中文。
一共三步：初始化，添加文件，提交
```shell
$ git init

$ git add readme.txt

$ git commit -m "提交的信息"
```

## 查看仓库的修改状态
```shell
$ git status
这个命令会告诉我们修改了哪些文件，在知道了修改的文件之后，通过
$ git diff readme.txt
这个命令可以查看修改的具体内容。
```
## 版本回退：
每当你觉得文件修改到一定程度的时候，就可以“保存一个快照”，这个快照在Git中被称为 *commit*。一旦你把文件改乱了，或者误删了文件，还可以从最近的一个commit恢复，然后继续工作。版本控制系统肯定有某个命令可以告诉我们历史记录，在Git中，我们用git log命令查看。

```shell

$ git log

$ git log --pretty=oneline

$ git reset --hard HEAD^（表示head指向回退到上一个版本）

$ cat readme.txt

$ git reset --hard 3628164
```

Git的版本回退速度非常快，因为Git在内部有个指向当前版本的HEAD指针，当你回退版本的时候，Git仅仅是把HEAD从指向改变了。然后顺便把工作区的文件更新了。所以你让HEAD指向哪个版本号，你就把当前版本定位在哪。

找不到新版本的commit id怎么办？

```shell
$ git reflog
```

**总之：**
HEAD指向的版本就是当前版本，因此，Git允许我们在版本的历史之间穿梭，使用命令git reset     --hard commit_id。
穿梭前，用git log可以查看提交历史，以便确定要回退到哪个版本。
要重返未来，用git reflog查看命令历史，以便确定要回到未来的哪个版本。

## 工作区和暂存区：
- 工作区：就是在电脑上能看到的目录；一般就是项目文件的根目录；
- 版本库：工作区有一个隐藏目录.git文件夹，属于仓库文件，不属于工作区，是Git的版本库；

Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的暂存区，还有Git为我们自动创建的第一个分支master，以及指向master的一个指针叫HEAD。
把文件往Git版本库里添加的时候，是分两步执行的：
1. 第一步是用git add把文件从工作区添加到暂存区，实际上就是把文件修改添加到暂存区；
2. 第二步是用git commit提交更改，实际上就是把暂存区的所有内容提交到当前分支。

### 管理修改：
工作区的内容，必须add到暂存区以后才会在提交的时候被提交到库里。每次修改，如果不add到暂存区，那就不会加入到commit中

### 撤销修改：
git checkout -- file可以丢弃工作区的修改.把file文件在工作区的修改全部撤销，这里有两种情况：
- 一种是readme.txt自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；
- 一种是readme.txt已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。
- 总之，就是让这个文件回到最近一次*git commit*或*git add*时的状态。

git reset HEAD file可以把暂存区的修改撤销掉（unstage），重新放回工作区
git reset命令既可以回退版本，也可以把暂存区的修改回退到工作区。当我们用HEAD时，表示最新的版本。

- 场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令 *git checkout -- file*。
- 场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令 *git reset HEAD file*，就回到了场景1，第二步按场景1操作。

### 删除文件：

在Git中，删除也是一个修改操作

直接在文件管理器中把没用的文件删了，Git知道你删除了文件，因此，工作区和版本库就不一致了，git status命令会立刻告诉你哪些文件被删除了。

1.  用命令git rm删掉，并且git commit，用来从版本库中删除该文件.
2.  用命令git checkout -- file ，这样实现用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。

## 远程仓库

生成公私密钥：$ ssh-keygen -t rsa -C "youremail@example.com"
在用户目录下有.ssh目录，id_rsa和id_rsa.pub这两个文件。
id_rsa是私钥，不能泄露出去，id_rsa.pub是公钥，可以放心地告诉任何人。

## 添加远程库

- 在github上新建一个仓库。
- GitHub告诉我们，可以从这个仓库克隆出新的仓库，也可以把一个已有的本地仓库与之关联，然后，把本地仓库的内容推送到GitHub仓库。

``` shell
$ git remote add origin git@github.com:tinggengyan/study.git
```
- 添加后，远程库的名字就是origin，这是Git默认的叫法，也可以改成别的，但是origin这个名字一看就知道是远程库。
- 下一步，就可以把本地库的所有内容推送到远程库上： *$ git push -u origin master*,把本地库的内容推送到远程，用git push命令，实际上是把当前分支master推送到远程。
- 由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送到远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。

从现在起，只要本地作了提交，就可以通过命令：

```shell
 $ git push origin master
```

### 小结
要关联一个远程库，使用命令
```shell
git remote add origin git@server-name:path/repo-name.git
```
关联后，使用命令*git push -u origin master*第一次推送master分支的所有内容；此后，每次本地提交后，只要有必要，就可以使用命令*git push origin master*推送最新修改；

## 远程克隆

git clone git@github.com:michaelliao/gitskills.git

## 分支创建与合并
Git里，有个分支叫主分支，即master分支。HEAD严格来说不是指向提交，而是指向master，master才是指向提交的，所以，HEAD指向的就是当前分支。
Git用master指向最新的提交，再用HEAD指向master，就能确定当前分支，以及当前分支的提交点。
当我们创建新的分支，例如dev时，Git新建了一个指针叫dev，指向master相同的提交，再把HEAD指向dev，就表示当前分支在dev上。
当创建一个新的分支的时候，新分支的指针和旧的指针指向是同一个，同时将head的指向修改到当前的新分支的指针上，之后提交就可以提交到新的分支了。
当在新的分支上将工作完成之后，只要合并这两个分支就可以了，简单的就是讲master指向新的分支的指向。然后删除旧的分支的指针即可。

```shell
$ git checkout -b dev==={$ git branch dev；$ git checkout dev}创建一个Dev分支并切换到Dev分支。
$ git branch查看当前的分支。git branch命令会列出所有分支，当前分支前面会标一个*号。
$ git merge dev 命令用于将当前的分支和dev分支合并。
```
## 变基操作
对于merge操作，会合并两个分支，并且产生一个新的提交，这对于整体的 log 查看并不美观。
```shell
$ git checkout experiment
$ git rebase master
$ git checkout master
$ git merge experiment
```
它的原理是首先找到这两个分支（即当前分支 experiment、变基操作的目标基底分支 master）的最近*共同祖先*，然后对比当前分支相对于该*祖先*的*历次提交*，
提取相应的修改并存为*临时文件*，然后将当前分支指向目标基底, 最后以此将之前另存为临时文件的修改依序应用。
一般我们这样做的目的是为了确保在向远程分支推送时能保持提交历史的整洁才需要这么做。

```shell
$ git rebase --onto master server client
```
以上命令的意思是：“取出 client 分支，找出处于 client 分支和 server 分支的共同祖先之后的修改，然后把它们在 master 分支上重演一遍”。

```shell
$ git checkout master
$ git merge client
```
切换到master分支进行合并。

```shell
$ git rebase master server
```
这样就可以省的切换到sever分支，直接指定rebase。

```shell
$ git checkout master
$ git merge server
```
快速合并。


Git鼓励大量使用分支：

- 查看分支：git branch
- 创建分支：git branch <name>
- 切换分支：git checkout <name>
- 创建+切换分支：git checkout -b <name>
- 合并某分支到当前分支：git merge <name>
- 删除分支：git branch -d <name>

## 冲突处理
Git用

```shell
<<<<<<<，=======，>>>>>>>
```
标记出不同分支的内容，通过status命令，找到冲突的内容，手动修改，处理之后，再重新提交.
用
```shell
git log --graph --pretty=oneline --abbrev-commit
```
命令，查看一下分支合并.

https://app.yinxiang.com/Home.action#n=f311ba79-a871-47a5-bea1-0fbcb2277ea8&b=403e83c5-7d1d-4991-bc1b-9a6d80235d0b&ses=4&sh=1&sds=5&

##  分支管理策略：
合并分支时，如果可能，Git会用Fast forward模式，但这种模式下，删除分支后，会丢掉分支信息。
如果要强制禁用Fast forward模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。
```shell
git merge --no-ff 方式就能实现禁用fast forward模式。
```

**策略：**
首先，master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；那在哪干活呢？干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，再把dev分支合并到master上，在master分支发布1.0版本；你和你的小伙伴们每个人都在dev分支上干活，每个人都有自己的分支，时不时地往dev分支上合并就可以了。

###  Bug分支：
当出现bug的时候需要修复BUG，但是此时的工作区还有文件没有提交，此时的文件又不能提交，此时可以使用git的暂存功能。Git还提供了一个stash功能，可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作。 执行完stash之后的工作区就是一个干净的工作区。
```shell
git stash list
```
可以查看储藏的内容列表。接着就是恢复储藏的现场。
有两个办法：

- 一是用*git stash apply*恢复，但是恢复后，stash内容并不删除，你需要用 *git stash drop*来删除；
- 另一种方式是用git stash pop，恢复的同时把stash内容也删了：

可以多次stash，恢复的时候，先用 *git stash list* 查看，然后恢复指定的stash，用命令：

```shell
$git stash apply stash@{0}
```
### feature分支：
新增一个功能的生活，最好新建一个分支，之后再合并。没有和和分支在删除的时候，git会给出友情提醒，所以删除的时候，需要使用git branch -D <name>强行删除。

### 多人协作开发：
当你从远程仓库克隆时，实际上Git自动把本地的master分支和远程的master分支对应起来了，并且，远程仓库的默认名称是origin。要查看远程库的信息，用 *git remote*；
用git remote -v显示更详细的信息；
```shell
$ git remote -v
origin  git@github.com:michaelliao/learngit.git (fetch)
origin  git@github.com:michaelliao/learngit.git (push)
```
显示了可以抓取和推送的origin的地址。如果没有推送权限，就看不到push的地址。
推送分支
-  推送分支，就是把该分支上的所有本地提交推送到远程库。
-  推送时，要指定本地分支，这样，Git就会把该分支推送到远程库对应的远程分支上：

```shell
$ git push origin master 把master分支推送。
$ git push origin dev    把dev分支进行推送。
```
在Git中，分支完全可以在本地自己藏着玩，是否推送，视你的心情而定！

### 抓取分支
git pull把最新的提交从origin下相应的分支抓下来。
因此，多人协作的工作模式通常是这样：
1. 首先，可以试图用git push origin branch-name推送自己的修改；如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；如果合并有冲突，则解决冲突，并在本地提交；没有冲突或者解决掉冲突后，再用git push origin branch-name推送就能成功！如果git pull提示“no tracking information”，则说明本地分支和远程分支的链接关系没有创建，用命令 *git branch --set-upstream branch-name origin/branch-name*。

#### 小结
- 查看远程库信息，使用git remote -v；
- 本地新建的分支如果不推送到远程，对其他人就是不可见的；
- 从本地推送分支，使用git push origin branch-name，如果推送失败，先用git pull抓取远程的新提交；
- 在本地创建和远程分支对应的分支，使用git checkout -b branch-name origin/branch-name，本地和远程分支的名称最好一致；
- 建立本地分支和远程分支的关联，使用git branch --set-upstream branch-name origin/branch-name；
- 从远程抓取分支，使用git pull，如果有冲突，要先处理冲突。

## 标签管理
发布一个版本时，我们通常先在版本库中打一个标签，这样，就唯一确定了打标签时刻的版本。将来无论什么时候，取某个标签的版本，就是把那个打标签的时刻的历史版本取出来。所以，标签也是版本库的一个快照。Git的标签虽然是版本库的快照，但其实它就是指向某个commit的指针（跟分支很像对不对？但是分支可以移动，标签不能移动），所以，创建和删除标签都是瞬间完成的。

### 创建标签
git tag <name>就可以打一个新标签；
git tag 查看所有标签；
默认标签是打在最新提交的commit上的。有时候，如果忘了打标签，可以通过
```shell
$ git log --pretty=oneline --abbrev-commit 找到历史提交的commit id。
$ git tag   tagname    commit id来为历史提交打上tag。
git show <tagname>查看标签信息.
```
还可以创建带有说明的标签，用-a指定标签名，-m指定说明文字：
比如：
```shell
$ git tag -a v0.1 -m "version 0.1 released" 3628164
```

#### 小结
- 命令git tag <name>用于新建一个标签，默认为HEAD，也可以指定一个commit id；
- git tag -a <tagname> -m "blablabla..."可以指定标签信息；
- git tag -s <tagname> -m "blablabla..."可以用PGP签名标签；
- 命令git tag可以查看所有标签。

#### 操作标签
- 命令git push origin <tagname>可以推送一个本地标签；
- 命令git push origin --tags可以推送全部未推送过的本地标签；
- 命令git tag -d <tagname>可以删除一个本地标签；
- 先在本地删除一个标签，再命令git push origin :refs/tags/<tagname>可以删除一个远程标签。


## 忽略特殊文件
忽略某些文件时，需要编写.gitignore；.gitignore文件本身要放到版本库里，并且可以对.gitignore做版本管理！

## 配置别名
```shell
$ git config --global alias.st status
```
是在全局，整个电脑上的所有仓库都使用st来表示status。执行 *git st*=*git status*.
加上*--global*是针对当前用户起作用的，如果不加，那只针对当前的仓库起作用。
单个仓库的Git配置文件都放在.git/config文件中。
当前用户的Git配置文件放在用户主目录下的一个隐藏文件.gitconfig中。
别名就在[alias]后面，要删除别名，直接把对应的行删掉即可。

## 搭建git服务器
后加.



## 感激,非常感激，万分的感激！
感谢以下的文章以及其作者和翻译的开发者们,排名不分先后
* [Pro Git](https://git-scm.com/book/zh/v1/)
* [Git教程 - 廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

