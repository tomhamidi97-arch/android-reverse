# 基于 Frida 动态 Hook 的 VMP + OLLVM 逆向分析方案

> 更新:2025-09-09
> **声明**:本方案仅用于授权安全研究、CTF 竞赛与自有 App 的安全测试。本人已用此脚本成功绕过 2 个 VMP+OLLVM 加固项目的防护,**严禁用于任何非法用途**。

## 1. 背景:为什么静态分析会失效

VMP 与 OLLVM 是目前 native 层最常见的两种代码保护手段:

- **OLLVM(Obfuscating LLVM)**:通过控制流平坦化(Control Flow Flattening)、虚假控制流(Bogus Control Flow)、指令替换等 pass,把原本清晰的函数体打散成一张混乱的状态机图。IDA / Ghidra 反汇编后几乎无法人工阅读。
- **VMP(虚拟机保护)**:把关键代码编译为自定义字节码,运行时由一个解释器(VM dispatcher)逐条解释执行。静态只能看到一个巨大的 `switch-case` 派发循环,看不到真实逻辑。

二者叠加之后,**直接静态逆向的成本极高、收益极低**。于是换一个思路:

> 不管 native 内部混淆成什么样,它最终都必须调用操作系统的接口才能产生副作用。我们不去"读懂"混淆代码,而是把这些"出口"全部监控起来。

## 2. 核心思路:三层边界 Hook

native 代码产生副作用只有三条通道,把它们全部 Hook,即可重建完整的函数调用链:

| 层级 | Hook 目标 | 作用 |
|---|---|---|
| **libc 层** | `strlen`、`strstr`、`strcmp`、`open`、`fopen` 等 | 捕获字符串比较、文件读写等底层行为,定位关键判断点 |
| **JNI 层** | `FindClass`、`GetStaticFieldID`、`GetMethodID`、`CallObjectMethodV` 等 | 捕获 native 与 Java 的交互:读哪个类、哪个字段、调哪个方法 |
| **Java 层** | 应用自身的敏感方法 | 捕获上层业务逻辑入口 |

每条 Hook 日志都带上**调用方地址(`called from: 0x...`)**——既能看到"做了什么",又能定位到"是 so 里哪段代码做的",便于和 IDA 的地址交叉对照。

## 3. 环境准备

- 一台 root 的 Android 设备或模拟器,已装 `frida-server`(版本需与 PC 端一致)
- PC 端:`pip install frida-tools`
- 待分析 App 的 VMP+OLLVM 加固 so 文件

## 4. 使用方法

1. 打开 `hook_jni_and_libc.js`,把脚本里默认的 so 名称替换为目标加固 so 的真实名称。
2. 启动 `frida-server`,确认 `frida-ps -U` 能列出设备进程。
3. 以 **spawn 模式**启动 App 并注入脚本(必须 spawn,否则会漏掉启动阶段早期的检测):

```bash
frida -U -f com.yourapp.name -l hook_jni_and_libc.js --no-pause
```

4. 操作 App 触发目标逻辑,观察控制台输出的调用链日志。

## 5. 日志解读实战:还原一段环境检测

下面这段截取自真实运行日志,我们逐块还原它背后的逻辑。

**第一块:读取设备硬件信息**

```
[JNIHook] FindClass:      Class Name: [android/os/Build]
[JNIHook] GetStaticFieldID called
    Class: android.os.Build
    Field Name: MANUFACTURER      Signature: Ljava/lang/String;
    Field Name: BRAND             Signature: Ljava/lang/String;
    Field Name: DISPLAY           Signature: Ljava/lang/String;
[JNIHook] GetStringUTFChars called ...
```

native 通过 JNI 拿到了 `Build.MANUFACTURER`、`Build.BRAND`、`Build.DISPLAY` 三个字符串字段——典型的设备指纹采集。

**第二块:字符串比对(关键判断点)**

```
[LibcHook] strlen  called str: Xiaomi
[LibcHook] strlen  called str: QKQ1.190828.002 test-keys
[LibcHook] strstr  called str1: xiaomi              str2: other1
[LibcHook] strstr  called str1: qkq1.190828.002 test-keys   str2: other1
```

上一步读到的值被传进了 `strlen` / `strstr`。注意 `DISPLAY` 字段里带有 `test-keys`,说明运行在**非官方签名 ROM** 上(模拟器、自定义固件或 root 后重打包的系统常出现该标记)。这段就是在判断:当前是不是一台"可疑设备"。

**第三块:确认自身身份**

```
[LibcHook] strlen called str: com.unity3d.player.UnityPlayerActivity
[JNIHook] GetObjectClass called, class name: android.app.Application
[JNIHook] GetMethodID, Method Name: [getPackageName]  Signature: ()Ljava/lang/String;
[JNIHook] CallObjectMethodV called - Class Name: android.app.Application
```

`com.unity3d.player.UnityPlayerActivity` 是 Unity 游戏的入口 Activity;随后又通过 `Application.getPackageName()` 拿到自身包名——很可能是做**防重打包 / 防克隆校验**(判断自己是不是跑在"正版"包名下)。

**还原结论**

> 加固 so 在启动阶段执行了一次环境安全检测:先采集 `Build` 系列字段识别设备品牌与 ROM 合法性(`test-keys`),再校验自身包名是否被篡改。任一条件命中,后续业务逻辑会走"风控 / 退出"分支。逆向时只要 Hook 这些判断的返回值,即可让检测逻辑走期望分支。

## 6. 方法论小结

1. **先广后深**:三层全量 Hook 先跑一遍,从日志中筛出"可疑片段"(出现敏感字段名、敏感字符串比对)。
2. **靠 `called from` 地址定位**:把高频出现的调用方地址拉到 IDA 里对照,定位到 so 内的检测函数。
3. **针对性 Hook**:锁定目标函数后,改成只 Hook 它的入参 / 返回值,做参数替换或返回值篡改完成绕过。
4. **注意反调试**:部分加固会检测 Frida,必要时配合 `frida-gadget`、改进程名,或使用 magisk 内核级隐藏方案。

逐条分析 Java、JNI、libc 的调用日志,即可还原原始业务逻辑——这正是动态分析相对于静态反混淆的性价比所在。
