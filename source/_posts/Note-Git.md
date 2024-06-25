---
title: 「学」 Git
category_bar: true
date: 2023-03-01 15:09:54
tags:
categories: 学习笔记
banner_img:
---

Git is very good!

<!--more-->

## 工作流程

* 克隆 Git 资源作为工作目录。
* 在克隆的资源上添加或修改文件。
* 如果其他人修改了，你可以更新资源。
* 在提交前查看修改。
* 提交修改。
* 在修改完成后，如果发现错误，可以撤回提交并再次修改并提交。

![工作流程](1.png)

## 基本概念

* **工作区**：就是你在电脑里能看到的目录。
* **暂存区**：英文叫 stage 或 index。一般存放在 .git 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
* **版本库**：工作区有一个隐藏目录 .git，这个不算工作区，而是 Git 的版本库。
* **远程版本库**：一般指的是 Git 服务器上所对应的仓库，如 github 仓库。

![关系](2.jpg)

## 基本操作

* `git init`：初始化仓库
* `git config`：配置开发者用户名和邮箱
* `git clone`：从 git 服务器拉取代码
* `git status`：查看文件变动状态
* `git add .`：添加文件到暂存区
* `git commit`：提交文件变动到版本库
* `git push`：将本地的代码改动推送到服务器
* `git pull`：将服务器上的最新代码拉取到本地
* `git log`：查看版本提交记录
* `git tag`：为项目标记里程碑
* `git reset`：回退版本
* `.gitignore`：设置哪些内容不需要推送到服务器，这是一个配置文件

具体参数可参考[这篇文章](https://mp.weixin.qq.com/s/Q_O0ey4C9tryPZaZeJocbA)

## 分支管理

* 分支（Branch）：分支是为了将修改记录的整个流程分开存储，让分开的分支不受其它分支的影响，所以在同一个数据库里可以同时进行多个不同的修改。
* 主分支（Master/Main）：前面提到过 master 是 Git 为我们自动创建的第一个分支，也叫主分支，其它分支开发完成后都要合并到 master。
* HEAD：指向的就是当前分支的最新提交。

![Branch](3.png)

* `git branch`：创建、重命名、查看、删除项目分支
* `git checkout`：切换分支
* `git merge`：合并分支

## Github

如果你想通过 Git 分享你的代码或者与其他开发人员合作。你就需要将数据放到一台其他开发人员能够连接的服务器上。

本例使用了 Github 作为远程仓库，可以阅读 [Github 简明教程](https://www.runoob.com/w3cnote/git-guide.html)。

![Github](4.png)

* `git fetch`：从远程仓库下载新分支与数据
* `git merge`：从远端仓库提取数据并尝试合并到当前分支

![提取](5.png)