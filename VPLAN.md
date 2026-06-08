# AXI-LITE2SPI 验证计划 (VPlan)

> 方法：按「**接口 × 维度**」网格列点，保证不漏——每个接口都要过一遍
> 功能 / 协议合规 / 边界 / 异常 / 复位。
> 图例：✅ 完成 ｜ ⚠ 部分或待确认 ｜ ❌ 缺(计划补)

---

## 一、AXI-Lite Slave 接口

### 功能
| 检查点 | 现状 | 实现 / 备注 |
|---|---|---|
| 寄存器写 | ✅ | 各 test 经 `axi_spi_cfg_seq` 写配置 |
| 寄存器读回 | ⚠ | 仅 `slv_reg1`(busy)、`slv_reg9`(miso) 可读；`slv_reg0/2~8` 为 write-only（读 case 注释掉，返回 0）。**设计意图，非 bug**——waiver |
| busy 标志 | ✅ | `axi_busy_test`：传输中 busy=1、结束后 busy=0 |

### 协议合规
| 检查点 | 现状 | 实现 / 计划 |
|---|---|---|
| 三种握手时序(VALID先 / READY先 / 同时) | ❌ | driver 单一时序 → 覆盖率 **W6** 洞；计划增强 AXI driver |
| WSTRB 字节选通 | ❌ | `axi_write_seq` 固定 `4'hF`；DUT 标准支持逐字节选通；计划补 test |
| 读写并发(5 通道独立) | ❌ | 计划 driver fork 读写；另需确认 DUT 是否支持 outstanding |
| 写/读响应 BRESP/RRESP | ⚠ | DUT 固定 OKAY，未专门断言 |

### 边界 / 异常
| 检查点 | 现状 | 备注 |
|---|---|---|
| 非法地址写 | ⚠ | DUT default 保持原值（不可达，COVERAGE waiver W4）|
| 非法地址读 | ⚠ | DUT default 返回 0 |

### 复位
| 检查点 | 现状 | 实现 |
|---|---|---|
| 复位时 slave 输出全清零 | ✅ | `axi_reset_sva.sv` → `AXI_RESET_CHK` |
| 复位释放后空闲保持 | ✅ | `axi_reset_sva.sv` → `AXI_RESET_RELEASE_CHK`（前提：master 无悬挂事务）|
| 复位时寄存器清零 | ✅ | RTL reset 块 |

---

## 二、SPI Master 接口

### 功能
| 检查点 | 现状 | 实现 |
|---|---|---|
| word_len (32/16/8/4) | ✅ | `spi_word_len_test` + 随机 |
| SPI Mode (0/1/2/3) | ✅ | `spi_mode_test` + 随机 |
| SCK 分频 (4 档) | ✅ | `spi_sck_speed_test` + 随机 |
| MOSI 数据 (含全 0 / 全 1) | ✅ | `spi_data_all_0_test` / `spi_data_all_1_test` + 随机 |
| MOSI 交替模式 | ✅ | `spi_alternating_test` (0x55/0xAA) |
| MISO 读回 | ✅ | `spi_miso_read_test`(0x5A) / `spi_miso_all_0_test`(0x00) / `spi_miso_all_1_test`(0xFF) |
| busy 标志 | ✅ | `axi_busy_test` |
| 背靠背多帧 | ✅ | `spi_burst_test` (4 帧) |
| 约束随机回归 | ✅ | `spi_random_test` (80 帧 mode×wlen×speed×data) |

### 协议合规 / 时序
| 检查点 | 现状 | 实现 |
|---|---|---|
| CS→SCK 延迟 | ✅ | SVA `spi_cs_sck_sva.sv` → `CS_SCK_CHK` / `CS_SCK_COV` + `spi_cs_sck_test` |
| SCK→CS 延迟 | ✅ | SVA `spi_sck_cs_sva.sv` → `SCK_CS_CHK` + `spi_sck_cs_test` |
| SCK 速度(周期) | ✅ | SVA `spi_sck_speed_sva.sv` → `SCK_SPEED_CHK` / `SCK_SPEED_COV` + `spi_sck_speed_test` |
| IFG 帧间隔 | ✅ | `spi_ifg_test` |

