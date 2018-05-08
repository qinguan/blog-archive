---
title: Git 合并分支多个commits
date: 2018-04-08 14:01:33
tags: 
    - Git
---

多人参与开发的项目不可避免会碰到代码合并的问题。
有人实现一个功能过程中，可能会在自己的分支上提交了很多次，产生了多次提交，如：
```
commit 1：add guava cache
commit 2：use redis
commit 3：add unit test
commit 4：fix bug
```
然后提了一个merge request，这时若直接merge，则在主分支上会新增4条commit记录。
为了维护主分支的干净整齐(尤其是强迫症患者)，可以采用一些方式来合并分支上的多条commit。

```
# 开始开发一个新 feature
$ git checkout -b new-feature master
# 改了一些代码
$ git commit -a -m "add guava cache"
# 改一下实现
$ git commit -a -m "use redis"
$ git commit -a -m "add unit test"
$ git commit -a -m "fix bug"
 
# 紧急修复，直接在 master 分支上改点东西
$ git checkout master
# 改了一些代码
$ git commit -a -m "fix typo"
 
# 开始交互式地 rebase 了
$ git checkout new-feature
$ git rebase -i master

```

进入交互页面，如下：
```
pick 12618c4 add guava cache
pick hde761d use redis
pick iau76h1 add unit test
pick l98ax6d fix bug
 
# Commands:
#  p, pick = use commit
#  r, reword = use commit, but edit the commit message
#  e, edit = use commit, but stop for amending
#  s, squash = use commit, but meld into previous commit
#  f, fixup = like "squash", but discard this commit's log message
#  x, exec = run command (the rest of the line) using shell
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

如上述Commands介绍，这时可以使用squash替换pick。
比如修改成如下：
```
pick 12618c4 add guava cache
squash hde761d use redis
squash iau76h1 add unit test
squash l98ax6d fix bug
```

然后保存，并在主分支上进行merge和push，这样在主分支上就只有一条commit记录了。

参考资料:
--
- [merging-vs-rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)
- [Git tips:合并 commit 保持分支干净整洁](https://www.lovelucy.info/git-tips-combine-commits-keep-your-branch-clean.html)