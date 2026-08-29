# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这是什么

一个托管在 GitHub Pages 上的个人技术博客（站点名"技术笔记"，文章以中文为主），使用 Jekyll 构建，主题为 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)，以 gem 方式引入（`gem "jekyll-theme-chirpy"`），**主题源码不在本仓库中**，位于 `vendor/bundle/.../jekyll-theme-chirpy-*`。不要修改那里的文件——那是 bundle 安装的依赖，不会进版本库。站点级定制通过仓库根目录的文件（`_config.yml`、`_data/`、`_tabs/`、`_posts/`、`assets/`）完成。

**部署方式**：GitHub Actions 构建部署（见 `.github/workflows/pages-deploy.yml`），push 到 `master` 后自动执行。注意：Chirpy 依赖 `jekyll-archives`（不在 GitHub Pages 白名单），因此不能用原生的 "branch build"，必须用 Actions。仓库 Settings → Pages → Source 应为 "GitHub Actions"。

## 环境要求

- Ruby（CI 用 3.1；本地需 ≥ 2.6，当前 6.5.x 支持 2.6–3.x；7.x 需要 Ruby ≥ 3.1）
- 本地依赖装在 `vendor/bundle`（已被 .gitignore 忽略，与 `_site/` 一样不入库）

## 常用命令

```sh
bundle install                              # 安装依赖
bundle exec jekyll serve --host 0.0.0.0 --port 8080   # 本地预览，http://localhost:8080
bundle exec jekyll build                    # 生产构建，输出到 _site/
```

修改 `_config.yml` 后必须重启 serve（不会热加载）。博客内容没有自动化测试。**如果本地没有可用的 Jekyll 环境，则不需要本地构建验证博文**（push 后由 GitHub Actions 构建部署），写完文章检查 front matter 和文件名格式即可。

## 写文章

- 文章放在 `_posts/`（允许子目录，如 `_posts/python/`），文件名 `年-月-日-标题.md`。
- front matter 至少包含 `title`；可选 `tags: [a, b]`、`categories: [c]`、`last_modified_at:`（Chirpy 用这个显示更新日期；统一用秒级格式 `YYYY-MM-DD HH:MM:SS +0800`，取文章最后一次 git 提交的时间）。
- 文章页自动生成摘要（从正文截取前 ~200 字）；`<!--more-->` 在 Chirpy 下不生效但保留无害。
- layout/toc/comments 等由 `_config.yml` 的 defaults 统一设置，文章里不用重复写。
- 标签/分类页由 `jekyll-archives` 自动生成，无需手工建页面。
- 站点 UI 语言为 `zh-CN`，文案在 gem 的 `_data/locales/zh-CN.yml`（只读，主题升级可能覆盖）。

## 配置

站点配置在 `_config.yml`。关键项：`theme_mode: dark`（固定暗色；留空则跟随系统并显示切换按钮）、`lang: zh-CN`、`timezone: Asia/Shanghai`、`permalink` 保持 `/ :year/:month/:day/:title.html` 以兼容旧链接。侧栏导航由 `_tabs/` 下的文件驱动（分类、标签、归档、关于），社交链接在 `_data/contact.yml`，文章底部分享在 `_data/share.yml`。

头像目前为空（侧栏只显示站点名）。若要加头像，把图片放 `assets/img/avatar/` 并在 `_config.yml` 设置 `avatar: /assets/img/avatar/xxx.jpg`。

## 升级主题

改 `Gemfile` 里 `jekyll-theme-chirpy` 的版本约束，`bundle update jekyll-theme-chirpy`，本地验证后提交 `Gemfile.lock`（注：当前 `.gitignore` 忽略了 `Gemfile.lock`；升级大版本时建议临时取消忽略并提交，以固定 CI 版本）。大版本升级前查 Chirpy 的 wiki/UPGRADE 指南。
