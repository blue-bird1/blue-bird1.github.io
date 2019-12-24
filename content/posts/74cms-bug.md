---
title: 对骑士cms的一次弱加密漏洞挖掘
date: 2019-12-24T12:10:30+08:00
lastmod: 2019-12-24T12:10:30+08:00
tags: ["网络安全", "漏洞挖掘"]
description:
---
### 对某cms的一次弱加密漏洞挖掘



#### 前言



之前在挖某cms漏洞, 由于是tp框架的老牌cms, 便不想机械性去看sql注入和xss之类的.开始探索这个cms是否有一些有趣的代码







#### 对加密算法的探索



很快我就发现这个cms使用了以下这段加密代码



```php

function decrypt($txt, $key = '_qscms') {

    // $txt 的结果为加密后的字串经过 base64 解码，然后与私有密匙一起，

    // 经过 passport_key() 函数处理后的返回值

    $txt = passport_key(base64_decode($txt), $key);





    // 变量初始化

    $tmp = '';

    // for 循环，$i 为从 0 开始，到小于 $txt 字串长度的整数

    for ($i = 0; $i < strlen($txt); $i++) {

        // $tmp 字串在末尾增加一位，其内容为 $txt 的第 $i 位，

        // 与 $txt 的第 $i + 1 位取异或。然后 $i = $i + 1

        $tmp .= $txt[$i] ^ $txt[++$i];

    }

    // 返回 $tmp 的值作为结果

    return $tmp;

}



function passport_key($txt, $encrypt_key) {

    // 将 $encrypt_key 赋为 $encrypt_key 经 md5() 后的值

    $encrypt_key = md5($encrypt_key);

    // 变量初始化

    $ctr = 0;

    $tmp = '';



    // for 循环，$i 为从 0 开始，到小于 $txt 字串长度的整数

    for ($i = 0; $i < strlen($txt); $i++) {

        // 如果 $ctr = $encrypt_key 的长度，则 $ctr 清零

        $ctr = $ctr == strlen($encrypt_key) ? 0 : $ctr;

        // $tmp 字串在末尾增加一位，其内容为 $txt 的第 $i 位，

        // 与 $encrypt_key 的第 $ctr + 1 位取异或。然后 $ctr = $ctr + 1

      //   echo ord($txt[$i]);

        $tmp .= $txt[$i] ^ $encrypt_key[$ctr++];

      //  echo $tmp;

    }

    // 返回 $tmp 的值作为结果

    return $tmp;

}

```







虽然我不会密码学,但我也知道异或加密是不安全的. 于是开始对这函数开始分析. 







#### 异或加密简介



异或的定义为 两个值相同时就返回0,否则返回1.  异或的特性为 对这个数进行两次异或会返回这个值本身.



它有以下性质 设密文为`A`  用来异或的密钥为`B`. 如果`B`二进制表示全为`0`  则写为0



``` =

A^0 = A

A^A = 0

(A^B)^C = A^(B^c)

(A^B)^A = B^(A^A) = B^0 = B 

```







例如 `0000000`^`10101111`,由于左侧都是0,所以右侧为0的还是0,1的还是1.导致并没有任何改变.







### 对加密函数的思考



为了将问题分解, 首先对`passport_key`函数分析.



```php

 $encrypt_key = md5($encrypt_key);

```



查看了`cms`对这个函数的调用, 实际传递的key是一个固定的`16`位随机生成数 . 所以爆破这个md5是不可能的. 但是可以发现实际上加密没用到这个`$encrypt_key`本身的值.



所以问题可以简化为 获取这个`$encrypt_key`的值.



```php

    // for 循环，$i 为从 0 开始，到小于 $txt 字串长度的整数

    for ($i = 0; $i < strlen($txt); $i++) {

        // 如果 $ctr = $encrypt_key 的长度，则 $ctr 清零

        $ctr = $ctr == strlen($encrypt_key) ? 0 : $ctr;

        // $tmp 字串在末尾增加一位，其内容为 $txt 的第 $i 位，

        // 与 $encrypt_key 的第 $ctr + 1 位取异或。然后 $ctr = $ctr + 1

      //   echo ord($txt[$i]);

        $tmp .= $txt[$i] ^ $encrypt_key[$ctr++];

      //  echo $tmp;

    }

```



