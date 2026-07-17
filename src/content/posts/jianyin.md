---
title: 简音 - 一个现代化的 Android 音乐播放器
published: 2026-07-17
description: "基于 Kotlin 和 Jetpack Compose 开发的多平台音乐播放器，支持网易云、Bilibili、本地音乐"
tags: ["Android", "Kotlin", "Jetpack Compose", "音乐", "开源"]
category: 项目
draft: false
---

## 关于简音

[简音](https://github.com/qianqianhhh2/jianyin) 是一款使用 **Kotlin** 和 **Jetpack Compose** 开发的现代化 Android 音乐播放器，采用 MVVM 架构，遵循 Material Design 3 设计规范，为你带来流畅、美观的音乐体验。

> 项目已不再依赖第三方 API 服务，独立实现了网易云音乐和 Bilibili 的音乐接口。

---

## 核心功能

### 多平台音乐聚合
集成网易云音乐、Bilibili 和本地音乐三大音源，一个应用畅听全网音乐。

### 精美 UI 设计
- **Material You 动态取色** - 根据壁纸自动生成主题色，或自定义种子色
- **毛玻璃效果** - 基于 Haze 库实现的精美模糊视觉
- **动态流光背景** - Android 13+ RuntimeShader (AGSL) 驱动的全屏播放器背景，颜色随封面实时取色
- **深色模式** - 完整的浅色/深色主题，支持跟随系统

### 播放体验
- 后台播放，支持锁屏控制和通知栏控制
- 桌面歌词悬浮窗
- Bilibili 分 P 视频音频播放
- 多种播放模式切换

### 实用工具
- **音乐下载** - 支持在线下载和本地文件导入，自动读取音频元数据（封面、标题、艺术家）
- **播放列表管理** - 创建自定义歌单，支持 Bilibili 收藏夹同步
- **数据备份恢复** - 一键备份/恢复播放列表和设置
- **用户统计** - 追踪播放次数、连续启动天数、常听时段

### 趣味细节
- 启动引导页带彩纸 (Konfetti) 粒子特效
- 连续启动 1/7/30/365 天的里程碑庆祝弹窗
- 首页「一言」展示

---

## 技术栈亮点

| 分类 | 技术 |
|------|------|
| 语言 | Kotlin 2.1.0 |
| UI | Jetpack Compose + Material Design 3 |
| 架构 | MVVM + Repository |
| 播放引擎 | Media3 ExoPlayer |
| 网络 | Retrofit + OkHttp |
| 图片 | Coil |
| 特效 | Haze 毛玻璃 / Konfetti 彩纸 / AGSL 着色器 |
| 存储 | DataStore + Room |
| 安全 | AndroidX Security Crypto |

---

## 快速体验

```bash
git clone https://github.com/qianqianhhh2/jianyin.git
```

用 Android Studio 打开项目，连接设备即可运行。也可以在 [Releases](https://github.com/qianqianhhh2/jianyin/releases) 页面直接下载 APK。

---

## 项目地址

::github{repo="qianqianhhh2/jianyin"}

---

*欢迎 Star 和 PR，QQ交流群：1082723263*
