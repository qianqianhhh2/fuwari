---
title: JADX 接入 MCP 工具完整教程：让 AI 驱动 Android 逆向分析
published: 2026-07-20
description: "手把手教你将 JADX-GUI 接入 MCP（Model Context Protocol），通过 Claude、Cursor、Trae 等 AI 客户端实现自动化反编译、Manifest 分析、交叉引用追踪、去混淆重命名等 Android 逆向工程任务。"
tags: ["JADX", "MCP", "Android 逆向", "人工智能", "安全分析"]
category: 教程
draft: false
image: "./cover.webp"
---

# JADX 接入 MCP 工具完整教程：让 AI 驱动 Android 逆向分析

## 前言

MCP（Model Context Protocol，模型上下文协议）是一种标准化协议，用于将 AI 大语言模型（LLM）与外部工具连接起来。在 Android 逆向工程领域，**JADX-AI-MCP** 是最热门的 MCP 实现，它由 [zinja-coder](https://github.com/zinja-coder/jadx-ai-mcp) 开发（2.5k+ stars），由两部分组成：

| 组件 | 语言 | 作用 |
|------|------|------|
| JADX-AI-MCP 插件 | Java | 安装在 JADX-GUI 内，启动 HTTP 服务（默认 8650 端口），暴露当前反编译工程的所有数据 |
| JADX-MCP-Server | Python | 独立进程，通过 FastMCP 将 HTTP 接口翻译为标准 MCP 工具，供 LLM 客户端调用 |

借助 JADX-AI-MCP，你可以用自然语言让 AI 完成以下工作：

- 获取任意类的反编译源码和 Smali 代码
- 自动搜索类/方法/关键词（支持大 APK 分页）
- 读取 AndroidManifest.xml、strings.xml 和所有资源文件
- 追踪交叉引用——谁调用了谁、谁访问了这个字段
- AI 辅助去混淆重命名（类、方法、字段、变量、包）
- 自动 SAST 漏洞扫描（硬编码密钥、不安全 API、WebView 注入等）
- 调试时查看堆栈和变量
- 批量分析并生成安全审计报告

本文将从零开始，带你完成 JADX-AI-MCP 的安装、配置与实战。

---

## 一、前置环境要求

### 1.1 版本硬性要求

| 组件 | 最低版本 | 推荐版本 |
|------|----------|----------|
| JADX | 1.5.1+ (r2333+) | 1.5.3+（带 JRE 的 GUI 版） |
| Java | 11+ | 17+（JDK 17 LTS） |
| Python | 3.10+ | 3.12+ |
| MCP 客户端 | 任意支持 MCP 的工具 | Claude Desktop / Cursor / Trae |

> **系统兼容性**：Windows（完全支持）、macOS（完全支持）、Linux（完全支持）。Docker、WSL 和远程虚拟机也可通过 `--host` 参数绑定。

### 1.2 环境预检查

在开始安装前，请确认以下条件：

- JADX-GUI 已下载并可以打开 APK 文件
- Python 版本符合要求：`python --version`
- Java 版本符合要求：`java -version`
- UV 包管理器（推荐）：`uv --version`

---

## 二、正式安装流程

### 步骤 1：下载 JADX GUI

到 [JADX Releases](https://github.com/skylot/jadx/releases) 下载对应平台版本。

**Windows 用户推荐**：下载 `jadx-gui-xxx-with-jre-win.zip`（内置 JRE，解压即用，无需单独安装 Java）。

```
D:\tools\jadx\
  ├─ jadx-gui.exe
  ├─ runtime\  (内置 JRE)
  └─ ...
```

双击 `jadx-gui.exe` 能正常打开即可。

### 步骤 2：安装 JADX-AI-MCP 插件

到 [JADX-AI-MCP Releases](https://github.com/zinja-coder/jadx-ai-mcp/releases) 下载最新 `jadx-ai-mcp-x.x.x.jar`。

JADX 插件目录（Windows）：

```
C:\Users\<用户名>\.local\share\jadx\plugins\
```

将下载的 `.jar` 文件放入该目录。目录不存在则手动创建。然后重启 JADX GUI。

**验证**：打开 JADX → `File` → `Preferences` → `Plugins`，应能看到 `JADX-AI-MCP Plugin`。加载 APK 后，JADX 底部状态栏会显示 **"AI MCP Server: Running, Port: 8650"**，说明插件端已就绪。

> **备选方案**（通用版 jadx，有命令行入口）：
> ```bash
> jadx plugins --install "github:zinja-coder:jadx-ai-mcp"
> ```

### 步骤 3：安装 UV 并配置 MCP Server

**安装 UV（推荐）：**

Windows PowerShell：
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

macOS / Linux：
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**下载并配置 MCP Server：**

1. 从 [Releases](https://github.com/zinja-coder/jadx-ai-mcp/releases) 下载 `jadx-mcp-server.zip`
2. 解压到本地目录（如 `D:\tools\jadx-mcp-server\`）
3. 创建虚拟环境并安装依赖：

```powershell
cd D:\tools\jadx-mcp-server
uv venv --python 3.10
.\.venv\Scripts\activate
uv pip install httpx fastmcp
```

macOS / Linux：
```bash
cd ~/tools/jadx-mcp-server
uv venv
source .venv/bin/activate
uv pip install httpx fastmcp
```

4. 测试 MCP Server：

```bash
uv run jadx_mcp_server.py
```

看到 `MCP Server listening on stdio...` 即表示服务端正常。

> **备用方案（系统 Python + pip）：**
> ```bash
> python -m venv .venv
> .\.venv\Scripts\activate  # Windows
> source .venv/bin/activate  # macOS/Linux
> pip install httpx fastmcp
> python jadx_mcp_server.py
> ```

### 步骤 4：配置 AI 客户端

#### Claude Desktop

编辑配置文件：

- **Windows**：`%APPDATA%\Claude\claude_desktop_config.json`
- **macOS**：`~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "jadx-mcp-server": {
      "command": "D:\\tools\\jadx-mcp-server\\.venv\\Scripts\\uv.exe",
      "args": [
        "--directory",
        "D:\\tools\\jadx-mcp-server",
        "run",
        "jadx_mcp_server.py"
      ]
    }
  }
}
```

> **注意**：配置完成后，必须从系统托盘完全退出 Claude Desktop（不只是关闭窗口），然后重新启动。

#### Cursor

打开 Cursor → `Settings` → `MCP Tools` → `Add a Custom MCP Server`，编辑 `mcp.json`：

```json
{
  "mcpServers": {
    "jadx-mcp-server": {
      "command": "D:\\tools\\jadx-mcp-server\\.venv\\Scripts\\uv.exe",
      "args": [
        "--directory",
        "D:\\tools\\jadx-mcp-server",
        "run",
        "jadx_mcp_server.py"
      ]
    }
  }
}
```

保存后确认 `jadx-mcp-server` 显示为 `Enabled` 状态。

#### Trae

Trae → `设置` → `MCP` → `添加 MCP 服务器`：

```json
{
  "mcpServers": {
    "jadx-mcp-server": {
      "command": "uv",
      "args": [
        "--directory",
        "D:\\tools\\jadx-mcp-server",
        "run",
        "jadx_mcp_server.py"
      ]
    }
  }
}
```

### 步骤 5：验证安装

⚠ **正确的启动顺序（顺序反了连不上）**：

1. 先启动 **JADX GUI**
2. 在 JADX 中**加载目标 APK**，等左侧类树加载完成
3. JADX 底部状态栏显示 **"AI MCP Server: Running, Port: 8650"**
4. 再启动 LLM 客户端
5. 在客户端中确认 MCP 工具列表出现 `mcp__jadx-mcp-server__*` 系列工具

在客户端发送一条测试指令：
> 用 `get_android_manifest` 获取当前 APK 的主 Activity 包名。

如果能正确返回 Manifest 信息，说明整个链路已连通。

---

## 三、核心功能详解

JADX-AI-MCP 提供了 **29+** 个工具，覆盖了 Android 逆向的各个环节。以下是常用工具的分类介绍：

### 3.1 类分析类

| 工具 | 功能 | 参数 |
|------|------|------|
| `get_all_classes` | 列出所有反编译的类名 | 无 |
| `fetch_current_class` | 获取当前 JADX 中选中的类的完整源码 | 无 |
| `get_class_source` | 获取指定类的完整源码 | `class_name` |
| `get_methods_of_class` | 列出类的所有方法 | `class_name` |
| `get_fields_of_class` | 列出类的所有字段 | `class_name` |
| `get_method_by_name` | 按名称获取方法的源码 | `class_name`, `method_name` |
| `get_smali_of_class` | 获取类的 Smali 代码 | `class_name` |
| `get_selected_text` | 获取当前 JADX 中选中的文本 | 无 |
| `get_main_activity_class` | 从 Manifest 解析主 Activity | 无 |
| `get_main_application_classes_code` | 获取包名下所有应用类的源码 | 无 |
| `get_main_application_classes_names` | 获取包名下所有应用类的类名 | 无 |

### 3.2 搜索类

| 工具 | 功能 | 参数 |
|------|------|------|
| `search_method_by_name` | 全局搜索方法名 | `method_name` |
| `search_classes_by_keyword` | 按关键词搜索类（支持分页） | `keyword`, `offset`, `count` |

### 3.3 资源类

| 工具 | 功能 | 参数 |
|------|------|------|
| `get_android_manifest` | 获取 AndroidManifest.xml 内容 | 无 |
| `get_manifest_component` | 获取 Manifest 指定组件信息 | `component` |
| `get_strings` | 获取 strings.xml | 无 |
| `get_all_resource_file_names` | 列出所有资源文件名 | 无 |
| `get_resource_file` | 获取指定资源文件内容 | `file_name` |

### 3.4 交叉引用类

| 工具 | 功能 | 参数 |
|------|------|------|
| `xrefs_to_class` | 查找类的所有引用类和方法 | `class_name`（支持分页） |
| `xrefs_to_method` | 查找方法的所有调用点（含重写） | `class_name`, `method_name`（支持分页） |
| `xrefs_to_field` | 查找字段的所有访问点 | `class_name`, `field_name`（支持分页） |

### 3.5 重构类

| 工具 | 功能 | 参数 |
|------|------|------|
| `rename_class` | 重命名类 | `class_name`, `new_name` |
| `rename_method` | 重命名方法 | `class_name`, `method_name`, `new_name` |
| `rename_field` | 重命名字段 | `class_name`, `field_name`, `new_name` |
| `rename_package` | 重命名整个包 | `package_name`, `new_name` |
| `rename_variable` | 重命名方法内局部变量 | `class_name`, `method_name`, `variable_name`, `new_name` |

### 3.6 调试类

| 工具 | 功能 |
|------|------|
| `debug_get_stack_frames` | 获取当前调试堆栈帧 |
| `debug_get_threads` | 获取当前调试线程信息 |
| `debug_get_variables` | 获取当前调试上下文变量值 |

---

## 四、实战示例：用 AI 逆向分析 APK

下面通过一个完整的 APK 安全审计案例，演示如何使用 JADX-AI-MCP。

### 4.1 准备工作

1. 在 JADX-GUI 中打开目标 APK 文件
2. 等待反编译完成（左侧类树加载完毕，底部状态栏显示 `AI MCP Server: Running`）
3. 在 MCP 客户端（如 Trae）中确认连接状态

### 4.2 编写分析 Prompt

一个好的 Prompt 能极大提升 AI 的分析效率和准确度。以下是一个经过验证的安全审计 Prompt 模板：

```text
你连接了 JADX MCP Server，可以对当前打开的 APK 进行自动化安全审计。

分析策略（按顺序执行，分层推进）：

1. Manifest 摸底：
   - 用 get_android_manifest 拉取 Manifest
   - 列出所有 exported=true 的组件（Activity/Service/Receiver/Provider）
   - 标注它们的权限保护情况

2. 硬编码密钥检测：
   - 用 get_strings 获取所有字符串资源
   - 用 search_classes_by_keyword 搜索 "key", "secret", "token", "password", "AES", "DES"
   - 标记出所有疑似硬编码的敏感信息

3. WebView 安全检查：
   - 用 search_method_by_name 搜索 "addJavascriptInterface"
   - 对每个匹配结果，用 get_class_source 获取完整源码
   - 检查 setJavaScriptEnabled 和 setAllowFileAccess 的配置

4. 加密算法审计：
   - 搜索 "Cipher", "MessageDigest", "Mac" 等加密相关类
   - 检查是否使用了不安全的算法（DES, RC4, ECB 模式）
   - 检查 IV 是否固定值或全零

5. 数据存储安全：
   - 搜索 "SharedPreferences", "openFileOutput", "SQLiteDatabase"
   - 检查是否有明文存储密码或敏感数据的风险

6. 生成审计报告：
   - 按严重程度（高危/中危/低危）分类
   - 每个问题包含：类名、代码位置、风险描述、修复建议

⚠ 重要规则：
- 每一步的结果都整理好再进入下一步
- 大 APK 使用 search_classes_by_keyword 的 offset/count 分页
- 可疑发现用 xrefs_to_method 追踪调用链确认影响范围
- 所有高危发现必须标注完整的类名和方法名
```

### 4.3 分析对话示例

以下是 AI 使用 MCP 工具审计 APK 的典型交互过程：

**AI**: `get_android_manifest()` → 获取 Manifest，发现 3 个 exported Activity

**AI**: `get_strings()` → 在 strings.xml 中发现 `api_key = "sk-xxxx"` 硬编码

**AI**: `search_classes_by_keyword("addJavascriptInterface")` → 找到 `WebViewBridge` 类

**AI**: `get_class_source("com.example.WebViewBridge")` → 发现 `setJavaScriptEnabled(true)` 未做域名白名单限制

**AI**: `xrefs_to_method("com.example.crypto.AESUtil", "encrypt")` → 追踪加密调用链，发现 ECB 模式 + 固定 IV

**AI**: 生成结构化审计报告，标注 5 个高危风险、3 个中危风险

---

## 五、Prompt 编写技巧

虽然 LLM 功能强大，但提示词仍然非常关键。以下是一些实践技巧：

1. **指明分析目标**：是 Manifest 安全审计、硬编码密钥检测、加密算法分析，还是混淆代码还原
2. **利用交叉引用**：别说"找这个函数在哪用了"，直接让 AI 调 `xrefs_to_method`
3. **分层推进，不要跳跃**：先摸底 Manifest → 定位入口类 → 关键字搜索 → 代码深度审计 → 交叉引用追踪 → 混淆还原
4. **分页处理大 APK**：类超过 10000 个时，提示 AI 使用 `offset` / `count` 参数
5. **用自然语言描述技术需求**："搜索所有包含 `addJavascriptInterface` 的方法，检查 JS 桥接是否存在注入风险"
6. **利用 Smali 做深度分析**：当反编译的 Java 代码不准确时，可以用 `get_smali_of_class` 查看字节码级别的实现

---

## 六、远程连接配置（WSL / Docker / 跨设备）

如果你在远端服务器运行 JADX，但 AI 客户端在另一台设备上，需要进行远程连接配置。

### 6.1 修改 MCP Server 监听地址

默认情况下 MCP Server 通过 stdio 与客户端通信，如需远程访问，使用 `--host` 参数：

```bash
uv run jadx_mcp_server.py --host 0.0.0.0 --port 9000
```

> ⚠ **安全提醒**：`--host 0.0.0.0` 会暴露未加密的 HTTP 流量到整个网络。仅在可信网络中使用，或通过 SSH 隧道转发。

### 6.2 指定 JADX 插件地址

如果 JADX 插件运行在非本机，通过 `--jadx-host` 和 `--jadx-port` 指定：

```bash
uv run jadx_mcp_server.py --jadx-host 192.168.1.100 --jadx-port 8650
```

### 6.3 Docker 部署

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY jadx-mcp-server/ .
RUN pip install httpx fastmcp
CMD ["python", "jadx_mcp_server.py", "--host", "0.0.0.0", "--jadx-host", "host.docker.internal"]
```

---

## 七、常见问题排查

### 7.1 MCP 工具列表为空 / `get_android_manifest` 返回空

**原因**：JADX 还没加载 APK 就发了指令。MCP 读取的是 JADX 当前会话状态。

**解决**：确保 JADX 先打开 APK，等类树加载完成，再启动客户端发指令。

### 7.2 插件状态栏没有显示 "AI MCP Server: Running"

- 检查 jar 文件是否在 `C:\Users\<用户名>\.local\share\jadx\plugins\` 目录下
- 在 JADX 中打开 `File` → `Preferences` → `Plugins`，确认 `JADX-AI-MCP Plugin` 已启用
- 重启 JADX GUI

### 7.3 `get_smali_of_class` 返回空

**原因**：JADX 反编译模式未开启 Smali 输出。

**解决**：`File` → `Preferences` → 勾选 `Java + Smali` 双输出模式。

### 7.4 Claude Desktop 连不上 MCP Server

- 确认已从系统托盘**完全退出** Claude Desktop（不是关闭窗口）
- 检查配置文件 JSON 格式是否正确（注意 Windows 路径用双反斜杠 `\\` 或单斜杠 `/`）
- 确认 MCP Server 能单独启动：手动执行 `uv run jadx_mcp_server.py` 看是否有报错

### 7.5 大 APK 分析超时

**原因**：类数量过多（1 万+），一次性返回所有数据会超时。

**解决**：提示 AI 使用 `search_classes_by_keyword` 的 `offset` / `count` 分页参数，每次只取一部分。

---

## 八、架构原理简述

```
┌─────────────────────────────────────────┐
│  LLM 客户端 (Claude / Cursor / Trae)     │
│  - 通过 stdio 与 MCP Server 通信         │
└──────────────┬──────────────────────────┘
               │ MCP 协议 (stdio)
               ▼
┌─────────────────────────────────────────┐
│  JADX MCP Server (Python)               │
│  - FastMCP 暴露 29+ 工具                 │
│  - 内部将 MCP 调用 → HTTP 请求           │
└──────────────┬──────────────────────────┘
               │ HTTP (127.0.0.1:8650)
               ▼
┌─────────────────────────────────────────┐
│  JADX-AI-MCP 插件 (Java)                │
│  - 运行在 JADX-GUI 进程内                │
│  - Javalin HTTP 服务 → JADX API 调用     │
└──────────────┬──────────────────────────┘
               │ JADX API
               ▼
┌─────────────────────────────────────────┐
│  JADX-GUI 当前打开的项目                 │
│  - 反编译后的类/方法/资源/Manifest       │
└─────────────────────────────────────────┘
```

两条独立的连接：
- `--host` / `--port`：MCP Server 监听 LLM 客户端的位置
- `--jadx-host` / `--jadx-port`：MCP Server 查找 JADX 插件的位置

本地单机使用时默认即可，无需修改。

---

## 九、与传统的 APK 逆向方案对比

| 方案 | 特点 | 适用场景 |
|------|------|----------|
| JADX 手动分析 | 在几千个类里人肉翻找，复制代码贴给 AI，效率低 | 小规模快速查看 |
| MobSF / QARK 等 SAST | 规则固定、误报率高，缺乏上下文理解 | 自动化门禁、CI/CD |
| JADX Python 脚本 | 灵活但需熟悉 JADX API，无法语义分析 | 定制化批量处理 |
| **JADX-AI-MCP + LLM** | 对话式逆向，AI 主动查数据，语义理解强 | 深度逆向、漏洞审计、去混淆 |

JADX-AI-MCP 属于 [Zin MCP Suite](https://github.com/zinja-coder) 的一部分，该套件还包括 APKTool-MCP-Server 等。未来整个 Android 逆向工具链都可以通过同一套 MCP 协议串联起来。

---

## 十、最佳实践总结

1. **启动顺序是关键**：先 JADX 开 APK，等反编译完成，再启动 AI 客户端
2. **写好 Prompt 是成功的一半**：分层推进，每一步明确要调用的工具和期望的输出
3. **善用交叉引用**：别只看一个类，用 `xrefs_to_method` 追踪完整调用链
4. **Smali 做深度兜底**：Java 反编译不准时，切到 Smali 级别分析
5. **大 APK 用分页**：1 万+ 类时明确告知 AI 使用 `offset` / `count`
6. **重命名保持规范**：去混淆时给 AI 一个命名规范（如 `m` 前缀成员变量、`s` 前缀静态变量）
7. **审计报告结构化**：要求 AI 按高危/中危/低危分类输出，每个问题附类名、行号和修复建议

---

## 参考资源

- [zinja-coder/jadx-ai-mcp GitHub 仓库](https://github.com/zinja-coder/jadx-ai-mcp)
- [JADX-AI-MCP 官方文档](https://jadx-ai-mcp.readthedocs.io/en/latest/)
- [MCP 协议官方文档](https://modelcontextprotocol.io/)
- [JADX MCP 原理与使用部署（CSDN）](https://blog.csdn.net/samlirongsheng/article/details/159548293)
- [Claude + JADX + MCP 逆向分析实战（CSDN）](https://blog.csdn.net/qq_45797625/article/details/154964176)
- [AI 辅助安卓逆向：TRAE+JADX-AI-MCP 插件实战（掘金）](https://juejin.cn/post/7601843004957900846)
- [JADX-MCP 逆向 APK 实战教程（腾讯云）](https://cloud.tencent.cn/developer/article/2701762)
