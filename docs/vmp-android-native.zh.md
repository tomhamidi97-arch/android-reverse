# Android Native 库上的 VMP：原理、工具链与去虚拟化实战

> 更新：2026-07-30
> **声明**：本文仅用于授权安全研究、CTF 竞赛、学术学习以及对自有或已获授权 App 的安全测试。文章以教学层面讲解保护机制与分析方法论，不提供任何用于未授权访问的工具。**请勿将其中任何内容应用于你无权分析的软件。**
> *原始出处：基于 `goldenfish689/android-reverse` 仓库 `OLLVM/readme.pdf` 改写。*

---

## 摘要

**虚拟机保护（VMP，Virtual Machine Protection）** 把函数原本的机器码替换为一段自定义的私有字节码，由内嵌的解释器执行。原始的控制流图被彻底破坏，静态分析只能看到一个不可读的派发循环。然而逆向的闭环是固定的：

> **定位 dispatcher → 枚举 handler → 还原操作码语义 → 跟踪一次真实执行 → 提升回 C → 差分测试。**

2026 年的主流技术栈是 **QBDI/Unicorn 跟踪 + Triton 符号化提升 + LLM 辅助标注 + SMT 验证**——而不是单纯的静态符号执行。本指南走完整闭环，并附带一个可复现的实验：亲手搭建一个微型 VM，再编写恢复其源码的提升器（lifter）。

---

## 1. VMP 到底做了什么

VMP 保护器对函数做三步变换：

| 阶段 | 输入 | 输出 |
|------|-------|--------|
| **提升** | 原生 IR / 机器码（如 ARM64） | 自定义字节码（私有 ISA） |
| **嵌入** | 字节码 | 一个数据块 + 一个 dispatcher，链接进二进制 |
| **派发** | 对被保护函数的调用 | 运行时由 VM 循环解释执行字节码 |

运行时，原始函数已经不存在了。取而代之的是一个桩（stub）：把一个*虚拟上下文*（虚拟寄存器、虚拟指令指针、虚拟标志位）压栈，然后进入 **dispatcher**——一个不断取下一条操作码、解码、跳转到对应 **handler** 的循环。每个 handler 改写虚拟上下文，正如真实 CPU 指令改写真实寄存器。

防守方的收益在于不对称：分析师必须同时重建 ISA（每个操作码的含义）和程序（操作码序列），才能恢复原始逻辑。设计良好的 VM 单个函数就可能耗费数天甚至数周。

### 1.1 Dispatcher 的三种形态

实战中主要有三种派发方式：

- **`switch` 派发**——单个 `while(1) { switch(op) { case ... } }` 循环。在反汇编器中最易识别（一张大跳转表），也最易跟踪。
- **`if/else` 链**——功能上等同 switch，但更难做模式匹配；常见于手写 VM。
- **线程化派发**（direct threading、Duff's device 风格，或计算跳转 `jmp`/`br`）——每个 handler 末尾直接尾调用或跳转到下一个 handler。不存在中心循环，CFG 呈现为一团乱麻。这是最难的一种，也是玩具 VM 与商业保护器的分水岭。

### 1.2 为什么静态分析会坍塌

被保护的函数通常**没有可恢复的控制流图**。原始意义上的条件分支消失了——源码里的 `CMP/JE` 变成了两条 VM 操作码，其控制流效果被藏在 handler 内部。IDA、Ghidra、Binary Ninja 只会产出一个巨大的基本块或一张扁平、退化的 CFG。分析师赖以判断结构的线索（循环、分支、调用）被蓄意摧毁。

---

## 2. 2026 年的保护生态

VMP 极少单独出现。严肃的 native 目标通常叠加多种变换——动手前先认清这个"栈"。

### 2.1 Android 上常见的保护器

