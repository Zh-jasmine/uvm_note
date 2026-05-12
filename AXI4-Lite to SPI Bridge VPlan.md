

## 模块概述

AXI4-Lite to SPI Bridge 是一个协议转换模块，将 AXI4-Lite 总线的寄存器读写操作翻译为 SPI Master 接口的串行时序。内部包含 AXI4-Lite Slave 接口、寄存器组、TX/RX 异步 FIFO 和 SPI Master 状态机。验证工程师通过 AXI 总线配置寄存器来控制 SPI 的发送和接收。

## 验证范围

**验什么：** 寄存器的读写正确性、SPI 四种 mode 下的时序正确性、AXI 写数据到 SPI 发出的完整路径、SPI 收到数据到 AXI 读回的完整路径、FIFO 空满行为与反压、地址译码与错误响应、异步时钟域数据完整性。

**不验什么：** 本验证计划不覆盖 SPI 外接从设备的完整行为模型（SPI Slave Agent 仅作简单回环或预设响应）、不覆盖 AXI4-Lite Full 协议特性（burst、outstanding、乱序返回）、不进行门级仿真或低功耗验证。

---

## 验证计划矩阵

| ID | Feature | Sub-Feature | Feature Description | Verification Goals | Pass/Fail Criteria | Test Type | Coverage Method |
|----|---------|-------------|---------------------|-------------------|-------------------|-----------|----------------|
| R1 | AXI4-Lite 基本读写 | 单笔写 | 验证验证工程师通过 AXI 总线向一个合法地址写入 32 位数据，SLVERR 不拉高，BVALID 在 expected 周期内返回 | 所有合法地址单笔写正确完成，B 通道响应 OKAY | 写数据通过 scoreboard 比对镜像寄存器值 | Directed | Code Coverage |
| R2 | AXI4-Lite 基本读写 | 单笔读 | 验证工程师向一个合法地址发起读请求，RVALID 返回时 RDATA 与寄存器当前值一致 | 所有合法地址读回正确数据 | RDATA 与 scoreboard 镜像一致 | Directed | Code Coverage |
| R3 | AXI4-Lite 基本读写 | 非法地址访问 | 验证工程师向未映射的地址空间发起读写，Slave 返回 SLVERR | 非法地址读写均返回 SLVERR，不产生误写 | SLVERR 拉高，目标寄存器值不变 | Directed | Functional Coverage |
| R4 | AXI4-Lite 握手 | VALID 先于 READY | 验证工程师驱动 AWVALID/WVALID 在 AWREADY/WREADY 之前到达，确认握手完成 | 所有 VALID 先来的场景正常握手 | 事务完成且数据正确 | Constrained-Random | Functional Coverage |
| R5 | AXI4-Lite 握手 | READY 先于 VALID | 验证工程师驱动 AWREADY/WREADY 在 AWVALID/WVALID 之前拉高，确认 Slave 不误触发 | READY 先来时 VALID 到达后正常握手 | 事务完成且数据正确 | Constrained-Random | Functional Coverage |
| R6 | AXI4-Lite 握手 | 地址非对齐 | 验证工程师驱动非 4 字节对齐地址访问寄存器 | Slave 返回 SLVERR，不产生误写 | SLVERR 拉高，寄存器值不变 | Directed | Functional Coverage |
| S1 | SPI 模式 | Mode 0 (CPOL=0, CPHA=0) | 验证工程师配置 ctrl 寄存器为 mode 0，写入 tx_data，确认 SPI 输出时序正确 | SCK 空闲低，数据在首个边沿（上升沿）采样 | spi_sclk 波形符合 mode 0 定义，spi_mosi 数据正确 | Directed | Functional Coverage |
| S2 | SPI 模式 | Mode 1 (CPOL=0, CPHA=1) | 验证工程师配置 ctrl 寄存器为 mode 1，确认 SPI 输出时序正确 | SCK 空闲低，数据在第二个边沿（下降沿）采样 | spi_sclk 波形符合 mode 1 定义，spi_mosi 数据正确 | Directed | Functional Coverage |
| S3 | SPI 模式 | Mode 2 (CPOL=1, CPHA=0) | 验证工程师配置 ctrl 寄存器为 mode 2，确认 SPI 输出时序正确 | SCK 空闲高，数据在首个边沿（下降沿）采样 | spi_sclk 波形符合 mode 2 定义，spi_mosi 数据正确 | Directed | Functional Coverage |
| S4 | SPI 模式 | Mode 3 (CPOL=1, CPHA=1) | 验证工程师配置 ctrl 寄存器为 mode 3，确认 SPI 输出时序正确 | SCK 空闲高，数据在第二个边沿（上升沿）采样 | spi_sclk 波形符合 mode 3 定义，spi_mosi 数据正确 | Directed | Functional Coverage |
| S5 | SPI 字长 | 8 位模式 | 配置字长为 8，写入 8 位数据，确认 SPI 只输出 8 个 SCK 周期 | MOSI 上输出 8 bit 后 SCK 停止 | 8 个 SCK 周期后数据传输完成标志拉高 | Directed | Code Coverage |
| S6 | SPI 字长 | 16 位模式 | 配置字长为 16，确认 SPI 输出 16 个 SCK 周期 | MOSI 上输出 16 bit 后 SCK 停止 | 16 个 SCK 周期后传输完成 | Directed | Code Coverage |
| S7 | SPI 字长 | 4 位 / 32 位模式 | 配置字长为 4 或 32，确认 SPI 输出正确周期数 | MOSI 上输出对应 bit 数 | 4 或 32 个 SCK 周期后传输完成 | Directed | Functional Coverage |
| B1 | Bridge 全路径 | AXI 写 → SPI 出 | 验证工程师向 tx_data 寄存器写入数据，触发 SPI 输出，scoreboard 比对写入值与 MOSI 上串行化后的值 | 写入的每 bit 数据与 SPI MOSI 上对应 bit 一致 | SPI Slave Agent 接收数据后 scoreboard 比对通过 | Constrained-Random | Functional Coverage |
| B2 | Bridge 全路径 | SPI 入 → AXI 读 | 验证工程师通过 SPI Slave Agent 向 MISO 输入数据，DUT 接收完成后验证工程师读取 rx_data | MISO 输入的值与 AXI 读回的值一致 | Scoreboard 比对通过 | Constrained-Random | Functional Coverage |
| B3 | Bridge 全路径 | 连续多笔传输 | 验证工程师连续多次写入 tx_data，确认 DUT 正确处理背靠背传输 | 多笔数据依次发出，无丢包、无顺序错误 | Scoreboard 中所有数据按序匹配 | Constrained-Random | Functional Coverage |
| F1 | FIFO | TX FIFO 空 | TX FIFO 为空时，写 tx_data 后数据正确进入 FIFO | 写操作后 FIFO 空标志拉低，数据能正常发出 | 数据最终通过 SPI 发出 | Directed | Functional Coverage |
| F2 | FIFO | TX FIFO 满 | 连续快速写入直到 FIFO 满，确认 DUT 正确置起满标志 | 满标志拉高后后续写操作被反压 | 满标志行为正确，无反压后写丢失 | Constrained-Random | Functional Coverage |
| F3 | FIFO | RX FIFO 溢出 | SPI Slave 持续输入数据超过 RX FIFO 深度，确认 DUT 正确响应 | FIFO 满后 MISO 数据丢失或反压，无 metastability | 无 X/Z 态传播，FIFO 状态信号正确 | Directed | Assertion Based |
| F4 | FIFO | 跨时钟域数据完整性 | AXI 时钟域与 SPI 时钟域频率比为 2:1、1:1、1:2，确认数据不丢失 | 所有数据在跨时钟域后正确 | Scoreboard 比对通过 | Constrained-Random | Functional Coverage |
| E1 | 错误与异常 | 传输中写新数据 | SPI 正在发送时，验证工程师再次写入 tx_data | DUT 应处理冲突（忽略新写入或等待） | DUT 行为与规格定义一致 | Directed | Functional Coverage |
| E2 | 错误与异常 | Reset 行为 | 断言复位，确认所有寄存器回到复位值，SPI 输出回到空闲态 | 复位后寄存器值 = reset value，spi_sclk 回到空闲电平 | RAL mirror 比对 + 波形检查 | Directed | Code Coverage |
| C1 | RAL | 前门访问 | 验证工程师通过 RAL 的 write()/read() 操作所有寄存器 | 通过 RAL 发出的读写行为与直接 AXI 驱动一致 | 写入后回读一致 | Directed | Code Coverage |
| C2 | RAL | 后门访问 | 验证工程师通过 RAL 后门直接修改 DUT 内部寄存器值 | 后门写后镜像值与硬件一致 | RAL mirror 值与 DUT 内寄存器值相同 | Directed | Functional Coverage |
| C3 | RAL | 寄存器复位值检查 | 验证工程师通过 RAL 的 reset() 检查所有寄存器复位值 | 每个寄存器的复位值与 spec 匹配 | RAL mirror 值与 spec 定义一致 | Directed | Assertion Based |

