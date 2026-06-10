# UVM RAL (Register Abstraction Layer) 知识点

> 基于 `axi_spi_reg_block.sv` / `axi_spi_reg_adapter.sv` / `ral_test.sv` 的实战对照。
> 代码路径：`UVMTB/ral/` + `UVMTB/test/ral_test.sv`
> DUT 寄存器：10 个 slv_reg，地址 0x00 ~ 0x24，32bit

---

## 1. RAL 是什么

RAL = UVM 官方提供的寄存器抽象层。它把 DUT 里的硬件寄存器建模成 UVM 对象，让 test 用 `reg.read() / reg.write()` 访问寄存器，**不需要在 test 里手写 AXI 序列**。

```
RAL 之前（spi_cfg_seq 的写法）:
  start_item(item);
  item.addr = 32'h18; item.wdata = 4; item.write = 1;
  finish_item(item);
  → 用 axi_seq_item 直接驱动总线

RAL 之后:
  reg_model.cs_sck.write(status, 4);
  → 读写走 reg 对象，底层由 adapter 自动转成 axi_seq_item
```

---

## 2. RAL 三层架构

```
┌─────────────────────────────────────────┐
│  ral_test.sv                            │
│  test 通过 reg 对象读写寄存器            │
├─────────────────────────────────────────┤
│  axi_spi_reg_block.sv                   │
│  10 个 reg 对象 + 地址映射 + hdl_path   │
├─────────────────────────────────────────┤
│  axi_spi_reg_adapter.sv                 │
│  reg2bus: RAL 请求 → axi_seq_item       │
│  bus2reg: axi_seq_item → RAL 结果       │
├─────────────────────────────────────────┤
│  axi_driver → AXI 总线 → DUT            │
└─────────────────────────────────────────┘
```

---

## 3. 三要素

### 3.1 `uvm_reg` — 单个寄存器

每个寄存器定义一个 class，继承 `uvm_reg`：

