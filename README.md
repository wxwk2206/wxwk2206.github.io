# wxwk2206 的安全研究笔记

> 个人技术博客，专注 **Web 安全 / 代码审计**，基于 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 主题构建，托管于 GitHub Pages。
>
> 记录从「原理 → 复现 → 修复」的安全学习过程，所有文章均来自亲手实操与代码审计。

## 内容方向
- **Web 漏洞**：XSS（反射 / 存储 / DOM）、SQL 注入、SSRF、XXE、SSTI、CSRF、IDOR、RCE、业务逻辑漏洞
- **Java 安全**：反序列化（CC 链 / ysoserial）、JNDI 注入、SpEL / OGNL 表达式注入
- **靶场实战**：Vulhub、SQLi-Labs、Upload-Labs 等等
- **基础铺垫**：网络数据包、安全核心概念、资产侦察与信息收集

## 目录结构
```
├── _config.yml   # 站点配置
├── _posts/       # 文章（Markdown，含 front matter）
├── _tabs/        # 导航页：分类 / 标签 / 归档 / 关于
├── _data/        # 数据文件（社交链接、导航等）
├── assets/       # 图片、字体等静态资源
└── tools/        # 辅助脚本
```

## 📬 交流联系

欢迎漏洞挖掘、代码审计方向的技术交流：
- **QQ**：2206382290
- 邮箱：jakupcakalarcon@gmail.com