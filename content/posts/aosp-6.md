---
title: 从零开始自定义安卓系统(6)  自定义模块 添加系统app
date: 2024-01-30T18:26:01+08:00
lastmod: 2024-01-30T18:26:01+08:00
categories: ["android"]
tags: ["aosp"]
description:
---
## 从零开始自定义安卓系统(6)  自定义模块 添加系统app 

### 前言

在之前就知道可以通过`PRODUCT_PACKAGES` 添加自己的包, 具体操作是 新建一个模块,然后

```makefile
PRODUCT_PACKAGES += 模块名
```



要定义一个模块 可以使用`soong` 或者`makefile`,  在新版本系统推荐使用`soong`



`soong` 使用 `.bp`文件定义模块 [参考](https://source.android.com/docs/setup/build?hl=zh-cn) 

从官方例子就可以看到其实格式和json一样简单, 

```
cc_binary {
    name: "gzip",
    srcs: ["src/test/minigzip.c"],
    shared_libs: ["libz"],
    stl: "none",
}
```



主要问题是找到需要的模块和属性 这个时候就需要[格式参考](https://ci.android.com/builds/submitted/11337780/linux/latest/view/soong_build.html)

里面主要根据语言分类, 想做一件事的时候查询对应类型,和对应模块名的格式参考



### 添加预下载apk

查看`android/soong/java`类型  看名字就知道 前面两个就是.

`android_app`和`android_app_import`

查阅文档克制区别在于 前者是需要编译的模块 后者是直接导入apk文件

[文档](https://ci.android.com/builds/submitted/11337780/linux/latest/raw/java.html#android_app_import)提供了一个例子

```
android_app_import {
	    name: "example_import",
	    apk: "prebuilts/example.apk",
	    dpi_variants: {
	        mdpi: {
	            apk: "prebuilts/example_mdpi.apk",
	        },
	        xhdpi: {
	            apk: "prebuilts/example_xhdpi.apk",
	        },
	    },
	    presigned: true,
}
```

文档里还提供5行的其他选项, 不过基本都是通用选项, 与`app`有关的不多 , 看旁边注释就能了解作用



如果要导入一个叫`vnc.apk`的文件

新建一个目录,放入apk文件,和`Android.bp`

```
# Android.bp
android_app_import {
    name: "vnc_import",
    apk: "./vnc.apk",
    # 解决可能的签名问题
    dex_preopt: {
       enabled: false,
    },
    presigned: true,
}
```

然后在`bluebird.mk` (产品mk文件)添加

```
PRODUCT_PACKAGES += vnc_import
```

然后编译即可在 `system/app` 看到 `vnc_import`目录



如果遇到` mismatch in the <uses-library> tags between the build system and the manifest:`

则按提示在 产品文件添加

```makefile
PRODUCT_BROKEN_VERIFY_USES_LIBRARIES := true
```

或在bp文件添加提示中对应的lib

```
android_app_import {
	# 省略
	uses_libs:["lib"],
	optional_uses_libs:["可选lib"]
}
```


