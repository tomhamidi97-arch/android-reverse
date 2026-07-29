# 互联网 8+ 年 · 安卓逆向 & 移动安全 · 独立接单

**Android / iOS 逆向接单 · 移动安全研究** — 联系我（Telegram）：[@pidanfuzi](https://t.me/pidanfuzi)

> [!IMPORTANT]
> 🟢 **接单方向**：协议分析 / 算法还原 / 加固脱壳修复 / APP 风控研究 / 移动端安全审计 / 授权渗透测试 / 自动化与 RPA / 系统级开发（定制 ROM、模拟器/虚拟机方案）
>
> 🔴 **不接方向**：未授权破解他人付费应用、规避支付、批量注册/黑产群控、窃取用户隐私、任何违反目标所在国法律的需求。
> 所有业务**仅限书面授权或自有 APP**，接单前签订 NDA + 授权说明。

---

## ⭐ Featured Write-up

> [**Reversing Promon SHIELD: From Emulator Detection to an Xposed Data-Only Patch**](docs/promon-shield.md)
>
> 一次**授权**的 RASP / app-shielding 对抗全流程：从崩溃栈回溯到 native 检测源、用 bit 级实验矩阵验证假设、把一次性 root 补丁演进为完全进程内的稳定 LSPosed 模块。覆盖 OLLVM 反混淆、多 base 映射、stale cache、数据-only 补丁方法论。这是我最能体现工程深度的案例。

---

## 📦 业务方向

> 以下均为**授权范围内**的合规业务方向（已脱敏表述）。

```mermaid
graph LR
  ROOT(["Android 逆向"])
  ROOT --> A["📊 数据采集（合规）"]
  ROOT --> B["🔌 协议定制 & 系统集成"]
  ROOT --> C["🛡 安全审计 & 对抗研究"]
  ROOT --> D["🤖 自动化 & RPA"]
  ROOT --> E["🖥 系统级开发"]

  A --> A1["授权 app / 网页数据采集"]
  A --> A2["竞品数据分析"]

  B --> B1["私有协议还原 / 接口 SDK 化"]
  B --> B2["企业内部系统集成（授权）"]

  C --> C1["脱壳 / 加固 / 算法提取"]
  C --> C2["风控检测研究（授权审计 / 自有 APP）"]

  D --> D1["脚本 / 插件开发"]
  D --> D2["抓包-分析-复现流水线"]

  E --> E1["定制 ROM"]
  E --> E2["模拟器 / 虚拟机方案"]

  classDef root fill:#4a4a4a,color:#fff,stroke:none
  classDef a fill:#e74c3c,color:#fff,stroke:none
  classDef b fill:#27ae60,color:#fff,stroke:none
  classDef c fill:#e67e22,color:#fff,stroke:none
  classDef d fill:#f1c40f,color:#333,stroke:none
  classDef e fill:#16a085,color:#fff,stroke:none
  classDef leaf fill:#ffffff,color:#333,stroke:#ccc
  class ROOT root
  class A a
  class B b
  class C c
  class D d
  class E e
  class A1,A2,B1,B2,C1,C2,D1,D2,E1,E2 leaf
```

| 方向 | 说明 |
|---|---|
| **协议分析 & 定制** | 私有协议还原 / 接口 SDK 化 / 与企业内部系统集成（授权）→ 作品站：[andriodanalysis.com](https://andriodanalysis.com) |
| **算法还原** | Java 层 & Native(so) 加解密、签名、token 逻辑还原，转 Python / Go / Node |
| **加固 & 脱壳** | 各类免费/企业壳脱壳、dex dump、修复运行、VMP / OLLVM / Java2C 分析 |
| **风控 & 检测对抗研究** | 设备指纹 / root / 模拟器 / Frida 检测 的对抗与绕过（**用于授权安全审计 / 自有 APP 防护测试**） |
| **自动化 & RPA** | 机械重复任务自动化、脚本 / 插件开发、抓包-分析-复现流水线 |
| **数据采集（合规）** | 授权范围内 app / 网页数据采集、竞品数据分析 |
| **系统级开发** | 定制 ROM、模拟器 / 虚拟机方案开发 → 作品站：[ypsmkj.us](https://ypsmkj.us) |
| **跨平台逆向** | Flutter / React Native / uni-app |

---

## 🧭 能力图谱

<p align="center">
  <img src="assets/capability-map.png" alt="Android 逆向能力图谱" width="780">
</p>

---

## 🛠 技术栈

<p align="center">
  <img src="assets/tech-stack.webp" alt="Android 逆向技术栈" width="820">
</p>

**逆向 / 分析**
![Frida](https://img.shields.io/badge/Frida-√-1a1a1a) ![Xposed](https://img.shields.io/badge/Xposed-√-1a1a1a) ![LSPosed](https://img.shields.io/badge/LSPosed-√-1a1a1a) ![Jadx](https://img.shields.io/badge/Jadx-√-1a1a1a) ![Apktool](https://img.shields.io/badge/Apktool-√-1a1a1a) ![JEB](https://img.shields.io/badge/JEB-√-1a1a1a) ![IDA Pro](https://img.shields.io/badge/IDA_Pro-√-1a1a1a) ![Ghidra](https://img.shields.io/badge/Ghidra-√-1a1a1a) ![unidbg](https://img.shields.io/badge/unidbg-√-1a1a1a)

**Native / 汇编**
![ARM64](https://img.shields.io/badge/ARM64-√-blue) ![OLLVM](https://img.shields.io/badge/OLLVM-反混淆-orange) ![VMP](https://img.shields.io/badge/VMP-分析-orange) ![C/C++](https://img.shields.io/badge/C/C++-√-blue)

**脚本 / 复现**
![Python](https://img.shields.io/badge/Python-√-3776AB) ![Smali](https://img.shields.io/badge/Smali-√-orange) ![Node.js](https://img.shields.io/badge/Node-√-339933) ![adb](https://img.shields.io/badge/adb-√-green)

**环境**
![Magisk](https://img.shields.io/badge/Magisk-√-green) ![Zygisk](https://img.shields.io/badge/Zygisk-√-green) ![Genymotion](https://img.shields.io/badge/Genymotion-√-blue) ![云手机](https://img.shields.io/badge/云手机-√-9cf)

---

## 🌐 作品 / 产品

| 作品 | 说明 | 链接 |
|---|---|---|
| 虚拟机 / 云手机方案 | 自研模拟器 / 虚拟机环境，逆向与自动化基础设施 | [ypsmkj.us](https://ypsmkj.us) |
| 插件 / 协议定制站 | 协议还原、算法提取、插件与 SDK 定制交付 | [andriodanalysis.com](https://andriodanalysis.com) |

---

## 💼 客户案例（脱敏）

> 以下均已脱敏，仅描述技术方向，不泄露客户与具体产品。

a) **某社交 APP** — 私有协议全量还原，导出 API 文档并交付 Python 复现脚本
b) **某金融类 APP** — 加固脱壳 + so 层签名算法还原（OLLVM 混淆），unidbg 模拟执行
c) **某直播/短视频** — 设备指纹与风控检测对抗研究（授权安全审计）
d) **某 Flutter 应用** — Flutter Release 模式逆向，Dart snapshot 分析，还原核心接口
e) **某出海应用** — 多语言/多地区包签名差异分析
f) **某游戏** — so 层加密逻辑定位与协议还原
g) **某电商** — 价格/库存接口签名算法还原，转 Python SDK
h) **RASP 加固应用（Promon SHIELD）** — 全链路检测对抗 + LSPosed 数据-only 补丁，详见 [Featured Write-up](docs/promon-shield.md)

> 想看更多？私信 [@pidanfuzi](https://t.me/pidanfuzi) 获取与您行业匹配的脱敏案例。

---

## 📑 研究分析文档

- ⭐ [Reversing Promon SHIELD — RASP 全链路对抗](docs/promon-shield.md) *(Featured)*
- [Android 逆向需求分析 (1) — 模拟器 / 云手机 / 真机](docs/01-环境选型.md)
- [Android 逆向需求分析 (2) — 主流加固方案对比](docs/02-加固对比.md)
- [2026 Android 逆向：技术与业务趋势](docs/03-2026趋势.md)

---

## 🌍 接单范围与语言

| 市场 | 支持 | 说明 |
|---|---|---|
| 🇨🇳 中文 | ✅ | 主力市场，看雪 / 吾爱 / 猿急送 / 私单 |
| 🌐 英文 | ✅ | Upwork / Fiverr / 授权漏洞赏金（Bugcrowd / HackerOne）|
| 其他语言 | ✅ | 文档交付支持英/中双语，沟通可机翻 |

---

## 📈 行业趋势判断

> 1. **需求饥渴**：移动端安全与协议分析岗位长期供不应求，独立接单启动成本低。
> 2. **技术壁垒上移**：OLLVM / VMP / unidbg / Flutter / RASP 成为分水岭，单价持续走高。
> 3. **出海增量**：TikTok / Shopee / WhatsApp / 东南亚钱包类需求旺盛。
> 4. **合规化**：纯破解灰产在收紧，**授权安全审计、自有 APP 防护测试**才是长线方向。
> 5. **AI 加持**：逆向 + LLM 辅助反混淆/注释成为新生产力。

---

## 📞 联系方式

| 渠道 | 账号 |
|---|---|
| Telegram | [@pidanfuzi](https://t.me/pidanfuzi) |
| Email | `tomhamidi97@gmail.com` |
| GitHub | [@tomhamidi97-arch](https://github.com/tomhamidi97-arch) |

**接单流程**：需求描述 → 报价 & 工期 → 签授权/NDA → 交付（脱敏报告 + 可运行脚本）→ 售后。

---

<sub>本仓库仅用于技术能力展示与合法授权业务承接，不提供任何未授权破解工具或服务。</sub>
