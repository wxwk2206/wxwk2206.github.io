# 字体说明

## 当前策略
本目录使用 `local()` 优先 + 系统字体 fallback 策略（见 `_sass/_custom.scss`）。

- 用户系统已装 Geist / JetBrains Mono → 直接使用
- 未安装 → 降级到 system-ui / Cascadia Code / Consolas（无 404）

## 如需完美还原 design-preview 视觉
下载以下 woff2 文件到本目录，并取消 `_sass/_custom.scss` 中 `@font-face` 的 `src` 注释：

### Geist（正文）
- 来源：https://github.com/vercel/geist-font/releases
- 或：https://fonts.google.com/specimen/Geist
- 文件：`Geist-Variable.woff2`（含 400-700 全字重，约 180KB）

### JetBrains Mono（代码）
- 来源：https://github.com/JetBrains/JetBrainsMono/releases
- 或：https://fonts.google.com/specimen/JetBrains+Mono
- 文件：`JetBrainsMono-Variable.woff2`（含 400-600 全字重，约 120KB）

## 为什么不直接 CDN？
访谈决策 D1 要求"100% 可用 + 零 CDN 依赖"。Google Fonts CDN 在国内访问不稳定，
本地化是最稳方案。当前因沙箱网络限制未能自动下载，故先用 fallback 策略。
