---
title: "Copy Fail（CVE-2026-31431）：潜伏 9 年的 Linux 内核提权漏洞全拆解"
published: 2026-07-19
description: "深度剖析 AI 辅助发现的 Linux 内核史诗级提权漏洞，一个 732 字节 Python 脚本即可通杀 2017 年以来几乎所有发行版，从页缓存、splice、AF_ALG 三子系统交汇点完整还原攻击链路。"
tags: ["Linux", "CVE", "内核", "提权", "漏洞分析", "Copy Fail"]
category: 安全
draft: false
image: "./cover.jpeg"
---

## 前言

2026 年 4 月 29 日，韩国 Theori 安全研究团队公开了一个代号 **"Copy Fail"** 的漏洞，编号 **CVE-2026-31431**（CVSS 7.8）。CISA 在 48 小时内将其纳入已知被利用漏洞目录（KEV），Microsoft Defender 也紧急发布了专项分析报告。

这个漏洞有几个让人不安的特点：

- **潜伏 9 年**：漏洞源于 2017 年的一次"无害"性能优化，此后所有主流 Linux 内核均受影响
- **732 字节 Python 脚本即可提权**：无竞争条件、无需硬编码偏移、跨发行版通用
- **磁盘无痕**：只篡改页缓存（Page Cache），磁盘文件丝毫不变，`sha256sum` 检测正常但系统实际已执行恶意代码
- **容器逃逸**：由于页缓存是宿主机级别共享的，容器内的攻击可突破隔离污染宿主机 setuid 二进制文件

更值得关注的是：这个漏洞由 **AI 辅助代码审计工具 Xint Code** 发现，标志着 AI 驱动的漏洞挖掘正式进入实战阶段。

---

## 一、漏洞速览

| 维度 | 详情 |
|------|------|
| CVE 编号 | CVE-2026-31431 |
| 漏洞名称 | Copy Fail |
| 漏洞类型 | 逻辑缺陷（Logic Flaw） |
| CVSS 评分 | 7.8（High） |
| 影响组件 | Linux 内核 AF_ALG / authencesn / splice |
| 引入时间 | 2017 年（commit `72548b093ee3`） |
| 披露时间 | 2026 年 4 月 29 日 |
| 披露方 | Theori（Xint Code / Taeyang Lee） |
| 攻击向量 | 本地低权限用户 → root |
| 漏洞发现方式 | AI 辅助代码审计 |

---

## 二、影响范围

### 2.1 受影响内核版本

| 内核版本 | 状态 |
|----------|------|
| Linux 4.14 ～ 6.18.21 | 受影响 |
| Linux 6.19.0 ～ 6.19.11 | 受影响 |
| Linux 6.18.22+ / 6.19.12+ / 7.0+ | 已修复 |

### 2.2 受影响发行版

几乎全军覆没：

| 发行版 | 受影响版本 |
|--------|-----------|
| Ubuntu | 16.04 / 18.04 / 20.04 / 22.04 / 24.04 LTS |
| RHEL | 8 / 9 / 10 |
| Debian | 9 / 10 / 11 / 12 |
| SUSE | SLES 15, openSUSE Leap/Tumbleweed |
| Amazon Linux | 2 / 2023 |
| Alpine Linux | 全部（容器基础镜像，影响巨大） |

---

## 三、根因分析：三个"无害"特性的致命交汇

Copy Fail 不是传统的堆溢出或 UAF。它是一个**逻辑缺陷**，恰好位于三个原本各自合理的内核子系统的交汇处。

### 3.1 三个"原料"

| 子系统 | 作用 | 为何存在 |
|--------|------|----------|
| **页缓存（Page Cache）** | 内核将磁盘文件的副本缓存在内存中，全系统共享 | 性能优化：避免每次读文件都访问磁盘 |
| **splice()** | 零拷贝系统调用，在文件描述符间传递页面引用而非拷贝数据 | 高性能 I/O：Web 服务器等场景 |
| **AF_ALG** | 用户态加密 API，通过套接字接口暴露内核加密引擎 | 硬件加密加速、密钥不离开内核态 |

三者各自都合理。但组合在一起时，产生了一个危险的后果：**普通用户可以将只读文件的页缓存页面作为可写缓冲区交给加密子系统**。

