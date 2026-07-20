---
title: Android APK 反编译与 Smali 修改入门
published: 2026-07-18
description: "从零开始学习使用 apktool 和 jadx 反编译 APK，理解 Smali 语法，修改代码并重新打包签名"
tags: ["Android", "逆向工程", "Smali", "apktool", "安全"]
category: 教程
draft: false
image: "./cover.jpeg"
---

# Android APK 反编译与 Smali 修改入门

## 前言

在日常开发或安全研究中，我们有时需要分析一个 APK 的内部逻辑——比如想看看某个 App 的网络请求是怎么加密的、某个功能是怎么实现的，或者想给自己写的 App 做一些自动化修改。这时候，反编译就是必不可少的技能。

本教程将从零开始，带你走完 **APK 反编译 → Smali 修改 → 重新打包签名** 的完整流程。

> ⚠️ **免责声明**：本文仅用于学习和安全研究目的。请勿将反编译技术用于侵犯他人知识产权、破解付费功能等违法行为。

---

## 一、APK 内部结构速览

APK（Android Package Kit）本质上是一个 ZIP 压缩包。直接改后缀名为 `.zip` 解压，你会看到：

```
app.apk
├── AndroidManifest.xml    # 二进制格式的清单文件（权限、组件声明）
├── classes.dex            # Dalvik 字节码（核心代码）
├── classes2.dex           # 多 DEX 时的扩展
├── res/                   # 资源文件（布局、图片、字符串等）
├── lib/                   # native .so 库（arm64-v8a, armeabi-v7a 等）
├── assets/                # 原始资源文件
├── META-INF/              # 签名信息
└── resources.arsc         # 编译后的资源索引表
```

其中 **`classes.dex`** 就是我们的核心目标——它包含了所有 Java/Kotlin 代码编译后的 Dalvik 字节码。而 **Smali** 就是 Dalvik 字节码的文本表示形式，类似于汇编语言。

---

## 二、工具链介绍

