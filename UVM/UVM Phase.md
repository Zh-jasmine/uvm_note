---
tags:
  - UVM
  - 验证
  - phase
date: 2026-05-02
---

# UVM Phase 详解笔记

> [[#前置知识 Component 的声明语法|前置知识：Component 语法]] · [[#前置知识 super.build_phase(phase) 的作用|super.build_phase]] · [[#各 phase 调用 super 的规则|super 规则]] · [[#为什么需要 Phase 机制|为什么需要 Phase]] · [[#Phase 分类 function phase 与 task phase|function vs task]] · [[#Phase 执行顺序与树结构|执行顺序]] · [[#build_phase — 创建子组件|build_phase]] · [[#connect_phase — 建立组件间连接|connect_phase]] · [[#run_phase — 执行核心逻辑|run_phase]] · [[#objection — 控制 run_phase 何时结束|objection]] · [[#report_phase — 输出测试结果|report_phase]] · [[#extract_phase、check_phase、final_phase|extract/check/final]] · [[#UVM 1.1 的 12 个 task phase|12 task phase]] · [[#Phase 执行全过程汇总|全过程汇总]]

---

## 前置知识：Component 的声明语法

每个 UVM phase 方法都写在 `uvm_component` 及其派生类中，因此在看 phase 之前先理解一个 component 类的基本结构。

```systemverilog
class my_driver extends uvm_driver #(my_transaction);
    `uvm_component_utils(my_driver)

    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction
endclass
```

三部分逐一说明。

**`extends uvm_driver #(my_transaction)`** 表示类继承自参数化的 `uvm_driver`。`#(my_transaction)` 告诉框架这个 driver 处理哪种 transaction。`uvm_driver` 是 `uvm_component` 的派生类，因此 `my_driver` 自动获得 phase 生命周期。

**`` `uvm_component_utils(my_driver) ``** 是 factory 注册宏。它将此类注册到 UVM factory 表中，使其可参与 override 和 create 机制，同时在类内生成 `type_id` 等辅助结构。

**`new` 函数**接受两个参数：第一个是实例名（树路径名），第二个是父组件指针（`this`）。`super.new(name, parent)` 将当前组件挂接到 UVM 组件树中。省略 parent 参数则组件无法参与 phase 调度。

↩ [[#UVM-Phase-详解笔记|回到目录]]

---

## 前置知识：`super.build_phase(phase)` 的作用

```systemverilog
function void build_phase(uvm_phase phase);
    super.build_phase(phase);    // 调父类的 build_phase
    drv = my_driver::type_id::create("drv", this);
endfunction
```

**为什么要调 `super.build_phase(phase)`？**

因为 `uvm_component` 的 build_phase 里封装了框架级的初始化逻辑（如处理 factory override 信息、初始化报告机制等）。如果在子类里 override 了 build_phase 但没有调 `super`，这部分框架逻辑就会丢失。

```systemverilog
// 继承链：my_driver → uvm_driver → uvm_component
// super.build_phase(phase) 从 uvm_driver 开始往上找，
// 如果 uvm_driver 没有 build_phase，就跳到 uvm_component 执行。
```

中间父类（如 `uvm_driver`、`uvm_agent`）通常没有自己的 build_phase，直接跳到 `uvm_component`。但固定写法依然是每一层都写 `super.build_phase(phase)`，因为你不确定某个中间父类未来是否会有自己的初始化逻辑。有则执行，无则跳过。

**`super` 是 SystemVerilog 关键字，代表当前类的直接父类。** `super.build_phase(phase)` 就是去父类里调用它的 build_phase。这和组件树上的父组件（env 是 agent 的父组件）完全是两回事——`super` 管的是继承关系，不是嵌套关系。

**`uvm_phase phase` 参数是 UVM 框架在调用 build_phase 时自动传入的。** 它指向当前 phase 的控制对象。在 function phase（build、connect、report 等）中基本用不到这个参数，但 override 要求签名一致，所以必须写。只有 run_phase（task phase）中需要通过它调用 `raise_objection` 和 `drop_objection`。

↩ [[#UVM-Phase-详解笔记|回到目录]]

## 各 phase 调用 super 的规则

只有 `build_phase` **必须**写 `super.build_phase(phase)`，因为父类 `uvm_component` 的 build_phase 内部处理 factory override 的自动配置。不调 super 可能导致某些 override 不生效。

其他所有 phase 的父类实现都是空的，调了也什么都不做。**标准做法是有事就写该 phase，没事就不写**，没必要为了"规范"写空壳调空函数。

| phase | 必须调 super？ | 原因 |
|---|---|---|
| `build_phase` | **必须** | 父类负责 factory override 自动配置 |
| `connect_phase` | 不用 | 父类为空 |
| `run_phase` | 不用 | 父类为空 |
| `report_phase` | 不用 | 父类为空 |
| `extract_phase` | 不用 | 父类为空 |
| `check_phase` | 不用 | 父类为空 |
| `final_phase` | 不用 | 父类为空 |

**常见误区：** 有些初学者在每个 phase 都写调 super 的空壳，认为这是"规范写法"。实际上多做了一件事——每个 phase 函数在仿真期间都会被调用一次，空函数加空 super 只是白跑一圈，没有意义，还多了需要维护的代码行数。**不写不等于不做，父类会帮你做。**

↩ [[#UVM-Phase-详解笔记|回到目录]]

---

## 为什么需要 Phase 机制

一个验证环境包含多个组件，组件之间有严格的先后依赖关系。

- test 必须在 env 之前创建，因为 env 是 test 的孩子。
- agent 必须在 driver 之前创建，因为 driver 是 agent 的孩子。
- 所有组件创建完成后，才能进行跨组件连接。
- 连接完成后，才能启动 sequence 发送数据。

Phase 机制将组件的生命周期划分为若干固定阶段，框架按严格顺序调度所有组件执行对应阶段。验证工程师只需要在对应的 phase 方法中填充在该阶段应该做的事。

↩ [[#UVM-Phase-详解笔记|回到目录]]

---

## Phase 分类：function phase 与 task phase

根据是否消耗仿真时间，phase 分为两类。

**function phase** 不消耗仿真时间，在一瞬间完成。build、connect、report、extract、check、final 都属于 function phase。它们内部不能包含耗时操作（`#delay`、`@(posedge clk)`、`wait()` 等）。

**task phase** 可以消耗仿真时间，内部可以包含延时、等待事件等阻塞操作。run_phase 是唯一需要重点关注的 task phase。

↩ [[#UVM-Phase-详解笔记|回到目录]]

---

## Phase 执行顺序与树结构

UVM 组件以树形结构组织，phase 的执行顺序与树结构密切相关。

**自顶向下执行的 phase（build_phase）：** 根节点 test 先执行，逐层向下到叶子组件。父组件必须先完成自身的构建，才能为子组件分配树路径。

**自底向上执行的 phase（connect_phase、report_phase、extract_phase、check_phase、final_phase）：** 叶子组件先执行，逐层向上到根节点。connect_phase 自底向上的原因是子组件完成内部连接后，父级才能使用子组件暴露的端口进行更上层的连接。

**全体并行执行的 phase（run_phase）：** 所有组件的 run_phase 在同一时刻启动，通过 objection 机制协调结束时间。

↩ [[#UVM-Phase-详解笔记|回到目录]]

---

## 主要 Phase 详解

### build_phase — 创建子组件

自顶向下、function phase。每个组件在此阶段通过 factory 创建其直接子组件。

```systemverilog
function void my_env::build_phase(uvm_phase phase);
    super.build_phase(phase);
    agent = my_agent::type_id::create("agent", this);
    scb   = my_scoreboard::type_id::create("scb",   this);
endfunction
```

**语法细节：**

`build_phase(uvm_phase phase)` 的参数 `uvm_phase` 是 UVM 库中定义的类，`phase` 是参数名。UVM 框架创建一个 `uvm_phase` 对象，调用 `build_phase` 时传入这个对象。参数名可以随便改（如 `uvm_phase b`），因为参数名只在本函数内部生效，框架只看类型和位置。行业习惯统一使用 `phase`。

`super.build_phase(phase)` 中的 `super` 是 SystemVerilog 关键字，指向当前类的直接父类。这里的作用是把uvm框架传入的 phase 对象转发给父类的 `build_phase`。

`type_id::create("agent", this)` 中的 `type_id` 是 factory 注册宏生成的静态成员，它内部维护着这个类的注册信息。`this` 是当前组件的指针，传给 factory 后，新创建的组件会被挂在当前组件下，成为它的孩子。

build_phase 中应当做的事：创建子组件、读取配置。不应当做的事：跨组件连接（放 connect_phase）、驱动信号（放 run_phase）。

**config_db 与 build_phase 的关系：**

build_phase 有一个特殊的性质——它是自顶向下执行的。这意味着父组件的 build_phase 先执行，子组件的后执行。这个顺序对 config_db 至关重要。

```systemverilog
// test 的 build_phase（最先执行）
function void my_test::build_phase(uvm_phase phase);
    super.build_phase(phase);
    uvm_config_db#(virtual my_interface)::set(
        this, "env.agent.driver", "vif", vif
    );
endfunction

// driver 的 build_phase（最后执行）
function void my_driver::build_phase(uvm_phase phase);
    super.build_phase(phase);   // ← 这一行触发 config_db 的读取
    uvm_config_db#(virtual my_interface)::get(
        this, "", "vif", vif
    );
endfunction
```

`super.build_phase(phase)` 的内部有一个关键操作：从 config_db 中读取所有设给当前组件的配置项并应用到组件内部。如果在 driver 的 build_phase 中跳过 `super.build_phase(phase)`，test 设的 interface 就传不到 driver 中。

driver 收到的 config 值在 build_phase 完成后才能使用——因此子组件的创建（`type_id::create`）必须在 `super.build_phase(phase)` 之后，因为创建子组件时子组件的 build_phase 需要读到父组件设好的配置。

**一句话关系：build_phase 是 config_db 的 get 时机，super.build_phase 是触发 get 的入口。**

↩ [[#主要-Phase-详解|回到主要 Phase]] · [[#UVM-Phase-详解笔记|回到目录]]

---

### connect_phase — 建立组件间连接

自底向上、function phase。组件在此阶段建立端口之间的连接。

```systemverilog
function void my_env::connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    agent.monitor.item_port.connect(scoreboard.analysis_export);
endfunction
```

**语法细节：**

`my_env::connect_phase` 中的 `::` 是作用域解析符，表示这个函数属于 `my_env` 类。也可以写为在类内部定义：

```systemverilog
class my_env extends uvm_env;
    function void connect_phase(uvm_phase phase);
        // ...
    endfunction
endclass
```

两种写法等价。

`super.connect_phase(phase)` 与 build_phase 中的规则完全一致——调父类的 connect_phase，确保框架逻辑不丢失。

`agent.monitor.item_port.connect(scoreboard.analysis_export)` 这行由三部分拼接而成：

- `agent.monitor.item_port` 是组件树路径。`agent` 是 env 的子组件，`monitor` 是 agent 的子组件，`item_port` 是 monitor 类中声明的一个 TLM 端口变量。点号 `.` 逐级导航到目标组件内部的端口。
- `.connect()` 是 TLM 端口的内建方法，用于将两个端口绑定在一起。数据从一个端口发出，通过连接通道流入另一个端口。
- `scoreboard.analysis_export` 是 scoreboard 组件内部声明的一个 TLM 导出端口，作为数据的接收端。

整行含义：将 monitor 的数据发送口连到 scoreboard 的接收口。连接完成后，monitor 发出的 transaction 对象自动到达 scoreboard。

connect_phase 的执行时机在所有组件的 build_phase 都完成之后、run_phase 开始之前。此时所有组件都已创建完毕，可以安全地通过树路径引用其他组件的端口。

应当做的事：port/export 绑定、TLM fifo 连接。不应当做的事：创建新组件（放 build_phase）、发送或接收数据（放 run_phase）。

↩ [[#主要-Phase-详解|回到主要 Phase]] · [[#UVM-Phase-详解笔记|回到目录]]

---

### run_phase — 执行核心逻辑

全体并行启动、task phase。所有组件的主要工作在此阶段完成。

- test：创建并启动 sequence，控制激励的发送数量和顺序
- driver：循环等待 sequencer 分发 item，按协议驱动到 DUT 引脚
- monitor：循环采样 DUT 引脚信号，拼成 item 向外广播
- scoreboard：等待接收来自 monitor 和 reference model 的数据，比对结果

```systemverilog
task run_phase(uvm_phase phase);
    my_sequence seq = my_sequence::type_id::create("seq");
    phase.raise_objection(this);
    seq.start(env.agt.sqr);
    phase.drop_objection(this);
endtask
```

**语法细节：**

`task` 是 SystemVerilog 关键字，与 `function` 的关键区别是 task 内部可以包含耗时操作（`#delay`、`@(posedge clk)`、`wait()`）。run_phase 必须是 task 而不能是 function，因为驱动 DUT 信号、等待时钟沿这些操作都需要消耗仿真时间。

`my_sequence seq = my_sequence::type_id::create("seq")` 是 factory 创建对象的方式。`seq` 是一个对象句柄，等号右侧通过 factory 创建了一个 `my_sequence` 实例，句柄赋给 `seq`。它与 `new("seq")` 的区别：`create` 会检查 factory 表中是否有 override 信息。

`phase.raise_objection(this)` 中的 `phase` 是 run_phase 的参数，通过点号 `.` 调用 `uvm_phase` 类中的 `raise_objection` 方法。`this` 是 SystemVerilog 关键字，指向当前组件自身，让框架知道"是哪个组件提了 objection"。

`seq.start(env.agt.sqr)` 启动 sequence 并将它挂到指定的 sequencer 上。`env` 是 test 的成员变量（子组件），`agt` 是 env 的子组件，`sqr` 是 agent 内的 sequencer。sequence 通过这个路径找到要挂载的 sequencer，之后造出的 item 会自动进入该 sequencer 的队列。

`phase.drop_objection(this)` 与 raise 对应，通知框架"当前组件的活干完了"。

run_phase 不会自动结束。框架等待所有组件完成工作后才会退出 run_phase。组件通过 objection 机制告诉框架是否还有工作。

↩ [[#主要-Phase-详解|回到主要 Phase]] · [[#UVM-Phase-详解笔记|回到目录]]

---

### objection — 控制 run_phase 何时结束

Objection 是 phase 框架中控制 task phase 结束时机的机制。

run_phase 启动时所有组件并行运行各自的 run_phase 任务。许多组件的 run_phase 内部是 forever 循环（driver 永远在等下一笔 item，monitor 永远在等下一次采样），它们不会主动结束。问题在于：只有当所有组件的 run_phase 都完成时，run_phase 才真正结束。

Objection 解决这个问题：组件调用 `phase.raise_objection(this)` 时，等于通知框架"我还有工作，不要结束"；调用 `phase.drop_objection(this)` 时，等于通知"我干完了"。当所有组件都 drop 后，run_phase 结束。

```systemverilog
task run_phase(uvm_phase phase);
    my_sequence seq = my_sequence::type_id::create("seq");

    phase.raise_objection(this);          // 通知框架：我还有活
    seq.start(env.agt.sqr);               // 启动 sequence
    phase.drop_objection(this);           // 通知框架：干完了
endtask
```

**为什么 raise 和 drop 必须调用在同一个 task 里？**

因为 UVM 要求 objection 的同步管理——raise 和 drop 之间的代码不能跨任务执行，否则框架无法确定一个 objection 何时应该释放。

通常只在 test 的 run_phase 中使用 objection。driver、monitor、scoreboard 的 run_phase 内部不碰 objection——它们永远在等待数据，不需要主动声明"还在工作"。如果没有任何组件调用 `raise_objection`，run_phase 在零时刻直接结束，sequence 来不及发送任何数据。

↩ [[#主要-Phase-详解|回到主要 Phase]] · [[#UVM-Phase-详解笔记|回到目录]]

---

### report_phase — 输出测试结果

自底向上、function phase。各组件输出各自的统计信息。

```systemverilog
function void report_phase(uvm_phase phase);
    super.report_phase(phase);
    `uvm_info("SCOREBOARD", $sformatf("通过 %0d 笔，失败 %0d 笔",
        pass_count, fail_count), UVM_LOW)
endfunction
```

**语法细节：**

`` `uvm_info `` 是 UVM 的消息宏，格式为：

```systemverilog
`uvm_info("ID字符串", "消息内容", 冗余度级别)
```

第一个参数是一个标签（ID），用于在仿真日志中快速筛选某类消息。第二个参数是具体消息文本。第三个参数是冗余度级别（`UVM_LOW`、`UVM_MEDIUM`、`UVM_HIGH`），用于控制消息的打印密度——当全局冗余度阈值设为 `UVM_MEDIUM` 时，`UVM_LOW` 级别的消息会打印，`UVM_HIGH` 级别的被过滤。

`$sformatf()` 是 SystemVerilog 系统函数，与 `$display()` 的格式化规则相同，但它返回一个字符串而不是直接打印。`%0d` 表示十进制整数，`%0d` 中的 `0` 表示不保留前导空格。`pass_count` 和 `fail_count` 是 scoreboard 类中定义的成员变量，记录比对结果。

report_phase 中常见的输出内容包括：总运行笔数、通过笔数、失败笔数、覆盖率统计。

应当做的事：打印最终统计结果、断言检查、覆盖率总结。不应当做的事：启动新仿真流程、修改组件状态。

↩ [[#主要-Phase-详解|回到主要 Phase]] · [[#UVM-Phase-详解笔记|回到目录]]

---

### extract_phase、check_phase、final_phase

这三个 phase 位于 report_phase 之后，同样自底向上执行。

**extract_phase** 用于从组件中提取最终数据（如寄存器值、最终状态），供 check 阶段使用。**check_phase** 用于最终合法性检查（断言、预期结果比对）。**final_phase** 用于收尾清理（关闭文件、释放内存）。实际工程中较少手动重写这三个 phase。

↩ [[#主要-Phase-详解|回到主要 Phase]] · [[#UVM-Phase-详解笔记|回到目录]]

---

## UVM 1.1 的 12 个 task phase（了解即可）

当前主流版本（UVM 1.2）使用单一的 `run_phase`，但白皮书和许多旧资料基于 UVM 1.1，其中 run 阶段被拆分为 12 个子 phase：

```text
run_phase（整体）
  ├── pre_reset_phase
  ├── reset_phase
  ├── post_reset_phase
  ├── pre_configure_phase
  ├── configure_phase
  ├── post_configure_phase
  ├── pre_main_phase
  ├── main_phase          ← 白皮书里常提到的"主 phase"
  ├── post_main_phase
  ├── pre_shutdown_phase
  ├── shutdown_phase
  └── post_shutdown_phase
```

这些子 phase 按顺序执行，每个都可以通过 objection 独立控制结束时机。设计目的是让不同组件的不同操作（复位、配置、主激励、结束）在时间上对齐。

实际使用中，大多数人不需要这么细的拆分。UVM 1.2 将其合并为单一的 `run_phase`，统一用 objection 控制。如果你的项目明确使用 UVM 1.1，则需要注意 `main_phase` 才是发送主要激励的阶段；如果使用 1.2 或更高版本，只用 `run_phase` 即可。

↩ [[#UVM-Phase-详解笔记|回到目录]]

---

## Phase 执行全过程汇总

```text
仿真开始
    │
build_phase      （自顶向下，function，创建子组件 + 读取配置）
    │
connect_phase    （自底向上，function，绑定 port/export）
    │
run_phase        （全体并行，task，驱动 DUT + 采集 + 比对）
    │               ← raise/drop objection 控制结束
report_phase     （自底向上，function，输出测试统计）
    │
extract_phase    （自底向上，function，提取最终状态）
    │
check_phase      （自底向上，function，最终合法性检查）
    │
final_phase      （自底向上，function，关闭文件/清理资源）
    │
仿真结束
```

↩ [[#UVM-Phase-详解笔记|回到目录]]

---