### 3.2 关键 Bug：2017 年的"原地优化"

2017 年，内核提交 `72548b093ee3` 为 AEAD（认证加密）操作引入了一项优化：

> 当源数据与目标数据位于同一内存区域时，跳过内存拷贝，直接在原地进行解密再加密。

问题在于：当 `splice()` 将页缓存页面零拷贝传入 AF_ALG 套接字后，内核的 `algif_aead` 模块将该页面链接到了**目标散列表（DST SGL）的可写链表**中。

### 3.3 authencesn 的"草稿纸"：seqno_lo

`authencesn` 是 IPsec ESP（封装安全载荷）扩展序列号的认证加密模板，组合了 `hmac(sha256)` 和 `cbc(aes)`。

处理 AEAD 解密时，`authencesn` 需要将 4 字节的 `seqno_lo`（序列号低 32 位）写入输出缓冲的 `assoclen + cryptlen` 偏移处——这相当于在处理输出时用输出缓冲做了草稿纸。

正常情况下这 4 字节写入的是独立的内核缓冲。但在优化路径中，**这 4 字节直接写入了链接到 DST SGL 的页缓存页面**。

### 3.4 攻击数据流

```
sendmsg(AAD seqno_lo)          splice(目标文件，如 /usr/bin/su)
      │                              │
      ▼                              ▼
┌──────────────┐   sg_chain   ┌──────────────────────┐
│   RX 缓冲区   │──────────────▶│   页缓存页面          │
│   (攻击者可控) │              │   (只读文件的内存副本)  │
└──────────────┘              └──────────────────────┘
                                        ▲
                                        │
                     authencesn 在此写入 seqno_lo
                     偏移 = assoclen + cryptlen
                     ═══════ 漏洞触发点 ═══════
```

完整利用步骤：

1. **创建加密套接字**：`socket(AF_ALG, ...)` 创建 AEAD 套接字，绑定 `authencesn(hmac(sha256),cbc(aes))`
2. **注入页缓存**：`splice()` 将目标文件（如 `/usr/bin/su`）的页缓存页面零拷贝传入套接字
3. **构造 AAD**：`sendmsg()` + `MSG_MORE` 发送关联认证数据，嵌入攻击者完全可控的 4 字节 shellcode
4. **触发写入**：内核处理 AEAD 解密，将 `seqno_lo` 写入 DST SGL → 落入页缓存页面
5. **HMAC 失败但已生效**：HMAC 校验失败返回 `EBADMSG`，但 **页缓存已被污染**
6. **执行污染文件**：下次执行被污染的 setuid 二进制时，加载的是内存中被篡改的版本

---

## 四、为什么这么危险

### 4.1 磁盘无痕

页缓存被篡改后，内核**不会将脏页写回磁盘**（因为没有经过正常的写路径标记 dirty flag）。结果：

- 磁盘文件 `sha256sum` 完全正常
- 文件完整性监控（如 AIDE、Tripwire）检测不到异常
- 但 `execve()` 加载的是内存中被污染的版本 —— 攻击者的代码以 root 身份执行

### 4.2 容器逃逸

页缓存是**宿主机级别共享**的。这意味着：

- 容器 A 污染了 `/usr/bin/su` 的页缓存
- 容器 B、宿主机上任何执行 `/usr/bin/su` 的进程都会加载被污染的版本
- 在 K8s 多租户集群中，一个租户可以破坏整台宿主机的安全

### 4.3 无需竞争条件

传统内核提权漏洞（如 Dirty COW 2016），需要精巧的竞态条件窗口，单次成功概率可能不足 10%。但 Copy Fail 是**确定性利用**——不需要碰运气，同一脚本在所有受影响发行版上稳定复现。

---

## 五、受影响的子系统关系图

