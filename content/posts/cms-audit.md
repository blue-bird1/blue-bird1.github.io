---
title: Cms代码审计方法
date: 2019-12-01T20:26:55+08:00
lastmod: 2019-12-01T20:26:55+08:00
categories: ["网络安全"]
tags: ["代码审计"]
description:
---
### 前言
记录cms挖掘漏洞的几种下手方法


### 基于危险函数
最常见的应该是这种了 通过搜索常见的危险函数如`assert|eval|system|file_put_contents|unserialize`
 来快速搜索可能存在的漏洞点 然后再追溯函数参数的引入位置.
 
寻找速度很快. 
能直接找到的漏洞比较低级.
在使用框架的cms上 搜索上述函数基本都应该搜到框架里面去了. 
对框架cms作用不大. 

但如果拥有前置框架知识 则可以用框架内的危险函数 代替上述函数.如thinkphp的 where order等sql函数 如果使用字符串做参数 将不会进行过滤. 同时由于使用了框架简化代码 这种方法速度将非常快.


### 基于输入点
基于危险函数搜索存在一个问题, 如果搜索结果过多,然后大部分输入实际上并不是由我们可控的,会导致效率很低. 
而基于输入点 则是按着只有我可控的输入才能导致漏洞的思路. 首先确定输入点 然后再按流程审计代码. 

这种方法速度不快, 但能找到的漏洞类型全面. 而在框架型cms下 由于封装很多 这种方法需要进入的函数很多 速度更加慢了.


### 基于信任
被开发信任的外部输入值出现问题往往将会是漏洞.基于信任过程的审计非常直接，但考验对代码的熟悉度

#### 验证
首先需要确认是否存在一些全局验证　和这些过滤器的实际效果是什么．
然后检查这些验证代码使用的位置(对象毫无疑问肯定是我们的输入)

#### 底层函数
在确认验证是否有误时 必须知道底层函数的安全性.
首先不管如何封装 最底层函数都是无防护的 开发人员假设上层调用提供了足够的安全验证.而开发人员如果假设底层函数做了安全防护. 这一误差往往会导致漏洞的出现.
通过审计这些底层函数 来了解那些底层函数是不安全的. 追溯到cms实际使用的函数,就可以知道这些函数的安全性需要什么样的验证来保障.


####　外部输入安全验证
在了解了函数的安全性后,检查外部输入的安全验证.
确认安全验证是否良好．比起从输入点审计全部代码的更快更直接.因为大部分代码是与实际漏洞无关的.
了解安全验证后,检查是否存在全局过滤.

#### 业务逻辑
到实际业务逻辑时, 我们已经对cms的验证了如指掌,对于漏洞的搜索,从已知函数下手会比较快.直接搜索,从搜索结果基本可以确认那些是可能有危害的.(例如知道函数传入字符串才可能有漏洞,我们忽略其他参数为数组的搜索结果.)

结合我们了解的全局过滤(例如转义了所有标签)也可以忽略一部分.
然后在搜索结果 找到开发使用的过滤函数 基本可以得出这个漏洞到底存不存在了.
也就是一个判定,在这么多验证下是否能达成这个函数所需要的安全性.如果不能就是漏洞

### 实战
以yumyecms为例
从搜索危险函数下手 
![2019-12-01 21-50-55 的屏幕截图.png](https://i.loli.net/2019/12/01/OmnRiW5qAkP67ef.png)
令人失望的是 搜索到的结果不多 并且大部分都在类库文件里 检查剩下的反序列化函数 发现也是反序列从数据库中查询出来的数据.



快速确认这个框架是否使用了获取输入的函数,幸运的发现这个cms仍然在使用`_
GET`等超全局变量 进行搜索
![2019-12-01 21-59-29 的屏幕截图.png](https://i.loli.net/2019/12/01/rJ59XLDiVy2IcCO.png)

首先我们可以忽略仅出现在判断语句里的结果,然后点开其他结果后发现 对变量进行了`usafestr`的过滤 并直接拼接到sql语句.在不查看这个过滤存在的情况下先认为这个变量已经是安全的. 
查看所有前台可访问函数内变量后 发现未经过编码的只有`core/app/shop/alipay.php`  但沮丧的发现里面对提交参数进行了其他验证.


对此只能对代码进行更深一步的了解了 我们先从简单的sql注入入手 .

在之前的了解中我们可以知道`this->db`操作了数据库(实际上在这个有model层的cms 直接看model类就行了)  我们追着父类查找(其中使用了`init.php`定义的函数加载文件 但函数都很简单)  最终在`core/lib/model.class.php` 它使用了`core/lib/yymysqli.class.php`操作数据库 

确认`yymysqli`的过滤 可以总结

1. execute/insert方法的key参数/update的where参数是不安全的

根据这些信息 很快就能确认出`model`类的不安全函数

确认危险函数后 也需要确认外部参数的过滤 我们找到`usafestr`函数

```
function usafestr($string,$flitersql=1,$fliter_script=1) {
	// 过滤url编码
    $string=str_ireplace("%","",$string);
    $string = str_ireplace('%20','',$string);
    $string = str_ireplace('%27','',$string);
    $string = str_ireplace('%2527','',$string);

	$string=str_ireplace("\t","",$string);
    $string = str_ireplace('*','',$string);
    $string = str_ireplace("'",'',$string);
    $string = str_ireplace(';','',$string);
    //$string = str_replace("{",'',$string);
    //$string = str_replace('}','',$string);
	$string=str_ireplace("#","",$string);
	$string=str_ireplace("--","",$string);
	$string=str_ireplace("\"","",$string);
	$string=str_ireplace("/","",$string);
    $string = str_ireplace('\\','',$string);
	if($flitersql){
		$string=safestring::fliter_sql($string);
		}
	if($fliter_script){
		$string=safestring::fliter_script($string);
		}
	$string=safestring::fliter_escape($string);
	$string=htmlspecialchars($string);
	$string = str_ireplace("$", "&#36;", $string);
	$string = str_ireplace("\n", "<br/>", $string);	
	$string = str_ireplace('%','%&lrm;',$string);
	$string=addslashes($string);
    return $string;
}

```
其中的`fliter_sql` 和`fliter_script` 只是过滤常见关键字.
从这段代码可以看出 过滤了大量字符 最关键的单引号和双引号也被过滤了. 从这点可以确认我们寻找sql注入时绝不能使用单引号和双引号.

我们首先尝试搜索`select`这个危险函数 
![2019-12-01 23-37-20 的屏幕截图.png](https://i.loli.net/2019/12/01/bESAda8WwYgD4TJ.png)

虽然都不需要引号 但只有最后一处是可控的.
```php 
		empty($_REQUEST['selcart'])?messagebox(Lan('goods_least_one')):$selcart=yytrim($_REQUEST['selcart']);
		empty($_REQUEST['num'])?messagebox(Lan('error_parameter')):$numarray=yytrim($_REQUEST['num']);
	    $cartstr=implode(",",$selcart);
	    $cartlist=$this->db->select("select * from `#yunyecms_cart` where userid={$member["id"]} and id in($cartstr) order by addtime desc");

```
很明显的漏洞 这cms还没报错处理 随便写点就报错了 .

![2019-12-02 00-47-18 的屏幕截图.png](https://i.loli.net/2019/12/02/9ecnhR7oTtwSuYH.png)


