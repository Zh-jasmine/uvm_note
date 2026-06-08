# AXI-LITE2SPI 项目 — CLAUDE.md

## 本文件自动维护规则（给 Claude 看，无需用户提示）

会话进行中，满足以下任一条件时**主动用 Edit 更新本文件**，更新后简短说一句「已同步 CLAUDE.md：xxx」：
1. 一个 test 编译通过且 PASS → 改「测试清单」表格状态
2. 一个 bug 定位+修复 → 更新「最新进展」+ 勾掉「已知问题」对应项
3. 新增/删测试、新增模块、改 DUT 接口、寄存器映射变动 → 同步对应章节
4. 仿真命令、Makefile target、目录结构变动
5. 阶段性签核（覆盖率、回归全过等）

**不要**为以下情况更新：小代码改动（typo/注释/单条 SVA 微调）、探索中没定型、用户只是问问题/看代码/调环境。

## 项目概况

这是一个 **AXI4-Lite 到 SPI 总线桥接 IP**，包含 RTL 设计 + 完整 UVM 验证平台。运行在 VMware CentOS 7.9 虚拟机中（IC 虚拟机），使用 Synopsys VCS O-2018.09-SP2 仿真。

## 虚拟机

- VM 名称：IC（`IC.vmx`）
- OS：CentOS Linux release 7.9.2009 (Core)，64-bit
- CPU：6 核（coresPerSocket=2），内存：8GB
- 共享文件夹：宿主机 `C:\Users\jasmine\Desktop\VMware\VMware share` ↔ 客户机 `/home/zh/vmshare/`
- 客户机 IP：`192.168.91.25`，用户 `zh`
- 仿真目录：`/home/zh/vmshare/AXI-LITE2SPI/UVMTB/sim/`

## DUT 架构（3 层）

| 层级 | 文件 | 职能 |
|------|------|------|
| `DUT/AXI_slave_top.v` | 顶层 wrapper | 例化 `AXI_SPI_n_regs`，透传 AXI 端口，暴露 SPI 引脚 |
| `DUT/AXI_SPI_n_regs.v` | 寄存器 + SPI 控制 | AXI-Lite slave 协议（地址锁存、读写握手、写响应），10 个寄存器（slv_reg0~9），例化 `SPI_master` |
| `DUT/SPI_master.v` | SPI 控制器 FSM | 5 状态：IDLE→INTERFRAME→PRE_TRANS→TRANSACTION→FINISH；支持 4 种 SPI mode、4 档 SCK 速度、4 种字长 |

`AXI_SPI_n_regs.v` 内部用 `` `include `` 包含 `SPI_master.v`；`AXI_slave_top.v` 用 `` `include `` 包含 `AXI_SPI_n_regs.v`。

## 寄存器映射

| 寄存器 | 地址偏移 | 类型 | 功能 |
|--------|----------|------|------|
| slv_reg0[0] | 0x00 | W | `start_i` — SPI 传输启动（写 0→1 产生上升沿，触发一次传输） |
| slv_reg1[0] | 0x04 | R | `busy_o` — 只读 busy 标志（传输中 = 1，空闲 = 0） |
| slv_reg2[1:0] | 0x08 | W | `spi_mode_i` — CPOL/CPHA（0/1/2/3） |
| slv_reg3[1:0] | 0x0C | W | `sck_speed_i` — SCK 分频（0=/128, 1=/64, 2=/32, 3=/16 of GCLK） |
| slv_reg4[1:0] | 0x10 | W | `word_len_i` — 字长（0=32b, 1=16b, 2=8b, 3=4b） |
| slv_reg5[7:0] | 0x14 | W | `IFG_i` — inter-frame gap（最小帧间隔，以 GCLK 周期计） |
| slv_reg6[7:0] | 0x18 | W | `CS_SCK_i` — CS 使能到首个 SCK 边沿的延迟 |
| slv_reg7[7:0] | 0x1C | W | `SCK_CS_i` — 最后 SCK 边沿到 CS 释放的延迟 |
| slv_reg8[31:0] | 0x20 | W | `mosi_data_i` — 待发送的 SPI 数据（高 bit 先出） |
| slv_reg9[31:0] | 0x24 | R | `miso_data_o` — 接收到的 SPI 数据 |

slv_reg1 和 slv_reg9 的 AXI 写通道被注释掉，改为硬件自动更新（`busy_flag → slv_reg1`，`miso_data → slv_reg9`）。slv_reg0（start）写入时用写 strobe（wstrb=4'hF），但 FSM 只关心 bit0 的上升沿，因此发送前需要先写 0 再写 1。

## SPI_master FSM 关键点