```
                    ┌─────────────────────┐
                    │   无权限用户进程      │
                    └──────┬──────┬───────┘
               socket(AF_ALG)  │  splice(target_file)
                    ┌──────────▼───▼──────────┐
                    │     内核态               │
                    │                          │
  ┌─────────────────▼────────┐  ┌──────────────▼───────────┐
  │  AF_ALG 加密套接字        │  │  页缓存 (Page Cache)      │
  │  algif_aead 模块         │  │  /usr/bin/su 的内存副本    │
  │  - bind(authencesn)      │  │  - 全系统共享              │
  │  - DST SGL 可写链表       │  │  - 脏页不写回磁盘          │
  └────────┬─────────────────┘  └────────────▲──────────────┘
           │                                 │
           │   sg_chain: 页缓存页面链接到      │
           │   DST SGL 的可写链表              │
           └─────────────────────────────────┘
                            │
                    authencesn 写入 seqno_lo
                    篡改页缓存中的 setuid 二进制
                            │
                            ▼
                    ┌───────────────┐
                    │  execve(su)   │
                    │  → root shell │
                    └───────────────┘
```

---

## 六、修复与缓解

### 6.1 内核升级（唯一根本方案）

```bash
# Ubuntu / Debian
sudo apt update && sudo apt upgrade 'linux-*' && sudo reboot

# RHEL / Rocky / AlmaLinux
sudo dnf --refresh update 'kernel*' && sudo reboot
```

升级后验证：

```bash
uname -r
# 确保版本 >= 6.18.22 或 >= 6.19.12 或 >= 7.0
```

### 6.2 模块黑名单（加固）

`CONFIG_CRYPTO_USER_API_AEAD` 在许多发行版中直接编译进内核（`=y`），无法通过 `rmmod` 禁用。但可以通过内核启动参数限制命名空间：

```bash
# 禁用非特权用户命名空间（会影响容器运行）
echo "kernel.unprivileged_userns_clone=0" >> /etc/sysctl.conf
```

### 6.3 检测思路

由于磁盘文件不受影响，传统文件完整性检测失效。可行方式：

- **页缓存对比**：用 `direct I/O`（绕过页缓存）读取文件并与页缓存版本做 diff
- **审计 splice 调用**：监控非特权进程使用 `splice()` 系统调用向 `AF_ALG` 套接字传输数据
- **定期清理页缓存**：`echo 3 > /proc/sys/vm/drop_caches`（仅作临时缓解）

---

## 七、启示：AI 驱动的漏洞发现时代已来

Copy Fail 的发现方式本身就是一个里程碑。研究员 **Taeyang Lee** 使用 Theori 的自研 AI 工具 **Xint Code** 完成了代码审计。这意味着：

1. **AI 不再是辅助，而是主力**：Xint Code 能够在千万行内核代码中追踪数据流，识别出 `splice → AF_ALG → authencesn` 这条跨越多子系统的攻击链路
2. **潜伏 9 年的 Bug 被 AI 数小时内定位**：传统人工审计可能需要数月乃至数年才有概率触及这个交汇点
3. **攻击面视角转变**：AI 倾向于发现**跨子系统组合产生的逻辑漏洞**，而非传统的缓冲区溢出——这正是现代漏洞挖掘的方向

就在 Copy Fail 披露后不到两周（2026 年 5 月中），Linux 内核又连续爆出 Dirty Frag、DirtyDecrypt、Fragnesia 等 **同一类页缓存污染类提权漏洞**。15 天内 8 个 root 提权漏洞集中曝光，频率从"年"加速到"天"。这并不是内核质量突然变差，而是 AI 辅助分析正在系统性地揭露过去十年积累的隐蔽缺陷。

而对于防守方，Linux 内核加密子系统维护者 Eric Biggers 的评价一针见血：

> "AF_ALG ... **不应该存在**。它将巨大的内核攻击面暴露给非特权用户空间程序，而且几乎完全没有必要。用户态加密库已经能满足几乎所有需求。"

---

## 参考资源

- [Theori / Xint Code 官方技术报告](https://xint.io/blog/copy-fail-linux-distributions)
- [copy.fail — 研究团队披露网站 & PoC](https://copy.fail/)
- [CVE-2026-31431 NVD 页面](https://nvd.nist.gov/vuln/detail/CVE-2026-31431)
- [Microsoft Defender 专项分析报告](https://www.microsoft.com/en-us/security/blog/2026/05/01/cve-2026-31431-copy-fail-vulnerability-enables-linux-root-privilege-escalation/)
- [CISA KEV 收录公告](https://www.cisa.gov/news-events/alerts/2026/05/01/cisa-adds-one-known-exploited-vulnerability-catalog)
- [Rocky Linux Advisory](https://rockylinux.org/news/2026-05-11-copyfail-cve-2026-31431)
