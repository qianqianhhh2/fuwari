---
title: MT 管理器 APK MCP 完整教程：让 AI 在手机上自主逆向分析 Android 应用
published: 2026-07-26
description: "手把手教你使用 MT 管理器内置的 APK MCP 服务，通过 RikkaHub、IMA Copilot 等 AI 客户端连接，实现 AI 驱动的 APK 分析、Smali 代码定位、资源修改与自动重打包签名。安卓逆向从未如此简单。"
tags: ["MT管理器", "MCP", "Android", "逆向工程", "APK", "Smali", "人工智能", "RikkaHub"]
category: 教程
image: "./cover.webp"
draft: false
---

# MT 管理器 APK MCP 完整教程：让 AI 在手机上自主逆向分析 Android 应用

## 前言

在之前的系列教程中，我们覆盖了 IDA Pro、Binary Ninja、Ghidra、JADX 等桌面端逆向工具接入 MCP 的方案。但如果你主要在 Android 手机上做逆向分析——检查 APK 结构、修改 Smali 代码、去除广告、汉化应用——每次都把 APK 传到电脑上再分析，效率其实很低。

**如果 AI 能直接在手机上操作 MT 管理器呢？**

这个想法已经在 2026 年变成了现实。从 v2.26.5 版本开始，MT 管理器内置了 **APK MCP** 服务——这是一项原生的 MCP（Model Context Protocol）功能，让 AI 客户端可以直接调用 MT 管理器的 APK 分析、修改与打包能力。2026 年 6 月的 v2.26.6 更是在原有只读分析的基础上增加了**修改与重打包**能力，打通了"分析 → 修改 → 签名 → 输出 APK"的完整链路。

你可以把它理解为：**AI 负责思考、规划和整理结果，MT 管理器负责在手机上实际操作 APK。** 整个过程在手机本地完成，不需要电脑中转。

本文将从零开始，完成 MT 管理器 APK MCP 的配置，覆盖 RikkaHub + DeepSeek、IMA Copilot、PC 端 MCP 客户端三种连接方案，并提供实战案例。

---

## 一、APK MCP 概述

### 1.1 什么是 APK MCP

APK MCP 是 MT 管理器内建的 MCP 服务端，在手机本地启动一个 HTTP 服务（默认端口 `8787`），采用 **Streamable HTTP** 传输协议。任何支持 MCP 的 AI 客户端都可以连接它。

连接后，AI 可以完成以下操作：

- 读取 APK 的应用信息（包名、版本、签名、权限、入口组件）
- 分析 AndroidManifest.xml 和资源文件
- 读取布局、字符串、图片等资源
- 浏览和搜索 Dex/Smali 代码
- 修改 Smali 代码、资源文件和 AXML
- 自动重打包并签名，生成新的 APK

### 1.2 版本要求

| 项目 | 要求 |
|------|------|
| MT 管理器 | **v2.26.5+**（只读分析）/ **v2.26.6+**（含修改重打包） |
| AI 客户端 | 任意支持 MCP Streamable HTTP 的客户端 |

APK MCP 的修改与重打包功能在 v2.26.6 中为非 VIP 限免。官方在 v2.26.7 中进一步优化了大型 APK 文件的处理性能。

### 1.3 架构

```
┌──────────────────────┐     Streamable HTTP     ┌──────────────────────┐
│  AI 客户端             │ ◄───────────────────► │  MT 管理器             │
│  (RikkaHub / IMA /    │     MCP 协议            │  (手机本地 HTTP 服务)   │
│   PC 端 MCP 客户端)     │                        │  127.0.0.1:8787/mcp   │
└──────────────────────┘                        └──────────────────────┘
                                                          │
                                                          ▼
                                               ┌──────────────────────┐
                                               │  APK 操作             │
                                               │  · 分析信息/资源/Smali │
                                               │  · 修改资源/代码       │
                                               │  · 重打包 + 签名       │
                                               └──────────────────────┘
```

---

## 二、快速开始

### 2.1 第一步：启动 APK MCP 服务

1. 打开 MT 管理器
2. 点击左上角三条横线，打开侧拉栏
3. 在 **工具** 分组中找到 **APK MCP**，点击进入
4. 点击 **启动**

启动成功后，页面会显示 MCP 连接地址。本地地址通常是：

```
http://127.0.0.1:8787/mcp
```

如果 AI 客户端在同一台手机上，复制**本地地址**；如果在同一局域网的电脑或其他设备上，复制**局域网地址**。

### 2.2 第二步：在 AI 客户端配置 MCP

在支持 MCP 的 AI 客户端中添加一个新的 MCP 服务：

