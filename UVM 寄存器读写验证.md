## 目录

- [[#一、验证环境总览]]
- [[#二、信号说明]]
- [[#三、总线时序]]
- [[#四、仿真顶层——Test Top]]
- [[#五、测试用例——Test]]
- [[#六、环境组装——Env]]
- [[#七、组件打包——Agent]]
- [[#八、数据载体——Transaction]]
- [[#九、信号连接——Interface]]
- [[#十、激励调度——Sequencer]]
- [[#十一、验证场景——Sequence]]
- [[#十二、时序驱动——Driver]]
- [[#十三、信号采样——Monitor]]
- [[#十四、结果比对——Scoreboard]]
- [[#十五、编译脚本——Makefile]]
- [[#十六、仿真运行与结果解读]]
- [[#十七、数据流总览]]
- [[#十八、常见编译与运行错误]]

## 一、验证环境总览

[⬆ 返回目录](#目录)

本项目的验证对象是一个简单的寄存器模块（reg_module），包含 256 个 32 位寄存器，通过 8 位地址总线访问。验证任务是确认该模块的读写功能正确：写入一个地址的数据能够被正确存储，并在后续读取时返回相同的值。

验证环境使用 UVM 搭建，采用标准的组件化架构。下表从信号到激励的分层视角列出了各组件的核心职责。

| 组件 | 文件 | 核心职责 |
|------|------|---------|
| Transaction | reg_transaction.sv | 封装一笔寄存器操作的地址、数据和读写标识 |
| Interface | reg_if.sv | 将 DUT 和验证环境之间的信号封装为一个独立结构 |
| Driver | reg_driver.sv | 接收 transaction，按总线时序驱动到 interface 上 |
| Monitor | reg_monitor.sv | 被动采样 interface 上的信号，恢复成 transaction |
| Sequencer | reg_sequencer.sv | 在 sequence 和 driver 之间路由 transaction |
| Sequence | reg_sequence.sv | 生成具体的读写激励序列 |
| Scoreboard | reg_scoreboard.sv | 比较预期值与实际值，报告通过或失败 |
| Agent | reg_agent.sv | 将 driver、monitor、sequencer 打包为可复用单元 |
| Env | reg_env.sv | 实例化 agent 和 scoreboard 并连接数据通路 |
| Test | reg_base_test.sv | 创建 env，控制仿真起止，启动 sequence |
| Test Top | test_top.sv | 生成时钟复位，实例化 DUT 和 interface，启动 UVM |

验证环境的数据流分为写路径和读路径两条。写路径从 sequence 产生写 transaction 开始，经 sequencer 转发给 driver，driver 驱动到 interface 后 ，DUT 存储数据。同时 monitor 从 interface 采样写信号，重建出 transaction 并通过分析端口发送给 scoreboard 的记录通道。读路径由 sequence 产生读 transaction，driver 驱动读地址后，DUT 输出读数据，driver 在下一个时钟沿采样读数据写回 transaction，然后直接汇报给 scoreboard 的比对通道。

项目文件按目录组织。`rtl/` 目录存放待验证的 RTL 代码，`tb/` 目录存放信号接口和仿真顶层，`verif/` 目录存放所有 UVM 验证组件，根目录的 Makefile 用于编译和运行。

### 待验证设计

文件位置：`rtl/reg_module.sv`

核心逻辑是一个 256x32 的寄存器阵列 `reg [31:0] mem [0:255]`。读操作是组合逻辑，`rdata` 始终等于 `mem[addr]`。写操作是时序逻辑，在时钟上升沿采样到 `write=1` 时将 `wdata` 写入 `mem[addr]`。`rvalid` 信号在复位结束后始终为高，不提供读完成握手信号。

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
        if (write) mem[addr] <= wdata;//dut收到写信号，就把写数据输出到对应地址的寄存器
    end
endmodule
```

## 二、信号说明

[⬆ 返回目录](#目录)

以下信号由顶层测试平台和 DUT 之间通过 interface 连接。每个信号在读写操作中承担特定的角色。

| 信号名 | 方向 | 位宽 | 功能 |
|--------|------|------|------|
| clk    | 输入 | 1bit | 时钟，所有时序操作同步于此信号 |
| rst_n  | 输入 | 1bit | 异步复位，低有效 |
| addr   | 输入 | 8bit | 地址总线，选择 256 个寄存器中的一个 |
| write  | 输入 | 1bit | 写使能，为 1 时将 wdata 写入 addr 指定的寄存器；为 0 时表示读操作（无独立的读使能信号） |
| wdata  | 输入 | 32bit | 写入数据，仅写操作时有效 |
| rdata  | 输出 | 32bit | 读出数据，由 addr 组合逻辑决定，不依赖时钟 |
| rvalid | 输出 | 1bit | 读有效指示，复位结束后始终为高 |

`addr` 和 `write` 由 driver 驱动，`wdata` 在写操作时由 driver 驱动，`rdata` 由 DUT 驱动，monitor 采样 `write` 和 `wdata`。clk 由 test top 生成，rst_n 在仿真开始时由 test top 控制。

`rvalid` 在标准协议中用于指示读数据有效——monitor 看到 `rvalid=1` 就去采样 `rdata`。但这个 DUT 的 `rvalid` 在复位结束后始终为高，相当于永远说"数据有效"，失去了握手意义。这就是后文 monitor 无法靠 rvalid 区分读状态、转而让 driver 汇报读结果的原因。

## 三、总线时序

[⬆ 返回目录](#目录)

本项目的总线协议非常简洁，读写各需两个时钟周期。

写操作。第一个时钟上升沿将 `addr` 设为目标地址，`wdata` 设为写入数据，`write` 设为 1。DUT 在此沿采样到 `write` 为高后将 `wdata` 存入 `mem[addr]`。第二个时钟上升沿将 `write` 设为 0，写操作完成。

读操作。第一个时钟上升沿将 `addr` 设为目标地址，`write` 设为 0。由于 `rdata` 是组合逻辑输出，地址设置后 `rdata` 立即变为 `mem[addr]`。第二个时钟上升沿 driver 采样 `rdata`，完成读操作。

所有信号变化均以时钟上升沿为基准。写操作涉及 addr、wdata、write 三个信号的驱动，读操作只驱动 addr 和 write，然后由 driver 采样 rdata。

## 四、仿真顶层——Test Top

[⬆ 返回目录](#目录)

文件位置：`tb/test_top.sv`

### 这一层加了什么

Test top 是一个 Verilog module（非 UVM 组件），是整个仿真的入口。它生成时钟和复位信号，实例化 DUT 和 interface，将 interface 通过 `uvm_config_db` 传递给 UVM 环境，最后调用 `run_test()` 启动 UVM。

### 为什么需要单独分出这一层

UVM 是一个基于 SystemVerilog 的类库，它无法生成硬件信号（时钟、复位）或实例化 DUT。这些硬件级的操作必须在 module 中完成。Test top 是连接硬件世界和 UVM 软件世界的桥梁。

### 时钟与复位

时钟周期设为 100MHz（每个 `#5` 切换一次）。`rst_n` 在仿真开始时保持低电平 5 个时钟周期，确保 DUT 内部状态正确复位后再拉高。

```systemverilog
bit clk;
initial forever #5 clk = ~clk;  // 100MHz

initial begin
    rst_n = 0;
    repeat (5) @(posedge clk);
    rst_n = 1;
end
```

### config_db 设置

`uvm_config_db` 是 UVM 中组件间共享配置的全局数据库。这里将 `virtual reg_if` 指针存入 config_db，供 UVM 组件的 `build_phase` 中使用 `::get` 取出。

```systemverilog
uvm_config_db #(virtual reg_if)::set(null, "uvm_test_top.env.agt.*", "vif", vif);
```

逐段拆解：`uvm_config_db` 是全局数据库；`#(virtual reg_if)` 指定存储的类型是虚接口指针，`reg_if` 是接口模块的名字（register interface 的缩写），`virtual` 表示这是指向硬件接口的指针而不是接口本身；`::set` 表示写入（`::get` 表示读出）；`null` 表示不限定上下文，这个配置对所有 UVM 组件可见（如果填一个组件句柄，就只有该组件及其子组件能看到）；`"uvm_test_top.env.agt.*"` 是路径字符串，`agt` 是 agent 实例的名称（由验证工程师在 env 中命名，不是关键字），`*` 通配符匹配 agent 下所有子组件（driver 和 monitor 都能看到）；`"vif"` 是键名，约定俗成用 `vif` 表示 virtual interface`，不是 UVM 的强制要求，验证工程师可以取任意名字，只要 set 和 get 用同一个字符串即可；最后的 `vif` 是要存入的值。

Driver 中取出的代码：
```systemverilog
if (!uvm_config_db #(virtual reg_if)::get(this, "", "vif", vif))
    `uvm_error("DRV", "获取 vif 失败")
```
`::get` 的路径从 `this`（driver 自身）的层次自动推导，因此第二个参数传空字符串即可。

## 五、测试用例——Test

[⬆ 返回目录](#目录)

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

`seq.start(env.agt.sqr)` 中的 `seq` 是一个 sequence 对象（如 `reg_write_read_seq`），`start` 是 UVM 内建方法，将其挂载到指定的 sequencer 上执行。`env.agt.sqr` 是层级路径：env → agent → sequencer。Sequence 必须挂载到 sequencer 才能通过 `start_item/finish_item` 与 driver 握手。

### 测试用例组织

```systemverilog
// 基类 test：创建 env
class reg_base_test extends uvm_test;
    reg_env env;
endclass

// 写后读测试：使用 reg_write_read_seq
class reg_write_read_test extends reg_base_test;
    // run_phase 中启动 reg_write_read_seq
endclass

// 边界测试：使用 reg_boundary_seq
class reg_boundary_test extends reg_base_test;
    // run_phase 中启动 reg_boundary_seq
endclass
```

## 六、环境组装——Env

[⬆ 返回目录](#目录)

文件位置：`verif/reg_env.sv`

### 这一层加了什么

Env 是整个验证环境的顶层容器，它实例化 agent 和 scoreboard，并将它们连接起来形成完整的数据通路。

### 为什么需要单独分出这一层

一个验证环境通常包含多个 agent（对应多个接口）和一个 scoreboard。Env 层将这些组件装配成一个整体，test 只需要实例化 env 即可获得完整的验证环境，不需要关心组件间的连接细节。

### 数据路径连接

```systemverilog
function void connect_phase(uvm_phase phase);
    // monitor 的写操作 → scoreboard 的记录通道
    agt.mon.ap.connect(scb.mon_fifo.analysis_export);
    // driver 的读结果 → scoreboard 的比对通道
    agt.drv_ap.connect(scb.drv_fifo.analysis_export);
endfunction
```

这两条连接定义了整个验证环境的数据流。Monitor 采到的写操作通过 `mon_fifo` 进入 scoreboard 的记录逻辑，driver 汇报的读结果通过 `drv_fifo` 进入 scoreboard 的比对逻辑。

这里使用的 TLM（Transaction-Level Modeling）是 UVM 组件之间传输 transaction 的一套接口标准。核心模式是生产者通过 analysis port 发出 transaction，消费者通过 analysis FIFO 接收。Agent 中的 `drv_ap` 是 `uvm_analysis_port #(reg_transaction)` 类型，它在 connect_phase 中被连接到 driver 内部的同名端口，然后通过 env 被连接到 scoreboard 的 `drv_fifo`。

## 七、组件打包——Agent

[⬆ 返回目录](#目录)

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

## 八、数据载体——Transaction

[⬆ 返回目录](#目录)

文件位置：`verif/reg_transaction.sv`

### 这一层加了什么

Transaction 在 UVM 中对应 `uvm_sequence_item` 的子类，它描述了验证环境中流动的一笔数据。在寄存器读写场景中，一笔 transaction 包含地址（addr）、数据（data）和读写标识（write）三个字段。

### 为什么需要单独分出这一层

如果没有 transaction 这一层，driver 和 scoreboard 之间就需要直接传递地址总线和数据总线的值，也就是多个信号的拼接。这样做的问题在于信号散布在多个组件中，一旦接口发生变化（比如地址位宽从 8bit 变成 16bit），每个组件都需要修改。Transaction 层将这些字段封装成一个独立的数据对象，所有组件都依赖这个对象定义，接口变化时只需要修改 transaction 一个地方。

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

### 对象创建与句柄语义

在 SystemVerilog 中，class 类型的变量本质上是一个句柄（handle），类似于指针。声明句柄和创建对象是两个独立的步骤。

声明 `reg_transaction tr;` 只产生一个空的句柄，其值为 null，此时访问 `tr.addr` 会导致空指针错误。必须先创建对象，句柄才能指向一个实际存在的对象实例。创建对象有两种方式：直接调用构造函数 `tr = new()`，或者使用 UVM 工厂方法 `tr = reg_transaction::type_id::create("tr")`。

两种方式的区别在于可覆盖性。UVM 工厂方法允许在运行时不修改代码的情况下将 `reg_transaction` 替换为其子类，这对于测试用例的灵活配置非常重要。直接调用 `new()` 则无法被工厂覆盖，创建出的永远是 `reg_transaction` 本身。在 UVM 环境中，建议统一使用 `type_id::create`。

由于 class 句柄的传参语义是引用传递而不是值传递，将一个句柄赋值给另一个变量或作为参数传入 task 时，传递的是对象的地址而不是对象的副本。多个句柄可以指向同一个对象，通过任何一个句柄修改对象的字段，其他句柄都能看到修改后的值。这一特性在 UVM 的数据流中反复出现——sequence 创建 transaction 后传递给 driver，driver 修改 transaction 的内容（如读回数据），sequence 在 driver 完成后读取修改后的结果。

## 九、信号连接——Interface

[⬆ 返回目录](#目录)

文件位置：`tb/reg_if.sv`

### 这一层加了什么

SystemVerilog 的 interface 将 DUT 与验证环境之间的所有信号封装到一个独立的结构中。在 UVM 中常用的传递方式是使用 `virtual interface` 指针，不是真正的接口，只是指向接口，所以是`vrtual`。通过 `uvm_config_db` 在 test top 中设置，在 UVM 组件中获取。

### 为什么需要单独分出这一层

在传统 Verilog 验证中，信号通过端口逐层传递。如果信号需要新增或修改，需要修改多个文件的端口列表。Interface 将一组相关的信号封装在一起，验证组件只需要持有这个 interface 的指针就可以驱动或采样所有信号，无需关心顶层是如何连接的。

此外，interface 可以包含时钟块（clocking block）和时序控制（timing skew），这些功能在 UVM 的高级使用场景中非常有用。

### 代码

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

## 十、激励调度——Sequencer

[⬆ 返回目录](#目录)

文件位置：`verif/reg_sequencer.sv`

### 这一层加了什么

Sequencer 继承自 `uvm_sequencer #(reg_transaction)`，本质是一个带仲裁的 FIFO 队列。它接收来自 sequence 的 transaction 请求，仲裁后转发给 driver。

### 为什么需要单独分出这一层

在实际项目中，可能有多个 sequence 同时运行，都需要向 driver 发送 transaction。Sequencer 解决谁先谁后的调度问题，它内部实现了 FIFO 和多种仲裁算法。对于这个简单的寄存器验证项目，只需要一个 sequence 运行，sequencer 的作用退化为在 sequence 和 driver 之间建立连接。

### 代码

```systemverilog
class reg_sequencer extends uvm_sequencer #(reg_transaction);
    `uvm_component_utils(reg_sequencer)
    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction
endclass
```

Sequencer 的父类实现了所有必要的功能，包括 get_next_item、item_done、仲裁等。验证工程师只需要声明参数化的 `uvm_sequencer #(transaction_type)` 即可。

`new(string name, uvm_component parent)` 的签名是 UVM 组件的规定。`name` 是组件在 UVM 树中的名字（打印日志和 config_db 路径时使用），`parent` 是父节点指针。这两个参数由 `type_id::create("sqr", this)` 在实例化时传入，验证工程师只需按这个固定格式写构造函数。

## 十一、验证场景——Sequence

[⬆ 返回目录](#目录)

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
    tr.addr  = addr;//把地址传给tr
    tr.write = 0;//发起读请求(=写拉低)，driver的逻辑是判断tr.write
    finish_item(tr);//结束读操作
    data = tr.data;//把读到的数据输出给data
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

`start_item` 向 sequencer 请求发送权限。Sequencer 内部维护一个仲裁队列，如果多个 sequence 同时请求，由 sequencer 决定谁先发送。`start_item` 会阻塞直到 sequencer 授予权限。

在 `start_item` 通过和 `finish_item` 调用之间，sequence 填充 transaction 的字段。这段时间内，transaction 已被 sequencer 标记为等待发送，但尚未传送到 driver。这种设计允许 sequence 在获取发送权之后再进行字段赋值甚至随机化（`tr.randomize()` 放在 start_item 和 finish_item 之间是 UVM 标准用法）。

`finish_item` 将 transaction 发送给 driver，并阻塞等待 driver 处理完成。Driver 通过 `get_next_item` 接收 transaction，执行 `drive_one` 进行时序驱动，最后调用 `item_done` 通知 sequencer，sequencer 再让 `finish_item` 返回。对于读操作，driver 在 `drive_one` 中将读回的数据写入了同一个 transaction 对象的 `data` 字段，因此 `finish_item` 返回后可以立即通过 `data = tr.data` 获取结果。

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

`reg_boundary_seq` 测试地址空间的边界和中间位置：地址 0（最小值）、地址 255（最大值）和地址 100（典型值）。使用固定的已知数据（0xDEAD_BEEF、0x12345678、0xABCD0000），方便人工检查。

## 十二、时序驱动——Driver

[⬆ 返回目录](#目录)

文件位置：`verif/reg_driver.sv`

### 这一层加了什么

Driver 继承自 `uvm_driver #(reg_transaction)`，负责从 sequencer 接收 transaction，按照 DUT 的总线时序驱动到 interface 上。它把 transaction 层面的数据（addr、data、write）转换成信号层面的时序（时钟沿、信号赋值）。

### 为什么需要单独分出这一层

验证工程师需要产生各种总线操作（读、写、空闲等），这些操作的时序是固定的。如果每个 sequence 都直接操作 interface 信号，代码会大量重复且容易引入时序错误。Driver 层将这些时序逻辑集中在一个地方，sequence 只需要描述要做什么，driver 负责怎么驱动。

### 工作机制

Driver 在其 `run_phase` 中进入一个无限循环，每个迭代执行以下步骤：

1. 从 sequencer 获取下一个 transaction：`seq_item_port.get_next_item(req)`
2. 调用 `drive_one()` 任务，将 transaction 驱动到 interface 上
3. 通知 sequencer 当前 item 已完成：`seq_item_port.item_done()`
4. 如果是读操作，将 transaction（带有读回的数据）通过 `drv_ap` 分析端口发送给 scoreboard

`get_next_item` 和 `item_done` 是 driver 侧的握手指令，它们分别对应 sequence 侧的 [[#十一、验证场景——Sequence|start_item 和 finish_item]]。sequence 调用 `start_item` 时阻塞等待 driver 调 `get_next_item`，sequence 调 `finish_item` 时阻塞等待 driver 调 `item_done`。这两对方法共同构成 UVM 的 sequence-driver 握手机制。

### 时序说明

读写各需两个时钟周期，具体时序已在[[#三、总线时序]]中说明。Driver 的职责是将时序逻辑用代码实现出来。

### 关键代码

```systemverilog
task drive_one(reg_transaction tr);

   @(posedge vif.clk);          // 等第一个时钟沿
vif.addr <= tr.addr;          // 把地址放上总线

if (tr.write) begin           // 判断这次是写还是读
    vif.wdata <= tr.data;      // 写：数据放总线
    vif.write <= 1'b1;         //     写使能拉高（通知 DUT：我要写）
    @(posedge vif.clk);        //     等一个周期让 DUT 完成写入
    vif.write <= 1'b0;         //     写使能拉低（释放总线）
end else begin
    vif.write <= 1'b0;         // 读：写使能拉低（告诉 DUT 我不写）
    @(posedge vif.clk);        //     等一个周期 DUT 把数据放上 rdata
    tr.data = vif.rdata;       //     从总线读回数据
end

endtask
```

读操作将 `vif.rdata` 的采样值写回 `tr.data`。这意味着同一个 transaction 对象在发送给 driver 时，`data` 字段保存的是写入数据（写操作）或无关值（读操作），在 driver 完成处理后，`data` 字段变成了读回的实际数据。这种方式使得 scoreboard 能够通过一个 transaction 同时获得读操作的地址和实际读回的数据。

`drive_one` 的参数 `reg_transaction tr` 在 SystemVerilog 中是一个 class 类型的句柄参数，传递的是对象的引用而不是对象的副本。读操作分支中 `tr.data = vif.rdata` 将读回的数据写入这个对象tr的date字段，sequence 在 `finish_item` 之后通过 `data = tr.data` 读取到的就是 driver 填入的结果。这种引用传递机制是 UVM 中 sequence 和 driver 之间数据返回的基础。

## 十三、信号采样——Monitor

[⬆ 返回目录](#目录)

文件位置：`verif/reg_monitor.sv`

### 这一层加了什么

Monitor 继承自 `uvm_monitor`，负责被动地观察 interface 上的信号变化，恢复成 transaction 后通过分析端口发送出去。它不驱动任何信号，只采样。

### 为什么需要单独分出这一层

验证环境需要知道 DUT 的实际行为来与预期行为对比。Monitor 是 scoreboard 的数据来源之一，它为验证环境提供了一个独立于 driver 的观察视角。driver 只知道自己发送了什么，而 monitor 能观察到 DUT 实际接收到了什么。

### 写操作为主

在这个项目中，monitor 只采样写操作（`vif.write` 为高时）。读操作由 driver 通过 `drv_ap` 直接汇报给 scoreboard。

这样设计的原因在于 DUT 的 `rvalid` 始终为高，monitor 无法区分读操作正在进行和读操作已完成这两个状态。如果 monitor 在 `rvalid` 为高时采样 `rdata`，每个时钟周期都会触发读采样，产生大量虚假的读 transaction。

将读操作的监测交给 driver 是一种务实的方案。Driver 知道自己发起了读请求，也知道读数据在下一个时钟周期稳定，因此它可以直接将结果汇报给 scoreboard。这种架构将写操作的采样（被动监测）和读操作的汇报（主动上报）分开，避免了 DUT 时序不完整带来的竞争问题。

### 关键代码

```systemverilog
task run_phase(uvm_phase phase);
    forever begin
        @(posedge vif.clk);
        if (vif.write) begin
            tr = reg_transaction::type_id::create("tr");
            tr.addr  = vif.addr;//把总线地址给tr
            tr.data  = vif.wdata;//把总线写数据给tr
            tr.write = 1'b1;//向dut发起写请求
            ap.write(tr);
        end
    end
endtask
```

父类的 `run_phase` 是空实现，调不调用super都可以。

Monitor 在每个时钟上升沿检查 `write` 信号，如果为高则采集当前地址和数据，创建一个 transaction 并通过分析端口发送。

`ap` 是 `uvm_analysis_port#(reg_transaction)` 类型的分析端口。`ap.write(tr)` 将当前 transaction 广播给所有连接到这个端口的组件。在 Env 的 `connect_phase` 中，monitor 的 `ap` 被连接到了 scoreboard 的 `mon_fifo`（一个 TLM analysis FIFO），因此 scoreboard 能接收到 monitor 发出的每一笔写 transaction。

## 十四、结果比对——Scoreboard

[⬆ 返回目录](#目录)

文件位置：`verif/reg_scoreboard.sv`

### 这一层加了什么

Scoreboard 是验证环境的裁判，继承自 `uvm_scoreboard`。它从两个数据路径接收信息，判断 DUT 的行为是否正确。写操作告诉它应该存什么值，读操作告诉它读回了什么值，它比较两者是否一致。

### 为什么需要单独分出这一层

验证的核心问题在于 DUT 的行为是否正确。Driver 负责发送数据，monitor 负责捕获数据，但检查数据正确性的逻辑应该独立于这两者。单独分出 scoreboard 层使得检查逻辑集中在一个地方，方便实现复杂的比较算法和统计功能。

### 双 FIFO 架构

Scoreboard 内部有两个 TLM analysis FIFO：

- `mon_fifo`：接收来自 monitor 的写操作 transaction。Scoreboard 使用这些信息构建一个预期存储器（expected_mem）。
- `drv_fifo`：接收来自 driver 的读操作 transaction。Scoreboard 使用这些信息提取实际读回的数据，与预期存储器中的值比较。

这两个 FIFO 使用 `fork...join_none` 在后台独立运行，各自处理各自的输入数据。`join_none` 使得 `run_phase` 不会因为两个 forever 循环而阻塞，仿真器可以在所有 transaction 处理完毕后结束 run phase。

### 比对逻辑

从 `mon_fifo` 收到写操作时，将 `addr` 和 `data` 存入 `expected_mem`。从 `drv_fifo` 收到读操作时，如果 `expected_mem` 中已有该地址的记录，比较实际数据和预期数据。匹配则 `pass_count` 增加，不匹配则 `fail_count` 增加并报告 `uvm_error`。如果该地址尚未被写入过，发出一个信息级别的消息。

### 报告阶段

在 `report_phase` 中输出最终统计结果：

```systemverilog
function void report_phase(uvm_phase phase);
    `uvm_info("SCB", $sformatf("=== 比对统计: 通过 %0d / 失败 %0d ===",
        pass_count, fail_count), UVM_LOW)
endfunction
```

这是验证工程师查看测试是否通过的重要参考。

## 十五、编译脚本——Makefile

[⬆ 返回目录](#目录)

文件位置：`Makefile`

### 基本用法

```bash
make sim TEST=reg_write_read_test   # 编译 + 运行写后读测试
make sim TEST=reg_boundary_test     # 编译 + 运行边界测试
make clean                          # 清理编译产物
make run_only                        # 仅运行（需已编译）
```

### 变量机制

Makefile 使用 `TEST` 变量控制运行哪个测试。`TEST ?= reg_write_read_test` 表示默认使用写后读测试，但可以通过命令行覆盖。`TEST` 变量在三个地方展开：编译输出文件名（`simv_$(TEST)`）、UVM 运行时参数（`+UVM_TESTNAME=$(TEST)`）、日志文件名（`$(TEST).log`）。

### VCS 编译选项

```
-full64             64 位模式
-sverilog           启用 SystemVerilog
-timescale=1ns/1ps  时间精度
+incdir+            包含目录
+define+UVM_NO_DPI  禁用 UVM 的 DPI 依赖
```

`UVM_NO_DPI` 这个宏定义非常重要。VCS 自带的 UVM 1.2 版本中的 DPI 功能在较老的 VCS 版本中可能无法正常链接。加上这个宏定义后，UVM 会通过 VCS 的内部机制获取测试名，而不是通过 DPI，避免了相关编译问题。

## 十六、仿真运行与结果解读

[⬆ 返回目录](#目录)

### 仿真输出结构

运行 `make sim TEST=reg_write_read_test` 后，VCS 输出以下关键信息。

**UVM 组件拓扑**

仿真开始时，UVM 打印完整的组件树形结构，用于确认环境层次关系是否正确：

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

**执行日志**

每笔 transaction 的执行过程中各个组件输出的信息。以一次写后读为例：

```
@ 15000: [MON] 采样到写 addr=0x95 data=0xeeadce97
@ 15000: [DRV] 写 addr=0x95 data=0xeeadce97
@ 15000: [SCB] 记录写 addr=0x95 data=0xeeadce97
@ 35000: [DRV] 读 addr=0x95 -> data=0xeeadce97
@ 35000: [SEQ] 写后读 addr=0x95 wdata=0xeeadce97 rdata=0xeeadce97
@ 35000: [SCB] 比对通过 addr=0x95 data=0xeeadce97
```

关键信息阅读要点：同一笔写操作，monitor 采样时间和 driver 驱动时间一致（@ 15000），表明时序正确；读回的数据 0xeeadce97 与写入的数据一致；Scoreboard 报告比对通过表示验证通过。

**UVM Report Summary**

仿真结束时的统计摘要，这是判断测试是否通过的核心依据：

```
--- UVM Report Summary ---
** Report counts by severity
UVM_INFO :   37
UVM_WARNING :    0
UVM_ERROR :    0
UVM_FATAL :    0
```

关键检查点在于 `UVM_ERROR` 和 `UVM_FATAL` 必须为 0。

**SCB 统计**

```
[SCB] === 比对统计: 通过 5 / 失败 0 ===
```

这是 scoreboard 自己的通过失败计数，直接给出了功能验证的结果。

### 写后读测试结果

```
写 addr=0x95 data=0xeeadce97 → 读 data=0xeeadce97 ✓
写 addr=0xe  data=0x62b884c7 → 读 data=0x62b884c7 ✓
写 addr=0x9f data=0x079204fe → 读 data=0x079204fe ✓
写 addr=0x70 data=0x234f278a → 读 data=0x234f278a ✓
写 addr=0xba data=0x1ad05721 → 读 data=0x1ad05721 ✓
UVM_ERROR: 0, UVM_FATAL: 0
SCB: 通过 5 / 失败 0
```

### 边界测试结果

```
写 addr=0x00 data=0xDEAD_BEEF → 读 data=0xdeadbeef ✓
写 addr=0xFF data=0x12345678 → 读 data=0x12345678 ✓
写 addr=0x64 data=0xABCD0000 → 读 data=0xabcd0000 ✓
UVM_ERROR: 0, UVM_FATAL: 0
SCB: 通过 3 / 失败 0
```

## 十七、数据流总览

[⬆ 返回目录](#目录)

整个验证环境的数据流可以概括为两条路径。

**写路径（从 sequence 到 scoreboard 的记录）：**

Sequence 创建 transaction，通过 `start_item()` 和 `finish_item()` 发送给 sequencer。Driver 获取 transaction 并驱动到 interface 上。DUT 接收写数据（`write=1` 时写入 `mem[addr]`）。同时，Monitor 从 interface 采样写信号，创建 transaction 通过分析端口发送，Scoreboard 的 `mon_fifo` 接收后记录 `expected_mem[addr] = data`。

**读路径（从 sequence 到 scoreboard 的比对）：**

Sequence 创建读 transaction 发送给 sequencer。Driver 获取 transaction 后驱动读地址到 interface。DUT 组合逻辑输出 `rdata = mem[addr]`。下一个时钟沿 driver 采样 `rdata` 写回 transaction。Driver 通过 `drv_ap` 发送读结果，Scoreboard 的 `drv_fifo` 接收后比对实际数据与 `expected_mem[addr]`。

这种双路径的设计解决了 DUT 无法提供可靠读握手信号的问题。Monitor 只采样写操作，不受 rvalid 的影响；driver 负责读操作的结果采集和汇报。

## 十八、常见编译与运行错误

[⬆ 返回目录](#目录)

### 文件未找到

```
make: *** No rule to make target `sim'.  Stop.
```

原因在于执行 make 的目录不是项目根目录。使用 `cd ~/reg_test` 切换到 Makefile 所在的目录。

### config_db 获取失败

```
[DRV] 从 config_db 获取 vif 失败
```

config_db 的路径字符串 `"uvm_test_top.env.agt.*"` 必须与 UVM 组件树中的实际路径匹配。如果 test 名称不同或 env 层次不同，需要对应调整。

### 仿真不结束

如果 scoreboard 的 `run_phase` 中使用了 `fork...join` 而不是 `fork...join_none`，两个 forever 循环会阻塞 `run_phase` 结束，仿真永远运行。因为 UVM 的 run phase 需要所有组件的 `run_phase` 都完成才能继续到 extract phase。使用 `fork...join_none` 将 forever 循环放在后台线程中，`run_phase` 执行到 `join_none` 后立即返回，不再阻塞。

### rvalid 竞争条件

如果 monitor 同时采样写和读，而 DUT 的 `rvalid` 始终为高，每个时钟周期都会被视为读操作。解决方案是让 monitor 只采样写操作，读操作由 driver 汇报。

### 编译时 UVM_TESTNAME 警告

`+UVM_TESTNAME` 是运行时参数，在编译阶段使用会在 VCS 编译日志中产生 warning。应该将其放在执行 `simv` 的命令行中。

### UVM_NO_DPI 未定义

如果没有定义 `+define+UVM_NO_DPI`，UVM 1.2 会尝试通过 DPI 获取测试名，在某些 VCS 版本中可能导致链接错误。加入该宏定义后，UVM 换用 VCS 内部接口获取测试名。

### 重复编译

每次运行 `make sim TEST=xxx` 时，VCS 都会生成一个独立的 `simv_xxx` 可执行文件。这是因为 VCS 是静态编译型仿真器，不同的 `UVM_TESTNAME` 实际上是在编译时确定的。如果要避免重复编译，可以使用 `make run_only TEST=xxx`，前提是 simv 文件已经编译过且代码未变化。

[⬆ 返回目录](#目录)

## 十九、学习问答

### 1. `function new(string name, uvm_component parent)` 为什么这么写？

`string` 是 SV 内置的字符串类型，`uvm_component` 是 UVM 库定义的类（所有组件的基类）。两个参数都是父类 `uvm_component` 的构造函数要求的——子类必须传这两个参数给 `super.new(name, parent)`，这是 SV 继承的语法规定。

### 2. `uvm_component` 是什么类型？

是一个**类**，不是 SV 关键字。UVM 库里定义的基类，所有组件（driver/monitor/agent/env/test）的祖先。

### 3. driver 里的 `req` 从哪来的？

`uvm_driver #(T)` 基类里已经声明了 `T req;`。当你写 `class reg_driver extends uvm_driver #(reg_transaction)`，T 被替换成 `reg_transaction`，所以 `req` 就是 `reg_transaction` 类型的句柄。

`#(type T = uvm_sequence_item)` 是参数化类的写法——T 是一个类型参数，默认是 `uvm_sequence_item`，子类可以指定具体类型。

### 4. `seq_item_port.get_next_item(req)` 怎么通信的？

`seq_item_port` 是 `uvm_driver` 基类内置的 port。它在 agent 的 connect_phase 里和 sequencer 的 `seq_item_export` 相连。`get_next_item` 阻塞等待，直到 sequence 通过 `finish_item` 把 transaction 发给 sequencer，sequencer 再转给 driver。

`get_next_item` 的参数是 `output` 方向——把值从函数里写出来给 `req`。变量名可以换（比如写成 `get_next_item(my_tr)`），效果一样。

`item_done()` 告诉 sequencer 这笔处理完了，sequence 的 `finish_item` 才能返回。

### 5. `uvm_analysis_port` 是什么？

是一个**类**（不是内置函数），功能是**广播**。`mon_port.write(tr)` 把 tr 发给所有连在这个 port 上的 receiver。

不是 `uvm_driver` 或 `uvm_monitor` 自带的——需要自己在组件里声明、在 build_phase 里 new、在 run_phase 里调 write、在 env 里 connect。

方法基本只有 `.write(tr)` 这一个。

### 6. modport 名字大小写有规定吗？

没有。`DRV` / `MON` / `drv` / `mon` / `A` / `B` 语法上都合法，只是标识符。

### 7. clocking block 的方向依据什么？

依据**硬件信号的真实流向**。从 Driver 发出给 DUT 的信号是 output（addr/wdata/write），从 DUT 返回给 Driver 的信号是 input（rdata/rvalid）。

### 8. `uvm_object_utils_begin...end` 比 `uvm_object_utils` 多了什么？

两者都注册 factory。`uvm_object_utils_begin` + `uvm_field_int` 额外注册了字段自动化——让 copy/compare/print 知道子类新增的字段。只用 `uvm_object_utils` 的话，父类不知道子类有 addr/data/write，copy 时这些字段会丢失。

### 9. 为什么子类构造函数必须调 `super.new(name)`？

SystemVerilog 规定：父类的构造函数有参数时，子类必须显式写构造函数来调用它。这是语言继承规则，不是 UVM 特有的。

### 10. `start_item` / `finish_item` 是谁的函数？

`uvm_sequence` 基类的内建 task。`start_item` 申请发送（拿号），`finish_item` 正式发出并等待 driver 处理完（交号）。中间的区域可以修改 transaction 字段。

### 11. `body()` 是什么？

UVM 约定好的入口 task。sequence 被 `seq.start(sequencer)` 启动后，系统自动调 `body()`。类似 test 的 `run_phase`、driver 的 `run_phase`。

### 12. `reg_transaction exp_mem[int]` 是什么语法？

关联数组（associative array），类似字典/哈希表。键是 int（地址），值是 reg_transaction。用 `exp_mem[addr] = tr` 存入，`exp_mem.exists(addr)` 检查键是否存在。

### 13. `uvm_tlm_analysis_fifo` 是什么？

全称 UVM Transaction-Level Modeling Analysis FIFO。自带缓存的 analysis port receiver——一头收 `write(tr)`，一头用 `get(tr)` 读取。`exp_fifo` 收期望值（写记录），`act_fifo` 收实际值（读结果）。

### 14. `fork...join_none` 的作用？

启动两个后台线程后立即继续，不等待。scoreboard 里两个 forever 循环（一个收写记录、一个收读结果）在后台独立运行，互不阻塞。

### 15. `report_phase` 必须加吗？为什么没有 `super`？

不加也可以，仿真结果一样只是没有统计输出。父类的 `report_phase` 是空函数，所以 `super.report_phase()` 调了也没效果——你是完全覆盖而不是扩展。

### 16. config_db 的 set/get 流程

```
test_top  set(null, "uvm_test_top", "vif", vif)     → 放到布告栏
test      get(this, "", "vif", vif)                   → 从布告栏取
test      set(this, "env.agent.driver", "vif", vif)   → 向下分发
test      set(this, "env.agent.monitor", "vif", vif)  → 向下分发
driver    get(this, "", "vif", vif)                   → 各自的 build 里取
monitor   get(this, "", "vif", vif)                   → 各自的 build 里取
```

### 17. 为什么要 package？

SystemVerilog 的命名空间容器。把所有类打包在一起，test_top 只需要 `import reg_pkg::*`，不需要写一大堆 `` `include ``。

### 18. sequencer 为什么每次都要继承，不能直接用基类？

可以直接用 `uvm_sequencer #(reg_transaction)`。继承是为了保留 factory override 的能力（`type_id::create`），以及未来扩展的入口。

### 19. `new` 函数到底在干什么？

创建对象时分配内存。`tr = new("tr")` 让 addr/data/write 在内存里真实存在。调 `super.new(name)` 让父类的字段也完成分配。`type_id::create` 内部最终还是调了 `new`。

[⬆ 返回目录](#目录)
