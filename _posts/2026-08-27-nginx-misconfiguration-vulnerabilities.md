---
title: "Nginx 配置错误导致的安全漏洞复现"
date: 2026-08-27 14:00:00 +0800
categories: [漏洞复现, Web安全]
tags: [Nginx, 配置错误, 解析漏洞]
---

## 一、产品介绍
**Nginx** 是高性能 Web 服务器/反向代理，常用于搭配 PHP-FPM 运行 PHP 应用。
## 二、漏洞复现
搭建靶场

```
cd vulhub\nginx\insecure-configuration
docker-compose up -d
```

## Mistake 1. CRLF 注入漏洞
Nginx 会将 `$uri` 进行解码，导致传入%0d%0a 即可引入换行符，造成 CRLF 注入漏洞。
错误的配置文件示例（原本的目的是为了让 http 的请求跳转到 https 上）：

```
location / {
    return 302 https://$host$uri;
}
```

Payload: `http://your-ip:8080/%0d%0aSet-Cookie:%20a=1`，可注入 Set-Cookie 头。
![pasted image 20260827212400](/assets/img/posts/2026-08-27-nginx-misconfiguration-vulnerabilities/pasted-image-20260827212400.png)

## Mistake 2. 目录穿越漏洞

Nginx 在配置别名（Alias）的时候，如果忘记加 `/`，将造成一个目录穿越漏洞。
错误的配置文件示例（原本的目的是为了让用户访问到/home/目录下的文件）：

```
location /files {
    alias /home/;
}
```

Payload: `http://your-ip:8081/files../` ，成功穿越到根目录：
![pasted image 20260827220107](/assets/img/posts/2026-08-27-nginx-misconfiguration-vulnerabilities/pasted-image-20260827220107.png)
## Mistake 3. add_header 被覆盖

Nginx 配置文件子块（server、location、if）中的 `add_header`，将会覆盖父块中的 `add_header` 添加的 HTTP 头，造成一些安全隐患。

如下列代码，整站（父块中）添加了 CSP 头：

```
add_header Content-Security-Policy "default-src 'self'";
add_header X-Frame-Options DENY;

location = /test1 {
    rewrite ^(.*)$ /xss.html break;
}

location = /test2 {
    add_header X-Content-Type-Options nosniff;
    rewrite ^(.*)$ /xss.html break;
}
```

但 `/test2` 的 location 中又添加了 `X-Content-Type-Options` 头，导致父块中的 `add_header` 全部失效：
请求父节点路径时，响应头中会返回CSP，但是请求/test2时，CSP头会消失，如果/test1和/test2页面中存在XSS，就可能造成更大的危害
![pasted image 20260827221621](/assets/img/posts/2026-08-27-nginx-misconfiguration-vulnerabilities/pasted-image-20260827221621.png)
![pasted image 20260827222154](/assets/img/posts/2026-08-27-nginx-misconfiguration-vulnerabilities/pasted-image-20260827222154.png)
## Mistake 4.PHP-FPM 解析漏洞（cgi.fix_pathinfo）
**1.Nginx 的 PHP 转发逻辑** 典型的 Nginx PHP 配置通过后缀匹配转发请求：

```
location ~ \.php$ {
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
}
```

规则：只要请求 URI 以 `.php` 结尾，就匹配该 location，将完整 URI 作为脚本路径直接转发给 PHP-FPM。 **关键缺陷**：Nginx 只校验 URI 后缀，**不会校验这个路径对应的文件是否真实存在**。

**2.PHP 的 cgi.fix_pathinfo 机制** 这是 PHP 的 CGI 配置项，**默认值为开启（1）**。
- 设计初衷：支持 PATH_INFO 风格的 URL（如 `/index.php/user/1`，`/index.php` 是执行脚本，`/user/1` 是附加路径信息）。
- 工作逻辑：当 PHP 收到的 SCRIPT_FILENAME 对应文件不存在时，**会自动从右向左回溯路径，找到第一个真实存在的文件，将其作为 PHP 脚本执行，剩余的路径部分作为 PATH_INFO 环境变量**。

这个漏洞产生本质是 **Nginx 与 PHP 对路径的判定逻辑不一致**：
- Nginx 只通过 URI 后缀 `.php` 判断是否转发 PHP 请求，不校验文件是否真实存在；
- PHP 收到不存在的路径时，会通过 `cgi.fix_pathinfo` 向前回溯真实文件，把非 PHP 后缀的文件当作脚本执行。 两者配合，直接打破了 “文件后缀决定执行方式” 的安全边界。

**所有我们可以通过上传图片马的形式进行攻击**
1. 上传一张内嵌 PHP 代码的图片（图片马）：

```bash
# 图片马制作
cat normal.jpg shell.php > shell.jpg
# shell.jpg 在二进制上还是图片，但末尾有 PHP 代码
```

2. 访问：

```bash
curl http://target/upload/shell.jpg/x.php
# Nginx 把请求转给 PHP-FPM
# PHP-FPM 看 x.php 不存在，fix_pathinfo 回退到 shell.jpg，当 PHP 解析
# 执行 shell.php 中的 PHP 代码
```

