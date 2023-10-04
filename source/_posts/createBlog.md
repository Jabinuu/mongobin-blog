---
title: 从0开始在云服务器上搭建Hexo博客
date: 2023-03-20 23:23:01
tags: [Hexo,云服务器, Git]
categories: [Hexo博客]
index_img: http://182.44.49.100:34/images/2023/05/06/createBlog.png
banner_img: http://182.44.49.100:34/images/2023/05/06/createBlog.png
comment: true
---

### 前言

​	本来博客是用wordpress在服务器搭建的，但苦于wordpress的后端语言是php，对markdown的支持也不尽如人意，最终还是放弃了wordpress，转战后端为Node的Hexo框架。

​	**整体思路：**

1. 在服务器上配置Git环境，创建Git仓库
2. 在主机安装Hexo，并生成Hexo静态文件，通过与服务器链接，将静态文件推送到服务器上的Git仓库
3. 通过Git-hooks，即Git钩子，实现将服务器Git仓库的文件自动部署到网页资源目录
4. 将Nginx作为静态文件服务器，实现外界对网页资源目录的访问。

​	**本文的配置环境为**

 + 天翼云服务器：[宝塔](https://www.bt.cn/new/index.html)面板，一键安装nginx

   > 没有宝塔可以用ssh链接服务器，敲命令行也是一样的`yum install nginx`

 + 本地主机：Git、Node.js、Hexo

   > Hexo安装：`npm install hexp -g ` 。`-g`意为全局安装。
   >
   > 如果第一次安装node，请注意配置环境变量，否则会出现`hexo不是内部或外部命令`的问题。

### 1. 在服务器安装Git

​	不管是宝塔提供的终端，还是Xshell的命令行都可以，安装命令`yum install git`。

> 安装git可能会出现这样的报错信息
>
> Loaded plugins: fastestmirror, langpacks
>
> Loading mirror speeds from cached hostfile
>
> No package yum-util available.
>
> Error: Nothing to do
>
> 解决方法可参考：[安装docker时，遇到Loaded plugins...怎么办](https://blog.csdn.net/weixin_51225684/article/details/128040380)

### 2.在宝塔面板添加站点

​	由于天翼云服务器在域名没有备案的情况下不开放80端口，所以手动设置一个空闲的32端口用于访问网页。



![image-20230320213201932](http://182.44.49.100:34/images/2023/05/06/image-20230320213201932.png)

​	将网站目录设置为如下（自定义即可）



![image-20230320213344566](http://182.44.49.100:34/images/2023/05/06/image-20230320213344566.png)

### 3.对服务器的Git进行搭建

#### 1. 添加一个git用户

```shell
adduser git       # 添加git用户
chmod 740 /etc/sudoers    #改变sudoers文件的权限为文件所有者可写
vim /etc/sudoers
#在root ALL=(ALL) ALL下方添加一行,按esc,再按:wq退出编辑
git ALL=(ALL) ALL
chmod 400 /etc/sudoers #将sudoers文件的权限改回文件所有者可读

sudo passwd git   #设置服务器的git密码，用于git连接。输入时看不到任何显示，输入完成回车即可
```

#### 2. 给服务器和主机的Git配置SSH密钥

​		如果**主机**已有ssh密钥则跳过这一步，直接到`C:\Users\你的用户名\.ssh`中找到`id_rsa.pub`。如果没有，按照如下步骤生成密钥：

```sh
git config --global user.name "你要设置的名字"
git config --global user.email "你要设置的邮箱"
ssh-keygen -t rsa -C "你刚刚设置的邮箱"
```

​		此时**主机**的git密钥已生成，存放在上述`id_rsa.pub`文件中。接着，打开宝塔的文件管理系统，在**服务器**的`/home/git`中新建`.ssh`文件夹，并在其中新建`authorized_keys`文件。将**主机**的`id_rsa.pub`中的内容复制到该新建文件中。



![image-20230320211205195](http://182.44.49.100:34/images/2023/05/06/image-20230320211205195.png)

​		通过配置ssh密钥，主机和服务器的git连接时将不再需要密码，简化了操作。

#### 3.在服务器中创建一个新的Git仓库

```sh
cd /home/git
git init --bare hexoblog.git  #在/home/git下初始化一个名为hexoblog的仓库
```

#### 4. 配置钩子实现自动部署

​	找到`/home/git/hexoblog.git/hooks`下的`post-receive`文件，如果没有则新建一个该文件，将其内容改为

``` sh
#!/bin/sh
git --work-tree=/home/www/mongobin --git-dir=/home/git/hexoblog.git checkout -f

```

​	以上内容是一条命令，前者为网页资源目录，后者为git仓库。意为当主机将静态文件推给服务器的git仓库后，服务器能够自动将文件部署到网页资源目录。

​	**然后设置网页资源目录的IO权限，否则git没有权限修改网页资源目录的内容，无法实现自动部署！！！**

``` sh
sudo chmod +x /home/git/hexoblog.git/hooks/pre-receive  #赋予其可执行权限
sudo chown -R git:git /home/git/ #仓库目录的所有者改为git
sudo chown -R git:git /home/www/ #站点文件夹所有者改为git
```

### 4. 主机配置与测试

#### 1. 在主机初始化博客文件夹并测试本地demo

​	执行以下命令在文件夹中创建一个新的博客文件夹（官方demo）。

```sh
cd D:\JaBinsProjects\mongobin
hexo init
```

**然后安装两个插件，用于部署，否则会报错！**

``` sh
npm install hexo-deployer-git --save
npm install hexo-server
```

​	执行以下命令即可在本机上查看自己的博客了，地址为<localhost:4000>

``` 
hexo g
hexo s
```



![image-20230320215235929](http://182.44.49.100:34/images/2023/05/06/image-20230320215235929.png)

#### 2.配置本地博客与服务器git的连接

​	在刚才生成的博客文件夹根目录中找到并打开`_config.yml`文件，把最下面的depoly处改为如下内容，目的是与服务器git仓库建立连接。

![image-20230320220031029](http://182.44.49.100:34/images/2023/05/06/image-20230320220031029.png)

​	**注意：**

+ type, repo, branch缩进2格
+ 冒号与其后面的内容必须有一空格
+ branch为master和main均可

#### 3. 测试连接和自动部署是否生效

cd到**博客的文件夹下执行**以下命令

``` sh
hexo new "Hello My First Blog"
hexo clean && hexo generate --deploy
```

也可以在`package.json`中添加`npm`脚本，简化操作，这样可以直接用`npm run dd`部署博客网页

``` json
"scripts": {
    "build": "hexo generate",   // 重新生成静态页面文件
    "clean": "hexo clean",    // 清除缓存
    "deploy": "hexo deploy",   // 将静态页面文件部署到服务器
	"dd": "hexo clean && hexo g -d",
    "server": "hexo server",
	"ss": "hexo clean && hexo g && hexo s"
  },

```

然后输入域名`www.mongobin.top:32`看博客是否更新了一篇文章。

### 5.最后

​	完成部署后可以去[Hexo](https://hexo.io/themes/)主题下载自己喜欢的主题，美化博客。现在去本地主机浏览器上输入域名或者公网IP，访问你的博客吧！
样例博客：[唐志远の博客](https://tzy1997.com/)

### 参考文章

+ [将Hexo部署到云服务器（使用宝塔面板）](https://blog.csdn.net/qq_43219561/article/details/116719535)

+ [基于云服务器的hexo博客搭建（稳）](https://blog.csdn.net/weixin_56301399/article/details/129270887)

+ [Hexo博客部署至服务器](https://blog.csdn.net/u013190417/article/details/122694959)

  
