---
title: Metinfo3.5.19 getshell
date: 2017-11-28T22:26:19+08:00
lastmod: 2017-11-28T22:26:19+08:00
categories: ["网络安全"]
tags: ["漏洞分析"]
---

### metinfo后台getshellexp分析
漏洞版本`3.5.19`  漏洞文件路径`/admin/app/physical/physical.php` 
有趣的是`3.5.18`修复的后台getshell也是这个文件  
`3.5.18`漏洞分析 [点我](https://bbs.ichunqiu.com/thread-29582-1-1.html)
上次分析完后顺手审计一下那个文件，发现居然还有一个getshell漏洞。提交后没人理就发出来分享

#### 漏洞代码
```
elseif($action=="op"){
	// 不相关代码
	$val=explode('|',$valphy);
    // 不相关代码
	switch($op){
	   // 不相关代码
		case 3:
			$fileaddr=explode('/',$val[1]);
			$filedir="../../../".$fileaddr[0];  
			if(!file_exists($filedir)){ @mkdir ($filedir, 0777); } 
			if($fileaddr[1]=="index.php"){
				if($val[2]){
					Copyindx("../../../".$val[1],$val[2]);
				}
```
需要`op`参数为3才能进入漏洞代码流程，并且要正常执行，需要`fileaddr[1]='index.php' `和`$val[2]`也就是valphy参数要为`xxx|xxx/index.php|xxx`格式

查看`Copyindex`函数
```
function Copyindx($newindx,$type){
    if(!file_exists($newindx)){
        if($type==3){
            //生成产品栏目index
            $oldcont ="<?php\n# MetInfo Enterprise Content Management System \n# Copyright (C) MetInfo Co.,Ltd (http://www.metinfo.cn). All rights reserved. \n\$filpy = basename(dirname(__FILE__));\n\$fmodule=$type;\n\$cmodule='product_index';\nrequire_once '../include/module.php'; \nrequire_once \$module; \n# This program is an open source system, commercial use, please consciously to purchase commercial license.\n# Copyright (C) MetInfo Co., Ltd. (http://www.metinfo.cn). All rights reserved.\n?>";
        }else{
            $oldcont ="<?php\n# MetInfo Enterprise Content Management System \n# Copyright (C) MetInfo Co.,Ltd (http://www.metinfo.cn). All rights reserved. \n\$filpy = basename(dirname(__FILE__));\n\$fmodule=$type;\nrequire_once '../include/module.php'; \nrequire_once \$module; \n# This program is an open source system, commercial use, please consciously to purchase commercial license.\n# Copyright (C) MetInfo Co., Ltd. (http://www.metinfo.cn). All rights reserved.\n?>";
        }
        $fp = fopen($newindx,w);
        fputs($fp, $oldcont);
        fclose($fp);
    }
}
```
可以看到如果传入type参数不等于3,进入第二个流程.发现是单纯的字符串拼接,然后这个函数的参数我们都可控

