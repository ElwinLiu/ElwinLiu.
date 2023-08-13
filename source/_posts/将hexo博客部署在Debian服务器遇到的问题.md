---
title: hexo+阿里云+个人域名
date: 2023-01-19 18:39:00
author: Elwin
abbrlink: 213245 # 自己可随意设置
cover: https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/5ef2c2cc5f8be9322bc998d1fedf3b92.png
summary: 	记录一下把博客部署到服务器上的步骤以及遇到的一些问题，或许将来能用到。
categories: 
  - Tech
tags:
  - Tech
  - Deploy
comments: true
---

​	主要分为两个章节，一个是nginx+hexo+git，将静态页面部署到服务器的git仓库中，并通过hooks把页面资源/home/hexo目录下，nginx通过域名-页面资源的映射索引到目标文件；一个是域名问题，其中涉及到ssl证书、域名映射发生冲突（本质上是对nginx各配置文件的原理的理解不够）。

<!--more-->

# 部署过程（Debian/ Ubuntu）

## 1. 安装

首先安装nginx & git

```bash
sudo apt-get update
sudo apt-get install git nginx -y
```

测试git是否安装成功

```bash
git --version
```

测试nginx是否安装成功

```bash
systemctl start nginx.service    # 启动服务
```

去浏览器输入服务器的公网ip,看见如下图类似的画面则已经安装成功

![img](https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/ae7b7f80be34755e72b1f455c17fdc91.png)

## 2. 创建git用户

（用root？太不明智了吧，还是另外创建一个比较好！）

创建git用户

```bash
sudo adduser git  # 创建git用户
sudo passwd 123456 # 设置git用户的密码
```

给git用户权限

```bash
sudo chmod 740 /etc/sudoers   #(该文件为只读，想要增加内容必须增加权限)
sudo vim /etc/sudoers
```

找到 `root ALL=(ALL:ALL) ALL`在其下面添加 `git ALL=(ALL:ALL) ALL`

修改完成后复原权限

```bash
chmod 400 /etc/sudoers
```

## 3. 存主机的公钥

```bash
su git
mkdir ~/.ssh
vim ~/.ssh/authorized_keys
```

将本地主机的`id_rsa.pub`内容复制到`authorized_keys`中即可

```bash
# 在本地主机上打开终端，通过ssh登入验证是否绑定成功，若不需要输入密码则为成功
ssh git@{{internet ip}}
```

修改这里的权限

```bash
cd ~
chmod 600 .ssh/authorzied_keys
chmod 700 .ssh
```

## 4. 建立网站的根目录

```bash
su root  # 切换到root用户
mkdir /home/hexo  # 作为网站根目录
chown git:git -R /home/hexo # 给git用户权限
```

## 5. 配置Nginx

```bash
vim /etc/nginx/sites-available/default    # 编辑配置
```

修改内容

```bash
server {
    listen       80 default_server;
    listen       [::]:80 default_server;
    # server_name  www.yours_server_name.com;    # 修改为自己的域名
    root         /home/hexo;    # 修改为网站的根目录
    index index.html index.php index.htm;
    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```

使用`nginx -t`命令检查配置文件的语法是否出错。然后使用`systemctl restart nginx.service` 命令重启服务即可。

## 6. 初始化git仓库

```bash
su root
cd /home/git   # 在 git 用户目录下创建
git init --bare blog.git   # 一定要加上--bare
chown git:git -R blog.git   #赋予git用户权限
```

这里使用的是`post-receive`这个钩子，当git有收发的时候就会调用这个钩子。 在`blog.git`裸库的 `hooks`文件夹中，新建`post-receive`文件。

```bash
vim blog.git/hooks/post-receive
```

在这个空白文件中加入以下语句

```bash
#!/bin/sh
git --work-tree=/home/hexo --git-dir=/home/git/blog.git checkout -f
```

添加权限，保证hooks能起作用

```bash
chmod +x /home/git/blog.git/hooks/post-receive
```

## 7. 本地配置

在hexo的根目录中打开`_config.yml`文件。部署到多个平台的方法如下，branch缺省为master

```yaml
# Deployment
## 详情参考文档: https://hexo.io/docs/one-command-deployment
deploy:
- type: git
  repo: https://github.com/ElwinLiu/ElwinLiu.github.io.git
  branch: main
- type: git
  repo: git@47.***.***.***:/home/git/blog.git
```

完事就可以尝试部署了，若还有问题，请见“遇到的问题”

# 在nginx服务器上部署域名证书

## 1. 下载域名的证书

