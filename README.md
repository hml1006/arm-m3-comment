# ARM Cortex-M3 DesignStart Eval RTL (r0p0-02rel0) — 详细设计文档

## 项目概述

本项目是 **ARM Cortex-M3 DesignStart 评估版** 的 RTL 源代码包，版本号 `r0p0-02rel0`。DesignStart 是 ARM 提供的一项计划，允许开发者免费评估和原型设计 Cortex-M3 处理器内核。该软件包包含：

- **Cortex-M3 处理器内核** 集成封装（含调试和跟踪基础设施）
- **CMSDK**（Cortex-M 系统设计套件）—— 可复用的 AHB/APB 总线 IP 和外设
- **FPGA 参考设计** —— 面向 V2M-MPS2 FPGA 原型开发板
- **TRNG**（真随机数发生器）子系统
- **SMM**（子系统模块）—— MPS2 板级外设（VGA、I2S、SSP、SCC）
- **RTC PL031** —— ARM PrimeCell 实时时钟
- **完整的验证环境** —— 含 20 个测试程序、仿真 Testbench 和 CXDT 调试测试器
- **CMSIS 软件包** —— 设备头文件、启动代码、驱动程序和示例程序

## 系统架构

### 四层层次结构

```
fpga_top                              — FPGA 引脚级顶层封装
  |
  +-- fpga_pll_speed                  — PLL 时钟生成
  +-- fpga_rst_sync                   — 复位同步器
  +-- fpga_100hz_gen                  — 100 Hz 节拍发生器
  +-- fpga_system                     — 系统级集成（固定分区）
        |
        +-- m3ds_user_partition       — 用户分区（可重配置分区）
              |
              +-- CORTEXM3INTEGRATIONDS  — Cortex-M3 处理器 + 调试 + 跟踪
              |     |
              |     +-- CORTEXM3          — 处理器内核（3条 AHB-Lite 总线：
              |     |                                   ICode, DCode, System）
              |     +-- DAPSWJDP          — SWD/JTAG 调试端口
              |     +-- CM3ETM            — 嵌入式跟踪宏单元
              |     +-- cm3_tpiu          — 跟踪端口接口单元
              |     +-- cm3_wic           — 唤醒中断控制器
              |     +-- cm3_rom_table     — CoreSight ROM 表
              |     +-- cm3_sync (x3)     — 时钟域交叉同步器
              |     +-- cm3_clk_gate (x4) — 架构时钟门控
              |
              +-- m3ds_peripherals_wrapper  — 外设集成层
              +-- m3ds_sram_subsystem_fpga  — SRAM 子系统 (4x8KB)
              +-- m3ds_user_partition       — 用户逻辑分区
```

### 总线架构

系统基于 **AMBA 3 AHB-Lite** 和 **APB** 协议构建。Cortex-M3 处理器具有三条独立的 AHB-Lite 总线接口：

| 总线 | 用途 | 连接目标 |
|------|------|----------|
| **ICode 总线** | 指令取指 | Flash/Code 存储器 (0x0000_0000) |
| **DCode 总线** | 数据访问 | SRAM 子系统 (0x2000_0000) |
| **System 总线** | 系统/外设访问 | 总线矩阵、AHB 解码器、APB 桥接 |
| **PPB APB** | 私有外设总线 | 调试寄存器、TPIU、ETM、ROM 表 |

## 目录结构