### 复位
| 检查点 | 现状 | 实现 |
|---|---|---|
| 复位期间 SPI 引脚 idle | ✅ | SVA `spi_reset_sva.sv` → `SPI_RESET_CHK` (CS=1, SCK=0, MOSI=0) |
| 传输中复位 + 恢复 | ✅ | `reset_mid_test`：中途 reset → CS 释放 → 重新配置 → 恢复帧 scoreboard 比对 |
| 复位 SVA 多场景 | ✅ | `reset_sva_test`：5 场景（空闲 / 配置写途中 / 传输中 / 背靠背 / 恢复帧）|
| SPI FSM 回 IDLE | ✅ | FSM 覆盖率确认 |

---

## 三、SVA 断言清单

| SVA 模块文件 | 断言标签 | 检查内容 |
|---|---|---|
| `spi/spi_cs_sck_sva.sv` | `CS_SCK_CHK` / `CS_SCK_COV` | CS↓ 到首个 SCK 沿的延迟（generate 3 档: EXP=2,4,8）|
| `spi/spi_sck_speed_sva.sv` | `SCK_SPEED_CHK` / `SCK_SPEED_COV` | SCK 上升沿间隔（generate 4 档: PER=128,64,32,16）|
| `spi/spi_sck_cs_sva.sv` | `SCK_CS_CHK` | 最后 SCK 沿到 CS↑ 的延迟（计数器方式）|
| `spi/spi_reset_sva.sv` | `SPI_RESET_CHK` | 复位期间 CS=1, SCK=0, MOSI=0 |
| `axi/axi_reset_sva.sv` | `AXI_RESET_CHK` | 复位期间 AXI slave 输出全清零 |
| `axi/axi_reset_sva.sv` | `AXI_RESET_RELEASE_CHK` | 复位释放后 slave 输出保持空闲（前提：master 无悬挂事务）|

---

## 四、测试清单

| # | 测试名 | 描述 | 状态 |
|---|--------|------|------|
| 1 | `spi_word_len_test` | 8b/16b/32b/4b 字长 | ✅ |
| 2 | `spi_mode_test` | SPI Mode 0/1/2/3 | ✅ |
| 3.1 | `spi_cs_sck_test` | CS→SCK 延迟 | ✅ |
| 3.2 | `spi_sck_cs_test` | SCK→CS 延迟 | ✅ |
| 3.3 | `spi_ifg_test` | 帧间隔 IFG | ✅ |
| 3.4 | `spi_sck_speed_test` | SCK 分频 | ✅ |
| 4.1 | `spi_miso_read_test` | MISO 回读 0x5A | ✅ |
| 4.2 | `spi_miso_all_0_test` | MISO 全 0 (0x00) | ✅ |
| 4.3 | `spi_miso_all_1_test` | MISO 全 1 (0xFF) | ✅ |
| 5 | `axi_busy_test` | busy 标志 (传输中=1, 空闲=0) | ✅ |
| 6.1 | `spi_burst_test` | 4 帧背靠背 | ✅ |
| 6.2 | `spi_alternating_test` | 0x55/0xAA 交替 | ✅ |
| 6.x | `spi_random_test` | 80 帧约束随机回归 | ✅ |
| 7.1 | `spi_data_all_0_test` | 全 0 数据 | ✅ |
| 7.2 | `spi_data_all_1_test` | 全 1 数据 | ✅ |
| 7.3 | `reset_mid_test` | 传输中复位 + 恢复 | ✅ |
| 8 | `reset_sva_test` | 复位 SVA 5 场景 | ✅ |

> 17/17 全 PASS（2026-06-08）

---

## 五、覆盖率

| 类型 | 结果 | 出处 |
|---|---|---|
| 功能覆盖率 (GROUP) | **100%** | `cg_spi_frame`（mode×wlen / mode×speed cross 各 16/16）|
| 断言覆盖率 (ASSERT) | **100%**（我们的 12/12 SVA 全 success，2 个 UVM 库内部断言无 attempt 不计入）| URG asserts.html |
| DUT 代码覆盖率 | line 90% / branch 90% + 7 条 waiver | COVERAGE.md |
| Cover property | **7/7 match > 0** | CS_SCK_COV × 3, SCK_SPEED_COV × 4 |

---

## 六、待办（按优先级）

1. ⬜ 握手三时序 —— 增强 AXI driver，填覆盖率 W6
2. ⬜ WSTRB 字节选通 —— 补 test
3. ⬜ RAL —— reg model + adapter + 内建 reg test
4. ⬜ 读写并发 —— driver fork + 确认 DUT 支持度
5. ⬜ MISO model 扩展 —— 当前仅 8-bit Mode0，扩到其他 word_len/mode
6. ⬜ 硬编码延时改 event 等待 —— `#500` → `@(posedge CS)` 等
