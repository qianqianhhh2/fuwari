---
title: Unity IL2CPP 逆向分析完全指南
published: 2026-07-17
description: "从 Il2CppDumper 到 Ghidra，深入理解 IL2CPP 的编译原理与逆向工具链"
tags: ["Unity", "IL2CPP", "逆向工程", "Ghidra", "安全"]
category: 技术
draft: false
image: "./cover.jpeg"
---

# Unity IL2CPP 逆向分析完全指南

## 前言

在 Unity 游戏开发中，IL2CPP 已成为跨平台发布的事实标准。超过 80% 的商业手游项目采用 IL2CPP 作为脚本后端。然而，IL2CPP 将 C# 编译为原生机器码的特性，使得传统的 .NET 反编译工具完全失效。本文将系统梳理 IL2CPP 的编译原理、逆向工具链和实战流程。

## IL2CPP 是什么？

IL2CPP（Intermediate Language To C++）是 Unity 提供的 **AOT（Ahead-of-Time）编译方案**。它的编译流程如下：

```
C# 源码
  ↓ (C# 编译器)
IL 中间语言 (Assembly-CSharp.dll)
  ↓ (il2cpp.exe)
C++ 代码
  ↓ (平台 C++ 编译器)
原生机器码 (GameAssembly.dll / libil2cpp.so / UnityFramework)
```

与传统的 Mono 后端不同，IL2CPP 在构建过程中将 C# 代码经过 IL 转换为 C++，再编译为原生机器指令。这意味着：
- `GameAssembly.dll` 中存储的是 CPU 指令，而非 .NET IL
- 原生的 C# 类结构、方法名和字段信息已丢失
- 类型信息被单独保存在 `global-metadata.dat` 中

## IL2CPP 项目的文件结构

打包后的 IL2CPP 项目典型结构如下：

### Windows
```
GameName/
├── GameName.exe
├── GameAssembly.dll            # 机器码逻辑
└── GameName_Data/
    └── il2cpp_data/
        └── Metadata/
            └── global-metadata.dat  # 类型信息
```

### Android
```
base.apk/
├── lib/armeabi-v7a/libil2cpp.so   # 机器码逻辑
└── assets/bin/Data/Managed/Metadata/
    └── global-metadata.dat        # 类型信息
```

### iOS
```
GameName.ipa/
├── Frameworks/UnityFramework.framework/UnityFramework  # 机器码逻辑
└── Data/Managed/Metadata/
    └── global-metadata.dat
```

核心文件只有两个：

| 文件 | 作用 |
|------|------|
| `GameAssembly.dll` / `libil2cpp.so` | 编译后的原生机器码，包含所有游戏逻辑 |
| `global-metadata.dat` | 类型元数据，包含类名、方法名、字段、参数等 |

两者分离存储是 IL2CPP 比 Mono 更难逆向的根本原因。

## 逆向工具链

### 必备工具

