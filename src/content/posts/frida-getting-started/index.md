---
title: Frida 动态插桩入门实战：从环境搭建到 Hook 脚本编写
published: 2026-07-19
description: "从零掌握 Frida 动态插桩框架，覆盖 Java 层 Hook、Native 层 Interceptor、Python 自动化注入，配合实战案例快速上手 Android/iOS/桌面端逆向分析。"
tags: ["Frida", "逆向工程", "Android", "Hook", "动态插桩", "安全"]
category: 教程
draft: false
image: "./cover.jpeg"
---

# Frida 动态插桩入门实战：从环境搭建到 Hook 脚本编写

## 前言

Frida 是由 Ole André V. Ravnås 开发的跨平台动态插桩工具包（Dynamic Instrumentation Toolkit）。与其在 IDA Pro 或 Ghidra 中进行「静态分析」不同，Frida 让你在**程序运行时**注入 JavaScript 脚本，实时读取内存、拦截函数调用、修改参数和返回值——也就是所谓的「动态插桩」。

Frida 的核心优势：

- **跨平台**：Windows / macOS / Linux / Android / iOS 一套 API 通吃
- **无需重编译**：直接注入运行中的进程，不碰 APK/IPA 分毫
- **JS + Python 双语言**：JavaScript 写 Hook 逻辑，Python 做自动化编排
- **Java 层 + Native 层通吃**：Android 上既能 Hook Java 方法，也能拦截 .so 中的 C/C++ 函数
- **社区活跃**：海量现成脚本，覆盖 SSL Pinning 绕过、Root 检测绕过、加密算法追踪等场景

本文将从零开始，带你搭建 Frida 环境并编写第一个 Hook 脚本。

---

## 一、Frida 架构速览

Frida 采用 **C/S 架构**，分为三个部分：

```
[你的电脑]  ←→  [手机/模拟器]  ←→  [目标 App 进程]
 PC 端              设备端              进程内
frida CLI    →    frida-server    →    frida-agent
(Python/JS)       (守护进程)           (JS 引擎)
```

1. **frida-server**：运行在目标设备上的守护进程，监听默认端口 27042，负责接收 PC 端指令并注入目标进程
2. **frida-agent**：被注入到目标进程中的 JS 引擎，你的 Hook 脚本就在这里执行
3. **frida CLI / Python API**：PC 端的客户端，用于发送指令和脚本

---

## 二、环境安装

### 2.1 PC 端安装

确保 Python 3.8+ 已安装，然后一行命令搞定：

```bash
pip install frida-tools
```

验证安装：

```bash
frida --version
# 输出类似: 16.7.x
```

> **注意**：`frida-tools` 包会自动安装 `frida` 核心库，无需单独安装。

### 2.2 Android 设备端配置

#### 步骤 1：查看设备 CPU 架构

```bash
adb shell getprop ro.product.cpu.abi
# 常见输出: arm64-v8a / armeabi-v7a / x86_64
```

#### 步骤 2：下载 frida-server

