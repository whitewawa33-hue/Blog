---
title: 搭建部署个人图床 CloudFlare ImgBed
published: 2026-09-19
description: '使用CloudFlare Imgbed 项目，免费搭建个人图床'
tags: [Cloundflare, project]
category: 项目
---
>前述：搭建图床可以为网站图片提供云端存储和引用，以提升网页的访问速度。  
# 一、采用 Cloudflare Workers 部署方式
## （1）Fork 项目
访问项目地址：https://github.com/MarSeventh/CloudFlare-ImgBed
## （2）准备 Cloudflare 资源
### 1.获取 API Token 和 Account ID
-登录 Cloudflare Dashboard
-点击右上角头像 → "我的个人资料" → "API 令牌"
-点击 "创建令牌"
-选择 "编辑 Cloudflare Workers" 模板
-确认权限并创建，记录生成的 Token
-返回 Dashboard 首页，在右侧栏找到并记录 Account ID
### 2.创建KV数据库
-在 Dashboard 中选择 "存储和数据库" → "Workers KV"
-点击 "创建实例"，名称填 img_url
-创建完成后，记录命名空间 ID
## （3）配置 GitHub Secrets
在 Fork 的仓库中，进入 Settings → Secrets and variables → Actions → Secrets，添加以下 Secrets：`CLOUDFLARE_API_TOKEN` `CLOUDFLARE_ACCOUNT_ID	` `KV_NAMESPACE_ID`
## （4）运行部署
-进入 Fork 仓库的 Actions 页面
-在左侧选择 Deploy to Cloudflare Workers
-点击 Run workflow
-选择要部署的分支（默认 main）
-可选修改 Worker 名称（优先级：手动输入 > Secrets 中的 WORKER_NAME > cloudflare-imgbed）
-点击 Run workflow 开始部署
-部署完成后，可以通过 https://<worker-name>.<account-subdomain>.workers.dev 访问。
## （5）存储渠道采用完全免费的 Telegram Bot
### 1.获取 TG_BOT_TOKEN
-在 Telegram 中搜索 @BotFather
-发送 /newbot 命令
-按提示输入 Bot 名称和用户名
-获得 Bot Token（格式：123456789:ABCdefGHIjklMNOpqrsTUVwxyz）
### 2.获取 TG_CHAT_ID
-创建一个新的 Telegram 频道（Channel）
-将创建的 Bot 添加为频道管理员
![Snipaste_2026-09-20_11-59-54.png](https://imgbed.whitewawa33.workers.dev/file/1789876856438_Snipaste_2026-09-20_11-59-54.png)
-给予 Bot 消息管理的权限
-在频道中发送一条测试消息
-向 @VersaToolsBot 转发这条消息
-获得频道 ID（示例：-1001234567890）