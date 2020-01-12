---
layout:     post
title:      "git 常用命令"
subtitle:   "merge/rebase/cherry-pick"
date:       2020-01-12 17:13:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Git
    - merge
    - rebase
---

## merge
`git merge` 将已提交的`commit`（自历史记录与当前分支分开以来的提交）合并到当前分支中。

原始分支
```code
	  A---B---C topic
	 /
    D---E---F---G master
```

`checkout`至`master`分支，使用命令`git merge topic`
```code
	  A---B---C topic
	 /         \
    D---E---F---G---H master
```

> `git merge --abort`将中止合并过程，并尝试重建合并前的状态。但是，如果在合并开始时有未提交的更改（尤其是在合并开始后进一步修改了这些更改），则`git merge --abort`在某些情况下将无法重建原始更改。因此警告：**不建议运行`git merge`合并重要的未提交更改**

## rebase

`git rebase`的使用场景
1. 合并多次`commit`为单次`commit`
2. 分支合并


变基的原理
1. 找到这两个分支（即当前分支 topic、变基操作的目标基底分支 master）的最近共同祖先  D
2. 对比当前分支相对于该祖先的历次提交，提取相应的修改并存为临时文件(A+B+C=C')
3. 然后将当前分支指向目标基底 C', 最后以此将之前另存为临时文件的修改依序应用。


原始分支
```code
	  A---B---C topic
	 /
    D---E---F---G master
```

在`topic`分支使用`git rebase master`

```code
	  C' topic
	 /
    D---E---F---G maste
```

`checkout`至`master`分支，使用命令`git merge topic`
```code
    D---E---F---G---C' master
```

*注意：**rebase 会改写历史记录，若该分支的提交已被其他使用者修改时，不建议使用*** 

### 可选操作
> - p, `pick` 保留该commit
> - r, `reword`  保留该commit，但修改注释
> - e, `edit`  保留该commit，但修改提交
> - s, `squash` 保留该commit，将其前一个commit合并
> - f, `fixup` 操作与`squash`相同，但丢弃注释
> - x, `exec` 执行shell命令
> - d, `drop` 丢弃该commit

## cherry-pick
`git-cherry-pick` 能应用(合并)已经存在的commit，即选择合并某个特定commit

原始分支
```code
	  A---B---C topic
	 /
    D---E---F---G master
```

假设commit C的版本号为`7289a5`，在`master`分支使用`git cherry-pick 7289a5`
```code
	  A---B---C topic
	 /
    D---E---F---G---C master
```

---
参考资料：
1. [git merge](https://git-scm.com/docs/git-merge)
2. [git book](https://git-scm.com/book/zh/v2)
3. [Git Community Book 中文版](https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/rebase)
4. [彻底搞懂 Git-Rebase](http://jartto.wang/2018/12/11/git-rebase/)
5. [Git合并特定commits 到另一个分支](https://blog.csdn.net/ybdesire/article/details/42145597)