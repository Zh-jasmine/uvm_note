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

| 组件          | 文件                 | 核心职责                                 |
| ----------- | ------------------ | ------------------------------------ |
| Transaction | reg_transaction.sv | 封装一笔寄存器操作的地址、数据和读写标识                 |
| Interface   | reg_if.sv          | 将 DUT 和验证环境之间的信号封装为一个独立结构            |
| Driver      | reg_driver.sv      | 接收 transaction，按总线时序驱动到 interface 上  |
| Monitor     | reg_monitor.sv     | 被动采样 interface 上的信号，恢复成 transaction  |
| Sequencer   | reg_sequencer.sv   | 在 sequence 和 driver 之间路由 transaction |
| Sequence    | reg_sequence.sv    | 生成具体的读写激励序列                          |
| Scoreboard  | reg_scoreboard.sv  | 比较预期值与实际值，报告通过或失败                    |
| Agent       | reg_agent.sv       | 将 driver、monitor、sequencer 打包为可复用单元  |
| Env         | reg_env.sv         | 实例化 agent 和 scoreboard 并连接数据通路       |
| Test        | reg_base_test.sv   | 创建 env，控制仿真起止，启动 sequence            |
| Test Top    | test_top.sv        | 生成时钟复位，实例化 DUT 和 interface，启动 UVM    |

验证环境的数据流分为写路径和读路径两条。写指的是写入dut，从 sequence 产生写 transaction 开始，经 sequencer 转发给 driver，driver 驱动到 interface 后 ，DUT 存储数据。同时 monitor 从 interface 采样写信号，重建出 transaction 并通过分析端口发送给 scoreboard 的记录通道。
读是指从dut中读出数据，sequence 产生读 transaction，driver 驱动读地址后，DUT 输出读数据，driver 在下一个时钟沿采样读数据写回 transaction，然后直接汇报给 scoreboard 的比对通道。

项目文件按目录组织。`rtl/` 目录存放待验证的 RTL 代码，`tb/` 目录存放信号接口和仿真顶层，`verif/` 目录存放所有 UVM 验证组件，根目录的 Makefile 用于编译和运行。

### 待验证设计

文件位置：`rtl/reg_module.sv`

核心逻辑是一个 256x32 的寄存器阵列 `reg [31:0] mem [0:255]`。读操作是组合逻辑，`rdata` 始终等于 `mem[addr]`。写操作是时序逻辑，在时钟上升沿采样到 `write=1` 时将 `wdata` 写入 `mem[addr]`。`rvalid` 信号在复位结束后始终为高，不提供读完成握手信号。因此在本设计中无意义，只在复杂的总时序中有作用。

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

addr和 write和wdata在写操作时由 driver 驱动，rdata由 DUT 驱动。monitor 采样 write和 wdata。clk 由 test top 生成，rst_n 在仿真开始时由 test top 控制。

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

时钟周期 10ns（`#5` 翻转一次 = 100MHz）。`rst_n` 低电平持续 20ns（2 个周期）后拉高。

```systemverilog
reg clk;
initial clk = 0;
always #5 clk = ~clk;  // 周期 10ns

initial begin
    rst_n = 0;
    #20 rst_n = 1;       // 复位 2 个周期
end
```

### config_db 的两跳传递

`uvm_config_db` 是 UVM 的**全局布告栏**——一个组件 set，另一个组件 get，两者不需要互相知道对方存在。

**实际代码是两跳传递，不是直接从 test_top 到 driver：**

```systemverilog
// test_top.sv：第一跳 — 贴到 uvm_test_top 节点下
uvm_config_db #(virtual reg_if)::set(null, "uvm_test_top", "vif", vif);

// reg_base_test.sv connect_phase：第二跳 — 从 test 节点取到，再向下分发
function void connect_phase(uvm_phase phase);
    virtual reg_if vif;
    uvm_config_db #(virtual reg_if)::get(this, "", "vif", vif);      // 从本节点取
    uvm_config_db #(virtual reg_if)::set(this, "env.agent.driver", "vif", vif);   // 分发
    uvm_config_db #(virtual reg_if)::set(this, "env.agent.monitor", "vif", vif);  // 分发
endfunction
```

