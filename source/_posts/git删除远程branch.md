---
title: git删除远程branch
date: 2018-01-12 17:44:08
tags:
	- Git
category: Git
comments: true
---


##### 删除远端分支
```bash
git push origin --delete <branch>	# Git version 1.7.0 or newser
git push origin :<branch>		# Git versions older than 1.7.0
```

<!-- more -->

##### 删除本地分支
```bash
git branch --delete <branch>
git branch -d <branch>	# Shorter version
git branch -D <branch>	# Force to delete un-merged branch
```

##### 删除本地remote-tracking分支
```bash
git branch --delete --remotes <remote/branch>
git branch -dr <remote/branch>	# Shorter

git fetch <remote> --prune	# Delete multiple obsolete tracking branches
git fetch <remote> -p 		# Shorter
```

##### 简单的讲
当你需要删除本地或远端分支时，记住这里`三个不同的分支`。

1. 本地分支`X`
2. 远端分支`X`
3. 本地remote-tracking分支`origin/X`，用于追踪远端分支`X`

![](/img/git_origin.png "")


删除remote-tracking分支
```bash
git branch -dr origin/bugfix
```

![](/img/git_delete_remote_tracking.png "")

删除remote分支
```bash
git push origin --delete bugfix
```

![](/img/git_delete_remote.png "")