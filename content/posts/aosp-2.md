---
title: 从零开始自定义安卓系统(2) 编译redroid 
date: 2024-01-23T18:11:54+08:00
lastmod: 2024-01-23T18:11:54+08:00
categories: ["Android"]
tags: ["aosp"]
description:
---
## 从零开始自定义安卓系统(2) 编译redroid

### 添加redroid补丁

`redroid`添加了自己的补丁

```bash
# apply redroid patches
git clone https://github.com/remote-android/redroid-patches.git ~/redroid-patches
~/redroid-patches/apply-patch.sh . 
```

具体补丁内容可以查看 [仓库](https://github.com/remote-android/redroid-patches) 对应文件夹



### 准备编译

```bash
 source build/envsetup.sh
 lunch redroid_x86_64_only-userdebug
```

这个`envsetup.sh`文件里添加了很多辅助函数 可以使用`hmm`查看. 最常用的几个

- `m`  编译整个项目
- `mm` 编译当前目录项目
- `croot` 回到项目根目录
- `lunch` 选择编译目标

大部分只是简单的命令套壳无需关心, 剩下的也只是调用 `build/soong/soong_ui.bash` 这个脚本



这个脚本有 `"--make-mode" "--dumpvar-mode" "--dumpvars-mode" "--build-mode"   `  四个选项

`--dumpvar-mode `用于从整个项目里获取编译变量设置,   `get_build_var` 就简单封装了这个功能



`lunch` 获取可用编译目标 就是调用 `get_build_var COMMON_LUNCH_CHOICES` (这个可以在命令行里直接执行)

可以看到输出里面出现了`redroid`开头的四个选项,

```
redroid_arm64-userdebug 
redroid_arm64_only-userdebug 
redroid_x86_64-userdebug 
redroid_x86_64_only-userdebug
```

这是`redroid`添加的 位于`device/redroid/AndroidProducts.mk`, 



然后执行

```
repo forall -c 'git lfs pull'
```

同步`git lfs`文件,  原版是不需要的.  但是`redroid` 在`external/chromium-webview/prebuilt/` 添加了`lfs`项目, 如果速度很慢,可以考虑直接下载而不是走`lfs



接着执行

`m`   # -j x

等待很长一段时间后,就能在 `out/target/ `看到产物



编译完成打包成`docker`

```bash
cd out/target/product/redroid_x86_64_only/
sudo mount system.img system -o ro
sudo mount vendor.img vendor -o ro
sudo tar --xattrs -c vendor -C system --exclude="./vendor" . | docker import -c 'ENTRYPOINT ["/init", "androidboot.hardware=redroid"]' - redroid
sudo umount system vendor
```



配置环境 参考[文档](https://github.com/remote-android/redroid-doc/blob/master/deploy/ubuntu.md)

```bash
docker run -itd  --privileged -v ~/data11:/data --name redroid --rm redroid
```



出现闪退错误使用命令排查

`sudo dmesg  -T`