| 工具 | 用途 | 下载 |
|---|---|---|
| **apktool** | 反编译/重打包 APK，输出 Smali 代码和可编辑资源 | [ibotpeaches.github.io/Apktool](https://ibotpeaches.github.io/Apktool/) |
| **jadx** | 将 DEX 反编译为可读的 Java 源码（GUI + CLI） | [github.com/skylot/jadx](https://github.com/skylot/jadx) |
| **keytool** | 生成签名密钥（JDK 自带） | 随 JDK 安装 |
| **apksigner / jarsigner** | 对 APK 进行签名 | apksigner 随 Android SDK，jarsigner 随 JDK |
| **zipalign** | APK 对齐优化（Android SDK Build Tools） | 随 Android SDK |

### 工具配合策略

- **jadx-gui**：第一步使用，用来**快速阅读 Java 源码**，定位关键函数和逻辑
- **apktool**：第二步使用，**反编译出 Smali + 资源**，进行修改和重打包
- **apksigner**：最后一步，给修改后的 APK **重新签名**

---

## 三、环境准备

### 3.1 安装 JDK

apktool 和签名工具都依赖 Java。前往 [Adoptium](https://adoptium.net/) 或 Oracle 下载安装 JDK 17+。

安装后验证：

```bash
java -version
```

### 3.2 安装 apktool

**Windows**：

1. 从 [GitHub Releases](https://github.com/iBotPeaches/Apktool/releases) 下载 `apktool_2.x.x.jar`
2. 在 jar 同目录创建 `apktool.bat`，内容如下：

```bat
@echo off
setlocal
set BASENAME=apktool_
chcp 65001 2>nul >nul

setlocal EnableDelayedExpansion
pushd "%~dp0"
if exist apktool.jar (
    set BASENAME=apktool
    goto skipversioned
)
set max=0
for /f "tokens=1* delims=-_.0" %%A in ('dir /b /a-d %BASENAME%*.jar') do if %%~B gtr !max! set max=%%~nB
:skipversioned
popd
setlocal DisableDelayedExpansion

if "%~1"=="" goto load
if not "%~2"=="" goto load
set ATTR=%~a1
if "%ATTR:~0,1%"=="d" set fastCommand=b
if "%ATTR:~0,1%"=="-" if "%~x1"==".apk" set fastCommand=d

:load
java -jar -Duser.language=en -Dfile.encoding=UTF8 "%~dp0%BASENAME%%max%.jar" %fastCommand% %*
```

3. 将 jar 和 bat 所在目录加入系统 PATH

**macOS / Linux**：

```bash
# macOS
brew install apktool

# Ubuntu/Debian
sudo apt install apktool
```

### 3.3 安装 jadx

从 [GitHub Releases](https://github.com/skylot/jadx/releases) 下载 jadx 的 zip 包，解压后双击 `jadx-gui.bat`（Windows）即可启动图形界面。

---

## 四、第一步：用 jadx 分析源码

拿到一个 APK 后，**不要一上来就改 Smali**——先拖进 jadx-gui 读 Java 代码，搞清楚要改什么、在哪里改。

### jadx 搜索技巧

| 目标 | 搜索关键词 |
|---|---|
| 登录逻辑 | `login`、`password`、`token`、`Authorization` |
| 网络请求 | `HttpURLConnection`、`OkHttp`、`Retrofit`、`http://` |
| 加密操作 | `AES`、`MD5`、`Base64`、`Cipher`、`encrypt` |
| 广告 SDK | `AdView`、`BannerAd`、`com.google.android.gms.ads` |
| Root 检测 | `isRooted`、`su`、`Superuser`、`test-keys` |
| 签名校验 | `getPackageInfo`、`signatures`、`Signature` |

jadx 支持**全文搜索**、**类名搜索**、**方法跳转**，用法和 IDE 很接近。

---

## 五、第二步：用 apktool 反编译

确定要修改的位置后，用 apktool 进行反编译：

```bash
apktool d target.apk -o target_src
```

参数说明：
- `d` = decode（解码）
- `-o` = 输出目录

反编译后的目录结构：

```
target_src/
├── AndroidManifest.xml    # 可读的 XML 清单文件
├── apktool.yml            # 构建配置文件
├── smali/                 # 主 dex 的 Smali 代码
├── smali_classes2/        # 次 dex（如果有）
├── res/                   # 资源文件（可直接编辑）
├── assets/                # 原始资源
└── original/              # 原始签名信息（META-INF）
```

**常用参数**：
- `-r`：不解码资源文件（只反编译 Smali，速度更快）
- `-s`：不反编译 dex（只解码资源文件）
- `--only-main-classes`：只反编译主 dex，跳过 classes2+

---

## 六、Smali 语言基础

Smali 是 Dalvik/ART 虚拟机的寄存器语言，语法类似汇编。在动手修改之前，至少要能看懂基本结构。

### 6.1 数据类型描述符

| 描述符 | Java 类型 | 说明 |
|---|---|---|
| `V` | void | 只用于返回值 |
| `Z` | boolean | 注意不是 B |
| `B` | byte | |
| `S` | short | |
| `C` | char | |
| `I` | int | |
| `J` | long | 64 位，占 2 个寄存器 |
| `F` | float | |
| `D` | double | 64 位，占 2 个寄存器 |
| `Lpackage/ClassName;` | 对象 | 如 `Ljava/lang/String;` |
| `[I` | int[] | 数组前加 `[` |

> 注：`J` 代表 long 是因为 `L` 已被对象类型占用。

### 6.2 文件结构

每个 `.smali` 文件对应一个 Java 类。一个典型的 Smali 文件结构：

```smali
.class public Lcom/example/app/MainActivity;
.super Landroid/app/Activity;
.source "MainActivity.java"

# 静态字段
.field private static final TAG:Ljava/lang/String; = "MainActivity"

# 实例字段
.field private count:I

# 方法
.method public onCreate(Landroid/os/Bundle;)V
    .locals 2

    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V

    const-string v0, "Hello World"
    return-void
.end method
```

- `.class`：类声明（类名用 `/` 分隔）
- `.super`：父类
- `.field`：成员变量
- `.method` ... `.end method`：方法定义

### 6.3 寄存器命名

Smali 使用两套命名方式：

- **v 命名法**：`v0`、`v1`、`v2`……（局部变量）
- **p 命名法**：`p0`、`p1`……（方法参数），其中非静态方法里 `p0` = `this`

实际代码中两套命名可能混用。`.locals N` 声明方法内局部变量的总数（不含参数）。

### 6.4 常用指令速查

| 指令 | 含义 |
|---|---|
| `const/4 v0, 0x1` | v0 = 1 |
| `const-string v0, "hello"` | v0 = "hello" |
| `move v0, v1` | v0 = v1 |
| `if-eqz v0, :label` | 如果 v0 == 0 则跳转 |
| `if-nez v0, :label` | 如果 v0 != 0 则跳转 |
| `if-eq v0, v1, :label` | 如果 v0 == v1 则跳转 |
| `invoke-virtual {p0, v0}, ...` | 调用虚方法 |
| `invoke-static {v0}, ...` | 调用静态方法 |
| `return-void` | 返回 void |
| `return v0` | 返回 v0 |
| `return-object v0` | 返回对象 |
| `add-int/lit8 v0, v1, 0x1` | v0 = v1 + 1 |
| `sget-object v0, ...` | 读取静态字段 |
| `sput-object v0, ...` | 写入静态字段 |
| `iget-object v0, p0, ...` | 读取实例字段 |
| `iput-object v0, p0, ...` | 写入实例字段 |

---

## 七、实战修改示例

### 7.1 修改字符串内容

**场景**：将首页标题从 "Hello" 改为 "你好，世界"。

**1. 在 jadx 中定位**

搜索 `"Hello"`，找到对应方法：

```java
public String getTitle() {
    return "Hello";
}
```

**2. 找到对应的 Smali**

在 `target_src/smali/` 下搜索 `"Hello"`，找到对应的方法：

```smali
.method public getTitle()Ljava/lang/String;
    .locals 1

    const-string v0, "Hello"

    return-object v0
.end method
```

**3. 修改**

```smali
.method public getTitle()Ljava/lang/String;
    .locals 1

    const-string v0, "\u4f60\u597d\uff0c\u4e16\u754c"  # "你好，世界" 的 Unicode

    return-object v0
.end method
```

> 非 ASCII 字符需要用 Unicode 转义。可以使用在线工具转换，或者直接保留 UTF-8 编码（新版 apktool 支持）。

### 7.2 绕过 Root 检测

**场景**：App 检测到设备已 Root 就拒绝运行。

**原始 Java 代码**：

```java
public static boolean isRooted() {
    // ... 检测逻辑 ...
    return true;  // 检测到 root
}
```

**对应的 Smali**：

```smali
.method public static isRooted()Z
    .locals 1

    # ... 检测逻辑 ...

    const/4 v0, 0x1    # true

    :goto_0
    return v0
.end method
```

**修改为始终返回 false**：

只需将 `const/4 v0, 0x1` 改为 `const/4 v0, 0x0`：

```smali
.method public static isRooted()Z
    .locals 1

    const/4 v0, 0x0    # 始终返回 false

    return v0
.end method
```

> 更彻底的改法：直接在方法开头 `const/4 v0, 0x0` 然后 `return v0`，删掉中间所有检测代码。

### 7.3 跳过广告加载

**场景**：App 在主界面加载广告，想去掉。

**原始 Smali**：

```smali
.method private showAd()V
    .registers 3

    # 加载并展示广告
    invoke-virtual {p0}, Lcom/example/MainActivity;->loadBannerAd()V
    invoke-virtual {p0}, Lcom/example/MainActivity;->loadInterstitialAd()V

    return-void
.end method
```

**修改后**：直接清空方法体，让广告加载函数什么都不做。

```smali
.method private showAd()V
    .registers 1

    return-void
.end method
```

> `.registers 1` 改为 `1` 是因为我们用不到寄存器了，寄存器数量需要与实际使用匹配。

### 7.4 修改条件判断逻辑

**场景**：VIP 功能检测，让所有用户都能使用。

**原始 Smali**：

```smali
    invoke-virtual {p0}, Lcom/example/app/UserInfo;->isVip()Z
    move-result v0

    if-eqz v0, :cond_not_vip   # 如果 v0 == 0（非VIP），跳到限制逻辑
    # ... VIP功能 ...
    return-void

    :cond_not_vip
    # ... 显示"请开通VIP" ...
    return-void
```

**修改方案 1**：反转条件

```smali
    invoke-virtual {p0}, Lcom/example/app/UserInfo;->isVip()Z
    move-result v0

    if-nez v0, :cond_not_vip   # 改为 if-nez，把条件反转
```

**修改方案 2**：直接删除条件判断

```smali
    # 删掉整个 isVip() 调用和 if 判断
    # 直接执行 VIP 功能代码
    # ... VIP功能 ...
    return-void
```

---

## 八、重新打包

修改完成后，用 apktool 重新构建 APK：

```bash
apktool b target_src -o modified.apk
```

参数说明：
- `b` = build（构建）
- `-o` = 输出文件名

如果报错，常见的解决方式：
- **资源编译失败**：尝试 `apktool b target_src -o modified.apk --use-aapt2`
- **framework 不匹配**：导入目标系统的 framework：`apktool if framework-res.apk`
- **多 dex 编译失败**：检查 `apktool.yml` 中的 dex 配置

---

## 九、签名

Android 系统不允许安装未签名的 APK。修改后的 APK 必须重新签名。

### 9.1 生成签名密钥

```bash
keytool -genkey -v -keystore mykey.jks -keyalg RSA -keysize 2048 -validity 10000 -alias myalias
```

执行后会提示输入密码和基本信息（随意填写即可）。

### 9.2 签名 APK

**方式一：使用 apksigner（推荐，支持 v2/v3 签名）**

```bash
apksigner sign --ks mykey.jks --ks-key-alias myalias --out signed.apk modified.apk
```

**方式二：使用 jarsigner（传统方式）**

```bash
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore mykey.jks modified.apk myalias
```

### 9.3 对齐优化（可选）

```bash
zipalign -v 4 signed.apk aligned.apk
```

> 注意：如果使用 apksigner，**先 zipalign 再 sign**；如果使用 jarsigner，**先 sign 再 zipalign**。

---

## 十、安装测试

### 10.1 通过 ADB 安装

```bash
adb install -r signed.apk
```

`-r` 表示覆盖安装（replace）。

### 10.2 常见安装错误

| 错误 | 原因 | 解决 |
|---|---|---|
| `INSTALL_PARSE_FAILED_NO_CERTIFICATES` | 未签名 | 重新签名 |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | 签名不一致 | 先卸载原版：`adb uninstall <package>` |
| `INSTALL_FAILED_VERIFICATION_FAILURE` | 签名校验失败 | 检查签名是否正确 |
| `INSTALL_FAILED_INVALID_APK` | APK 结构损坏 | 检查回编译是否成功 |

### 10.3 查看日志调试

如果修改后的 App 崩溃，通过 logcat 查看错误：

```bash
adb logcat | grep -E "AndroidRuntime|FATAL"
```

---

## 十一、常见陷阱与注意事项

### 11.1 签名校验

很多 App（特别是金融类）会在运行时校验签名，如果发现签名与原版不一致就拒绝运行。这需要在 Smali 中找到签名校验函数并 patch 掉——仅靠 apktool 重打包是不够的，通常需要配合 Frida 等动态 Hook 工具。

### 11.2 加固/加壳

如果 APK 被加固过（如 360 加固、腾讯乐固），`classes.dex` 里只有壳代码，真实的 dex 在运行时才会解密释放。此时需要**先脱壳再分析**。

判断是否加固：
- 用 jadx 打开 APK，如果只有几个无关类（如 `com.stub.StubApp`），大概率是加固了
- `lib/` 下有 `libjiagu.so`、`libtup.so` 等特征 so 文件

### 11.3 Smali 修改容易出错的地方

- **寄存器数量不匹配**：`.locals` 或 `.registers` 的数量必须与实际使用的寄存器一致
- **类型描述符写错**：`Ljava/lang/String;` 后面的分号不能省略
- **跳转标签丢失**：删代码时注意别把 `:goto_xxx` 标签删掉了
- **p 命名和 v 命名的关系**：非静态方法中 `p0` = `this`，增加参数时小心编号偏移

### 11.4 推荐编辑器

- **VS Code + smalise 插件**：语法高亮 + 错误提示，强烈推荐
- **APKLab 插件**（VS Code）：一键反编译、修改、打包、签名，适合快速操作

---

## 十二、完整流程总结

```
原始 APK
    │
    ├──→ jadx-gui 分析 Java 源码（定位关键逻辑）
    │
    ├──→ apktool d 反编译为 Smali + 资源
    │
    ├──→ 修改 Smali 代码 / 资源文件
    │
    ├──→ apktool b 重新打包
    │
    ├──→ apksigner / jarsigner 签名
    │
    ├──→ zipalign 对齐（可选）
    │
    └──→ adb install 安装测试
```

---

## 参考资源

- [apktool 官方文档](https://ibotpeaches.github.io/Apktool/)
- [Smali/Baksmali GitHub](https://github.com/JesusFreke/smali)
- [Dalvik 字节码指令集](https://source.android.com/devices/tech/dalvik/dalvik-bytecode)
- [jadx GitHub](https://github.com/skylot/jadx)
