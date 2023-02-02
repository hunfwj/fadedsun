---
title: pprof
date: 2021-05-19 20:42:08
tags: 
- go
- pprof
---

pprof可以用来做程序分析， 包含内存， 堆，goroutine， block，锁等
<!--more-->
起一个简单的http
```
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof"
)

func main() {

	err := http.ListenAndServe("127.0.0.1:80", nil)
	if err != nil {
		fmt.Println(err)
		return
	}
}
```

访问 http://127.0.0.1/debug/pprof/
即可进入web页debug，如下图


![img](https://fadedsun.oss-cn-beijing.aliyuncs.com/img/2021519-21511.png)


点击profile会采集一会儿数据， 然后下载下来一个文件
在命令行执行命令：
```
go tool pprof ~/Downloads/profile
```

执行svg前需要先安装graphviz
```
brew install graphviz
```

进入pprof， 执行
```
svg
```
可以看到生成了一张图， 打开如下：

![img](https://fadedsun.oss-cn-beijing.aliyuncs.com/img/2021519-213336.png)
