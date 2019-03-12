---
layout:     post
title:      "Linux 查看及修改时区"
subtitle:   "Linux Timezone"
date:       2019-03-12 22:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Linux
    - Timezone
---

## 查看时间及时区

查看当前时间
```bash
date
```

查看时区
```bash
cat /etc/timezone
```

## 修改时区

1.修改或设置Linux服务器时区
```bash
tzselect
```

RedHat Linux/CentOS
```bash
timeconfig
```

Debian
```bash
dpkg-reconfigure tzdata
```

2.设置环境变量
```bash
echo "export TZ='Asia/Shanghai'"  >> /etc/profile
source /etc/profile
```

3.创建软链接
```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```


> 使用`timedatectl`命令    
> ```bash
> timedatectl set-timezone Asia/Shanghai
> ```

## 硬件时间

#### 查看硬件时间

```bash
hwclock  --show
```

或

```bash
clock  --show
```

#### 设置硬件时间

```bash
hwclock --set --date="19/03/12 21:55"
```

或

```bash
clock --set --date="19/03/12 21:55"
```

#### 同步系统及硬件时钟

> `hc`代表硬件时间，`sys`代表系统时间，`systohc`表示以系统时间为基准，同步至硬件时间

```bash
hwclock --systohc
```

或

```bash
clock --systohc
```

