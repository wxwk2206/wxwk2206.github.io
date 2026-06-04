# wxwk2206.github.io

基于 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 主题的个人技术博客，托管于 GitHub Pages。

## 特性

- 🎨 暗色 / 亮色自动切换 + 手动切换
- 🔍 全文搜索
- 📝 Markdown 写作
- 📊 代码语法高亮
- 📂 分类 & 标签归档
- 📱 PWA 支持（可安装为桌面应用）
- 📈 SEO 优化

## 目录结构

```
├── _config.yml     # 站点配置
├── _posts/         # 文章 (Markdown)
├── _tabs/          # 导航标签页
├── _data/          # 数据文件（联系信息、语言文件等）
├── assets/         # 静态资源
└── tools/          # 辅助脚本
```

## 写新文章

在 `_posts/` 目录下创建 `YYYY-MM-DD-标题.md`，内容格式：

```yaml
---
title: 文章标题
date: 2026-06-04 12:00:00 +0800
categories: [分类, 子分类]
tags: [标签1, 标签2]
---

文章内容（Markdown）...
```

## 本地预览（可选）

```bash
bundle install
bundle exec jekyll serve
```

推送后 GitHub Pages 会自动构建部署。
