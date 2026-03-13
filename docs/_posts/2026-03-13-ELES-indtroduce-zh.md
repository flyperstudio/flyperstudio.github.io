---
layout: post
title:  "ELES 介绍"
date:   2026-03-13 09:18:16 +1100
categories: tech blog
---

# ELES

## 产品介绍

ELES 是一个面向教学辅导场景的考试题目生成系统。

简单来说，ELES 可以：
- 复用已有考试题目
- 生成风格与难度一致的新题
- 帮助团队更快准备更大规模的练习与评估题库

目标是在保持题目质量与格式一致的同时，节省老师和内容团队的时间。

ELES 的一个核心优势是数据安全与部署简单。ELES 以本地优先方式运行，常规使用下文件保留在你的本地设备中。在部分 AI 工作流中，文件需要通过 URL 访问，因此我们的后端会在处理期间临时保存上传文件。AI 处理完成后，这些文件会被删除。这样可以避免额外存储成本，并尽量减少数据留存；后端仍主要用于提供模型访问能力和账单用量统计。

## 安装与启动（Docker）

### 1. 先安装 Docker

请从 Docker 官方网站（https://www.docker.com/）安装 Docker Desktop，并确认 Docker Desktop 已经启动。

### 2. 从 GitHub Container Registry 拉取镜像

ELES 生产镜像托管在 GitHub Container Registry。

在 Docker 终端中拉取最新镜像：

```bash
docker pull ghcr.io/flyperstudio/eightlegsessay-client:latest
```

<img src="https://www.flyperstudio.com/images/ELES/docker-open-terminal.png" width="200" />

镜像下载完成后，你可以在 Docker Desktop 中查看。

<img src="https://www.flyperstudio.com/images/ELES/docker-downloaded-image.png" width="200" />

### 3. 用 Docker Desktop 界面启动容器（无需命令行）

这一步为所有用户设计，包括非开发人员。

1. 打开 Docker Desktop，进入 Images 页面。
2. 找到 ghcr.io/flyperstudio/eightlegsessay-client:latest，点击 Run。
3. 在运行配置里设置端口映射：
   Host port: 8080
   Container port: 8080
4. 配置卷映射（bind mount）：
   Host path: 点击文件夹选择器，从你的电脑中选择一个文件夹（例如 Documents 下的某个文件夹）。
   Container path: /data/blob_storage
5. 启动容器。

卷映射会把 ELES 生成的中间文件保存到你本地选择的文件夹中，之后可以随时查看和复用。

<img src="https://www.flyperstudio.com/images/ELES/docker-config-port.png" width="200" />

### 4. 确认容器正在运行

你可以在 Docker Desktop 中确认镜像已经成功启动。

<img src="https://www.flyperstudio.com/images/ELES/docker-image-launch.png" width="200" />

<img src="https://www.flyperstudio.com/images/ELES/docker-launch-image.png" width="200" />

### 5. 在浏览器中打开 ELES

访问 http://localhost:8080，你应该可以看到初始页面：

<img src="https://www.flyperstudio.com/images/ELES/Init-page.png" width="200" />

### 6. 不使用时关闭容器（节省资源）

为了节省 CPU 和内存资源，使用完成后请停止容器：

1. 打开 Docker Desktop，进入 Containers 页面。
2. 找到你启动的 ELES 容器。
3. 点击 Stop。

你可以参考下图，Docker Desktop 界面中有 Stop 按钮：

<img src="https://www.flyperstudio.com/images/ELES/docker-image-launch.png" width="200" />

下次需要使用 ELES 时，点击 Start 即可继续使用同一个容器。
