---
title: 从零开始自定义安卓系统(8)  用复制文件与叠加层自定义系统
date: 2024-01-31T22:48:00+08:00
lastmod: 2024-01-31T22:48:00+08:00
categories: ["android"]
tags: ["aosp"]
description:
---
## 从零开始自定义安卓系统(8)  用复制文件与叠加层自定义系统

### 复制文件

产品级拥有一个选项可以直接复制文件到系统

```makefile
PRODUCT_COPY_FILES += test.sh:$(TARGET_COPY_OUT_VENDOR)/bin/test.sh \
```

格式上很简单, 一个`:`分割的对应关系. 比较特殊的是使用`$(TARGET_COPY_OUT_VENDOR)`来获取指定目录.



这样有利于后续维护,这些常用的全局变量位于`build/core/envsetup.sh`

里面定义了全局变量,  如

```makefile
TARGET_COPY_OUT_SYSTEM := system
TARGET_COPY_OUT_SYSTEM_DLKM := system_dlkm
TARGET_COPY_OUT_SYSTEM_OTHER := system_other
TARGET_COPY_OUT_DATA := data
TARGET_COPY_OUT_ASAN := $(TARGET_COPY_OUT_DATA)/asan
TARGET_COPY_OUT_OEM := oem
TARGET_COPY_OUT_ROOT := root
TARGET_COPY_OUT_RECOVERY := recovery

# 节选 全列表在此文件中搜索 `TARGET_OUT`即可
```

如果要复制到` /system/xxx` 可以使用`$(TARGET_COPY_OUT_SYSTEM)/xxx`

一般用于放一些配置文件 例如`init.rc`



### 叠加层

叠加层也叫`overlay`层, 比起直接复制文件的区别在于. 复制文件是面向最终输出, 会复制到输出目录,  支持任意文件. 

 叠加层是面向编译系统, 可以实现在编译时替换资源文件. 但是也只能修改资源文件.

但是可以实现修改已有代码的配置,而无需修改安卓源代码.



要使用叠加层只需要

```
PRODUCT_PACKAGE_OVERLAYS += ${LOCAL_PATH}/overlay 
```



然后根据要修改的文件路径新建对应的目录

例如要修改`frameworks/base/core/res/res/values/config.xml`

```bash
mkdir -p overlay/frameworks/base/core/res/res/values
touch overlay/frameworks/base/core/res/res/values/config.xml
```

 然后按照`xml`格式 写上需要修改的内容

```xml
<resouce>
     <bool name="config_disableLockscreenByDefault">false</bool>
</resouce>
```

只会影响所写的值,不影响其他默认值.



要使用叠加层并不容易, 首先要知道到底有什么配置项,和在哪个路径.  而在了解这些之前几乎没什么用,  需要先阅读源码中的`framework/base/core/res/`里的`xml`文件中的注释.



### 动态叠加层

也称为`运行时资源覆盖`(RRO),  其实就是从编译时覆盖变成了系统运行时覆盖,为了加入系统, 实际上安装形式是`apk` 

要想使用`RRO`,首先需要新建一个模块目录,然后添加`Android.bp`

```
runtime_resource_overlay {
    name: "ExampleOverlay",
    sdk_version: "current",
}
```



然后添加`app`必要的`AndroidManifest.xml`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.overlay">
    <application android:hasCode="false" />
    <overlay android:targetPackage="com.android.contacts"
                   android:targetName="OverlayableResources"
             	   android:resourcesMap="@xml/overlays"
             />
</manifest>
```



`targetPackage` 是想要覆盖的apk名, 这里是`aosp`自带的联系人应用

`@xml/overlays`是覆盖资源文件名 实际路径是`res/xml/overlays.xml`

实际内容像这样

```xml
<?xml version="1.0" encoding="utf-8"?>
<overlay xmlns:android="http://schemas.android.com/apk/res/android" >
     <!-- Overlays string/config1 and string/config2 with the same resource. -->
    <item target="string/config1" value="@string/overlay1" />
    <item target="string/config2" value="@string/overlay1" />

    <!-- Overlays string/config3 with the string "yes". -->
    <item target="string/config3" value="@android:string/yes" />

    <!-- Overlays string/config4 with the string "Hardcoded string". -->
    <item target="string/config4" value="Hardcoded string" />

    <!-- Overlays integer/config5 with the integer "42". -->
    <item target="integer/config5" value="42" />
</overlay>
```

value的格式与`apk` 定义资源格式没有区别, target是需要替换的资源id, 需要查看源码或反编译目标app读取资源文件.

安卓自带的联系人资源文件位于`packages/apps/Contacts/res/` 然后默认的英语在`values`, 如果要修改字符串在`strings.xml`

如果想要修改开屏显示的`Add account , 搜索这个文件可以看到是`

```xml
<string name="contacts_unavailable_add_account">Add account</string>
```

 然后在覆盖层文件添加一行

````xml
 <item target="string/contacts_unavailable_add_account" value="Add account by code" />
````



添加到产品

`PRODUCT_PACKAGES += rro`

然后执行`mm`编译 后会出现在`system/product/overlay`



### 动态覆盖层管理

更新完系统后可以用`adb shell cmd overlay` 查看动态叠加层状态

` cmd overlay list`查看当前的`rro`  apk

可以看到这种输出

```
android
[ ] com.android.internal.display.cutout.emulation.corner
[ ] com.android.internal.display.cutout.emulation.double
[ ] com.android.internal.systemui.navbar.gestural_wide_back
[ ] com.android.internal.display.cutout.emulation.hole
[ ] com.android.internal.display.cutout.emulation.tall
[ ] com.android.internal.systemui.navbar.threebutton
[ ] com.android.internal.systemui.navbar.gestural_extra_wide_back
[ ] com.android.theme.font.notoserifsource
[ ] com.android.internal.display.cutout.emulation.waterfall
[ ] com.android.internal.systemui.navbar.gestural
[ ] com.android.internal.systemui.navbar.gestural_narrow_back
[x] com.android.systemui:neutral
[x] com.android.systemui:accent

com.android.contacts
[ ] com.example.overlay
```



包出现在列表但是没有启用, 需要使用 `cmd overlay enable com.example.overlay  ` 启用.

执行后可以看到
![contacts](/images/image_2024-01-31_22-42-48.png)
