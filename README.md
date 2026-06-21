# 我的个人博客

使用 Hugo 搭建的个人博客，自动部署到 GitHub Pages。

## 功能特点

- 📝 支持 Markdown 写作
- 🎨 PaperMod 主题，简洁美观
- 🔍 全站搜索功能
- 📚 文章归档
- 🏷️ 标签分类
- 📄 个人简历页面
- 🚀 GitHub Actions 自动部署

## 本地开发

### 前置要求

- Hugo (v0.150.0+)
- Git

### 安装 Hugo

macOS:
```bash
brew install hugo
```

Linux/Windows:
请访问 [Hugo官网](https://gohugo.io/installation/) 查看安装说明

### 运行项目

1. 克隆仓库：
```bash
git clone --recurse-submodules https://github.com/lesshash/lesshash.git
cd lesshash
```

2. 启动本地服务器：
```bash
hugo server -D
```

3. 在浏览器中访问 `http://localhost:1313`

## 写作指南

### 创建新文章

```bash
hugo new posts/my-new-post.md
```

### 文章模板

```markdown
---
title: "文章标题"
date: 2025-01-18T10:00:00+08:00
draft: false
tags: ["标签1", "标签2"]
categories: ["分类"]
summary: "文章摘要"
---

文章内容...
```

## 部署到 GitHub Pages

### 步骤 1: 创建 GitHub 仓库

1. 在 GitHub 创建一个新仓库
2. 仓库名称可以是 `lesshash.github.io`（用户页面）或任意名称（项目页面）

### 步骤 2: 配置仓库

1. 修改 `hugo.yaml` 中的 `baseURL`：
   - 用户页面: `https://lesshash.github.io/`
   - 项目页面: `https://lesshash.github.io/repo-name/`

2. 推送代码到 GitHub：
```bash
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/lesshash/lesshash.git
git push -u origin main
```

### 步骤 3: 启用 GitHub Pages

1. 进入仓库的 Settings → Pages
2. Source 选择 "GitHub Actions"
3. 等待 Actions 运行完成
4. 访问你的网站：
   - 用户页面: `https://lesshash.github.io`
   - 项目页面: `https://lesshash.github.io/lesshash`

## 自定义配置

### 修改网站信息

编辑 `hugo.yaml` 文件：

- `title`: 网站标题
- `params.author`: 作者名称
- `params.description`: 网站描述
- `params.socialIcons`: 社交媒体链接

### 修改个人简历

编辑 `content/resume.md` 文件，更新你的个人信息。

### 更换主题

如需更换主题，请访问 [Hugo主题库](https://themes.gohugo.io/) 选择喜欢的主题。

## 文件结构

```
my-blog/
├── .github/
│   └── workflows/
│       └── hugo.yaml        # GitHub Actions 配置
├── archetypes/              # 文章模板
├── content/                 # 内容目录
│   ├── posts/              # 博客文章
│   ├── resume.md           # 简历页面
│   ├── search.md           # 搜索页面
│   └── archives.md         # 归档页面
├── themes/                  # 主题目录
│   └── PaperMod/           # PaperMod 主题
├── hugo.yaml               # Hugo 配置文件
└── README.md               # 说明文档
```

## 常用命令

```bash
# 创建新文章
hugo new posts/article-name.md

# 本地预览（包括草稿）
hugo server -D

# 构建网站
hugo

# 清理缓存
hugo --gc
```

## 故障排查

### 主题不显示

确保已正确安装主题：
```bash
git submodule update --init --recursive
```

### GitHub Actions 失败

1. 检查仓库 Settings → Pages 是否选择了 "GitHub Actions"
2. 查看 Actions 标签页的错误日志
3. 确保 `hugo.yaml` 中的配置正确

## 许可证

MIT License

## 联系方式

- Email: your-email@example.com
- GitHub: https://github.com/lesshash