```
arm-m3-comment/
├── README.md                            ← 本文件
├── LICENSE                              — 许可证文件
├── ARM_Cortex_M3_DesignStart_Eval_*.pdf — 发布说明
│
├── cmsdk/                               ← CMSDK（Cortex-M 系统设计套件）
│   └── logical/                         — 30+ 个 IP 模块
│       ├── cmsdk_ahb_busmatrix/         — 可配置 AHB 总线矩阵
│       ├── cmsdk_ahb_master_mux/        — 3:1 AHB 主机多路复用器
│       ├── cmsdk_ahb_slave_mux/         — 10:1 AHB 从机多路复用器
│       ├── cmsdk_ahb_default_slave/     — 默认从机（ERROR 响应）
│       ├── cmsdk_ahb_timeout_mon/       — AHB 超时监视器
│       ├── cmsdk_ahb_bitband/           — 位带操作逻辑
│       ├── cmsdk_ahb_gpio/              — AHB GPIO（16位，每引脚中断）
│       ├── cmsdk_ahb_master_mux/        — AHB 主机多路复用器
│       ├── cmsdk_ahb_slave_mux/         — AHB 从机多路复用器
│       ├── cmsdk_ahb_downsizer64/       — 64→32 位宽度缩减器
│       ├── cmsdk_ahb_upsizer64/         — 32→64 位宽度扩展器
│       ├── cmsdk_ahb_to_apb/            — AHB→APB 桥接
│       ├── cmsdk_ahb_to_apb_async/      — 异步 AHB→APB 桥接
│       ├── cmsdk_ahb_to_ahb_sync/       — 同步 AHB→AHB 桥接
│       ├── cmsdk_ahb_to_ahb_sync_up/    — 同步上行宽度桥接
│       ├── cmsdk_ahb_to_ahb_sync_down/  — 同步下行宽度桥接
│       ├── cmsdk_ahb_to_ahb_apb_async/  — 异步 AHB→AHB+APB 桥接
│       ├── cmsdk_ahb_to_sram/           — AHB→SRAM 接口
│       ├── cmsdk_ahb_to_flash16/32/     — AHB→Flash 接口
│       ├── cmsdk_ahb_to_extmem16/       — AHB→外部 16位 存储器接口
│       ├── cmsdk_ahb_eg_slave/          — 示例 AHB 从机
│       ├── cmsdk_ahb_fileread_masters/  — 仿真用文件读取主机
│       ├── cmsdk_apb_uart/              — APB UART
│       ├── cmsdk_apb_timer/             — APB 定时器（32位递减）
│       ├── cmsdk_apb_dualtimers/        — APB 双定时器
│       ├── cmsdk_apb_watchdog/          — APB 看门狗
│       ├── cmsdk_apb_slave_mux/         — APB 从机多路复用器
│       ├── cmsdk_apb_subsystem/         — APB 子系统
│       ├── cmsdk_apb3_eg_slave/         — 示例 APB3 从机
│       ├── cmsdk_apb4_eg_slave/         — 示例 APB4 从机
│       ├── cmsdk_apb_timeout_mon/       — APB 超时监视器
│       ├── cmsdk_mcu_mtx4x2/            — 4主机×2从机 MCU 总线矩阵
│       ├── cmsdk_fpga_sram/             — FPGA SRAM 封装
│       ├── cmsdk_iop_gpio/              — IO 焊盘 GPIO
│       ├── cmsdk_debug_tester/          — 调试测试器
│       └── models/                      — 行为模型
│           ├── memories/                — RAM/ROM/Flash 行为模型
│           └── protocol_checkers/       — AHB-Lite/APB 协议检查器
│               ├── AhbLitePC/
│               └── ApbPC/
│
├── m3designstart/                       ← Cortex-M3 DesignStart 集成包
│   ├── logical/                         — 逻辑设计源文件
│   │   ├── fpga_top/                    — FPGA 顶层（fpga_top, fpga_system）
│   │   ├── cortexm3integration_ds/      — Cortex-M3 集成封装
│   │   ├── cortexm3integration_ds_obs/  — 混淆版 RTL（授权用户）
│   │   ├── m3ds_peripherals/            — 外设封装（AHB/APB 解码器、SRAM）
│   │   ├── m3ds_user_partition/         — 用户分区（Flash、时间戳计数器）
│   │   ├── models/                      — 通用模型（static_reg）
│   │   └── testbench/                   — 验证环境
│   │       ├── execution_tb/            — 仿真控制（Makefile）
│   │       ├── testcodes/              — 20个测试程序
│   │       ├── verilog/                — Testbench 源文件
│   │       ├── cxdt/                   — CoreSight 调试测试器
│   │       └── integration_cssoc/      — CXDT 二进制镜像
│   ├── fpga/                            — FPGA 综合文件（Quartus）
│   ├── selftest/                        — FPGA 板级自检固件
│   └── software/                        — 软件包
│       ├── cmsis/                       — CMSIS-Core 设备头文件/启动代码
│       └── common/                      — 通用软件（Bootloader、Demo、Dhrystone）
│
├── cortexm3_model/                      ← Cortex-M3 DSM 周期模型
│   ├── CORTEXM3INTEGRATIONDS_dsm.cpp/h  — DSM 封装
│   ├── verilog/                         — Verilog 封装
│   └── carbon/                          — Carbon 模型接口头文件
│
├── trng/                                ← 真随机数发生器子系统
│   └── logical/
│       ├── top/                         — 顶层集成 + 反相器链
│       ├── rng/                         — RNG 引擎（PRNG + TRNG 核心）
│       │   └── trng/                    — TRNG 核心（熵提取 + 健康测试）
│       ├── rosc/                        — 环形振荡器熵源
│       ├── dx_macros/                   — 数字宏单元（FF、MUX、同步器）
│       └── inc/                         — 参数定义文件
│
├── smm/                                 ← 子系统模块（MPS2 外设）
│   └── logical/
│       ├── mps2_ahb_decoder/            — AHB 地址解码器
│       ├── smm_common_fpga/             — FPGA 通用逻辑（块 RAM、ZBT RAM、PLL）
│       ├── pl022_ssp/                   — ARM PrimeCell SSP（SPI 控制器）
│       ├── vga_edk/                     — VGA 显示控制器
│       ├── apb_i2s/                     — I2S 音频控制器
│       └── ds703_scc_r0p3/              — 系统配置控制器
│
├── rtc_pl031/                           ← ARM PrimeCell PL031 实时时钟
│   └── logical/rtc_pl031/verilog/
│       ├── Rtc.v                        — 顶层
│       ├── RtcApbif.v                   — APB 接口
│       ├── RtcCounter.v                 — 32位计数器
│       ├── RtcControl.v                 — 控制逻辑
│       ├── RtcInterrupt.v               — 中断生成
│       └── RtcSynctoPCLK.v              — 时钟同步
│
├── boards/                              ← FPGA 板级恢复文件
│   └── Recovery/
│       ├── MB/HBI0263C/                 — 板级配置
│       └── SOFTWARE/                    — 预编译 .axf 二进制文件
│
└── docs/                                ← 官方 ARM 文档
    ├── arm_cortex_m3_designstart_eval_rtl_and_testbench_user_guide.pdf
    ├── arm_cortex_m3_designstart_eval_rtl_and_fpga_quick_start_guide.pdf
    ├── arm_cortex_m3_designstart_eval_fpga_user_guide.pdf
    └── arm_cortex_m3_designstart_eval_customization_guide.pdf
```