**为什么两跳？** test_top 是一个 module（非 UVM 组件），它不知道 UVM 组件树的内部结构。test_top 只能把 interface 放到 test 节点下，由 test 组件负责分发到各子组件。这保持 module 只管硬件层，UVM 内部路由由 UVM 组件自己处理。

**语法拆解：** interface 本质是硬件（一串 wire/reg），UVM 组件是软件（class 对象）。软件里没法"复制"一根导线，只能拿一个引用指着它，以后通过这个引用操作它的信号。所以 `uvm_config_db #(virtual reg_if)` 读作：「config_db 存储的类型是 指向 reg_if 的引用」。
`::set` / `::get` — 写入/读出；`null` — 第一个参数为 null 表示绝对路径,即第二个参数为绝对路径必须完整引用。`this` 表示相对路径（从当前组件开始）`"vif"` — 键名，可以随便起，但 receiver 必须用同样的名字取；最后的 `vif`是指实例化的interface。

所以整句话的意思： 在 UVM 树根节点（null）下，找到 `uvm_test_top` 这个节点，在里面放一个名叫 `"vif"` 的 virtual reg_if 引用，值指向 `test_top` 里例化的那个 interface。

当 driver 调 `get(this, "", "vif", vif)` 时，它在自己的 build_phase 里向上一层层找，直到在 `uvm_test_top` 下找到这个 `"vif"`，取出引用写到自己的 `vif` 句柄里。

**driver 和 monitor 各自 get：**
```systemverilog
uvm_config_db #(virtual reg_if)::get(this, "", "vif", vif);
// this = driver/monitor 自己，"" = 就在本节点找
```

每个需要 interface 的组件各自调一次 get，从全局布告栏自己取。

## 五、测试用例——Test