- **VMProtect / Themida**——桌面血统，偶尔被移植到跨平台 SDK；成熟的 VM，多种派发模式 + 变异（mutation）。
- **Tigress**——学术派，源码到源码；能产出多样、可配置的 VM，常见于研究与 CTF。
- **OLLVM 家族**——`ollvm`、**Hikari**、**Arkari**、**goron**、**pluto**，以及移动 SDK 中活跃的各分支。提供控制流平坦化、虚假控制流、指令替换、字符串加密。Android 生态里很多所谓的 "VMP" 样本其实是 *OLLVM 控制流平坦化 + 部分解释器*，并非完整 VM。
- **移动端原生保护器**——Promon SHIELD、DexGuard / Guardsquare、Arxan、梆梆、爱加密、腾讯 Legu、阿里聚安全、360 加固。它们同时包裹 DEX 层与 native 层；其 native VMP 是最难的一档。

### 2.2 VMP 旁常见的伴随混淆

- **不透明谓词（opaque predicate）**——插入恒真/恒假条件，制造不可达边，破坏 CFG 重建。
- **控制流平坦化（CFF）**——用一个状态变量 dispatcher 路由所有流程，摧毁调用/循环结构。常是完整虚拟化的*前奏*。
- **指令替换 / MBA**——把 `a + b` 替换为等价的混合布尔算术表达式（`(a ^ b) + 2*(a & b)`），让人与符号执行都更难读。
- **字符串/常量加密**——运行时才解密，静态字符串交叉引用全部消失。
- **反调试 / 反 Frida**——`ptrace` 自附加、`TracerPid` 检查、对 `/maps` 的 inotify 监控、时序检测、系统调用级检测。*在碰到 VM 之前*就会先碰到这些。
- **反模拟**——区分真实硅片与 Unicorn/QEMU 的检查（缓存行为、勘误、NEON 特性、TLB 语义）。

---

## 3. 分析与去虚拟化工具链

按*阶段*选工具。

| 阶段 | 目标 | 工具（2026） |
|------|------|--------------|
| **分诊** | 定位 VM、dispatcher、handler | IDA Pro 9.x、**Ghidra**（免费）、**Binary Ninja**、**JEB Native** |
| **静态提升** | 符号化还原操作码语义 | **Triton**、**MAAT**、**angr**、**Miasm** |
| **跟踪 / 模拟** | 在插桩下执行 VM | **QBDI**（支持 AArch64）、**Unicorn**、**Qiling**、**Frida + Stalker** |
| **轨迹挖掘** | 从轨迹重建数据流 | 自研 lifter、vmtrace 风格脚本、PIN/DynamoRIO（桌面） |
| **Native 调试** | 在设备上单步 dispatcher | **LLDB**（NDK）、**GDB** + gdbserver、IDA/Ghidra 调试器 |
| **Hook / 绕过** | 跳过反分析、快照上下文 | **Frida**、**Xposed/LSPosed**、**objection** |
| **AI 辅助** | 标注 handler、推断 ISA | LLM 辅助提升（见 §6.4） |

### 3.1 推荐流程

1. **找到 dispatcher。** 找一个紧凑循环：从按"虚拟 IP"索引的缓冲区读取字节/半字，紧跟一个计算跳转（跳转表、AArch64 的 `br x-寄存器`，或一串尾调用）。
2. **枚举 handler。** 每个派发目标落点即一个 handler 入口。把"派发目标 → handler 入口"全部映射出来。handler 数量 ≈ ISA 宽度。
3. **逐个符号化 handler。** 用 Triton 或 angr 计算 handler 对虚拟上下文的影响，得到一个符号化的传递函数。这就是一条操作码的*语义*。
4. **跟踪一次真实执行。** 在 QBDI/Unicorn/Frida 下用已知输入驱动 VM，记录操作码序列。得到的是*一段私有 ISA 程序*。
5. **提升到可读 IR**（VEX、BNIL 或自研），再到 C。把还原出的 C 与原始实现跑差分测试，证明等价。

---

## 4. 实战实验：搭一个微型 VMP，再拆掉它

理解 VMP 最快的办法是亲手搭一个。我们把一个平凡的 C 函数翻译成私有字节码，实现解释器，然后在 §5 编写恢复原始逻辑的提升器。

### 4.1 目标函数

