---
title: 文件上传靶场 Upload-Labs 通关笔记（Pass-01 ~ Pass-07）
date: 2026-08-10 12:00:00 +0800
categories:
  - 靶场
  - 文件上传漏洞
  - 漏洞复现
tags:
  - upload-labs
  - 文件上传
  - 漏洞复现
  - webshell
  - 靶场
excerpt: Upload-Labs 前七关：前端 JS 校验、MIME 绕过、黑名单不全、.htaccess/.user.ini 解析、大小写与空格加点绕过的实战记录。
---

## Pass-01 前端 JS 校验绕过

第一关，源码中是对前端做了限制
![Pasted image 20260816141244](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260816141244.png)

我们可以直接使用burpsuite绕过前端限制，在数据包中对文件类型进行更改
![Pasted image 20260816141356](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260816141356.png)

上方的响应数据包给了我们一个地址，我们访问一下
如下，出现空白内容就是上传webshell成功了（没成功应该是报错404，如下）
![Pasted image 20260816141208](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260816141208.png)
![Pasted image 20260816141619](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260816141619.png)

## Pass-02 MIME 类型（Content-Type）绕过

第2关
提示了本pass在服务端对数据包的MIME进行检查！
![Pasted image 20260817152725](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817152725.png)

在数据包中将Content-Type改成允许上传的类型，骗过MIME检查
![Pasted image 20260817153631](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817153631.png)

访问一下给我们的地址
如下，出现空白内容就是上传webshell成功了
![Pasted image 20260817153935](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817153935.png)

## Pass-03 黑名单不全之 php5 后缀绕过

这一关卡是黑名单过滤，但是过滤规则不全，我们可以使用php5进行绕过
![Pasted image 20260817154514](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817154514.png)

### 注意
如果是在phpstudy中启动的靶场要在D:\software\phpstudy\phpstudy_pro\Extensions\Apache2.4.39\conf\vhosts中更改upload_8012.conf文件的配置
如果不配置他是无法解析php5代码的
更改后的配置如下
```conf
<VirtualHost *:8012>
    DocumentRoot "D:/software/phpstudy/phpstudy_pro/WWW/upload"
    ServerName upload
    ServerAlias 
    FcgidInitialEnv PHPRC "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts"
    AddHandler fcgid-script .php .php1 .php2 .php3 .php4 .php5 .php6 .php7 .pht .phtm .phtml .phar .inc .phpinc
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php1
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php2
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php3
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php4
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php5
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php6
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .php7
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .pht
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .phtm
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .phtml
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .phar
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .inc
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .phpinc
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .png
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .jpg
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe" .jpeg
     # ⭐ 不区分大小写：所有 .php/.PHP/.pHp 等变体都交给 fcgid-script 处理
    <FilesMatch "(?i)\.php$">
        SetHandler fcgid-script
    </FilesMatch>
    # ⭐ 不区分大小写：所有 .png/.jpg/.jpeg 大小写变体也交给 fcgid-script 处理
    <FilesMatch "(?i)\.(png|jpg|jpeg)$">
        SetHandler fcgid-script
    </FilesMatch>
    # ⭐ 兜底 wrapper：任何未显式配置扩展名的 fcgid-script 都走 PHP-CGI
    FcgidWrapper "D:/software/phpstudy/phpstudy_pro/Extensions/php/php5.3.29nts/php-cgi.exe"
  <Directory "D:/software/phpstudy/phpstudy_pro/WWW/upload">
      Options FollowSymLinks ExecCGI
      AllowOverride All
      Order allow,deny
      Allow from all
      Require all granted
	  DirectoryIndex index.php index.html error/index.html
  </Directory>
  ErrorDocument 400 /error/400.html
  ErrorDocument 403 /error/403.html
  ErrorDocument 404 /error/404.html
  ErrorDocument 500 /error/500.html
  ErrorDocument 501 /error/501.html
  ErrorDocument 502 /error/502.html
  ErrorDocument 503 /error/503.html
  ErrorDocument 504 /error/504.html
  ErrorDocument 505 /error/505.html
  ErrorDocument 506 /error/506.html
  ErrorDocument 507 /error/507.html
  ErrorDocument 510 /error/510.html
</VirtualHost>

```

上方我们已经上传了3.php5这个文件，我们访问一下响应包返回的upload/202608171543162430.php5
如下，上传webshell成功
![Pasted image 20260817163051](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817163051.png)

## Pass-04 .htaccess 解析绕过

第四关源代码
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".php1",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".pHp1",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".ini");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //收尾去空

        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.$file_name;
            if (move_uploaded_file($temp_file, $img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件不允许上传!';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

审计完源代码发现没有禁止上传.htaccess文件，我们上传一个.htaccess文件，让所有 .png 都按 PHP 解析！
![Pasted image 20260817174005](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817174005.png)

上传.htaccess之后，我们再上传一个4.png，内容是一句话木马
![Pasted image 20260817174109](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817174109.png)

访问一下地址
![Pasted image 20260817173824](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817173824.png)

## Pass-05 .user.ini 自动包含绕过

第五关提示上传目录存在php文件（readme.php）
并且没有黑名单过滤.user.ini
我们上传一个.user.ini（内容：auto_prepend_file=shell.jpg）
![Pasted image 20260817175302](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817175302.png)

上传 shell.jpg（一句话木马，伪造图片头）
![Pasted image 20260817175514](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817175514.png)

访问http://upload:8012/upload/readme.php，并且能连接上蚁剑
![Pasted image 20260817175658](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817175658.png)

## Pass-06 后缀大小写绕过

第六关源码如下
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
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

这一关代码中没有转小写，直接使用PHP就能绕过
![Pasted image 20260817180129](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817180129.png)

访问http://upload:8012/upload/202608171801097244.PHP
显示白板，连接成功
![Pasted image 20260817181319](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817181319.png)

## Pass-07 后缀名首尾空格绕过

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
        $file_name = $_FILES['upload_file']['name'];
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        
        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
            if (move_uploaded_file($temp_file,$img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件不允许上传';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

这一关代码中没有首尾去空，直接加空格就能绕过
![Pasted image 20260817181817](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817181817.png)

出现空白，上传成功
![Pasted image 20260817181950](/assets/img/posts/upload-labs-pass-01-07/pasted-image-20260817181950.png)
