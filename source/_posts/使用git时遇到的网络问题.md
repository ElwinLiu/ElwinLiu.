---
title: Git使用时的网络问题
date: 2023-01-15 13:25:00
author: Elwin
abbrlink: 213245 # 自己可随意设置
summary: 
categories: 
  - Tech
tags:
  - Tech
  - Network
comments: true
---

# 网络连接问题

在使用git deploy, pull, push等命令的过程中，电脑没开vpn可能连不上github最后报错；如果电脑开了vpn（以机场举例）仍然有这种情况出现。如下图所示

<!--more-->

![image-20230114173520107](https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/image-20230114173520107.png)

在使用了网上的方法后，依然报错

![image-20230114173550187](https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/image-20230114173550187.png)

网上的方法大致是先用`git config --global -l`发现自己用过了网络代理，于是通过`git config --global http.proxy http://127.0.0.1:1080`命令神奇的修复了错误，但这个方法于我无效。

后来在一个博客下的评论中发现，`git config --global http.proxy http://127.0.0.1:1080`中的1080代表vpn的端口。因此在查找到我的vpn端口号以后，把http和https代理均改为对应的端口号，问题解决。

```GIT
git config --global http.proxy http://127.0.0.1:{{vpn port}}
git config --global https.proxy http://127.0.0.1:{{vpn port}}
```

# ssh失效

挂了机场vpn后，会出现代理服务器对防火墙产生干扰的情况，导致防火墙拒绝了ssh连接。在进行git操作的时候会出现下面的问题

```bash
git@github.com's password:
Permission denied, please try again.
```

要测试通过 HTTPS 端口的 SSH 是否可行，请运行以下 SSH 命令：

```bash
$ ssh -T -p 443 git@ssh.github.com
> Hi USERNAME! You've successfully authenticated, but GitHub does not
> provide shell access.
```

如果这样有效，万事大吉！

接下来启用通过https的ssh连接。如果你能在端口 443 上通过 SSH 连接到 `git@ssh.github.com`，则可覆盖你的 SSH 设置来强制与 GitHub.com 的任何连接均通过该服务器和端口运行。

要在 SSH 配置文件中设置此行为，请在 `~/.ssh/config` 编辑该文件，并添加以下部分：

```bash
Host github.com
Hostname ssh.github.com
Port 443
User git
```

接下来测试是否生效

```bash
$ ssh -T git@github.com
> Hi USERNAME! You've successfully authenticated, but GitHub does not
> provide shell access.
```

最后总结一下，遇到bug太久找不到了，最好还是直接去看官方文档，大部分问题都可以在文档中找到：[在 HTTPS 端口使用 SSH - GitHub Docs](https://docs.github.com/zh/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)
