---
title: Fuzz
date: 2018-09-02T22:40:07+08:00
lastmod: 2018-09-02T22:40:07+08:00
categories: ["网络安全"]
tags: ["fuzz"]
---

fuzz工具非常多 如`libfuzz` `honggfuzz` `KernelFuzzer` 也有专注进行web fuzz的`wfuzz` 但是fuzz功能可以分成两种 只是生成测试用例和检测程序使用测试用例后异常  这次使用radamsa和afl作为这两类工具

## radamsa
### 安装
官方给出的命令
```bash
$ # please please please fuzz your programs. here is one way to get data for it:
$ sudo apt-get install gcc make git wget
$ git clone https://gitlab.com/akihe/radamsa.git && cd radamsa && make && sudo make install
$ echo "HAL 9000" | radamsa
```

### 如何使用
```bash
 $ echo "aaa" | radamsa
 aaaa
```
`echo "aaa"`可换成任意输出内容的命令 如`cat test.txt`


常用选项
```
-o / --output 指定输出方法
-o -    输出到终端
-o output.txt 输出到文件
-o :80 网络请求 适用于对服务器程序fuzz

--seek num 指定随机数 用来方便复现

-g | --generators  指定输入
-g stdio 默认从命令行输出获取
-g file filename 读文件
-g random 随机数据

-n 生成数量
```

### fuzz 命令行程序
#### 从命令行读取数据
典型就是`md5sum`
测试fuzz命令例子
`echo "test" | radamsa | md5sum  - `

当然不可能fuzz一次就执行一行命令 编写一个简单的脚本
shell or python? python!

```
import os                                                                                                                                                                      
while True:                                              
   ret = os.system('echo "test" | radamsa | md5sum -')                                                                                                                       
```

只是简单的单线程执行 但是存在两个问题 如何记录崩溃和崩溃输入 这个时候`-seek`就能使用了 同时通过判断`system`函数的返回值 修改为

```
import os                                                                                                                               num = 0                                       
while True:                                              
   ret = os.system('echo "test" | radamsa -seek {}| md5sum -'.format(num))
   num = num +1
   if ret != 0:
      with open（"crash.txt","w+"） as f
      f.write("{}\n".format(num))                                                                                                   
```

我们的第一个fuzz程序(虽然这个脚本很简陋)


### 文件fuzz
和命令行主要区别只是读取方式和 文件格式一般会有一定要求 否则不能进入程序执行流程.所以一般fuzz严格的文件格式要使用专门的生成框架.

生成fuzz文件方法很简单
`echo "test" | radamsa --output 'testfile' `

读取文件生成
`radamsa --output 'testfile'  -g file testfile `

### 网络fuzz
虽然`radmasa`提供了这个选项 由于协议格式基本都是一个错误字节就报错 所以推荐使用`Mutiny`来进行网络fuzz
`echo "test" | radamsa --output ':80'`

## afl
### 介绍
这款工具除了可以自动从输入fuzz 还能自动检测崩溃 超时. 最大亮点是使用了 编译器插桩 在运行时通过编译时插入的代码可以了解到代码运行路径 覆盖率等信息

### 安装
```
wget http://lcamtuf.coredump.cx/afl/releases/afl-latest.tgz
tar -xf afl-latest.tgz
cd afl*
make
make install
```

### 使用
常用参数
```
alf-fuzz programs
-i 样本目录 fuzz时会使用这些样本生成fuzz文件
-o 输出目录 生成fuzz文件存放目录
-n 普通fuzz模式 只有检测崩溃功能 不能检测路径覆盖率
-q qmeu模式

```

例子
```
afl-fuzz  -i test/markdup/ -o out -n md5sum -  
```



## 实际操作
从github选一个作为实际对象 我使用高级搜索 指定>500star c语言的项目后随意找了一个
```
git clone https://github.com/samtools/samtools
```

实际选择指南
1. star星数是很好的指标 代表发现的漏洞的影响力
2. 选择 c/c++
3. 优先选择可以直接编译成程序的 fuzz库还需要去学习怎么写成库入口程序进行fuzz
4. 优先选择有完善的测试用例的 如jpg xml 或者自带
5. 代码越多漏洞越多
6. 如果是本地命令行程序或难以利用的程序 做好发现漏洞被忽略的准备


需要使用afl-gcc执行了编译  常用将gcc替换成afl-gcc的方法 (如果不经常进行编译)
```
CC=你的afl路径/afl-gcc
export CC
```
然后
```
make
```
实际使用编译工具不同 可能需要查询文档 当然你只fuzz不做其他编译直接`mv afl-gcc gcc`也行


编译
```
apt-get install autoconf automake make gcc perl zlib1g-dev libbz2-dev liblzma-dev libcurl4-gnutls-dev libssl-dev libncurses5-dev
git clone https://github.com/samtools/htslib                                          
git clone https://github.com/samtools/bcftools
cd samtools
autoheader
autoconf -Wno-syntax
./configure
```
确认输出中的 `checking for gcc... xxxx ` 是你的afl-fuzz 如果不是请执行上面的替换方法

```
make
```

这个程序提供了测试用例在`test`目录 如果fuzz其他程序没有提供 就需要自己寻找
需要注意几点
1. 尽量覆盖全部可能的格式
2. 畸形并符合文件格式
3. 不要存在大量无用数据 像一个3000像素大小的红色正方形图片比一个30像素的 在fuzz时并没有功能提升 只会让fuzz程序大量修改到没有用的图片数据区  

这个`samtools` 有多个功能 我们测试split这个功能 (q:为什么 a:因为我只在这个找到了崩溃 你想试试其他功能和用例也可以)
```
afl-fuzz -i test/markdup/ -o out samtools split @@  
```

稍等一会 就可以看到产生了崩溃
![](https://i.loli.net/2018/09/23/5ba6a9cb52673.png)

停止后 文件保存位置 `你选择的输出目录这次是out/crashes/`

测试崩溃
`./samtools split out/crashes/id:000000,sig:11,src:000015,op:flip2,pos:2 `
应用崩溃了
`fish: “./samtools split out/crashes/id…” terminated by signal SIGSEGV (Address boundary error)`


## 之后?
可以选择构造exp当自己的0day 或者提交给开发者 我是倾向于提交给开发者的 而且构造exp已经有很多书籍

### 提交是什么
通知开发者程序存在问题 让开发者进行修复。

### 怎么提交?
根据程序开发者的不同 具体可能是(非全面)
1. 小型商业公司 无漏洞奖金计划  我们可以从网上找到它的联系邮件 邮件通知
2. 大型商业公司 有漏洞奖金计划 使用计划中的提交方式
3. 开发者 非开源  邮件通知
4. 开发者 托管在github等 如果问题不大可以直接使用issue 如果是远程利用之类请通知开发者

一般都需要提供以下信息
1. os信息
2. 程序版本信息
3. 崩溃样本
4. 其他信息


### 为什么要提交?
修复漏洞 防止用户收到攻击 为网络安全做贡献 当然也有其他现实因素 比如危害太小不提交也没用 简历加分等等
