---
title: "网络数据包基础：从报文结构理解 Web 通信"
date: 2025-01-12 12:00:00 +0800
categories: [安全基础, 网络]
tags: [数据包, 网络, TCP, UDP]
excerpt: "网络传输时，数据被切分成一小块一小块的数据包。从报头与载荷的结构讲起，理清 TCP / UDP / IP 数据包各自的作用与适用场景。"
---

## 数据包（Packet）核心概念
## 1. 什么是数据包？
+ 网络传输时，**数据被切分成一小块一小块**，每一块就是**数据包**。

## 2. 数据包的作用
+ 让大数据能**稳定、高效、可靠**地在网络里传输
+ 出问题只重传出错的那一小段，不用重传整个文件
+ 支持多路复用（多应用同时上网）

## 3. 数据包基本结构
每个数据包都由两部分组成：

1. **报头（Header）**
    - 来源 IP、目标 IP
    - 协议类型（TCP/UDP/ICMP 等）
    - 端口号、序号、校验和等→ 相当于**快递单**
2. **数据部分（Payload / Data）**
    - 真正要传输的内容（网页、图片、消息等）→ 相当于**快递里的东西**

## 4. 常见协议与数据包
### TCP 数据包
+ 可靠传输、有连接、保证顺序、不丢包
+ 用于：网页、文件、邮件、登录等

### UDP 数据包
+ 不可靠、无连接、速度快
+ 用于：直播、游戏、DNS、视频通话

### IP 数据包
+ 负责**寻址和路由**，让数据包从一台电脑送到另一台
+ 是网络层的核心

### HTTP/HTTPS 数据包
+ 应用层数据，封装在 TCP 里
+ 我们看到的网页内容、接口请求都在这里

## 5. 关键配套概念
### 帧（Frame）
+ 数据在**链路层**的单位（最底层）
+ 数据包 → 封装成帧 → 物理网线上传输

### 分片与重组
+ 数据包太大时会被**分片**，到达目标后再**重组**

### 端口
+ 一台电脑里区分不同应用（QQ、浏览器、游戏各自用不同端口）

### 五元组（唯一标识一条连接）
`源IP + 目标IP + 源端口 + 目标端口 + 协议`

## 6. 数据包和安全的关系（你可能会用到）
+ 抓包工具（Wireshark、tcpdump）抓的就是**数据包**
+ 流量分析、漏洞复现、SQL 注入、XSS、上传漏洞，都能在数据包里看到明文
+ HTTPS 会加密数据包内容，无法直接查看

## Request请求数据包
![pasted image 20260702203047](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203047.png)

## 1.请求行
```
#请求行 
请求行由三个标记组成：请求方法、请求URL和HTTP版本，它们用空格分隔。 
例如：GET /index.html HTTP/1.1

HTTP：规范定义了8种可能的请求方法 
GET：检索URL中标识资源的一个简单请求 
HEAD：与GET方法相同，服务器只返回状态行和头标，并不返回请求文档 
POST：服务器接受被写入客户端输出流中的数据的请求 
PUT：服务器保存请求数据作为指定URL新内容的请求 
DELETE：服务器删除URL中命令的资源的请求 
OPTIONS：关于服务器支持的请求方法信息的请求 
TRACE：web服务器反馈Http请求和其头标的请求 
CONNECT ：已定义，但当前未实现的一个方法，预留做隧道处理
```
最常用：get和post

## 2.请求头
![pasted image 20260702203058](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203058.png)

![pasted image 20260702203106](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203106.png)

![pasted image 20260702203112](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203112.png)

## 3.空行
![pasted image 20260702203120](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203120.png)

**get 请求也必须有空行！！！**

## 4.请求数据（请求体）
![pasted image 20260702203142](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203142.png)

## Response返回数据包
![pasted image 20260702203148](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203148.png)

![pasted image 20260702203153](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203153.png)

![pasted image 20260702203203](/assets/img/posts/2025-01-12-network-packets-basics/pasted-image-20260702203203.png)


