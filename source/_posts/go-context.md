---
title: go-context
date: 2021-06-04 16:52:56
tags:
---

在go语言的官方包中， context是一个接口
```
type Context interface {
	Deadline() (deadline time.Time, ok bool)

	Done() <-chan struct{}

	Err() error

	Value(key interface{}) interface{}
}
```
去掉注释， 我们可以看到只有四个方法。

