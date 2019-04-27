---
title: 在golang项目开始使用Bazel 
date: 2019-04-27T19:32:12+08:00
lastmod: 2019-04-27T19:32:12+08:00
categories: ["开发工具"]
description:
---

# 在golang项目开始使用Bazel

`Bazel`是一个由java编写的编译工具,支持多语言编译,扩展,远程缓存等大量功能.



## 下载

推荐通过`https://github.com/bazelbuild/bazel/releases`下载,

`wget https://github.com/bazelbuild/bazel/releases/download/0.24.1/bazel-0.24.1-installer-linux-x86_64.sh`

由于大家都知道的原因,下载速度很慢.建议在国外服务器下载



## 快速开始

`Bazel`使用两个特殊文件名 `WORKSPACE` 和`BUILD`定义项目. 

`WORKSPACE`定义一个项目工作区,例如外部依赖和工具链,`BUILD`定义如何编译这个项目.允许用多个`BUILD`定义不同部分的编译操作.



在项目创建一个`WORKSPACE`文件.然后需要载入Bazel的golang扩展 .以下均为官方基础例子

```python
# 载入http_archive函数
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# 从github下载扩展
http_archive(
    name = "io_bazel_rules_go",
    urls = ["https://github.com/bazelbuild/rules_go/releases/download/0.18.3/rules_go-0.18.3.tar.gz"],
    sha256 = "86ae934bd4c43b99893fc64be9d9fc684b81461581df7ea8fc291c816f5ee8c5",
)

# 从下载的扩展里载入 go_rules_dependencies go_register_toolchains 函数
load("@io_bazel_rules_go//go:deps.bzl", "go_rules_dependencies", "go_register_toolchains")

# 注册一堆常用依赖 如github.com/google/protobuf golang.org/x/net
go_rules_dependencies()

# 下载golang工具链
go_register_toolchains()
```



一般来说都会使用gazelle工具来自动生成`BUILD`文件,而不是手写.添加以下加入`WORKSPACE`

```shell
# 下载 gazelle
http_archive(
    name = "bazel_gazelle",
    urls = ["https://github.com/bazelbuild/bazel-gazelle/releases/download/0.17.0/bazel-gazelle-0.17.0.tar.gz"],
    sha256 = "3c681998538231a2d24d0c07ed5a7658cb72bfb5fd4bf9911157c0e9ac6a2687",
)

# 载入依赖
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")
gazelle_dependencies()
```

在`BUILD`文件写入

```shell
load("@bazel_gazelle//:def.bzl", "gazelle")

# 下面这行是必要的注释 注明了你的包前缀 例如github.com/example/project
# gazelle:prefix go-common
gazelle(name = "go-common")
```



然后运行

```shell
# 如果上面修改了名字 这里也需要修改
bazel run  //:gazelle
```



成功输出如下

```
INFO: 0 processes.
INFO: Build completed successfully, 1 total action
INFO: Build completed successfully, 1 total action
```



进行编译

```
bazel build //...
```





