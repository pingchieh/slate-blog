---
title: 低成本搭建Navidrome音乐服务
description: 结合 Navidrome 的元数据管理能力、Cloudflare Workers 的边缘计算能力以及已有的云存储（如 Cloudflare R2、AWS S3，或通过 Alist 挂载的各种网盘），在存储空间极其有限的 VPS 或容器服务上，搭建功能完善、响应迅速的个人音乐流媒体服务。
tags: [Navidrome, Cloudflare, 音乐流媒体, 云存储]
pubDate: 2025-04-19
draft: false
---

## 前言

为了搭建个人音乐库并节省会员费用，我尝试了多种方案。然而，传统方案如个人PC（耗电、多平台访问受限）、Emby/Jellyfin（资源要求高）、Alist+WebDAV（播放器兼容性差）和Navidrome+Rclone（扫描慢、播放卡顿）均存在不足。
经过分析，我发现可以通过**劫持 Navidrome 的流服务请求**，并将其重定向到云存储上的实际音乐文件，从而在存储空间极小的服务器上搭建个人音乐流媒体服务。本文将介绍如何利用 Navidrome、Cloudflare Workers 和云存储实现这一目标。

## 核心思路

利用本地计算机生成 Navidrome 的音乐元数据数据库，然后将此数据库部署到容器上的 Navidrome 实例（该实例不扫描实际文件）。接着，通过 Cloudflare Workers 拦截 Navidrome 的音乐文件请求，将请求重定向到实际存储音乐文件的云端地址。

## 前提条件

* **音乐文件：** 本地计算机存有一份完整的音乐库，同时在云存储（如 Cloudflare R2, AWS S3, Google Cloud Storage, 或通过 Alist 暴露的网盘等）也有一份完全一致的副本。
* **本地环境：** 一台可以运行 Navidrome 并访问完整音乐库的本地计算机，用于初次扫描。
* **服务器环境：** 一台存储空间较小的 VPS 或容器服务，用于运行 Navidrome 实例。
* **Cloudflare 账户：** 用于 DNS 解析、Workers 和 D1 数据库服务。域名需使用 Cloudflare DNS 并开启代理。
* **云存储访问：** 云存储需支持通过公开或临时签名 URL 访问文件。

## 工具

* Navidrome 程序
* SQLite 编辑工具（如 DBeaver、sqlitestudio）
* Cloudflare `wrangler` CLI 工具 (`npm install -g wrangler`)

## 实施步骤

### 第一步：本地扫描与数据库准备

1. 在本地计算机配置并运行 Navidrome，确保 `MusicFolder` 指向完整本地音乐库路径。等待 Navidrome 完成首次全量扫描。
2. 扫描完成后，在 Navidrome 的 `DataFolder` 中找到 `navidrome.db` 文件。此文件包含所有音乐元数据和文件路径信息。
3. 将 `navidrome.db` 上传到计划运行远程 Navidrome 的服务器数据目录中。

### 第二步：远程部署 Navidrome (仅元数据模式)

在存储有限的服务器上配置 Navidrome，使其仅加载上传的数据库，而不进行文件扫描。
创建或修改 Navidrome 配置文件 (`navidrome.toml`)，配置如下：

```toml
# 指定数据目录，确保 navidrome.db 文件在此目录下
#DataFolder = "/path/to/your/navidrome/data" 
# MusicFolder 理论上可以随意指定一个空目录，因为扫描已禁用
#MusicFolder = "/path/to/dummy/music/folder" 

# 禁用内置的指标收集（可选）
EnableInsightsCollector = false 
# 设置默认语言（可选）
DefaultLanguage = 'zh-Hans' 

# --- 关键配置：禁用扫描 ---
# 完全禁用扫描器
Scanner.Enabled = false 
# 禁用文件监控等待（无需监控本地文件变化）
Scanner.WatcherWait = 0 
# 禁用启动时扫描
Scanner.ScanOnStartup = false 
```

使用 `navidrome -c navidrome.toml` 命令启动 Navidrome 服务。

### 第三步：使用 Cloudflare 代理

确保你的域名已通过 Cloudflare DNS 解析，并且 Navidrome 服务对应的 DNS 记录已开启 Cloudflare 代理（小云朵图标为橙色）。这是后续使用 Workers 的前提。

### 第四步：创建文件 ID 到云端路径的映射数据库 (Cloudflare D1)

这一步建立 Navidrome 内部文件 ID 与云存储实际文件路径的对应关系。