| 工具 | 用途 | 获取方式 |
|------|------|----------|
| **Il2CppDumper** | 解析元数据，重建符号映射 | [GitHub](https://github.com/Perfare/Il2CppDumper) |
| **Ghidra** | 反汇编与反编译 | [NSA 开源](https://ghidra-sre.org/) |
| **dnSpy** | 查看 Dummy DLL 的类结构 | [GitHub](https://github.com/dnSpy/dnSpy) |

### 辅助工具

- **AssetStudio** — 提取和预览游戏资源（贴图、音效等）
- **UnityPy** — Python 资源文件提取
- **Cheat Engine** — 运行时内存扫描与调试
- **Il2CppInspector** — 替代 Il2CppDumper，支持更多混淆方案

## 实战流程

### 第一步：获取关键文件

以 Android 为例：

```bash
# 导出 APK 路径
adb shell pm path com.example.game
# 拉取 APK
adb pull /data/app/~~[hash]==/base.apk
# 解压获取目标文件
# lib/armeabi-v7a/libil2cpp.so
# assets/bin/Data/Managed/Metadata/global-metadata.dat
```

### 第二步：Il2CppDumper 解析

```bash
# 基本用法
Il2CppDumper.exe libil2cpp.so global-metadata.dat output/
```

**输出文件说明：**

| 文件 | 内容 |
|------|------|
| `dump.cs` | 所有类、方法、字段的 C# 定义 |
| `script.json` | Ghidra/IDA 脚本所需的符号映射 |
| `il2cpp.h` | C++ 结构体定义 |
| `DummyDll/Assembly-CSharp.dll` | 可供 dnSpy 查看的伪 DLL |

示例输出（`dump.cs` 片段）：

```csharp
// Namespace: 
public class PlayerController : MonoBehaviour // TypeDefIndex: 1234
{
    // Fields
    public int hp; // 0x10
    private float moveSpeed; // 0x14
    // ...
    
    // Methods
    [Address(RVA = "0x32B490", Offset = "0x329E90", VA = "0x18032B490")]
    public void TakeDamage(float amount) { }
}
```

### 第三步：Ghidra 静态分析

Il2CppDumper 生成的 `script.json` 可以通过脚本导入 Ghidra，自动：

- 重命名函数为原始 C# 方法名
- 添加方法签名注释
- 标记字段和属性偏移
- 恢复字符串引用

```bash
# 在 Ghidra 中
Window → Script Manager → 运行 Ghidra script → 选择生成的 script.py
```

### 第四步：分析代码逻辑

以 `TakeDamage` 方法为例，导入脚本后在 Ghidra 中可以看到：

```c
void PlayerController_TakeDamage(float amount)
{
    if (this->hp <= 0) {
        return;
    }
    this->hp = this->hp - (int)amount;
    if (this->hp <= 0) {
        this->hp = 0;
        PlayerController_Die(this);
    }
}
```

## IL2CPP 编译管线深度解析

理解 IL2CPP 的完整编译管线有助于我们在二进制中找到关键逻辑：

### 1. C# → IL

C# 编译器将源码编译为 .NET 中间语言（IL），保留完整的类结构、方法签名和字段信息。这也是为什么 Mono 后端的 `Assembly-CSharp.dll` 可以直接用 dnSpy 反编译出接近源码的代码。

### 2. IL → C++

`il2cpp.exe` 将 IL 转换为 C++ 代码。这一步会：
- 将类结构转换为 C++ 的 struct/class
- 将虚方法调用转换为直接地址跳转
- 生成类型注册代码和元数据

### 3. C++ → 原生机器码

平台编译器（MSVC / Clang / GCC）将 C++ 编译为机器码。这一步会：
- 剥离所有调试符号（Release 构建）
- 内联小函数
- 优化循环和分支

### 方法命名规则

IL2CPP 生成的 C++ 函数有特定的命名规律：

```
Namespace_Class_Method  // 普通方法
Namespace_Class_Method_m1234567890  // 重载方法（带参数哈希）
```

了解这些命名规律有助于即使在未完全符号化的情况下也能定位关键函数。

## 常见的混淆与保护

商业游戏通常会对 IL2CPP 输出做额外处理：

| 技术 | 描述 | 典型案例 |
|------|------|----------|
| **元数据加密** | 对 `global-metadata.dat` 进行 XOR/ROT 加密 | Arknights, CoD Mobile |
| **符号混淆** | 替换方法名为无意义字符串 | Beebyte 混淆器 |
| **字符加密** | 加密字符串字面量 | League of Legends: Wild Rift |
| **元数据重排** | 打乱注册元数据顺序 | Riot Games 系列 |
| **自定义加密** | 厂商自研加密方案 | miHoYo (原神/崩坏3) |

对于加密的元数据，需要在运行时从内存中 dump 解密后的数据，或使用 Il2CppInspector 等支持反混淆的工具。

## 动态分析：Frida Hook

静态分析只能看到代码结构，动态分析能验证我们的理解：

```javascript
// Frida Hook 示例 — 修改玩家血量
Interceptor.attach(Module.findExportByName("libil2cpp.so", "PlayerController_TakeDamage"), {
    onEnter: function(args) {
        console.log("TakeDamage called, amount:", args[1]);
        // 修改伤害为 0，实现上帝模式
        args[1] = 0;
    }
});
```

常见动态分析技巧：

- **Hook 关键函数** — 记录参数、修改返回值
- **内存扫描** — 用 Cheat Engine 搜索特定值（如血条数值）
- **字符串搜索** — 搜索日志字符串定位函数地址
- **交叉验证** — 静态找到的函数 + 动态断点验证

## 总结

IL2CPP 逆向的核心思路是：**分离的代码与元数据 → 重新关联 → 符号恢复 → 静态分析 + 动态验证**。

完整工作流：

```
GameAssembly.dll + global-metadata.dat
       ↓
   Il2CppDumper → dump.cs / script.json / DummyDll
       ↓
   dnSpy → 浏览类结构和方法签名
       ↓
   Ghidra / IDA → 导入 script.json → 分析函数实现
       ↓
   Frida / Cheat Engine → 动态验证和 Hook
```

> 声明：本文仅用于技术研究与学习目的。未经授权的商业软件逆向分析可能违反相关法律法规，请确保在合法授权范围内进行研究。
