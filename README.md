# 📚 LeisureLinux Readings

> 读书笔记、书评与阅读思考 —— 我的读书笔记都发到这里。

[![GitHub Pages](https://img.shields.io/badge/docs-GitHub%20Pages-059669)](https://read.freelamp.com)
[![License](https://img.shields.io/badge/license-CC%20BY--SA%204.0-blue)](https://creativecommons.org/licenses/by-sa/4.0/)
[![RSS](https://img.shields.io/badge/RSS-feed-FF6600)](https://read.freelamp.com/rss.xml)

**LeisureLinux Readings** 是独立于主博客 [freelamp.com](https://freelamp.com) 的读书笔记站点。题材横跨技术、管理与人文，每篇笔记都强调 **「读了之后能带走什么」**，而不是停留在摘抄金句。

访问地址：**[https://read.freelamp.com](https://read.freelamp.com)**

---

## 📖 最新笔记

| 日期 | 标题 | 领域 |
|------|------|------|
| 2026-08-19 | [专家警告的中国 AI 模型来了——GLM 5.3 与开源模型的双刃剑](https://read.freelamp.com/articles/2026-08-19_zai-glm53-open-weight-cyber/) | AI · 网络安全 · 译文 |
| 2026-08-19 | [职业生涯第一年，七个新的成功法则——Forbes 职场指南](https://read.freelamp.com/articles/2026-08-19_seven-rules-career-first-year/) | 职场 · 译文 |
| 2026-08-05 | [我的读书笔记怎么写——Readings 笔记模板](https://read.freelamp.com/articles/2026-08-05_reading-note-template/) | 阅读方法 · 方法论 |

👉 [**查看全部笔记 →**](https://read.freelamp.com)

---

## 🏗️ 仓库结构

```
readings/
├── articles/                    # Markdown 源文件（唯一事实源）
│   └── YYYY-MM-DD_slug/
│       ├── article.md           # 笔记正文
│       └── metadata.yaml        # 元数据（标题、标签、SEO 描述）
├── scripts/                     # 辅助脚本（如提交 sitemap 到 GSC）
├── build.py                     # 静态站点构建脚本（Python）
├── llms.txt                     # LLM 友好的站点索引
└── .github/workflows/deploy.yml # GitHub Actions 部署到 Pages
```

## ✍️ 如何发布一篇笔记

1. 在 `articles/` 下新建 `YYYY-MM-DD_slug/` 目录；
2. 编写 `article.md`（正文）与 `metadata.yaml`（标题/日期/标签/summary/description）；
3. `git add . && git commit -m "..." && git push`；
4. GitHub Actions 自动执行 `build.py` 并部署到 `read.freelamp.com`。

## 🛠️ 本地构建

```bash
python build.py   # 生成 docs/ 静态站点，可直接本地预览
```

## 📜 协议

本文档及站点内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 协议开源。

---

作者：LeisureLinux（Albert Xu） · 主博客 [freelamp.com](https://freelamp.com)