```c
int logic(int x, int y) {
    int z = x * 2 + y;
    if (z == 10) return 1;
    else         return 0;
}
```

### 4.2 设计私有 ISA

四个虚拟寄存器 `r0–r3`（`r3` 兼作比较标志位），六条操作码：

| 操作码 | 助记符 | 编码（字节） | 语义 |
|--------|----------|------------------|-----------|
| `0x01` | `LOAD`  | `01 reg imm`     | `r[reg] = imm`（魔法立即数 `0xFF`=arg0、`0xFE`=arg1） |
| `0x02` | `ADD`   | `02 dst src`     | `r[dst] += r[src]` |
| `0x03` | `MUL`   | `03 dst imm`     | `r[dst] *= imm` |
| `0x04` | `CMP`   | `04 reg imm`     | `r[3] = (r[reg] == imm)` |
| `0x05` | `JE`    | `05 off`         | `if (r[3]) ip = off` |
| `0x06` | `RET`   | `06 reg`         | `return r[reg]` |

### 4.3 手工编译为字节码

```text
LOAD r0, x        ; 01 00 FF
LOAD r1, y        ; 01 01 FE
MUL  r0, 2        ; 03 00 02
ADD  r0, r1       ; 02 00 01
CMP  r0, 10       ; 04 00 0A
JE   0x0D         ; 05 0D   -> 相等则跳到 "return 1"
LOAD r2, 0        ; 01 02 00
RET  r2           ; 06 02   (假分支)
LOAD r2, 1        ; 01 02 01  (偏移 0x0D)
RET  r2           ; 06 02   (真分支)
```

写成 C 数组：

```c
unsigned char bytecode[] = {
    0x01, 0x00, 0xFF, // LOAD r0, arg0
    0x01, 0x01, 0xFE, // LOAD r1, arg1
    0x03, 0x00, 0x02, // MUL  r0, 2
    0x02, 0x00, 0x01, // ADD  r0, r1
    0x04, 0x00, 0x0A, // CMP  r0, 10
    0x05, 0x0D,       // JE   0x0D
    0x01, 0x02, 0x00, // LOAD r2, 0
    0x06, 0x02,       // RET  r2      (假)
    0x01, 0x02, 0x01, // LOAD r2, 1   (偏移 0x0D)
    0x06, 0x02,       // RET  r2      (真)
};
```

### 4.4 解释器（dispatcher）

```c
#include <stdio.h>

int run_vm(unsigned char *code, int arg0, int arg1) {
    int r[4] = {0};   // 虚拟寄存器组（r3 = 标志位）
    int ip    = 0;    // 虚拟指令指针

    for (;;) {
        unsigned char op = code[ip++];
        switch (op) {
            case 0x01: {                            // LOAD
                int reg = code[ip++];
                int val = code[ip++];
                if (val == 0xFF) val = arg0;
                if (val == 0xFE) val = arg1;
                r[reg] = val;
                break;
            }
            case 0x02: {                            // ADD
                int dst = code[ip++], src = code[ip++];
                r[dst] += r[src];
                break;
            }
            case 0x03: {                            // MUL
                int dst = code[ip++], imm = code[ip++];
                r[dst] *= imm;
                break;
            }
            case 0x04: {                            // CMP
                int reg = code[ip++], imm = code[ip++];
                r[3] = (r[reg] == imm);             // 标志位
                break;
            }
            case 0x05: {                            // JE
                int off = code[ip++];
                if (r[3]) ip = off;
                break;
            }
            case 0x06: {                            // RET
                int reg = code[ip++];
                return r[reg];
            }
            default:
                return -1;                          // 非法操作码
        }
    }
}
```

### 4.5 工程结构与构建

```
mini_vmp/
├── src/
│   ├── main.c        # 测试桩：调用 run_vm(bytecode, x, y)
│   ├── vm.c          # run_vm() dispatcher
│   └── disasm.py     # 字节码 -> 助记符提升器（见 §5）
└── README.md
```

```bash
cd mini_vmp/src
gcc -O2 main.c vm.c -o vmtest
./vmtest          # Result = 0   (尝试改变 x、y)
```

