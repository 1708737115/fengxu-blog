# 配置并更新内容

修改文档的基础信息和文档的内容介绍

## 下载文档仓库

把新创建的文档仓库clone到本地

```bash
git clone your_repo_git_url
```

## 本地编辑器安装和预览 - **可选**

**安装依赖**

执行安装vscode编辑器和mdbook的命令

```bash
xlings install
```

**预览文档**

执行命令启动浏览器, 动态显示文档内容并进行预览

```bash
xlings book
```

> **注: xlings工具安装**
>
>   - 下载 [xlings压缩包](https://github.com/d2learn/xlings/archive/refs/heads/main.zip)
>   - 解压到本地 并运行其中tools目录下的`install.win.bat`
>   - linux系统安装及更多细节见 -> [xlings](https://github.com/d2learn/xlings)

## 修改文档配置信息

按照自己的需求修改文档配置文件 `book/book.toml`

- **title:** 文档|书籍名
- **author:** 作者
- **git-repository-url：** 仓库链接

```bash
[book]
title = "Xlings Book Template | 文档 | 书籍 | 笔记"
author = "Author Name"
language = "en"

[build]
build-dir = "book"

[output.html]
git-repository-url = "https://github.com/d2learn/xlings-book-template"

[preprocessor.foo]
# Add any additional configurations
```

## 修改文档内容

- `book/src/SUMMARY.md`: 为文档目录文件
- `book/src/chapter_*.md`: 文档对应的章节

**编辑内容**

可以使用vscode或其他任何的编辑器修改`boo/src`下的内容

**修改时并预览 - 可选**

```bash
xlings book
```

然后使用git把本地内容更新到仓库即可触发在线文档自动更新并部署