## CMSDK AHB 总线矩阵架构

### 概述

`cmsdk_ahb_busmatrix` 是一个可配置的 **N×M AHB 总线矩阵**，由 Perl 脚本 `BuildBusMatrix.pl` 根据 XML 配置文件生成。它实现了多主设备多从设备之间的 AHB 总线互连。

### 核心模块

| 模块 | 文件 | 功能 |
|------|------|------|
| **总线矩阵顶层** | `cmsdk_ahb_busmatrix.v` | 模板生成的可配置 N×M 矩阵 |
| **AHB-Lite 封装** | `cmsdk_ahb_busmatrix_lite.v` | 全功能 AHB 到 AHB-Lite 的封装 |
| **输入级** | `cmsdk_ahb_bm_input_stage.v` | 每端口保持寄存器 + 突发提前终止 |
| **地址解码器** | `cmsdk_ahb_bm_decode.v` | 每端口地址解码 + 默认从机 |
| **输出级** | `cmsdk_ahb_bm_output_stage.v` | 每端口仲裁器 + 地址/数据多路复用 |
| **简化输出级** | `cmsdk_ahb_bm_single_output_stage.v` | 单连接组合输出级 |
| **轮询仲裁器** | `cmsdk_ahb_bm_round_arb.v` | 轮询仲裁 + 突发跟踪 |
| **固定优先级仲裁器** | `cmsdk_ahb_bm_fixed_arb.v` | 固定优先级（端口0最高） |
| **突发门控仲裁器** | `cmsdk_ahb_bm_burst_arb.v` | 固定优先级 + 突发边界门控 |
| **单连接仲裁器** | `cmsdk_ahb_bm_single_arb.v` | 单连接请求即授权 |
| **默认从机** | `cmsdk_ahb_bm_default_slave.v` | 未映射地址返回 ERROR |

