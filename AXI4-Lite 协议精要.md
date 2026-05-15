---
tags: #AMBA #AXI #协议
---

# AXI4-Lite 协议精要

> 针对 AXI4-Lite to SPI Bridge 验证项目的协议笔记。
> 参考 RTL: `AXI_SPI_n_regs.v` (Vivado 模板)

---

## 1. 这一层加了什么

AXI4-Lite 在 CPU/主设备 和 外设从设备 之间定义了一套**标准化的读写接口**。它的核心是：所有通信通过五条独立的通道完成，每个通道使用 **VALID-READY 握手** 来控制数据传输。

相比更简单的 APB（只靠 PSEL/PENABLE 两个控制信号），AXI4-Lite 的通道分离让读写可以**并行**进行，且每个通道独立流水。

---

## 2. 五通道结构

| 通道方向 | 通道名 | 用途 | 关键信号 |
|----------|--------|------|----------|
| 写地址 (Master→Slave) | AW | 携带写地址 | AWVALID, AWREADY, AWADDR |
| 写数据 (Master→Slave) | W | 携带写数据 | WVALID, WREADY, WDATA, WSTRB |
| 写响应 (Slave→Master) | B | 返回写结果 | BVALID, BREADY, BRESP |
| 读地址 (Master→Slave) | AR | 携带读地址 | ARVALID, ARREADY, ARADDR |
| 读数据 (Slave→Master) | R | 返回读数据 | RVALID, RREADY, RDATA, RRESP |

**AXI4-Lite vs AXI4-Full 的区别：**
- AXI4-Lite 不支持 burst（每笔事务只传 1 个数据）
- AXI4-Lite 没有 ID 信号（不支持乱序）
- AXI4-Lite 数据宽度固定为 32/64bit，常见是 32bit

---

## 3. VALID-READY 握手

握手是 AXI 最核心的机制。VALID 表示发送方数据已就绪，READY 表示接收方可以接收。两者**同时为高**时传输发生。

### 三种握手依赖关系

**类型 A：VALID 先有效，Slave 再拉 READY**

```
CLK      ▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇
VALID    ───────▄▄▄▄▄▄──────────────
READY    ─────────────▄▄▄▄▄▄────────
DATA     ───────████████████────────
```

**类型 B：READY 先有效（Slave 提前准备好），Master 再拉 VALID**

```
CLK      ▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇
READY    ────▄▄▄▄▄▄────────────────
VALID    ────────────▄▄▄▄▄▄────────
DATA     ────────────████████───────
```

**类型 C：同一拍同时拉高**

```
CLK      ▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇
VALID    ───────▄▄▄▄▄▄──────────────
READY    ───────▄▄▄▄▄▄──────────────
DATA     ───────████████─────────────
```

对于验证工程师：**三种场景都要覆盖**。BFM 的 driver 应该能随机产生这三种情况。

### 依赖规则
- VALID 拉高后，在对应 READY 拉高**之前不能撤**
- READY 可以等 VALID，也可以不等（类型 B）

---

## 4. 写事务完整流程（对照 RTL）

### 写地址通道

RTL 第 150 行：

```verilog
if (~axi_awready && S_AXI_AWVALID && S_AXI_WVALID && aw_en)
    axi_awready <= 1'b1;
```

这个实现要求 AWVALID 和 WVALID **同时有效** Slave 才响应地址。这是 Vivado 模板的保守设计——不违反协议（因为 AXI4-Lite 不要求地址和数据独立到达），但验证时需要注意这个行为。

### 写数据通道

WREADY 与 AWREADY 同一拍产生。

### 写响应通道

RTL 第 182-190 行：

```verilog
axi_bvalid <= axi_awready && S_AXI_AWVALID && axi_wready && S_AXI_WVALID;
```

写事务完成后，Slave 拉 BVALID 通知 Master。Master 通过 BREADY 表示收到。BRESP 固定返回 `2'b00`（OKAY）。

### 完整写时序