[⬆ 返回目录](#目录)

文件位置：`verif/reg_base_test.sv`

### 这一层加了什么

Test 是 UVM 验证层次的顶层组件，继承自 `uvm_test`。它负责创建 env、控制仿真的开始和结束、以及启动 sequence。

### 为什么需要单独分出这一层

不同的测试场景（写后读测试、边界测试、随机测试等）共享同一个 env 和 agent，但使用不同的 sequence 和配置。通过继承一个基类 test 并在子类中更换 sequence，可以在不修改 env 的情况下运行不同的测试。

### Objection 机制

UVM 的 objection 机制控制仿真何时结束。`raise_objection` 告诉 UVM「我还有活没干完，别结束」，`drop_objection` 说「干完了」。没有 objection，run_phase 一进来就立即退出，什么都不发生。

```systemverilog
task run_phase(uvm_phase phase);
    reg_write_read_seq seq;
    phase.raise_objection(this);                              // 告诉 UVM：我还有活
    seq = reg_write_read_seq::type_id::create("seq");
    seq.start(env.agent.sequencer);                            // 在 sequencer 上跑 sequence
    #100;                                                      // 等 100ns 让最后的比对完成
    phase.drop_objection(this);                                // 告诉 UVM：我干完了
endtask
```

`seq.start(env.agent.sequencer)` — `start` 是 `uvm_sequence` 基类的内建方法，把 sequence 挂到 sequencer 上执行。`env.agent.sequencer` 是层级路径：env → agent → sequencer。

### 完整代码（含 connect_phase 分发 interface）

```systemverilog
class reg_base_test extends uvm_test;
    `uvm_component_utils(reg_base_test)
    
    reg_env env;
    
    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction

    function void build_phase(uvm_phase phase);
        env = reg_env::type_id::create("env", this);
    endfunction

    function void connect_phase(uvm_phase phase);
        virtual reg_if vif;
        if (!uvm_config_db #(virtual reg_if)::get(this, "", "vif", vif))
            `uvm_fatal("TEST", "config_db 中没有 vif")
        uvm_config_db #(virtual reg_if)::set(this, "env.agent.driver", "vif", vif);
        uvm_config_db #(virtual reg_if)::set(this, "env.agent.monitor", "vif", vif);
    endfunction
    
    task run_phase(uvm_phase phase);
        reg_write_read_seq seq;
        phase.raise_objection(this);
        seq = reg_write_read_seq::type_id::create("seq");
        seq.start(env.agent.sequencer);
        #100;
        phase.drop_objection(this);
    endfunction
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
    agent.monitor.mon_port.connect(scoreboard.exp_fifo.analysis_export);
    agent.driver.rd_port.connect(scoreboard.act_fifo.analysis_export);
endfunction
```

两条线对应两条数据路径：monitor 的写操作进 `exp_fifo`（期望值），driver 的读结果进 `act_fifo`（实际值）。Scoreboard 里从两个 FIFO 分别取出比对。

## 七、组件打包——Agent

[⬆ 返回目录](#目录)

文件位置：`verif/reg_agent.sv`

### 这一层加了什么

Agent 将 driver、monitor 和 sequencer 这三个与同一套信号相关的组件打包在一起。它对上层的 env 隐藏了内部组件的创建和连接细节。

### 为什么需要单独分出这一层

在复杂的芯片验证项目中，同一个总线协议（如 AXI、APB）可能在多个模块中使用。将 driver、monitor、sequencer 打包成 agent 后，env 可以直接实例化这个 agent 多次，而不需要为每个模块重复创建和连接这些组件。

Agent 有两种工作模式。`UVM_ACTIVE` 模式包含 driver 和 sequencer（可发送激励），`UVM_PASSIVE` 模式只包含 monitor（仅观察）。对于只需要监测信号但不驱动信号的场景（比如验证环境中的从机接口），passive 模式非常有用。

### 内部连线

agent 只做两件事：创建子组件 + 连接 driver ↔ sequencer。

```systemverilog
function void build_phase(uvm_phase phase);
    driver    = reg_driver::type_id::create("driver", this);
    monitor   = reg_monitor::type_id::create("monitor", this);
    sequencer = reg_sequencer::type_id::create("sequencer", this);
endfunction

function void connect_phase(uvm_phase phase);
    driver.seq_item_port.connect(sequencer.seq_item_export);
endfunction
```

monitor 的 `mon_port` 和 driver 的 `rd_port` 不经 agent 中转——在 env 里直接连到 scoreboard 的 FIFO。agent 只负责把 driver 和 sequencer 的握手管道接通。

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
    rand bit [7:0]  addr;    // 寄存器地址，可随机化
    rand bit [31:0] data;    // 数据，可随机化
    rand bit        write;   // 读写标识，1=写，0=读

    constraint addr_range { addr inside {[0:9], [100:109], 200}; }

    `uvm_object_utils_begin(reg_transaction)
        `uvm_field_int(addr,  UVM_ALL_ON)
        `uvm_field_int(data,  UVM_ALL_ON)
        `uvm_field_int(write, UVM_ALL_ON)
    `uvm_object_utils_end

    function new(string name = "reg_transaction");
        super.new(name);
    endfunction
endclass
```

字段中 `addr`、`data` 和 `write` 被声明为 `rand`，允许 UVM 的 `randomize()` 自动产生随机值。

地址范围约束 `addr_range` 限制地址只能在特定范围（0~9、100~109、200），匹配 DUT 的寄存器地址空间。如果没有这个约束，随机化可能产生超出 DUT 地址范围的 transaction。

### `uvm_object_utils_begin...end` 和 `uvm_object_utils` 的区别

两个宏都注册 factory——让 UVM 知道 `reg_transaction` 这个类的存在，可以通过 `type_id::create("tr")` 创建实例。

但 `uvm_object_utils_begin` + `uvm_field_int` 额外做了**字段自动化**。它展开后自动生成 `do_copy`、`do_compare`、`do_print`、`do_pack`、`do_unpack` 这些函数，每个函数都会遍历 addr/data/write 三个字段。

如果不写 `uvm_object_utils_begin`，只用 `uvm_object_utils`，那么 UVM 只知道这个类的名字（可以 create），但不知道它有哪几个字段。在 scoreboard 里调 `tr1.compare(tr2)` 时，父类里默认的 `do_compare` 只比较了 `m_name`（对象名），不会比较 addr/data/write——因为父类根本不知道这些字段存在。

父类（`uvm_sequence_item` → `uvm_transaction` → `uvm_object`）编写的时候，`reg_transaction` 还不存在，父类自然不知道 addr/data/write。`uvm_field_int` 的作用就是告诉父类：「我新增了这些字段，copy/compare/print 的时候你也带上它们」。

`UVM_ALL_ON` 是一个位掩码，表示所有自动化功能对这三个字段全部开启。

### 构造函数 `function new(string name = "reg_transaction")`

这个 `new` 函数的第一责任是**创建对象、分配内存**。SystemVerilog 里声明 `reg_transaction tr;` 只产生一个空的句柄（值是 null），此时对象还不存在。只有 `tr = new("tr")` 执行之后，addr/data/write 这三个字段才在内存里真实存在，才能被读写。

那为什么要在子类里写一个 `new` 而不是直接用父类的 `new`？SystemVerilog 规定：**如果父类的构造函数有参数，子类必须显式写构造函数来调用它。** 父类 `uvm_object` 的构造函数签名是 `function new(string name)`，它有一个参数 `name`。编译器不会自动帮你调用 `super.new("reg_transaction")`，你必须手写。

`super.new(name)` 调用父类构造函数，把 `name` 参数往下传。父类内部把这个 `name` 存到 `m_name` 变量里，以后 `uvm_info` 打印的时候前面会显示这个对象的名字。如果不传 `super.new(name)`，对象名字就是空的，debug 时分不清这笔和那笔。

但细看这个构造函数，它什么都没干——唯一的逻辑就是 `super.new(name)`。这是绝大多数 transaction 类的真实模样：什么都不加，纯粹满足语法要求。

`name` 参数写 `= "reg_transaction"` 是默认值。这样 `new()` 不传参也能工作，`new("my_tr")` 传参也行。

### `type_id::create` 和 `new` 的关系

在 UVM 环境里，标准创建方式是 `tr = reg_transaction::type_id::create("tr")` 而不是 `tr = new("tr")`。`create` 内部最终也是调用 `new("tr")`，效果一样。

两者的区别：`create` 是 UVM factory 的入口。如果以后有人写了一个 `class reg_transaction_v2 extends reg_transaction`，通过 factory override 把这个新类注册到名字 `reg_transaction` 下，`type_id::create` 会自动创建新类的实例——不修改任何调用代码。直接 `new()` 则永远创建 `reg_transaction` 本身。这个能力就叫 factory override。

### 引用传递

SystemVerilog 的 class 句柄是引用传递，不是值传递。将 `tr` 作为参数传入 task 时，传的是对象地址而非副本。Driver 修改 `tr.data` 后，sequence 那端能看到改后的值。这一特性是 UVM 中 sequence 和 driver 之间数据回传的物理基础——driver 读完 DUT 后把 rdata 写入 `tr.data`，sequence 在 `finish_item` 返回后读 `tr.data` 就能拿到读结果。

## 九、信号连接——Interface

[⬆ 返回目录](#目录)

文件位置：`tb/reg_if.sv`

### 这一层加了什么

SystemVerilog 的 interface 将 DUT 与验证环境之间的所有信号封装到一个独立的结构中。在 UVM 中常用的传递方式是使用 `virtual interface` 指针——`virtual` 表示这是一个指向硬件接口的指针而不是接口本身。通过 `uvm_config_db` 在 test top 中设置，在 UVM 组件中获取。

### 为什么需要单独分出这一层

在传统 Verilog 验证中，信号通过端口逐层传递。如果信号需要新增或修改，需要修改多个文件的端口列表。Interface 将一组相关的信号封装在一起，验证组件只需要持有这个 interface 的指针就可以驱动或采样所有信号，无需关心顶层是如何连接的。

此外，interface 可以包含时钟块（clocking block）和 modport——前者定义时序关系，后者控制访问权限。

### 代码

```systemverilog
interface reg_if (input bit clk);
    logic [7:0]  addr;
    logic [31:0] wdata;
    logic        write;
    logic [31:0] rdata;
    logic        rvalid;

    clocking drv_cb @(posedge clk);
        output addr, wdata, write;
        input  rdata, rvalid;
    endclocking

    clocking mon_cb @(posedge clk);
        input addr, wdata, write, rdata, rvalid;
    endclocking

    modport DRV (clocking drv_cb);
    modport MON (clocking mon_cb);
endinterface
```

### 时钟参数 `input bit clk`

interface 把时钟作为参数单独传进来——因为所有时序操作都依赖它，时钟是 interface 的心跳。`bit` 是 SV 的二值逻辑（只有 0/1），相比 `logic`（四值：0/1/X/Z），`bit` 初始值默认是 0，不需要复位，适合做时钟这种从 0 开始翻转的信号。

### clocking block 的方向依据

`drv_cb` 和 `mon_cb` 里信号方向不是随便定的，依据的是**硬件信号的真实流向**。

看 DUT 的引脚：addr、wdata、write 是 driver 产生给 DUT 的，所以从 driver 角度看是 **output**。rdata、rvalid 是 DUT 返回给 driver 的，所以是 **input**。

```systemverilog
clocking drv_cb @(posedge clk);
    output addr, wdata, write;   // driver → DUT
    input  rdata, rvalid;        // DUT → driver
endclocking
```

`mon_cb` 从 monitor 角度看，所有信号都是被观察的，所以全是 input。Monitor 只看不驱动，只有一个 `input` 权限就够了。

方向不是由代码风格决定的——是硬件引脚的方向决定的。

### modport 名字有语法规定吗？

没有。`DRV` / `MON` 只是标识符，写成 `drv` / `mon` / `A` / `B` 语法上都合法。大写目的是在代码里一眼看出是 modport 引用，就像你给变量取名字一样——自己觉得清楚就好。

### driver 和 monitor 各取各的 clocking block

driver 声明句柄时指定 `DRV` 模式（或不指定，但驱动时用 `drv_cb`），monitor 声明时指定 `MON` 模式。同一个 interface 通过 modport 限制后，driver 不能错误地只用 input 视角，monitor 不能错误地驱动信号。这种从**物理上杜绝误操作**的机制，比靠人遵守规范更安全。

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

### 为什么需要继承——不能直接用基类吗？

可以。在 agent 里直接声明 `uvm_sequencer #(reg_transaction) sequencer;` 然后 `new("sequencer", this)` 也完全能跑。

定义一个空的 `reg_sequencer` 再继承的意义有两层：

1. **factory override** ——通过 `type_id::create` 创建实例，以后可以 override 成别的类。直接用 `new()` 创建基类实例，factory 没法管理
2. **预留扩展入口** ——复杂项目可能需要在 sequencer 里加排程策略或回调函数。空壳子留着，以后往里填东西不伤筋动骨

`new(string name, uvm_component parent)` 的签名是 UVM 组件的规定。`name` 是组件在 UVM 树中的名字（打印日志和 config_db 路径时使用），`parent` 是父节点指针。这两个参数由 `type_id::create("sqr", this)` 在实例化时传入，验证工程师只需按这个固定格式写构造函数。

## 十一、验证场景——Sequence

[⬆ 返回目录](#目录)

文件位置：`verif/reg_sequence.sv`

### 这一层加了什么

Sequence 描述了验证工程师关心的测试场景——生成 transaction 并通过 sequencer 发送给 driver。一个 UVM sequence 本质上是一系列 transaction 的产生和调度逻辑。

### 为什么需要单独分出这一层

将测试场景与驱动时序分离意味着修改测试内容时不需要改动 driver 或 DUT 的任何代码。每次修改都可以减少修改范围，降低引入新 bug 的风险。需要跑不同的测试（随机测试、边界测试）时，只需要写一个新的 sequence 类即可。

### `body()` 是什么

`body()` 是 UVM 约定好的**入口 task**。sequence 被 `seq.start(sequencer)` 启动后，系统自动调 `body()`。和 test 的 `run_phase`、driver 的 `run_phase` 是同一个概念——都是 UVM 规定好的函数名，你把执行逻辑填进去，UVM 在对应的时间点调用它。

`body()` 是 `uvm_sequence` 基类里定义的虚 task，子类来重写它。正因为是重写，父类的 `body()`（空函数）被覆盖了，所以 `super.body()` 可调可不调——调了也没效果。

### `start_item` / `finish_item` 是谁的函数

它们是 **`uvm_sequence` 基类的内建 task**，不是哪个组件独有的。每个 sequence 在 `body()` 里直接调用。

`start_item(tr)` — 向 sequencer 申请发送权限，阻塞直到 sequencer 同意。sequencer 内部维护一个仲裁队列，如果多个 sequence 同时请求，由 sequencer 决定谁先发。

`finish_item(tr)` — 将 transaction 正式发给 sequencer，sequencer 转交给 driver，然后阻塞等待 driver 调 `item_done()` 说"处理完了"才返回。

`start_item` 和 `finish_item` 之间的区域叫 grab 区——transaction 已被标记为等待发送，但 driver 还没开始动它。sequence 可以在这段时间里修改 transaction 的字段，或执行 `randomize()`。把 `randomize()` 放在 start/finish 之间是 UVM 的标准用法。

### 基类 sequence

基类 `reg_base_sequence` 的 `body()` 里只打了一句 warning，它是一个空壳——只给别人继承用，不直接跑。所有子类 sequence 继承它，各自实现自己的 `body()`。

为什么写这个空壳？如果以后所有 sequence 都要在执行前后打印日志，改 base_sequence 的 `body()` 一次就够了，不需要改每个 sequence。

```systemverilog
class reg_base_sequence extends uvm_sequence #(reg_transaction);
    `uvm_object_utils(reg_base_sequence)
    function new(string name = "reg_base_sequence");
        super.new(name);
    endfunction
    task body();
        `uvm_info("SEQ", "base_sequence body 不应被直接调用", UVM_LOW)
    endtask
endclass
```

### 写后读测试

```systemverilog
class reg_write_read_seq extends reg_base_sequence;
    task body();
        reg_transaction tr;
        repeat (5) begin
            tr = reg_transaction::type_id::create("tr");
            start_item(tr);
            if (!tr.randomize())
                `uvm_error("SEQ", "randomize 失败")
            finish_item(tr);
        end
    endtask
endclass
```

每次循环创建一笔 transaction，`randomize()` 在 start/finish 之间执行，随机出 addr 和 data。由于 transaction 的 `addr_range` 约束了地址范围，randomize 的结果不会超出 DUT 可处理的地址。

### 边界测试

```systemverilog
class reg_boundary_seq extends reg_base_sequence;
    task body();
        reg_transaction tr;
        tr = reg_transaction::type_id::create("tr");
        start_item(tr);
        tr.addr = 8'h00; tr.data = 32'hDEAD_BEEF; tr.write = 1;
        finish_item(tr);
        // ... 同样方式测试 0xFF 和 100
    endtask
endclass
```

边界测试不随机化，直接手动赋值 addr 和 data，测试地址最小、最大和中间值。

### 注意：sequence 的注册宏

sequence 继承 `uvm_sequence`（不是 `uvm_component`），所以注册用 `uvm_object_utils` 而不是 `uvm_component_utils`。object 注册用于数据类，component 注册用于结构类。两者的核心区别：component 创建时需要 parent 参数来挂进树，object 不需要。

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

### `req` 从哪来的——参数化类 `#(T)`

代码里 `req` 没有声明过，它从 `uvm_driver #(T)` 基类来：

```systemverilog
// UVM 源码（简化）：
class uvm_driver #(type T = uvm_sequence_item) extends uvm_component;
    T   req;                                    // 这里声明了
    T   rsp;
    uvm_seq_item_pull_port #(T) seq_item_port;  // port 也声明了
endclass
```

`#(type T = uvm_sequence_item)` 是参数化类——T 是类型参数，默认 `uvm_sequence_item`。当 `class reg_driver extends uvm_driver #(reg_transaction)`，基类里所有 T 被替换成 `reg_transaction`。所以 `req` 和 `seq_item_port` 都是基类内置的。

### `get_next_item(req)` 和 `item_done()` ——握手协议

`get_next_item` 的参数是 `output` 方向——把值**写出来**给 `req`。所以 `get_next_item(req)` 的意思是：等 sequencer 有 transaction 了，把它塞进 `req`。变量名可换成 `get_next_item(my_tr)`，效果相同。

`item_done()` 告诉 sequencer「这笔处理完了」，sequencer 让 sequence 的 `finish_item` 返回。

完整握手：start_item → sequencer 仲裁 → finish_item 阻塞 → driver get_next_item 拿到 → 驱动 DUT → item_done → finish_item 返回。

### `uvm_analysis_port` — 不是内置的

`rd_port` **不是 `uvm_driver` 自带的**，是自己手动加上去的：

```systemverilog
uvm_analysis_port #(reg_transaction) rd_port;   // 声明句柄
rd_port = new("rd_port", this);                  // build_phase 里 new
rd_port.write(tr);                               // run_phase 里广播
```

`uvm_analysis_port` 是一个类，唯一方法是 `.write(tr)`——广播 transaction。这里连的是 scoreboard 的 `act_fifo`。

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
        @(vif.mon_cb);                          // 等时钟沿
        if (vif.mon_cb.write) begin              // 采到写操作
            tr = reg_transaction::type_id::create("tr");
            tr.addr  = vif.mon_cb.addr;          // 采地址
            tr.data  = vif.mon_cb.wdata;         // 采写数据
            tr.write = 1;
            mon_port.write(tr);                   // 广播给 scoreboard
        end
    end
endtask
```

父类的 `run_phase` 是空实现，调不调用super都可以。

Monitor 在每个时钟上升沿检查 `write` 信号，如果为高则采集当前地址和数据，创建一个 transaction 并通过分析端口发送。

`mon_port` 是 `uvm_analysis_port #(reg_transaction)` 类型的分析端口——一个类实例，**不是 `uvm_monitor` 基类自带的**，需要手动在 monitor 里声明并 new。`ap.write(tr)` 将当前 transaction 广播给所有连接到这个端口的组件。在 Env 的 `connect_phase` 中，monitor 的 `ap` 被连接到了 scoreboard 的 `mon_fifo`（一个 TLM analysis FIFO），因此 scoreboard 能接收到 monitor 发出的每一笔写 transaction。

## 十四、结果比对——Scoreboard

[⬆ 返回目录](#目录)

文件位置：`verif/reg_scoreboard.sv`

### 这一层加了什么

Scoreboard 是验证环境的裁判，继承自 `uvm_scoreboard`。它从两个数据路径接收信息，判断 DUT 的行为是否正确。写操作告诉它应该存什么值，读操作告诉它读回了什么值，它比较两者是否一致。

### 为什么需要单独分出这一层

验证的核心问题在于 DUT 的行为是否正确。Driver 负责发送数据，monitor 负责捕获数据，但检查数据正确性的逻辑应该独立于这两者。单独分出 scoreboard 层使得检查逻辑集中在一个地方，方便实现复杂的比较算法和统计功能。

### 双 FIFO 架构

Scoreboard 内部有两个 TLM analysis FIFO：

- `exp_fifo`（expected FIFO）：接收来自 monitor 的写操作 transaction，用于构建预期值
- `act_fifo`（actual FIFO）：接收来自 driver 的读操作 transaction，即实际读回的值

### `uvm_tlm_analysis_fifo` 是什么

全称 UVM Transaction-Level Modeling Analysis FIFO。它是一个**自带缓存的 analysis port receiver**——一头通过 `analysis_export` 收 `.write(tr)` 广播，一头用 `get(tr)` 阻塞读取。当 FIFO 为空时，`get(tr)` 会等待直到有数据到达才返回。

和普通 `uvm_analysis_port` 的区别在于：port 只是一个出口，fifo 是"出口 + 缓存 + 入口"。Monitor 和 driver 通过 port 发出来，scoreboard 通过 fifo 接住并排队处理。

### `reg_transaction exp_mem[int]` ——关联数组

这行声明了一个**关联数组**（associative array），类似字典/哈希表。键是 `int` 类型（地址），值是 `reg_transaction` 类型。`exp_mem[addr] = tr` 在内部分配空间存储这个键值对。`exp_mem.exists(addr)` 检查该键是否存在。

普通数组 `arr[256]` 要求下标 0~255 全部分配好；关联数组用到哪个下标才分配哪个，不浪费内存。

### `fork...join_none` ——三个变体

```systemverilog
fork
    forever begin exp_fifo.get(exp_tr); exp_mem[exp_tr.addr] = exp_tr; end  // 线程 1
    forever begin act_fifo.get(act_tr); /* 比对 */                      end  // 线程 2
join_none
```

`fork...join_none` ——启动两个后台线程后**立即继续，不等待**。如果用 `fork...join`，run_phase 会永远卡在这，仿真不结束。`join_none` 让 run_phase 返回，两个线程在后台并行跑，仿真的结束由 test 的 objection 机制控制。

三种 fork 行为对比：

| 变体 | 行为 |
|---|---|
| `fork...join` | 等所有线程跑完才继续 |
| `fork...join_any` | 等任意一个线程跑完就继续 |
| `fork...join_none` | 启动后立即继续，不等待 |

### 比对逻辑

从 `mon_fifo` 收到写操作时，将 `addr` 和 `data` 存入 `expected_mem`。从 `drv_fifo` 收到读操作时，如果 `expected_mem` 中已有该地址的记录，比较实际数据和预期数据。匹配则 `pass_count` 增加，不匹配则 `fail_count` 增加并报告 `uvm_error`。如果该地址尚未被写入过，发出一个信息级别的消息。

### 报告阶段

`report_phase` 不是必须加的——不加它仿真能照样跑完，只是没有统计输出。父类 `uvm_component` 的 `report_phase` 是空函数，所以 `super.report_phase(phase)` 调不调都无所谓——你是**完全覆盖**而不是扩展。

```systemverilog
function void report_phase(uvm_phase phase);
    `uvm_info("SCB", $sformatf("PASS=%0d  FAIL=%0d", pass_count, fail_count), UVM_LOW)
endfunction
```

### 完整代码

```systemverilog
class reg_scoreboard extends uvm_scoreboard;
    `uvm_component_utils(reg_scoreboard)

    reg_transaction exp_mem[int];  // 关联数组，addr → 期望值
    uvm_tlm_analysis_fifo #(reg_transaction) exp_fifo;  // 收写记录
    uvm_tlm_analysis_fifo #(reg_transaction) act_fifo;  // 收读结果
    int pass_count, fail_count;

    function void build_phase(uvm_phase phase);
    exp_fifo = new("exp_fifo", this);
    act_fifo = new("act_fifo", this);
    endfunction

    task run_phase(uvm_phase phase);
        fork
            forever begin  // 线程 1：收写记录
                reg_transaction exp_tr;
                exp_fifo.get(exp_tr);
                exp_mem[exp_tr.addr] = exp_tr;
            end
            forever begin  // 线程 2：收读结果，比对
                reg_transaction act_tr;
                act_fifo.get(act_tr);
                if (exp_mem.exists(act_tr.addr)) begin
                    if (act_tr.data == exp_mem[act_tr.addr].data)
                        pass_count++;
                    else
                        fail_count++;
                end
            end
        join_none
    endtask

    function void report_phase(uvm_phase phase);
        `uvm_info("SCB", $sformatf("PASS=%0d  FAIL=%0d", pass_count, fail_count), UVM_LOW)
    endfunction
endclass
```

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

Sequence 创建 transaction，通过 `start_item` 和 `finish_item` 发给 sequencer。Driver 获取后驱动到 interface。DUT 在 `write=1` 时将 wdata 写入 `mem[addr]`。同时 Monitor 采样写信号，通过 `mon_port` 发送，Scoreboard 的 `exp_fifo` 接收后记录 `exp_mem[addr] = tr`。

**读路径（从 sequence 到 scoreboard 的比对）：**

Sequence 创建读 transaction，Driver 驱动读地址到 interface。DUT 组合逻辑输出 `rdata = mem[addr]`。下一个时钟沿 driver 采样 `rdata` 写回 transaction，通过 `rd_port` 发送。Scoreboard 的 `act_fifo` 接收后从 `exp_mem[addr]` 取出期望值比对。

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
