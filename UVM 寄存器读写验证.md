## 目录

- [[#一、验证环境总览]]
- [[#二、信号说明]]
- [[#三、总线时序]]
- [[#四、数据载体——Transaction]]
- [[#五、信号连接——Interface]]
- [[#六、时序驱动——Driver]]
- [[#七、信号采样——Monitor]]
- [[#八、激励调度——Sequencer]]
- [[#九、验证场景——Sequence]]
- [[#十、结果比对——Scoreboard]]
- [[#十一、组件打包——Agent]]
- [[#十二、环境组装——Env]]
- [[#十三、仿真顶层——Test Top]]
- [[#十四、测试用例——Test]]
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

核心逻辑是一个 256x32 的寄存器阵列 `logic [31:0] mem [0:255]`。读写操作都在时钟上升沿触发，rdata 和 rvalid 均为寄存器输出。

```systemverilog
module reg_module (
    input  logic       clk,
    input  logic       rst_n,
    input  logic [7:0] addr,
    input  logic [31:0] wdata,
    input  logic       write,
    output logic [31:0] rdata,
    output logic       rvalid
);

    logic [31:0] mem [0:255];

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            rvalid <= 0;
        end else begin
            if (write) begin
                mem[addr] <= wdata;
                rvalid <= 0;
            end else begin
                rdata <= mem[addr];
                rvalid <= 1;
            end
        end
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
| rvalid | 输出 | 1bit | 读有效指示，读操作时拉高一个周期，复位和写操作时拉低 |

addr和 write和wdata在写操作时由 driver 驱动，rdata由 DUT 驱动。monitor 采样 write和 wdata。clk 由 test top 生成，rst_n 在仿真开始时由 test top 控制。
## 三、总线时序

[⬆ 返回目录](#目录)

本项目的总线协议非常简洁，读写各需两个时钟周期。

写操作。第一个时钟上升沿将 `addr` 设为目标地址，`wdata` 设为写入数据，`write` 设为 1。DUT 在此沿采样到 `write` 为高后将 `wdata` 存入 `mem[addr]`。第二个时钟上升沿将 `write` 设为 0，写操作完成。

读操作。第一个时钟上升沿将 `addr` 设为目标地址，`write` 设为 0。由于 `rdata` 是组合逻辑输出，地址设置后 `rdata` 立即变为 `mem[addr]`。第二个时钟上升沿 driver 采样 `rdata`，完成读操作。

所有信号变化均以时钟上升沿为基准。写操作涉及 addr、wdata、write 三个信号的驱动，读操作只驱动 addr 和 write，然后由 driver 采样 rdata。
## 四、数据载体——Transaction

[⬆ 返回目录](#目录)

文件位置：`verif/reg_transaction.sv`

### 这一层加了什么

Transaction 在 UVM 中对应 `uvm_sequence_item` 的子类，它描述了验证环境中流动的一笔数据。在寄存器读写场景中，一笔 transaction 包含地址（addr）、数据（data）和读写标识（write）三个字段。

### 为什么需要单独分出这一层

如果没有 transaction 这一层，driver 和 scoreboard 之间就需要直接传递地址总线和数据总线的值，也就是多个信号的拼接。这样做的问题在于信号散布在多个组件中，一旦接口发生变化（比如地址位宽从 8bit 变成 16bit），每个组件都需要修改。Transaction 层将这些字段封装成一个独立的数据对象，所有组件都依赖这个对象定义，接口变化时只需要修改 transaction 一个地方。

此外，`uvm_sequence_item` 提供了 `randomize()` 方法支持随机化，这是 UVM 验证方法学的核心优势之一。通过在 transaction 中声明 `rand` 字段和约束，sequence 可以方便地生成随机激励。

### 完整代码

文件位置：`verif/reg_transaction.sv`

```systemverilog
class reg_transaction extends uvm_sequence_item;

    rand bit [7:0]  addr;
    rand bit [31:0] data;
    rand bit        write;

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

### 字段定义

transaction 的三个字段 `addr`、`data` 和 `write` 被声明为 `rand`，允许 UVM 的 `randomize()` 自动产生随机值。

地址范围约束 `addr_range` 限制地址只能在特定范围（0~9、100~109、200），匹配 DUT 的寄存器地址空间。如果没有这个约束，随机化可能产生超出 DUT 地址范围的 transaction。

### `uvm_object_utils_begin...end` 和 `uvm_object_utils` 的区别

两个宏都注册 factory——让 UVM 知道 `reg_transaction` 这个类的存在，可以通过 `type_id::create("tr")` 创建实例。

但 `uvm_object_utils_begin` + `uvm_field_int` 额外做了**字段自动化**。它展开后自动生成 `do_copy`、`do_compare`、`do_print`、`do_pack`、`do_unpack` 这些函数，每个函数都会遍历 addr/data/write 三个字段。

如果不写 `uvm_object_utils_begin`，只用 `uvm_object_utils`，那么 UVM 只知道这个类的名字（可以 create），但不知道它有哪几个字段。在 scoreboard 里调 `tr1.compare(tr2)` 时，父类里默认的 `do_compare` 只比较了 `m_name`（对象名），不会比较 addr/data/write——因为父类根本不知道这些字段存在。

父类（`uvm_sequence_item` → `uvm_transaction` → `uvm_object`）编写的时候，`reg_transaction` 还不存在，父类自然不知道 addr/data/write。`uvm_field_int` 的作用就是告诉父类：「我新增了这些字段，copy/compare/print 的时候你也带上它们」。

`UVM_ALL_ON` 是一个位掩码，表示所有自动化功能对这三个字段全部开启。

### 构造函数 `function new(string name = "reg_transaction")`

这个 `new` 函数的第一责任是**创建对象、分配内存**。SystemVerilog 里声明 `reg_transaction tr;` 只产生一个空的句柄（值是 null），此时对象还不存在。只有 `tr = new("tr")` 执行之后，addr/data/write 这三个字段才在内存里真实存在，才能被读写。

`new()` 的特殊之处在于它是 SystemVerilog 中唯一强制链式调用的函数。普通 `function` 在子类中定义同名函数时叫隐藏（hiding），父类版本被完全覆盖，调不调 `super` 都行。`virtual function` 调用哪种版本取决于实际对象类型而不是句柄类型，`super` 可选——调了就保留父类逻辑，不调就完全重写。但 `new()` 既不是隐藏也不是 virtual——它是链式调用的构造函数，子类必须写 `super.new()`，如果子类不调用父类的new函数，则父类的new完全不构造，编译会报错。

### `type_id::create` 和 `new` 的关系

在 UVM 环境里，标准创建方式是 `tr = reg_transaction::type_id::create("tr")` 而不是 `tr = new("tr")`。`create` 内部最终也是调用 `new("tr")`，效果一样。

两者的区别：`create` 是 UVM factory 的入口。如果以后有人写了一个 `class reg_transaction_v2 extends reg_transaction`，通过 factory override 把这个新类注册到名字 `reg_transaction` 下，`type_id::create` 会自动创建新类的实例——不修改任何调用代码。直接 `new()` 则永远创建 `reg_transaction` 本身。这个能力就叫 factory override。

### 引用传递

SystemVerilog 的 class 句柄是引用传递，不是值传递。将 `tr` 作为参数传入 task 时，传的是对象地址而非副本。Driver 修改 `tr.data` 后，sequence 那端能看到改后的值。这一特性是 UVM 中 sequence 和 driver 之间数据回传的物理基础——driver 读完 DUT 后把 rdata 写入 `tr.data`，sequence 在 `finish_item` 返回后读 `tr.data` 就能拿到读结果。
## 五、信号连接——Interface

[⬆ 返回目录](#目录)

文件位置：`tb/reg_if.sv`

### 这一层加了什么

SystemVerilog 的 interface 将 DUT 与验证环境之间的所有信号封装到一个独立的结构中。在 UVM 中常用的传递方式是使用 `virtual interface` 指针——`virtual` 表示这是一个指向硬件接口的指针而不是接口本身。通过 `uvm_config_db` 在 test top 中设置，在 UVM 组件中获取。

### 为什么需要单独分出这一层

在传统 Verilog 验证中，信号通过端口逐层传递。如果信号需要新增或修改，需要修改多个文件的端口列表。Interface 将一组相关的信号封装在一起，验证组件只需要持有这个 interface 的指针就可以驱动或采样所有信号，无需关心顶层是如何连接的。

此外，interface 可以包含时钟块（clocking block）和 modport——前者定义时序关系，后者控制访问权限。

### 完整代码

文件位置：`tb/reg_if.sv`

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

### 概念讲解

#### 1. `interface`

**是什么**

`interface` 是 SystemVerilog 的关键字，它是一种封装结构，将一组相关的信号打包在一起。相比 Verilog 的 `module`，interface 多了 clocking block、modport、断言等验证专用的功能，所以验证环境与 DUT 之间的信号连接优先使用 interface 而不是 module。

**语法/用法**

```systemverilog
interface 接口名 (端口参数);
    信号声明;
    // 可包含 clocking block、modport、task、function、断言等
endinterface
```

例化方式与 module 相同：

```systemverilog
reg_if vif(clk);     // 例化名为 vif 的 interface，时钟参数为 clk
```

**细节/注意的点**

- 在 UVM 中，interface 不能直接在 class 内部声明——硬件（wire/reg）和软件（class 对象）生活在两个世界里。必须通过 `virtual` 指针的方式来引用，`virtual reg_if vif` 的意思是"一个指向 reg_if 类型接口的指针"。
- 一个 interface 实例可以被多个 virtual interface 指针引用（driver 和 monitor 各持一个指针指向同一个 interface）。
- 传递方式：在 test top（module）中例化，通过 `uvm_config_db` 传到 UVM 组件中。

#### 2. `clocking block`

**是什么**

clocking block 是 interface 内部的一种声明结构，定义信号在时钟沿上的输入输出方向和时序关系。它规定了一组信号在哪个时钟沿采样和驱动，以及从使用者的视角看这些信号是输入还是输出。

**语法/用法**

```systemverilog
clocking 名称 @(posedge 时钟信号);
    output 信号1, 信号2;       // 从当前组件输出给 DUT
    input  信号3, 信号4;       // 从 DUT 输入给当前组件
endclocking
```

本项目定义了两个 clocking block，给不同的使用者：

```systemverilog
clocking drv_cb @(posedge clk);
    output addr, wdata, write;   // driver 输出给 DUT
    input  rdata, rvalid;        // DUT 返回给 driver
endclocking

clocking mon_cb @(posedge clk);
    input addr, wdata, write, rdata, rvalid;   // monitor 只看，全是输入
endclocking
```

方向依据的是硬件信号的真实流向：addr/wdata/write 是 driver 产生给 DUT 的，所以从 driver 角度看是 output；rdata/rvalid 是 DUT 返回给 driver 的，所以是 input。monitor 只看不驱动，所以 mon_cb 所有信号都是 input。

**细节/注意的点**

- clocking block 中的 output 信号驱动时默认使用非阻塞赋值（`<=`），在时钟沿之后改变（`#1step`）。
- clocking block 中的 input 信号在时钟沿之前采样（`#0`）。driver 在时钟沿之后驱动，DUT 在时钟沿捕获，monitor 在时钟沿之前采样。这个时序保证了采样到的信号是稳定值。
- 所以 clocking block的时序满足保持时间和建立时间，比起在项目中手动使用`@(posedge clk)`和延时即方便也更稳定
- 在 driver/monitor 中使用 `@(vif.drv_cb)` 可以同步到时钟沿，不需要再写 `@(posedge clk)`。
- clocking block 的方向是从**使用它的组件看出去**的方向，不是从 DUT 看出去的方向。
- 同一组物理信号可以在不同的 clocking block 中有不同的方向声明——driver 和 monitor 各取所需。

#### 3. `modport`

**是什么**

modport 是 interface 中为不同使用者定义的不同"窗口"或"视角"。它控制使用者可以访问 interface 中的哪些 clocking block 和信号，以及以什么方向访问。

**语法/用法**

```systemverilog
modport 名称 (clocking 时钟块名);
```

本项目定义了两个 modport：

```systemverilog
modport DRV (clocking drv_cb);   // driver 的视角：drv_cb
modport MON (clocking mon_cb);   // monitor 的视角：mon_cb
```

driver 声明句柄时指定 modport：

```systemverilog
virtual reg_if.DRV vif;   // 采用 reg_if 的 DRV 模式
```

这句话的意思：采用 reg_if 接口的 DRV 模式作为这个 driver 的信号连接方式，名字叫 vif，后续通过 `vif.drv_cb.addr`、`vif.drv_cb.rdata` 引用具体信号。

**细节/注意的点**

- modport 的名字没有语法限制——`DRV`、`MON`、`drv`、`mon`、甚至 `A`、`B` 都合法，只是标签。
- modport 不创建新信号，只是为同一组信号提供不同的访问视图。
- 使用 modport 可以在编译层面防止误操作：如果 monitor 拿到的是 MON 模式（只有 input 权限），代码中误写了驱动语句会直接编译报错。这种从物理上杜绝误操作的机制比靠人遵守规范更安全。
- 如果不指定 modport，组件可以访问 interface 中的所有信号（可读可写）。
- modport 也可以不通过 clocking block 直接声明信号方向：`modport DRV (output addr, wdata, write, input rdata, rvalid)`。本项目把所有方向信息放在 clocking block 中，modport 只负责指定使用哪个 clocking block——分工更清晰。

Modport 和 clocking block 解决的是不同层面的问题，不冲突也不重复。Clocking block 是一个纯验证环境侧的时序构造，它定义了信号相对于时钟沿的采样和驱动时刻，能保证信号符合建立/保持时间，但 clocking block 不是一组可连线的物理信号，因此不能用于例化 DUT 时连接端口。Modport 从 interface 的信号集合中挑选出一个子集，为每根信号声明方向（input/output/inout），主要服务于两个场景：一是给 DUT 例化提供可连线的接口视图（DUT 端口必须连真实的 wire，无法连 clocking block），二是让 monitor 等组件获得仅采样的方向保护，编译器可以在误驱动时报错。
在 UVM 组件内部只做激励或采样、不需要连线 DUT 也不需方向保护的场景下，直接引用 clocking block 就足够了，不一定需要modport。

#### 4. 时钟参数 `input bit clk`

**是什么**

interface 把时钟作为端口参数单独传入，因为 clocking block 的所有时序操作（`@(posedge clk)`）都依赖它。

**语法/用法**

```systemverilog
interface reg_if (input bit clk);
    clocking drv_cb @(posedge clk);  // 引用 clk
endinterface
```

**细节/注意的点**

- `bit` 是 SystemVerilog 的二值逻辑（只有 0 和 1），`logic` 是四值逻辑（0/1/X/Z）。时钟信号不需要 X/Z，用 `bit` 足够，且 `bit` 默认初始值是 0（`logic` 默认是 X），适合做从 0 开始翻转的时钟信号。用 `logic` 也可以，效果一样。
- 时钟作为参数绑在 interface 里，而不是分开声明，是为了让 clocking block 能引用到它。

### driver 和 monitor 如何使用 modport

driver 声明 `virtual reg_if vif`（不指定 modport），但驱动时用 `vif.drv_cb.addr` 来引用 drv_cb 中的信号。monitor 同样声明 `virtual reg_if vif`，使用 `vif.mon_cb.addr`。同一个 interface 实例被两个 virtual 指针指向，但通过不同的 clocking block 获得不同的输入输出方向——driver 能驱动信号，monitor 只能采样信号。

## 六、时序驱动——Driver

[⬆ 返回目录](#目录)

文件位置：`verif/reg_driver.sv`

### 这一层加了什么

Driver 继承自 `uvm_driver #(reg_transaction)`，负责从 sequencer 接收 transaction，按照 DUT 的总线时序驱动到 interface 上。它把 transaction 层面的数据（addr、data、write）转换成信号层面的时序（时钟沿、信号赋值）。

### 为什么需要单独分出这一层

验证工程师需要产生各种总线操作（读、写、空闲等），这些操作的时序是固定的。如果每个 sequence 都直接操作 interface 信号，代码会大量重复且容易引入时序错误。Driver 层将这些时序逻辑集中在一个地方，sequence 只需要描述要做什么，driver 负责怎么驱动。

### 完整代码

文件位置：`verif/reg_driver.sv`

```systemverilog
class reg_driver extends uvm_driver #(reg_transaction);

    `uvm_component_utils(reg_driver)

    virtual reg_if vif;

    uvm_analysis_port #(reg_transaction) rd_port;

    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction

    function void build_phase(uvm_phase phase);
        rd_port = new("rd_port", this);
        if (!uvm_config_db #(virtual reg_if)::get(this, "", "vif", vif))
            `uvm_fatal("DRV", "vif 没有从 config_db 传进来")
    endfunction

    task run_phase(uvm_phase phase);
        forever begin
            seq_item_port.get_next_item(req);
            drive_item(req);
            seq_item_port.item_done();
        end
    endtask

    task drive_item(reg_transaction tr);
        @(vif.drv_cb);
        vif.drv_cb.addr  <= tr.addr;
        vif.drv_cb.wdata <= tr.data;
        vif.drv_cb.write <= tr.write;

        @(vif.drv_cb);
        vif.drv_cb.write <= 0;

        if (!tr.write) begin
            @(vif.drv_cb);
            tr.data = vif.drv_cb.rdata;
            rd_port.write(tr);
        end
    endtask

endclass
```

### 概念讲解
#### 1. `uvm_driver #(T)` — 参数化驱动基类

**是什么**

`uvm_driver #(T)` 是一个参数化类，`#(T)` 表示它接受一个类型参数，默认是 `uvm_sequence_item`。当你写 `extends uvm_driver #(reg_transaction)` 时，基类源码里所有的 `T` 都被替换成 `reg_transaction`。

**语法/用法**

```systemverilog
// UVM 源码中的定义（简化）
class uvm_driver #(type T = uvm_sequence_item) extends uvm_component;
    T   req;                                    // 类型为 T 的句柄
    T   rsp;                                    // 类型为 T 的句柄
    uvm_seq_item_pull_port #(T) seq_item_port;  // 通信端口
    ...
endclass

// 实际声明时指定具体类型
class reg_driver extends uvm_driver #(reg_transaction);
    // 继承后基类中的展开结果：
    // reg_transaction req;
    // reg_transaction rsp;
    // uvm_seq_item_pull_port #(reg_transaction) seq_item_port;
```

**细节/注意的点**

- `#(T)` 里的 `T` 表示「这是一个参数，具体类型后面再定」。写 `uvm_driver #(reg_transaction)` 就是告诉编译器：「把基类里所有 T 替换成 reg_transaction」。
- `req` 是基类内置的句柄，类型和参数 T 一致。你也可以不用内置的 `req`，自己声明一个变量传进去：

```systemverilog
reg_transaction my_tr;
seq_item_port.get_next_item(my_tr);    // 一样能用
```

- `uvm_driver` 继承自 `uvm_component`，所以构造函数需要 `(string name, uvm_component parent)` 两个参数，注册用 `uvm_component_utils`。

#### 2. `seq_item_port` — driver 与 sequencer 的管道

**是什么**

`seq_item_port` 是 `uvm_driver #(T)` 内置的一个成员变量，它的类型是 `uvm_seq_item_pull_port #(T)`——这个类本身。

``` systemverilog
// uvm_driver 内部（简化）
class uvm_driver #(type T = uvm_sequence_item) extends uvm_component;
    uvm_seq_item_pull_port #(T) seq_item_port;   // ← 成员变量，类型是这个 port 类
    //                        ↑ 这个才是类名
endclass

```

`seq_item_port` 是 driver 和 sequencer 之间的 TLM 通信端口，类型 
`uvm_seq_item_pull_port #(T)`这个类。它由 `uvm_driver` 基类内置，driver 不需要手动创建。
所以`seq_item_port`是指向 `uvm_seq_item_pull_port #(T)`类的句柄，`seq_item_port.get_next_item(req);`的意思就是引用其get_next_item方法。get_next_item的作用就是阻塞等待，直到 sequencer 那边有 transaction 可以拿，然后把 transaction 的句柄塞进 `req`。

TLM：Transaction-Level Modeling，是一种用于数字系统设计的高级建模方法，通过分离功能模块与通信机制实现系统级抽象。

**语法/用法**

```systemverilog
seq_item_port.get_next_item(req);    // 阻塞：等 sequencer 分发 transaction
// ... 驱动 transaction 到 DUT ...
seq_item_port.item_done();           // 通知 sequencer：这笔处理完了
```

**细节/注意的点**

- `get_next_item` 的参数是 `output` 方向——sequencer 把 transaction **写进** `req`。
- `item_done()` 告诉 sequencer「我干完了」，sequencer 让 sequence 的 `finish_item` 返回，sequence 继续造下一笔。
- 完整握手链路：sequence `start_item(tr)` → 等待仲裁 → `finish_item(tr)` → sequencer 转发 → driver `get_next_item(req)` 拿到 → 驱动 DUT → `item_done()` → sequence `finish_item` 返回。
- `get_next_item` 和 `item_done` 是 driver 侧的方法，它们分别对应 sequence 侧的 [[#九、验证场景——Sequence|start_item 和 finish_item]]。



#### 3. `uvm_analysis_port`

**是什么**

`uvm_analysis_port` 是一个广播端口类。调用它的 `.write(tr)` 方法时，所有连接在这个 port 上的 receiver 都会收到 `tr` 的拷贝。**它不是 `uvm_driver` 或 `uvm_monitor` 基类自带的，需要手动声明和创建。**

**语法/用法**

```systemverilog
// 1. 声明句柄（类内部，不在函数内）
uvm_analysis_port #(reg_transaction) rd_port;

// 2. 创建实例（build_phase 中）
rd_port = new("rd_port", this);

// 3. 广播（run_phase 中）
rd_port.write(tr);
```

**细节/注意的点**

- 分析端口只有 `.write(tr)` 一个方法。——它只负责往外扔，不关心谁接收、有几个接收、接收后怎么处理。
- 与 `seq_item_port` 的核心区别：`analysis_port` 是一对多广播、不阻塞、不需要应答。`seq_item_port` 是一对一手握、阻塞、需要 `item_done()` 回执。
- 这里 `rd_port` 在 env 的 connect_phase 中连到 scoreboard 的 `act_fifo`。所以在 driver 里调 `rd_port.write(tr)` 后，scoreboard 那边能从 `act_fifo` 拿到这笔 transaction。

### drive_item 详解

```systemverilog
task drive_item(reg_transaction tr);
    @(vif.drv_cb);                           // 等第一个时钟沿
    vif.drv_cb.addr  <= tr.addr;             // 地址放上总线
    vif.drv_cb.wdata <= tr.data;             // 写数据放上总线
    vif.drv_cb.write <= tr.write;            // 写使能（1=写，0=读）

    @(vif.drv_cb);                           // 等第二个时钟沿
    vif.drv_cb.write <= 0;                   // 写使能拉低

    if (!tr.write) begin                     // 如果是读操作
        @(vif.drv_cb);                       // 等第三个时钟沿
        tr.data = vif.drv_cb.rdata;          // 采样读回的数据
        rd_port.write(tr);                   // 发给 scoreboard 比对
    end
endtask
```

写操作的流程：第一个时钟沿发出 addr/wdata/write=1，DUT 采样到 write=1 后把 wdata 写入 mem[addr]。第二个时钟沿 write=0 释放总线。

读操作的流程：第一个时钟沿发出 addr/write=0，DUT 采样到 write=0 后将 rdata <= mem[addr] 并拉高 rvalid（寄存器输出，一个周期后稳定）。第三个时钟沿 driver 采样 rdata，通过 `rd_port.write(tr)` 发给 scoreboard 的 `act_fifo`。

`drive_item` 的参数 `reg_transaction tr` 在 SystemVerilog 中是引用传递——传的是对象地址，不是副本。读操作分支中 `tr.data = vif.drv_cb.rdata` 把读回的数据写进 tr 对象本身，sequence 在 `finish_item` 返回后读 `tr.data` 就能拿到结果。这是 UVM 中 sequence 和 driver 之间数据回传的基础。
## 七、信号采样——Monitor

[⬆ 返回目录](#目录)

文件位置：`verif/reg_monitor.sv`

### 这一层加了什么

Monitor 继承自 `uvm_monitor`，负责被动地观察 interface 上的信号变化，恢复成 transaction 后通过分析端口发送出去。它不驱动任何信号，只采样。

### 为什么需要单独分出这一层

验证环境需要知道 DUT 的实际行为来与预期行为对比。Monitor 是 scoreboard 的数据来源之一，它为验证环境提供了一个独立于 driver 的观察视角。driver 只知道自己发送了什么，而 monitor 能观察到 DUT 实际接收到了什么。

### 完整代码

```systemverilog
class reg_monitor extends uvm_monitor;

    `uvm_component_utils(reg_monitor)

    virtual reg_if vif;
    uvm_analysis_port #(reg_transaction) mon_port;

    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction

    function void build_phase(uvm_phase phase);
        mon_port = new("mon_port", this);
        if (!uvm_config_db #(virtual reg_if)::get(this, "", "vif", vif))
            `uvm_fatal("MON", "vif 没有从 config_db 传进来")
    endfunction

    task run_phase(uvm_phase phase);
        reg_transaction tr;
        forever begin
            @(vif.mon_cb);
            if (vif.mon_cb.write) begin
                tr = reg_transaction::type_id::create("tr");
                tr.addr  = vif.mon_cb.addr;
                tr.data  = vif.mon_cb.wdata;
                tr.write = 1;
                mon_port.write(tr);
            end
        end
    endtask

endclass
```

### 概念讲解

#### 1. `uvm_monitor`

**是什么**

`uvm_monitor` 是 UVM 提供的基类，用于被动观察信号。与 `uvm_driver` 不同，它**没有内置**任何端口——没有 `seq_item_port`、没有 `req`、没有 `rsp`。所有需要的端口和功能都需要手动声明。

**语法/用法**

```systemverilog
class reg_monitor extends uvm_monitor;
    `uvm_component_utils(reg_monitor)
    // 端口全部手动声明
    uvm_analysis_port #(reg_transaction) mon_port;
    ...
endclass
```

**细节/注意的点**

- `uvm_monitor` 和 `uvm_driver` 都继承自 `uvm_component`，所以都有 `build_phase`、`run_phase`、`config_db` 等机制。
- monitor 只看不驱动，所以不需要 sequencer 端口或 TLM 握手。
- 父类 `run_phase` 是空函数，`super.run_phase` 和 `super.build_phase` 可调可不调——调了也没副作用。

#### 2. 采样策略

Monitor 只采样写操作（`vif.mon_cb.write` 为高时）。读操作由 driver 通过 `rd_port` 直接汇报给 scoreboard——driver 知道自己发起了读请求，知道读数据在哪个时钟沿稳定，直接从内部汇报比分出一根线给 monitor 更简洁。

`mon_port` 是 `uvm_analysis_port #(reg_transaction)` 类型，在 env 的 connect_phase 中连到 scoreboard 的 `exp_fifo`。`mon_port.write(tr)` 把采样到的写操作广播出去，scoreboard 的 `exp_fifo` 收到后记录到 `exp_mem`。

## 八、激励调度——Sequencer

[⬆ 返回目录](#目录)

文件位置：`verif/reg_sequencer.sv`

### 这一层加了什么

Sequencer 继承自 `uvm_sequencer #(reg_transaction)`，本质是一个带仲裁的 FIFO 队列。它接收来自 sequence 的 transaction 请求，仲裁后转发给 driver。

### 为什么需要单独分出这一层

在实际项目中，可能有多个 sequence 同时运行，都需要向 driver 发送 transaction。Sequencer 解决谁先谁后的调度问题，它内部实现了 FIFO 和多种仲裁算法。对于这个简单的寄存器验证项目，只需要一个 sequence 运行，sequencer 的作用退化为在 sequence 和 driver 之间建立连接。

### 完整代码

```systemverilog
class reg_sequencer extends uvm_sequencer #(reg_transaction);
    `uvm_component_utils(reg_sequencer)
    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction
endclass
```

### 概念讲解

#### 1. `uvm_sequencer #(T)`

**是什么**

`uvm_sequencer #(T)` 是 UVM 提供的基类，本质是一个带仲裁的 FIFO 队列。它接收来自 sequence 的 transaction 请求，仲裁后转发给 driver。所有核心功能（`get_next_item`、`item_done`、仲裁算法）都由基类实现，验证工程师只需要声明参数化类型即可。

**语法/用法**

```systemverilog
class reg_sequencer extends uvm_sequencer #(reg_transaction);
    `uvm_component_utils(reg_sequencer)
    // 不需要写任何逻辑——父类全包了
endclass
```

**细节/注意的点**

- **可以直接用基类uvm_sequencer吗？** 可以。在 agent 里直接声明 `uvm_sequencer #(reg_transaction) sequencer;` 然后 `new("sequencer", this)` 也完全能跑。
- 定义一个空壳再继承的两层意义：一是通过 `type_id::create` 获得 factory override 能力（以后可以 override 成别的类而不改 agent 代码）；二是预留扩展入口——复杂项目可能在 sequencer 里加排程策略或回调函数。
- `new(string name, uvm_component parent)` 的签名是所有 UVM 组件的固定格式。`name` 是组件在 UVM 树中的名字（打印日志和 config_db 路径时使用），`parent` 是父节点指针。这两个参数由 `type_id::create("sqr", this)` 在实例化时传入。
## 九、验证场景——Sequence

[⬆ 返回目录](#目录)

文件位置：`verif/reg_sequence.sv`

### 这一层加了什么

Sequence 描述了验证工程师关心的测试场景——生成 transaction 并通过 sequencer 发送给 driver。一个 UVM sequence 本质上是一系列 transaction 的产生和调度逻辑。

### 为什么需要单独分出这一层

将测试场景与驱动时序分离意味着修改测试内容时不需要改动 driver 或 DUT 的任何代码。每次修改都可以减少修改范围，降低引入新 bug 的风险。需要跑不同的测试（随机测试、边界测试）时，只需要写一个新的 sequence 类即可。

### 完整代码

```systemverilog
// ===== Base Sequence =====
class reg_base_sequence extends uvm_sequence #(reg_transaction);

    `uvm_object_utils(reg_base_sequence)

    function new(string name = "reg_base_sequence");
        super.new(name);
    endfunction

    task body();
        `uvm_info("SEQ", "base_sequence body 不应被直接调用", UVM_LOW)
    endtask

endclass


// ===== 写后读测试 =====
class reg_write_read_seq extends reg_base_sequence;

    `uvm_object_utils(reg_write_read_seq)

    function new(string name = "reg_write_read_seq");
        super.new(name);
    endfunction

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


// ===== 边界测试 =====
class reg_boundary_seq extends reg_base_sequence;

    `uvm_object_utils(reg_boundary_seq)

    function new(string name = "reg_boundary_seq");
        super.new(name);
    endfunction

    task body();
        reg_transaction tr;

        tr = reg_transaction::type_id::create("tr");
        start_item(tr);
        tr.addr = 8'h00; tr.data = 32'hDEAD_BEEF; tr.write = 1;
        finish_item(tr);

        tr = reg_transaction::type_id::create("tr");
        start_item(tr);
        tr.addr = 8'hFF; tr.data = 32'h12345678; tr.write = 1;
        finish_item(tr);

        tr = reg_transaction::type_id::create("tr");
        start_item(tr);
        tr.addr = 8'd100; tr.data = 32'hABCD_0000; tr.write = 1;
        finish_item(tr);
    endtask

endclass
```

### 概念讲解

#### 1. `uvm_sequence` 和 `body()`

**是什么**

`uvm_sequence` 是 UVM 的基类，用于描述测试场景——生成一系列 transaction 并通过 sequencer 发送给 driver。它**不继承 `uvm_component`**，所以不是 UVM 树中的节点结构类，而是一个数据对象（可被复制、随机化）。

`body()` 是 `uvm_sequence` 基类的**入口 task**。sequence 被 `seq.start(sequencer)` 启动后，`body()` 被自动调用。和 test 的 `run_phase`、driver 的 `run_phase` 是同一个概念——UVM 规定好的函数名，你把执行逻辑填进去。

**语法/用法**

```systemverilog
class my_seq extends uvm_sequence #(数据);
    `uvm_object_utils(my_seq)   // 注意：是 object_utils，不是 component_utils
    
    task body();
        // 生成 transaction，通过 start_item/finish_item 发送
    endtask
endclass
```

启动方式（在 test 的 run_phase 里）：

```systemverilog
seq = my_seq::type_id::create("seq");
seq.start(sequencer（句柄）);  // 把 sequence 挂到 sequencer 上执行
```

**细节/注意的点**

- sequence 不挂入 UVM 树，所以构造函数没有 `parent` 参数，`new(string name = "..." )` 创建时不需要传递 parent。
- 注册用 `uvm_object_utils` 而不是 `uvm_component_utils`，因为它是 object 不是 component——component 需要 parent 参数来挂进树，object 不需要。
- `body()` 是父类的虚 task，子类重写后父类的空版本被覆盖，所以 `super.body()` 可调可不调。
- `seq.start(sequencer)` 是一个阻塞调用——sequence 的 `body()` 跑完后它才返回。

#### 2. `start_item` / `finish_item`

**是什么**

它们是 **`uvm_sequence` 基类的内建 task**，不是哪个组件的函数。sequence 在 `body()` 里直接调用，不需要加任何前缀。

**作用：把一笔 transaction 从 sequence 发送到 driver 的完整握手。** 两步分开的原因——`start_item` 先获取发送权（告诉 sequencer "我要发了"），`finish_item` 等到 driver 实际处理完才返回（告诉 sequence "可以造下一笔了"）。两步之间留有 grab 区，允许在发出前修改或随机化 transaction。

**语法/用法**

```systemverilog
task body();
    reg_transaction tr;
    tr = reg_transaction::type_id::create("tr");
    start_item(tr);        // 申请发送权限，阻塞直到 sequencer 同意
    // 这里是 grab 区——可修改 tr 或 randomize
    tr.randomize();
    finish_item(tr);       // 正式发出，阻塞直到 driver 处理完
endtask
```

**细节/注意的点**

- `start_item(tr)` 阻塞直到 sequencer 仲裁同意。sequencer 内部维护一个队列，多个 sequence 同时请求时决定谁先发。
- `finish_item(tr)` 阻塞直到 driver 调 `item_done()`。driver 那边 `get_next_item` 拿到 transaction → 驱动 DUT → `item_done`，之后这边的 `finish_item` 才返回。
- `start_item` 和 `finish_item` 之间的 grab 区是 UVM 设计的精妙之处——transaction 已被标记为待发送但 driver 还没动它，可以在此时修改数据或执行 `randomize()`。
- 完整握手链路：sequence `start_item` → sequencer 仲裁 → `finish_item` → sequencer 转发 → driver `get_next_item` 拿到 → 驱动 DUT → `item_done` → sequence `finish_item` 返回。

#### 3. 基类 sequence

基类 `reg_base_sequence` 的 `body()` 只打了一句 warning，是一个空壳——只给别人继承用，不直接跑。

```systemverilog
class reg_base_sequence extends uvm_sequence #(reg_transaction);
    `uvm_object_utils(reg_base_sequence)
    task body();
        `uvm_info("SEQ", "base_sequence body 不应被直接调用", UVM_LOW)
    endtask
endclass
```

为什么写这个空壳？如果以后所有 sequence 都要在执行前后打印日志，改 base_sequence 的 `body()` 一次就够了。

#### 4. 写后读测试

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

#### 5. 边界测试

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
## 十、结果比对——Scoreboard

[⬆ 返回目录](#目录)

文件位置：`verif/reg_scoreboard.sv`

### 这一层加了什么

Scoreboard 是验证环境的裁判，继承自 `uvm_scoreboard`。它从两个数据路径接收信息，判断 DUT 的行为是否正确。写操作告诉它应该存什么值，读操作告诉它读回了什么值，它比较两者是否一致。

### 为什么需要单独分出这一层

验证的核心问题在于 DUT 的行为是否正确。Driver 负责发送数据，monitor 负责捕获数据，但检查数据正确性的逻辑应该独立于这两者。单独分出 scoreboard 层使得检查逻辑集中在一个地方，方便实现复杂的比较算法和统计功能。

### 完整代码

```systemverilog
class reg_scoreboard extends uvm_scoreboard;

    `uvm_component_utils(reg_scoreboard)

    reg_transaction exp_mem[int];  // 预期存储器

    uvm_tlm_analysis_fifo #(reg_transaction) exp_fifo;
    uvm_tlm_analysis_fifo #(reg_transaction) act_fifo;

    int pass_count;
    int fail_count;

    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction

    function void build_phase(uvm_phase phase);
        exp_fifo = new("exp_fifo", this);
        act_fifo = new("act_fifo", this);
        pass_count = 0;
        fail_count = 0;
    endfunction

    task run_phase(uvm_phase phase);
        fork
            // 线程 1：处理写操作，构建预期存储器
            forever begin
                reg_transaction exp_tr;
                exp_fifo.get(exp_tr);
                exp_mem[exp_tr.addr] = exp_tr;
                `uvm_info("SCB", $sformatf("写 addr=0x%0h data=0x%0h", exp_tr.addr, exp_tr.data), UVM_LOW)
            end
            // 线程 2：处理读结果，比对
            forever begin
                reg_transaction act_tr;
                act_fifo.get(act_tr);
                if (exp_mem.exists(act_tr.addr)) begin
                    if (act_tr.data == exp_mem[act_tr.addr].data) begin
                        pass_count++;
                        `uvm_info("SCB", $sformatf("PASS addr=0x%0h wdata=0x%0h rdata=0x%0h",
                            act_tr.addr, exp_mem[act_tr.addr].data, act_tr.data), UVM_LOW)
                    end else begin
                        fail_count++;
                        `uvm_error("SCB", $sformatf("FAIL addr=0x%0h expect=0x%0h got=0x%0h",
                            act_tr.addr, exp_mem[act_tr.addr].data, act_tr.data))
                    end
                end else begin
                    `uvm_error("SCB", $sformatf("addr=0x%0h 未找到写记录", act_tr.addr))
                end
            end
        join_none
    endtask

    function void report_phase(uvm_phase phase);
        `uvm_info("SCB", $sformatf("PASS=%0d  FAIL=%0d", pass_count, fail_count), UVM_LOW)
    endfunction

endclass
```

### 概念讲解

#### 1. `uvm_scoreboard`

**是什么**

`uvm_scoreboard` 是 UVM 提供的基类，继承自 `uvm_component`，专门用于数据比对。它与 `uvm_env`、`uvm_agent` 同属结构类，有 `build_phase`、`run_phase`、`report_phase` 等所有 phase。

**语法/用法**

```systemverilog
class reg_scoreboard extends uvm_scoreboard;
    `uvm_component_utils(reg_scoreboard)
    // 有 build_phase、run_phase、report_phase
endclass
```

**细节/注意的点**

- 对比 `uvm_env`、`uvm_agent`：它们都继承 `uvm_component`，都有相同 phase 机制。区别在于**语义上的分工**——agent 打包一组接口组件，env 组合多个 agent 和 scoreboard，scoreboard 专门做数据比对。UVM 通过继承不同的类名让代码结构更清晰。

#### 2. `uvm_tlm_analysis_fifo`

**是什么**

全称 UVM Transaction-Level Modeling Analysis FIFO。它是一个**自带缓存的 analysis port receiver**——一头接收广播，一头阻塞读取。

**语法/用法**

```systemverilog
// 声明
uvm_tlm_analysis_fifo #(reg_transaction) exp_fifo;

// build_phase 中创建
exp_fifo = new("exp_fifo", this);

// driver/monitor 那头通过 analysis_port 发过来
// 这头通过 .get(tr) 阻塞读取
exp_fifo.get(exp_tr);
```

**细节/注意的点**

- FIFO 有两头：`analysis_export` 负责接收 `write(tr)` 广播，`get(tr)` 负责阻塞读出。当 FIFO 为空时 `get(tr)` 会等待直到有数据到达。
- 和 `uvm_analysis_port` 的区别：port 只是**出口**（只往外发），fifo 是"出口 + 缓存 + 入口"（一头收一头读）。
- 两条数据流：
  - **写路径**：monitor `mon_port.write(tr)` → `exp_fifo` 排队缓存 → scoreboard `exp_fifo.get(exp_tr)` 取出 → 存入 `exp_mem[addr]`
  - **读路径**：driver `rd_port.write(tr)` → `act_fifo` 排队缓存 → scoreboard `act_fifo.get(act_tr)` 取出 → 与 `exp_mem[addr]` 比对
- 两个 FIFO 分开排队互不阻塞——`exp_fifo` 收写记录、`act_fifo` 收读结果，后台两个 forever 线程各自跑各自的，互不卡住。

#### 3. `reg_transaction exp_mem[int]` ——关联数组

**是什么**

`exp_mem[int]` 声明了一个**关联数组**（associative array），类似 Python 的字典。键是 `int` 类型（地址），值是 `reg_transaction` 类型。

**语法/用法**

```systemverilog
reg_transaction exp_mem[int];           // 声明：键 int，值 reg_transaction
exp_mem[addr] = tr;                     // 存入
if (exp_mem.exists(addr))               // 检查键是否存在
    data == exp_mem[addr].data;         // 取值
```

**细节/注意的点**

- 与普通数组 `logic [31:0] mem [0:255]` 的区别：普通数组要求下标 0~255 连续分配好空间；关联数组用到哪个下标才分配哪个，不浪费内存。
- 这里用关联数组的原因是：DUT 有 256 个寄存器，但测试只访问其中少数几个（0~9、100~109、200），关联数组自然只给被访问的地址分配存储空间。
- `exp_mem[addr] = tr` 的含义是：用 `addr` 字段的值做**钥匙（键）**，把整个 `tr` 对象存进去。取出时也必须用同一个字段做钥匙——`exp_mem[act_tr.addr]` 才能取到刚才存的那个对象。理论上可以用其他字段做键，比如 `exp_mem[data] = tr`，但 `data` 值可能重复（不同地址写相同数据），后面的覆盖前面的导致丢数据。`addr` 在寄存器地址空间中天然唯一，所以最适合做键。核心原则：**存和取的键必须是同一个字段**。

#### 4. `fork...join_none`

**是什么**

SystemVerilog 的并发语句，用于在仿真中**启动多个并行线程**。三种变体的区别在于等待方式：

| 变体                 | 行为                 |
| ------------------ | ------------------ |
| `fork...join`      | 等所有子线程跑完才继续        |
| `fork...join_any`  | 等任意一个子线程跑完就继续      |
| `fork...join_none` | 启动子线程后**立即继续**，不等待 |


**语法/用法**

```systemverilog
fork
    forever begin exp_fifo.get(exp_tr); exp_mem[exp_tr.addr] = exp_tr; end
    forever begin act_fifo.get(act_tr); /* 比对处理 */ end
join_none
// 执行到这里时两个 forever 线程已经在后台跑了，run_phase 可以返回
```

**细节/注意的点**

- 这里必须用 `join_none`——如果用 `fork...join`，两个 `forever` 永远不会结束，`run_phase` 永远卡在 `join` 那里不返回，仿真不结束。
- `join_none` 让 run_phase 返回后，两个线程在后台独立运行。仿真的结束由 test 的 `drop_objection` 控制。
- `fork...join_any` 在这里也用不了——两个线程都是 `forever`，永远不会有"任意一个跑完"的情况。

#### 5. `report_phase`

**是什么**

`report_phase` 是 UVM 的最后一个 phase，在所有其他 phase（包括 run_phase）结束后自动执行，专门用于输出统计报告。

**语法/用法**

```systemverilog
function void report_phase(uvm_phase phase);
    `uvm_info("SCB", $sformatf("PASS=%0d  FAIL=%0d", pass_count, fail_count), UVM_LOW)
endfunction
```

**细节/注意的点**

- `report_phase` **不是必须加的**——不写它仿真也能跑完，只是没有统计输出。
- 父类 `uvm_component` 的 `report_phase` 是空函数，子类重写它是完全覆盖。`super.report_phase(phase)` 可调可不调——调了也没效果。
- 为什么需要这个 phase？你当然可以在 `run_phase` 结束时打印统计信息，但 `report_phase` 保证了在所有组件都完成 `run_phase` 后才统一输出报告，不会出现 "一个组件还在跑、另一个已经开始打印统计" 的情况。

#### 6. 比对逻辑

从 `exp_fifo` 收到写操作（收到tr）时，将 `addr` 和 `data` 存入 `exp_mem`。从 `act_fifo` 收到读操作时，如果 `exp_mem` 中已有该地址的记录，比较实际数据和预期数据。匹配则 `pass_count` 增加，不匹配则 `fail_count` 增加并报告 `uvm_error`。如果该地址尚未被写入过，发出 `uvm_error` 提示未找到写记录。

## 十一、组件打包——Agent

[⬆ 返回目录](#目录)

文件位置：`verif/reg_agent.sv`

### 这一层加了什么

Agent 将 driver、monitor 和 sequencer 这三个与同一套信号相关的组件打包在一起。它对上层的 env 隐藏了内部组件的创建和连接细节。

### 为什么需要单独分出这一层

在复杂的芯片验证项目中，同一个总线协议（如 AXI、APB）可能在多个模块中使用。将 driver、monitor、sequencer 打包成 agent 后，env 可以直接实例化这个 agent 多次，而不需要为每个模块重复创建和连接这些组件。

Agent 有两种工作模式。`UVM_ACTIVE` 模式包含 driver 和 sequencer（可发送激励），`UVM_PASSIVE` 模式只包含 monitor（仅观察）。对于只需要监测信号但不驱动信号的场景（比如验证环境中的从机接口），passive 模式非常有用。

### 完整代码

```systemverilog
class reg_agent extends uvm_agent;

    `uvm_component_utils(reg_agent)

    reg_driver     driver;
    reg_monitor    monitor;
    reg_sequencer  sequencer; //声明句柄

    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction

    function void build_phase(uvm_phase phase);
        driver    = reg_driver::type_id::create("driver", this);
        monitor   = reg_monitor::type_id::create("monitor", this);
        sequencer = reg_sequencer::type_id::create("sequencer", this);
    endfunction

    function void connect_phase(uvm_phase phase);
        driver.seq_item_port.connect(sequencer.seq_item_export);
    endfunction

endclass
```

### 概念讲解

#### 1. `uvm_agent`

**是什么**

`uvm_agent` 是 UVM 提供的基类，用于将 driver、monitor、sequencer 三个与同一套信号相关的组件打包在一起。它让上层的 env 可以通过一个 agent 句柄操作一组完整的接口验证逻辑。

`uvm_agent` 有两种工作模式：

| 模式 | 包含组件 | 用途 |
|---|---|---|
| `UVM_ACTIVE` | driver + monitor + sequencer | 主动发送激励和采样 |
| `UVM_PASSIVE` | 仅 monitor | 只观察不驱动（如从机接口） |

本项目使用 `UVM_ACTIVE`（默认值），包含 driver、monitor、sequencer 全部三个组件。

**语法/用法**

```systemverilog
class reg_agent extends uvm_agent;
    `uvm_component_utils(reg_agent)

    reg_driver     driver;
    reg_monitor    monitor;
    reg_sequencer  sequencer;

    function void build_phase(uvm_phase phase);
        driver    = reg_driver::type_id::create("driver", this);
        monitor   = reg_monitor::type_id::create("monitor", this);
        sequencer = reg_sequencer::type_id::create("sequencer", this);
    endfunction

    function void connect_phase(uvm_phase phase);
        driver.seq_item_port.connect(sequencer.seq_item_export);
    endfunction
endclass
```

**细节/注意的点**

- agent 只负责组件创建和内部连线，不包含业务逻辑。driver/monitor/sequencer 各司其职，agent 是组装工。
- 连接 driver 和 sequencer 的是 `driver.seq_item_port.connect(sequencer.seq_item_export)`。这行代码把 driver 的 TLM 端口和 sequencer 的出口对接，让 sequence 的 transaction 能流到 driver。
- agent 只打包自己内部组件。monitor 的 `mon_port`和 driver 的 `rd_port`需要和scoreboard交互，这部分的连线agent不管，由env负责。
## 十二、环境组装——Env

[⬆ 返回目录](#目录)

文件位置：`verif/reg_env.sv`

### 这一层加了什么

Env 是整个验证环境的顶层容器，它实例化 agent 和 scoreboard，并将它们连接起来形成完整的数据通路。

### 为什么需要单独分出这一层

一个验证环境通常包含多个 agent（对应多个接口）和一个 scoreboard。Env 层将这些组件装配成一个整体，test 只需要实例化 env 即可获得完整的验证环境，不需要关心组件间的连接细节。

### 完整代码

```systemverilog
class reg_env extends uvm_env;

    `uvm_component_utils(reg_env)

    reg_agent      agent;
    reg_scoreboard scoreboard;

    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction

    function void build_phase(uvm_phase phase);
        agent      = reg_agent::type_id::create("agent", this);
        scoreboard = reg_scoreboard::type_id::create("scoreboard", this);
    endfunction

    function void connect_phase(uvm_phase phase);
        // monitor → scoreboard（写操作）
        agent.monitor.mon_port.connect(scoreboard.exp_fifo.analysis_export);
        // driver → scoreboard（读结果）
        agent.driver.rd_port.connect(scoreboard.act_fifo.analysis_export);
    endfunction

endclass
```

### 概念讲解

#### 1. `uvm_env`

**是什么**

`uvm_env` 是 UVM 提供的最顶层的容器基类，用于装配整个验证环境——实例化 agent 和 scoreboard，并将它们之间的数据通路连接起来。

**语法/用法**

```systemverilog
class reg_env extends uvm_env;
    `uvm_component_utils(reg_env)

    reg_agent      agent;
    reg_scoreboard scoreboard;

    function void build_phase(uvm_phase phase);
        agent      = reg_agent::type_id::create("agent", this);
        scoreboard = reg_scoreboard::type_id::create("scoreboard", this);
    endfunction

    function void connect_phase(uvm_phase phase);
        agent.monitor.mon_port.connect(scoreboard.exp_fifo.analysis_export);
        agent.driver.rd_port.connect(scoreboard.act_fifo.analysis_export);
    endfunction
endclass
```

**细节/注意的点**

- env 的 connect_phase 做的是**跨组件连线**——把 agent 内部的端口和 scoreboard 的 FIFO 接起来。agent 内部 driver 和 sequencer 的连线在 agent 自己的 connect_phase 中完成。
- 端口连线的写法是 `源端口.connect(目标端口)`。所以数据由 `mon_port`(monitor的`uvm_analysis_port`)流向scoreboard的`analysis_export`  。connect 之后，调 `mon_port.write(tr)` 时，数据自动流进 `exp_fifo`。

#### 2. 数据路径

写完 connect 后，两条数据路径形成闭环：

- **写操作**：monitor 采样 → `mon_port.write(tr)` → `exp_fifo.analysis_export` → `exp_fifo` 缓存 → scoreboard 读取 → 存入 `exp_mem`
- **读操作**：driver 读回数据 → `rd_port.write(tr)` → `act_fifo.analysis_export` → `act_fifo` 缓存 → scoreboard 读取 → 与 `exp_mem` 比对
## 十三、仿真顶层——Test Top

[⬆ 返回目录](#目录)

文件位置：`tb/test_top.sv`

### 这一层加了什么

Test top 是一个 Verilog module（非 UVM 组件），是整个仿真的入口。它生成时钟和复位信号，实例化 DUT 和 interface，将 interface 通过 `uvm_config_db` 传递给 UVM 环境（uvm环境需要和DUT交互的地方，如driver和monitor就依靠config_db取出interface），最后调用 `run_test()` 启动 UVM。

### 为什么需要单独分出这一层

UVM 是一个基于 SystemVerilog 的类库，它无法生成硬件信号（时钟、复位）或实例化 DUT。这些硬件级的操作必须在 module 中完成。Test top 是连接硬件世界和 UVM 软件世界的桥梁。

### 完整代码

```systemverilog
module test_top;

    reg clk;
    reg rst_n;

    initial clk = 0;
    always #5 clk = ~clk;

    initial begin
        rst_n = 0;
        #20 rst_n = 1;
    end

    reg_if vif (clk);

    reg_module dut (
        .clk   (clk),
        .rst_n (rst_n),
        .addr  (vif.addr),
        .wdata (vif.wdata),
        .write (vif.write),
        .rdata (vif.rdata),
        .rvalid(vif.rvalid)
    );

    initial begin
        uvm_config_db #(virtual reg_if)::set(null, "uvm_test_top", "vif", vif);
        run_test();
    end

    // 仿真时长限制
    initial begin
        #1ms;
        $display("TIMEOUT: 仿真超时 1ms");
        $finish;
    end

endmodule
```

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

### 概念讲解

#### 1. `module` 和 UVM class 的区别

**是什么**

Test top 是一个 `module`（硬件模块），不是 UVM class。module 能做的事和 class 不同——module 可以直接例化硬件（DUT、interface）、生成时钟和复位信号。UVM class 运行在软件层，不能直接操作硬件引脚。

**语法/用法**

```systemverilog
module test_top;                   // 硬件模块
    reg clk;                       // 硬件寄存器
    always #5 clk = ~clk;          // 硬件时钟生成（module 可以，class 不行）
    reg_if vif (clk);              // 硬件例化（module 可以，class 不行）
    reg_module dut (...);          // 硬件例化
    initial begin
        uvm_config_db #(virtual reg_if)::set(null, "uvm_test_top", "vif", vif);
        run_test();                // 从 module 启动 UVM
    end
endmodule
```

**细节/注意的点**

- `module` 里 `initial` 块中的代码在时间 0 执行，`run_test()` 在其中被调用，之后 UVM phase 自动启动。
- module 没有 `this`，所以在 config_db 中使用 `null` 表示"从根开始"。

#### 2. `run_test()`

**是什么**

`run_test()` 是 UVM 的入口函数，在 `uvm_root` 中定义。它创建 test 实例（如 `reg_base_test`）、构建 UVM 树、启动所有 phase 调度。去哪里找 test 类名？通过命令行 `+UVM_TESTNAME=reg_base_test` 指定。

**语法/用法**

```systemverilog
initial begin
    run_test();    // 从 +UVM_TESTNAME 命令行参数读 test 名
end
```

**细节/注意的点**

- `run_test()` 内部执行：创建 test 实例（实例名固定为 `uvm_test_top`）→ 调用 `build_phase`（从上到下）→ `connect_phase` → `run_phase` → `report_phase` 等。
- `uvm_test_top` 这个名字不是用户自取的，是 `run_test()` 内部固定的 test 实例名。所以 config_db 中写 `"uvm_test_top"`。
- 如果命令行没有 `+UVM_TESTNAME`，UVM 会报错并列出所有可用的 test 类。

## 十四、测试用例——Test

[⬆ 返回目录](#目录)

文件位置：`verif/reg_base_test.sv`

### 这一层加了什么

Test 是 UVM 验证层次的顶层组件，继承自 `uvm_test`。它负责创建 env、控制仿真的开始和结束、以及启动 sequence。

### 为什么需要单独分出这一层

不同的测试场景（写后读测试、边界测试、随机测试等）共享同一个 env 和 agent，但使用不同的 sequence 和配置。通过继承一个基类 test 并在子类中更换 sequence，可以在不修改 env 的情况下运行不同的测试。

### 完整代码

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
    endtask

endclass
```

### 概念讲解

#### 1. `uvm_test`

**是什么**

`uvm_test` 是 UVM 提供的顶层组件基类，继承自 `uvm_component`。它是 UVM 树的根（`run_test()` 创建的实例固定叫 `uvm_test_top`），负责创建 env、控制仿真起止、启动 sequence。

**语法/用法**

```systemverilog
class reg_base_test extends uvm_test;
    `uvm_component_utils(reg_base_test)

    reg_env env;

    function void build_phase(uvm_phase phase);
        env = reg_env::type_id::create("env", this);  // 创建 env
    endfunction

    function void connect_phase(uvm_phase phase);
        // config_db 第二跳：取 vif 并分发给子组件
    endfunction

    task run_phase(uvm_phase phase);
        phase.raise_objection(this);
        // 启动 sequence，做测试
        phase.drop_objection(this);
    endtask
endclass
```

**细节/注意的点**

- `uvm_test` 是所有其他组件的间接父节点。它创建 env，env 创建 agent，agent 创建 driver/monitor/sequencer——形成完整的 UVM 树。
- 不同测试场景（写后读、边界测试）通过继承 `reg_base_test` 并更换 `run_phase` 中的 sequence 实现，不需要重新搭建整个环境。

#### 2. `raise_objection` / `drop_objection`

**是什么**

UVM 的 objection 机制控制仿真何时结束。这是一个计数锁——`raise_objection` 增加计数（「我还有活」），`drop_objection` 减少计数（「我干完了」）。当所有 objection 都 drop 后，UVM 认为没有组件在忙，结束 run_phase。

如果没有 objection，run_phase 一进来就立即退出，仿真刚启动就结束，什么都不会发生。

**语法/用法**

```systemverilog
task run_phase(uvm_phase phase);
    phase.raise_objection(this);       // 加锁：我还有活
    // 执行测试逻辑（启动 sequence、等待驱动、采样、比对）
    #100;                              // 等 100ns 让最后的比对完成
    phase.drop_objection(this);        // 解锁：我干完了
endtask
```

**细节/注意的点**

- 这个项目只有一个 objection 源——test 的 `run_phase`。Test 跑完 sequence 后等 100ns，然后 drop objection，仿真结束。Scoreboard 的 `fork...join_none` 后台线程不持有 objection，所以它们不会阻止仿真结束。
- `raise_objection` 的参数 `this` 表示谁提出的 objection，多 objection 源时用于调试追踪。这个项目只有一个源所以不需要纠结这个。

## 十五、编译脚本——Makefile

[⬆ 返回目录](#目录)

### 什么是 Makefile

Makefile 是一个文本文件，里面写好了**编译仿真需要的全部命令**。你只需要在终端输入 `make 目标名字`，它就会自动执行对应的命令，不用每次都手动敲一长串。

比如你输入 `make sim`，它就自动帮你执行"编译 → 运行"两个步骤。你不需要记住 `vcs -full64 ...` 那一堆参数。

### 怎么用

在桌面的 `reg_test/` 目录下，打开终端（MobaXterm 或 Git Bash）：

```bash
make sim                   # 默认跑一次完整的写后读测试
make sim TEST=reg_boundary_test   # 跑边界测试
make clean                 # 删除编译产生的临时文件
```

每次都先 `make clean` → `make sim` 也可以，但建议直接 `make sim`——它会自动检查哪些文件没变、跳过那些文件的重新编译，比 clean 后从头编译快很多。

建议在 reg_test 目录下打开 MobaXterm 终端 输入 `make sim` 就可以了，不需要理解下面那些命令的细节。

### make 是什么

`make` 是一个自动化工具：你告诉它"我想做什么"，它就去 Makefile 里找到对应的命令来执行。

- `make sim` → 执行 Makefile 里 `sim:` 下面的命令（编译 + 运行）
- `make clean` → 执行 `clean:` 下面的命令（删临时文件）

### 各个命令在干啥（了解一下就行）

你不需要记住这些，但如果你好奇 make sim 背后到底跑了什么：

```makefile
compile:                     # 编译阶段
	vcs 编译参数 源文件列表 -o simv_测试名

run:                         # 运行阶段
	./simv_测试名 +UVM_TESTNAME=测试名

sim: compile run             # sim = 先编译再运行

clean:                       # 清理
	rm -rf 编译产物
```

**`vcs`** — VCS 编译器的命令。`vcs .sv文件` → 把这些 SystemVerilog 代码编译成一个可执行文件（仿真程序）。

**`+incdir+目录`** — 告诉 VCS "去这个目录找被 include 的文件"。

**`+define+UVM_NO_DPI`** — 加一个编译宏，告诉 UVM 不要用 DPI 功能，因为老版本 VCS 的 DPI 链接有问题。

**`./simv_xxx`** — 编译出来的可执行文件的名字。`./xxx` 表示执行当前目录下的 xxx 文件。

**`+UVM_TESTNAME=xxx`** — UVM 运行时参数，告诉 UVM "这次跑 xxx 这个测试用例"。

**`-o simv_$(TEST)`** — 编译输出的文件名。`$(TEST)` 会被替换成你指定的测试名，比如 `simv_reg_write_read_test`。

**`-l run.log`** — 把仿真输出的所有信息写到 `run.log` 文件里，方便回头查看。

## 十六、仿真运行与结果解读

[⬆ 返回目录](#目录)

### 你会在终端看到什么

运行 `make sim` 后，终端会输出大量信息。你需要关注的主要分四段：

**第一段：编译信息**
```
*** VCS* Compiler ...
```
一大串 VCS 的编译输出，不用看。只要没报 `Error`，就说明编译通过了。

**第二段：UVM 组件树**
```
uvm_test_top               reg_write_read_test
  env                      reg_env
    agent                  reg_agent
      drv                  reg_driver
      mon                  reg_monitor
      sqr                  reg_sequencer
    scb                    reg_scoreboard
```
UVM 在启动时打印的组件层次结构，用来检查环境是否正确搭起来了。

**第三段：执行日志（每笔 transaction 的过程）**
```
@ 15000: [MON] 采样到写 addr=0x95 data=0xeeadce97
@ 15000: [DRV] 写 addr=0x95 data=0xeeadce97
@ 35000: [DRV] 读 addr=0x95 -> data=0xeeadce97
@ 35000: [SCB] 比对通过 addr=0x95 data=0xeeadce97
```
读法：时间 → 谁 → 干了什么。`@ 15000` 是仿真时间（ns），`[DRV]` 是哪个组件。重点看最后一行 `比对通过` 就是对的，`比对失败` 就有问题。

**第四段：最后的结果汇总**
```
UVM_ERROR :     0        ← 错误数量，必须为 0
UVM_FATAL :     0        ← 致命错误，必须为 0
[SCB] PASS=5 FAIL=0      ← scoreboard 自己的统计
```

### 测试结果怎么看

**正常情况：**

```
写 addr=0x95 data=0xeeadce97 → 读 data=0xeeadce97 ✓
...
SCB: 通过 5 / 失败 0
UVM_ERROR: 0, UVM_FATAL: 0
```

每一行都打 ✓，SCB 和 UVM 的 error 都是 0——验证通过。

**出问题的情况：**

```
写 addr=0x95 data=0xeeadce97 → 读 data=0x00000000 ✗     ← 读回全 0，没写进去
SCB: 通过 4 / 失败 1                                       ← 有一笔比对失败
UVM_ERROR :     1                                          ← UVM 也记了一次错误
```

出现 `✗` 或 `UVM_ERROR` 不是 0，就说明有 bug。需要 debug（看具体是 DUT 的问题还是验证环境的问题）。
## 十七、数据流总览

[⬆ 返回目录](#目录)

整个验证环境的数据流可以概括为两条路径。

整个验证环境执行一次"写后读"的数据流可以逐拍展开如下。先明确 DUT 的行为：clk 上升沿采样到 `write=1` 则写 `mem[addr]`，采样到 `write=0` 则组合逻辑 `rdata = mem[addr]`，同时 rvalid 拉高一个周期。

**写路径（从 sequence 造 transaction 到 scoreboard 记录写操作）：**

写路径串起 5 个角色：Sequence → Sequencer → Driver → DUT → Monitor → Scoreboard。

1. **Sequence** 调 `start_item(tr)` 向 sequencer 申请发送权，拿到许可后 `tr.randomize()` 随机出 addr、data，并设 `write=1`，最后调 `finish_item(tr)` 把 transaction 发给 sequencer。
2. **Sequencer** 把 transaction 转发给 driver。
3. **Driver** 的 `get_next_item(req)` 阻塞等待，从 sequencer 收到 transaction 后继续。把 `tr.addr`、`tr.wdata`、`tr.write=1` 驱动到 interface 的信号上。
4. **DUT** 在下一个 clk 上升沿检测到 `write=1`，将 `wdata` 写入 `mem[addr]`。
5. **Monitor** 在同一个时钟沿采样到 `write=1`，抓取 addr 和 wdata，打包成 `reg_transaction`，通过 `mon_port.write(tr)` 发出。
6. **Scoreboard** 的 `exp_fifo` 收到 monitor 发来的 transaction。`exp_fifo.get(exp_tr)` 取出后存入 `exp_mem[exp_tr.addr] = exp_tr`——以地址为键保存写操作的预期值。
7. **Driver** 调 `item_done()` 通知 sequencer 这笔写处理完了。Sequencer 释放管道，允许下一笔 transaction 通行。

**读路径（从 sequence 造读 transaction 到 scoreboard 比对结果）：**

1. **Sequence** 创建第二笔 transaction——addr 与写操作相同（读同一个地址），`write=0` 表示读。调 `start_item/finish_item` 发出。
2. **Sequencer** 转发给 driver。
3. **Driver** `get_next_item(req)` 收到读 transaction。把 addr 和 `write=0` 驱动到 interface 上。
4. **DUT** 在 clk 上升沿检测到 `write=0`，组合逻辑输出 `rdata = mem[addr]`，同时拉高 `rvalid` 一个时钟周期。
5. **Driver** 在下一个 clk 上升沿看到 `rvalid=1`，采样 `rdata`，写回 `req.data`。然后通过 `rd_port.write(tr)` 发给 scoreboard。这一步是 driver 自己采读结果并汇报——不需要专门的读 monitor。
6. **Scoreboard** 的 `act_fifo` 收到读结果。从 `exp_mem[act_tr.addr].data` 取出之前写路径存好的期望值，和 `act_tr.data` 比对。匹配则 `pass_count` 加一，不匹配则 `fail_count` 加一并打印 `uvm_error`。
7. **Driver** 调 `item_done()` 释放管道。

**关键设计点：**

- **Monitor 只采样写操作**。读操作由 driver 自己采 rdata 并通过 `rd_port` 汇报。不需要再写一个读 monitor 去盯着 interface 上的 rdata/rvalid——driver 知道自己什么时候发了读请求，自然知道什么时候去采 rdata。
- **Driver 把 rdata 写回 req.data 再发出去**。读 transaction 进来时 `req.data` 是 sequence 随机出来的无关值（因为只读了 data 字段、没有写）。driver 驱动 `write=0` 后，等 DUT 返回 `rdata`，把真正的读回值写进 `req.data`，然后通过 `rd_port` 发出去。这样 scoreboard 收到的 `act_tr.data` 就是真正的读回值。
- **Scoreboard 用地址把写和读关联起来**。写路径存 `exp_mem[addr] = tr`（键是地址，值是那笔写操作），读路径从 `exp_mem[addr].data` 取期望值比对。同一个地址，先写后读，数据对得上就 pass。
- **写和读是两条独立线程**。scoreboard 里用 `fork...join_none` 同时跑了 `exp_fifo.get` 和 `act_fifo.get` 两个 forever 循环，分别处理写记录和读结果，互不阻塞。

