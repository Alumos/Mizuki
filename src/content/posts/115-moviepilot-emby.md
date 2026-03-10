---
title: 筑巢引凤：115网盘 + MoviePilot + Emby 自动化影视库搭建实践
published: 2026-03-10
description: '通过 115 网盘、MoviePilot 与 Emby 构建自动化影视库，实现资源管理、STRM 生成与云端直连播放的一体化家庭影音系统。'
image: 'https://img.alumos.cn/file/MizukiBlog/1773158382720_image.webp'
tags: [NAS, Docker, 115网盘, MoviePilot, Emby, STRM]
category: '技术分享'
draft: false
lang: 'zh'
---

在数字化内容日益丰富的今天，许多影音爱好者都会面临一个问题：

> **如何在保证资源规模的同时，获得类似 Netflix 的观影体验？**

理想的家庭影视系统应该具备以下特征：

- 海量资源存储  
- 自动整理与刮削  
- 精美的海报墙界面  
- 跨设备流畅播放  
- 尽可能减少人工维护  

经过一段时间的实践与调整，我最终构建了一套 **以 115 网盘为资源核心、MoviePilot 为自动化管理中心、Emby 为播放终端** 的自动化影视库系统。

这套方案的优势在于：

- **云端资源存储**
- **自动生成 STRM**
- **客户端直连播放**
- **几乎零人工维护**

本文将完整记录这套系统的架构与部署思路。

---

# 系统整体架构

整个系统可以划分为三个核心层级：

1️⃣ **存储层：115 网盘**  
2️⃣ **管理层：MoviePilot**  
3️⃣ **展示层：Emby**

系统逻辑如下：

```
115 网盘
   │
   │ (MoviePilot 插件挂载)
   ▼
MoviePilot
   │
   │ 自动整理 / 生成 STRM
   ▼
Emby
   │
   │ 海报墙 / 播放
   ▼
多终端设备
TV / PC / 手机 / VR
```

与传统 NAS 影视方案不同的是：

> **115 网盘挂载、STRM 文件生成以及播放优化全部由 MoviePilot 插件体系完成。**

这使得整个系统的自动化程度大幅提升。

---

# MoviePilot：自动化核心

![MoviePilot](https://img.alumos.cn/file/MizukiBlog/1773158148260_image.webp)

在这套系统中，**MoviePilot 是真正的控制中心**。

它负责：

- 媒体整理
- 文件重命名
- 元数据刮削
- STRM 文件生成
- 与 Emby 的媒体库联动

相比传统媒体管理工具，MoviePilot 的优势在于 **插件生态非常灵活**。

本方案的关键能力，主要依赖于 **DDSRem 开发的三个插件**。

---

# 关键插件体系

项目作者：

DDSRem  
https://github.com/DDSRem-Dev

在本方案中，需要使用以下三个插件：

### 1️⃣ 115网盘储存

插件作用：

- 直接在 MoviePilot 中挂载 115 网盘
- 将云端文件映射为本地路径

例如：

```
/volume3/115-MP
```

这样 Docker 容器即可像访问本地磁盘一样访问 115 网盘中的资源。

优势：

- 无需额外 rclone
- 无需 WebDAV
- 挂载稳定性高

---

### 2️⃣ 115网盘 STRM 助手

该插件负责：

- 自动生成 **STRM 文件**
- 维护媒体目录结构
- 与 MoviePilot 的整理规则协同工作

生成后的结构示例：

```
电影
 └─ 沙丘 (2021)
     ├─ 沙丘 (2021).strm
     └─ poster.jpg
```

STRM 文件本质上只是一个 **指向云端视频地址的文本文件**。

优点：

- 不占用本地存储
- 媒体库加载速度快
- 适合云盘影视库

---

### 3️⃣ Emby 302 反向代理

这是播放体验提升的关键插件。

它的作用是：

> **让播放器直接从 115 服务器获取视频流，而不是通过 NAS 中转。**

工作原理：

```
播放器
   │
   │ 请求视频
   ▼
Emby
   │
   │ 302 重定向
   ▼
115 CDN
```

优势：

- NAS 几乎不占带宽
- 播放速度更快
- 大幅降低卡顿概率

---

# Docker 部署

整个系统通过 **Docker Compose** 管理。

部署环境：

- NAS：绿联 NAS
- 容器管理：1Panel
- 媒体目录：`/volume3/115-MP`

---

# MoviePilot 部署

```yaml
version: "3.8"

services:
  moviepilot:
    image: jxxghp/moviepilot-v2:latest
    container_name: moviepilot-v2
    hostname: moviepilot-v2
    network_mode: bridge

    ports:
      - "3000:3000"
      - "9000:9000"

    stdin_open: true
    tty: true

    volumes:
      - /volume3/115-MP:/media
      - /volume1/docker/moviepilot/config:/config
      - /volume1/docker/moviepilot/core:/moviepilot/.cache/ms-playwright
      - /var/run/docker.sock:/var/run/docker.sock:ro

    environment:
      NGINX_PORT: 3000
      PORT: 3001
      PUID: 0
      PGID: 0
      UMASK: 000
      TZ: Asia/Shanghai
      SUPERUSER: admin
      SUPERUSER_PASSWORD: admin

    restart: always
```

部署完成后，建议：

- **立即修改管理员密码**
- 安装上述三个关键插件

---

# Emby 媒体服务器

![Emby](https://img.alumos.cn/file/MizukiBlog/1773158174013_image.webp)

Emby 是最终的 **媒体展示与播放平台**。

主要功能包括：

- 海报墙展示
- 自动元数据
- 字幕管理
- 多终端播放
- 硬件转码

为了提升播放性能，我启用了 **Intel 核显硬件转码**。

---

# Emby Docker 配置

```yaml
version: "3.8"

services:
  emby:
    image: amilys/embyserver
    container_name: emby-server
    restart: always

    devices:
      - /dev/dri:/dev/dri

    environment:
      - PUID=0
      - PGID=0
      - TZ=Asia/Shanghai

    volumes:
      - /volume1/docker/emby/config:/config
      - /volume3/115-MP:/media

    ports:
      - 8096:8096
```

需要注意：

MoviePilot 与 Emby 的媒体路径必须一致：

```
/volume3/115-MP
```

否则 Emby 无法正确识别 STRM 文件。

---

# 实际使用体验

经过一段时间的运行，这套系统的使用体验非常接近商业流媒体平台。

日常流程如下：

1️⃣ 资源进入 115 网盘  
2️⃣ MoviePilot 自动整理  
3️⃣ 自动生成 STRM  
4️⃣ Emby 自动刷新媒体库  
5️⃣ 客户端直接播放  

整个过程 **几乎不需要人工干预**。

当打开 Emby，看到整齐的海报墙时，所有的技术折腾都会变得值得。

---

# 总结

这套 **115 + MoviePilot + Emby** 的架构具备几个明显优势：

- 云端存储，无需本地大容量硬盘  
- 自动生成 STRM，媒体库结构清晰  
- 302 直连播放，NAS 几乎零带宽压力  
- 自动整理与刮削，维护成本极低  

如果你也希望搭建一套 **自动化、稳定且优雅的家庭影音系统**，这套方案值得尝试。

---

**Alumos**  
写于 2026 年 3 月 10 日