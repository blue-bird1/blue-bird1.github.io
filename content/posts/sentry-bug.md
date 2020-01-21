---
title: Sentry 过时依赖导致的安全问题
date: 2020-01-22T02:29:26+08:00
lastmod: 2020-01-22T02:29:26+08:00
categories: ["网络安全"]
description:
---

### Sentry因过时依赖导致的安全问题

### 前言

这次挖的其实没啥技术含量 但是从依赖挖掘到漏洞还是比较少见的. 



### 挖掘过程

下载源码后,发现src目录有`social_auth`目录,并查看`views.py`(django的主要业务逻辑文件).

并注意到

```python
   # Save any defined next value into session
    if REDIRECT_FIELD_NAME in data:
        # Check and sanitize a user-defined GET/POST next field value
        redirect = data[REDIRECT_FIELD_NAME]
        # NOTE: django-sudo's `is_safe_url` is much better at catching bad
        # redirections to different domains than social_auth's
        # `sanitize_redirect` call.
        if not is_safe_url(redirect, host=request.get_host()):
            redirect = DEFAULT_REDIRECT
        request.session[REDIRECT_FIELD_NAME] = redirect or DEFAULT_REDIRECT
```

sentry使用了`django-sudo`来做url验证,搜索这个库.发现这个库的实际新代码提交是`2016`年的.

这意味着如果在近些年出现过bypass,这个库并未进行修复. 快速的进行了谷歌搜索.很快就发现`CVE-2017-7233` django的is_safe_url的绕过. 

参考文章 `https://paper.seebug.org/274/#cve-2017-7233-django-is95safe95url-urlbypass`

通过对调用的分析  选择了logout作为简单的poc`http://127.0.0.1:9000/auth/logout/?next=https:1029415385`.`  点击`sign out`将跳转到一个谷歌的ip



### 漏洞时间线

`2019-12-24`: 漏洞提交

`2019-1-14`: 问题已修复