```systemverilog
class reg_cs_sck extends uvm_reg;
    `uvm_object_utils(reg_cs_sck)
    rand uvm_reg_field cs_sck;          // 寄存器里的"字段"

    function new(string name = "reg_cs_sck");
        super.new(name, 32, UVM_NO_COVERAGE);   // 32bit, 不生成覆盖率
    endfunction

    virtual function void build();
        cs_sck = uvm_reg_field::type_id::create("cs_sck");
        // configure(parent, bit宽, lsb位置, access, volatile, reset值)
        cs_sck.configure(this, 8, 0, "WO", 0, 8'h0, 1, 1, 1);
    endfunction
endclass
```

**关键参数**：

| 字段         | 你在项目里用到的值          | 含义               |
| ---------- | ------------------ | ---------------- |
| `size`     | 8 / 2 / 1 / 32     | 该字段的 bit 位宽      |
| `lsb_pos`  | 0                  | 最低位在寄存器的位置       |
| `access`   | `WO` / `RO` / `RW` | **必须跟 DUT 实情一致** |
| `volatile` | WO=0, RO=1         | 读操作前是否重新从硬件取     |
| `reset`    | 对应复位值              | DUT 复位后寄存器值      |
|            |                    |                  |

**你项目里的 access policy — 这决定了 RAL 能不能跑通**：

| 地址        | 寄存器                  | access | 原因                          |
| --------- | -------------------- | ------ | --------------------------- |
| 0x00      | start                | **WO** | DUT 读译码只保留 0x04/0x24，其余读回 0 |
| 0x04      | status               | **RO** | 硬件更新 busy 标志                |
| 0x08~0x20 | spi_mode ~ mosi_data | **WO** | 写得进，但 frontdoor 读不回（读回 0）   |
| 0x24      | miso_data            | **RO** | 硬件更新接收数据                    |

WO 的寄存器必须用 **backdoor peek** 才能验证是否写入成功。

---

### 3.2 `uvm_reg_block` — 寄存器集合模块

```systemverilog
class axi_spi_reg_block extends uvm_reg_block;
    rand reg_cs_sck cs_sck;   // 声明所有 reg 对象
    uvm_reg_map map;           // 地址映射表

    virtual function void build();
        // ① 创建 + build 每个 reg
        cs_sck = reg_cs_sck::type_id::create("cs_sck");
        cs_sck.build();
        // ② configure：把 reg 挂到 block 上
        cs_sck.configure(this, null, "");
        // ③ 地址映射
        map = create_map("map", 32'h0, 4, UVM_LITTLE_ENDIAN, 1);
        map.add_reg(cs_sck, 32'h18, "WO");
        // ④ backdoor HDL 路径
        add_hdl_path("tb_top.DUT.AXI_SPI_n_regs0");
        cs_sck.add_hdl_path_slice("slv_reg6", 0, 32);
        lock_model();  // build 结束后锁定
    endfunction
endclass
```

`lock_model()` 很重要——建完之后调用，之后不能再改结构。

**背 景：add_hdl_path_slice — backdoor 能否工作的关键**：

```systemverilog
add_hdl_path("tb_top.DUT.AXI_SPI_n_regs0");          // block 级别的 root
cs_sck.add_hdl_path_slice("slv_reg6", 0, 32);        // reg 级别的 slice
```

完整 hdl 路径 = `tb_top.DUT.AXI_SPI_n_regs0.slv_reg6`。

这样 peek/poke 底层调的就是 `uvm_hdl_read("tb_top.DUT.AXI_SPI_n_regs0.slv_reg6", val)` 和 `uvm_hdl_deposit(...)`，**不走总线，直接物理读写信号**。

---

### 3.3 `uvm_reg_adapter` — 总线协议转换器

```systemverilog
class axi_spi_reg_adapter extends uvm_reg_adapter;
    // RAL 写请求 → axi_seq_item
    virtual function uvm_sequence_item reg2bus(const ref uvm_reg_bus_op rw);
        item.write = (rw.kind == UVM_WRITE);
        item.addr  = rw.addr;
        item.wdata = rw.data;
        item.wstrb = 4'hF;       // reg 级写：全字节
        return item;
    endfunction

    // axi_seq_item → RAL 读结果
    virtual function void bus2reg(uvm_sequence_item bus_item,
                                   ref uvm_reg_bus_op rw);
        rw.data = item.rdata;     // 读返回数据
        rw.status = UVM_IS_OK;
    endfunction
endclass
```

两个设置：

| 字段 | 值 | 含义 |
|------|-----|------|
| `supports_byte_enable = 0` | 0 | RAL 不控制 wstrb（reg 级写固定全字节） |
| `provides_responses = 0` | 0 | driver 在同一个 req 上回填 rdata，不需独立 rsp |

---

## 4. 前门 (Frontdoor) vs 后门 (Backdoor)

这是 RAL 最重要的概念。

|           | **前门**              | **后门**                                 |
| --------- | ------------------- | -------------------------------------- |
| 怎么访问      | 走 AXI 总线（driver 驱动） | `uvm_hdl_read/deposit`（DPI 直接读 DUT 信号） |
| 耗时        | 需要 AXI 握手，慢         | 零延时，仿真时间不前进                            |
| 能读 WO 寄存器 | ❌ 读回 0（DUT 行为）      | ✅ 读到物理值                                |
| 应用场景      | 正常读写验证              | 复位检查 / 覆盖率采样 / 快速设置                    |

```systemverilog
// 前门写：
reg_model.cs_sck.write(status, 4);   // → reg2bus → AXI driver → DUT

// 后门读：
reg_model.cs_sck.peek(status, val);   // → uvm_hdl_read 直接读 DUT 信号

// 后门写：
reg_model.cs_sck.poke(status, 4);      // → uvm_hdl_deposit 直接写 DUT 信号
```

**你的 test 中的典型用法—WO 寄存器验证**：

```systemverilog
// 第 1 步：前门写入（走真实 AXI 总线）
wo_write_peek(reg_model.cs_sck, 32'h0C);

// 第 2 步：后门 peek 读出物理寄存器值
// （因为 cs_sck 是 WO，前门读回 0，必须用 peek）
reg_model.cs_sck.peek(st_p, pv);
// pv 应该等于之前写入的 0x0C
```

这就是 RAL 的**双通路验证**：front door 确认总线协议通，back door 确认物理值真写进去了。

---

## 5. RAL 预测机制 (Prediction)

RAL 内部维护一份 `mirror`（影子值）用来跟踪寄存器的当前值。更新 mirror 有两种模式：

| 模式 | 设置 | 优点 | 缺点 |
|------|------|------|------|
| **Auto Predict** | `map.set_auto_predict(1)` | 简单，write 后自动更新 mirror | 假设总线一定写入成功 |
| **Explicit Predict** | `predict()` 或独立 predictor | 准确，考虑总线实际结果 | 难写 |

你项目用的是 Auto Predict：

```systemverilog
reg_model.map.set_auto_predict(1);   // → connect_phase
```

之后每次 `reg.write()` 执行完后，RAL 会自己把 `wdata` 写入 mirror。不需要外部 predictor。

---

## 6. RAL 冒烟测试流程 (ral_test.sv)

你的 test 验证了四个场景：

```
 1. backdoor 复位检查：8 个 WO 寄存器 peek 应 = 0
    → 确认 hdl_path 通 + 复位值对

 2. WO: frontdoor write → backdoor peek 比对
    → 确认前门写得进、后门确认写到物理值

 3. RO: frontdoor read → busy=0（空闲时）
    → 确认读通道通

 4. backdoor poke → frontdoor read
    → 演示 poke 调用链通，但硬件刷新导致预期内读回 0
```

---

## 7. RAL 在验证流程中的位置

```
VPlan → RAL 建模 → frontdoor 验证总线协议
                  → backdoor 验证物理值
                  → RAL sequence（替代 spi_cfg_seq）
                  → Coverage（reg 访问覆盖率）
```

你的 RAL 当前停留：已建模 + 冒烟 test。下一步如果要替代 `spi_cfg_seq`，需要写 RAL sequence——用 `reg.write()` 代替 `write_reg(ADDR_xxx, val)`。不需要马上做，但在面试里你可以说"已经搭好了模型，序列迁移是自然的下一步"。