- 5 状态：`IDLE → INTERFRAME → PRE_TRANS → TRANSACTION → FINISH → IDLE`
- `trans_start_v = start_i & !d_ff_v`（上升沿检测），只在 IDLE 状态触发
- `trans_done_v = (bit_cnt_v - chosen_word_len_v) == 0`
- **Mode 3 (CPOL=1,CPHA=1)** 的 TRANSACTION 中有特殊的 `last_bit_v` 机制（延迟一个 SCK 边沿进入 FINISH），但 Mode 0 中没有，需确认 Mode 0 的最后 bit 时序是否正确
- 每帧结束后自动回到 IDLE，可被下一帧 `start_i` 上升沿唤醒

## UVM 验证平台

```
tb_top.sv
├── AXI agent (active)          — 驱动 AXI-Lite 写/读事务
│   ├── axi_driver.sv           — 执行 AXI 总线时序
│   ├── axi_monitor.sv          — 监测 AXI 事务 → 发送到 scoreboard
│   ├── axi_sequencer.sv
│   ├── axi_seq_item.sv         — 事务对象（addr, wdata, wstrb, write flag, rdata）
│   └── axi_sequence_lib.sv     — 基础 sequences：axi_write_seq, axi_read_seq, axi_spi_cfg_seq
├── SPI agent (passive)         — 仅监听 SPI 总线，不驱动
│   ├── spi_monitor.sv          — 等 CS↓ → 按 word_len 采 num_bits → 发送到 scoreboard（fork 监听 ARESETN，reset 中止当前帧）
│   ├── spi_seq_item.sv
│   └── spi_interface.sv
├── SVA 模块（独立文件，tb_top 例化）
│   ├── spi/spi_cs_sck_sva.sv   — CS→SCK 延迟 `CS_SCK_CHK`/`CS_SCK_COV`（generate 3 档: EXP=2,4,8）
│   ├── spi/spi_sck_speed_sva.sv — SCK 速度 `SCK_SPEED_CHK`/`SCK_SPEED_COV`（generate 4 档: PER=128,64,32,16）
│   ├── spi/spi_sck_cs_sva.sv   — SCK→CS 延迟 `SCK_CS_CHK`（计数器方式）
│   ├── spi/spi_reset_sva.sv    — SPI 复位引脚 `SPI_RESET_CHK`
│   └── axi/axi_reset_sva.sv    — AXI 复位 `AXI_RESET_CHK` + `AXI_RESET_RELEASE_CHK`
├── tb_scoreboard.sv            — AXI 写期望 vs SPI 监测实际，比对
├── tb_environment.sv           — 连接 agent / monitor / scoreboard
└── test_base.sv                — UVM test 基类（创建 config，获取 vif，创建 env）
```

### Scoreboard 比对机制

1. AXI monitor 侦听 AXI 写操作 → 对 slv_reg8（mosi_data）写入按当前 word_len mask 后 push 到 `expected_data_q[$]`
2. Scoreboard 同时动态跟踪 slv_reg2（spi_mode）、slv_reg4（word_len）的 AXI 写入，同步给 `spi_cfg`，保证 SPI monitor 用正确的位数采集
3. SPI monitor 在 CS↓ 后按 spi_mode 对应的边沿采集 num_bits 位 MOSI → 打包 `spi_seq_item` → 发到 scoreboard
4. Scoreboard 收到 SPI 数据后从 `expected_data_q` pop 期望值比对
5. `report_phase` 输出统计和 PASS/FAIL（Makefile 的 `all_test` 用 grep `UVM_ERROR ... 0` && `UVM_FATAL ... 0` 判断测试通过，覆盖所有 error 来源）

### axi_spi_cfg_seq 使用方法

```systemverilog
axi_spi_cfg_seq seq = axi_spi_cfg_seq::type_id::create("seq");
// 开关控制写哪些寄存器
seq.cfg_word_len  = 1;  seq.word_len_enc  = 2'b10;   // 8-bit
seq.cfg_spi_mode  = 1;  seq.spi_mode_enc  = 2'b00;   // Mode 0
seq.cfg_sck_speed = 1;  seq.sck_speed_enc = 2'b11;   // /16
seq.cfg_cs_sck    = 1;  seq.cs_sck        = 8'h4;
seq.cfg_sck_cs    = 1;  seq.sck_cs        = 8'h4;
seq.cfg_ifg       = 1;  seq.ifg           = 8'h4;
seq.cfg_mosi_data = 1;  seq.mosi_data     = 32'hA5;
seq.do_start      = 1;  seq.wait_ns       = 20000;
seq.start(m_sequencer);
```

cfg_* 设为 0 时跳过对应寄存器写，可将配置和触发分两次调用实现"配一次、发多帧"。

## 测试清单（VPlan）

