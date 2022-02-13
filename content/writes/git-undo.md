---
title: Git撤消概括
subtitle: 谁还没有一颗想改变历史的心呢？
tags:
- git
---

### I.Git撤消之谜

[Stackoverflow按投票(votes)排序搜索git](https://stackoverflow.com/questions/tagged/git?sort=votes&pageSize=30)，前15+的问题全部都是一个主题：**求解各式各样的撤消**。Github也早在2015年就贴心地把git常见撤消分类总结成简单易懂的情景再现:[How to undo(almost) anything with git](https://blog.github.com/2015-06-08-how-to-undo-almost-anything-with-git/)。可见初学者对git撤消是有很多疑惑点。不过，不论是Stackoverflow还是Github都侧重针对具体问题/场景提出纠正流程，虽然受益良多，但是总感觉太零碎，没能连成系统，于是就有了这篇总结。

### II.Git工作区/暂存区/版本库

![git-work](/writes/images/git-work.jpg)

- 工作区(work space)：就是你可以直接查看和编辑的文件目录。
-  暂存区(index)：是单个大型二进制文件(*.git/index*)，它描述了当前分支上所有的文件的sha1校验/时间戳/文件名。它并不是一个别一个类似工作区的副本。
- 版本库(repository)：它隐藏于*.git/objects*下，包涵了每一个文件(本地和远程)的所有版本。

**所以：不要将上图中表示的4个磁盘视为repo文件的单独副本。**

从上图的流程可以看出，撤消无非就是分成四大类：

* 撤消工作区改动。
* 撤消暂存区改动。
* 撤消本地版本库改动。
* 撤消远程版本库改动。

### III.HEAD~和HEAD^区别

[HEAD](https://stackoverflow.com/questions/8196544/what-are-the-git-concepts-of-head-master-origin)表明**Repo目前指向的是那个位置**。大多数情况下，HEAD指向当前分支中的最新提交。比如，你刚使用checkout 切换到新分支时，它就指向着新分支的最新提交。但是你可以任意改变HEAD的指向：

```shell
git branch -f Branch <SHA>
```

- **HEAD~**是**HEAD~1**的简写，它指HEAD的上一个commit，**HEAD~2**指HEAD的上一个commit的上一个commit。以此类推。
- **HEAD^**是**HEAD^1**的简写，它指HEAD的上一个commit，但是如果当前commit是由两个commit合并而成时，父commit就有了2个commit，所以区别是**HEAD^1**指父commit中的第一个commit，**HEAD^2**则指父commit的第二个commit,如果区分它们谁是第一/二，可以使用*git log* 看出来。

![head](/writes/images/git-graph.svg)

还想了解HEAD,Branch更多的内容。推荐玩一下游戏:[LearnGitBranching](https://learngitbranching.js.org/)，通关后，你将对它们有一个非常直观深刻的认识。

本文中的所有*SHA*都可以用**HEAD^**或**HEAD~**来代表相对的位置的SHA。

### IV.撤消工作区改动

工作区存放的是还未使用`git add`添加到暂存区的改动：

```shell
git checkout <bad filename>
```

警告：这会让文件的修改完全消失掉，恢复成git能记录的最新状态。所以你必须非常确信是不是有必要这样做。可以使用`git diff` 先比较下改动的内容是不是真的不需要了。

### V.撤消暂存区改动

暂存区存放的是使用了`git add `过，但未使用`git commit -m `的改动:

```shell
git reset <FileName> 
git reset --hard <FileName>
```

如果reset后面是一个文件名，那么了就把这个文件从暂存区移到工作区，如果加了"--hard"，则会把这个文件的改动直接移除掉，就像改动从来没有发生过一样。

### VI.撤消本地版本库改动

本地版本库存放的是使用了`git commit `过，但未使用`git push`到远端的改动:

```shell
git reset <last good SHA> 
git reset --hard <last good SHA>
```

如果reset后面是指定的*SHA*。 默认情况，它会在还原时把改动都放到工作区----commit消失了，但是改动还在硬盘上。如果你非常确信已不需要这部分改动。可以加上“--hard”选项。它会直接把改动丢弃掉。总之：默认会把改动移到工作区，加入“-hard”会把改动直接移除掉。

👉 **SHA**可以使用**HEAD^~**来指定相对位置或使用以下命令查看：

```shell
git log --oneline
```

还有一种常见的情况：重新编辑最近一次的commit改动。

已使用`git commit -m "Fixes bug #123"`但是还没`git push`到远端。这里你意识到日志写错了，应该是*Fixes bug #1234*。

```shell
git commit --amend
git commit --amend -m "Fiexd bug #1234"
```

`—amend`会把上一次的commit和在暂存区(使用`git add`过的文件)组合在成一条新的commit。如果暂存区里没有改动。它就相当于重新写一下上一条的commit日志。

### VII.撤消远程版本库改动

远程版本库存放的是已使用`git push`到远端的改动:

```shell
git revert <SHA>
```

revert会创建一条与给定*SHA*相反(逆向)的提交改动。所有在旧commit里新增改动都会在新commit里移除，所有在旧commit里面移除的部分都会在新commit中找回来。它不会改变git的历史日志，是git最安全的撤回方式。这样你就不需要使用`git push -f`操作，避免覆盖掉别人的修改。

### VIII.撤消其它常见的误操作

* 删除已加入版本库的文件(即不在Track此文件).

  ```shell
  git rm --cached no_need_filename
  ```

  然后在*.gitignore*文件中把文件名列进去。

* 本该使用新分支开始工作的，结果却在master上做了修改.

  ```shell
  git branch feature
  git reset --hard origin/master
  git checkout feature
  ```

  为当前的修改新建一个分支feature，然后把master分支reset到和远程master一样的状态，然后再切换到新分支即可。

* 工作分支与最新的master分支差距非常大时，使用rebase同步最新内容到feature上.

  ```shell
  git checkout feature
  git rebase master
  ```

* 删除分支.

  ```shell
  git branch -d LocalBranchName    #删除本地分支
  git branch -D LocalBranchName    #强制删除本地分支
  git push origin --delete <remote>/<branch>  #删除远程分支
  git branch --delete	--remotes <remote>/<branch> #删除local remote-tracking分支
  git fetch <remote> --prune #删除无用的tracking分支
  ```


### IX.Rebase

[Rebase](https://git-scm.com/docs/git-rebase)是一个功能非常复杂的命令，本文只讲如何利用它来改变历史！

```shell
 git rebase --interactive HEAD~7
```

用编辑器打开前7条的commit，根据提示进行整理。一共有6个选项。

* **pick**: 不改变commit任何内容，你可以通过上下移动这条commit来改变排列顺序。

* **reword**: 与pick类似，不改变内容，但可以重写提交日志。

* **edit**： 修改commit内容，可以再继续rebase前进行更多提交，这就允许装大型提交分割成较小的提交，或删除提交中的错误更改。

* **squash**：把多个commit合并成一个commit，并使用一个新的commit message来描述它们。对整理message非常有用。

* **fixup**: 与squash类似，但是引commit会merged到上一条commit中，并丢弃当前的commit message。

* **exec**: 可以直接另启一行，执行任意的shell命令。比如你在每个commit后都执行exec run test，确保你的每一步commit都成功跑过test。不过最常用的方式还是：

  ```shell
  git rebase --interactive --exec "cmd" some-ref
  #或者
  git rebase -ix "cmd" some-ref
  ```

### X.拓展阅读

- [LearnGitBranching](https://learngitbranching.js.org/).
- [Git的奇技淫巧](https://github.com/521xueweihan/git-tips).
- [ThinkLikeAGit](http://think-like-a-git.net/tldr.html).
- [ExplainGitWithD3](http://onlywei.github.io/explain-git-with-d3/).
- [KeepGitHistoryCleanUsingRebase](https://blog.theodo.fr/2018/09/keep-git-history-clean-using-rebase/#disqus_thread).
- [WhatAreTheGitConceptsOfHeadMasterOrigin](https://stackoverflow.com/questions/8196544/what-are-the-git-concepts-of-head-master-origin).