### 数据流路径

```
AHB 主机
  │
  ├──→ 输入级（保持寄存器）
  │      ├── 直接路径：主机可直接访问从机
  │      └── 保持路径：当从机忙时锁存地址/控制信号
  │
  ├──→ 地址解码器（每端口一个）
  │      ├── 地址区域匹配 → 选择对应输出端口
  │      └── 无匹配 → 默认从机（ERROR 响应）
  │
  └──→ 输出级（每从机端口一个）
         ├── 仲裁器（选择哪个输入端口获得访问权）
         ├── 地址/控制多路复用器
         └── 写数据多路复用器
              │
              └──→ AHB 从机
```

### 仲裁方案

| 方案 | 特性 | 适用场景 |
|------|------|----------|
| **单连接** | 直接授权，无竞争 | 稀疏矩阵中单输入→单输出连接 |
| **固定优先级** | 端口0→最高，端口N→最低 | 确定性优先级场景 |
| **轮询** | 循环优先级，支持突发完整性 | 公平共享总线场景 |
| **突发门控** | 固定优先级 + 突发边界保持 | 需要保证突发完整性的场景 |

## 系统地址映射

### 顶层地址空间

| 起始地址 | 结束地址 | 大小 | 描述 |
|----------|----------|------|------|
| `0x0000_0000` | `0x0003_FFFF` | 256 KB | Code/Flash 存储器 |
| `0x0040_0000` | `0x007F_FFFF` | 4 MB | ZBT SRAM 1（外部） |
| `0x2000_0000` | `0x2001_FFFF` | 128 KB | 片上 SRAM（硬件实现 32 KB） |
| `0x2040_0000` | `0x207F_FFFF` | 4 MB | ZBT SRAM 2（外部） |
| `0x2100_0000` | `0x21FF_FFFF` | 16 MB | PSRAM（外部，可选） |
| `0x4000_0000` | `0x4000_FFFF` | 64 KB | 紧耦合 APB 外设 |
| `0x4001_0000` | `0x4001_FFFF` | 64 KB | AHB 外设 |
| `0x4002_0000` | `0x4003_1FFF` | ~80 KB | MPS2 板级外设 |
| `0x4020_0000` | `0x4020_FFFF` | 64 KB | 以太网控制器 |
| `0x4100_0000` | `0x411F_FFFF` | 2 MB | VGA 控制器 + 帧缓冲 |
| `0xE000_0000` | `0xE00F_FFFF` | 1 MB | Cortex-M3 系统控制 + NVIC + MPU |
| `0xE004_0000` | `0xE004_FFFF` | 64 KB | 私有外设总线（调试） |
| `0xE00F_F000` | `0xE00F_FFFF` | 4 KB | CoreSight ROM 表 |

### 外设地址映射

#### 紧耦合 APB 外设（基址 `0x4000_0000`）

| 地址 | 外设 | CMSIS 符号 |
|------|------|------------|
| `0x4000_0000` | 定时器 0 | `CM3DS_MPS2_TIMER0` |
| `0x4000_1000` | 定时器 1 | `CM3DS_MPS2_TIMER1` |
| `0x4000_2000` | 双定时器 | `CM3DS_MPS2_DUALTIMER` |
| `0x4000_4000` | UART 0 | `CM3DS_MPS2_UART0` |
| `0x4000_5000` | UART 1 | `CM3DS_MPS2_UART1` |
| `0x4000_6000` | RTC | `CM3DS_MPS2_RTC` |
| `0x4000_8000` | 看门狗 | `CM3DS_MPS2_WATCHDOG` |
| `0x4000_F000` | TRNG | `CM3DS_MPS2_TRNG` |

#### AHB 外设（基址 `0x4001_0000`）

