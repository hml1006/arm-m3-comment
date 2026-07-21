# ARM Cortex-M3 内存屏障指令分析：DMB / DSB / ISB

## 概述

ARM Cortex-M3 (ARMv7-M 架构) 提供了三条内存屏障指令，用于解决处理器流水线、写缓冲器和总线系统带来的**内存访问顺序和完成时间**的不确定性。这三条指令在 CMSIS 库中通过内联汇编封装，位于本项目：

- **CMSIS 定义**: [core_cmInstr.h](file:///home/louis/code/arm-m3-comment/m3designstart/software/cmsis/CMSIS/Include/core_cmInstr.h)
- **CMSIS 使用**: [core_cm3.h](file:///home/louis/code/arm-m3-comment/m3designstart/software/cmsis/CMSIS/Include/core_cm3.h)
- **项目使用**: 分散在 `validation/`、`bootloader/`、`demos/` 等目录

---

## 一、指令对比总览

| 特性 | DMB (Data Memory Barrier) | DSB (Data Synchronization Barrier) | ISB (Instruction Synchronization Barrier) |
|------|--------------------------|-----------------------------------|----------------------------------------|
| **CMSIS 宏** | `__DMB()` | `__DSB()` | `__ISB()` |
| **汇编助记符** | `dmb` | `dsb` | `isb` |
| **核心作用** | 保证内存访问顺序 | 等待所有内存访问完成 | 刷新流水线 |
| **流水线影响** | 不暂停流水线 | **暂停流水线**直到完成 | **清空并刷新**流水线 |
| **写缓冲器** | 不影响 | 等待排空 | 不影响 |
| **执行代价** | 低 (1-2 周期) | 高 (取决于未完成事务) | 中 (约 3 周期 + 取指) |
| **适用场景** | 共享数据/外设寄存器访问排序 | 复位/Flash 编程/系统控制 | MPU 修改/异常处理/自修改代码 |

---

## 二、指令详细分析

### 2.1 DMB — 数据内存屏障

#### CMSIS 实现

[core_cmInstr.h:391-394](file:///home/louis/code/arm-m3-comment/m3designstart/software/cmsis/CMSIS/Include/core_cmInstr.h#L391-L394)

```c
__attribute__((always_inline)) __STATIC_INLINE void __DMB(void)
{
  __ASM volatile ("dmb");
}
```

#### 执行过程

```
内存操作序列:  [写 A] [写 B] [写 C] [DMB] [读 D] [写 E]

时间线:
  写 A ──→ AHB 总线 ──→ 完成
  写 B ──→ AHB 总线 ──→ 完成
  写 C ──→ AHB 总线 ──→ 完成
          DMB 执行
  读 D ──→ AHB 总线 ──→ 完成  (保证在 DMB 之后)
  写 E ──→ AHB 总线 ──→ 完成  (保证在 DMB 之后)

关键点:
  ✓ DMB 前的所有内存操作在 DMB 之前对**所有从机可见**
  ✓ DMB 后的所有内存操作在 DMB 之后对**所有从机可见**
  ✗ DMB 前的操作可能**尚未完成**就被后续指令依赖
  ✗ DMB 后的指令可以**立即继续执行**，不等 DMB 前操作完成
```

#### 微架构执行流程

```
处理器流水线:
  ┌──────┐  ┌──────┐  ┌──────┐
  │ 取指  │→│ 译码  │→│ 执行  │
  └──────┘  └──────┘  └──────┘
                         │
                    DMB 到达执行级
                         │
                    ┌────▼────┐
                    │ 标记屏障 │ ← 处理器内部设置内存排序标记
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ 后续指令 │ ← 流水线继续运行，不等待
                    │ 继续执行 │
                    └─────────┘

AHB 总线层面:
  写操作 A ──→ 写缓冲器 ──→ AHB 矩阵 ──→ 从机
  写操作 B ──→ 写缓冲器 ──→ AHB 矩阵 ──→ 从机
  DMB        ──→ (标记点)
  读操作 D   ──→ 写缓冲器 ──→ AHB 矩阵 ──→ 从机
               (硬件保证 D 在屏障之后才被其他总线主机看到)
```

#### 在本项目中的使用

**本项目未直接使用 `__DMB()`**，但它是 DSB 的"轻量版"，在 CMSIS 标准库中作为基础 API 提供。典型场景：

- 共享数据结构访问（如 RTX 操作系统的任务队列）
- 外设寄存器访问顺序保证
- 双核/多主系统中的数据一致性

---

### 2.2 DSB — 数据同步屏障

#### CMSIS 实现

[core_cmInstr.h:380-383](file:///home/louis/code/arm-m3-comment/m3designstart/software/cmsis/CMSIS/Include/core_cmInstr.h#L380-L383)

```c
__attribute__((always_inline)) __STATIC_INLINE void __DSB(void)
{
  __ASM volatile ("dsb");
}
```

#### 执行过程

```
内存操作序列:  [写 A] [写 B] [写 C] [DSB] [读 D] [写 E]

时间线:
  写 A ──→ 写缓冲器 ═══ 等待 ═══→ AHB 总线 ──→ 完成
  写 B ──→ 写缓冲器 ═══ 等待 ═══→ AHB 总线 ──→ 完成
  写 C ──→ 写缓冲器 ═══ 等待 ═══→ AHB 总线 ──→ 完成
          DSB: 等待所有未完成事务完成↓↓↓
               ├─ 写缓冲器排空 ✓
               ├─ AHB 总线空闲 ✓
               └─ 所有写操作完成 ✓
  读 D ──→ AHB 总线 ──→ 完成  (此时读到的值一定包含所有写操作的结果)
  写 E ──→ AHB 总线 ──→ 完成

关键点:
  ✓ DSB 前的所有内存操作**必须全部完成**后 DSB 才返回
  ✓ DSB 后的指令可以安全地依赖 DSB 前操作的副作用
  ✗ 流水线停顿，影响性能
  ✗ 等待时间取决于未完成事务的数量和类型
```

#### 微架构执行流程

```
处理器流水线:
  ┌──────┐  ┌──────┐  ┌──────┐
  │ 取指  │→│ 译码  │→│ 执行  │  ← DSB 到达执行级
  └──────┘  └──────┘  └──┬───┘
                          │
                     ┌────▼────────────────────────────┐
                     │      DSB 执行引擎                │
                     │                                  │
                     │  ① 冻结流水线                     │
                     │     ├─ 停止取指 (Fetch stall)     │
                     │     └─ 停止译码 (Decode stall)    │
                     │                                  │
                     │  ② 扫描所有未完成事务              │
                     │     ┌────────────────────┐       │
                     │     │ 写缓冲器 (Write     │       │
                     │     │ Buffer)            │       │
                     │     │  └─ 等待所有条目清空│       │
                     │     ├────────────────────┤       │
                     │     │ 存储缓冲器 (Store   │       │
                     │     │ Buffer)            │       │
                     │     │  └─ 等待所有存储完成│       │
                     │     ├────────────────────┤       │
                     │     │ 预取缓冲器 (Prefetch│       │
                     │     │ Buffer)            │       │
                     │     │  └─ 等待未决请求完成│       │
                     │     ├────────────────────┤       │
                     │     │ AHB ICode 总线     │       │
                     │     │  └─ 等待 HREADY=1  │       │
                     │     ├────────────────────┤       │
                     │     │ AHB DCode 总线     │       │
                     │     │  └─ 等待 HREADY=1  │       │
                     │     ├────────────────────┤       │
                     │     │ AHB System 总线    │       │
                     │     │  └─ 等待 HREADY=1  │       │
                     │     └────────────────────┘       │
                     │                                  │
                     │  ③ 全部完成 ✓                    │
                     └────────┬─────────────────────────┘
                              │
                     ┌────────▼────────┐
                     │ 解除流水线冻结   │
                     │ 继续执行后续指令  │
                     └─────────────────┘
```

#### AHB 总线级传播

DSB 的等待效应会通过 AHB 总线矩阵传播到所有从机：

```
处理器核
  │
  ├── ICode 总线 ──→ AHB 矩阵 ──→ Flash/SRAM
  │                              └─ 等待 HREADY=1
  │
  ├── DCode 总线 ──→ AHB 矩阵 ──→ SRAM
  │                              └─ 等待 HREADY=1
  │
  └── System 总线 ─→ AHB 矩阵 ──→ AHB→APB 桥 ──→ UART/Timer/GPIO/看门狗...
                                  │              └─ 等待 APB 写完成
                                  └─ 等待 HREADY=1
```

对于 APB 外设写操作，DSB 的等待路径还包括 AHB→APB 桥的同步延迟：

```
DSB 等待 APB 写完成的过程:
  处理器写 APB 寄存器
    → System AHB 总线 (1 周期)
    → AHB→APB 桥 (2-3 周期同步)
    → APB 总线写外设 (1-2 周期)
    → 写完成返回信号
    → AHB→APB 桥同步返回 (2-3 周期)
    → System AHB 总线完成 HREADY
    → DSB 检测到事务完成
```

#### 在本项目中的使用

**场景 1: 系统复位** — [core_cm3.h:1493-1502](file:///home/louis/code/arm-m3-comment/m3designstart/software/cmsis/CMSIS/Include/core_cm3.h#L1493-L1502)

```c
__STATIC_INLINE void NVIC_SystemReset(void)
{
  __DSB();                               // ① 确保所有缓冲写完成
  SCB->AIRCR = ... | SYSRESETREQ_Msk;   // 写入复位请求
  __DSB();                               // ② 确保复位请求已写入
  while(1);                              // 等待复位
}
```

**场景 2: 中断使能同步** — [uart_tests.c:869-870](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/uart_tests.c#L869-L870)

```c
NVIC_EnableIRQ(UARTOVF_IRQn);   // 使能 NVIC 中断
__DSB();                         // 等待 NVIC 寄存器写入完成
__ISB();                         // 刷新流水线，确保中断立即响应
```

**场景 3: Flash 重映射** — [bootloader.c:106-107](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/bootloader/bootloader.c#L106-L107)

```c
CM3DS_MPS2_SYSCON->REMAP = 0;  // 修改重映射寄存器
__DSB();                        // 等待 REMAP 生效
__ISB();                        // 刷新流水线，从新地址取指
```

---

### 2.3 ISB — 指令同步屏障

#### CMSIS 实现

[core_cmInstr.h:369-372](file:///home/louis/code/arm-m3-comment/m3designstart/software/cmsis/CMSIS/Include/core_cmInstr.h#L369-L372)

```c
__attribute__((always_inline)) __STATIC_INLINE void __ISB(void)
{
  __ASM volatile ("isb");
}
```

#### 执行过程

```
ISB 执行前流水线状态:
  ┌──────┐  ┌──────┐  ┌──────┐
  │ 取指  │→│ 译码  │→│ 执行  │
  │ instr_N│ │ instr_N│ │ ...   │
  │ instr_N│ │ ...   │ │ ISB   │
  │ ...   │ │ ...   │ │       │
  └───────┘  └───────┘  └───┬───┘
                             │
                    ISB 到达执行级
                             │
                    ┌────────▼────────┐
                    │  ① 清空流水线   │
                    │  ├─ 丢弃取指级  │
                    │  ├─ 丢弃译码级  │
                    │  └─ 等待执行级  │
                    │     完成当前指令 │
                    │                 │
                    │  ② 强制重新取指 │
                    │  ├─ 从 ICode 或 │
                    │  │  DCode 总线  │
                    │  │  重新获取    │
                    │  └─ 经过 MMU/   │
                    │     MPU 地址翻译│
                    │                 │
                    │  ③ 重新填充流水线│
                    │  ├─ 取指级填充  │
                    │  ├─ 译码级填充  │
                    │  └─ 恢复执行    │
                    └────────────────┘

ISB 执行后流水线状态:
  ┌──────┐  ┌──────┐  ┌──────┐
  │ 取指  │→│ 译码  │→│ 执行  │
  │ instr_0│ │ instr_0│ │ instr_0│ ← 全新获取，非旧流水线遗留
  │ instr_1│ │ instr_1│ │       │
  │ ...   │ │       │ │       │
  └───────┘  └───────┘  └───────┘
```

#### 关键特性

ISB 是**唯一**能保证处理器使用**更新后的内存属性**执行后续指令的屏障：

| 场景 | 无 ISB 的问题 | 有 ISB 的效果 |
|------|--------------|--------------|
| MPU 配置修改 | 流水线中预取的指令使用旧 MPU 配置 | 强制重新取指，使用新 MPU 配置 |
| 自修改代码 | 指令可能已预取到流水线 | 清空流水线，重新获取新代码 |
| 异常处理 | 异常返回后可能使用旧指令 | 确保异常后正确获取指令 |
| 系统控制寄存器 | 修改后可能不立即生效 | 流水线刷新后生效 |

#### 在本项目中的使用

**场景 1: 用作小延迟** — 这是本项目中最常见的用法

[gpio_driver_tests.c:41](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/gpio_driver_tests.c#L41)

```c
/* Using ISB instruction to create a three cycles delay */
#define small_delay  __ISB
```

[gpio_tests.c:52-53](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/gpio_tests.c#L52-L53)

```c
/* A trick to add a small delay in program */
/* small_delay() is the same as __ISB()    */
#define small_delay  __ISB
```

**原因**: ISB 强制清空流水线并重新填充，这一过程消耗约 3 个以上的时钟周期，且不需要编译器优化干预（`volatile` 保证），因此被用作一个**编译器安全、周期稳定**的短延迟。

**场景 2: 定时器波形产生** — [timer_driver_tests.c:388-389](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/timer_driver_tests.c#L388-L389)

```c
CM3DS_MPS2_GPIO0->DATAOUT = 0x2;    /* 方波高电平 */
__ISB();                              /* 延迟，保证脉宽 */
CM3DS_MPS2_GPIO0->DATAOUT = 0x000;   /* 方波低电平 */
```

**场景 3: UART 时序控制** — [uart_tests.c:974](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/uart_tests.c#L974)

```c
for (i=0; i<3;i++){ __ISB(); }  /* 小延迟 */
for (i=0; i<50;i++){ __ISB(); } /* 中等延迟 */
```

**场景 4: 通用延迟函数** — [designtest_m3.c:354-359](file:///home/louis/code/arm-m3-comment/m3designstart/software/logical/testbench/testcodes/designtest_m3/designtest_m3.c#L354-L359)

```c
void delay(void)
{
  int i;
  for (i=0;i<10;i++){
    __ISB();
  }
}
```

---

## 三、三种指令的完整对比表

| 比较维度 | DMB | DSB | ISB |
|---------|-----|-----|-----|
| **ARM 架构参考手册编号** | Barrier | Synchronization | Flush |
| **影响写缓冲器** | 仅排序，不等完成 | 等待排空 | 不影响 |
| **影响读操作** | 仅排序，不等完成 | 等待所有读完成 | 不影响 |
| **影响预取缓冲器** | 不影响 | 等待未决预取完成 | **清空并重新填充** |
| **影响流水线** | 不影响 | **冻结**直到所有事务完成 | **清空**并重新填充 |
| **影响后续指令的 MPU 属性** | 不影响 | 不影响 | **强制使用新属性** |
| **跨时钟域同步** | 不保证 | **保证**所有域同步完成 | 不直接相关 |
| **典型执行周期数** | ~1 | 3+（取决于总线延迟） | ~3（流水线刷新开销） |
| **CMSIS 宏** | `__DMB()` | `__DSB()` | `__ISB()` |
| **ARM 指令编码** | `0xF3BF 0x8F5F` | `0xF3BF 0x8F4F` | `0xF3BF 0x8F6F` |

---

## 四、标准组合模式

### 模式 1: DSB + ISB — 系统控制修改

```c
// 修改系统控制寄存器
SCB->某寄存器 = 某值;
__DSB();  // 等待修改完成
__ISB();  // 刷新流水线，新配置生效
```

**本项目实例**: [bootloader.c:106-107](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/bootloader/bootloader.c#L106-L107)、[uart_tests.c:869-870](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/uart_tests.c#L869-L870)

### 模式 2: DSB + DSB — 复位序列

```c
__DSB();                    // 等待所有之前的写操作完成
SCB->AIRCR = 复位请求;      // 写入复位请求
__DSB();                    // 等待复位请求写入完成
while(1);                   // 等待复位
```

**本项目实例**: [core_cm3.h:1493-1502](file:///home/louis/code/arm-m3-comment/m3designstart/software/cmsis/CMSIS/Include/core_cm3.h#L1493-L1502)

### 模式 3: ISB-only — 短延迟

```c
__ISB();  // 流水线刷新消耗约 3 周期，用作通用短延迟
```

**本项目实例**: [gpio_tests.c:53](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/gpio_tests.c#L53)、[timer_driver_tests.c:41](file:///home/louis/code/arm-m3-comment/m3designstart/software/common/validation/gpio_driver_tests.c#L41)

---

## 五、总结

| 指令 | 一句话总结 |
|------|-----------|
| **DMB** | 保证内存访问**顺序**，但不等待完成 — 适用于多主系统中的数据排序 |
| **DSB** | 等待所有内存访问**完成**后才继续 — 适用于系统控制/复位/FIFO 操作 |
| **ISB** | **清空流水线**并重新取指 — 适用于 MPU 配置/自修改代码/作为短延迟 |

在 Cortex-M3 的典型应用中，**DSB + ISB 组合**是最常见的屏障模式，用于确保系统控制寄存器的修改安全生效。本项目中的 `uart_tests.c`、`bootloader.c` 和 CMSIS 核心库 `core_cm3.h` 均采用此模式。