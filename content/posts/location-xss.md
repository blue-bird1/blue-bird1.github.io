---
title: XSS实战 跳转XSS
date: 2019-04-20T12:43:40+08:00
lastmod: 2019-04-20T12:43:40+08:00
categories: ["网络安全"]
description:
---

## XSS实战: 跳转XSS

### 前言

跳转XSS实际上并不是一种新类型的攻击方式,

主要形成原因是 以前网站外部跳转时是直接接受参数然后跳转,导致了URL重定向漏洞.而现在网站喜欢加个跳转页,不会直接跳转,而是接受参数然后用js跳转. 这就有一个问题 如果未验证参数, js跳转时是可以接受JavaScript伪协议执行js代码的.



### 漏洞代码示例

```php+HTML
<?php
	echo "<script>window.location.href = $_GET['url']</script>";
?>
```



### 实战例子

以拉勾网为例,作者打开页面都会先看看js里有什么信息.很快发现js里有这段代码

 ![Screenshot_15.png](https://i.loli.net/2019/04/20/5cbb2a45d44a9.png)

显然只有有参数的才能引起兴趣.作者快速尝试了`https://sec.lagou.com/verify.html?e=test1&f=test2`

发现url参数直接进入了页面,当然与本次主题有关的是参数`f`.虽然另一个更直接. 它在页面的位置是

```
    function submit1(data){
        var host = "test2";
        $.ajax({
            url: 'parseSession',
            type: "post",
            dataType: "json",
            data: {
                challenge: data.challenge,
                errcode:1
            },
            success: function (result) {
                window.location.href = host;
            }
        })

        //$.getJSON('/user/sendCode.json', {t: new Date().getTime(), type: $('#contactSelect').val()}, function (data)
    }
```



尝试引入双引号.被转义了.所以这个参数就没问题了么?并不是.下面的代码使用了这个参数进行跳转.

可以使用`https://sec.lagou.com/verify.html?e=1&f=JavaScript:alert(1)`弹框.



### 最后

首先拉勾网是没有src的,别想了. 另外现在这个问题似乎已经修复了?访问 https://sec.lagou.com/parseSession会302..

e参数没修.e出现的代码为

```javascript
    $('#submit').click(function() {
        var code = document.getElementById('code').value;
        if (!code) {
            showError('请填写验证码');
            return;
        }
        else{
            console.log("111111")
            console.log(test1)
            $.ajax({
                url: 'checkcode',
                type: "post",
                dataType: "json",
                data: {
                    t: new Date().getTime(),
                    code: code,
                    errcode :test1;
                },
                success:function (data) {
                    console.log(data)
                    if (data.state == 1) {
                        goToUrl(getUrlParam("f"))
                    } else {
                        showError("验证码错误，请重新输入！")
                    }
                }
            })
        }
    });

```

双引号和单引号均被转义,有兴趣的读者可以作为对自己的挑战.还有过滤的waf









 
