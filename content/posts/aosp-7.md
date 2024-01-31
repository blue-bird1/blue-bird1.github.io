---
title: 从零开始自定义安卓系统(7)  预装二进制文件和删除预装模块
date: 2024-01-31T19:07:48+08:00
lastmod: 2024-01-31T19:07:48+08:00
categories: ["android"]
tags: ["aosp"]
description:
---
## 从零开始自定义安卓系统(7)  预装二进制文件和删除预装模块

### 预编译二进制文件

操作上类似于预装系统`app` 不过变成了使用`cc_prebuilt_binary`

推荐在`external/` 目录新建一个文件夹,然后添加一个`Android.bp`

```json
cc_prebuilt_binary {
   name:"ecapture",
   srcs:["./ecapture"]
   strip:{
     none:true
   }
}
```



比起系统`app`, 二进制文件的一大问题是 预编译的架构不一样是无法运行的, 这个时候可以使用`arch`参数

```
cc_prebuilt_binary {
   name:"ecapture",
   arch:{
     x86_64: {
       srcs:["./ecapture-v0.7.3-linux-x86_64/ecapture"]
     },
     # 有四种架构 arm, x86,arm64, x86_64
   },
   strip:{none:true}
}
```

为每个不同的架构指定不同的二进制文件.

另一个问题就是未必想要放到`/system/bin`, 在`soong`大部分模块都具有以下两个属性

`vendor` 和`product_specific`  , 设置为true后 会放到`/vendor` 和 `/product ` 分区, 如果没有将会放入`/system/vendor` 和 `/system/product`



### 删除模块

#### 如何查看目标文件的对应代码路径

很多情况下都是要删除一个系统`apk`,但是要删除一个文件,首先要知道他对应的源码路径.  在`out/target/product/<产品名>`  路径下有个`module-info.json`

里面包含了模块与其对应的路径, 直接搜索对应的文件名, 会看到类似这样的`json`格式

```json
 "ActivityManagerPerfTestsUtils": { "class": ["JAVA_LIBRARIES"],  "path": ["frameworks/base/tests/ActivityManagerPerfTests/utils"],  "tags": ["tests"],  "installed": ["out/target/product/test/testcases/ActivityManagerPerfTestsUtils/x86_64/ActivityManagerPerfTestsUtils.jar"],  "compatibility_suites": ["null-suite"],   "module_name": "ActivityManagerPerfTestsUtils",  "supported_variants": ["DEVICE"] },
```

`installled` 对应的就是路径, 而`module_name` 则是对应的名字, `path`对应源码路径. 之后再根据模块名

去删除.



#### 删除和替换模块

如果要删除的话,需要修改对应产品的`Android.mk` 或者 系统app定义在`/build/make/core.mk` .移除对应的`PRODUCT_PACKAGES` 属性, 就是添加模块的反向操作.



另一种更可维护的方法就是使用替换.  `aosp` 实际上没有编译时删除模块的概念, 实际上使用的是`替换` 

因为版本问题, 以前博客都是使用`.mk`的形式, 例如 [秋少的博客](https://qiushao.net/2019/12/12/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/6-%E5%88%A0%E9%99%A4%E5%8E%9F%E7%94%9F%E5%86%85%E7%BD%AEAPK/). 

不过`soong`实际上也有类似的功能, 实际上也是填写了`LOCAL_OVERRIDES_PACKAGES`这个值



在部分模块下, 具有`overrides`属性, 可以通过这个属性后覆盖替换掉 其他模块.

例子

```
android_app_import {
    # 省略
    overrides: ["Email","Music"]
}
```

 

拥有这个属性的主要集中在`android` 模块, 在源码中主要用于替换`apk`


