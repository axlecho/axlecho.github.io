---
title: repo的强制更新
date: 2019-01-12 14:11:17
tags: android
---

repo是git仓库管理工具，一般用`repo sync`去更新代码，实质上是在每个仓库下执行`git pull`,这样就比较蛋疼了，当你本地有提交的话它会自动帮你merge进去，还装做一切都ok的样子。在被坑了数遍之后，终于发现了这个问题。

彻底的同步服务器代码

```shell
$ repo sync -d
# Remove all working directory (and staged) changes.
$ repo forall -c 'git reset --hard'   
# Clean untracked files
$ repo forall -c 'git clean -f -d'
```