---
title: Binary Ninja 接入 MCP 工具完整教程：让 AI 驱动二进制逆向分析
published: 2026-07-21
description: "手把手教你将 Binary Ninja 通过 MCP（Model Context Protocol）接入 Claude Desktop、Cursor、Trae 等 AI 客户端，实现自动反编译、函数分析、交叉引用追踪、类型定义等逆向工程任务。"
tags: ["Binary Ninja", "MCP", "逆向工程", "人工智能", "安全分析"]
category: 教程
draft: false
image: "./cover.webp"
---

# Binary Ninja 接入 MCP 工具完整教程：让 AI 驱动二进制逆向分析

## 前言

在逆向工程领域，IDA Pro、Ghidra 和 Binary Ninja 是三大主流平台。继 IDA Pro MCP 和 JADX MCP 之后，**Binary Ninja MCP** 将 Binary Ninja 的分析能力通过 MCP（Model Context Protocol）暴露给 AI 助手，实现"对话式逆向工程"。

Binary Ninja MCP 由社区开发者 [fosdickio](https://github.com/fosdickio/binary_ninja_mcp) 维护，架构上分为两部分：**Binary Ninja 插件**（HTTP 端点暴露分析能力）和 **MCP 桥接组件**（连接 AI 客户端与插件服务器）。支持的 MCP 客户端包括 Claude Desktop、Cursor、Cline、Roo Code、Windsurf、Claude Code 和 LM Studio。

借助 Binary Ninja MCP，你可以用自然语言让 AI 完成：

- 反编译指定函数并返回 C 伪代码
- 搜索和重命名函数、变量、数据标签
- 追踪交叉引用（Code References）
- 管理注释（地址注释和函数注释）
- 查看导入/导出符号表、内存段、命名空间
- 定义结构体、枚举、联合体等自定义类型
- 编辑函数签名和变量类型

本文将从零开始，完成 Binary Ninja MCP 的安装、配置与实战。

---

## 一、前置环境要求

### 1.1 版本要求

| 组件 | 最低版本 | 说明 |
|------|----------|------|
| Binary Ninja | 任意当前版本 | 需 Commercial 或 Personal License |
| Python | 3.12+ | 用于虚拟环境创建 |
| MCP 客户端 | 最新版本 | Claude Desktop / Cursor / Cline 等 |

> Binary Ninja Free 版本不支持的场景，可改用 GhidraMCP 替代。Python 版本需 ≥ 3.12。

### 1.2 环境预检查

```bash
python --version    # 需 ≥ 3.12
git --version       # 手动安装时需要
```

MCP 客户端需**在安装插件之前**先安装好，插件安装过程会自动检测并配置已安装的客户端。

---

## 二、安装方式

Binary Ninja MCP 提供两种安装方式：**插件管理器安装**（推荐）和**手动安装**。两种方式均支持 MCP 客户端自动配置。

### 2.1 方式一：通过 Binary Ninja 插件管理器安装（推荐）

1. 打开 Binary Ninja
2. 导航到 `Plugins > Manage Plugins`
3. 搜索 **"Binary Ninja MCP"**
4. 点击 `Install`
5. 重启 Binary Ninja

插件管理器会自动完成以下操作：

- 下载插件到 Binary Ninja 插件目录
- 创建 Python 虚拟环境（`.venv`）
- 安装 Python 依赖（`bridge/requirements.txt`）
- **检测并自动配置已安装的 MCP 客户端**
- 创建 `.mcp_auto_setup_done` 标记文件，避免重复配置

### 2.2 方式二：手动安装

如需手动安装，将仓库克隆到 Binary Ninja 插件目录：

```bash
# macOS
cd ~/Library/Application\ Support/Binary\ Ninja/plugins/
git clone https://github.com/fosdickio/binary_ninja_mcp

# Linux
cd ~/.binaryninja/plugins/
git clone https://github.com/fosdickio/binary_ninja_mcp

# Windows
cd %APPDATA%\Binary Ninja\plugins\
git clone https://github.com/fosdickio/binary_ninja_mcp
```

手动安装后重启 Binary Ninja，插件会在首次加载时自动执行配置流程。

### 2.3 自动客户端配置

插件支持自动检测和配置以下 MCP 客户端：

| 序号 | 客户端 | 推荐场景 |
|------|--------|----------|
| 1 | Cline（推荐） | VS Code 用户 |
| 2 | Roo Code | VS Code 用户 |
| 3 | Claude Desktop（推荐） | 独立使用 |
| 4 | Cursor | IDE 集成 |
| 5 | Windsurf | IDE 集成 |
| 6 | Claude Code | 命令行使用 |
| 7 | LM Studio | 本地模型用户 |

---

## 三、MCP 桥接配置

### 3.1 使用 npm 包配置（推荐）

Binary Ninja MCP 提供了 npm 包，配置最为简洁。在 MCP 客户端配置文件中添加：

```json
{
  "mcpServers": {
    "binary-ninja-mcp": {
      "command": "npx",
      "args": ["-y", "binary-ninja-mcp", "--host", "localhost", "--port", "9009"]
    }
  }
}
```

### 3.2 使用 Python 桥接配置（手动模式）

Claude Desktop 用户可通过配置文件路径进行手动配置：

**macOS** 配置文件位置：
```
~/Library/Application Support/Claude/claude_desktop_config.json
```

**Windows** 配置文件位置：
```
%APPDATA%\Claude\claude_desktop_config.json
```

添加以下内容（注意替换绝对路径）：

```json
{
  "mcpServers": {
    "binary_ninja_mcp": {
      "command": "/绝对路径/到/binary_ninja_mcp/.venv/bin/python",
      "args": [
        "/绝对路径/到/binary_ninja_mcp/bridge/binja_mcp_bridge.py"
      ]
    }
  }
}
```

> 注意：必须使用虚拟环境中的 Python 解释器，确保依赖项可用。

### 3.3 命令行工具管理

插件提供了命令行管理工具用于手动控制客户端配置：

```bash
# 自动配置所有已安装的 MCP 客户端
python scripts/mcp_client_installer.py --install

# 移除所有 MCP 客户端配置
python scripts/mcp_client_installer.py --uninstall

# 输出通用 JSON 配置片段
python scripts/mcp_client_installer.py --config
```

---

## 四、启动与使用

### 4.1 启动 MCP 服务器

1. 打开 Binary Ninja，加载目标二进制文件
2. 启动 MCP 服务器：`Plugins > MCP Server > Start MCP Server`
3. 状态栏左下角会显示服务器运行状态

### 4.2 连接 AI 客户端

以 Claude Desktop 为例：

1. 确保 Binary Ninja 中 MCP 服务器已启动
2. 启动 Claude Desktop
3. 集成自动生效，在对话中直接询问当前二进制文件的相关问题

### 4.3 多文件支持

Binary Ninja MCP 支持同时打开多个二进制文件。当切换活动标签页时，AI 客户端的分析上下文会自动跟随当前活跃的目标。

---

## 五、支持的工具列表

Binary Ninja MCP 暴露 25 个工具，涵盖二进制分析的完整工作流：

### 5.1 元信息与导航

| 工具名 | 功能 |
|--------|------|
| `get_binary_status` | 获取已加载二进制文件的当前状态 |
| `list_segments` | 列出所有内存段 |
| `list_exports` | 列出导出函数和符号 |
| `list_imports` | 列出导入符号 |
| `list_methods` | 列出所有函数名 |
| `list_classes` | 列出命名空间和类名 |
| `list_namespaces` | 列出所有非全局命名空间 |
| `list_data_items` | 列出定义的数据标签及其值 |
| `list_sections` | 列出所有 sections |

### 5.2 反编译与分析

| 工具名 | 功能 |
|--------|------|
| `decompile_function` | 按函数名反编译，返回 C 伪代码 |
| `get_assembly_function` | 按名称或地址获取函数汇编表示 |
| `function_at` | 获取指定地址所属的函数名 |
| `code_references` | 获取调用指定函数的所有引用位置 |

### 5.3 重命名

| 工具名 | 功能 |
|--------|------|
| `rename_function` | 按当前名称重命名函数 |
| `rename_data` | 重命名指定地址的数据标签 |
| `rename_variable` | 重命名指定函数中的变量 |
| `search_functions_by_name` | 按子串搜索函数名 |

### 5.4 注释管理

| 工具名 | 功能 |
|--------|------|
| `set_comment` | 在指定地址设置注释 |
| `set_function_comment` | 为函数设置注释 |
| `get_comment` | 获取指定地址的注释 |
| `get_function_comment` | 获取函数的注释 |
| `delete_comment` | 删除指定地址的注释 |
| `delete_function_comment` | 删除函数的注释 |

### 5.5 类型编辑

| 工具名 | 功能 |
|--------|------|
| `get_user_defined_type` | 获取自定义类型定义（结构体/枚举/联合体） |
| `define_types` | 从 C 字符串添加类型定义 |
| `retype_variable` | 修改函数中变量的类型 |
| `edit_function_signature` | 编辑函数签名 |

---

## 六、实战示例

### 6.1 生成二进制分析报告

**提示词**：

```
为当前二进制文件生成一份完整的分析报告，包括文件基本信息、段布局、
导入/导出符号概览、关键函数识别和潜在的安全关注点。
```

AI 将依次调用 `get_binary_status`、`list_segments`、`list_exports`、`list_imports` 等工具，聚合信息生成结构化报告。

### 6.2 批量函数重命名

**提示词**：

```
分析当前二进制文件的导入表，将每个导入函数被调用的所有包装函数
（如 sub_xxxxxx 形式的函数）重命名为有意义的名字，格式为 wrapper_<导入函数名>。
只处理直接包装函数（函数体小于 10 行汇编）。
```

AI 会先调用 `list_imports` 获取导入符号，然后对每个符号使用 `code_references` 查找调用者，调用 `get_assembly_function` 判断函数大小，最后通过 `rename_function` 批量重命名。

### 6.3 数据结构逆向

**提示词**：

```
搜索当前二进制文件中所有引用 malloc 且分配大小 ≤ 0x100 的位置。
对每个分配点，推断被分配的结构体类型，使用 define_types 定义对应结构体，
然后用 retype_variable 为变量设置类型。
```

AI 会追踪 `malloc` 调用的交叉引用，分析分配后的字段访问模式来推断结构体布局，再定义类型并应用。

### 6.4 CTF 解题辅助

**提示词**：

```
目标：找出用于验证输入的密钥或密码逻辑。
步骤：
1. 搜索 "Correct"、"Wrong"、"Success"、"Fail" 等字符串
2. 追踪这些字符串的交叉引用，找到判断逻辑所在的函数
3. 反编译这些函数，分析密码验证算法
4. 如果需要逆向算法，推导出正确输入
```

### 6.5 漏洞搜索

**提示词**：

```
在当前二进制文件中搜索潜在的内存安全漏洞：
1. 找到所有调用 strcpy、sprintf、gets 的函数
2. 分析每次调用，判断目标缓冲区大小是否足以容纳源数据
3. 对于缓冲区可能不足的情况，标注为目标漏洞点
4. 对每个潜在漏洞点，检查是否有栈保护（canary）、ASLR 等缓解措施
```

---

## 七、与三大平台 MCP 方案对比

| 对比维度 | IDA Pro MCP | GhidraMCP | Binary Ninja MCP |
|----------|-------------|-----------|-------------------|
| 工具数量 | 40+ | 50+ | 25 |
| 许可证 | 开源 | 开源 | 开源 |
| 平台许可证 | 商业（$） | 免费 | 商业（$） |
| 自动客户端配置 | 手动 | 手动 | 自动检测 7 种客户端 |
| npm 分发 | 不支持 | 不支持 | 支持 |
| 多文件支持 | 不支持 | 不支持 | 支持（标签页切换） |
| 类型系统 | 基础 | 中等 | 完整（结构体/枚举/联合体/签名） |
| MCP 传输 | stdio | stdio | HTTP + stdio 桥接 |

Binary Ninja MCP 的主要优势在于**自动化配置能力**（检测已安装客户端自动配置）和 **npm 分发**（无需管理 Python 虚拟环境），适合希望快速上手、减少配置摩擦的用户。工具数量虽然少于 GhidraMCP 和 IDA Pro MCP，但覆盖了逆向工程的核心工作流。

---

## 八、常见问题

### Q1：插件安装后 MCP 客户端没有自动配置？

A：检查 MCP 客户端是否**先于插件安装**。如果安装顺序反了，可以重新安装插件触发自动检测，或使用命令行工具手动配置：

```bash
python scripts/mcp_client_installer.py --install
```

### Q2：npm 模式下连接失败？

A：确认 Binary Ninja 中 MCP 服务器已启动（`Plugins > MCP Server > Start MCP Server`）。默认端口为 9009，可在启动时确认日志输出。

### Q3：Claude Desktop 报 "Could not find module"？

A：手动安装模式下需确保虚拟环境中已安装依赖：

```bash
cd /path/to/binary_ninja_mcp
python3 -m venv .venv
source .venv/bin/activate       # macOS/Linux
.venv\Scripts\activate          # Windows
pip install -r bridge/requirements.txt
```

### Q4：能否在无图形界面的服务器上使用？

A：当前版本依赖 Binary Ninja GUI 启动 MCP 服务器。无头（headless）模式暂不支持，可关注项目后续更新。

---

## 参考资源

- [GitHub: fosdickio/binary_ninja_mcp](https://github.com/fosdickio/binary_ninja_mcp)
- [Binary Ninja 官方文档](https://docs.binary.ninja/)
- [MCP 协议规范](https://modelcontextprotocol.io/)
- [DeepWiki: Binary Ninja MCP Installation Guide](https://deepwiki.com/fosdickio/binary_ninja_mcp/2-installation-and-setup)
