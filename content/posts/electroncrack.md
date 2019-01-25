---
title: Electron软件简单破解
date: 2019-01-24T23:14:25+08:00
lastmod: 2019-01-24T23:14:25+08:00
categories: ["网络安全"]
tags: ["软件破解"]
description:
---

### electron 软件破解与修改入门



electron是一款流行的桌面软件框架, 可以用js来写桌面软件, 快速开发.为了提高开发效率,不少公司比如白帽汇直接采用了这种技术编写客户端,而不是传统的c++,c#.. 以下均采用白帽汇的fofa客户端作为例子讲解,目的是让fofa客户端的扫描功能无需验证. 这个功能命令是直接调用cli的, 并不需要网络验证.



#### 解包

electron的代码在`resources`目录,根据打包方式的不同, 可能看到`app`目录或者`app.asar`. 目录就不需要解包了. 

`asar`并不是加密格式, 只是压缩格式. 可以使用`asar`工具直接解包.

下载方式

`npm install -g asar`

解包命令

`asar extract app.asar  <目录名>`

例如`asar extract app.asar  fofa`



#### 代码目录

代码目录是什么样子全看开发者. 通常都会存在`main.js`和`node_modules`目录.`main.js`是启动文件

```
»»»» ls                                                                       
com  css  data  fonts  images  js  main.js  myjs  node_modules  package.json  tpl
```

目录命名很清晰



#### 调试功能

electron自带调试. 而且fofa的开发非常友好, 所有代码只有`com`目录的代码混淆了, 而且用的还是

```javascript
/*  This obfuscated code was created by Javascript Obfuscator Free Version.*/
/*  Javascript Obfuscator Free Version can be downloaded here              */
/*  http://javascriptobfuscator.com                                        */
```

注释也很友好,直接打开就看到

```
  // 指定一个入口的html文件
  mainWindow.loadURL('file://' + __dirname + '/tpl/login.html');

  // 打开调试工具，其实就是chrome的那套调试工具
    //  mainWindow.webContents.openDevTools();
```



直接解开注释就行了.

` mainWindow.webContents.openDevTools();` 



#### 重打包

重打包很简单

`asar pack fofa app.asar`

直接覆盖原`app.asar`就行了

上面的修改效果如下

![Screenshot_5.png](https://i.loli.net/2019/01/25/5c4b21a447162.png)



很眼熟吧, 就是chrome的调试功能

然后我们需要寻找功能点.其实很简单,fofa的网页都放在`tpl`目录了. 在页面查看元素对比一下就知道了.  我们找到`tpl\edit-poc.html`

网页没混淆... 功能点也没在混淆的js里,直接在`script`标签里了(我白解密了)..

看扫描按钮的文字是`开始扫描`, 直接在网页搜索一下, 直接搜到功能代码了..开发有良好的注释习惯,让我们为fofa开发点赞

```
$(".start_scan").click(function() {//开始扫描
		bugnum = 0;
		$('#san_result').html('');//默认清空 ,显数据
		$('#san_result1').html('');//默认清空 ,显提示
		isvipaaaa();
		$('#san_result1').html('');
		$('#san_result_have').html('');
		$('#san_result_no').html('');
		cvs_content = [];
	    Buglist = '';
	});
```



查看`isvipaaa`函数

主要代码

```
        if(data.fcoin == 0 && $("#scan_free").val() != 100 && UserInfo.vip_level != 2) {                                    $('#san_result1').append('<tr><td colspan="4"rowspan="4">你的账号fofa币不足不能进行扫描。</td></tr>');                    return;                
        }                
        if(($("#scan_free").val() != 100 && UserInfo.vip_level !=2)||($("#scan_free").val() =='' && UserInfo.vip_level == 2) ) {                    var scanok = scan_sure();                   
        if(scanok == false) {    
        return         
        }               
        }
        if(data.is_verified){
            /*扫描代码*/
        }else{
              $('#contentarea').html("");
                    wrongmas1("对不起你还没有通过审核,请联系管理员!!");
        }
```

可以看到就是三个`if`, 直接删除验证代码就完了..

```javascript

	function isvipaaaa() { //扫描
		$.ajax({
			url: login_url + '?email=' + UserInfo.email + '&key=' + UserInfo.key,
			dataType: 'json',
			type: 'get',
			success: function(data) {
					$('#query_loading_tr').css('display', '');
					$(".but_bc").click();
					$('#scan_rate').css('width', '0%');
					$('#scan_rate_value').html('0%');
					$('#san_result').html('');
					$('.start_scan').attr("disabled", true);
					$('.stop_scan').attr("disabled", false);
					$('.export_scan').attr("disabled", false);
					$('.Report_scan').attr("disabled", false);
					// $('#graphbox').css('display','');
					var FileName = $('#FileName').val();
					
					if(from == 'store') {
						scanPoc(temp_params2, 1, $("#scan_free").val());
					}else{
						scanPoc(temp_params2, 0, $("#scan_free").val());
					}
```



让我们看看效果,重打包替换

![Screenshot_6.png](https://i.loli.net/2019/01/25/5c4b2518939b5.png)





#### 尾声

感觉这个功能限制并没有什么意义, 本质上是执行一条系统命令用fofascan扫描而已
