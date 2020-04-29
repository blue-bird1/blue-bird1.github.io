---
title: Codeql入门教程
date: 2020-04-30T01:57:12+08:00
lastmod: 2020-04-30T01:57:12+08:00
categories: ["网络安全"]
tags: ["代码审计"]
description:
---

## Codeql 入门教程

> 首发于先知社区

codeql是一个可以对代码进行分析的引擎, 安全人员可以用它作为挖洞的辅助或者直接进行挖掘漏洞,节省进行重复操作的精力 

### 安装

虽然官方提供了可以进行查询的网站 但是由于速度不快和一些c/c++项目 需要自定义编译命令来编译 实际上在网站是不能查询的 

首先找一个放codeql的目录  作者用的是`/opt/codeql` 

然后从[这里](https://github.com/github/codeql-cli-binaries/releases)下载后解压到目录 然后下载semmle的库 

执行 `cd codeql && git clone https://github.com/Semmle/ql  ` 

完成后 目录下应该有两个目录 `codeql  ql`

接下来安装vscode插件 在插件市场直接搜索codeql即可  编写时安装量只有3k多 说明用codeql的群体暂时还不多.



### 创建数据库

使用`codeql database create` 来创建一个用于查询的数据库 `--language=python`指定语言是python

例子

`codeql database create ./codeql -s .  --language=python`

在这种解释性语言上并不困难 问题在于对于c编译语言 需要用`--command=xxx`提供编译命令 虽然codeql会自动检测编译系统 但是在一些项目上不行  这也导致你编译不了的项目就用不了codeql

![image.png](https://i.loli.net/2020/04/22/J5vogNVEkbyDc7f.png)

在vscode把创建的codeql目录添加为数据库 就可以正式准备开始查询了



### hello world

codeql语言的查询格式如下

```
from int i where i = 3 select i
```

和sql比较像 `from`定义变量  `where` 声明限制条件 `select` 选择要输出的数据



可以使用的定义只有类和函数 例子代码

```
// 函数
predicate Name(int i) {
  i >5
}

// 类声明
class Name extend int {
   // 类变量声明
   int i
   // 覆盖父类函数
   override string func(){
   
   }
}
```



导入包语法和python一致 也是`import 名字`



作为每个语言都有的惯例 运行一下以下代码吧

`select "hello world"`



### 审计使用

在这里你应该有了自己的数据库了   作者选取的是python的一个django项目 (很遗憾的是由于动态语言的特性 python的污点跟踪效果不怎么好)  

codeql 支持的语言有`python` `java` `JavaScript` `c/c++`  `c#` `go`

并没有安全人员最喜欢目标用的语言 php, 也不用对以后抱太大期望 以php的动态特性和开发人员动不动就全局变量或者动态字符串导入文件的做法  污点跟踪和变量分析也没法用    



进行代码查询 首先要导入对应的编程语言包

```
import python
```



需要注意的是不同语言包的使用方法不一样 而且目前的文档不是很好 也没有全面的教程 

作者查https://help.semmle.com/qldoc/python  把python改成其他语言也能进去对应的文档



codeql的python库把对象分为了几种类型分别是

`Scope` 作用域 像函数或者类

`Expr` 表达式 像 `1+1`

`Stmt` 语句 例如 `if(xxx)`

`Variable` 变量

作为代码审计的开始   让我们先看看这个库调用的危险函数 在这里查了 django的重定向函数`redirect`

```
import python

from Call c ,Name n where c.getFunc() = n and n.getId() = "redirect" 
select c,"redirect"
```

我们选择了`call`和`name`变量, call是一个函数调用 

然后调用 `c.getFunc()` 来获取调用的函数, 为什么函数是一个`Name`呢

在python中 `test()` 实际上是对test这个变量进行调用 而在语法树上`test`是一个变量名 

最后我们要求`n.getId()` 获得的名字是`redirect `



可以发现这里能查到的都是`redirect()` 而不是`xx.redirect`

如果我们想要寻找`request.GET.get(xxx)`的调用 必须使用`Attribute`

`Attribute.getName` 获取自身名字  `Attribute.getName` 获取.之前的`Expr` 在我们的需求中它还是一个`Attribute` 因为它前面还有`request.`

```
import python

from Attribute a,Attribute b  
where a.getName() = "get" and a.getObject() =b 
and (b.getName() = "GET" or b.getName() = "post")
select a,"get request var"
```



可以发现随着查询复杂度的增加 代码行数在不断增加 这个时候就应该使用函数来解耦

假设我们查询一个`Expr` 像上面的例子 但是不想查到`test` 或者`debug` 开头的文件 在`Expr`或者`Stmt`都可以通过`getLocation()`来获取当前位置  可以写一个函数

```
predicate isNoTest(Expr e){
  not e.getLocation().getFile().getBaseName().matches("debug%") and 
  not e.getLocation().getFile().getBaseName().matches("test%")
}
```



通过不断添加限制条件 在代码审计中可以锁定自己想要看到的函数调用. codeql不仅如此,还可以通过结合判断条件来寻找自己目标中的代码 

例如我们希望找到一个函数中有获取请求数据并赋值的语句 还进行了重定向

首先作为一个赋值语句的终点 `.get(xx)` 是一个调用 添加

```
select Call c
// 省略
and c.getFunc() = a
```

然后添加一个赋值语句 要求右端是上面的调用

```
select Assign  assign
// 省略
and  assign.getValue() = c
```

再要求它们的作用范围是同一个函数

```
from Function f
// 省略
and (assign.getScope() = f and  redirectCall.getScope() =f)
```

最后代码

```
import python

from Attribute a,Attribute b,Call c, 
 Function f, Assign  assign, Name n,
 Call redirectCall,Name n1
where a.getName() = "get" and a.getObject() =b  
and (b.getName() = "GET" or b.getName() = "post")
and f.getAStmt() =  assign
and c.getFunc() = a
and  assign.getValue() = c
and  redirectCall.getFunc() = n1  and n1.getId() = "redirect"
and (assign.getScope() = f and  redirectCall.getScope() =f)
select f 
```

也可以查询变量是否直接进行危险函数,但是由于赋值和各种字符串操作之类的关系 应该属于污点分析的内容了





### 后言

这篇教程讲了如何去用codeql去做代码审计辅助.在拥有思路后去编写这种查询的最大难点 就是文档差了 使用人数少 你无法谷歌到xxx如何去查询 只能自己去查文档去查那些函数到底怎么用.











 





 
