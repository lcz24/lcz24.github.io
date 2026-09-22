---
title: "Hello World"
date: 2026-09-22
draft: false
tags: ["Hugo", "PaperMod"]
categories: ["随笔"]
summary: "站点从 Hexo 迁到了 Hugo + PaperMod,顺手记一下以后怎么发新文章。"
---

## 为什么重建

原来的博客是 Hexo,但仓库里只剩构建产物,源码丢了,而且主题多年没维护、
关键资源还挂在 `unpkg.com/...@latest` 上,随时可能崩。所以干脆推倒重建。

现在这一版:

- **[Hugo](https://gohugo.io/)** —— 单二进制、构建毫秒级,不依赖 Node
- **[PaperMod](https://github.com/adityatelange/hugo-PaperMod)** —— 活跃维护的主题
- **GitHub Actions** —— 推代码自动构建发布,本地什么都不用装

## 怎么写一篇新文章

在 `content/posts/` 下新建一个 `.md` 文件即可:

```markdown
---
title: "文章标题"
date: 2026-09-22
draft: false
tags: ["标签一", "标签二"]
categories: ["分类"]
summary: "列表页显示的摘要。"
---

正文写这里。
```

保存并提交后,GitHub Actions 会自动构建并把站点发布出去。

> 也可以直接在 GitHub 网页上新建文件,改完点 Commit changes 就行。

## 本地预览(可选)

```bash
hugo server -D
```

然后打开 <http://localhost:1313>。

## 代码高亮

```bash
nmap -sV -Pn -p- 10.0.0.1
```

这行开始,慢慢写点东西吧。
