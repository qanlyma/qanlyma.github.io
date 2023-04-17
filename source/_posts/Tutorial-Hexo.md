---
title: 「学」 Hexo 基础命令
category_bar: true
date: 2022-11-07 22:21:58
tags:
categories: 学习笔记
---

Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

<!-- more -->

## Quick Start

### 创建新网站

``` bash
$ hexo init
```

### 创建新文章

``` bash
$ hexo new "title"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### 创建新页面

``` bash
$ hexo new page "title"
```

### 清除缓存文件

``` bash
$ hexo clean
```

### 在本地启动 hexo

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### 生成静态文件

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### 部署到 Github

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

### 更换电脑

hexo 目录下的文件和 github 上的文件是不同的，public 文件夹的文件通过 `hexo d` 上传到 github，其他的文件则留在本地目录下。

1. 将本地文件传入 github 新建分支 `hexo`，并设为默认。
2. 在新电脑上克隆新分支到本地，切换到 `username.github.io` 目录，执行 `npm install` 安装依赖（node_modules文件）。
3. 安装 hexo： `npm install -g hexo-cli`，安装必要的插件，例如需要部署到 gitPage： `npm install hexo-deployer-git --save`。
4. 更改后依次执行 `git add .`、`git commit -m "..."`、`git push`。
5. 更新前使用 `git pull`。

**.md 文件建议使用 UTF-8，其他格式可能会乱码。**
