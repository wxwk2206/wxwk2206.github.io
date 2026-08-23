---
title: Pikachu 靶场复现（一）：XSS / CSRF / SSRF
date: 2026-08-21 00:00:00 +0800
categories:
  - 靶场
  - Web安全
  - 漏洞复现
tags:
  - Pikachu
  - XSS
  - CSRF
  - SSRF
---

本篇汇总 Pikachu 靶场中 XSS、CSRF、SSRF 三类漏洞的复现过程，按漏洞类型分节记录触发点与 payload。

## XSS - 01 反射型XSS（GET）

get型XSS，js代码直接拼接进url中的
![pasted image 20260818155854](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260818155854.png)

直接在这里输入payload：`<script>alert(1)</script>`
![pasted image 20260818160026](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260818160026.png)

如下，敲回车就会出现弹窗
![pasted image 20260818160118](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260818160118.png)
![pasted image 20260818160131](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260818160131.png)

## XSS - 02 反射型XSS（POST）

POST型XSS，这一关与上一关payload一样，与上一关的唯一区别就是传参方式是post
![pasted image 20260823203222](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823203222.png)

## XSS - 03 存储型XSS

这一关是存储型XSS，就是当我们写入xss之后，后续再次进入当前页面还是会执行XSS语句
![pasted image 20260823203452](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823203452.png)

再次进入
![pasted image 20260823203651](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823203651.png)

## XSS - 04 DOM型XSS

DOM型XSS在本地执行，不向网站发起请求，客户端的脚本程序可以通过DOM动态地检查和修改页面内容，不依赖服务器，而从客户端获得DOM中的数据在本地执行
这里是把用户输入拼接进了，如果我们把xss的payload拼接进去，这里就会执行，造成DOM型XSS
![pasted image 20260823205311](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823205311.png)

当我们输入'onclick=alert(1)>并点击what do you see时就会执行XSS
![pasted image 20260823205744](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823205744.png)

或者当我们输入' onmouseover="alert('xss')">并将鼠标悬停在what do you see时就会执行XSS
![pasted image 20260823205928](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823205928.png)

## XSS - 05 DOM型XSS-X

和上一关一样，只不过这次可以直接从IRL输入。与上一关的区别就是只要构造 URL，诱导受害者点开链接，就可以执行受害者浏览器上的 JS，更容易造成危害
当我们输入'onclick=alert(1)>并点击'>就让往事都随风,都随风吧时就会执行XSS
![pasted image 20260823211008](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823211008.png)
或者当我们输入' onmouseover="alert('xss')">并将鼠标悬停在'>就让往事都随风,都随风吧时就会执行XSS
![pasted image 20260823211042](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823211042.png)

## XSS - 06 XSS之盲打

存储型 XSS 的特殊场景，注入点和触发点不在同一个页面
这一关就是把XSS写入了用户后台
输入`<script>alert(/xss/)</script>`之后，进入/xssblind/admin_login.php就会出现如图的弹窗
![pasted image 20260823211730](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823211730.png)

## XSS - 07 XSS之过滤

当我们尝试输入`<script>alert(/xss/)</script>`之后，就会发现输入被过滤了
![pasted image 20260823213900](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823213900.png)

我们先尝试其他的payload，不使用script
使用：`<img src=x onerror=prompt(1);>` 就会发现绕过了
![pasted image 20260823214143](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823214143.png)

## XSS - 08 XSS之htmlspecialchars

查了一下htmlspecialchars()这个函数，介绍如下
`htmlspecialchars()` 是 PHP 函数，**把特殊符号转成 HTML 实体，用来防御 XSS，但这一关用的参数不全，防护被绕过**
我们可以使用单引号来绕过
payload：`' onclick=alert(1)//`

![pasted image 20260823220300](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823220300.png)
![pasted image 20260823220526](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823220526.png)

## XSS - 09 XSS之href输出

