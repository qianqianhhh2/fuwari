---
title: IDA Pro 接入 MCP 工具完整教程：让 AI 驱动逆向工程
published: 2026-07-18
description: "手把手教你将 IDA Pro 接入 MCP（Model Context Protocol），通过 Cursor、Claude、Trae 等 AI 客户端实现自动化反编译、函数分析、交叉引用追踪等逆向工程任务。"
tags: ["IDA Pro", "MCP", "逆向工程", "人工智能", "安全分析"]
category: 教程
draft: false
image: "./cover.jpeg"
---

# IDA Pro 接入 MCP 工具完整教程：让 AI 驱动逆向工程

## 前言

MCP（Model Context Protocol，模型上下文协议）是一种标准化协议，用于将 AI 大语言模型（LLM）与外部工具连接起来。在逆向工程领域，**ida-pro-mcp** 是最热门的 MCP 实现，它由 [mrexodia](https://github.com/mrexodia/ida-pro-mcp)（x64dbg 的作者）开发，能让 AI 助手直接读写 IDA Pro 的数据库（`.idb`/`.i64`），实现"对话式逆向"（vibe reversing）。

借助 ida-pro-mcp，你可以用自然语言让 AI 完成以下工作：

- 自动反编译函数并分析逻辑
- 重命名函数和变量（带类型推断）
- 追踪交叉引用（xrefs）
- 搜索字符串、立即数和字节序列
- 设置注释和修改函数原型
- 批量分析导出报告
- 识别混淆和加密算法
- 迅速且精确的静态分析速度

本文将从零开始，带你完成 IDA Pro MCP 的安装、配置与实战。

---

## 一、前置环境要求

### 1.1 版本硬性要求

| 组件 | 最低版本 | 推荐版本 |
|------|----------|----------|
| IDA Pro | 8.3+（Professional） | 9.0+ |
| Python | 3.11+ | 3.12+ |
| MCP 客户端 | 任意支持 MCP 的工具 | Cursor / Claude Desktop / Trae |

> ⚠️ **注意**：IDA Free 免费版不支持该插件。Python 版本需与 IDA 内置 Python 一致，可使用 `idapyswitch` 工具切换。

### 1.2 环境预检查

在开始安装前，请确认以下条件：

- IDA Pro 已正常安装并可以打开二进制文件
- Python 版本符合要求：`python --version`
- Git 已安装（用于从 GitHub 拉取依赖）：`git --version`
- 网络连接正常（pip 需要下载依赖包）

---

## 二、正式安装流程

### 步骤 1：安装 ida-pro-mcp 包

打开终端，使用 pip 从 GitHub 直接安装最新版：

```bash
# 如果有旧版本，先卸载
pip uninstall ida-pro-mcp -y

# 安装最新版本
pip install git+https://github.com/mrexodia/ida-pro-mcp.git
```

如果国内网络访问 GitHub 较慢，可以使用镜像加速：

```bash
pip install git+https://gitcode.com/gh_mirrors/id/ida-pro-mcp.git
```

### 步骤 2：安装 IDA 插件并配置客户端

执行安装命令，该命令会自动完成三件事：
1. 将插件安装到 IDA Pro 的 `plugins` 目录
2. 配置常见的 MCP 客户端（Cursor、Claude Desktop、Windsurf 等）
3. 生成 MCP 配置文件

```bash
ida-pro-mcp --install
```

安装成功后，终端会输出类似以下信息：

```
✅ Installed plugin to C:\Users\xxx\AppData\Roaming\Hex-Rays\IDA Pro\plugins\mcp-plugin.py
✅ Configured Cursor MCP
✅ Configured Claude Desktop MCP
```

### 步骤 3：验证安装

1. **完全退出** IDA Pro 和 MCP 客户端（注意 Claude 可能在系统托盘中后台运行）
2. 重新启动 IDA Pro，打开任意二进制文件
3. 检查 IDA 顶部菜单栏是否出现 **`IDA MCP`** 菜单项

如果菜单出现，说明插件已成功加载。

### 步骤 4：手动补充客户端配置（自动写入失败时）

如果自动配置未生效，可以手动获取配置：

```bash
ida-pro-mcp --config
```

然后将输出的 JSON 配置复制到对应客户端的 MCP 配置文件中：

- **Cursor**：`~/.cursor/mcp.json`
- **Claude Desktop**：`~/Library/Application Support/Claude/claude_desktop_config.json`
- **Trae**：设置 → MCP → 添加服务器
- **VS Code**：`.vscode/mcp.json`

一个典型的 MCP 配置如下：

```json
{
  "mcpServers": {
    "ida-pro": {
      "command": "python",
      "args": ["-m", "ida_pro_mcp.server"],
      "env": {}
    }
  }
}
```

---

## 三、核心功能详解

ida-pro-mcp 提供了 **30+** 个工具，覆盖了逆向工程的各个环节。以下是常用工具的分类介绍：

### 3.1 基础查询类

| 工具 | 功能 | 参数 |
|------|------|------|
| `check_connection` | 检查 IDA 插件连接状态 | 无 |
| `get_metadata` | 获取当前 IDB 元数据（文件名、架构、基址等） | 无 |
| `get_entry_points` | 获取程序入口点列表 | 无 |

### 3.2 函数分析类

| 工具 | 功能 | 参数 |
|------|------|------|
| `get_function_by_name` | 按名称查找函数 | `name` |
| `get_function_by_address` | 按地址查找函数 | `address` |
| `decompile_function` | 反编译函数（Hex-Rays 伪代码） | `address` |
| `disassemble_function` | 获取函数汇编代码 | `start_address` |
| `list_functions` | 分页列出所有函数 | `offset`, `count` |

### 3.3 数据查询类

| 工具 | 功能 | 参数 |
|------|------|------|
| `list_strings` | 分页列出所有字符串 | `offset`, `count` |
| `list_globals` | 分页列出全局变量 | `offset`, `count` |
| `list_local_types` | 列出所有本地类型定义 | 无 |
| `get_xrefs_to` | 获取指定地址的交叉引用 | `address` |
| `get_xrefs_to_field` | 获取结构体字段的交叉引用 | `struct_name`, `field_name` |

### 3.4 修改类

| 工具 | 功能 | 参数 |
|------|------|------|
| `rename_function` | 重命名函数 | `function_address`, `new_name` |
| `rename_local_variable` | 重命名局部变量 | `function_address`, `old_name`, `new_name` |
| `rename_global_variable` | 重命名全局变量 | `old_name`, `new_name` |
| `set_comment` | 设置地址注释 | `address`, `comment` |
| `set_function_prototype` | 设置函数原型 | `function_address`, `prototype` |
| `set_local_variable_type` | 设置局部变量类型 | `function_address`, `variable_name`, `new_type` |
| `declare_c_type` | 声明 C 类型（结构体/枚举） | `c_declaration` |

### 3.5 调试类（需加 `--unsafe` 标志）

| 工具 | 功能 |
|------|------|
| `dbg_start_process` | 启动调试器 |
| `dbg_set_breakpoint` | 设置断点 |
| `dbg_continue_process` | 继续执行 |
| `dbg_get_registers` | 获取寄存器值 |
| `dbg_get_call_stack` | 获取调用栈 |

---

## 四、实战示例：用 AI 分析 CrackMe

下面通过一个完整的 CrackMe 分析案例，演示如何使用 ida-pro-mcp。

### 4.1 准备工作

1. 在 IDA Pro 中打开目标 CrackMe 二进制文件
2. 等待初始自动分析完成
3. 在 MCP 客户端（如 Cursor）中确认连接状态

### 4.2 编写分析 Prompt

LLM 处理数字进制转换容易产生幻觉，因此编写 Prompt 时需要特别强调使用工具而非手动转换。以下是一个经过验证的 Prompt 模板：

```text
你的任务是分析 IDA Pro 中的一个 CrackMe 程序。

分析策略：
1. 先 get_metadata 获取基本信息，get_entry_points 获取入口点
2. 定位 main 函数（通过入口点追踪或搜索字符串定位）
3. 使用 decompile_function 获取函数伪代码
4. 分析伪代码逻辑，识别关键比较和条件跳转
5. 使用 rename_function 和 rename_local_variable 重命名函数和变量
6. 使用 set_comment 为关键地址添加注释
7. 使用 set_function_prototype 修正函数原型
8. 使用 set_local_variable_type 修正变量类型（尤其是指针和数组类型）

⚠️ 重要规则：
- 绝不手动转换数字进制，始终使用 convert_number 工具
- 在读取分析结果前，确保 IDA 自动分析已完成
- 找到密码/flag 后，明确告知用户
```

### 4.3 分析对话示例

以下是 AI 使用 MCP 工具分析 CrackMe 的典型交互过程：

**AI**: `get_metadata` → 获取到 x64 PE 文件信息，基址 0x140000000

**AI**: `get_entry_points` → 发现入口点 `start` 在 0x140001000

**AI**: `get_function_by_name("main")` → 定位到 main 函数在 0x140001200

**AI**: `decompile_function(0x140001200)` → 获取反编译伪代码，发现 `strcmp` 调用

**AI**: `rename_local_variable(0x140001200, "v5", "user_input")` → 重命名变量

**AI**: `set_comment(0x140001250, "密码验证：将用户输入与硬编码密钥比较")` → 添加注释

**AI**: 最终分析出密码为 `P@ssw0rd_2024!`

---

## 五、常用客户端配置详解

### 5.1 Cursor

安装 ida-pro-mcp 后，Cursor 会自动配置。手动配置方式：

在 `~/.cursor/mcp.json` 中添加：

```json
{
  "mcpServers": {
    "ida-pro": {
      "command": "python",
      "args": ["-m", "ida_pro_mcp.server"]
    }
  }
}
```

然后在 Cursor 的 Composer 面板（<kbd>Ctrl+I</kbd>）中，选择 Agent 模式即可使用。

### 5.2 Claude Desktop

Claude Desktop 配置在安装时自动完成。如需手动配置，编辑对应平台的配置文件：

- **Windows**：`%APPDATA%\Claude\claude_desktop_config.json`
- **macOS**：`~/Library/Application Support/Claude/claude_desktop_config.json`

配置格式与 Cursor 一致。配置完成后**必须从系统托盘完全退出** Claude Desktop 再重新启动。

### 5.3 Trae / VS Code

在 Trae 中：
1. 打开设置 → MCP → 添加 MCP 服务器
2. 输入名称 `ida-pro`
3. 命令选择 `python`，参数为 `-m ida_pro_mcp.server`

在 VS Code 中使用 Agent Mode（GitHub Copilot），配置方式与 Cursor 相同。

### 5.4 Windsurf

Windsurf 的配置文件位于 `~/.codeium/windsurf/mcp.json`，内容与其他客户端一致。

---

## 六、远程连接配置（WSL2 / 跨设备）

如果你在 Windows 上运行 IDA Pro，但 AI 客户端在 WSL2 或其他设备上，需要进行远程连接配置。

### 6.1 修改插件监听地址

默认情况下，IDA MCP 插件只监听 `127.0.0.1:13337`，需要改为 `0.0.0.0` 以允许外部连接。

**方法 A：修改插件源文件（永久生效，推荐）**

编辑 IDA plugins 目录下的 `mcp-plugin.py`，找到绑定地址的代码行：

```python
# 将 127.0.0.1 改为 0.0.0.0
server = await asyncio.start_server(handle_client, "0.0.0.0", 13337)
```

**方法 B：通过 IDA MCP 菜单设置**

在 IDA Pro 的 `IDA MCP` 菜单中修改监听地址并保存。

### 6.2 防火墙配置

确保端口 13337 允许入站连接：

```powershell
# 以管理员身份运行
New-NetFirewallRule -DisplayName "IDA MCP" -Direction Inbound -Port 13337 -Protocol TCP -Action Allow
```

### 6.3 客户端配置

客户端 MCP 配置中将地址改为服务器的实际 IP：

```json
{
  "mcpServers": {
    "ida-pro": {
      "command": "python",
      "args": ["-m", "ida_pro_mcp.server", "--host", "192.168.1.100"]
    }
  }
}
```

---

## 七、常见问题排查

### 7.1 IDA 顶部没有 "IDA MCP" 菜单

- 确认已加载二进制文件（空项目不会显示菜单）
- 检查 IDA Python 版本：打开 IDA 的 `Output` 窗口查看 Python 版本号
- 查看 `Output` 窗口中是否有插件加载报错信息

### 7.2 连接失败：Connection Refused（端口 13337）

- 确认 IDA Pro 已打开并加载了二进制文件
- 确认防火墙未阻止 13337 端口
- 确认 `mcp-plugin.py` 文件在正确的 plugins 目录中

### 7.3 反编译失败

- 确认 IDA Pro 安装了 Hex-Rays 反编译器（x86/x64/ARM 授权）
- 检查 `decompile_function` 传入的地址是否在一个有效的函数范围内

### 7.4 Python 版本不匹配

- 运行 `idapyswitch` 查看 IDA 当前使用的 Python 版本路径
- 确保安装 ida-pro-mcp 的 Python 环境版本与之一致

---

## 八、其他 MCP 实现方案对比

除了 mrexodia 的 `ida-pro-mcp`，社区还有多个优秀的替代方案：

| 项目 | 特点 | 适用场景 |
|------|------|----------|
| [mrexodia/ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp) | 功能最全面，社区最活跃 | 通用逆向分析 |
| [fdrechsler/mcp-server-idapro](https://github.com/fdrechsler/mcp-server-idapro) | TypeScript 实现，HTTP 桥接 | 需要 Node.js 环境的项目 |
| [jtsylve/ida-mcp](https://github.com/jtsylve/ida-mcp) | 基于 idalib 的无头模式 | 自动化批量分析、CI/CD 集成 |
| [taida957789/ida-mcp-server-plugin](https://github.com/taida957789/ida-mcp-server-plugin) | 轻量级，SSE 协议 | IDA 9.0+ 的简约方案 |
| [cybermaxluo/IDAProMCP_Max](https://github.com/cybermaxluo/IDAProMCP_Max) | 增强版，混淆检测/算法识别/报告生成 | 恶意软件分析、深度逆向 |

---

## 九、最佳实践总结

1. **写好 Prompt 是成功的一半**：明确告诉 AI 你的分析目标（如"找出序列号验证逻辑"），并强调使用 `convert_number` 工具而非手动转换数字
2. **迭代式分析**：先概览（`get_metadata` + `get_entry_points`），再聚焦关键函数，最后深入细节
3. **善用注释**：让 AI 通过 `set_comment` 记录分析发现，方便后续人工复查
4. **变量重命名保持一致性**：给 AI 一个命名规范（如 `g_` 前缀代表全局变量，`s_` 前缀代表字符串）
5. **调试功能谨慎使用**：调试类工具需要 `--unsafe` 标志，仅在可控环境中启用
6. **注意安全**：不要让 AI 在你的主机上运行别人提供的 IDA Python 脚本，避免代码注入风险

---

## 参考资源

- [mrexodia/ida-pro-mcp GitHub 仓库](https://github.com/mrexodia/ida-pro-mcp)
- [MCP 协议官方文档](https://modelcontextprotocol.io/)
- [mcp-reversing-dataset（练习数据集）](https://github.com/mrexodia/mcp-reversing-dataset)
- [IDA Pro MCP 视频演示](https://www.youtube.com/results?search_query=Automated+AI+Reverse+Engineering+with+MCP+for+IDA)