> **AArch64 提示。** 要直接面向 Android，用 NDK 交叉编译：
> ```bash
> $NDK/toolchains/llvm/prebuilt/<host>/bin/aarch64-linux-android24-clang \
>     -O2 -shared -fPIC main.c vm.c -o libvmtest.so
> ```
> 推送到 `/data/local/tmp`，在设备上用 `lldb-server` / Frida 运行。

---

## 5. 拆解 VM：编写提升器

§4.4 的解释器就是你在被保护 `.so` 里*会看到*的东西。逆向工程师的任务是从 dispatcher + 字节码块恢复出 §4.1。两个子问题：(a) 还原**操作码表**，(b) 写一个**反汇编器**。

### 5.1 还原操作码表

从 dispatcher 的 `switch` 里，每个 `case` 暴露一条操作码的意图：

```python
INSTR_TABLE = {
    0x01: "LOAD",
    0x02: "ADD",
    0x03: "MUL",
    0x04: "CMP",
    0x05: "JE",
    0x06: "RET",
}
```

在真实目标里，你是在 Triton/angr 里符号化执行每个 handler、按其符号效果给操作码命名——而不是直接从页面上读助记符。

### 5.2 反汇编器（字节码 → 汇编）

```python
def disasm(bytecode):
    i = 0
    while i < len(bytecode):
        op = bytecode[i]
        if   op == 0x01:
            print(f"{i:02X}: LOAD r{bytecode[i+1]}, {bytecode[i+2]}"); i += 3
        elif op == 0x02:
            print(f"{i:02X}: ADD  r{bytecode[i+1]}, r{bytecode[i+2]}"); i += 3
        elif op == 0x03:
            print(f"{i:02X}: MUL  r{bytecode[i+1]}, {bytecode[i+2]}"); i += 3
        elif op == 0x04:
            print(f"{i:02X}: CMP  r{bytecode[i+1]}, {bytecode[i+2]}"); i += 3
        elif op == 0x05:
            print(f"{i:02X}: JE   {bytecode[i+1]}");                 i += 2
        elif op == 0x06:
            print(f"{i:02X}: RET  r{bytecode[i+1]}");                i += 2
        else:
            print(f"{i:02X}: ???  0x{op:02X}");                      i += 1
```

在 §4.3 上运行，能逐字复现那份汇编清单。随后 `LOAD/MUL/ADD/CMP/JE` 序列坍缩回 `if (x*2 + y == 10) return 1; else return 0;`。**这种从 VM 汇编到 C 的最终坍缩，才是规模化场景下的核心难点。**

### 5.3 从手工到自动：符号化提升

手写反汇编器无法扩展到 200 条操作码的商业 VM。生产级做法：

1. **插桩 dispatcher**，用 QBDI 或 Frida-Stalker 记录每条执行指令的 `(ip, opcode, 虚拟上下文前, 虚拟上下文后)`，跨多组输入。
2. **按效果聚类 handler。** 产生相同符号传递函数的 handler，其实是同一条操作码的伪装（商业 VM 会复制 handler 以对抗计数）。
3. **用 Triton 合成语义：** 对每个 handler，具体化输入、提升为 AST、用 MBA 感知的改写化简，输出一条 C 语句。
4. **输出还原的 C 并做差分测试：** 让还原函数与被保护的 `.so` 跑同一份 fuzz 语料，断言结果相等。

---

## 6. 2026 年的前沿

### 6.1 轨迹驱动去虚拟化已成主流

"跟踪 → 聚类 → 提升"流水线（QBDI 轨迹 → handler 聚类 → Triton 提升）在大规模 VM 上已超越纯静态符号执行，因为 angr 式的路径在线程化 dispatcher 上会爆炸。关键是*偏序*轨迹归约：大量 handler 实例坍缩为少数规范语义。

### 6.2 AArch64 是默认战场

`arm64-v8a` 已是主要 ABI；`armeabi-v7a` 属遗留。实际影响：

