# duanyzhi.github.io

个人博客，线上地址：<https://duanyzhi.github.io/>

平时记点 LLM 推理、算子优化和计算机视觉的东西，踩过的坑也顺手写下来。

## 技术栈

用 [Astro](https://astro.build/) 搭的，主题基于
[AstroPaper](https://github.com/satnaing/astro-paper)，样式走 Tailwind，
站内搜索用 Pagefind，托管在 GitHub Pages。

## 本地跑

需要 Node >= 22.12。

```shell
npm install
npm run dev       # 本地预览，默认 http://localhost:4321
npm run build     # 构建到 dist/，顺带生成搜索索引
npm run preview   # 预览构建好的结果
```

## 发新文章

在 `src/content/posts/` 下新建一个 `.md`，开头写上这几项：

```yaml
---
title: 文章标题
description: 一句话摘要，会出现在列表页和 SEO 里
pubDatetime: 2026-09-30T00:00:00Z
tags: [shell, linux]
---
```

正文就是普通 markdown。想让文章置顶或先不公开，加 `featured: true`
或 `draft: true`。

## 改哪儿

- 站点信息（标题、作者、社交链接）：`astro-paper.config.ts`
- 关于页：`src/content/pages/about.md`
- 首页介绍：`src/pages/index.astro`
- 样式：`src/styles/`

## 部署

推到 `master` 分支就会自动跑 GitHub Actions，构建并发布到 GitHub Pages，
不用手动操作。
