---
title: 漏洞挖掘技巧
date: 2019-12-08T12:01:04+08:00
lastmod: 2019-12-08T12:01:04+08:00
categories: ["网络安全"]
description:
---
仅作为个人的漏洞类型和技巧记录 于阅读漏洞报告时记录



## 类型

不需要特殊技巧 简单就可以确认的类型

### GraphQL查询漏洞

Graphql作为一种前端查询语言 如未对查询进行限制 可以构造恶意查询 恶意消耗服务器资源.同时GraphQl的权限限制也是一大漏洞点

#### 参考

https://graphql.org/learn/

<https://blog.apollographql.com/securing-your-graphql-api-from-malicious-queries-16130a324a6b>

### jsonp响应头问题

jsonp响应头是`text/html` 可直接当作反射xss



### 重定向后执行

在对客户端重定向后 未终止程序 导致后面代码未预料的执行

```php
// 例子 
header("Location:page1.php");
 eval($_GET["a"])
```



### web缓存欺骗攻击

网站在访问不存在文件时会返回404页面, 而假如网站对.js/.css 文件会进行缓存 且未对404情况处理. 404页面内保存有客户敏感信息(如csrf cookies)的话,使他访问一个不存在的.js文件.将会把他的敏感信息保存下来让攻击者查看

快速确认方法, 确认404页面构造和是否有缓存



### XXE

解析XML文件的实体将导致执行任意命令  快速确认:任何使用xml作为输入的api都是值得尝试的



### 正则dos

一些错误的正则表达式将导致一个指数级的复杂度 输入一个特殊的匹配字符串将导致dos.

使用<http://regex101.com/>可以查看正则表达式匹配时实际使用的步数和时间. 

常见模式`\d{1:>20}` `(\d*)+`

这个漏洞非常有趣 



### CORS配置问题

网站错误的配置将导致恶意网站可以跨域访问用户在此网站的信息



### 备份文件可猜测

网站生成备份文件名可猜测并且未防止访问的话 可以访问所有数据



### 平行越权

访问其他用户的数据或代表其他用户操作 常见于用`userid`参数的api 替换id就可以用对应id权限



### 验证码dos

验证码接受了长宽参数. 通过输入一个足够大的数字将消耗大量服务器资源. 快速确认 对验证码接口输入`width`和`height`  和修改



### 用户名枚举

对于不存在用户名和存在的情况存在两种响应 并不存在验证手段 攻击者可以枚举已存在的用户名



### http头攻击

输入参数控制了响应头的一部分 并且可以插入`\r\n` 在http头输入任意内容.

可能性快速确认:输入参数存在于返回响应头 



### 模板注入

控制模板文件内容 导致在解析时执行未预料的指令. 

快速确认:输入模板常用的分割符和简单计算如 `{{ 1+1 }}` `[[ 1+1 ]]`



### CSS注入

css也可以执行js代码



### onMessage

js中的`onMessage`时间如果不进行限制 默认将会接收所有网站发出的`postMessage ` . 现在开发一般对`message`信任  如果代码直接使用这部分数据作为html 将导致xss. 



## 技巧

### IE/EDGE浏览器未编码window.location.href

直接使用window.location.href作为html时 IE/edge浏览器未对这参数进行url编码



### svg XSS

svg文件可以导致xss, 如果上传图片未限制.svg的话将存在漏洞



### pdf xss

pdf可以加入js代码 将会导致xss



### %00终止bash执行

bash执行时 如果命令中引号内有%00 将直接抛出错误



### bypass 域名验证 

如果是以正则 `^www.test.com`作为验证  通过`www.test.com@evil.com`绕过

如果是通过解析url再通过验证域名 可以通过浏览器对错误url的解析来绕过 例如

`evil.com\\@test.com` 实际上是`evil.com` 而解析url时域名解析成`test.com` 了

同样的`/\google.com` 将前往google.com



### sentry SSRF

sentry是一个错误报告服务 但是配置不当 可能通过它的api来`SSRF`



### GET CSRF

常见框架只对于`POST` 请求验证csrf 如果api允许get方法 将直接csrf



### 上传符号链接文件

访问符号链接文件时将访问到对应的真实文件 导致文件读取

### `<script>`块内xss
`'`和`"`和`<>`任意一个未转义都可能导致xss
`<script>`块内不需要考虑逃脱`'`和`"`
例如
```
<script>
var a ="</script><img>"
```

### 跳转xss
常见跳转页都是JavaScript操作的,如果将跳转地址改为`JavaScript:xxx`将把重定向漏洞升级为xss

### jsonp xss
jsonp的`content-type`设置为`text/html` 未过滤`callback`的话就等于反射xss
