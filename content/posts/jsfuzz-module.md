---
title: 使用jsfuzz对nodejs模块进行模糊测试
date: 2020-04-19T20:35:37+08:00
lastmod: 2020-04-19T20:35:37+08:00
categories: ["网络安全"]
tags: ["模糊测试","漏洞挖掘"]
description:
---

### 使用jsfuzz对nodejs模块进行模糊测试

####　前言

[jsfuzz](https://github.com/fuzzitdev/jsfuzz)是一个基于覆盖率指导的模糊测试工具,能对JavaScript/nodejs模块进行模糊测试. 只需要编写一个接受输入的函数即可. 

虽然nodejs是现代语言,即使出现了越界读写 也不会像c/c++一样直接导致安全问题. 但在审查一个复杂逻辑的nodejs模块时,仍是一个值得考虑的选项.



#### 下载安装

`npm i -g jsfuzz`



#### 实战

##### 编写模糊文件

一个标准的格式是

```javascript
function fuzz(buf) {
  // call your package with buf  
}
module.exports = {
    fuzz
};
```

更常见的是

```javascript
function fuzz(buf) {
    try {
        // 调用模块函数
    } catch (e) {
        // 检查是否是模块自己的自定义错误
        if (e.message.indexOf('xxx1') !== -1 ||
            e.message.indexOf('xxx2') !== -1 ) {
        } else {
            throw e;
        }
    }
}
```



作为一个例子 作者挑选了[mqtt-packet](https://github.com/mqttjs/mqtt-packet) 作为例子,这个模块代码不多,还提供了生成样本的函数,相当方便.

首先下载包

```
mkdir jsfuzz
cd jsfuzz
npm i mqtt-packet
```

通过阅读包文档,很容易就能写出(复制粘贴)使用这个包解析函数的代码

```javascript
var mqtt = require('mqtt-packet')
var opts = { protocolVersion: 4 } // default is 4. Usually, opts is a connect packet

function fuzz(buf) {
       var parser = mqtt.parser(opts)
       // 这个包自己产生的异常都会通过这个函数调用
       parser.on('error', function(error) {
           
       })
       parser.parse(buf)
}

module.exports = {
    fuzz
};
```

运行

`jsfuzz fuzz2.js `

输出大致如

```
#0 READ units: 0
#1 NEW     cov: 605 corp: 2 exec/s: 0 rss: 35.83 MB
#2 NEW     cov: 717 corp: 3 exec/s: 200 rss: 35.88 MB
#10 NEW     cov: 726 corp: 4 exec/s: 615 rss: 35.88 MB
#28 NEW     cov: 731 corp: 5 exec/s: 1058 rss: 36.07 MB
#43 NEW     cov: 747 corp: 6 exec/s: 1500 rss: 36.07 MB
#56 NEW     cov: 749 corp: 7 exec/s: 1444 rss: 36.07 MB
#63 NEW     cov: 773 corp: 8 exec/s: 1750 rss: 36.07 MB
#108 NEW     cov: 775 corp: 9 exec/s: 1607 rss: 36.3 MB

```

`NEW`表示发现了新的路径  

`cov`代表行数, `corp`代表的不同路径数 `exec/s`代表每秒执行的次数 `rss`代表内存占用

由于这个库代码量不大,预期上能产生崩溃最多几分钟,不然就不可能了. 模糊测试的效果和运行时间并不是线性的.

不过由于这次运行并未提供任何一个样本,指望模糊测试器能覆盖太多路径是不太现实的. 



##### 覆盖率

在终止运行后,jsfuzz将会产生一个覆盖率文件. 可以通过nyc生成覆盖率报告

```bash
npm i -D nyc 
node_modules/.bin/nyc report --reporter=html --exclude-node-modules=false 
```

输出路径在`couverage`目录,打开html文件后

页面如下,我们fuzz的包在`node_modules/mqtt-packet`, 可以看出覆盖率很低.点开后可以发现`parser.js`的覆盖率只有57%. 这是一个相当低的数值

![image_2020-01-19_15-37-35.png](https://i.loli.net/2020/01/19/V4HfF8tNyPgw29E.png)



#####　生成一些样本

可以通过`mqtt.generate(obj)` 生成正确的mqtt格式, 尝试用包自带的生成例子就能得到一个样本

```javascript
var mqtt = require('mqtt-packet')
var object = {
  cmd: 'publish',
  retain: false,
  qos: 0,
  dup: false,
  length: 10,
  topic: 'test',
  payload: 'test' // Can also be a Buffer
}
var opts = { protocolVersion: 4 } // default is 4. Usually, opts is a connect packet

console.log(mqtt.generate(object))
// 写入文件的方法 
var fs = require('fs')
fs.writeFile("test.txt", mqtt.generate(object),  "binary",function(err) { });
```



用mqtt-packet仓库说明里的mqtt object格式生成更多样本后得到了16个样本.

![image_2020-01-19_15-51-21.png](https://i.loli.net/2020/01/19/rOdIE48WsP5Yh7w.png)

 运行后

`jsfuzz fuzz2.js test_corpus/ `

可以看到路径数从50提高了67 并且在67个路径上爆出了一个错误

![image_2020-01-19_15-54-33.png](https://i.loli.net/2020/01/19/zGkrldgUZiCOyA4.png)

文本版

```
RangeError [ERR_BUFFER_OUT_OF_BOUNDS]: Attempt to write outside buffer bounds
    at boundsError (internal/buffer.js:70:11)
    at Buffer.readUInt8 (internal/buffer.js:238:5)
    at BufferList.<computed> [as readUInt8] (/home/bluebird/fuzz/jsfuzz/node_modules/bl/bl.js:1:33980)
    at Parser._parseByte (/home/bluebird/fuzz/jsfuzz/node_modules/mqtt-packet/parser.js:1:71555)
    at Parser._parseProperties (/home/bluebird/fuzz/jsfuzz/node_modules/mqtt-packet/parser.js:1:72858)
    at Parser._parseConnect (/home/bluebird/fuzz/jsfuzz/node_modules/mqtt-packet/parser.js:1:55778)
    at Parser._parsePayload (/home/bluebird/fuzz/jsfuzz/node_modules/mqtt-packet/parser.js:1:51576)
    at Parser.parse (/home/bluebird/fuzz/jsfuzz/node_modules/mqtt-packet/parser.js:1:49930)
    at Worker.fuzz [as fn] (/home/bluebird/fuzz/jsfuzz/fuzz2.js:1:1523)
    at process.<anonymous> (/usr/local/lib/node_modules/jsfuzz/build/src/worker.js:63:30)

```

一个尝试越界读导致的错误(nodjs会阻止越界读写buffer) 



查看覆盖率的话 发现已经达到了85% 简单浏览就能发现未能覆盖到的都是需要解析器使用协议的版本为5 而不是例子里的4.

![image_2020-01-19_16-03-43.png](https://i.loli.net/2020/01/19/QSChfLHVKOGBlYw.png)



#### 后言

在像nodejs这类现代语言,使用fuzz来测试module能发现的大多数只是bug,在检查了缓冲区越界读写后你没法利用它.能发现的安全漏洞常见是dos,用错误的输出导致内存占用过大或者崩溃. 

但JavaScript的依赖管理导致依赖了太多的包,如果你能发现一个底层包的问题,你将能对大量使用它的包进行攻击.



 



  