---

## 测试用例清单

### 定向测试

| Test Name | 对应 ID | 场景描述 |
|-----------|---------|---------|
| basic_write_read | R1, R2 | 遍历所有寄存器，写一次读一次，确认读写正确 |
| invalid_addr | R3 | 向 0xF00、0xFFC 等非法地址发起读写 |
| spi_mode_0 | S1 | 配 mode 0，写 tx_data，抓 SPI 波形 |
| spi_mode_1 | S2 | 配 mode 1，写 tx_data，抓 SPI 波形 |
| spi_mode_2 | S3 | 配 mode 2，写 tx_data，抓 SPI 波形 |
| spi_mode_3 | S4 | 配 mode 3，写 tx_data，抓 SPI 波形 |
| spi_wordlen_8 | S5 | 配字长为 8 |
| spi_wordlen_16 | S6 | 配字长为 16 |
| reset_test | E2 | 拉低复位，观察所有信号回到初值 |
| ral_reset_check | C3 | RAL 后门检查所有寄存器复位值 |
| ral_backdoor | C2 | RAL 后门写 + 前门读确认 |

### 约束随机测试

| Test Name | 对应 ID | 场景描述 |
|-----------|---------|---------|
| rand_reg_rw | R4, R5 | 随机地址 + 随机数据 + 随机握手延迟，读写验证 |
| rand_spi_tx | B1, F1, F2 | 随机 mode + 随机字长 + 随机数据连续发送 |
| rand_spi_rx | B2, F3 | SPI Slave Agent 随机输入 + AXI 读回比对 |
| rand_multi_tx | B3, F4 | 背靠背多笔传输 + 随机时钟频率比 |

