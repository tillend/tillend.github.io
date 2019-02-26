---
layout:     post
title:      "Linux 用户组及文件权限详解"
subtitle:   "chmod"
date:       2019-02-26 21:19:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Linux
    - chmod
---

Linux系统中的每个文件和目录都有访问许可权限，本文内容为用户组、组命令及文件权限详解

## 用户组

Linux 系统中的目录和文件的访问身份分为`user`, `group`,`others`,`all`,分别简写为`u`,`g`,`o`,`a`;

- user 是文件的所有者
- group 是文件所有者所在组的其他成员
- others 就是不在所有者的所在组的其他用户
- all 代表所有用户

> 当某个用户创建了一个文件后，这个文件的所有者为该用户、所在组则为该用户所在的组(直至所有者或所在组被修改)

## 组命令

修改文件属性的常用命令

- [chgrp](http://www.runoob.com/linux/linux-comm-chgrp.html)
- [chown](http://www.runoob.com/linux/linux-comm-chown.html)
- [chmod](http://www.runoob.com/linux/linux-comm-chmod.html)
- [umask](http://www.runoob.com/linux/linux-comm-umask.html)

#### chgrp

> `chgrp`命令用于变更文件或目录的所属群组

> chgrp [OPTION]... GROUP FILE... 
> chgrp [参数]... [所在组] 文件或目录

将文件的所在组修改为`www`

```bash
chgrp www install.sh
```

#### chown

> `chown`命令用于改变文件拥有者    

> chown [OPTION]... [OWNER][:[GROUP]] FILE...    
> chown [参数]... [所有者][:[所在组]] 文件或目录

将文件的所有者及所在组修改为`www`

```bash
chown www:www install.sh
```

#### chmod

> Linux/Unix 的文件调用权限分为三级: 文件拥有者、群组、其他。利用`chmod`可以藉以控制文件如何被他人所调用。    
> 使用权限: 所有使用者    

> chmod [OPTION]... MODE[,MODE]... FILE...    
> chown [参数]... [模式][,模式] 文件或目录

详细操作见下方[文件权限]()

#### umask

> `umask`可用来设定[权限掩码]。[权限掩码]是由3个八进制的数字所组成，将现有的存取权限减掉权限掩码后，即可产生建立文件时预设的权限。

> umask [-S][权限掩码]

当前的存取权限
```bash
umask -S
u=rwx,g=rwx,o=rx
```

> umask为002，减掉权限掩码后，当前的存取权限即为775(`u=rwx,g=rwx,o=rx`)

## 文件权限

Linux系统是一种典型的多用户系统，不同的用户处于不同的地位，拥有不同的权限。为了保护系统的安全性，Linux系统对不同的用户访问同一文件（包括目录文件）的权限做了不同的规定。

在Linux中我们可以使用`ll`或者`ls –l`命令来显示一个文件的属性以及文件所属的用户和组，如：

```bash
[root@www /]# ls -l
total 64
dr-xr-xr-x   2 root root 4096 Dec 14  2018 bin
dr-xr-xr-x   4 root root 4096 Apr 19  2018 boot
```

![](/img/in-post/post-2019-02/authority.png)

- 第一个字符代表这个文件的类型(如目录、文件或链接文件等等)：
    - 当为[`d`]则是目录,例如上表档名为『.gconf』的那一行；
    - 当为[`-`]则是文件,例如上表档名为『install.log』那一行；
    - 若是[`l`]则表示为连结档(link file)；
    - 若是[`b`]则表示为装置文件里面的可供储存的接口设备(可随机存取装置)；
    - 若是[`c`]则表示为装置文件里面的串行端口设备,例如键盘、鼠标(一次性读取装置)
- 接下来的字符中,以三个为一组,且均为『`rwx`』的三个参数的组合
([`r`]代表可读(read)、[`w`]代表可写(write)、[`x`]代表可执行(execute) 要注意的是,这三个权限的位置不会改变,如果没有权限,就会出现减号[`-`]而已)
    - 第一组为『文件拥有者的权限』
    - 第二组为『同群组的权限』
    - 第三组为『其他非本群组的权限』

### 修改文件权限

修改文件的权限用命令`chmod`来执行 , 有两种权限定义方式: 数据方式和符号方式

#### 数字方式修改文件权限

> read(r):4、write(w):2、execute(x):1

权限设置为
- 所有者可读可写可以执行
- 同组可读可执行不可写
- 其他组的用户可读

此时符号权限为`-rwxr-xr--`，则权限的分数应为[4+2+1][4+0+1][4+0+0]=754
```bash
chmod 754 install.sh
```

#### 符号方式修改文件权限

![](/img/in-post/post-2019-02/chmod.png)

权限设置为`-rwxr-xr--`，同上数字方式
```bash
chmod u=rwx,g=rx,o=r install.sh
```

其他组用户删除可读权限
```bash
chmod o-r install.sh
```

所有用户增加执行权限
```bash
chmod a+x install.sh
```


#### 特殊权限SUID, SGID, SBIT:


---
参考资料：    
1.[linux文件属性与权限](https://www.cnblogs.com/kzloser/articles/2673790.html)
2.[Linux 文件基本属性](http://www.runoob.com/linux/linux-file-attr-permission.html)