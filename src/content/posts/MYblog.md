---
title: 使用Firefly模板搭建个人网站
published: 2026-08-29
description: '核心需求：制作一个个人的网站博客,使用Firefly网站博客模板部署'

tags: [Markdown, Project]
category: 项目
draft: true
---

# 一、环境配置
需要安装Node.js、Pnpm、Git、Github Desktop、VS Code
## 1.下载 Node.js 
下载地址：https://nodejs.org/zh-cn/download
## 2.安装 Pnpm
在powershell中执行 `npm install -g pnpm`
## 3.安装 Git
下载地址：https://git-scm.cn/install/windows
## 4.安装 Github Desktop
下载地址：https://desktop.github.com/download/
## 5.下载 VS Code
下载地址：https://code.visualstudio.com/Download?_exp_download=fb315fc982

# 二、Firefly库clone
## 1.搜索Firefly Github库
链接：https://github.com/CuteLeaf/Firefly
## 2.fork新建库
## 3.本地创建文件夹-在文件夹中右键然后点击 Open Git Bash here
## 4.clone库

```Bash
-git clone [新建库链接]
```
(注：如下载失败可能是因为网络超时，可在Git Bash中配置w网络代理 ）
```Bash
-git config --global http.proxy [代理地址]
```
## 6.网站预览 
打开本地服务器，获得localhost。
```Bash
-pnpm dev 
```
在浏览器中输入localhost，打开博客网站进行效果预览。

# 三、使用VSCode进行配置
## 1.使用VSCode打开项目目录
## 2.找到配置文件
文件路径：`src/config/README.md`
## 3.自定义配置文档
文档地址：https://docs-firefly.cuteleaf.cn/zh/guide/site.html
## 4.文章编写文档
文档地址：https://docs-firefly.cuteleaf.cn/zh/guide/writing.html