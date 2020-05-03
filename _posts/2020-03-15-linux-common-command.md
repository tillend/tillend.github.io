---
layout:     post
title:      "Linux常用命令"
subtitle:   "netstat/ps/zgrep"
date:       2020-03-15 19:13:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Linux
    - netstat
    - ps
---


## Linux常用命令

### netstat

> netstat命令用于显示网络状态

动作说明：

- `r` ：显示[路由表](https://zh.wikipedia.org/wiki/%E8%B7%AF%E7%94%B1%E8%A1%A8)内容
- `i` ：显示[网络接口](https://zh.wikipedia.org/wiki/%E7%B6%B2%E8%B7%AF%E6%8F%92%E5%BA%A7)及统计信息
- `g` ：显示[多播组](https://zh.wikipedia.org/wiki/%E5%A4%9A%E6%92%AD#IP%E5%A4%9A%E6%92%AD)信息
- `s` ：按网络协议显示统计信息。默认情况下，显示TCP、UDP、ICMP和IP协议的统计信息。
- `n` ：显示活动中的TCP连接，但主机地址和端口号以数字形式表示，不会尝试确定实际主机名
- `p` ：显示哪些进程正在使用哪些网络接口
- `l` ：显示监听服务器socket
- `a` ：显示所有socket(默认为连接中的socket)

显示所有连接中的TCP连接，进程所使用的网络接口情况
```bash
netstat -nap
```

### ps
> ps命令用于显示当前进程 (process) 的状态

动作说明：
- `w`： 显示加宽可以显示较多的资讯
- `e`： 列出所有的进程
- `A`： 列出所有的进程，同`-e`
- `f`： 显示程序间的关系
- `au`： 显示较详细的资讯
- `aux`： 显示所有包含其他使用者的进程

```bash
ps -ef
```

### zgrep & zcat
> `zgrep`命令为避免解压文件，来查找文件里符合条件的字符串
`zgrep`及`zcat`命令均为便于对压缩文件进行操作,原命令的使用详解见[Linux常用命令](https://blog.csdn.net/why_still_confused/article/details/84863795)

模糊搜索(查询文件中包含'abc'的记录)
```bash
zgrep 'abc' <*.tar.gz/*.gz>
```


## 常用场景

### 查看TCP连接的进程
查看连接远程ip端口的进程
```bash
netstat -nap | grep 'ip:port'
ps -ef | grep port
```

---
参考资料：
1. [Linux 命令大全](http://www.runoob.com/linux/linux-command-manual.html)
2. [netstat](https://zh.wikipedia.org/wiki/Netstat)