| Section | 测试 | 描述 | 状态 |
|---------|------|------|------|
| 1 | `spi_word_len_test` | 8b/16b/32b/4b 写入验证 | ✓ |
| 2 | `spi_mode_test` | SPI Mode 0/1/2/3 | ✓ |
| 3.1 | `spi_cs_sck_test` | CS→SCK 延迟 | ✓（有 Bug7 根因） |
| 3.2 | `spi_sck_cs_test` | SCK→CS 延迟 | ✓ |
| 3.3 | `spi_ifg_test` | 帧间隔 IFG | ✓ |
| 3.4 | `spi_sck_speed_test` | SCK 分频 | ✓ |
| 4.1 | `spi_miso_read_test` | MISO 回读 0x5A | ✓ PASS |
| 4.2 | `spi_miso_all_0_test` | MISO 全 0 (0x00) | ✓ PASS |
| 4.3 | `spi_miso_all_1_test` | MISO 全 1 (0xFF) | ✓ PASS |
| 5.1/5.2 | `axi_busy_test` | busy 标志 | ✓ |
| 6.1 | `spi_burst_test` | 4 帧背靠背 | ✓ |
| 6.2 | `spi_alternating_test` | 0x55/0xAA 交替 | ✓ |
| 7.1 | `spi_data_all_0_test` | 全 0 数据 | ✓ PASS |
| 7.2 | `spi_data_all_1_test` | 全 1 数据 | ✓ PASS |
| 7.3 | `reset_mid_test` | 传输中复位 | ✓ PASS |
| 6.x | `spi_random_test` | 80 帧约束随机回归（mode×wlen×speed×data） | ✓ PASS |
| 一.复位 | `reset_sva_test` | 复位 SVA 多场景（空闲/写途中/传输中/背靠背/恢复） | ✓ PASS |

## 最新仿真结果（2026-06-08）

**17/17 全 PASS**，零 mismatch，零 assertion failure。

### 覆盖率（URG 报告 `UVMTB/sim/cov_report/`）
| 类型 | 结果 |
|---|---|
| 功能覆盖率 (GROUP) | **100%**（`cg_spi_frame` mode×wlen / mode×speed cross 各 16/16）|
| 断言覆盖率 (ASSERT) | **100%**（我们的 12 条 SVA 全 success；2 条 UVM 库内部断言无 attempt 不计入）|
| Cover property | **7/7 match > 0**（CS_SCK_COV × 3 + SCK_SPEED_COV × 4）|
| DUT 代码覆盖率 | line 90% / branch 90% + 7 条 waiver（W1–W7，详见 `cov_report/COVERAGE.md`）|

### 关键修复记录
- **`reset_mid_test` 修复**：SPI monitor reset 感知（fork + disable）+ scoreboard FIFO flush + 恢复帧全配置
- **`AXI_RESET_RELEASE_CHK` 修复**：加前提 `!AWVALID && !WVALID && !ARVALID`，避免 master 悬挂事务引发误报

## 仿真命令

```shell
# 位于 UVMTB/sim/
make comp              # 仅编译（VCS O-2018.09-SP2，UVM-1.2）
make sim TEST=xxx      # 编译 + 运行单个 test
make all_test          # 批量运行 17 个 test + 覆盖率报告
make clean             # 清理
make cov               # 单独生成 URG 覆盖率报告

# 可用 TEST 值（17 个）：
# spi_word_len_test, spi_mode_test, spi_cs_sck_test, spi_sck_cs_test,
# spi_ifg_test, spi_sck_speed_test, axi_busy_test, spi_burst_test,
# spi_alternating_test, spi_data_all_0_test, spi_data_all_1_test,
# reset_mid_test, spi_miso_read_test, spi_miso_all_0_test,
# spi_miso_all_1_test, spi_random_test, reset_sva_test

# 快捷 target：make test_word_len, test_mode, test_cs_sck, ...
# 注意：改了 RTL/TB 源码后要先 make comp 重编
```

FSDB dump 到 `tb_top.fsdb`（Verdi 查看波形）。

## 待改进（按优先级）

1. **AXI 握手三时序**：driver 目前单一时序（VALID 先），未覆盖 READY 先 / 同时 → 覆盖率 W6 洞
2. **WSTRB 字节选通**：`axi_write_seq` 固定 `4'hF`，未验证逐字节写入
3. **RAL**：reg model + adapter + 内建 reg test（`uvm_reg_hw_reset_test` 等）
4. **MISO model 扩展**：当前仅 8-bit Mode0，扩到其他 word_len / mode
5. **硬编码延时改 event 等待**：`axi_busy_test` 的 `#500` → `@(posedge CS)` 等
6. **读写并发**：driver fork 读写，确认 DUT 是否支持 outstanding

### 设计意图备忘
- **slv_reg0/2~8 write-only**：DUT 读译码 case 只保留 `slv_reg1`(busy) / `slv_reg9`(miso)，配置寄存器读回返回 0（设计意图，非 bug，waiver）
- **SPI_master Mode 0 最后 bit**：Mode 3 有 `last_bit_v` 延迟机制，Mode 0 没有类似处理——已通过 17 test 全 PASS 间接验证时序正确
