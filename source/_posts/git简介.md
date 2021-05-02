---
title: git简介
date: 2020-12-10 00:36:12
tags:
	[git]
categories: 工具

---

# 1.什么是git？

git是一个分布式的版本控制工具，git的设计简单，是完全分布式的，允许成千上万的并行开发的分支（branch），有能力管理超大规模的项目，是首选的版本控制工具。

* 版本控制：一种记录一个或者若干文件内容的变化，以便查阅特定的版本修订情况的系统
* 分布式版本控制系统：把代码仓库完整的镜像下来，包括完整的历史记录。每一次克隆操作实际上都是一次对代码仓库的完整备份

当我们在使用git来管理我们的代码时，就可以很轻易的回溯到我们任意时间点所做的修改和提交，由于git的特性，我们可以很轻易就可以看到自己的修改，同时还可以利用分支管理在相同的时间做并行开发

<!--more-->

# 2.初次运行git前的配置

git自带一个git config工具可以用来帮助我们控制git的外观和行为的配置变量，一般来说，这些变量储存在三个位置上：

1. /etc/gitconfig文件：包含系统上每一个用户以及他们仓库的通用配置，如果在执行git config上加上--system选项，那么修改的就是这个文件的配置变量，这个的作用范围最广
2. ～/.gitconfig:只针对电脑的当前用户。可以传递--global参数来修改这个文件里的配置变量
3. 当前使用的git仓库里的.git会有一个config文件：这个只针对该仓库，可以传递--local选项修改，默认也是这个

每一个级别会覆盖上一个级别的配置，也就是说你给某个git仓库的user.name修改后，上面的两个gitconfig的相应配置变量就对这个git仓库无效了

可以使用 `git config --list --show-origin`查看所有的配置以及他们所在的文件

