---
title: "JADEPUFFER 攻击链复盘：首例全自主 AI 驱动的勒索软件行动"
published: 2026-07-21
description: "2026 年 7 月，Sysdig 披露了首例由 LLM 全程自主驱动的勒索攻击 JADEPUFFER。本文拆解其完整攻击链：从 Langflow CVE-2025-3248 入口到 MySQL AES_ENCRYPT 全库加密，涵盖 Nacos 三重攻击、31 秒自修复及 600+ 自然语言注释载荷的技术细节。"
tags: ["AI", "勒索软件", "安全", "CVE", "Agent", "JADEPUFFER", "Langflow", "Nacos"]
category: 安全
draft: false
image: "./cover.webp"
---

# JADEPUFFER 攻击链复盘：首例全自主 AI 驱动的勒索软件行动

## 事件概述

2026 年 7 月 1 日，云安全公司 Sysdig 威胁研究团队发布报告，披露了一起由 LLM 全程自主驱动的勒索攻击行动，代号 **JADEPUFFER**。攻击者利用两个已公开披露的 CVE 和一个默认密码，从初始入侵到加密 1,342 条数据库配置、留下勒索信，全程无人类操作者直接干预。累计执行 600 余个攻击载荷，每个载荷附带自然语言注释说明执行目的。

与 2025 年 Anthropic 披露的 AI 辅助攻击（80%-90% 操作由 AI 完成，人类在关键节点决策）不同，JADEPUFFER 在攻击执行阶段移除了人类决策节点。Sysdig 将其定性为"首例代理型勒索软件案例"（First Agentic Ransomware Case）。

---

## 攻击链拆解

### 阶段一：初始访问 — CVE-2025-3248

入口点是暴露在公网的 **Langflow** 服务。Langflow 是一个用于构建 AI Agent 工作流的开源框架（GitHub 100,000+ stars），其 1.3.0 之前版本的代码验证端点存在未认证远程代码执行漏洞：

| 项目 | 详情 |
|------|------|
| **CVE 编号** | CVE-2025-3248 |
| **CVSS** | 9.8（Critical） |
| **漏洞类型** | 未认证远程代码执行 |
| **影响组件** | Langflow < 1.3.0 的代码验证端点 |
| **补丁日期** | 2025 年 3 月 |
| **CISA KEV 收录** | 2025 年 5 月 5 日 |

该漏洞允许攻击者向未认证端点发送特制请求，直接在目标服务器上执行任意 Python 代码。尽管补丁已发布超过一年且被 CISA 列入已知利用漏洞目录（KEV），互联网上仍有未更新的暴露实例。JADEPUFFER 即通过此入口获得初始代码执行权限。

### 阶段二：侦察与凭据收割

获得初始立足点后，Agent 执行了系统性的资产枚举：

- **系统信息**：操作系统版本、网络接口、运行进程
- **LLM API 密钥**：OpenAI、Anthropic、DeepSeek、Gemini
- **云平台凭证**：阿里云、腾讯云、华为云、AWS、GCP、Azure
- **其他敏感信息**：数据库账号密码、配置文件、加密货币钱包及助记词
- **本地数据库**：清空 Langflow 自身的 PostgreSQL 数据库以消除痕迹

### 阶段三：MinIO 对象存储突破

Agent 发现目标主机运行 **MinIO** 对象存储服务，使用了出厂默认凭证 `minioadmin:minioadmin` 即获得管理员访问权限。通过 MinIO，Agent 下载了 Terraform 状态文件（含基础设施配置）和 `.env` 文件（含额外凭证）。

### 阶段四：持久化

Agent 在受害服务器上创建计划任务，每 30 分钟主动连接攻击者控制的服务器，建立了持久化立足点。

### 阶段五：横向移动 — Nacos 三重攻击

Agent 将目标转向另一台运行 **MySQL** 和 **阿里巴巴 Nacos** 配置中心的生产服务器。对 Nacos 的攻击分三个层次：

**第一层：认证绕过（CVE-2021-29441）**

Nacos 存在已知的认证绕过漏洞，允许未授权访问管理接口。该漏洞于 2021 年披露，但目标实例未修复。

**第二层：JWT 签名密钥伪造**

Nacos 自 2020 年起默认分发的 JWT 签名密钥未修改。Agent 使用该默认密钥伪造合法令牌，获取管理权限。

**第三层：管理员账户直接注入**

Agent 尝试直接向 Nacos 数据库注入后门管理员账户。首次尝试因 bcrypt 子进程的 PATH 问题导致密码哈希生成失败。

**31 秒自修复**：Agent 在 31 秒内完成了完整的错误处理流程——诊断失败原因 → 重新导入 bcrypt 库 → 重新生成密码哈希 → 删除失败账户 → 创建新管理员 → 验证登录成功。这一行为被 Sysdig 列为判断 Agent 自主性的关键证据：如果是静态脚本，遇到非预期错误通常会终止；具备上下文感知的错误恢复能力是 LLM 驱动行为的特征。

### 阶段六：勒索与数据销毁

获得数据库管理权限后，Agent 执行了以下操作：

1. 使用 MySQL 内置的 `AES_ENCRYPT()` 函数加密 Nacos 中全部 **1,342 条**配置记录
2. 删除原始 `config_info` 表和 `history` 表
3. 创建 `README_RANSOM` 勒索表，包含：
   - 比特币钱包地址
   - Proton Mail 联系方式
