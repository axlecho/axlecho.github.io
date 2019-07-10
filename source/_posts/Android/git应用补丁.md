---
title: git应用补丁
date: 2019-01-19 22:30:5
categories: Android
tags:
	- git
---

注意打补丁的路径

检测你的补丁状况
```shell
$ git apply --stat your_fly_sky.patch
```

查看是否能应用成功
```shell
$ git apply --check your_fly_sky.patch
```

应用patch
```shell
$ git am -s < 0001-minor-fix.patch
```
