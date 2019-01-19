---
title: think5审计与调试技巧1
date: 2019-01-17T23:19:42+08:00
lastmod: 2019-01-17T23:19:42+08:00
author: bluebird
categories: ["网络安全"]
tags: ["审计"]
---

### think5审计与调试技巧1

<!--more-->
think5是一个非常流行的框架, 现在的cms很多都采用了think5作为开发框架.这就带来一个问题, 没用过的安全人员审计的时候就非常懵逼了. 

例如 程序入口在哪?  orm操作都是这种函数  `Db::name($modeln['tablename'])->where('id',$id)->setInc('click');`  怎么操作才会出现sql注入?

这就需要框架知识了,但是学习整个框架又太多, 不学又不知道怎么审计.所以这个系列旨在带来足以审计的think5框架知识,而不太复杂



#### 整体目录结构

think5 主要需要关注的目录如下

```
├── application 应用目录（可设置）
├── config 配置目录 
├── extend 扩展类库目录（可定义）
├── public 网站目录
├── route   路由
├─- runtime  应用的运行时目录（可写，可设置）
├── thinkphp 框架目录
└── vendor 第三方库
```



我们审计主要看`application` 目录 



5.0 官方给的目录参考是

```
├─application           应用目录（可设置）
│  ├─common             公共模块目录（可更改）
│  ├─index              模块目录(可更改)
│  │  ├─config.php      模块配置文件
│  │  ├─common.php      模块函数文件
│  │  ├─controller      控制器目录
│  │  ├─model           模型目录
│  │  ├─view            视图目录
│  │  └─ ...            更多类库目录
│  ├─command.php        命令行工具配置文件
│  ├─common.php         应用公共（函数）文件
│  ├─config.php         应用（公共）配置文件
│  ├─database.php       数据库配置文件
│  ├─tags.php           应用行为扩展定义文件
│  └─route.php          路由配置文件
```

但是事实可能缺失很多部分 例如`nonecms` 的目录是

```

├── admin
│   ├── behavior
│   ├── config
│   ├── controller
│   ├── rbac.php
│   ├── tags.php
│   ├── validate
│   └── view
├── command.php
├── common
│   ├── lib
│   ├── model
│   └── taglib
├── common.php
├── index
│   ├── config
│   └── controller
├── mobile
│   ├── config
│   └── controller
└── push
    ├── controller
    └── service
```



#### think5 url

最常见的think5 url是

`http://serverName/index.php（或者其它应用入口文件）/模块/控制器/操作/[参数名/参数值...]`

`http://serverName/index.php（或者其它应用入口文件）?s=/模块/控制器/操作/[参数名/参数值...]`

像

`index.php/index/blog/read` `index.php?s=/index/blog/read`

其他方式也有 但是基本大同小异 例如`index/listing/index/cid/47.html`



#### 配置文件

think5.1 配置文件在config目录  5.0在`application/config.php` 

常见的配置文件

`app.php  cache.php  cookie.php  database.php  log.php  session.php  template.php  trace.php`

最重要的配置文件是`app.php`

主要需要关注的配置如下

```
// 应用调试模式
'app_debug'              => true,
// 应用Trace
'app_trace'              => true,
// 默认全局过滤方法 用逗号分隔多个
'default_filter'         => '',
// 禁止访问模块
'deny_module_list'       => ['common'],
```

`app_debug` 和 `app_trace` 建议设置成true. 

调试模式下异常会显示详细信息,而不是通用报错界面

`app_trace` 则会在右下角显示一个按钮,根据设置可以显示执行路径,执行sql等等

如果没有显示,需要添加

```
// Trace信息
'trace'     =>  [
    //支持Html,Console
    'type'  =>  'html',
] 
```



如果没有看到这个设置  可能在`trace.php`中设置

```
<?php
return [
    // 内置Html Console 支持扩展
    'type' => 'Html',
];
```

`xxx.php` 对应的是`app.php`里的`xxx`设置

`default_filter` 可能的值是函数, 例如 `strip_tags` 等于对所有用户传入的参数执行过滤.

`deny_module_list` 则是禁止访问的模块



#### 日志

`config/log.php`

```
return [
    // 日志记录方式，内置 file socket 支持扩展
    'type'  => 'File',
    // 日志保存目录
    'path'  => '',
    // 日志记录级别
    'level' => [],
];
```

默认路径是在`runtime/log`



#### 数据库trace

`app_trace`设置后会发现并没有sql记录,这个需要在database.php添加

```
'debug'       => true,
```

不过就算你看到你的sql注入进入了显示的语句, 但是由于thinkphp5的参数绑定, 很可能并没有生效.



#### 路由

路由对审计影响其实不大, 毕竟url怎么改, 真正的执行代码也不会变.建议扫描性的看一下,是否有开发不小心把调试用的路由留在上面了..

