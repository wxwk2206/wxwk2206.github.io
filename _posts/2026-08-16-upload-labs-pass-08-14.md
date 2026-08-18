---
title: "文件上传靶场 Upload-Labs 通关笔记（Pass-08 ~ Pass-14）"
date: 2026-08-16 12:00:00 +0800
categories: [Web安全, 文件上传漏洞]
tags: [upload-labs, 文件上传, 漏洞复现, webshell, 靶场]
excerpt: "Upload-Labs 第八到十四关：末尾加点、::$DATA、点空格点、双写后缀、%00 截断与图片马+文件包含的绕过思路。"
---

## Pass-08 后缀名末尾加点绕过

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.$file_name;
            if (move_uploaded_file($temp_file, $img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件类型不允许上传！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

这一关没有删除文件名末尾的点
![Pasted image 20260817182204](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817182204.png)

访问8.php. 出现空白，上传webshell成功
![Pasted image 20260817182145](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817182145.png)

## Pass-09 Windows ::$DATA 绕过

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
            if (move_uploaded_file($temp_file, $img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件类型不允许上传！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

这一关没有去除字符串::$DATA
![Pasted image 20260817182447](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817182447.png)

访问http://upload:8012/upload/202608171824215102.php
![Pasted image 20260817182812](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817182812.png)

## Pass-10 点+空格+点 绕过（deldot 只删一次）

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.$file_name;
            if (move_uploaded_file($temp_file, $img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件类型不允许上传！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

源码中删除点和空格，只删除了一次，我们使用点+空格+点进行绕过
![Pasted image 20260817182950](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817182950.png)

出现空白，上传成功
![Pasted image 20260817183125](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817183125.png)

## Pass-11 双写后缀绕过（str_ireplace 替换为空）

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array("php","php5","php4","php3","php2","html","htm","phtml","pht","jsp","jspa","jspx","jsw","jsv","jspf","jtml","asp","aspx","asa","asax","ascx","ashx","asmx","cer","swf","htaccess","ini");

        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = str_ireplace($deny_ext,"", $file_name);
        $temp_file = $_FILES['upload_file']['tmp_name'];
        $img_path = UPLOAD_PATH.'/'.$file_name;        
        if (move_uploaded_file($temp_file, $img_path)) {
            $is_upload = true;
        } else {
            $msg = '上传出错！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

这一关的漏洞点在 `$file_name = str_ireplace($deny_ext,"", $file_name); `使用`str_ireplace()`**不区分大小写**，把黑名单后缀字符串替换为空字符串。
所以这一关可以使用双写php进行绕过，例如写成pphphp，将黑名单后缀替换为空字符串之后pphphp就会变成php了
![Pasted image 20260817184347](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817184347.png)

出现空白，上传成功
![Pasted image 20260817184416](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817184416.png)

## Pass-12 %00 截断绕过（GET）

```php
$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $ext_arr = array('jpg','png','gif');
    $file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
    if(in_array($file_ext,$ext_arr)){
        $temp_file = $_FILES['upload_file']['tmp_name'];
        $img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;

        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        } else {
            $msg = '上传出错！';
        }
    } else{
        $msg = "只允许上传.jpg|.png|.gif类型文件！";
    }
}
```

本关卡可以使用%00截断绕过

更改magic_quotes_gpc配置：D:\software\phpstudy\phpstudy_pro\Extensions\php\php5.2.17nts这个目录下的php.ini文件中更改
PHP 5.2.x 版本默认开启 `magic_quotes_gpc`，该配置会自动转义 GET/POST/COOKIE 中的特殊字符：空字节 `%00` 会被转义为字面量 `\0`（反斜杠 + 数字 0），不再是空字节，无法触发截断。
![Pasted image 20260817213819](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817213819.png)

POST /Pass-12/index.php?save_path=../upload/12.php%00 HTTP/1.1
![Pasted image 20260817214123](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817214123.png)

访问此路径http://upload:8012/upload/12.php 出现空白，上传webshell成功
![Pasted image 20260817214336](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817214336.png)

## Pass-13 %00 截断绕过（POST）

第十三关和第十二关是差不多的，只不过是接受值变成了post,她两的差别呢就是get会自行解码，post不会自行解码，我们需要对%00进行编码,选中%00右键，按下图操作来
![Pasted image 20260817215637](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817215637.png)

编码好，我们就可以上传了
![Pasted image 20260817215843](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817215843.png)

访问此路径http://upload:8012/upload/13.php 出现空白，上传webshell成功
![Pasted image 20260817215955](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817215955.png)

## Pass-14 图片马 + 文件包含漏洞

制作图片马
在图片码的末端加一行一句话木马或者在终端执行命令
```
copy /b cat.jpg + shell.php shell.jpg     # Windows
        正常图片   木马文件   输出文件
```
![Pasted image 20260817221201](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817221201.png)

![Pasted image 20260817221147](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817221147.png)

任务里面说明了这关是有一个文件包含漏洞的，就是include.php
![Pasted image 20260817223357](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817223357.png)

我们直接利用这个文件包含漏洞连接一下蚁剑，测试到成功上传webshell
![Pasted image 20260817223321](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817223321.png)
![Pasted image 20260817223250](/assets/img/posts/upload-labs-pass-08-14/pasted-image-20260817223250.png)