| 地址 | 外设 | CMSIS 符号 |
|------|------|------------|
| `0x4001_0000` | GPIO 0 | `CM3DS_MPS2_GPIO0` |
| `0x4001_1000` | GPIO 1 | `CM3DS_MPS2_GPIO1` |
| `0x4001_2000` | GPIO 2 | `CM3DS_MPS2_GPIO2` |
| `0x4001_3000` | GPIO 3 | `CM3DS_MPS2_GPIO3` |
| `0x4001_F000` | 系统控制器 | `CM3DS_MPS2_SYSCON` |

#### MPS2 外设（基址 `0x4002_0000`）

| 地址 | 外设 | CMSIS 符号 |
|------|------|------------|
| `0x4002_0000` | 外部 SPI | `CM3DS_MPS2_EXTSPI_BASE` |
| `0x4002_1000` | CLCD SPI | `CM3DS_MPS2_CLCDSPI_BASE` |
| `0x4002_2000` | CLCD 触摸 | `CM3DS_MPS2_CLCDTOUCH_BASE` |
| `0x4002_3000` | 音频配置 I2C | `CM3DS_MPS2_AUDIOCFG_BASE` |
| `0x4002_4000` | I2S 音频 | `CM3DS_MPS2_AUDIO_BASE` |
| `0x4002_5000` | SPI ADC | `CM3DS_MPS2_SPIADC_BASE` |
| `0x4002_6000` | SPI Shield 0 | `CM3DS_MPS2_SPISH0_BASE` |
| `0x4002_7000` | SPI Shield 1 | `CM3DS_MPS2_SPISH1_BASE` |
| `0x4002_8000` | FPGA 系统寄存器 | `CM3DS_MPS2_FPGASYS_BASE` |
| `0x4002_C000` | UART 2 | `CM3DS_MPS2_UART2_BASE` |
| `0x4002_D000` | UART 3 | `CM3DS_MPS2_UART3_BASE` |
| `0x4002_E000` | UART 4 | `CM3DS_MPS2_UART4_BASE` |
| `0x4002_F000` | SCC | `CM3DS_MPS2_SCC_BASE` |
| `0x4003_0000` | GPIO 4 | `CM3DS_MPS2_GPIO4_BASE` |
| `0x4003_1000` | GPIO 5 | `CM3DS_MPS2_GPIO5_BASE` |

### 中断映射

| IRQn | 中断源 | 描述 |
|------|--------|------|
| 0 | UART0 | UART 0 组合中断 |
| 2 | UART1 | UART 1 组合中断 |
| 5 | RTC | 实时时钟 |
| 6 | PORT0_ALL | GPIO 0 组合中断 |
| 7 | PORT1_ALL | GPIO 1 组合中断 |
| 8 | TIMER0 | 定时器 0 |
| 9 | TIMER1 | 定时器 1 |
| 10 | DUALTIMER | 双定时器 |
| 12 | UARTOVF | UART 溢出 |
| 15 | TSC | 触摸屏控制器 |
| 16-31 | PORT0_0~PORT0_15 | GPIO 0 每引脚中断 |
| 32 | SYSERROR | 系统错误 |
| 33 | EFLASH | Flash 控制器 |
| 42 | PORT2_ALL | GPIO 2 组合中断 |
| 43 | PORT3_ALL | GPIO 3 组合中断 |
| 44 | TRNG | 真随机数发生器 |
| 45 | UART2 | UART 2 |
| 46 | UART3 | UART 3 |
| 47 | ETHERNET | 以太网 |
| 48 | I2S | I2S 音频 |
| 49-53 | SPI0~SPI4 | SPI 0~4 |
| 54 | PORT4_ALL | GPIO 4 组合中断 |
| 55 | PORT5_ALL | GPIO 5 组合中断 |
| 56 | UART4 | UART 4 |

## TRNG 子系统架构

### 数据路径

