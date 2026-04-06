---
title: '使用 Astro 构建高性能静态网站'
description: '深入了解 Astro 框架，学习如何利用 Islands 架构构建快速、现代的静态网站。'
pubDate: 2026-04-06
tags: ['Astro', '前端', '教程']
author: 'liuzhne'
draft: false
---

## 什么是 Astro？

[Astro](https://astro.build) 是一个现代化的静态网站生成器，它的核心特点是 **Islands 架构** —— 一种在静态页面中嵌入交互式组件的创新方式。

## 为什么选择 Astro？

### 1. 极致的性能

Astro 默认生成零 JavaScript 的静态 HTML，这意味着：

- 更快的首屏加载速度
- 更好的 SEO 表现
- 更低的带宽消耗

### 2. Islands 架构

只有在需要交互的地方才会加载 JavaScript，这被称为 **部分水合（Partial Hydration）**：

```astro
---
// 这段代码只在服务器端运行
const data = await fetch('https://api.example.com/data');
---

<!-- 静态 HTML，无 JavaScript -->
<h1>欢迎来到我的网站</h1>

<!-- 交互式组件，按需加载 -->
<InteractiveChart client:visible data={data} />
```

### 3. 支持多种前端框架

Astro 可以与任何主流前端框架配合使用：

- React
- Vue.js
- Svelte
- Solid.js
- Preact

## 快速开始

### 安装

```bash
# 使用 npm
npm create astro@latest

# 使用 yarn
yarn create astro

# 使用 pnpm
pnpm create astro@latest
```

### 项目结构

```
my-astro-project/
├── src/
│   ├── components/     # 可复用组件
│   ├── layouts/        # 页面布局
│   ├── pages/          # 路由页面
│   └── content/        # 内容集合（博客文章等）
├── public/             # 静态资源
├── astro.config.mjs    # 配置文件
└── package.json
```

### 创建第一个页面

在 `src/pages/index.astro` 中：

```astro
---
const title = '我的 Astro 网站';
---

<html lang="zh-CN">
  <head>
    <meta charset="UTF-8">
    <title>{title}</title>
  </head>
  <body>
    <h1>欢迎来到 {title}</h1>
  </body>
</html>
```

## 内容集合

Astro 的内容集合功能让管理 Markdown 内容变得非常简单：

```typescript
// src/content.config.ts
import { defineCollection, z } from 'astro:content';

const postsCollection = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    tags: z.array(z.string()).default([]),
  }),
});

export const collections = {
  posts: postsCollection,
};
```

## 部署

Astro 支持多种部署平台：

- **GitHub Pages** - 免费，适合个人项目
- **Vercel** - 优秀的性能和分析工具
- **Netlify** - 强大的表单处理和身份验证
- **Cloudflare Pages** - 全球 CDN 加速

## 总结

Astro 是一个非常优秀的静态网站生成器，特别适合：

- 内容密集型网站（博客、文档）
- 营销页面
- 电子商务网站

如果你正在寻找一个性能优秀、开发体验良好的静态网站解决方案，Astro 绝对值得一试！

## 相关资源

- [Astro 官方文档](https://docs.astro.build)
- [Astro 教程](https://docs.astro.build/en/tutorial/0-introduction/)
- [Astro 集成](https://astro.build/integrations/)
