---
layout:     post
title:      "Git多账户登录及代理配置"
subtitle:   ""
date:       2019-03-01 22:19:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Git
    - SSH
---

## ssh 秘钥别名登录

在`/etc/ssh/ssh_config` 或 `~/.ssh/config`中输出以下行 （若文件不存在，新建即可）

```
Host dev
    HostName www.ex.com
    User root
```

然后使用`ssh dev`即可登录至远程服务器

> 详见[ssh 秘钥别名登录](https://blog.csdn.net/why_still_confused/article/details/82532254)

## git多账户

同上，在`/etc/ssh/ssh_config` 或 `~/.ssh/config`中配置git多账户登录

```
# 配置github.com
Host github.com                 
    HostName github.com
    IdentityFile C:\\Users\\god\\.ssh\\id_rsa_github
    PreferredAuthentications publickey
    User tillend

# 配置git.oschina.net 
Host git.oschina.net 
    HostName git.oschina.net
    IdentityFile C:\\Users\\god\\.ssh\\id_rsa_oschina
    PreferredAuthentications publickey
    User tillend
```

显示对应账户
```bash
$ ssh -T git@github.com
Hi tillend! You've successfully authenticated, but GitHub does not provide shell access.
```

> 在使用时需要注意，不能设置全局的`username`和`email`

```bash
# 取消全局 username, email
git config --global --unset user.name
git config --global --unset user.email
```


## git代理

#### http代理

命令形式
```bash
git config --global http.https://github.com.proxy socks5://127.0.0.1:1080
git config --global https.https://github.com.proxy socks5://127.0.0.1:1080
```

配置形式
```
Host github.com
    HostName github.com
    User root
    ProxyCommand connect -H 127.0.0.1:1080 %h %p 
    # ProxyCommand connect -S 127.0.0.1:1080 %h %p (socks5)
```

#### ssh代理

配置形式
```
Host github.com
    HostName github.com
    User root
    ProxyCommand nc -v -x 127.0.0.1:1080 %h %p
```

---
参考资料：    
1.[ssh 秘钥别名登录](https://blog.csdn.net/why_still_confused/article/details/82532254)    
2.[Windows Git多账号配置](https://www.cnblogs.com/popfisher/p/5731232.html)