```
环形振荡器（熵源）
  │
  ├── 4 个可选振荡器（rosc_sel[1:0]）
  ├── 可编程延迟参数（deltax, deltay）
  │
  └──→ rnd_src（原始比特流）
       │
       ├──→ trng_sync（同步到 rng_clk 域）
       │
       ├──→ trng_sample_cntr（采样计数阈值，默认 65535）
       │
       ├──→ trng_balancefilter（冯·诺依曼去偏，可旁路）
       │
       ├──→ trng_collector（比特收集，16位块）
       │
       ├──→ crngt_to_trng（NIST SP 800-90B 健康测试，可旁路）
       │
       ├──→ ehr（192 位熵累加器寄存器）
       │
       ├──→ autocorrelation（自相关健康测试，可旁路）
       │
       └──→ PRNG（AES-128 CTR DRBG，SP 800-90A）
              │
              └──→ rng_readout[127:0]（128 位随机数输出）
```

### 关键参数

| 参数 | 值 | 描述 |
|------|-----|------|
| EHR_WIDTH | 192 | 熵累加器寄存器宽度 |
| SAMPLE_CNT_LOCAL_SIZE | 32 | 采样计数器位宽 |
| SAMPLE_CNTR_RST_VAL | 32'hFFFF | 采样计数器复位值 |
| RNG_KEY_WIDTH | 128 | PRNG 密钥宽度 |
| RNG_DIN_AES_DATA_WIDTH | 128 | PRNG 数据路径宽度 |

## 软件架构

### CMSIS 设备层

```
m3designstart/software/cmsis/
├── CMSIS/Include/                  — CMSIS 核心头文件
│   ├── core_cm3.h                  — Cortex-M3 核心访问
│   ├── core_cmFunc.h               — 核心寄存器函数
│   └── core_cmInstr.h              — 核心内联指令
│
└── Device/ARM/CM3DS/
    ├── Include/
    │   ├── CM3DS_MPS2.h            — 外设寄存器定义和地址映射
    │   ├── CM3DS_MPS2_driver.h     — 外设驱动函数
    │   └── system_CM3DS.h          — 系统初始化
    └── Source/
        ├── ARM/startup_CM3DS.s     — ARM 编译器启动代码
        ├── GCC/startup_CM3DS.s     — GCC 启动代码
        ├── system_CM3DS.c          — SystemInit() + SystemCoreClockUpdate()
        └── CM3DS_MPS2_driver.c     — 外设驱动实现
```

### 系统初始化流程

1. **复位向量** → 设置堆栈指针（来自向量表第一项）
2. **SystemInit()** → 配置系统控制块（SCB），设置 SystemCoreClock = 25 MHz
3. **启动代码** → 复制 `.data` 段，清零 `.bss` 段
4. **main()** → 应用程序入口

### 测试程序清单

| 测试名称 | 类别 | 描述 |
|----------|------|------|
| hello | 基础 | 打印 "Hello world"，验证工具链和仿真流程 |
| designtest_m3 | 核心/存储 | LDM/STM 突发测试、地址解码、外设寄存器访问 |
| dhry | 性能 | Dhrystone 基准测试 |
| rtx_demo | RTOS | Keil RTX 实时操作系统演示 |
| interrupt_demo | 中断 | NVIC 中断处理演示 |
| sleep_demo | 功耗管理 | WFI/WFE 睡眠模式 |
| self_reset_demo | 复位 | 系统复位（NVIC/SYSRESETREQ） |
| memory_tests | 存储 | SRAM 和外部存储器读写验证 |
| apb_mux_tests | 总线 | APB 从机多路复用器功能测试 |
| default_slaves_tests | 总线 | AHB/APB 默认从机（总线错误）响应测试 |
| gpio_tests | GPIO | GPIO 寄存器读写和输出引脚翻转 |
| gpio_driver_tests | GPIO | GPIO 驱动级功能测试 |
| timer_tests | 定时器 | 双定时器寄存器读写和超时测试 |
| timer_driver_tests | 定时器 | 定时器驱动级功能测试 |
| uart_tests | UART | UART 寄存器访问和数据收发 |
| uart_driver_tests | UART | UART 驱动级功能测试 |
| dualtimer_demo | 定时器 | 双定时器演示 |
| watchdog_demo | 看门狗 | 看门狗定时器演示 |
| cxdt | 调试 | CoreSight 调试测试器（JTAG/SWD） |

## 仿真验证环境

### 支持的工具链

