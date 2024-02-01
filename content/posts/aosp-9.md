---
title: 从零开始自定义安卓系统(9)  初始化脚本
date: 2024-02-02T00:31:04+08:00
lastmod: 2024-02-02T00:31:04+08:00
categories: ["Android"]
tags: ["aosp"]
description:
---
## 从零开始自定义安卓系统(9)  初始化脚本

### init.rc

`init.rc`是安卓用于定义系统启动时操作的格式, 安卓启动会尝试读取

`/{system,system_ext,vendor,odm,product}/etc/init/` 目录下的rc文件



如果要实现开机时运行一个自定义脚本, 只需要在产品的mk文件里添加

```makefile
PRODUCT_COPY_FILES +=                  $(LOCAL_PATH)/init.bluebird.rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/init.bluebird.rc
 $(LOCAL_PATH)/autostart.sh:$(TARGET_COPY_OUT_SYSTEM)/bin/autostart.sh
```

然后在`init.bluebird.rc` 里编辑即可

如果出于测试目的可以选择直接在系统内编辑,上述文件然后重启



### rc文件语法

`rc`文件语法有两块 一块是定义服务, 一块是定义触发动作.

[完整参考](https://android.googlesource.com/platform/system/core/+/master/init/README.md)



服务定义语法  

```
# 格式
# service <name> <pathname> [ <argument> ]*
#   <option>
#   <option>
# 参考
service test_bluebird /system/bin/autostart.sh
       user root 
       disabled
       oneshot
```

`disabled` 禁止自启, 直到被调用. `oneshot` 使得服务不会被重复调用和重启, 用于只执行一次的脚本



然后定义什么时候触发

```
# on <trigger> [&& <trigger>]*
#   <command>
#   <command>
#   <command>
```

 `aosp`定义的阶段触发器有 

1. `early-fs` - Start vold.
2. `fs` - Vold is up. Mount partitions not marked as first-stage or latemounted.
3. `post-fs` - Configure anything dependent on early mounts.
4. `late-fs` - Mount partitions marked as latemounted.
5. `post-fs-data` - Mount and configure `/data`; set up encryption. `/metadata` is reformatted here if it couldn't mount in first-stage init.
6. `zygote-start` - Start the zygote.
7. `early-boot` - After zygote has started.
8. `boot` - After `early-boot` actions have completed.



也可以通过`prop`属性方式控制触发, 例如 `dev.bootcomplete`

```
on property:dev.bootcomplete=1
   start test_bluebird
   # 也可以 exec -- /system/bin/test.sh
   # setprop ro.test 1 
```





> 如果使用服务形式,可能需要修改`selinux` 策略, 以增加服务权限以进行操作 如文件操作  参考 https://source.android.com/docs/security/features/selinux?hl=zh-cn 


