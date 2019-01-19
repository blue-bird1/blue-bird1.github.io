---
title: docker搭建复现环境
date: 2019-01-19T22:52:13+08:00
lastmod: 2019-01-19T22:52:13+08:00
categories: ["docker"]
description:
---

### docker搭建复现环境

安全人员进行漏洞复现经常需要搭建漏洞环境, docker能够很方便搭建漏洞环境,同时提供相当好的性能,管理功能.

docker安装请参照空格表哥的[这篇文章](https://bbs.ichunqiu.com/forum.php?mod=viewthread&tid=47689&highlight=docker) 

搭建漏洞环境也有几种情况  

- 已经有现成的docker镜像

- 只有源代码压缩包

- 漏洞软件开源

  

#### 寻找已有docker镜像

dockerhub是官方的镜像仓库, 提供免费的公开镜像储存,也支持搜索   [网站url](https://hub.docker.com/) 

可以直接搜索镜像 `https://hub.docker.com/search?q=<keyword>&type=image`

docker命令行也支持搜索镜像 命令格式 `docker search "keyword" `

例如 `docker search "think5"`



#### 从源代码搭建镜像

以zzzphp为例. [github地址](https://github.com/Earth-Online/poc_test/tree/master/zzzphp) 



##### Dockerfile知识快速普及

dockerfile是一个描述建造镜像流程的文件 每一行格式 

`<关键字> <n个参数>`  

搭建漏洞环境常用的几个关键字 

`FROM 镜像名 `  说明是从哪个镜像开始建造 

`COPY 本地路径 镜像路径`  拷贝文件到镜像里

`ENV 变量名 变量值` 定义镜像内的环境变量

`RUN 命令` 在镜像运行一条命令

`WORKDIR 路径`   更改之后像`RUN`这类执行命令的路径



##### 构建镜像

我挑选`webdevops/php-apache-dev:ubuntu-15.10`作为基础镜像, 这个镜像提供了非常快捷的搭建方法.

```dockerfile
FROM webdevops/php-apache-dev:ubuntu-15.10
COPY --chown=application:application .  /var/www/html
ENV WEB_DOCUMENT_ROOT /var/www/html
```

 只需要三行就能搭建出一个php环境

第二行 `COPY --chown=application:application .  /var/www/html` 拷贝当前目录到`/var/www/html` 并更改所有者成`application:application` .

`application:application` 是这个镜像的服务器用户, 更改文件权限否则服务器不能读写文件

`ENV WEB_DOCUMENT_ROOT /var/www/html` 

 `WEB_DOCUMENT_ROOT`是这个镜像的一个特殊环境变量 指向web目录



使用 `docker build -t site . ` 建造镜像和命名为`site`



#### 从git构建镜像

和从源代码构建的区别只是在于一个用`COPY` 一个在容器直接用`RUN`执行命令. 

例子

```dockerfile
FROM webdevops/php-apache-dev:ubuntu-15.10
RUN git clone https://github.com/DaBoQuan/cmseasy_decode /var/www/cmseasy && chown -R application:application /var/www/cmseasy
ENV WEB_DOCUMENT_ROOT /var/www/cmseasy
```

只是简单的`git clone`到web目录和 `chown -R`更改文件权限



#### 运行和数据库

用docker运行很简单`docker run -d  -p 外部端口:镜像端口 <镜像名>` 

如果运行site镜像就是 `docker run -d  -p 80:80 site`

一个网站当然需要一个数据库  如果你的服务器有一个外部ip 那么很简单

`docker run --name mysql -e MYSQL_ROOT_PASSWORD=password -p 3306:3306  mysql:5.6`

填数据库地址的时候填外部ip即可

如果没有 可以用`docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mysql` 获取docker内部ip 填写内部ip即可