| 选项 | 仿真器 | 工具 |
|------|--------|------|
| mti（默认） | Mentor ModelSim/Questa | vlib, vlog, vsim, sccom |
| vcs | Synopsys VCS | vlogan, syscan, vcs, simv |
| ius | Cadence Incisive/Xcelium | irun |

### 支持两款编译工具

| 选项 | 工具链 |
|------|--------|
| gcc（默认） | ARM GCC（裸机） |
| ds5 | ARM DS-5 |
| keil | Keil MDK（提供 uVision 项目文件） |

### 仿真模式

- **混淆 RTL 模式**（DSM=no）：使用授权用户可用的混淆版 Cortex-M3 RTL
- **周期模型模式**（DSM=yes）：使用 Carbon 周期近似模型，支持 TARMAC 指令跟踪

### 使用方法

```bash
# 进入仿真目录
cd m3designstart/logical/testbench/execution_tb

# 编译仿真环境
make compile

# 编译单个测试程序
make testcode TESTNAME=hello

# 运行单个测试
make run TESTNAME=hello

# 运行所有测试
make runall

# 使用 VCS 仿真器
make compile SIMULATOR=vcs
make run TESTNAME=dhry SIMULATOR=vcs

# 启用波形输出
make run TESTNAME=uart_tests SIM_VCD=yes
```

## FPGA 综合

### 目标平台

- **目标板**：V2M-MPS2 FPGA 原型开发板
- **FPGA 器件**：Altera Cyclone V / Arria 系列（MPS2 板载）
- **综合工具**：Altera Quartus II（项目文件由 Quartus 工具链支持）

### 构建选项

| 宏定义 | 描述 |
|--------|------|
| REVC | 目标板版本 C |
| INCLUDE_PSRAM | 包含 PSRAM 控制器 |
| INCLUDE_VGA | 包含 VGA 子系统 |
| INCLUDE_BLOCKRAM | 使用块 RAM 作为 SRAM |
| INCLUDE_TRNG | 包含真随机数发生器 |
| DX_FPGA | TRNG 使用 FPGA 结构 |

### 综合流程

```bash
cd m3designstart/fpga/AN511_SMM_CM3DS/synthesis
make                   # 运行 Quartus 综合和布局布线
```

## 设计参数

### Cortex-M3 配置参数

| 参数 | 值 | 描述 |
|------|------|------|
| MPU_PRESENT | 1 | 包含存储器保护单元 |
| NUM_IRQ | 64 | 64 个中断输入 |
| LVL_WIDTH | 3 | 3 位中断优先级（8 级） |
| TRACE_LVL | 3 | 完整跟踪（ITM + DWT + ETM + HTM） |
| DEBUG_LVL | 3 | 完整调试（含 DWT 数据匹配） |
| JTAG_PRESENT | 1 | 包含 JTAG-DP（SW-DP 始终存在） |
| WIC_PRESENT | 1 | 包含唤醒中断控制器 |
| BB_PRESENT | 1 | 包含位带操作 |
| CLKGATE_PRESENT | 0 | 无架构时钟门控 |

### 系统时钟

| 时钟 | 频率 | 描述 |
|------|------|------|
| FCLK | 25 MHz | 自由运行时钟（始终开启） |
| HCLK | 25 MHz | 系统时钟（可门控） |
| cclk | 25 MHz | 内核时钟（睡眠时门控） |
| PCLK | 25 MHz | APB 外设时钟 |

## 许可

本项目包含 ARM Cortex-M3 DesignStart 评估版 RTL，受 ARM 许可协议约束。详情请参阅 `LICENSE` 文件。

## 参考文档

`docs/` 目录下包含以下 ARM 官方文档（PDF 格式）：

1. **ARM Cortex-M3 DesignStart 评估版 RTL 和 Testbench 用户指南**（100894_0000_00_en）
2. **ARM Cortex-M3 DesignStart 评估版 RTL 和 FPGA 快速入门指南**（100895_0000_00_en）
3. **ARM Cortex-M3 DesignStart 评估版 FPGA 用户指南**（100896_0000_00_en）
4. **ARM Cortex-M3 DesignStart 评估版定制指南**（100897_0000_00_en）