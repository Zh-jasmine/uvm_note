# UVM 寄存器读写验证环境教程

## 目录

- [一、验证任务概述](#一验证任务概述)
- [二、项目结构](#二项目结构)
- [三、待验证设计——寄存器模块](#三待验证设计寄存器模块)
- [四、数据载体——Transaction](#四数据载体transaction)
- [五、信号连接——Interface](#五信号连接interface)
- [六、时序驱动——Driver](#六时序驱动driver)
- [七、信号采样——Monitor](#七信号采样monitor)
- [八、激励调度——Sequencer](#八激励调度sequencer)
- [九、验证场景——Sequence](#九验证场景sequence)
- [十、结果比对——Scoreboard](#十结果比对scoreboard)
- [十一、组件打包——Agent](#十一组件打包agent)
- [十二、环境组装——Env](#十二环境组装env)
- [十三、测试用例——Test](#十三测试用例test)
- [十四、仿真顶层——Test Top](#十四仿真顶层test-top)
- [十五、编译脚本——Makefile](#十五编译脚本makefile)
- [十六、仿真运行与结果解读](#十六仿真运行与结果解读)
- [十七、数据流总览](#十七数据流总览)
- [十八、常见编译与运行错误](#十八常见编译与运行错误)


## 一、验证任务概述

本项目的验证对象是一个简单的寄存器模块（reg_module），它包含 256 个 32 位寄存器，通过一个 8 位地址总线访问。验证任务是确认该模块的读写功能正确：写入一个地址的数据能够被正确存储，并在后续读取时返回相同的值。

验证环境使用 UVM（Universal Verification Methodology）搭建，采用了标准的组件化架构。整个验证环境按照 UVM 的分层思想组织，从下到上依次为信号层（interface）、数据对象层（transaction）、驱动采样层（driver/monitor）、激励层（sequencer/sequence）、数据比对层（scoreboard）、组件封装层（agent/env）以及测试用例层（test）。每一层在上层和下层之间增加了特定的抽象，使得验证环境具备可复用性和可扩展性。

## 二、项目结构

```
reg_test/
├── Makefile                # 编译和运行脚本
├── rtl/
│   └── reg_module.sv       # 待验证设计（DUT）
├── tb/
│   ├── reg_if.sv           # 信号接口
│   └── test_top.sv         # 仿真顶层
└── verif/
    ├── reg_transaction.sv  # 数据对象（transaction）
    ├── reg_driver.sv       # 驱动
    ├── reg_monitor.sv      # 监视器
    ├── reg_sequencer.sv    # 序列器
    ├── reg_sequence.sv     # 激励序列
    ├── reg_scoreboard.sv   # 比对器
    ├── reg_agent.sv        # 代理
    ├── reg_env.sv          # 验证环境
    └── reg_base_test.sv    # 测试用例
```

整个项目按目录划分意图清晰。`rtl/` 目录存放待验证的 RTL 代码，`tb/` 目录存放验证环境与 DUT 连接所需的信号接口和仿真顶层，`verif/` 目录存放所有 UVM 验证组件。这种组织方式将设计和验证分开，是实际项目中的常规做法。

## 三、待验证设计——寄存器模块

文件位置：`rtl/reg_module.sv`

### 模块接口

```
信号名      方向  位宽    功能
clk         输入  1bit    时钟
rst_n       输入  1bit    异步复位（低有效）
addr        输入  8bit    地址（支持 256 个寄存器）
write       输入  1bit    写使能（1=写，0=读）
wdata       输入  32bit   写入数据
rdata       输出  32bit   读出数据（组合逻辑输出）
rvalid      输出  1bit    读有效指示
```

### 行为描述

这个 DUT 非常简单，核心是一个 256x32 的寄存器阵列 `reg [31:0] mem [0:255]`。

读操作是组合逻辑的。`rdata` 始终等于 `mem[addr]`，即地址线上的值直接决定输出，不依赖时钟。这种设计在读写操作的时序上有重要影响，后文的监测策略部分会详细讨论。

写操作是时序逻辑的。在时钟上升沿采样到 `write=1` 时，将 `wdata` 写入 `mem[addr]`。

`rvalid` 信号在复位结束后始终为高。这意味着 DUT 不提供明确的读完成握手信号，验证环境需要自己决定何时采样读数据。

### 关键代码

```systemverilog
module reg_module (
    input        clk,
    input        rst_n,
    input [7:0]  addr,
    input        write,
    input [31:0] wdata,
    output logic [31:0] rdata,
    output logic        rvalid
);
    reg [31:0] mem [0:255];
    assign rdata = mem[addr];

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            rvalid <= 1'b0;
        else
            rvalid <= 1'b1;
        if (write) mem[addr] <= wdata;
    end
endmodule
```

## 四、数据载体——Transaction

文件位置：`verif/reg_transaction.sv`

### 这一层加了什么

Transaction 在 UVM 中对应 `uvm_sequence_item` 的子类，它描述了验证环境中流动的一笔数据。在寄存器读写场景中，一笔 transaction 包含地址（addr）、数据（data）和读写标识（write）三个字段。

### 为什么需要单独分出这一层

如果没有 transaction 这一层，driver 和 scoreboard 之间就需要直接传递地址总线和数据总线的值，也就是多个信号的拼接。这样做的问题是信号散布在多个组件中，一旦接口发生变化（比如地址位宽从 8bit 变成 16bit），每个组件都需要修改。Transaction 层将这些字段封装成一个独立的数据对象，所有组件都依赖这个对象定义，接口变化时只需要修改 transaction 一个地方。

此外，`uvm_sequence_item` 提供了 `randomize()` 方法支持随机化，这是 UVM 验证方法学的核心优势之一。通过在 transaction 中声明 `rand` 字段和约束，sequence 可以方便地生成随机激励。

### 字段定义

```systemverilog
class reg_transaction extends uvm_sequence_item;
    `uvm_object_utils(reg_transaction)

    rand bit [7:0]  addr;    // 寄存器地址，可随机化
    rand bit [31:0] data;    // 数据，可随机化
         bit        write;   // 读写标识，1=写，0=读

    constraint addr_range { addr inside {[0:255]}; }
```

字段中 `addr` 和 `data` 被声明为 `rand`，允许 UVM 的 `randomize()` 自动产生随机值。`write` 不声明为 `rand`，因为它由 sequence 在产生激励时明确指定，不需要随机。

地址范围约束 `addr_range` 限制地址只能在 0 到 255 之间，匹配 DUT 的 256 个寄存器深度。如果没有这个约束，随机化可能产生超出 DUT 地址范围的 transaction。

## 五、信号连接——Interface

文件位置：`tb/reg_if.sv`

### 这一层加了什么

SystemVerilog 的 interface 将 DUT 与验证环境之间的所有信号封装到一个独立的结构中。在 UVM 中常用的传递方式是使用 `virtual interface` 指针，通过 `uvm_config_db` 在 test top 中设置，在 UVM 组件中获取。

### 为什么需要单独分出这一层

在传统 Verilog 验证中，信号通过端口逐层传递。如果信号需要新增或修改，需要修改多个文件的端口列表。Interface 将一组相关的信号封装在一起，验证组件只需要持有这个 interface 的指针就可以驱动或采样所有信号，无需关心顶层是如何连接的。

此外，interface 可以包含时钟块（clocking block）和时序控制（timing skew），这些功能在 UVM 的高级使用场景中非常有用。

### 信号集合

```systemverilog
interface reg_if (input bit clk);
    logic        rst_n;
    logic [7:0]  addr;
    logic        write;
    logic [31:0] wdata;
    logic [31:0] rdata;
    logic        rvalid;
endinterface
```

## 六、时序驱动——Driver

文件位置：`verif/reg_driver.sv`

### 这一层加了什么

Driver 继承自 `uvm_driver #(reg_transaction)`，负责从 sequencer 接收 transaction，按照 DUT 的总线时序驱动到 interface 上。它把 transaction 层面的数据（addr、data、write）转换成信号层面的时序（时钟沿、信号赋值）。

### 为什么需要单独分出这一层

验证工程师需要产生各种总线操作（读、写、空闲等），这些操作的时序是固定的。如果每个 sequence 都直接操作 interface 信号，代码会大量重复且容易引入时序错误。Driver 层将这些时序逻辑集中在一个地方，sequence 只需要描述"要做什么"，driver 负责"怎么驱动"。

### 工作机制

Driver 在 `run_phase` 中进入一个无限循环，每个迭代执行以下步骤：

1. 从 sequencer 获取下一个 transaction：`seq_item_port.get_next_item(req)`
2. 调用 `drive_one()` 任务，将 transaction 驱动到 interface 上
3. 通知 sequencer 当前 item 已完成：`seq_item_port.item_done()`
4. 如果是读操作，将 transaction（带有读回的数据）通过 `drv_ap` 分析端口发送给 scoreboard

### 写操作时序

```
时钟周期 1：tr.addr → addr, tr.data → wdata, write = 1
时钟周期 2：write = 0（完成写）
```

### 读操作时序

```
时钟周期 1：tr.addr → addr, write = 0
时钟周期 2：采样 rdata，存入 tr.data（组合逻辑读）
```

### 关键代码

```systemverilog
task drive_one(reg_transaction tr);
    @(posedge vif.clk);
    vif.addr <= tr.addr;

    if (tr.write) begin
        vif.wdata <= tr.data;
        vif.write <= 1'b1;
        @(posedge vif.clk);
        vif.write <= 1'b0;
    end else begin
        vif.write <= 1'b0;
        @(posedge vif.clk);
        tr.data = vif.rdata;
    end
endtask
```

注意读操作将 `vif.rdata` 的采样值写回了 `tr.data`。这意味着同一个 transaction 对象在发送给 driver 时，`data` 字段保存的是写入数据（写操作）或期望地址（读操作），在 driver 完成处理后，`data` 字段变成了读回的实际数据。这种方式使得 scoreboard 能够通过一个 transaction 同时获得读操作的地址和实际读回的数据。

## 七、信号采样——Monitor

文件位置：`verif/reg_monitor.sv`

### 这一层加了什么

Monitor 继承自 `uvm_monitor`，负责被动地观察 interface 上的信号变化，恢复成 transaction 后通过分析端口发送出去。它不驱动任何信号，只采样。

### 为什么需要单独分出这一层

验证环境需要知道 DUT 的实际行为来与预期行为对比。Monitor 是 scoreboard 的数据来源之一，它为验证环境提供了一个独立于 driver 的观察视角。driver 只知道自己发送了什么，而 monitor 能观察到 DUT 实际接收到了什么。

### 写操作为主

在这个项目中，monitor 只采样写操作（`vif.write` 为高时）。读操作由 driver 通过 `drv_ap` 直接汇报给 scoreboard。

之所以这样设计，是因为 DUT 的 `rvalid` 始终为高，monitor 无法区分"读操作正在进行"和"读操作已完成"这两个状态。如果 monitor 在 `rvalid` 为高时采样 `rdata`，在每个时钟周期都会触发读采样，产生大量虚假的读 transaction。

将读操作的监测交给 driver 是一种务实的方案。Driver 知道自己发起了读请求，也知道读数据在下一个时钟周期稳定，因此它可以直接将结果汇报给 scoreboard。这种架构将写操作的采样（被动监测）和读操作的汇报（主动上报）分开，避免了 DUT 时序不完整带来的竞争问题。

### 关键代码

```systemverilog
task run_phase(uvm_phase phase);
    forever begin
        @(posedge vif.clk);
        if (vif.write) begin
            tr = reg_transaction::type_id::create("tr");
            tr.addr  = vif.addr;
            tr.data  = vif.wdata;
            tr.write = 1'b1;
            ap.write(tr);
        end
    end
endtask
```

Monitor 在每个时钟上升沿检查 `write` 信号，如果为高则采集当前地址和数据，创建一个 transaction 并通过分析端口发送。

## 八、激励调度——Sequencer

文件位置：`verif/reg_sequencer.sv`

### 这一层加了什么

Sequencer 继承自 `uvm_sequencer #(reg_transaction)`，本质是一个带仲裁的 FIFO 队列。它接收来自 sequence 的 transaction 请求，仲裁后转发给 driver。

### 为什么需要单独分出这一层

在实际项目中，可能有多个 sequence 同时运行，都需要向 driver 发送 transaction。Sequencer 解决"谁先谁后"的调度问题，它内部实现了 FIFO 和多种仲裁算法。对于这个简单的寄存器验证项目，只需要一个 sequence 运行，sequencer 的作用退化为在 sequence 和 driver 之间建立连接。

### Sequencer 不需要写代码

```systemverilog
class reg_sequencer extends uvm_sequencer #(reg_transaction);
    `uvm_component_utils(reg_sequencer)
    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction
endclass
```

Sequencer 的父类实现了所有必要的功能，包括 get_next_item、item_done、仲裁等。验证工程师只需要声明参数化的 `uvm_sequencer #(transaction_type)` 即可。

## 九、验证场景——Sequence

文件位置：`verif/reg_sequence.sv`

### 这一层加了什么

Sequence 描述了验证工程师关心的测试场景。它生成 transaction 并通过 sequencer 发送给 driver。一个 UVM sequence 本质上是一系列 transaction 的产生和调度逻辑。

### 为什么需要单独分出这一层

将测试场景与驱动时序分离意味着修改测试内容时不需要改动 driver 或 DUT 的任何代码。每次修改都可以减少修改范围，降低引入新 bug 的风险。

### 基类 sequence

基类 `reg_base_sequence` 封装了读写操作的方法，供具体的 sequence 继承。

```systemverilog
// 读操作封装
task read_reg(input bit [7:0] addr, output bit [31:0] data);
    tr = reg_transaction::type_id::create("tr");
    start_item(tr);
    tr.addr  = addr;
    tr.write = 0;
    finish_item(tr);
    data = tr.data;
endtask

// 写操作封装
task write_reg(input bit [7:0] addr, input bit [31:0] data);
    tr = reg_transaction::type_id::create("tr");
    start_item(tr);
    tr.addr  = addr;
    tr.data  = data;
    tr.write = 1;
    finish_item(tr);
endtask
```

这里的关键是 `start_item()` 和 `finish_item()` 这对方法。`start_item` 向 sequencer 请求发送权限，`finish_item` 通知 driver 当前 item 已准备好。这中间的 `tr.write = 1` 等赋值操作在 sequence 中完成，时序驱动部分交给 driver。

注意读操作调用结束后 `tr.data` 中已经包含了 driver 写回的实际读数据。

### 写后读测试

`reg_write_read_seq` 是验证的核心场景：对每个随机地址，先写入一个随机数据，再立即读取同一地址，确认读回的值与写入的值一致。

```systemverilog
repeat (5) begin
    addr  = $urandom_range(0, 200);
    wdata = $urandom();
    write_reg(addr, wdata);
    read_reg(addr, rdata);
end
```

### 边界测试

`reg_boundary_seq` 测试地址空间的边界和中间位置：地址 0（最小值）、地址 255（最大值）和地址 100（普通值）。使用固定的已知数据（0xDEAD_BEEF、0x12345678、0xABCD0000），方便人工检查。

## 十、结果比对——Scoreboard

文件位置：`verif/reg_scoreboard.sv`

### 这一层加了什么

Scoreboard 是验证环境的"裁判"，继承自 `uvm_scoreboard`。它从两个数据路径接收信息，判断 DUT 的行为是否正确。写操作告诉它"应该存什么值"，读操作告诉它"读回了什么值"，它比较两者是否一致。

### 为什么需要单独分出这一层

验证的核心问题是"DUT 的行为是否正确"。Driver 负责发送数据，monitor 负责捕获数据，但检查数据正确性的逻辑应该独立于这两者。单独分出 scoreboard 层使得检查逻辑集中在一个地方，方便实现复杂的比较算法和统计功能。

### 双 FIFO 架构

Scoreboard 内部有两个 TLM analysis FIFO：

- `mon_fifo`：接收来自 monitor 的写操作 transaction。Scoreboard 使用这些信息构建一个"预期存储器"（expected_mem）。
- `drv_fifo`：接收来自 driver 的读操作 transaction。Scoreboard 使用这些信息提取实际读回的数据，与预期存储器中的值比较。

这两个 FIFO 使用 `fork...join_none` 在后台独立运行，各自处理各自的输入数据。`join_none` 使得 `run_phase` 不会因为两个 forever 循环而阻塞，仿真器可以在所有 transaction 处理完毕后结束 run phase。

### 比对逻辑

1. 从 `mon_fifo` 收到写操作：将 `addr` 和 `data` 存入 `expected_mem`
2. 从 `drv_fifo` 收到读操作：
   - 如果 `expected_mem` 中已有该地址的记录，比较实际数据和预期数据
   - 匹配则 `pass_count` 增加
   - 不匹配则 `fail_count` 增加并报告 `uvm_error`
   - 如果该地址未写过，发出一个信息级别的消息

### 报告阶段

在 `report_phase` 中输出最终统计结果：

```systemverilog
function void report_phase(uvm_phase phase);
    `uvm_info("SCB", $sformatf("=== 比对统计: 通过 %0d / 失败 %0d ===",
        pass_count, fail_count), UVM_LOW)
endfunction
```

这是验证工程师查看测试是否通过的重要参考。

## 十一、组件打包——Agent

文件位置：`verif/reg_agent.sv`

### 这一层加了什么

Agent 将 driver、monitor 和 sequencer 这三个与同一套信号相关的组件打包在一起。它对上层的 env 隐藏了内部组件的创建和连接细节。

### 为什么需要单独分出这一层

在复杂的芯片验证项目中，同一个总线协议（如 AXI、APB）可能在多个模块中使用。将 driver、monitor、sequencer 打包成 agent 后，env 可以直接实例化这个 agent 多次，而不需要为每个模块重复创建和连接这些组件。

Agent 有两种工作模式。`UVM_ACTIVE` 模式包含 driver 和 sequencer（可发送激励），`UVM_PASSIVE` 模式只包含 monitor（仅观察）。对于只需要监测信号但不驱动信号的场景（比如验证环境中的从机接口），passive 模式非常有用。

### 端口透传

Agent 内部持有一个 `uvm_analysis_port #(reg_transaction)` 类型的 `drv_ap`，它连接了 driver 内部的 `drv_ap` 端口。这样 scoreboard 可以通过 env 访问 agent 暴露的这个端口来接收读数据。

```systemverilog
function void connect_phase(uvm_phase phase);
    drv.seq_item_port.connect(sqr.seq_item_export);
    drv.drv_ap.connect(drv_ap);
endfunction
```

## 十二、环境组装——Env

文件位置：`verif/reg_env.sv`

### 这一层加了什么

Env 是整个验证环境的顶层容器，它实例化 agent 和 scoreboard，并将它们连接起来形成完整的数据通路。

### 为什么需要单独分出这一层

一个验证环境通常包含多个 agent（多个接口）和一个 scoreboard。Env 层将这些组件装配成一个整体，test 只需要实例化 env 即可获得完整的验证环境，不需要关心组件间的连接细节。

### 数据路径连接

```systemverilog
function void connect_phase(uvm_phase phase);
    // monitor 的写操作 → scoreboard 的记录通道
    agt.mon.ap.connect(scb.mon_fifo.analysis_export);
    // driver 的读结果 → scoreboard 的比对通道
    agt.drv_ap.connect(scb.drv_fifo.analysis_export);
endfunction
```

这两条连接定义了整个验证环境的数据流。monitor 采到的写操作通过 `mon_fifo` 进入 scoreboard 的记录逻辑，driver 汇报的读结果通过 `drv_fifo` 进入 scoreboard 的比对逻辑。

## 十三、测试用例——Test

文件位置：`verif/reg_base_test.sv`

### 这一层加了什么

Test 是 UVM 验证层次的顶层组件，继承自 `uvm_test`。它负责创建 env、控制仿真的开始和结束、以及启动 sequence。

### 为什么需要单独分出这一层

不同的测试场景（写后读测试、边界测试、随机测试等）共享同一个 env 和 agent，但使用不同的 sequence 和配置。通过继承一个基类 test 并在子类中更换 sequence，可以在不修改 env 的情况下运行不同的测试。

### Objection 机制

UVM 的 objection 机制用于控制仿真何时结束。当 test 的 `run_phase` 中 raise objection 时，仿真器知道当前还有未完成的工作。当 sequence 执行完毕，test drop objection 后，仿真器确认所有工作已完成，结束 run phase 进入后续的检查阶段。

```systemverilog
task run_phase(uvm_phase phase);
    phase.raise_objection(this);
    seq.start(env.agt.sqr);
    phase.drop_objection(this);
endtask
```

### 测试用例组织

测试用例的组织方式在 UVM 中采用了继承和配置分离的设计。基类定义了公共结构，子类以不同方式配置：

```systemverilog
// 基类 test：创建 env
class reg_base_test extends uvm_test;
    reg_env env;
    // build_phase 中创建 env
endclass

// 写后读测试：使用 reg_write_read_seq
class reg_write_read_test extends reg_base_test;
    // 在 run_phase 中启动 reg_write_read_seq
endclass

// 边界测试：使用 reg_boundary_seq
class reg_boundary_test extends reg_base_test;
    // 在 run_phase 中启动 reg_boundary_seq
endclass
```

## 十四、仿真顶层——Test Top

文件位置：`tb/test_top.sv`

### 这一层加了什么

Test top 是一个 Verilog module（非 UVM 组件），它是整个仿真的入口。它生成时钟和复位信号，实例化 DUT 和 interface，将 interface 通过 `uvm_config_db` 传递给 UVM 环境，最后调用 `run_test()` 启动 UVM。

### 为什么需要单独分出这一层

UVM 是一个基于 SystemVerilog 的类库，它无法生成硬件信号（时钟、复位）或实例化 DUT。这些硬件级的操作必须在 module 中完成。Test top 是连接硬件世界和 UVM 软件世界的桥梁。

### 时钟生成

```systemverilog
bit clk;
initial forever #5 clk = ~clk;  // 100MHz
```

### 复位时序

仿真开始时 rst_n 保持低电平 5 个时钟周期，然后拉高。这确保 DUT 正确复位后再开始正常操作。

```systemverilog
initial begin
    rst_n = 0;
    repeat (5) @(posedge clk);
    rst_n = 1;
end
```

### config_db 设置

`uvm_config_db` 是 UVM 中组件间共享配置的全局数据库。这里将 `virtual reg_if` 指针存入 config_db，路径为 `"uvm_test_top.env.agt.*"`，这样 agent 下的所有组件（driver 和 monitor）都可以通过相同的字符串键获取这个 interface 指针。

```systemverilog
initial begin
    uvm_config_db #(virtual reg_if)::set(null, "uvm_test_top.env.agt.*", "vif", vif);
    run_test();
end
```

## 十五、编译脚本——Makefile

文件位置：`Makefile`

### 基本用法

```bash
make sim TEST=reg_write_read_test   # 编译 + 运行写后读测试
make sim TEST=reg_boundary_test     # 编译 + 运行边界测试
make clean                          # 清理编译产物
make compile_only                    # 仅编译
make run_only                        # 仅运行（需已编译）
```

### 变量机制

Makefile 使用 `TEST` 变量控制运行哪个测试。`TEST ?= reg_write_read_test` 表示默认使用写后读测试，但可以通过命令行覆盖。

`TEST` 变量在三个地方展开：
1. 编译输出文件名：`simv_$(TEST)`
2. UVM 运行时参数：`+UVM_TESTNAME=$(TEST)`
3. 日志文件名：`$(TEST).log`

### VCS 编译选项

```
-full64                # 64 位模式
-sverilog              # 启用 SystemVerilog
-timescale=1ns/1ps     # 时间精度
+incdir+               # 包含目录
+define+UVM_NO_DPI     # 禁用 UVM 的 DPI 依赖
```

`UVM_NO_DPI` 这个宏定义非常重要。VCS 自带的 UVM 1.2 版本中的 DPI（Direct Programming Interface）功能在较老的 VCS 版本中可能无法正常链接。加上这个宏定义后，UVM 会通过 VCS 的内部机制获取测试名（UVM_TESTNAME）而不是通过 DPI，避免了相关编译问题。

## 十六、仿真运行与结果解读

### 仿真输出结构

运行 `make sim TEST=reg_write_read_test` 后，VCS 输出两个重要信息：

**1. UVM 组件拓扑**

仿真开始时，UVM 打印完整的组件树形结构。这用于验证环境的层次关系是否正确。

```
uvm_test_top               reg_write_read_test
  env                      reg_env
    agt                    reg_agent
      drv                  reg_driver
        drv_ap             uvm_analysis_port
      mon                  reg_monitor
        ap                 uvm_analysis_port
      sqr                  reg_sequencer
    scb                    reg_scoreboard
      drv_fifo             uvm_tlm_analysis_fifo
      mon_fifo             uvm_tlm_analysis_fifo
```

**2. 执行日志**

每笔 transaction 的执行过程中各个组件输出的信息。以一次写后读为例：

```
@ 15000: [MON] 采样到写 addr=0x95 data=0xeeadce97
@ 15000: [DRV] 写 addr=0x95 data=0xeeadce97
@ 15000: [SCB] 记录写 addr=0x95 data=0xeeadce97
@ 35000: [DRV] 读 addr=0x95 -> data=0xeeadce97
@ 35000: [SEQ] 写后读 addr=0x95 wdata=0xeeadce97 rdata=0xeeadce97
@ 35000: [SCB] 比对通过 addr=0x95 data=0xeeadce97
```

关键信息阅读要点：
- 同一笔写操作，monitor 采样时间和 driver 驱动时间一致（@ 15000），表明时序正确
- 读回的数据 0xeeadce97 与写入的数据一致
- Scoreboard 报告"比对通过"表示验证通过

**3. UVM Report Summary**

仿真结束时的统计摘要，这是判断测试是否通过的核心依据：

```
--- UVM Report Summary ---
** Report counts by severity
UVM_INFO :   37
UVM_WARNING :    0
UVM_ERROR :    0
UVM_FATAL :    0
```

关键检查点：`UVM_ERROR` 和 `UVM_FATAL` 必须为 0。

**4. SCB 统计**

```
[SCB] === 比对统计: 通过 5 / 失败 0 ===
```

这是 scoreboard 自己的通过/失败计数，直接给出了功能验证的结果。

### 写后读测试结果（5/5 通过）

```
写 addr=0x95 data=0xeeadce97 → 读 data=0xeeadce97 ✓
写 addr=0xe  data=0x62b884c7 → 读 data=0x62b884c7 ✓
写 addr=0x9f data=0x079204fe → 读 data=0x079204fe ✓
写 addr=0x70 data=0x234f278a → 读 data=0x234f278a ✓
写 addr=0xba data=0x1ad05721 → 读 data=0x1ad05721 ✓
UVM_ERROR: 0, UVM_FATAL: 0
SCB: 通过 5 / 失败 0
```

### 边界测试结果（3/3 通过）

```
写 addr=0x00 data=0xDEAD_BEEF → 读 data=0xdeadbeef ✓
写 addr=0xFF data=0x12345678 → 读 data=0x12345678 ✓
写 addr=0x64 data=0xABCD0000 → 读 data=0xabcd0000 ✓
UVM_ERROR: 0, UVM_FATAL: 0
SCB: 通过 3 / 失败 0
```

## 十七、数据流总览

整个验证环境的数据流可以概括为两条路径：

**写路径（从 sequence 到 scoreboard 的记录）：**

```
Sequence 创建 transaction
  → start_item() / finish_item() 发送给 Sequencer
    → Driver 获取 transaction 并驱动到 Interface
      → DUT 接收写数据（write=1 时写入 mem[addr]）
      → 同时，Monitor 从 Interface 采样写信号
        → Monitor 创建 transaction 通过分析端口发送
          → Scoreboard 的 mon_fifo 接收
            → Scoreboard 记录 expected_mem[addr] = data
```

**读路径（从 sequence 到 scoreboard 的比对）：**

```
Sequence 创建读 transaction
  → start_item() / finish_item() 发送给 Sequencer
    → Driver 获取 transaction 驱动读地址到 Interface
      → DUT 组合逻辑输出 rdata = mem[addr]
        → 下一个时钟沿 Driver 采样 rdata 写回 transaction
          → Driver 通过 drv_ap 发送读结果
            → Scoreboard 的 drv_fifo 接收
              → Scoreboard 比对实际数据与 expected_mem[addr]
```

这种双路径的设计解决了 DUT 无法提供可靠读握手信号的问题。Monitor 只采样写操作不受 rvalid 的影响，driver 负责读操作的结果采集和汇报。

## 十八、常见编译与运行错误

### 文件未找到

```
make: *** No rule to make target `sim'.  Stop.
```

原因是在 Makefile 所在的目录之外执行了 make。使用 `cd ~/reg_test` 切换到项目目录。

### config_db 获取失败

```
[DRV] 从 config_db 获取 vif 失败
```

config_db 的路径字符串 `"uvm_test_top.env.agt.*"` 必须与 UVM 组件树中的实际路径匹配。如果 test 名称不同或 env 层次不同，需要对应调整。

### 仿真不结束

如果 scoreboard 的 `run_phase` 中使用了 `join` 而不是 `join_none`，两个 forever 循环会阻塞 `run_phase` 结束，导致仿真永远运行。这是因为 UVM 的 run phase 需要所有组件的 run_phase 都完成才能继续到 extract phase。使用 `fork...join_none` 将 forever 循环放在后台线程中，run_phase 执行到 `join_none` 后立即返回，不再阻塞。

### rvalid 竞争条件

如果 monitor 同时采样写和读，而 DUT 的 `rvalid` 始终为高，会导致每个时钟周期都被视为读操作。解决方案是 monitor 只采样写操作，读操作由 driver 汇报。

### 编译时 UVM_TESTNAME 警告

```
UVM_TESTNAME=reg_write_read_test 不是合法的编译选项
```

`+UVM_TESTNAME` 是运行时参数，在编译阶段使用会在 VCS 编译日志中产生 warning。应该将其放在执行 `simv` 的命令行中，而不是 VCS 编译命令中。

### UVM_NO_DPI 未定义

如果没有定义 `+define+UVM_NO_DPI`，UVM 1.2 会尝试通过 DPI 获取测试名，在某些 VCS 版本中可能导致链接错误或在运行时找不到函数。加入该宏定义后，UVM 换用 VCS 内部接口获取测试名，避免了这个问题。

### 编译结果不同的 simv 文件

每次运行 `make sim TEST=xxx` 时，VCS 都会生成一个独立的 `simv_xxx` 可执行文件。这是因为 VCS 是静态编译型仿真器，不同的 UVM_TESTNAME 实际上是在编译时确定的。如果要避免重复编译，可以使用 `make run_only TEST=xxx`（但前提是 simv 文件已经编译过且代码未变化）。
