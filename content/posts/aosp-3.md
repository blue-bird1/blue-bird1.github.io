---
title: 从零开始自定义安卓系统(3) 配置开发环境与新建product
date: 2024-01-28T02:26:53+08:00
lastmod: 2024-01-28T02:26:53+08:00
categories: ["android"]
tags: ["aosp"]
description:
---

## 从零开始自定义安卓系统(3) 配置开发环境与新建product

### aidegen

`aidegen`是一个自动生成项目配置文件的工具 在运行完`lunch`后 会自动配置这个工具路径

主要参数有    [完整文档参考](https://android.googlesource.com/platform/tools/asuite/+/refs/heads/main/aidegen/README.md)

- `-s`  跳过构建

- `-n ` 不自动运行`IDE` 
- `-i`   选择`IDE`类型  

- `-p` 指定`IDE`路径

```bash
# 用vscode编辑整个项目 v是vscdoe c=clion a=android stuido i=idea
aidegen -i v -s .
# 编辑单个项目  检测后是打开android studio
aidegen packages/apps/Settings
```



### 目录结构

AOSP目录结构非常复杂 由非常多项目构成,但是大部分并不需要修改 

```
- art 
- Dalvik
app运行时, 执行dex文件, 系统修改通常可以无视, 客户端岗位八股文经典题目


- bootable
硬件boot层 如fastboot 如果不做硬件开发可以直接无视
- bionic 
实现libc这种底层库, 通常不需要改动

- cts
- platform_testing
兼容性和平台测试相关 自己玩玩可以无视

- developers 
提供一些例子代码
- development 
一些开发与例子代码


- external  
- prebuilt 
都是存放的外部代码 开发不需要看

-libcore
-libnativehelper
只需要用

- build
- toolchain
- tools
编译用

```

需要主要关注的

```
# rom修改
/packages  自带的应用包
/framework 安卓系统框架

#  系统定制
/vendor 厂商定制
/device 厂商设备
/hardware  硬件层
/kernel 系统内核

# 如果对如何在AOSP 实现某个功能有疑问,如设置什么选项才能赋予root权限, 可能需要查阅代码编译流程
/build
```

### 新建device和product

在编译时已经知道 编译时需要选择编译目标.而编译目标在`AndroidProducts.mk` 这种文件下设置



参考`redroid` 

```
PRODUCT_MAKEFILES := \
    $(LOCAL_DIR)/redroid_x86_64.mk \
    $(LOCAL_DIR)/redroid_x86_64_only.mk \
    $(LOCAL_DIR)/redroid_arm64.mk \
    $(LOCAL_DIR)/redroid_arm64_only.mk \

COMMON_LUNCH_CHOICES := \
    redroid_x86_64-userdebug \
    redroid_x86_64_only-userdebug \
    redroid_arm64-userdebug \
    redroid_arm64_only-userdebug \
```



可知要新建一个`device` 首先在device目录下创建一个子目录 例如 `device/bluebird`

然后创建一个``AndroidProducts.mk` ` 

```
# 这两个是一一对应关系,如果文件同名  `bluebird_redroid_x86_64.mk` 可以省略前面
bluebird_redroid_x86_64:
PRODUCT_MAKEFILES := \
    bluebird_redroid_x86_64:$(LOCAL_DIR)/bluebird.mk 

COMMON_LUNCH_CHOICES := \
    bluebird_redroid_x86_64-userdebug 
    
```





然后新建一个`bluebird.mk `

```
# 直接复制redroid的配置
$(call inherit-product, $(LOCAL_PATH)/../redroid/redroid_x86_64_only.mk)

# 定义自己产品的名字
PRODUCT_NAME := bluebird_x86_64_only
PRODUCT_DEVICE := bluebird_x86_64_only
PRODUCT_BRAND := blulebird
PRODUCT_MODEL := bluebird_x86_64_only
```

再新建一个 `bluebird_x86_64_only/BoardConfig.mk`

```
# 直接导入redroid的对应配置
include device/redroid/redroid_x86_64_only/BoardConfig.mk
```

最后结果

```
○ → tree device/bluebird/
device/bluebird/
|-- AndroidProducts.mk
|-- README.md
|-- bluebird.mk
`-- bluebird_x86_64_only
    `-- BoardConfig.mk
```





然后就能进行 `lunch bluebird_x86_64_only-userdebug`
