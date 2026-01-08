---
tag: tool jekyll
---

<!--more-->

# github pages
github pages是github提供的一个免费服务，允许从githbu仓库发布静态网站。

# jekyll
jekyll是一个静态网站生成器；
jekyll使用ruby语言实现，在ruby语言中，软件包成为gem；bundler是一个gem，它允许使用Gemfile管理gem，它的命令名是bundle；

## 概要

### 配置文件
_config.yml: 使用[yaml格式](https://www.ruanyifeng.com/blog/2016/07/yaml.html)。

### 实现机制
* liquid模板语言
    支持引用配置，也支持条件判断和循环等编程元素，使支持一定的动态生成。

* yaml前置元数据(每个内容文件开头都需要有这部分)

* 最终生成html文件。

### 主题
主题主要是实现了一些layout文件，并定义了一些配置项。使得使用者仅需要关心_config.yml,_data目录，并专心在_posts目录中创作内容就可以了。

主题分为jekyll自带的主题（ruby里的gem），以及remote theme（github中的仓库）。

jekyll自带的主题，在_config.yml中使用theme配置项指定。
remote theme，在_config.yml中使用remote_theme配置项指定。

### 内容创作
它允许使用markdown编写博客内容。