1. 在本地复制一份 `navidrome.db`。使用 SQLite 编辑工具打开副本。
2. 访问 `media_file` 表，删除 `id`, `album_id`, `path` 列之外的所有列。
3. 将修改后的 `media_file` 表导出为 SQL 文件 (`media_file.sql`)。
4. 打开 `media_file.sql` 文件，删除文件开头和结尾的事务控制语句（`BEGIN TRANSACTION;` 和 `COMMIT TRANSACTION;`）。
5. 使用 `wrangler` CLI 或 Cloudflare 控制台创建一个 D1 数据库，例如命名为 `navidrome-db`。
6. 使用 `wrangler` 将清理后的 SQL 文件导入 D1 数据库。确保已登录 `wrangler` 并执行：

  ```bash
  npx wrangler d1 execute navidrome-db --remote --file=media_file.sql
  ```

### 第五步：创建 Cloudflare Worker 拦截并重定向请求

创建一个 Worker 脚本，拦截 Navidrome 请求，查询 D1 数据库，并重定向到云存储。

1. 使用 `wrangler` 或 Cloudflare 控制台创建一个新的 Worker 服务。
2. 在 Worker 设置中，绑定第四步创建的 D1 数据库 (`navidrome-db`)。设置绑定名称为 `DB`。
3. 编辑 Worker 代码文件 (`index.js` 或 `src/index.js`)，粘贴以下代码：

    ```javascript
    // !! 重要安全提示 !!
    // 此基础实现未处理身份验证。任何知道您 Navidrome 域名的人理论上
    // 可通过构造 /rest/stream?id=... 请求尝试访问音乐文件。

    const BASE_URL = "https://your.cloud/path/"; // <-- 替换为你的云存储基础路径

    export default {
      async fetch(request, env, ctx) {
        const url = new URL(request.url);
        const { pathname } = url;

        // 拦截音乐流和封面请求
        if (pathname === "/rest/stream" || pathname === "/rest/getCoverArt") {
          let file_id = url.searchParams.get("id");
          if (!file_id) {
            return new Response("Missing id parameter", { status: 400 });
          }

          // 处理封面请求的特殊ID格式
          if (file_id.startsWith("al-")) {
            file_id = file_id.substring(3);
          }

          // 查询 D1 数据库获取文件路径
          const filePathResult = await env.DB.prepare(
            "SELECT path FROM media_file where id = ?1 or album_id = ?1 LIMIT 1"
          )
            .bind(file_id)
            .first("path");

          if (!filePathResult) {
            return new Response("File not found", { status: 404 });
          }
          let filePath = filePathResult; // 获取路径字符串

          // 根据请求类型调整路径 (示例：封面路径处理)
          if (pathname === "/rest/getCoverArt") {
            // 示例：假设封面图片与音乐文件同名但扩展名不同，且存放在 /COVER/ 目录下
            filePath = filePath.replace(/\.[^/.]+$/, ".jpeg"); // 替换扩展名
            filePath = "/COVER/" + encodeURIComponent(filePath).replace(/%2F/g, "/"); // 添加封面目录并编码
          } else {
             // 示例：音乐文件存放在 /MP3/ 目录下
            filePath = "/MP3/" + encodeURIComponent(filePath).replace(/%2F/g, "/"); // 添加音乐目录并编码
          }

          // 构建完整的云存储文件 URL
          const fileURL = BASE_URL + filePath;

          // 将请求重定向或转发到云存储地址，并利用 Cloudflare 缓存
          return fetch(fileURL, {
            cf: {
              cacheTtl: 3600 * 24 * 365, // 缓存一年
              cacheEverything: true,    // 缓存所有内容
            },
          });
        }

        // 非 /rest/stream 或 /rest/getCoverArt 请求，可选择处理或返回默认响应
        // 例如，可配置 Worker 仅处理特定路径
         return new Response("OK", { status: 200 }); // 或其他默认响应
      },
    };
    ```

    **注意：** 请将 `BASE_URL` 替换为你的云存储实际访问基础路径，并根据你的云存储文件组织结构调整封面和音乐文件路径的拼接逻辑 (`/COVER/`, `/MP3/`)。

4. 在 Cloudflare 控制台为你的 Worker 添加路由规则，确保它拦截所有发往 `/rest/stream` 和 `/rest/getCoverArt` 的请求：
    * `你的 Navidrome 域名/rest/stream*`
    * `你的 Navidrome 域名/rest/getCoverArt*`

至此，你已成功搭建了一个低成本、高性能的个人音乐流媒体服务。