---

## 功能覆盖率定义

| Coverage Group | 覆盖点 | 仓数 | 说明 |
|---------------|--------|------|------|
| axi_write | addr[7:2], wdata[31:0], awready_delay, wready_delay | 组合 | AXI 写操作的地址和数据覆盖 |
| axi_read | addr[7:2], rdata[31:0], arready_delay | 组合 | AXI 读操作的地址和数据覆盖 |
| spi_cfg | cpol, cpha, word_len | 4x3=12 | SPI 配置组合全覆盖 |
| fifo_state | tx_empty, tx_full, rx_empty, rx_full | 4 | 所有 FIFO 状态覆盖 |
| error_resp | slverr_on_invalid, slverr_on_unaligned | 2h | | error_resp | slverr_on_invalid, slverr_on_unaligned | 2 | 错误响应覆盖 |
| busy_conflict | tx_during_busy | 2 | 忙时写数据冲突覆盖 |

## 验证出口标准

本项目的验证出口标准如下：所有定向测试通过。所有约束随机测试通过，无死锁或超时。功能覆盖率超过百分之九十。无需修复的断言违规。至少一次回归（同一 test 用不同 seed 跑三轮以上）。一份可展示的 debug 记录（定位并修复至少一个 RTL bug）。

## 项目文件结构

```
AXI4-Lite_to_SPI_Bridge/
├── docs/                    ← 本文件所在目录
│   └── VPlan.md
├── rtl/                     ← DUT RTL 代码
│   ├── axi_lite_slave.sv
│   ├── spi_master.sv
│   ├── async_fifo.sv
│   └── axi2spi_bridge_top.sv
├── tb/                      ← 验证环境
│   ├── axi_master_agent/    (driver/monitor/seqr)
│   ├── spi_slave_agent/     (driver/monitor/seqr)
│   ├── reg_model/           (RAL: reg_block + adapter)
│   ├── scoreboard.sv
│   ├── env.sv
│   ├── test_lib.sv          (所有 test case)
│   └── top_tb.sv
├── sim/                     ← 仿真脚本
│   ├── Makefile
│   └── waves.tcl
└── README.md
```
