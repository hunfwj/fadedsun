---
title: goland编辑器设置
date: 2021-04-13 16:47:54
tags: goland
---

双击shift唤出搜索框，输入下面搜索， 打开开关， 即可在右下角看到内存使用情况。
<!-- more -->
```
show memory indicator
```

如果使用内存接近满值，就要考虑优化了。
idea系列产品是使用java开发的， 那么我们可以使用jvm调优的思路来设置ide编辑器。

首先设置内存大小
打开Help -》 Edit Custom VM Options，修改下面的参数
```
-Xms2048m
-Xmx2048m
-XX:ReservedCodeCacheSize=1024m
```

依次为最小内存， 最大内存， 代码缓存


在设置里打开下面的设置， 用来发现包
Go =》Go Modules =》 Enable Go modules integration