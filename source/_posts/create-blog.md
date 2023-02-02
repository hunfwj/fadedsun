---
title: 如何搭建公网博客
date: 2021-03-23 00:49:10
tags: blog
---

购买一台服务器
<!-- more -->
系统选择 Centos7.6
选择无限制月流量的.

yum 是一个软件包管理器
yum update

安装 docker
yum install docker

启动 docker 服务
systemctl start docker

查看帮助
systemctl --help

验证 docker 是否启动
docker ps

启动 nginx 容器, 并暴露 80 端口
docker run --name some-nginx -d -p 80:80 nginx

访问测试是否成功, 此处 ip 地址为服务器公网地址, 请在控制台寻找
http://82.156.47.125/

如果访问不通:
ping ip
telnet ip 80

先查看 ip 是否可通, 再验证端口

如果不 ok, 请先在腾讯云控制台配置安全组, 阿里云类似, 放通 icmp, tcp, ssh, http, https 等端口

停掉 nginx, 注意替换 xxx
docker stop xxx

使用 docker 启动一个 halo 博客网站
docker run -it -d --name halo -p 80:8090 -v ~/.halo:/root/.halo --restart=always halohub/halo

访问http://82.156.47.125/

emmm, 感觉不太友好

或者使用 hexo
