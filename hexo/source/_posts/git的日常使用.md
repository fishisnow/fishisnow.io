---
title: git的日常使用
date: 2017-08-01 21:27:24
tags: [git]
---
### 注意事项
1. 分支开发的时候记得及时合并主分支代码，避免出现冲突
2. 不要将两个分支的代码都合并到同一个分支<!--more-->
### 删除本地分支
git branch -D br
### 重命名本地分支
git branch -m  oldbranchname newname
### 查看stash工作现场
git stash list 查看stash列表
git stash　pop将最上面的工作现场恢复
git stash show 查看工作现场的文件
注意：工作现场是一个栈，跟分支没有对应关系，在不同分支要注意恢复的现场编号，不能盲目使用pop命令
### 修改commit message
如果没有push到服务器，只是本地进行了commit,并没有进行新的commit:git commit --amend
git reset HARD 取消最后一次提交
### 新建远程分支关联分支
git branch --set-upstream debug origin/debug 
