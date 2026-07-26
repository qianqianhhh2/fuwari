---
title: Ghidra 接入 MCP 工具完整教程：补齐逆向工程 AI 化的最后一块拼图
published: 2026-07-26
description: "手把手教你将 Ghidra 通过 MCP（Model Context Protocol）接入 Claude Desktop、Cursor 等 AI 客户端，实现自动反编译、符号重命名、交叉引用追踪等逆向工程任务。补齐 IDA Pro / Binary Ninja / JADX 之后的最后一块拼图。"
tags: ["Ghidra", "MCP", "逆向工程", "人工智能", "安全分析", "NSA"]
category: 教程
image: "./cover.webp"
draft: false
---

# Ghidra 接入 MCP 工具完整教程：补齐逆向工程 AI 化的最后一块拼图

## 前言

在逆向工程四大主流平台中，我们已经覆盖了 IDA Pro MCP、Binary Ninja MCP 和 JADX MCP 的接入教程，唯一缺席的重量级选手就是 **Ghidra**——NSA 开源的逆向工程框架。2025 年 3 月，安全研究者 [LaurieWired](https://github.com/LaurieWired/GhidraMCP) 发布了 **GhidraMCP** 项目，将 Ghidra 的分析能力通过 MCP（Model Context Protocol）暴露给 AI 助手，让"对话式逆向工程"覆盖到了这个免费且功能强大的平台。

截至 2026 年 7 月，GhidraMCP 在 GitHub 上已获得近 **9,000 stars**，最新版本 1.4（兼容 Ghidra 11.3.2），Apache 2.0 开源许可。更值得关注的是，NSA 官方也在 2026 年 7 月 8 日收到了社区提交的 [MCP Server PR #9356](https://github.com/NationalSecurityAgency/ghidra/pull/9356)，这意味着未来 Ghidra 可能**原生内置 MCP 支持**。

GhidraMCP 让 AI 能够自主完成：

- 反编译任意函数并返回类 C 伪代码
- 批量重命名函数、变量、数据标签
- 追踪交叉引用（XREF），构建调用链
- 查看导入/导出符号表、内存段、类/命名空间
- 搜索字符串、定位敏感信息
- 添加注释（EOL、Plate、Pre/Post 多种类型）

本文将从零开始，完成 GhidraMCP 的安装、配置与实战。

---

## 一、GhidraMCP 架构概述

GhidraMCP 由两个核心组件构成：

```
┌─────────────────┐     MCP 协议      ┌──────────────────┐     HTTP/JSON      ┌─────────────────┐
│  AI 客户端        │ ◄──────────────► │  Python 桥接脚本   │ ◄───────────────► │  Ghidra 插件     │
│  (Claude etc.)   │                  │  bridge_mcp_ghidra │                   │  (Java HTTP Svr) │
└─────────────────┘                   └──────────────────┘                   └─────────────────┘
                                              │                                       │
                                        MCP SDK + Requests                   Ghidra API (FlatProgramAPI)
                                        翻译 MCP 工具调用                         暴露 ~10 个 REST 端点
                                              │                                       │
                                        默认端口: 8081                           默认端口: 8080
```

1. **Ghidra 插件（Java）**：在 Ghidra CodeBrowser 内运行一个嵌入式 HTTP 服务器，将 Ghidra 内部 API 封装为 REST 端点（`/decompile_function`、`/list_functions`、`/renameFunction` 等）。
2. **Python 桥接脚本**：实现 MCP 协议，将 AI 客户端的工具调用翻译为对 Ghidra HTTP 服务器的请求，并将结果返回给 AI。

### 1.1 支持的 MCP 客户端

| 客户端 | 配置方式 | 备注 |
|--------|----------|------|
| Claude Desktop | 编辑 `claude_desktop_config.json` | 最常用 |
| Cline | 远程 SSE 服务器 | 需手动启动 Python 桥接 |
| 5ire | 命令行工具配置 | 支持多模型后端 |
| Cursor / Trae 等 | 通过 MCP 配置文件 | 需手动配置 |

### 1.2 关键限制

> 插件仅在 **CodeBrowser** 中运行，不工作在 Project Manager。必须先导入并打开一个二进制文件，HTTP 服务器才会启动。默认绑定 `127.0.0.1:8080`，仅监听本地连接。

---

## 二、前置环境要求

### 2.1 版本要求

| 组件 | 最低版本 | 说明 |
|------|----------|------|
| Ghidra | 11.0+（推荐 11.3.2） | 完全免费，NSA 开源 |
| Python | 3.9+ | MCP SDK 依赖 |
| Java | JDK 21 | 仅从源码构建时需要 |
| MCP 客户端 | 最新版本 | Claude Desktop / Cline / 5ire |

### 2.2 环境预检查

```bash
python --version    # 需 ≥ 3.9
java --version      # 仅从源码构建时需要，JDK 21+
```

---

## 三、安装方式

### 3.1 方式一：插件安装（推荐）

**下载 Release**

从 [GhidraMCP Releases](https://github.com/LaurieWired/GhidraMCP/releases) 下载最新版本 ZIP，例如 `GhidraMCP-1-2.zip`。

**安装插件到 Ghidra**

1. 启动 Ghidra
2. `File` → `Install Extensions`
3. 点击 `+` 按钮
4. 选择下载的 `GhidraMCP-1-2.zip`
5. 重启 Ghidra

**启用插件**

> 关键步骤：确保你在 **CodeBrowser** 而非 Project Manager 中操作。

1. 在 Project Manager 中创建一个项目或打开已有项目
2. 导入目标二进制文件（`File` → `Import File`）
3. 双击打开 CodeBrowser
4. `File` → `Configure` → `Developer`
5. 勾选启用 **GhidraMCPPlugin**

插件启用后，HTTP 服务器自动在 `http://127.0.0.1:8080/` 启动。

**可选配置**：通过 `Edit` → `Tool Options` → `GhidraMCP HTTP Server` 修改端口。

**安装 Python 依赖**

```bash
pip install mcp requests
```

### 3.2 方式二：从源码构建

如果你需要自行构建（例如适配不同 Ghidra 版本）：

1. 从 Ghidra 安装目录复制以下 JAR 文件到项目的 `lib/` 目录：
   - `Ghidra/Features/Base/lib/Base.jar`
   - `Ghidra/Features/Decompiler/lib/Decompiler.jar`
   - `Ghidra/Framework/Docking/lib/Docking.jar`
   - `Ghidra/Framework/Generic/lib/Generic.jar`
   - `Ghidra/Framework/Project/lib/Project.jar`
   - `Ghidra/Framework/SoftwareModeling/lib/SoftwareModeling.jar`
   - `Ghidra/Framework/Utility/lib/Utility.jar`
   - `Ghidra/Framework/Gui/lib/Gui.jar`

2. 构建：

```bash
mvn clean package assembly:single
```

生成的 ZIP 文件位于 `target/` 目录，包含 `GhidraMCP.jar`、`extension.properties` 和 `Module.manifest`。

---

## 四、MCP 客户端配置

### 4.1 Claude Desktop

编辑 Claude Desktop 配置文件（或通过 `Claude` → `Settings` → `Developer` → `Edit Config`）：

```json
{
  "mcpServers": {
    "ghidra": {
      "command": "python",
      "args": [
        "/ABSOLUTE_PATH_TO/bridge_mcp_ghidra.py",
        "--ghidra-server",
        "http://127.0.0.1:8080/"
      ]
    }
  }
}
```

配置文件路径（macOS）：`~/Library/Application Support/Claude/claude_desktop_config.json`

配置完成后重启 Claude Desktop，应在聊天界面看到 Ghidra 工具集。

### 4.2 Cline

Cline 需要通过 SSE 传输协议手动启动桥接：

```bash
python bridge_mcp_ghidra.py --transport sse --mcp-host 127.0.0.1 --mcp-port 8081 --ghidra-server http://127.0.0.1:8080/
```

然后在 Cline 中添加远程服务器：
- **Server Name**: GhidraMCP
- **Server URL**: `http://127.0.0.1:8081/sse`

### 4.3 5ire

在 5ire 中进入 `Tools` → `New`：

| 配置项 | 值 |
|--------|-----|
| Tool Key | ghidra |
| Name | GhidraMCP |
| Command | `python /ABSOLUTE_PATH_TO/bridge_mcp_ghidra.py` |

---

## 五、工具集详解

GhidraMCP 1.4 通过 MCP 暴露了以下核心工具：

### 5.1 反编译与分析

| 工具 | 功能 | 示例 |
|------|------|------|
| `decompile_function` | 反编译指定函数为 C 伪代码 | `decompile_function("main")` |
| `list_functions` | 列出二进制中所有函数 | — |
| `list_methods` | 列出所有方法（同 `list_functions`） | — |
| `list_classes` | 列出所有类 / 命名空间 | — |
| `list_segments` | 显示内存分段信息 | — |
| `list_imports` | 列出导入符号表 | — |
| `list_exports` | 列出导出符号表 | — |
| `list_strings` | 列出所有已定义字符串 | — |

### 5.2 重命名与注释

| 工具 | 功能 | 示例 |
|------|------|------|
| `rename_function` | 按当前名称重命名函数 | `rename_function("FUN_001012a0", "decrypt_payload")` |
| `rename_data` | 重命名数据标签 | `rename_data("DAT_00405000", "c2_server_url")` |
| `set_comment` | 为地址添加注释 | `set_comment("0x401000", "此处分发 C2 指令")` |

### 5.3 交叉引用

| 工具 | 功能 | 示例 |
|------|------|------|
| `get_xrefs_to` | 获取指向指定地址的所有引用 | `get_xrefs_to("0x4012a0")` |
| `get_xrefs_from` | 获取从指定地址出发的引用 | `get_xrefs_from("0x4012a0")` |
| `get_function_xrefs` | 获取函数的交叉引用 | `get_function_xrefs("decrypt_payload")` |

### 5.4 搜索

| 工具 | 功能 | 示例 |
|------|------|------|
| `search_functions_by_name` | 按名称搜索函数 | `search_functions_by_name("crypt")` |
| `search_strings` | 搜索包含指定模式的字符串 | `search_strings("http")` |

---

## 六、实战案例

### 6.1 一键标注恶意软件

将可疑 EXE 导入 Ghidra，在 AI 客户端输入：

> 我刚刚加载了一个 32 位的 dropper 样本。请完成以下操作：
> 1. 反编译 main 函数
> 2. 根据分析结果重命名每个函数
> 3. 为每个函数添加一句话注释
> 4. 高亮所有看起来像 C2 地址的字符串

AI 会依次调用 `list_functions` → 遍历 → `decompile_function` → 自动命名（如 `download_payload`、`decode_base64`、`persist_registry`）→ `rename_function` → `set_comment` → `search_strings("http")`。

**效果**：人工 45 分钟的工作缩短为 30 秒的提示词输入。

### 6.2 跨二进制调用链追踪

当一个固件解包为多个独立二进制（bootloader、kernel、用户态辅助程序）：

> 追踪 "upgrade" 命令从 Web UI 到 flash-write 的完整调用路径，包括跨二进制调用。

AI 会在不同二进制间切换，追踪完整的控制流。

### 6.3 批量反编译分析

> 反编译所有名称包含 "crypto" 的函数，总结这个二进制的加密逻辑。

AI 会批量调用 `search_functions_by_name("crypto")` → 逐个 `decompile_function` → 汇总分析结果。

---

## 七、常见问题排查

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| 插件不在 `Configure → Developer` 中 | 未重启 Ghidra 或不在 CodeBrowser 中 | 重启 Ghidra，确保在 CodeBrowser 操作 |
| HTTP 服务器无响应 | 插件未启用或没有加载程序 | 确认 CodeBrowser 中已打开二进制文件且插件已勾选 |
| `Connection refused` | 端口不一致或被防火墙阻止 | 检查端口配置，确认防火墙允许 localhost 连接 |
| Python 桥接报错 | 缺少 MCP SDK | `pip install mcp requests` |

---

## 八、与三大平台的对比总结

至此，逆向工程四大主流平台全部完成 MCP 接入。从功能完整度来看：

| 维度 | IDA Pro MCP | Binary Ninja MCP | GhidraMCP | JADX MCP |
|------|-------------|------------------|-----------|----------|
| 许可证 | 商业（昂贵） | 商业（需 License） | **完全免费开源** | 免费开源 |
| 插件语言 | Python | Python | **Java（原生）** | Java |
| 安装复杂度 | 中等 | 简单 | 简单 | 中等 |
| AI 能力覆盖 | 反编译/重命名/XREF/类型 | 反编译/重命名/XREF/类型 | 反编译/重命名/XREF/注释 | 反编译/类/方法 |
| 社区活跃度 | 高 | 中 | **高（9k stars）** | 中 |

Ghidra 的免费开源 + MCP 的组合，使其成为学习和入门 AI 辅助逆向工程的**最佳选择**。无论是学生、独立安全研究员还是预算有限的小团队，都可以零成本获得与商业平台相当的 AI 驱动逆向体验。

---

## 参考资源

- [GhidraMCP GitHub 仓库](https://github.com/LaurieWired/GhidraMCP)
- [Ghidra 官方下载](https://ghidra-sre.org/)
- [MCP 协议规范](https://modelcontextprotocol.io/)
- [NSA Ghidra PR #9356 — 官方 MCP 支持提案](https://github.com/NationalSecurityAgency/ghidra/pull/9356)
- [Letting LLMs Automate Reverse Engineering in Ghidra — BrightCoding Blog](https://blog.brightcoding.dev/2025/09/25/letting-llms-automate-reverse-engineering-in-ghidra)
