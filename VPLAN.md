# AXI-LITE2SPI 验证计划 (VPlan)

> 方法:按「**接口 × 维度**」网格列点,保证不漏——每个接口都要过一遍
> 功能 / 协议合规 / 边界 / 异常 / 复位。
> 图例:✅ 完成 ｜ ⚠ 部分或待确认 ｜ ❌ 缺(计划补)

---

## 一、AXI-Lite Slave 接口

### 功能
| 检查点 | 现状 | 实现 / 备注 |
|---|---|---|
| 寄存器写 | ✅ | 各 test 经 `axi_spi_cfg_seq` 写配置 |
| 寄存器读回 | ⚠ | 仅 `slv_reg1`(busy)、`slv_reg9`(miso) 可读;`slv_reg0/2~8` 为 write-only(读 case 注释掉,返回 0)。**待确认 spec:这些配置寄存器是否应可读回** |

### 协议合规
| 检查点                          | 现状  | 实现 / 计划                                                                               |
| ---------------------------- | --- | ------------------------------------------------------------------------------------- |
| 三种握手时序(VALID先 / READY先 / 同时) | ❌   | driver 单一时序 → 覆盖率 **W6** 洞;计划增强 AXI driver                                            |
| WSTRB 字节选通                   | ❌   | `axi_write_seq` 固定 `4'hF`;DUT 标准支持逐字节选通;计划借 **RAL mirror** 验(slv_reg8 读不回,只能 SPI 路径验) |
| 读写并发(5 通道独立)                 | ❌   | 计划 driver fork 读写;另需确认 DUT 是否支持 outstanding                                           |
| 写/读响应 BRESP/RRESP            | ⚠   | DUT 固定 OKAY,未专门断言                                                                     |

### 边界 / 异常
| 检查点 | 现状 | 备注 |
|---|---|---|
| 非法地址写 | ⚠ | DUT default 保持原值(不可达,COVERAGE waiver W4) |
| 非法地址读 | ⚠ | DUT default 返回 0 |

### 复位
| 检查点                 | 现状  | 实现                            |
| ------------------- | --- | ----------------------------- |
| 复位时 ready/valid 全清零 | ✅   | **`AXI_RESET_CHK` SVA(本次新增)** |
| 复位时寄存器清零            | ✅   | RTL reset 块                   |

---

## 二、SPI Master 接口

### 功能
| 检查点 | 现状 | 实现 |
|---|---|---|
| spi_mode / word_len / sck_speed | ✅ | covergroup 100% + 定向 + 随机 |
| data(含全 0 / 全 1) | ✅ | `axi_data_zero/ff_test` + 随机 |
| MISO 读回 | ✅ | `axi_miso_tests` |
| busy 标志 | ✅ | `axi_busy_test` |

### 协议合规
| 检查点 | 现状 | 实现 |
|---|---|---|
| CS / SCK / MOSI 时序 | ✅ | SVA: `cs_sck_chk` / `sck_period_chk` / `SCK_CS_CHK` |
| CS_SCK / SCK_CS / IFG 延迟 | ✅ | SVA + 定向 timing tests |

### 边界 / 异常 / 复位
| 检查点 | 现状 | 实现 |
|---|---|---|
| 4 档 word_len 边界 | ✅ | `axi_write_word_len_test` |
| 传输中复位 + 恢复 | ✅ | `axi_reset_mid_test` |
| SPI FSM 复位 | ✅ | FSM 回 IDLE(覆盖率确认) |

---

## 三、寄存器抽象 (RAL)
| 检查点 | 现状 | 计划 |
|---|---|---|
| reg model / adapter / predictor | ❌ | 建 `uvm_reg_block` + adapter + predictor |
| 内建 reg test | ❌ | `uvm_reg_hw_reset_test`(复位值)、`uvm_reg_bit_bash`(逐位读写) |

---

## 四、覆盖率
| 类型 | 现状 | 出处 |
|---|---|---|
| 功能覆盖率 | ✅ 100% | `cg_spi_frame`(mode×wlen / mode×speed cross) |
| 代码覆盖率 | ✅ DUT line/branch ~90% + waiver | COVERAGE.md |
| 断言覆盖率 | ✅ 100% | |

---

## 待办(按优先级)
1. ✅ 复位 AXI 协议 SVA(本次完成)
2. ⬜ 握手三时序 —— 增强 AXI driver,填覆盖率 W6
3. ⬜ RAL —— reg model + adapter + 内建 reg test
4. ⬜ WSTRB 字节选通 —— 借 RAL mirror 验
5. ⬜ 读写并发 —— driver fork + 确认 DUT 支持度
6. ⬜ write-only 寄存器读回 —— 与 spec 确认后定豁免或补测
