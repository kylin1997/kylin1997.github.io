# 老张的博客（Hexo + Butterfly）

Hexo 8 静态博客，主题为 [Butterfly](https://butterfly.js.org/) `5.5.1`，通过 GitHub Actions 部署到 GitHub Pages。

## 目录约定

- `_config.yml`：Hexo 站点配置（标题、作者、URL、语言等）
- `_config.butterfly.yml`：Butterfly 主题配置（菜单、侧边栏卡片、代码高亮等）
- `source/_posts/`：文章（Markdown + front-matter）
- `source/assets/`：文章引用的图片（正文用 `/assets/xxx.png` 绝对路径）
- `.github/workflows/deploy.yml`：push 到 `main` 后自动构建并发布到 `gh-pages`
- `docs/butterfly/`：Butterfly 官方文档离线备份（见下方链接）
- `docs/category-taxonomy.md`：**文章分类总表（唯一来源）**，新文章只从这里选分类

## 文章分类

分类写在每篇文章 front-matter 的 `categories` 字段，聚合结果由 Hexo 自动生成（无独立配置文件）；
分类索引页是 `source/categories/index.md`（`type: categories`）。

- 可选分类清单：[文章分类总表](./docs/category-taxonomy.md)
- 新文章的分类必须从该表里选；需要新分类时先在该表登记，再写进文章。
- 层级用**数组嵌套**表示（`/` 写法会被 Hexo 扁平化，别用）：

  ```yaml
  categories:
    - [harness, 01-architecture]
  ```

## Butterfly 主题参考文档

需要查主题配置项、Front-matter、标签页/分类页/友链页/图库等用法时，优先看本地备份（无需联网）：

- [主题页面（Front-matter / 标签页 / 分类页 / 友链 / 图库 / 404 / 说说）](./docs/butterfly/theme-pages.md) —— 原文档：<https://butterfly.js.org/posts/dc584b87/>
- [主题配置（导航 / 代码块 / 头像 / 横幅 / 侧边栏 / 搜索 / 评论等全部配置项）](./docs/butterfly/theme-config.md) —— 原文档：<https://butterfly.js.org/posts/4aa8abbe/>

## 常用命令

```bash
hexo clean && hexo generate   # 清理并构建到 public/
hexo server -p 4000           # 本地预览 http://localhost:4000
hexo new "文章标题"            # 新建文章到 source/_posts/
```

## 约定

- 不要直接修改 `node_modules/` 里的主题源码；样式/配置改动走 `_config.butterfly.yml` 或 `source/`。
- 文章图片放在 `source/assets/`，正文用 `/assets/<文件名>` 引用。
- 不接入需要付费/注册审核的服务（评论、统计等）。
