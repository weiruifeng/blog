# 博客

基于 Jekyll 搭建的个人博客，部署在 GitHub Pages。

## 环境要求

系统自带的 Ruby 版本过低，需要使用 Homebrew 安装的 Ruby 4.x。

```bash
# 检查 Homebrew Ruby 是否已安装
ls /usr/local/Cellar/ruby/

# 如果未安装，执行
brew install ruby
```

## 本地开发

每次启动前需要将 Homebrew Ruby 加入 PATH：

```bash
export PATH="/usr/local/Cellar/ruby/4.0.1/bin:$PATH"
```

> 提示：可以将上面这行加入 `~/.zshrc` 或 `~/.bash_profile`，避免每次手动执行。

### 安装依赖

```bash
bundle install
```

### 启动本地服务

```bash
bundle exec jekyll serve
```

访问 http://localhost:4000/blog/

### 编译（不启动服务）

```bash
bundle exec jekyll build --watch
```

## 项目结构

```
├── _config.yml        # 站点配置（作者、标题、motto 等）
├── _data/             # 导航菜单数据
├── _includes/         # 公共组件（header、script）
├── _layouts/          # 页面模板
│   ├── default.html   # HTML 基础框架
│   ├── structure.html # 左侧栏 + 内容区布局
│   ├── post.html      # 文章页
│   └── category.html  # 分类页
├── _posts/            # 博客文章（Markdown）
├── _sass/             # 样式源文件
│   ├── main.scss      # 主布局样式
│   ├── header.scss    # 导航侧栏样式
│   └── public.scss    # 全局基础样式
├── assets/            # 静态资源（图片、图标、CSS）
├── index.html         # 首页
├── about.md           # 关于页
├── category.html      # 分类汇总页
└── Gemfile            # Ruby 依赖
```

## 写文章

在 `_posts/` 目录下新建 Markdown 文件，命名格式：

```
YYYY-MM-DD-文章标题.md
```

文件头部加入 Front Matter：

```yaml
---
layout: post
title: "文章标题"
date: 2024-01-01
categories: 分类名
tags: [标签1, 标签2]
---
```

## 移动端适配

≤ 768px 屏幕下的表现：

- **首页**：全屏 Hero 模式，保持不变
- **其他页面**：左侧导航栏折叠为顶部 Header，右侧显示汉堡菜单按钮，点击展开导航

## 部署

推送到 `gh-pages` 分支后 GitHub Pages 自动部署。

```bash
git add .
git commit -m "feat: ..."
git push origin gh-pages
```