使用payload：`' onclick=alert(1)//` ，发现输入被过滤了
![pasted image 20260823221029](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823221029.png)

那我们尝试使用`javascript:`伪协议进行绕过
payload：`javascript:alert(1)`
![pasted image 20260823221236](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823221236.png)

## XSS - 10 XSS之js输出

输入：`<script>alert(1)</script>` 之后查看源码发现没有任何过滤
![pasted image 20260823221659](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260823221659.png)

那我们可以提前闭合$ms='
payload：`</script> <script>alert(1)</script>`
源码如下：

```html
<script> 
$ms='</script> <script>alert(1)</script>'; 
if($ms.length != 0){ 
    if($ms == 'tmac'){ 
        $('#fromjs').text('tmac确实厉害,看那小眼神..') 
    }else {
        // alert($ms); 
        $('#fromjs').text('无论如何不要放弃心中所爱..') 
    } 
}
</script>
```

这里实际被浏览器拆成了三段：
第一段（第一个 script，语法不完整，会报错但不影响后面）：

```html
<script> 
$ms='
</script>
```

第二段（新的 script 标签，正常执行，弹窗）：

```html
<script>alert(1)</script>
```

第三段（普通文本 / 残缺代码，不执行）：

```html
'; 
if($ms.length != 0){ 
    ...
}
</script>
```

## CSRF - 1 CSRF（GET）

这里登录进去之后发现一个可以修改个人信息的地方
![pasted image 20260819164655](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819164655.png)
![pasted image 20260819165126](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819165126.png)

我们对修改个人信息这里进行抓包就会发现修改的参数都是通过url传参的
![pasted image 20260819164958](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819164958.png)

我们直接将这个信息伪造一下，让别人去点的话，别人的浏览器如果访问过这个网站，点我们的链接的时候，浏览器就会自动携带上当前网站的cookie，达到修改信息的目的，造成了CSRF
我们自己尝试一下输入这样一个链接，访问这个链接之后，个人信息就被修改了
![pasted image 20260819165644](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819165644.png)
![pasted image 20260819165701](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819165701.png)

## CSRF - 2 CSRF（POST）

post型的CSRF和get型差不多，只不过是请求内容放入了请求体，如下所示
![pasted image 20260819170216](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819170216.png)

post的请求伪造可以制作一个钓鱼网站（自动提交表单），然后诱导用户点击钓鱼网站链接
这里的钓鱼网站是可以让burp自动生成的
![pasted image 20260819170442](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819170442.png)
![pasted image 20260819170532](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819170532.png)

生成之后可以诱导用户点击来实现攻击，这里我们本地模拟一下
![pasted image 20260819171539](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819171539.png)
这里显示也是修改成功了
![pasted image 20260819171529](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819171529.png)

burp中生成的钓鱼网站不是自动提交表单，这里我们可以更改一下，也是可以修改成功
![pasted image 20260819173059](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819173059.png)
![pasted image 20260819173207](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819173207.png)

## SSRF - 1 SSRF（curl）

点一下那个来读一首诗吧，就会发现url请求里面拼接了其他的url
![pasted image 20260819180303](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819180303.png)
![pasted image 20260819180350](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819180350.png)

我们知道SSRF存在的必要条件是可控地址、服务发包、过滤失效、网络通。
靶场里面肯定是SSRF，我们直接利用一下
`http://pikachu:8013/vul/ssrf/ssrf_curl.php?url=http://www.baidu.com`
![pasted image 20260819174948](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819174948.png)

## SSRF - 2 SSRF（file_get_contents）

![pasted image 20260819181025](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819181025.png)

与上一关一样，这里肯定存在协议为file的漏洞的，我们直接漏洞利用一下
`http://127.0.0.1/pikachu/vul/ssrf/ssrf_fgc.php?file=C:/Windows/win.ini`
![pasted image 20260819181303](/assets/img/posts/2026-08-22-pikachu-xss-csrf-ssrf/pasted-image-20260819181303.png)
