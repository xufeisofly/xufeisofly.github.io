# Saliva Group 技术博客

个人中文技术博客，记录区块链、共识协议及工程实现。基于 Jekyll 和 [WhatATheme](https://github.com/thedevslot/WhatATheme)。

- [正式站点](https://www.xufeisofly.xyz/) · [文章列表](https://www.xufeisofly.xyz/blog)
- [PicX 图片管理](https://picx-ih.xpoet.cn/#/management) · [图片仓库](https://github.com/xufeisofly/picx-images-hosting)
- [Agent 编辑约定](AGENTS.md)

## 项目结构

| 路径 | 用途 |
| --- | --- |
| `_posts/YYYY-MM-DD-slug.md` | 文章和 front matter |
| `assets/images/<slug>-img/` | 文章最终源图，包括头图 |
| `_layouts/post-with-content.html` | 头图、自动目录、正文及评论模板 |
| `_includes/toc.html` | 从正文标题生成目录 |
| `_includes/latex.html` | MathJax 公式支持 |
| `_includes/clickable-img.html` | 图片点击放大 |
| `_config.yml` | 站点及构建配置 |
| `CNAME` | 自定义域名 |
| `_site/` | 构建产物，不提交 |

## 本地运行

需要 Ruby 和 Bundler，依赖版本以 `Gemfile.lock` 为准。此项目已在 Ruby 3.4.7 上完成本地构建。

```sh
bundle check
# 如果缺少依赖，再执行：
bundle install
bundle exec jekyll serve --host 127.0.0.1 --port 4000
```

打开 [本地首页](http://127.0.0.1:4000/) 或 `http://127.0.0.1:4000/blog/<slug>`。Jekyll 监听文件自动重新构建，浏览器需刷新。若 4000 端口已有本仓库服务，优先复用，不直接终止不明进程。修改 `_config.yml` 后重启服务。

## 发布文章流程

### 1. 整理原稿

确认原稿位置、文章名和允许的修改范围，先备份原稿。用户说的 `~/Download` 可能实际是 macOS 的 `~/Downloads`，先查找文件。

默认保留作者表达和逻辑。统一术语、修正明显错字、补全提供的引用，技术疑点单独记录。公式使用 `$N = 3f + 1$` 等 LaTeX 写法。正文主章节使用 `##`，尽量避免三级及更深标题；定义、规则等小项用正文列表。

### 2. 准备并上传图片

1. 收集原稿图片，最终源图保存到 `assets/images/<slug>-img/`。
2. 参考既有文章制作头图，文字与最终文章名一致。持续更新的文章不自动增加篇号。
3. 在浏览器打开已登录的 PicX，核对仓库 `xufeisofly/picx-images-hosting`，为文章选择或新建独立目录。
4. 选择图片、点击上传，等待全部成功。PicX 可能转为 WebP 并给文件名添加哈希。
5. 从 UI 复制实际链接，替换原稿地址，补上清楚的 alt 文本。不要点击无关的图床部署或授权功能。
6. 在浏览器确认远程图片可以加载。

旧文章使用过 GitHub Pages 图片地址；当前 PicX 也可能返回如下链接，以实际返回并成功验证的地址为准：

```text
https://github.com/xufeisofly/picx-images-hosting/raw/master/<directory>/<filename>.webp
```

头图地址放入 `post-image`，正文使用 `![说明](实际图片链接)`。修订图片后重新上传、替换链接，避免继续使用旧缓存。

### 3. 加入 Jekyll

创建 `_posts/YYYY-MM-DD-slug.md`，front matter 示例：

```yaml
---
title: Substrate 接入 Fast-HotStuff 开发记录
layout: post-with-content
post-image: "实际上传得到的头图 URL"
tags:
- Substrate
- Fast-HotStuff
- consensus
- blockchain
---
```

之后接正文，模板已经显示文章标题，不重复写正文标题。当前 permalink 为 `blog/:title`，其中 `:title` 默认来自文件名的 slug。例如 `_posts/2026-10-13-substrate-fast-hotstuff-p1.md` 的 URL 是 `/blog/substrate-fast-hotstuff-p1`。

更新同一文章时保持文件日期和 slug，避免破坏已有链接。如果用户仍需 Downloads 中的 Markdown，同步修改整理稿，保留原稿备份。是否写“待续”及其措辞按作者要求处理。

### 4. 本地验证

```sh
bundle exec jekyll build
git diff --check
```

在本地浏览器检查：

- 标题、头图、正文与原稿和最新修改要求一致。
- 目录仅含预期章节，锚点跳转正确。
- 图片全部加载，修改过的图中文字正确，点击可放大。
- LaTeX 正常渲染，没有裸露定界符或 MathJax 错误。
- 引用、代码块、表格、结尾正确，文章出现在首页或博客列表。

区分 Ruby、Sass 弃用警告与构建错误；不为发布文章擅自升级整套依赖。

### 5. 合并到 main

用户要求发布后执行。检查脏文件、fetch 远端，显式暂存本次文章、最终图片和必要文档，保留无关修改。

```sh
git status --short
git fetch origin main
git rev-list --left-right --count main...origin/main
git switch -c codex/publish-article
# 根据本次实际路径逐个 git add，不使用 git add .
git diff --cached --stat
git diff --cached --check
git commit -m "docs: publish article"
git switch main
git merge --ff-only codex/publish-article
git push origin main
```

此示例假设本地 `main` 与 `origin/main` 一致。如果远端前进，先整合提交、处理冲突并重新检查，再合并发布；禁止强推或丢弃现有脏文件。按文章选择分支名，不覆盖已有分支。

### 6. 确认部署

2026-10-13 核验：GitHub Pages 为 `legacy` 分支构建，来源是 `main` 根目录 `/`，域名为 `www.xufeisofly.xyz`，启用 HTTPS。配置可能变化，每次发布重新查询：

```sh
gh api repos/xufeisofly/xufeisofly.github.io/pages \
  --jq '{status,html_url,build_type,source,cname,https_enforced}'
gh api repos/xufeisofly/xufeisofly.github.io/pages/builds/latest \
  --jq '{status,commit,error,created_at,updated_at}'
git rev-parse HEAD
```

push 到 `main` 会触发部署。等待构建的 `commit` 与发布 SHA 一致，且 `status` 为 `built`。若已改为 Actions 部署，则检查对应提交的 workflow。失败时查看 `error` 或构建日志，修复后再发布。

最后打开 `https://www.xufeisofly.xyz/blog/<slug>`，核对新标题、正文和图片。仅 HTTP 200 不代表内容最新，需要验证本次修改。完成后报告正式链接、提交 SHA 和部署结果。如果用户要求关闭本地预览，部署验证成功后停止本次启动的 Jekyll 服务，并确认端口不再监听；不要误停其他进程。

## 主题与许可

主题来自 [WhatATheme / TheDevsLot](https://github.com/thedevslot/WhatATheme)，原作者包括 [Sneha Omer](https://sassyecoder.github.io/) 和 [Harsh Trivedi](https://harsh98trivedi.github.io/)。保留原项目的 [GPL v2 LICENSE](LICENSE) 和页脚署名。
