# AXI-LITE2SPI 验证计划

## 0. 当前状态

| 项目 | 状态 |
|---|---|
| DUT | AXI4-Lite Slave -> SPI Master 桥接 IP |
| 验证范围 | AXI-Lite 寄存器/协议 + SPI 功能/时序 + 复位 + RAL |
| 回归规模 | `UVMTB/sim/Makefile` 中 20 个测试 |
| 主要结果 | 功能覆盖率 100%，SVA 覆盖率 100%，DUT 行/分支覆盖率约 90%，剩余项按设计意图 waiver |
| 剩余事项 | 读写并发属于可选协议压力测试，不作为当前项目完整性的阻塞项 |

图例：`完成` 表示已由回归覆盖；`豁免` 表示符合设计意图或低收益；`待办` 表示可选后续工作。

---

## 1. 覆盖矩阵

| 模块/接口 | 检查点 | 状态 | 测试 / 实现 |
|---|---|---:|---|
| AXI-Lite | 寄存器 frontdoor 写 | 完成 | 通用配置 sequence + 定向测试 |
| AXI-Lite | 可读寄存器（`busy`, `miso_data`） | 完成 | `axi_busy_test`、MISO 测试、`ral_test` RO 读 |
| AXI-Lite | write-only 寄存器行为 | 完成 | RAL frontdoor 写 + backdoor peek |
| AXI-Lite | AW/W 握手时序 | 完成 | `axi_handshake_test`：同时、AW 先到、W 先到、长延迟场景 |
| AXI-Lite | WSTRB 字节选通 | 完成 | `axi_wstrb_test`：单 byte lane + sparse mask |
| AXI-Lite | busy/status 状态 | 完成 | `axi_busy_test`：传输中 busy=1，结束后 busy=0 |
| AXI-Lite | BRESP/RRESP OKAY 路径 | 完成 | driver 捕获响应；DUT 固定返回 OKAY |
| AXI-Lite | 非法地址读写 | 豁免 | 默认保持原值 / 返回 0；不是当前项目重点 |
| AXI-Lite | 读写并发事务 | 待办 | 可选协议压力测试；当前 driver 是事务串行结构 |
| SPI | Mode 0/1/2/3 | 完成 | `spi_mode_test`、`spi_random_test` |
| SPI | 字长 32/16/8/4 | 完成 | `spi_word_len_test`、`spi_random_test` |
| SPI | SCK 四档分频 | 完成 | `spi_sck_speed_test`、SVA |
| SPI | MOSI 数据模式 | 完成 | 全 0、全 1、0x55/0xAA、随机数据 |
| SPI | MISO 回读 | 完成 | `spi_miso_read_test`、全 0、全 1 |
| SPI | 背靠背多帧 | 完成 | `spi_burst_test` |
| SPI | 约束随机回归 | 完成 | `spi_random_test`：80 帧约束随机 |
| SPI 时序 | CS->SCK 延迟 | 完成 | `spi_cs_sck_test`、`CS_SCK_CHK`、`CS_SCK_COV` |
| SPI 时序 | SCK 周期 | 完成 | `spi_sck_speed_test`、`SCK_SPEED_CHK`、`SCK_SPEED_COV` |
| SPI 时序 | SCK->CS 延迟 | 完成 | `spi_sck_cs_test`、`SCK_CS_CHK` |
| SPI 时序 | IFG 帧间隔 | 完成 | `spi_ifg_test` |
| 复位 | AXI 复位期间保持空闲 | 完成 | `axi_reset_sva.sv` |
| 复位 | SPI 引脚复位期间保持 idle | 完成 | `spi_reset_sva.sv` |
| 复位 | 传输中复位与恢复 | 完成 | `reset_mid_test`、`reset_sva_test` |
| RAL | 寄存器模型 + adapter | 完成 | `axi_spi_reg_block.sv`、`axi_spi_reg_adapter.sv` |
| RAL | frontdoor/backdoor 冒烟测试 | 完成 | `ral_test` |

---

## 2. 回归测试清单

