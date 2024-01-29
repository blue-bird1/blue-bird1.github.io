---
title: 从零开始自定义安卓系统(5)  修改prop属性
date: 2024-01-30T00:36:53+08:00
lastmod: 2024-01-30T00:36:53+08:00
categories: ["android"]
tags: ["aosp"]
description:
---
## 从零开始自定义安卓系统(5)  修改prop属性

### 从product.mk修改

在上一节已经知道

```
PRODUCT_SYSTEM_PROPERTIES
PRODUCT_SYSTEM_EXT_PROPERTIES
PRODUCT_VENDOR_PROPERTIES
PRODUCT_ODM_PROPERTIES
PRODUCT_PRODUCT_PROPERTIES
```

这些可以修改prop属性

只需要加一行 

```
PRODUCT_SYSTEM_PROPERTIES += ro.bluebird = 0
```

就可以在输出目录里的 `out/target/product/bluebird_x86_64_only/system/build.prop ` 找到新属性.



其他属性对应的是其他文件, 整体对应关系是

```
PRODUCT_SYSTEM_PROPERTIES: system/build.prop
PRODUCT_SYSTEM_EXT_PROPERTIES: system_ext/etc/build.prop
PRODUCT_VENDOR_PROPERTIES: vendor/build.prop
PRODUCT_PRODUCT_PROPERTIES: product/etc/build.prop
PRODUCT_ODM_PROPERTIES: odm/etc/build.prop
```

对应的文件级设置为

```
TARGET_SYSTEM_PROP: system/build.prop
TARGET_SYSTEM_EXT_PROP: system_ext/etc/build.prop
TARGET_VENDOR_PROP 默认值(/vendor.prop):  vendor/build.prop
TARGET_PRODUCT_PROP 默认(/product.prop): product/etc/build.prop
TARGET_ODM_PROP 默认(/odm.prop): odm/etc/build.prop
```

要删除就需要使用

```
PRODUCT_SYSTEM_PROPERTY_BLACKLIST : system
PRODUCT_VENDOR_PROPERTY_BLACKLIST : vendor
```



具体实现在`build/make/core/sysprop.mk` 里面

拿一段举例就是

```makefile
# ----------------------------------------------------------------
# odm/etc/build.prop
#

# _prop_files_ 决定了文件级参数, 取TARGET_ODM_PROP or /odm.prop
_prop_files_ := $(if $(TARGET_ODM_PROP),\
    $(TARGET_ODM_PROP),\
    $(wildcard $(TARGET_DEVICE_DIR)/odm.prop))
    
# 参数级参数  PRODUCT_ODM_PROPERTIES
_prop_vars_ := \
    ADDITIONAL_ODM_PROPERTIES \
    PRODUCT_ODM_PROPERTIES
    
# 输出路径
INSTALLED_ODM_BUILD_PROP_TARGET := $(TARGET_OUT_ODM)/etc/build.prop

# 调用build prop文件的函数 参数位置与上述一直
$(eval $(call build-properties,\
    odm,\
    $(INSTALLED_ODM_BUILD_PROP_TARGET),\
    $(_prop_files_),\
    $(_prop_vars_),\
    # 从上面删除的参数
    $(empty),\
    $(empty),\
    $(empty)))
```



实际应用中就是 添加一个 product.prop 

```
├── AndroidProducts.mk
├── README.md
├── bluebird.mk
├── bluebird_x86_64_only
│   └── BoardConfig.mk
└── product.prop
```

并在bluebird.mk中添加

```
PRODUCT_SYSTEM_PROPERTIES += ro.bluebird=1⏎ 
```

编译运行后就能看到 自己新添加的prop属性了


