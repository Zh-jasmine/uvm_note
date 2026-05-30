# AXI-Lite 地址与寄存器（tmp）

## 基本概念
- **bit（位）**：最小数据单位，0 或 1
- **byte（字节）**：8 个 bit 组成 1 个 byte
- 内存每个地址指向 1 个 byte

## AXI-Lite 的两个 "32 位"
- **地址总线 32 位**：可编址范围 0x00000000 ~ 0xFFFFFFFF（4GB），每个地址指向 1 个 byte
- **数据总线 32 位**：一次读写传 4 个 byte（32 bit）
- 一次 AXI-Lite 事务读/写连续 4 个地址的 byte

## 寄存器宽度
- AXI-Lite 数据宽度 32 位 → 寄存器也定义成 32 位宽
- 每个寄存器占 4 个地址（4 byte）

## 地址译码
- 地址本身无物理意义，由 RTL 译码器（decoder）决定映射
- RTL 里 `case (axi_awaddr[5:2])` 做地址译码
- `ADDR_LSB = (32/32) + 1 = 2`
- `OPT_MEM_ADDR_BITS = 3`

## 本 DUT 寄存器映射
| addr[5:2] | 完整地址 | 寄存器      | 位段                 | 作用                              | 读写  |
| --------- | ---- | -------- | ------------------ | ------------------------------- | --- |
| 4'h0      | 0x00 | slv_reg0 | [0] = start        | ctrl: 写 1 触发 SPI 传输             | W   |
| 4'h1      | 0x04 | slv_reg1 | [0] = busy         | SPI 忙标志（RTL 自动更新）               | R   |
| 4'h2      | 0x08 | slv_reg2 | [1:0] = spi_mode   | CPOL/CPHA: 0/1/2/3              | R/W |
| 4'h3      | 0x0C | slv_reg3 | [1:0] = sck_speed  | SCK 分频: 0=÷2, 1=÷4, 2=÷8, 3=÷16 | R/W |
| 4'h4      | 0x10 | slv_reg4 | [1:0] = word_len   | 0=32b, 1=16b, 2=8b, 3=4b        | R/W |
| 4'h5      | 0x14 | slv_reg5 | [7:0] = IFG        | 帧间隔（SCK 周期数）                    | R/W |
| 4'h6      | 0x18 | slv_reg6 | [7:0] = CS_SCK     | CS 拉低到首个 SCK 的延迟                | R/W |
| 4'h7      | 0x1C | slv_reg7 | [7:0] = SCK_CS     | 末个 SCK 到 CS 拉高的延迟               | R/W |
| 4'h8      | 0x20 | slv_reg8 | [31:0] = mosi_data | 待发送的 MOSI 数据                    | R/W |
| 4'h9      | 0x24 | slv_reg9 | [31:0] = miso_data | 接收到的 MISO 数据（RTL 自动更新）          | R   |

地址换算：`addr[5:2] = N` → 完整地址 = `N × 4`

## 本 DUT test 的写序列
1. 写 `0x10` → `slv_reg4[1:0] = 2` → 8-bit 模式
2. 写 `0x20` → `slv_reg8 = 0xAB` → mosi_data
3. 写 `0x00` → `slv_reg0[0] = 1` → start bit，触发 SPI 发送

## verilog 数值写法
```
4'd1  →  4-bit, 十进制 1
4'h1  →  4-bit, 十六进制 1
4'b1  →  4-bit, 二进制 0001
```

---

*2026-05-28 临时笔记，待整理进正式文档*
