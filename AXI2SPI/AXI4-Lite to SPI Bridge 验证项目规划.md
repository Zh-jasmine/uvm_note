---
tags: [project, verification, UVM, AXI, SPI, RAL]
created: 2026-05-12
---

# AXI4-Lite to SPI Bridge 验证项目规划

## 项目定位

本项目是一个完整的数字 IC 验证项目，目标是向实习面试官展示 UVM 基础扎实、理解验证流程、具备 debug 能力的"可培养性"。核心原则：完整且能讲透，胜过复杂但烂尾。

DUT 是一个 AXI4-Lite to SPI Bridge——通过 AXI4-Lite 总线配置寄存器，控制 SPI Master 发送和接收数据。项目同时涉及设计（RTL）和验证（UVM + RAL）。


## DUT 规格概要

> **注：以下规格为初始设计目标，具体实现过程中可能调整。**

```
┌──────────────────────────────────────────────────┐
│              AXI4-Lite to SPI Bridge              │
│                                                   │
│  AXI4-Lite 总线接口                                │
│  ├── 5 通道握手（AW/W/B/AR/R）                     │
│  ├── 地址译码 → 寄存器组映射                        │
│  └── 32-bit 数据宽度，单拍传输                      │
│                                                   │
│  寄存器组（RAL 建模）                               │
│  ├── CTRL   (0x00): enable / CPOL / CPHA / div    │
│  ├── STATUS (0x04): busy / done / error           │
│  ├── TXDATA (0x08): 8-bit 发送数据（写触发传输）     │
│  └── RXDATA (0x0C): 8-bit 接收数据                 │
│                                                   │
│  SPI Master 逻辑                                   │
│  ├── FSM: IDLE → START → SHIFT → DONE             │
│  ├── 4 种模式 (CPOL/CPHA)                          │
│  ├── 可配置分频 (clk_div)                          │
│  └── 字长 8-bit                                    │
│                                                   │
│  TX FIFO / RX FIFO                                │
│  ├── 异步 FIFO（跨 AXI 时钟域 ↔ SPI 时钟域）        │
│  ├── 深度 16，宽度 8-bit                           │
│  └── 满/空标志                                     │
└──────────────────────────────────────────────────┘
```

## 验证架构

```
UVM Test (Base → 派生多种 test)
│
├── AXI Master Agent
│   ├── sequencer → driver  → AXI-Lite 总线写/读
│   └── monitor   → 采样 AXI transaction
│
├── SPI Slave Agent
│   ├── sequencer → driver  → 模拟 SPI 从机行为
│   └── monitor   → 采样 SPI 四线波形
│
├── Scoreboard
│   ├── AXI 写 → 预测 SPI 波形（参考模型）
│   ├── SPI 实际波形 → AXI 读回比对
│   └── TX FIFO / RX FIFO 行为检查
│
├── RAL（寄存器抽象层）
│   ├── reg_block: 封装 4 个寄存器
│   ├── adapter: reg2bus / bus2reg 转换
│   └── sequence: 前门/后门访问
│
├── Functional Coverage
│   ├── SPI 4 种 mode cross
│   ├── 分频值覆盖
│   ├── FIFO 满/空边界
│   └── AXI 错误响应
│
└── Assertion (SVA)
    ├── AXI VALID/READY 握手超时
    ├── SPI 模式时序约束
    └── FIFO 溢出检测
```

## 项目推进顺序

时间不规划，按以下五个阶段递进：

**阶段一：协议理解。** 先弄清 AXI4 和 AXI4-Lite 的区别（为什么本项目选 Lite），理解 AXI4-Lite 五通道握手机制和地址对齐规则。然后复习 SPI 四种模式（CPOL/CPHA）的时序关系。最后阅读参考 RTL（rooinasuit/AXI_to_SPI 和 arhamhashmi01/Axi4-lite），理解握手信号在代码中如何实现。

**阶段二：RTL 设计。** 依次实现 AXI4-Lite Slave 接口（地址译码 + 五通道握手）、寄存器组（CTRL/STATUS/TXDATA/RXDATA）、SPI Master 状态机和移位逻辑、TX/RX 异步 FIFO，最后封装为顶层 Bridge 模块。此阶段产出完整的可综合 DUT。

**阶段三：UVM 环境搭建。** 定义 interface 和 transaction 类型，实现 AXI Master Agent（driver 发起读写、monitor 采样总线）、SPI Slave Agent（driver 模拟从机返回数据、monitor 采样 SPI 波形）、Scoreboard（全路径数据比对），最后搭建 Env + Base Test + Makefile，跑通最基本的 transaction。

**阶段四：RAL 集成。** 搭建寄存器模型（reg_block + reg_map），实现 adapter（reg2bus 和 bus2reg 两个核心函数），编写 RAL sequence（前门写入配置寄存器、后门读取比对 mirror 值）。这一阶段是区分"背了 UVM 框架"和"真正用过 UVM"的分水岭。

**阶段五：覆盖率与收尾。** 添加功能覆盖率（cross SPI mode × divider、FIFO 边界条件），编写关键 SVA 断言，覆盖多模式传输、reset 场景、FIFO 空满和溢出、AXI 错误响应路径。最终整理 debug 故事和简历描述。

## 技术栈

- **设计语言**: SystemVerilog (RTL)
- **验证语言**: SystemVerilog + UVM 1.2
- **工具链**: VCS + Verdi（运行于 CentOS 7 虚拟机）
- **参考资源**: rooinasuit/AXI_to_SPI（RTL 结构参考）、arhamhashmi01/Axi4-lite（AXI 握手参考）、amrelbatarny/UVM_RAL-Based（RAL 模式参考）

## 交付目标

- 完整 UVM 验证环境可运行，覆盖 5 个以上 test case
- RAL 集成，支持前门和后门两种访问方式
- 功能覆盖率超过 90%
- 至少一个可讲述的 debug 案例（例如 SPI mode 3 下 bit shift 错误的 Verdi 波形追踪定位过程）
