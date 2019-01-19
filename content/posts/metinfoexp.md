---
title: metinfo3.5.18后台getshell分析
date: 2017-11-25T22:26:31+08:00
lastmod: 2017-11-25T22:26:31+08:00
categories: ["网络安全"]
tags: ["漏洞分析"]
---

此漏洞已经在最新版修复
### exp
 ` /admin/app/physical/physical.php?action=op&op=3&valphy=test|文件名&address=包含文件`
 
### 代码分析
查看关键代码
```php
case 3:
			$fileaddr=explode('/',$val[1]);
			$filedir="../../../".$fileaddr[0];  
			if(!file_exists($filedir)){ @mkdir ($filedir, 0777); } 
			if($fileaddr[1]=="index.php"){
				if($val[2]){
					Copyindx("../../../".$val[1],$val[2]);
				}
			}else{
			// 漏洞点
			switch($val[2]){
				case 1:
					$address="../about/$fileaddr[1]";
				break;
				case 2:
					$address="../news/$fileaddr[1]";
				break;
				case 3:
					$address="../product/$fileaddr[1]";
				break;
				case 4:
					$address="../download/$fileaddr[1]";
				break;
				case 5:
					$address="../img/$fileaddr[1]";
				break;
				case 8:
					$address="../feedback/$fileaddr[1]";
				break;
			

			}   
				$newfile  ="../../../$val[1]"; 
				
				Copyfile($address,$newfile);
				
			}
			echo $lang_physicalgenok;
			break;
```
我们可以看到我们可控参数`$address`和`newfile`传入了`Copyfile`
主要是程序员写代码的时候忽略了异常参数导致address参数没有被覆盖,应该添加不是正常参数时不执行`Copyfile`

查看`Copyfile`函数
```php
function Copyfile($address,$newfile){
	$oldcont  = "<?php\n# MetInfo Enterprise Content Management System \n# Copyright (C) MetInfo Co.,Ltd (http://www.metinfo.cn). All rights reserved. \nrequire_once '$address';\n# This program is an open source system, commercial use, please consciously to purchase commercial license.\n# Copyright (C) MetInfo Co., Ltd. (http://www.metinfo.cn). All rights reserved.\n?>";
	if(!file_exists($newfile)){
		$fp = fopen($newfile,w);
		fputs($fp, $oldcont);
		fclose($fp);
	}
}
```
此函数可以创建并写入文件
可以看到此函数我们可以控制文件名,不过单引号导致我们只能控制require_once参数.
不过也造成了文件包含漏洞,上传一个php代码头像,即可getshell


查看最新版本修复方法
```php
	default:
		 $address = "";
	break;
```
添加了不是正常参数时候默认为空