这段代码可以总结为 用`$encrypt_key`对`$txt`逐位异或. 



从上面的知识知道 利用`A^0`的特性, 传递`0`,  返回值将也是`$encrypt_key`



也就是只要我们可控`$txt`, 甚至不需要任何解密操作, 直接就能从返回值得到密钥.







但是`cms`并未直接调用这个函数, 需要进一步分析`decrypt`函数



```php

 // 经过 passport_key() 函数处理后的返回值

    $txt = passport_key(base64_decode($txt), $key);





    // 变量初始化

    $tmp = '';

    // for 循环，$i 为从 0 开始，到小于 $txt 字串长度的整数

    for ($i = 0; $i < strlen($txt); $i++) {

        // $tmp 字串在末尾增加一位，其内容为 $txt 的第 $i 位，

        // 与 $txt 的第 $i + 1 位取异或。然后 $i = $i + 1

        $tmp .= $txt[$i] ^ $txt[++$i];

    }

```







这个函数将`passport_key`返回值两位两位异或后返回,导致返回值位数减半. 



考虑我们之前的利用, 在这个函数运行后得到的其实是8位密钥两两异或后的值.



如果对异或值进行爆破, 密钥的值范围也就是`php`的`md5`函数的返回值范围  `a-z0-9`. 







编写一个爆破函数



```php

function crack($char)

{

    $ret = array();

    // 可能的值范围

    $chars = '0123456789abcdefghijklmnopqrstuvwxyz';

    for ($i = 0; $i < strlen($chars); $i++) {

        for ($i2 = 0; $i2 < strlen($chars); $i2++) {

            // 如果异或后的值为要破解的字符 就加入返回数组

            if( ($chars[$i] ^ $chars[$i2]) ===  $char){

                $ret[] = $chars[$i].$chars[$i2];

            }

        }

    }

    return $ret;

}



```







尝试破解一个字符



`var_dump(crack("v"));`



实际返回为



```php

array(18) {

  [0]=>

  string(2) "A7"

  // 省略

 }

```







实际上单个字符的可能性空间是不定的,`18`是最少的了  而md5一共有32位字符 ,也就是最少也有`18^16=1.21439531e20`种组合,不可能两个字符两个字符的爆破成功.



不过这只是表象, 计算一下就会发现

设未知字符为`x1`,`x2`,爆破出来的是`y1`,`y2`,加密的两个字符是`z1,z2` 



证 

`(z1^x1)^(z2^x2) ` 根据上面的交换律解括号得到 `(x1^x2)^z1^z2` 然`(y1^y2)=(x1^x2)` 

得`(z1^x1)^(z2^x2)=(z1^y1)^(z2^y2)` 



所以只需要随便在可能里选一种就行.









#### 利用



已知加密是弱加密,只需要有一处可控输入和可知输出的接口就可以利用. 尝试搜索`decrypt`函数



![2019-12-23 23-37-31 的屏幕截图.png](https://i.loli.net/2019/12/23/2MLDQWvEzOAsiN9.png)



只发现这处函数



```php

    public function get_font_img(){

        $str = I('request.str','','trim');

        // 异或加密

        $str = decrypt($str,C('PWDHASH'));

        \Common\ORG\Image::buildString($str,array(100,50),'','png',0,false);

    }

```



满足条件,但是输出的是图片比较尴尬,尝试了一下全`\x00`



![image.png](https://i.loli.net/2019/12/23/MYl7UGFCTHSQExA.png) 虽说参数故意没添加干扰,这也没法肉眼辨认可能的不可视字符.



只能通过更改攻击载荷来使得下面的字符变得可视化，只需要最后再与攻击载荷再进行一次异或就行了．
