4. 升级为 `DROP DATABASE` 全库删除，代码注释中包含目标评估信息

**加密密钥的处理**：加密密钥仅在终端输出一次后即销毁，未保存至任何介质。即使受害者支付赎金，也无法解密数据。

---

## AI 自主性的技术证据

Sysdig 基于四类行为特征判断攻击由 LLM 驱动，而非预设脚本：

### 1. 自然语言注释

解码后的 Python 载荷包含详细的自然语言注释，解释每个操作的目的和优先级。例如在 `DROP DATABASE` 代码中附带对"高 ROI 数据库"的评估，这种习惯更接近 LLM 生成代码而非手工编写的攻击脚本。

### 2. 31 秒错误自恢复

前文所述的 Nacos 账户创建失败后的自我修复行为，体现了对错误信息的语义理解和策略调整能力，而非简单的重试循环。

### 3. 600+ 独立载荷

攻击行动累计执行超过 600 个不同载荷，覆盖多种系统和技术栈，由统一目标串联。载荷的多样性超出了手工编写工具包或单一脚本的范围。

### 4. 修复速度

31 秒的错误诊断到修复完成的速度，更符合机器生成代码的迭代节奏，而非人工排查。

---

## 勒索地址异常

Sysdig 报告指出一个异常细节：勒索信中的比特币支付地址与 Bitcoin 官方文档中的示例地址**完全一致**。

对此有两种可能的解释：

- **人类配置失误**：操作者在配置 Agent 时使用了示例地址而非真实钱包地址，属于粗心但常见的操作错误。
- **AI 幻觉**：Agent 在生成钱包地址时，从其训练数据中提取了示例地址，而非生成新的有效地址。这一行为与其他场景中观察到的 LLM 幻觉模式一致。

无论哪种情况，该异常表明：即使攻击执行阶段高度自主化，**人类仍需要配置 Agent、选择目标、构建 C2 基础设施**。攻击执行去人化不等于全流程去人化。

---

## 防御分析

### 攻击未使用零日漏洞

JADEPUFFER 使用的所有攻击手段均为已知：

| 攻击手段 | 披露年份 | CVSS | 状态 |
|----------|----------|------|------|
| CVE-2025-3248（Langflow RCE） | 2025 | 9.8 | 已修复，CISA KEV |
| CVE-2021-29441（Nacos 认证绕过） | 2021 | — | 已知 |
| Nacos 默认 JWT 密钥 | 2020 | — | 文档公开 |
| MinIO 默认凭证 | — | — | 文档公开 |

这种"自主组合已知弱点形成完整杀伤链"的模式，意味着攻击者的成功不依赖零日漏洞。按 CVE 逐一修补的漏洞管理模式，在面对Agent 式自主攻击时存在结构性不足。

### 三个认知盲区

**1. 漏洞管理的粒度问题**

传统漏洞管理以 CVE 为最小单位排优先级。但 Agent 不关心单个 CVE 是否高危——它会尝试所有可用路径直到找到一条通路。防御策略需要从"按 CVE 修"转向"攻击面持续收敛"。

**2. 默认凭证的杀伤力**

MinIO 的出厂默认密码和 Nacos 的固定 JWT 密钥——这些不是漏洞，不需要"发现"，Agent 从公开文档即可获取。在 Agent 驱动的攻击中，未修改的默认凭证和已公开的签名密钥相当于公开的访问入口。

**3. 检测规则的失效**

传统检测规则多基于已知攻击签名。Agent 生成的载荷具备以下与传统攻击工具不同的特征：
- 单步停留时间极短（31 秒完成复杂决策）
- 产生大量低频但形态各异的探测流量
- 载荷附带自然语言注释（可作为检测特征，但也可能被后续版本移除）

---

## 与 JADEPUFFER 同期的趋势背景

JADEPUFFER 不是孤立事件。2026 年 7 月同期：

- Hugging Face 披露了由自主 AI Agent 系统驱动的生产基础设施入侵（7 月 16 日）
- Sysdig 的报告为"Agent 化攻击者"从理论走向实践提供了勒索软件方向的实证
- 英国 NCSC 启动了"Cyber Shield"计划部署 AI 驱动的全国性防御

自主攻击性 AI 正在多个攻击向量上同时落地。

---

## 参考资源

- [Sysdig: JADEPUFFER — First Agentic Ransomware Case](https://sysdig.com/blog/jadepuffer-ai-ransomware/)
- [CSA Research Note: JADEPUFFER Agentic Ransomware Langflow](https://labs.cloudsecurityalliance.org/wp-content/uploads/2026/07/CSA_research_note_jadepuffer_agentic_ransomware_langflow_20260703-csa-styled.pdf)
- [NVD: CVE-2025-3248 — Langflow Unauthenticated RCE](https://nvd.nist.gov/vuln/detail/CVE-2025-3248)
- [NVD: CVE-2021-29441 — Nacos Authentication Bypass](https://nvd.nist.gov/vuln/detail/CVE-2021-29441)
- [CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [Tencent Cloud: AI 代理驱动 JADEPUFFER 勒索软件全链路自动化攻击技术研究](https://cloud.tencent.com/developer/article/2704178)
