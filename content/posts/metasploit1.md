---
title: metasploit python 模块
date: 2018-02-02T22:57:52+08:00
lastmod: 2018-02-02T22:57:52+08:00
categories: ["网络安全"]
---
metasploit在2017年尾将python作为官方支持语言,并且已经有python模块加入主分支.这使得我们开发metasploit模块可以不去学习ruby

### 为什么将python作为官方支持语言
1. 很多不是metasploit官方人员编程的模块都是使用python编写
2. 现在python流行程度非常高 很多渗透人员python熟练程度比ruby高

### metasploit的python模块是什么
主分支的一个python模块 https://github.com/rapid7/metasploit-framework/blob/778e69f92912c555e72bc3318278443126704b75/modules/auxiliary/dos/http/slowloris.py

python模块实际是通过json-rpc调用与metasploit通信

metasploit获取元数据如图(来自官方博客)
~~~
+------------+
| Metasploit |
|            |  Describe yourself  +-------------------+
|            +-------------------> |  some_module.py   |
|            |                     |                   |
|            |                     |                   |
|            |   Some metadata     |                   |
|            | <-------------------+                   |
|            |                     |                   |
|            |                     +-------------------+
|            |
|            |
+------------+
~~~

模块调用如图
~~~
+------------+
| Metasploit |  Do a thing with
|            |   these options     +-------------------+
|            +-------------------> |  some_module.py   |
|            |                     |                   |
|            |                     |                   |
|            |   A bit of status   |                   |
|            | <-------------------+                   |
|            |                     |                   |
|            |  Moar status        |                   |
|            | <-------------------+                   |
|            |                     |                   |
|            |  I found a thing    |                   |
|            | <-------------------+                   |
|            |                     |                   |
|            |                     +-------------------+
|            |
+------------+
~~~

### 将会发生什么
实际上对于原来的开发方式没有影响,完全可以使用原来的ruby编写方式.但是对于不熟悉ruby的开发者可以使用python来方便的编写模块

### python在metasploit能做什么
可以使用的和ruby模块并没有区别

## 如何编写一个python模块
首先需要导入需要的模块
~~~
#!/usr/bin/env python
# another:bluebird
from metasploit import module
~~~
这个metasploit实际上路径是 `lib/msf/core/modules/external/python/`

然后定义元数据 格式和ruby模块的一样.详细可参考[这里的文档](https://www.kancloud.cn/bluebird/metasploit/486941)

~~~
metadata = {
	# 模块名字
    'name': 'metasploit python module demo ',
    # 模块的描述
    'description': '''
        send a http request metasploit python module demo
     ''',
     # 模块作者
    'authors': [
        'bluebird', 
    ],
    # 编写时间
    'date': '2018-02-02',
    # 漏洞参考
    'references': [
       
     ],
     # 漏洞类型 只能在已有的类型选项 
    'type': 'dos',
    # 模块选项
    'options': {
        'rhost': {'type': 'address', 'description': 'The target address', 'required': True, 'default': None},
        'rport': {'type': 'port', 'description': 'The target port', 'required': True, 'default': 80},
     }}
~~~

然后一般应该定义一个`run`方法.这个demo输出了helloworld 
~~~
def run(args):
    module.log('helloworld')
   
~~~

最后定义主方法
~~~
if __name__ == "__main__":
    module.run(metadata, run)
~~~

让我们实际跑一下(注意请给你的python文件添加执行权限)
~~~
msf5 auxiliary(test/demo) > set rhost 127.0.0.1
rhost => 127.0.0.1
msf5 auxiliary(test/demo) > run

[*] Starting server...
[*] hello world
[*] Auxiliary module execution completed
msf5 auxiliary(test/demo) > 
~~~