前往 [Frida GitHub Releases](https://github.com/frida/frida/releases) 下载与 **PC 端版本一致** 的 frida-server。

> 版本必须一致！PC 端 `frida --version` 和 frida-server 版本号要完全匹配。

以 arm64 设备为例：

```bash
# 下载（以 16.7.x 版本为例）
wget https://github.com/frida/frida/releases/download/16.7.14/frida-server-16.7.14-android-arm64.xz

# 解压
unxz frida-server-16.7.14-android-arm64.xz
```

#### 步骤 3：推送到设备并启动

```bash
# 推送到设备临时目录
adb push frida-server-16.7.14-android-arm64 /data/local/tmp/frida-server

# 赋予执行权限
adb shell "chmod 755 /data/local/tmp/frida-server"

# 启动 frida-server（后台运行）
adb shell "/data/local/tmp/frida-server &"
```

> **Root 设备提示**：如果提示权限不足，先执行 `adb root`。生产版系统无法 `adb root`，可通过 `adb shell "su -c /data/local/tmp/frida-server &"` 解决。

#### 步骤 4：验证连接

```bash
frida-ps -U
```

如果看到设备上的进程列表就说明连接成功：

```
  PID  NAME
 1590  com.android.chrome
13194  com.android.settings
13282  com.twitter.android
...
```

### 2.3 桌面端（Windows/macOS/Linux）使用

桌面端不需要 frida-server，Frida 直接通过进程 PID 或名称附加：

```bash
# 列出本机进程
frida-ps

# 附加到进程
frida -n notepad.exe
```

---

## 三、两种注入模式：Spawn vs Attach

| 模式 | 命令 | 原理 | 适用场景 |
|------|------|------|----------|
| **Spawn（冷启动）** | `frida -U -f 包名 -l script.js` | Frida 启动 App 并在初始化前注入 | 需要 Hook 启动阶段的逻辑（如 `Application.onCreate()`、so 加载） |
| **Attach（热注入）** | `frida -U -n 进程名 -l script.js` | 附加到已在运行的进程 | 中途分析、规避启动注入检测 |

### 常用参数速查

```
-U          通过 USB 连接设备
-f 包名     Spawn 模式，启动指定包名的 App
-n 进程名   Attach 模式，附加到指定进程名
-p PID      Attach 模式，附加到指定 PID
-l script   加载 JS 脚本文件
--no-pause  Spawn 后自动恢复执行（不加则暂停等待）
-F          附加到前台进程
```

---

## 四、Java 层 Hook 实战

Java 层 Hook 是 Android 逆向最常用的操作。所有 Java Hook 代码必须包裹在 `Java.perform()` 中，确保 ART/Dalvik 虚拟机已就绪。

### 4.1 Hello World：Hook 一个方法

```javascript
// hook_basic.js
Java.perform(function() {
    var MainActivity = Java.use("com.example.app.MainActivity");

    MainActivity.checkPassword.implementation = function(password) {
        console.log("[+] checkPassword 被调用，参数: " + password);

        // 调用原方法
        var result = this.checkPassword(password);
        console.log("[+] 原始返回值: " + result);

        // 修改返回值：强制返回 true
        return true;
    };
});
```

注入运行：

```bash
frida -U -f com.example.app -l hook_basic.js --no-pause
```

### 4.2 处理重载方法（Overload）

Java 中同名方法可能有多个重载版本，必须用 `overload()` 指定参数类型：

```javascript
Java.perform(function() {
    var Utils = Java.use("com.example.app.Utils");

    // 指定重载版本: encrypt(String)
    Utils.encrypt.overload('java.lang.String').implementation = function(data) {
        console.log("[+] encrypt(String) 参数: " + data);
        var result = this.encrypt(data);
        console.log("[+] 加密结果: " + result);
        return result;
    };

    // 指定重载版本: encrypt(String, String) — key + data
    Utils.encrypt.overload('java.lang.String', 'java.lang.String').implementation = function(key, data) {
        console.log("[+] encrypt(String, String) Key: " + key + ", Data: " + data);
        return this.encrypt(key, data);
    };
});
```

`overload()` 类型速查：

| Java 类型 | overload 参数 |
|-----------|---------------|
| `int` | `'int'` |
| `boolean` | `'boolean'` |
| `String` | `'java.lang.String'` |
| `byte[]` | `'[B'` |
| `int[]` | `'[I'` |
| `List<String>` | `'java.util.List'` |

### 4.3 Hook 构造函数与静态方法

```javascript
Java.perform(function() {
    var User = Java.use("com.example.app.User");

    // Hook 构造函数
    User.$init.overload('java.lang.String', 'int').implementation = function(name, age) {
        console.log("[+] User 构造: name=" + name + ", age=" + age);
        return this.$init(name, age);
    };

    // Hook 静态方法
    var Utils = Java.use("com.example.app.Utils");
    Utils.getSecret.implementation = function() {
        console.log("[+] 静态方法 getSecret 被调用");
        var result = this.getSecret();
        console.log("[+] Secret: " + result);
        return result;
    };
});
```

### 4.4 枚举已加载类 & 搜索目标类

不知道要 Hook 哪个类时，先枚举所有类：

```javascript
Java.perform(function() {
    Java.enumerateLoadedClasses({
        onMatch: function(className) {
            // 只打印目标包名下的类
            if (className.includes("com.example")) {
                console.log(className);
            }
        },
        onComplete: function() {
            console.log("[*] 类枚举完成");
        }
    });
});
```

找到类后，列出其所有方法：

```javascript
Java.perform(function() {
    var target = Java.use("com.example.app.AESHelper");
    var methods = target.class.getDeclaredMethods();
    methods.forEach(function(method) {
        console.log(method.toString());
    });
});
```

### 4.5 实战：Hook 加密算法抓取密钥

```javascript
Java.perform(function() {
    // Hook AES Cipher.doFinal 抓明文和密文
    var Cipher = Java.use('javax.crypto.Cipher');
    Cipher.doFinal.overload('[B').implementation = function(input) {
        var bytesToHex = function(bytes) {
            var hex = [];
            for (var i = 0; i < bytes.length; i++) {
                hex.push(('0' + (bytes[i] & 0xFF).toString(16)).slice(-2));
            }
            return hex.join('');
        };

        console.log('[AES] 输入明文: ' + bytesToHex(input));
        var result = this.doFinal(input);
        console.log('[AES] 输出密文: ' + bytesToHex(result));
        return result;
    };

    // Hook Base64 编解码
    var Base64 = Java.use('android.util.Base64');
    Base64.encodeToString.overload('[B', 'int').implementation = function(input, flags) {
        var result = this.encodeToString(input, flags);
        console.log('[Base64] 编码结果: ' + result);
        return result;
    };
});
```

---

## 五、Native 层 Hook 实战

对于 .so 库中的 C/C++ 函数，使用 `Interceptor.attach()` 进行拦截。

### 5.1 基础 Native Hook

```javascript
// 拦截 libc.so 中的 open 函数
Interceptor.attach(Module.findExportByName("libc.so", "open"), {
    onEnter: function(args) {
        // args[0] 是第一个参数（文件路径）
        var path = Memory.readUtf8String(args[0]);
        console.log("[open] 打开文件: " + path);
    },
    onLeave: function(retval) {
        // retval 是返回值（文件描述符）
        console.log("[open] 返回 fd: " + retval.toInt32());
    }
});
```

### 5.2 修改参数和返回值

```javascript
Interceptor.attach(Module.findExportByName("libnative.so", "check_license"), {
    onEnter: function(args) {
        console.log("[check_license] 原始参数: " + Memory.readUtf8String(args[0]));
        // 替换参数
        args[0] = Memory.allocUtf8String("VALID-LICENSE-KEY");
    },
    onLeave: function(retval) {
        console.log("[check_license] 原始返回值: " + retval);
        // 强制返回 1（验证通过）
        retval.replace(ptr(1));
    }
});
```

### 5.3 Hook 未导出函数（通过偏移地址）

某些函数没有导出符号，需要用 `Module.findBaseAddress()` + 偏移量定位：

```javascript
var module = Process.findModuleByName("libnative.so");
var baseAddr = module.base;
// 假设目标函数在 offset 0x4A2C 处
var targetAddr = baseAddr.add(0x4A2C);

Interceptor.attach(targetAddr, {
    onEnter: function(args) {
        console.log("[+] 未导出函数被调用，地址: " + targetAddr);
    }
});
```

### 5.4 实战：Hook JNI 函数追踪注册

```javascript
// Hook RegisterNatives 抓 JNI 动态注册
var registerNatives = Module.findExportByName("libart.so", "RegisterNatives");
if (registerNatives) {
    Interceptor.attach(registerNatives, {
        onEnter: function(args) {
            var methods = args[2];
            var count = args[3].toInt32();
            console.log("[RegisterNatives] 注册 " + count + " 个方法");
            for (var i = 0; i < count; i++) {
                var name = Memory.readUtf8String(methods.add(i * 24).readPointer());
                var signature = Memory.readUtf8String(methods.add(i * 24 + 8).readPointer());
                console.log("  - " + name + " :: " + signature);
            }
        }
    });
}
```

---

## 六、Python 自动化注入

当需要批量操作或自动化流程时，使用 Python API 比命令行更灵活。

### 6.1 基础模板

```python
import frida
import sys

# JS Hook 脚本
js_code = """
Java.perform(function() {
    var MainActivity = Java.use("com.example.app.MainActivity");
    MainActivity.checkPassword.implementation = function(pwd) {
        console.log("[+] Password: " + pwd);
        return true;  // 绕过验证
    };
});
"""

def on_message(message, data):
    if message['type'] == 'send':
        print(f"[JS] {message['payload']}")
    else:
        print(message)

# Spawn 模式启动
device = frida.get_usb_device()
pid = device.spawn(["com.example.app"])
session = device.attach(pid)

script = session.create_script(js_code)
script.on('message', on_message)
script.load()

# 恢复执行
device.resume(pid)

# 保持运行
print("[*] 脚本运行中，Ctrl+C 退出...")
sys.stdin.read()
```

### 6.2 Attach 到已运行的进程

```python
device = frida.get_usb_device()
session = device.attach("com.example.app")  # 通过进程名
# 或者 session = device.attach(12345)  # 通过 PID

script = session.create_script(js_code)
script.on('message', on_message)
script.load()
sys.stdin.read()
```

### 6.3 Python 与 JS 双向通信

JS 脚本通过 `send()` 把数据发给 Python 端：

```javascript
// JS 端
Java.perform(function() {
    var Base64 = Java.use('android.util.Base64');
    Base64.encodeToString.overload('[B', 'int').implementation = function(input, flags) {
        var result = this.encodeToString(input, flags);
        // 发回 Python 端记录
        send({type: "base64", data: result});
        return result;
    };
});
```

```python
# Python 端接收并写入文件
def on_message(message, data):
    if message['type'] == 'send':
        payload = message['payload']
        if payload.get('type') == 'base64':
            with open('base64_log.txt', 'a') as f:
                f.write(payload['data'] + '\n')
```

---

## 七、常用场景速查

### 7.1 绕过 SSL Pinning

```javascript
Java.perform(function() {
    var X509TrustManager = Java.use('javax.net.ssl.X509TrustManager');
    var TrustManager = Java.registerClass({
        name: 'com.frida.TrustManager',
        implements: [X509TrustManager],
        methods: {
            checkClientTrusted: function(chain, authType) {},
            checkServerTrusted: function(chain, authType) {},
            getAcceptedIssuers: function() { return []; }
        }
    });
    var TrustManagers = [TrustManager.$new()];

    var SSLContext = Java.use('javax.net.ssl.SSLContext');
    SSLContext.init.overload(
        '[Ljavax.net.ssl.KeyManager;',
        '[Ljavax.net.ssl.TrustManager;',
        'java.security.SecureRandom'
    ).implementation = function(km, tm, sr) {
        console.log('[+] SSL Pinning 已绕过');
        SSLContext.init.call(this, km, TrustManagers, sr);
    };
});
```

### 7.2 绕过 Root 检测

```javascript
Java.perform(function() {
    // 常见 Root 检测绕过
    var Runtime = Java.use('java.lang.Runtime');
    Runtime.exec.overload('java.lang.String').implementation = function(cmd) {
        if (cmd.includes('su') || cmd.includes('which')) {
            console.log('[+] 拦截敏感命令: ' + cmd);
            throw new Error('Command blocked');
        }
        return this.exec(cmd);
    };

    // 屏蔽 Build.TAGS 检测
    var Build = Java.use('android.os.Build');
    Build.TAGS.value = 'release-keys';
});
```

### 7.3 使用 frida-trace 快速追踪

不需要写脚本，一行命令追踪函数调用：

```bash
# Android: 追踪所有 open 调用
frida-trace -U -i open com.android.chrome

# Windows: 追踪 msvcrt.dll 中所有 mem* 函数
frida-trace -p 1372 -i "msvcrt.dll!*mem*"

# iOS: 追踪 crypto 相关调用
frida-trace -U -I "libcommonCrypto*" Twitter
```

---

## 八、比 Xposed 强在哪

| 维度 | Frida | Xposed / LSPosed |
|------|-------|------------------|
| 注入方式 | 运行时注入，无需重启 | 需要重启/Zygote Hook |
| 支持语言 | Java + Native（C/C++） | 仅 Java 层 |
| 跨平台 | Android / iOS / Win / Mac / Linux | 仅 Android |
| 对 APK 修改 | 无需修改 | LSPosed 无需修改 |
| 隐蔽性 | 较低（进程名可检测） | 较高 |
| 上手难度 | 中等 | 较高（需了解模块/作用域） |

一句话：**快速逆向分析用 Frida，长期模块化改造用 LSPosed。**

---

## 九、常见问题排查

### 9.1 frida-ps -U 看不到设备

- 确认 `adb devices` 能列出设备
- 确认 frida-server 已启动：`adb shell "ps | grep frida"`
- 确认 frida-server 版本与 PC 端一致

### 9.2 App 闪退 / Process terminated

- App 可能存在 Frida 检测（端口扫描、`frida-server` 字符串检测）
- 尝试重命名 frida-server：`mv frida-server random_name`
- 使用非标准端口：`./random_name -l 0.0.0.0:8888`

### 9.3 Spawn 模式报错

- 确认包名正确：`frida-ps -Uai | grep 关键词`
- 某些加固 App 可能阻止 Spawn，改用 Attach 模式

### 9.4 overload 报错 "specified overload not found"

- 检查参数类型拼写是否正确
- 使用 `class.getDeclaredMethods()` 确认方法签名

---

## 十、最佳实践总结

1. **先枚举再 Hook**：用 `enumerateLoadedClasses` 摸清目标类结构，再精确 Hook
2. **善用 `send()` 导出数据**：不要在 console.log 里打印大量数据，通过 send 回传 Python 端存入文件
3. **封装成 Python 自动化脚本**：重复分析同一 App 时，Python 脚本可以自动完成 Spawn → 注入 → 收集数据全流程
4. **Native 层优先排查 dlopen**：Hook `android_dlopen_ext` 观察 so 加载顺序，识别加固壳的关键 so
5. **注意版本匹配**：每次升级 Frida 后，PC 端和设备端都要同步更新
6. **绕过检测三件套**：重命名 frida-server + 非标准端口 + 使用 Magisk Hide 隐藏 Root

---

## 参考资源

- [Frida 官方文档](https://frida.re/docs/home/)
- [Frida JavaScript API 参考](https://frida.re/docs/javascript-api/)
- [Frida GitHub 仓库 & 发行版](https://github.com/frida/frida)
- [learnfrida.info — Frida 交互式教程](https://learnfrida.info/)
- [picoCTF Binary Instrumentation 系列](https://picoctfsolutions.com/posts/frida-binary-instrumentation-ctf)
