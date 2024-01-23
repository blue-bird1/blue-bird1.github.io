---
title: 从零开始自定义安卓系统(1) 下载代码
date: 2024-01-18T17:19:55+08:00
lastmod: 2024-01-18T17:19:55+08:00
categories: ["aosp"]
tags: ["android", "aosp"]
description:
---

## 从零开始自定义安卓系统(1) 下载代码

### 前言 

aosp 魔改教程已经有很多了, 但是都比较零碎或者太过古老. 出于记录的想法,写下这些.

本篇教程基于`ubuntu22` 和 `Android 13 ` 和`Redroid`.



### 下载代码

所有教程里都必须拥有的阶段 (导致作者实际上看过很多次)

简单说明步骤 (可以先不执行)

```bash
# 下载repo  
sudo apt-get install repo
# 在项目目录 执行初始化
repo init -u https://android.googlesource.com/platform/manifest --git-lfs --depth=1 -b android-13.0.0_r82
# 同步
repo sync -j8
```



repo实际上做的事情是

1. 从 https://android.googlesource.com/platform/manifest 这个git仓库的`android-13.0.0_r82`分支获取 `default.xml` 这个文件.  具体可用分支列表可以看链接
2. 进行一些操作后写入到当前目录的 `.repo`目录
3. 按xml文件执行git clone和同步, xml文件里描述了目录与git仓库的对应关系.



而如果需要添加自己代码仓库 就可以往`.repo`里写自己的xml文件  [文件格式参考](https://gerrit.googlesource.com/git-repo/+/HEAD/docs/manifest-format.md)

```bash
mkdir .repo/local_manifests
touch .repo/local_manifests/bluebird.xml
```



然后编辑

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
    <remote name="bluebird" fetch="https://github.com/blue-bird1/"/>
    <!-- Our own components -->
    <project path="device/bluebird" name="device"  remote="bluebird" revision="main" />
</manifest>
```



其中

```xml
<remote name="bluebird" fetch="https://github.com/blue-bird1/"  />
```

的fetch为自己的git用户地址 name可以任意修改



```xml
<project path="device/bluebird" name="device"  remote="bluebird" />
```

`path`为相对于项目的相对路径  `remote`需要与上面的 `name`对应 name则是仓库名, 实际上对应的url 为 `https://github.com/blue-bird1/device` 



执行`repo sync`后 可以发现 `device/bluebird` 出现了自己的项目代码.

而`redroid`项目也用了同样的方法实现自定义 `AOSP`

```
# 来自https://github.com/remote-android/redroid-doc/tree/master/android-builder-docker 文档的操作
git clone https://github.com/remote-android/local_manifests.git .repo/local_manifests -b 13.0.0 
```

可以查看目录下的两个`manifest`文件了解做了什么操作 ,主要是添加五个自己的项目到代码里.

如果想要直接对`redroid` 进行fork修改可以直接对此文件进行修改

