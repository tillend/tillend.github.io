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