![image-20230119201712201](https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/image-20230119201712201.png)

![image-20230119201722414](https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/image-20230119201722414.png)

## 2. 编辑 Nginx 根目录下的 `nginx.conf` 文件。修改内容如下：

> 如找不到以下内容，可以手动添加。可执行命令 nginx -t ，找到nginx的配置文件路径。
>
> 如下图示例：
>
> ![tapd_10132091_base64_1665978617_57.png](https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/b772c59a0bdd1d7bfc97a2320cb95df7.png)
>
> 此操作可通过执行 `vim /etc/nginx/nginx.conf` 命令行编辑该文件。
>
> 由于版本问题，配置文件可能存在不同的写法。例如：Nginx 版本为 `nginx/1.15.0` 以上请使用 `listen 443 ssl` 代替 `listen 443` 和 `ssl on`。

**请仔细阅读注释！**

```yaml
server {
     #SSL 默认访问端口号为 443
     listen 443 ssl; 
     #请填写绑定证书的域名
     server_name cloud.tencent.com; 
     #请填写证书文件的相对路径或绝对路径
     ssl_certificate {{}}.crt; 
     #请填写私钥文件的相对路径或绝对路径
     ssl_certificate_key {{}}.key; 
     ssl_session_timeout 5m;
     #请按照以下协议配置
     ssl_protocols TLSv1.2 TLSv1.3; 
     #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
     ssl_prefer_server_ciphers on;
     location / {
         #网站主页路径。此路径仅供参考，具体请您按照实际目录操作。
         #例如，您的网站主页在 Nginx 服务器的 /etc/www 目录下，则请修改 root 后面的 html 为 /etc/www。
         root html; 
         index  index.html index.htm;
     }
 }
```

## 3. 重载nginx

通过执行以下命令验证配置文件问题。

```bash
nginx -t
```

通过执行以下命令重载 Nginx。

```bash
nginx -s reload
```

成功以后可以访问`https://{{your domain}}`验证

## 4. http自动跳转为https（可选）

在**部署过程**的第5点中编辑的配置文件里加跳转语句即可

```bash
vim /etc/nginx/sites-available/default    # 编辑配置
```

在    `root         /home/hexo;    # 修改为网站的根目录`语句下方（只要是server花括号都可以）加以下语句

```yaml
rewrite ^(.*)$ https://$host$1; #将所有HTTP请求通过rewrite指令重定向到HTTPS。
```

返回第3点，重载nginx。

成功以后可以访问`http://{{your domain}}`验证，观察是否跳转到https页面

# 遇到的问题

## 1. server name端口冲突

修改nginx配置参数后，使用nginx -t检查配置.

提示successfull后就可以使用 nginx -s reload来重新加载配置

我配置的过程中遇到这样的问题，就是绑定了主机名后，重新加载配置时会出现警告

```
nginx: [warn] conflicting server name "localhost" on 0.0.0.0:80, ignored
```

问题根源：在server块中填写配置信息时，每个server_name（域名）只能绑定一个目录。如果出现了在不同的server块中使用了同一个server_name，且绑定了不同目录的情况，就会报上述警告。

该问题不会影响服务器正常运作，但是会出现其中一个被ignored的server块的配置功能**无效**的情况。

## 2. 访问https域名出现nginx初始化页面

在写`nginx.conf`配置文件时，没有修改location块中root后面的根目录路径，此处我使用的是绝对路径`/home/hexo`

## 3. 本地hexo d没有推送至服务器git仓库

**问题分析：**

这个问题出现的时候你会发现访问自己的域名时没有加载页面元素，空空如也。去hexo目录检查也发现是空的，hooks的配置也没有问题，问题就只能是在deploy时出现的。在初始化的时候我们没有创建分支，git仓库默认分支为master，而有些人习惯使用main分支进行推送，导致推送不成功。

**解决方法：**

把_config.yml文件中deploy块里branch删掉，或者写成master即可。

## 4. 403 forbidden，nginx无法正常显示

**问题分析：**

一般来讲，即使hexo静态资源推送失败，nginx也至少会把index.html放在网页上，而不是403 forbidden，除非nginx啥也索引不到。通过查看nginx的错误日志，发现了`481 directory index of "/home/hexo/" is forbidden`

**解决方法：**

去nginx根目录下，配置文件`nginx.conf` 加上配置 `autoindex on;`自动索引

![img](https://elwinliu-blog-bucket.oss-cn-hangzhou.aliyuncs.com/elwinliu-blog/1364760-20220111173146165-740927078.png)