- dispatcher 用 `br xN` / `blr xN`（经寄存器的间接跳转）而非跳转表——按寄存器加载的目标做模式匹配，而不是按 `.rodata` 表。
- 现代 SoC 上的 PAC（指针认证）与 BTI（分支目标识别）限制 handler 的位置以及你重定向它们的方式。
- MTE（内存标签扩展）能检测被破坏的虚拟上下文——对你*自己*做 handler 语义模糊测试时很有用。

### 6.3 OLLVM 后继模糊了 CFF 与 VMP 的界限

现代分支（Arkari、goron、pluto）实现*部分虚拟化*：仅个别基本块被提升为私有 ISA，其余保持 native 但被平坦化。要把每个函数独立看待——同一个二进制里可能并存 native-CFF、仅 MBA、完全虚拟化三种区域。

### 6.4 LLM 辅助逆向工程

到 2026 年，LLM 已是流水线内真正的放大器——但不是替代品：

- **handler 标注。** 用固定 prompt 把单个反编译 handler 喂给强模型，它能可靠地命名操作码（`ADD`、`ROL`、`XOR-key`）并提议传递函数，你再符号化验证。
- **MBA 去混淆。** 那些让化简器束手无策的混合布尔算术表达式，常被模型化简，再用 Triton 的 `simplify()` 确认。
- **ISA 模式推断。** 给定一组 `(操作码, 效果)` 语料，模型能比人工更快地提出一套自洽的寄存器/标志位模型。
- **注意。** 切勿在没有符号化或差分验证的情况下信任 LLM 给出的传递函数。模型会幻觉出副作用、漏掉标志位。正确模式是 *LLM 提议，SMT 裁决*。

### 6.5 开源提升器与参考文献

- **vmtrace / unvmp**——社区轨迹式提升器；在 GitHub 搜索。
- **Triton**——符号 + 污点，handler 语义的主力工具。
- **QBDI**——生产级 AArch64 DBI，在没有反调试摩擦的前提下跟踪 VM 的最干净方式。
- **angr**——在顽固二进制上做符号执行与 CFG 恢复。
- 基础文献：*Mavroudakis 等，"Virtualization Obfuscators: A Threat for Reverse Engineering"* 及后续去虚拟化论文。

---

## 7. 进阶练习（Bonus）

把微型 VM 扩展到商业保护器的难点：

1. **栈与调用。** 加入 `PUSH`/`POP`、虚拟栈指针，以及带虚拟返回地址的 `CALL`/`RET`。VM 现在能承载嵌套函数——而你的 lifter 必须重建调用图。
2. **线程化派发。** 把 `switch` 换成直接线程化代码（每个 handler 经函数指针尾跳到下一个）。重跑静态工具，观察 CFG 溶解。
3. **handler 复制与随机化。** 把每条操作码的 handler 复制 N 份、用 MBA 等价体改写、每次构建打乱操作码。把反汇编器改成*聚类*而非*枚举*——这正是击败天真分析的关键。
4. **handler 内的不透明谓词。** 插入恒真的守卫，分支进垃圾 handler。确认符号执行现在必须先求解谓词才能聚类。
5. **反模拟。** 加一个读系统寄存器（`cntvct_el0`）或检查缓存行属性的 handler，在 Unicorn 与真实硅片下返回不同结果。然后写一个 Frida 桩把该值固定住，让模拟成功。

完成这五项，§4 的玩具就变成了商业保护器的忠实微缩——而 §5 的 lifter 也就成了真正的去虚拟化工具链。

---

## 8. 总结

VMP 用内嵌解释器执行的私有字节码替换了原始 native 代码，摧毁了原始 CFG。但恢复闭环是固定且可学的：**定位 dispatcher → 枚举 handler → 还原操作码语义 → 跟踪真实执行 → 提升回 C → 差分测试。** 搭好 §4 的实验，在 §5 拆掉它，在 §7 加固它。如果你能提升自己加固过的 VM，你就能提升战场上遇到的大多数目标。

---

*文档版本：2.0——国际版。原始出处 `goldenfish689/android-reverse` 的 `OLLVM/readme.pdf`；为清晰、准确与时效性重写。*