```
CLK    ▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄
AWADDR ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀ADDR▀▀▀▀▀▀▀▀▀▀▀▀
AWVALID ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▄▄▄▄▀▀▀▀▀▀▀▀
AWREADY ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▄▄▄▄▀▀
WDATA   ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀DATA▀▀▀▀▀▀▀▀▀▀▀
WVALID  ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▄▄▄▄▀▀▀▀▀▀▀▀
WREADY  ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▄▄▄▄▀▀
BVALID  ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▄
BREADY  ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▄
```

关键点：地址和数据同拍到达，下一拍返回写响应。

---

## 5. 读事务完整流程（对照 RTL）

RTL 第 220-240 行：

```verilog
if (~axi_arready && S_AXI_ARVALID)
    axi_arready <= 1'b1;
```

读地址通道 ARREADY 不依赖写通道的信号，独立响应。

读响应通道使用组合逻辑直接输出数据：

```verilog
always @(*) begin
    case (axi_araddr[ADDR_LSB+OPT_MEM_ADDR_BITS:ADDR_LSB])
        3'h0: reg_data_out <= slv_reg0;
        3'h1: reg_data_out <= slv_reg1;
        // ...
        default: reg_data_out <= 0;
    endcase
end
```

### 完整读时序

```
CLK    ▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇▄▄▄▇▇▇
ARADDR ▀▀▀▀▀▀▀▀▀▀▀▀▀ADDR▀▀▀▀▀▀▀▀▀▀▀▀
ARVALID ▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▀▀▀▀▀▀▀▀▀▀▀▀
ARREADY ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄▄▄▄▀▀▀▀▀▀
RDATA   ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀DATA▀
RVALID  ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄
RREADY  ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▄▄▄▄▄
```

---

## 6. 地址映射与对齐

地址总线宽度：`s00_axi_awaddr [5:0]`，即 6bit（64 字节空间）。

由于数据总线宽度为 32bit（4 字节），地址按字节寻址但每次传输 4 字节。地址的最低 2bit `[1:0]` 为 0（对齐到字地址）。

参数定义：
- `ADDR_LSB = 2`（字节偏移位数）
- `OPT_MEM_ADDR_BITS = 3`（寄存器寻址位数，$2^3=8$ 个 32bit 寄存器）

地址与寄存器的对应关系：

| 地址 | 寄存器 | 属性 | 功能 |
|------|--------|------|------|
| 0x00 | slv_reg0 | 可读写 | SPI 控制（bit0=start, bit1=busy） |
| 0x04 | slv_reg1 | 可读写 | SPI 配置（divider） |
| 0x08 | slv_reg2 | 可读写 | SPI 模式（CPOL, CPHA） |
| 0x0C | slv_reg3 | 可读写 | SPI 控制续 |
| 0x10 | slv_reg4 | 可读写 | SPI 控制续 |
| 0x14 | slv_reg5 | 可读写 | 帧间隙配置 |
| 0x18 | slv_reg6 | 可读写 | CS-SCK timing |
| 0x1C | slv_reg7 | 可读写 | SCK-CS timing |
| 0x20 | slv_reg8 | 可读写 | MOSI 数据（发送） |
| 0x24 | slv_reg9 | 只读 | MISO 数据（接收） |

---

## 7. 验证关注点

### 握手覆盖
- AWVALID 先到，WVALID 后到（你 DUT 的处理方式）
- WVALID 先到，AWVALID 后到
- 同一拍同时到达（类型 C）
- VALID 拉高后在中途撤掉（协议违规，用于验证 Slave 是否容错）

### WSTRB 字节选通
AXI4-Lite 支持按字节写使能。如果 WSTRB ≠ 4'b1111，只有对应字节被写入。验证时需要覆盖：
- 全字节写（WSTRB=4'b1111）
- 单字节写（如 WSTRB=4'b0001，只写最低字节）
- 多字节但非全字节（如 WSTRB=4'b0011）

### 读写重叠
AXI4-Lite 五通道独立，读和写可以同时进行：
- AW 和 AR 可以在同一拍有效
- B 和 R 可以在同一拍返回
- 验证需要覆盖同时读写的场景

### 复位行为
- ARESETN 拉低时，所有 READY/VALID 必须清零
- 复位释放后，Slave 回到 idle 状态，不响应任何事务