| # | 测试名 | 目的 |
|---:|---|---|
| 1 | `spi_word_len_test` | 32/16/8/4-bit 字长 |
| 2 | `spi_mode_test` | SPI mode 0/1/2/3 |
| 3 | `spi_cs_sck_test` | CS 到 SCK 的启动延迟 |
| 4 | `spi_sck_cs_test` | SCK 到 CS 释放延迟 |
| 5 | `spi_ifg_test` | 帧间隔 IFG |
| 6 | `spi_sck_speed_test` | SCK 分频 / 周期 |
| 7 | `axi_busy_test` | busy/status 行为 |
| 8 | `axi_handshake_test` | AXI AW/W 握手时序变化 |
| 9 | `axi_wstrb_test` | WSTRB 字节选通 |
| 10 | `spi_burst_test` | SPI 背靠背多帧 |
| 11 | `spi_alternating_test` | MOSI 0x55 / 0xAA 交替模式 |
| 12 | `spi_data_all_0_test` | MOSI 全 0 数据 |
| 13 | `spi_data_all_1_test` | MOSI 全 1 数据 |
| 14 | `reset_mid_test` | 传输中复位 |
| 15 | `spi_miso_read_test` | MISO 回读 0x5A |
| 16 | `spi_miso_all_0_test` | MISO 全 0 |
| 17 | `spi_miso_all_1_test` | MISO 全 1 |
| 18 | `spi_random_test` | 80 帧约束随机回归 |
| 19 | `reset_sva_test` | 复位 SVA 多场景测试 |
| 20 | `ral_test` | RAL frontdoor/backdoor 冒烟测试 |

---

## 3. SVA 计划

| 文件 | 断言 / cover | 检查内容 |
|---|---|---|
| `UVMTB/spi/spi_cs_sck_sva.sv` | `CS_SCK_CHK`, `CS_SCK_COV` | CS 下降沿到首个 SCK 边沿的延迟 |
| `UVMTB/spi/spi_sck_speed_sva.sv` | `SCK_SPEED_CHK`, `SCK_SPEED_COV` | 不同分频档位下的 SCK 周期 |
| `UVMTB/spi/spi_sck_cs_sva.sv` | `SCK_CS_CHK` | 最后一个 SCK 边沿到 CS 释放的延迟 |
| `UVMTB/spi/spi_reset_sva.sv` | `SPI_RESET_CHK` | 复位期间 SPI 引脚保持 idle |
| `UVMTB/axi/axi_reset_sva.sv` | `AXI_RESET_CHK` | 复位期间 AXI slave 输出保持空闲 |
| `UVMTB/axi/axi_reset_sva.sv` | `AXI_RESET_RELEASE_CHK` | 复位释放后 AXI slave 继续保持空闲 |

---

## 4. RAL 计划

| 寄存器组 | 访问属性 | 验证方法 |
|---|---|---|
| `start`、配置寄存器、`mosi_data` | WO | RAL frontdoor write + backdoor peek |
| `busy` | RO | 空闲状态下 RAL frontdoor read |
| `miso_data` | RO | RAL backdoor poke/read 路径冒烟 |

实现文件：

- 寄存器模型：`UVMTB/ral/axi_spi_reg_block.sv`
- Adapter：`UVMTB/ral/axi_spi_reg_adapter.sv`
- 测试：`UVMTB/test/ral_test.sv`
- 接入方式：通过 `reg_model.map.set_sequencer(...)` 连接到现有 AXI sequencer

---

## 5. 覆盖率总结

| 覆盖类型 | 目标 | 结果 / 说明 |
|---|---:|---|
| 功能覆盖率 | 100% | `cg_spi_frame` cross bins 全命中 |
| SVA / cover property | 100% | 项目自定义断言与 cover property 已覆盖 |
| DUT 行覆盖率 | 约 90% | 剩余项记录为 waiver |
| DUT 分支覆盖率 | 约 90% | 剩余项记录为 waiver |

Waiver 示例：

- 配置类 write-only 寄存器按设计不可通过 frontdoor 读回。
- 非法地址/default 分支收益较低，当前项目中按设计意图豁免。
- UVM 库内部无 attempt 的断言不计入项目自定义 SVA 覆盖缺口。

---

## 6. 剩余 / 可选工作

| 优先级 | 项目            | 原因                                            |
| --: | ------------- | --------------------------------------------- |
|  P1 | AXI 读写并发      | 可选协议压力测试；若要真正并发，需要改 driver 架构                 |
|  P2 | MISO model 扩展 | 当前定向 MISO 测试已满足项目展示；后续可扩展到所有 mode/word_len 组合 |
|  P2 | 固定延时改事件等待     | 提升 test 鲁棒性，例如用 CS/busy 事件替代硬编码 `#` 延时        |

当前结论：作为实习导向的验证项目，关键功能、协议、时序、复位与 RAL 已覆盖。剩余项属于进一步打磨，不影响当前项目完整性。
