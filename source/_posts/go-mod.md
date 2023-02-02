---
title: go的mod命令理解
date: 2021-04-11 19:53:51
tags: go mod
---

```
go mod init example.com/mymodule
```

<!-- more -->
使用上述命令， 实际上只是在当前目录创建了go.mod文件,文件内容如下：
```
module example.com/mymodule

go 1.14
```

分别指明了模块名， 和go版本

执行go get命令获取依赖

```
go get github.com/beego/beego/v2@v2.0.0
```

可以看到只是在文件中添加了require这一行， 和java的gradle类似
```
module example.com/mymodule

go 1.14

require github.com/beego/beego/v2 v2.0.0 // indirect
```


```
go mod download -x
```
使用download下载依赖， -x参数可以输出执行详情
```
# get https://go-mod-proxy..org/gopkg.in/mgo.v2/@v/v2.0.0-20190816093944-a6b53ec6cb22.zip: 200 OK (0.667s)
```


有上述佐证， 那么我们可以得知go获取依赖是通过http协议

那么在公司内， 我们经常需要依赖私有库，一般私有库都需要ssh验证

ssh有公私钥验证， kerberos验证


如果我们依赖里有私有库
那么可能就会爆 unknown revision xxx

解决方案

开启模块
```
go env -w GO111MODULE=on
```

设置哪些包是私有
```
go env -w GOPRIVATE="xxx.xxx.com"
```

在.gitconfig文件中添加下面的行，表示用git获取依赖的时候， 将协议替换成http
```
[url "ssh://username@xxx.xxx.com/"]
    insteadOf = https://xx.xxx.com/
[url "git@xxx.xxx.com:"]
    insteadOf = https://xxx.xxx.com/
```
最后记得配置ssh密钥，
如果使用了kerberos验证，记得执行kinit命令

因为kerberos验证优先于密钥， 即使有密钥， 配置了kerberos，密钥也可能不起作用

安装这个可以可视化查看项目依赖
```
go get -u github.com/PaulXu-cn/go-mod-graph-chart/gmchart
```

使用
```
go mod graph | gmchart
```

原文链接： https://segmentfault.com/a/1190000038897207