![屏幕快照 2020-12-10 上午1.11.41](https://tva1.sinaimg.cn/large/0081Kckwly1gli3to5yt4j314g03mwf9.jpg)

这个是我的所有git目录，当然我不是在某个git仓库里执行的命令,所以找不到第三个gitconfig，而我们进入一个git目录就会发现：

![屏幕快照 2020-12-10 上午1.14.54](https://tva1.sinaimg.cn/large/0081Kckwly1gli3x2idrmj314u0ay76o.jpg)

内容会多出一些，关于仓库的远程连接，一般在安装好git以后，我们只需要执行：

```
git config --global user.name "your username"
git config --global user.email yourEmail
```

把自己的用户信息写进gitconfig里，这样，当我们在git仓库里做修改或着提交时候就知道你的信息了

![屏幕快照 2020-12-10 上午1.18.34](https://tva1.sinaimg.cn/large/0081Kckwly1gli413hr5dj317e0gujtn.jpg)

​	比如这几个提交，我们就可以轻易的知道来自于哪里了

# 3.git的工作流程

git的三个工作区域对应着三个工作状态：

![6A5682E0F6FBFF11C78ADA00C48A2B3F](https://tva1.sinaimg.cn/large/0081Kckwly1gli45heht1j33340mmq9s.jpg)

工作流程：

1. 在工作目录里修改文件（modified）；
   1. 我们在工作区修改了文件，而没有用git add加入到暂存区时，就是modified
2. 暂存文件，将文件快照放入暂存区（staged）；
   1. 使用git add添加以后，就变成了staged
3. 提交更新，找到暂存区的工作文件，将文件永久的存储到git的仓库目录里（committed）

git关于本地文件的状态：

编辑某些文件以后，由于自上次的提交后，你对他们做了修改，git将他们标记为已修改文件。在工作时，你可以选择性地将这些修改过的文件放入暂存区，然后提交所有已经暂存的修改，如此反复：

![IMG_125B9A301149-1](https://tva1.sinaimg.cn/large/0081Kckwly1glljf3ubkfj318f0hjaee.jpg)

# 4.git的简单使用

* git init 初始化一个空的git仓库，这时候相应的文件夹下会自动生成.git的文件夹，里面就有刚刚说的gitconfig

* git status:查看文件的工作状态

* git add 跟踪文件，运行

  * 此时被add的文件已经被跟踪，处于暂存的状态，显示为changes to be committed

* git commit -m "提交所做的注解" 提交更新到仓库

* git diff:显示暂未暂存的改动，并非所有改动

* git rm 文件名：删除掉暂存区的文件,一般加上后缀--cached表示不删除原文件，只是从git暂存区删除，而-f则是直接删除掉这个文件

* git log:查看提交的历史，可以查看一个项目提交历史的各个人的提交

* 版本回退：在git log里可以看到 commit后面跟着的这一大串散列码，这就是git的版本号，我们可以使用git reset --hard 版本号回退到之前的任何一个版本（本地是reset，远程是revert）

  ![屏幕快照 2020-12-10 上午1.50.46](https://tva1.sinaimg.cn/large/0081Kckwly1gli4ydr3pzj30u00ufdlq.jpg)

# 5.关于忽略文件.gitignore

有时候会有一些文件无需要纳入git管理，通常是一些自动生成的文件，比如.exe或者一些日志文件，这种情况下我们可以创建一个名为.gitignore文件

例如：

![屏幕快照 2020-12-13 上午12.44.50](https://tva1.sinaimg.cn/large/0081Kckwly1glljwvkko2j30dy0i4wfq.jpg)

倒数第二行意思就是git忽略所有.o或者.a结尾的文件，一般这类文件都是在编译过程中出现的，最后一行就时忽略所有～结尾的文件。

最简单的情况下，一个仓库可能只有根目录下有一个.gitignore文件，会递归的调用到整个仓库中，而当子目录下也有额外的.gitignore文件。子目录中的.gitignore文件的规则只作用于它所在的目录。

文件.gitignore的格式规范如下：

1. 所有空行和#开头的行都被视为注释而忽略掉
2. 使用标准的glob模式匹配，会递归的调用到整个文件区（？）
3. 匹配模式可以以（/）开头防止递归
4. 匹配模式可以以（/）结尾指定目录
5. 要忽略指定模式以外的文件或者目录，可以在模式前加！取反

![IMG_7193A70E1EED-1](https://tva1.sinaimg.cn/large/0081Kckwly1gllk9eypvij31210jt42i.jpg)

关于glob模式：

一种在shell里简化了的正则表达式，用*等匹配模式来匹配文件，在解析路径模式时，大括号内用逗号进行分割，分割的部分可以包含/，所以a{/b/c,bcd}会被展开为a/b/c和abcd

在路径匹配模式里，以下字符有一些特殊的作用：

* *：匹配单路径下0个或者多个字符串

* ？：匹配一个字符串

* [...]：匹配指定范围内的字符串，如果[]中第一个字符是！或者^ ，则匹配不在范围内的任意字符串

知道这么多差不多够用了

  

# 6.git的分支管理

## 1.分支的简单图解

在版本回退里，我们可以回溯到之前的任意一个版本里，而这些版本都处在一条时间线上，这条时间线就是一条分支

默认的分支是main（之前是master，后来由于这个词有歧视黑人的意思，就换了这里的图都是用的master），本身可以把它看成一个指针，head指针就是一个特殊的指针，这里就指向master指针

![image-20201210015840577](https://tva1.sinaimg.cn/large/0081Kckwly1gli5672o9lj308d0473yf.jpg)

我们在每次提交的时候，都会多出一个节点，指针也会随之移动，当我们创建一个新的分支，也就是创建一个新的指针，我们可以通过命令将head指针移动到新的指针上：

![image-20201210020044381](https://tva1.sinaimg.cn/large/0081Kckwly1gli58b6b3mj30dq06h3yj.jpg)

我们还可以将两个分支合并：

![image-20201210020232237](https://tva1.sinaimg.cn/large/0081Kckwly1gli5a76lkcj30br066jre.jpg)

还可以在这里把dev分支给删掉

![image-20201210020314110](https://tva1.sinaimg.cn/large/0081Kckwly1gli5awchx8j30br04fwef.jpg)

## 2.分支的使用

1. git branch 查看当前所有分支，及所在分支
2. git branch 分支名 ：创建新的分支
3. git checkout 分支名：切换到别的分支
4. （处于某个分支下）git merge 分支名，这就是将两个分支合并，此时两个分支就处在同一个节点下
5. git branch -d ：删除分支

# 7.关于git远程仓库的使用

## 仓库服务器

如果想查看你已经配置的远程仓库的服务器，可以运行`git remote`命令，会列出你的远程服务器的简写，一般来说链接了远程仓库，至少能够看到origin，这是给你克隆的仓库服务器的默认名字，也可以指定选项-v，会显示需要读写远程仓库使用的git保存的简写与其对应的url

![屏幕快照 2020-12-13 上午1.21.19](https://tva1.sinaimg.cn/large/0081Kckwly1gllkyky485j30qo07adgr.jpg)

## 添加远程仓库：

可以使用git clone自动自行添加远程仓库的，也可以自行配置，比如：

`git remote add <shortname> <url>`:添加一个新的远程git仓库，同时指定一个方便实用的简写

![IMG_6B07EE8387AE-1](https://tva1.sinaimg.cn/large/0081Kckwly1glllgrj6rlj314a0qpgw0.jpg)

果你想要查看它，可以剪出一个指向改点的本地分支

## 从远程仓库中抓取和拉取：

git fetch  < remote> 会将远程的仓库的数据下载到本地仓库，但不会自动合并并修改当前的工作，需要手动合并，如果当前分支设置了跟踪的远程分支，可以使用git pull自动抓取远程分支的数据合并到本地

## 推送到远程分支：

git push < remote> < branch>,如果要将master分支推送到origin服务器，那么就可以运行`git push origin master`

比如第一次对远程仓库进行操作，使本地与github同步

> git remote add origin +git地址 (添加远程仓库到本地)
>
> git pull --rebase origin main(更新远程仓库的更新（mian分支)到本地）
>
> git push -u origin main将本地仓库与远程合并

# 8.git 与github的多人协作

在一个github的仓库里，我们一般会有一个主分支，会有许多许多开发分支，每个人可以自己拉取一个分支，也可以都在一个分支里开发，开发者单独在这样的分支里开发自己的代码和功能，最后提交到拉取的分支上，最后再pull request一下，等待确认无误，再merge到主分支上，这就是一个规范的多人协作开发的流程。

将项目克隆到本地打开以后，在本地的仓库里建立新分支并连接远程分支

```git
git checkout -b dev origin/dev
```

之后就可以在本地的dev分支进行提交（add, commit, push)，可能会出现提交不成功，应该是因为远程分支的版本比本地的更新，这个时候只需要git pull一下，将远程更新拉取到本地就行，这里可能出现的问题有两种：

1. 如图：

   ![屏幕快照 2021-01-23 下午7.49.34](https://tva1.sinaimg.cn/large/008eGmZEgy1gmxvej76dvj30zg0b63zs.jpg)只需要在按提示在命令行输入 gi t branch --set-upstream-to=origin/dev dev 就可以，这样会把本地的dev分支追踪到远程仓库的dev分支

2. 注意这个时候也有可能会出现冲突，可以使用git status查看那些文件有问题，进入文件查看几个相应的地方出现的>>>>>>>> =========<<<<<<<<这样的东西，删掉冲突的地方再add就可以了

完毕后，我们就将自己在本地dev上的更改提交到了github的远程仓库上，这个时候只需要在github上提交一个pull request，注意是在dev分支向main分支上提pull request，就完成自己的提交任务。

在项目开发的过程中，可能会有其他的人的提交被main分支merge上了，我们需要在本地切换到main分支上，git pull一下，同步更新，之后再进入dev分支，也要调用git merge origin/main 合并主分支上的最新更新，如果有冲突，就像前面那样处理冲突，在就可以git add 等，最后git merge -continue