- **地址**：粘贴上一步复制的地址
- **传输类型**：选择 **Streamable HTTP**

保存后客户端会自动连接并拉取可用工具列表。

---

## 三、三种连接方案详解

### 3.1 方案一：RikkaHub + DeepSeek（手机端，最推荐）

RikkaHub 是一款支持 MCP 的 AI 聊天客户端，在 Android 端体验最佳，搭配 DeepSeek API 性价比极高。

**步骤：**

1. 从 [DeepSeek 开放平台](https://platform.deepseek.com/) 注册并获取 API Key（充 10 元够用很久）
2. 下载安装 RikkaHub
3. 在 RikkaHub 中进入 **设置 → 模型提供商 → DeepSeek**，填入 API Key
4. 在 RikkaHub 中进入 **设置 → MCP**，粘贴 MT 管理器的本地地址，保存
5. 返回对话界面，确认底部已选择 DeepSeek 模型，MCP 状态显示已连接

> 如果 RikkaHub 和 MT 管理器在同一台手机上，直接使用 `127.0.0.1` 本地地址即可，无需额外配置。

### 3.2 方案二：IMA Copilot（手机端，免费 Token）

腾讯 IMA 内置 Copilot 功能，支持连接 MCP 服务，且每日签到可领取免费算力。

**步骤：**

1. 在应用商店下载 IMA（确保是 Copilot 版本）
2. 开启 MT 管理器的 MCP 服务
3. 在 IMA 中输入 MT 管理器的 MCP 地址，让 AI 连接

**进阶：通过 bore 做内网穿透**

如果你想让 PC 端 AI 客户端连接手机的 MCP 服务，可以使用 bore 进行端口转发：

```bash
# 在 Termux 中安装 bore
pkg install bore

# 启动转发（8787 为 MT 管理器的 MCP 端口）
bore local 8787 --to bore.pub
```

运行后会输出类似 `listening at bore.pub:43062` 的信息，外部可通过 `http://bore.pub:43062/mcp` 连接。

### 3.3 方案三：PC 端 MCP 客户端（同一局域网）

如果你在电脑上使用 Claude Desktop、Cursor、Trae 等客户端，且手机和电脑在同一局域网：

1. 在 MT 管理器的 MCP 页面复制**局域网地址**（例如 `http://192.168.1.100:8787/mcp`）
2. 在 PC 端 MCP 客户端中添加远程 MCP 服务，填入该地址
3. 传输类型选择 Streamable HTTP

> 注意：确保手机和电脑在同一 Wi-Fi 下，且手机防火墙允许该端口通信。建议在 MT 管理器中开启电池白名单，防止服务被系统杀死。

---

## 四、如何把 APK 提供给 AI

APK MCP 支持三种方式让 AI 找到要处理的目标。

### 4.1 方式一：当前 APK 文件（推荐）

1. 在 MT 管理器文件列表中点击某个 APK 文件
2. 停留在 **APK 信息** 对话框中，**不要关闭**
3. 回到 AI 客户端，对 AI 说"当前 APK 文件"

```
请简单分析下当前 APK 文件，告诉我包名、应用名、入口 Activity 和权限列表
```

这是最便捷的方式，不需要记路径，也不会受 MCP 操作目录限制。

### 4.2 方式二：已安装应用

1. 在 MT 管理器进入 **已安装应用** 界面
2. 点击某个 APP，停留在应用信息对话框中
3. 同样让 AI 处理"当前 APK 文件"

```
请分析下当前 APK 文件，重点看它的包名、签名信息和入口组件
```

适合从已安装应用直接开始分析，不需要先手动找到对应的 APK 文件。

### 4.3 方式三：MCP 操作目录

把 APK 文件放到 APK MCP 设置页中指定的 **MCP 操作目录**（默认路径为 `/storage/emulated/0/MT2/mcp/`），然后在聊天时告诉 AI 文件名、应用名或包名。

```
请简单分析下 test.apk
```

```
请简单分析下 MTestAPP 这个应用
```

如果目录下 APK 较多，建议给出明确的文件名；如果只记得应用名或包名，可以先让 AI 列出目录中的可用 APK。

---

## 五、实战案例

### 5.1 快速分析 APK 结构

```
请简单分析下当前 APK 文件，告诉我：
1. 包名、应用名、版本号
2. 入口 Activity 和 Application 类
3. 申请的权限列表
4. 是否有加固或自定义 ClassLoader
5. 主要 Dex 文件的概况
```

AI 会调用 MT 管理器的工具读取 AndroidManifest.xml 和 Dex 信息，返回结构化的分析结果。

### 5.2 搜索 Smali 代码中的关键逻辑

```
在这个 APK 的 Smali 代码中搜索以下关键词，告诉我每个关键词出现在哪些类中：
- checkSignature
- verifyLicense
- isVip
- isPro
- showAd
- purchase
```

AI 会逐类搜索 Dex 中的 Smali 代码，帮你快速定位签名校验、会员验证、广告显示等关键逻辑的位置。

### 5.3 修改应用名并重打包

```
请把当前 APK 的应用名改成"测试版"，然后重新打包签名
```

AI 会自动修改 AndroidManifest.xml 或资源文件中的应用名，重新打包并签名，生成可直接安装的 APK。

### 5.4 去除启动广告

```
分析当前 APK 的 AndroidManifest.xml，找到所有 Activity 组件，定位广告 SDK 相关的 Activity。然后帮我去除广告相关组件和权限，重新打包。
```

AI 会先分析 Manifest，定位广告 SDK（如穿山甲、优量汇、AdMob 等），然后移除相关组件声明和权限，再进一步清理 Smali 代码中的广告调用，最后重打包。

### 5.5 社区实战案例

社区已有大量使用 APK MCP 完成实际逆向任务的案例：

- **林林远程控制破解**：通过 RikkaHub 连接 MT 管理器，AI 自主完成去签 → 分析 → 修改 Smali → 重打包 → 输出破解版
- **微信防撤回 Hook 开发**：AI 搜索 Smali 中的 `revokemsg` 关键词，分析撤回机制，生成 Frida Hook 脚本
- **会员功能解锁**：AI 在 Smali 代码中搜索 `isVip`、`isPro` 等方法，分析会员验证逻辑，提出修改方案

---

## 六、设置与优化

### 6.1 MCP 设置

在 APK MCP 页面右上角进入设置：

| 设置项 | 说明 |
|--------|------|
| MCP 操作目录 | AI 可访问的 APK 文件存放路径，默认 `/storage/emulated/0/MT2/mcp/` |
| 服务端口 | 默认为 8787，可自定义 |
| 历史工作区 | 查看之前 AI 操作过的项目记录 |

### 6.2 性能优化建议

1. **开启电池白名单**：防止系统将 MT 管理器的后台服务杀死
2. **处理大型 APK**：v2.26.7 已优化大型 APK 的 MCP 处理性能，但仍建议处理前先做一次 Dex 对比或去签
3. **分步操作**：复杂修改建议分多步进行，每步确认结果后再继续，便于定位问题

### 6.3 安全提醒

- 只处理你有权分析或修改的 APK
- AI 给出的结论和修改结果需要人工确认
- 修改后的 APK 建议先在测试环境验证
- 不要将 MCP 服务暴露到公网（使用 bore 等工具时注意安全）

---

## 七、与桌面端逆向工具 MCP 的对比

| 维度 | MT 管理器 APK MCP | IDA Pro / Ghidra MCP | JADX MCP |
|------|-------------------|---------------------|----------|
| 运行平台 | **Android 手机本地** | PC/Mac | PC/Mac |
| 目标格式 | APK | 任意二进制 | APK/Dex |
| 修改能力 | **完整（分析 + 修改 + 打包 + 签名）** | 分析为主 | 分析为主 |
| Smali 支持 | **原生** | 无 | 无 |
| 成本 | 免费（非 VIP 功能限免） | 商业 / 免费 | 免费 |
| 学习门槛 | **极低**（手机操作） | 高 | 中 |
| 适用场景 | 安卓应用逆向美化汉化 | 原生二进制 / 恶意软件分析 | APK Java 层分析 |

对于纯 Android APK 的逆向分析和修改需求，MT 管理器 APK MCP 是目前**最高效、最低门槛**的方案。它把"分析 → 修改 → 输出 APK"的完整流程收束在手机上，AI 只需发号施令即可。

---

## 参考资源

- [MT 管理器官方 APK MCP 文档](https://mt2.cn/guide/reverse/apk-mcp.html)
- [MT 管理器更新记录](https://mt2.cn/releases/)
- [RikkaHub — MCP AI 客户端](https://rikka-ai.com/)
- [DeepSeek 开放平台](https://platform.deepseek.com/)
- [MT-MCP-AnalYzer — 社区 APK 分析框架](https://github.com/kggzs/MT-MCP-AnalYzer)
- [BinMT 社区 — AI 逆向实战](https://bbs.binmt.cc/thread-168881-1-1.html)
- [52破解 — MT + DeepSeek 逆向教程](https://www.52pojie.cn/thread-2119351-1-1.html)
