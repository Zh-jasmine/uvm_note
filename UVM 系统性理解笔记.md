---
tags:
  - UVM
  - 验证
date: 2026-05-01
---

# UVM 系统性理解笔记

## 核心原则

不背继承链，每个机制先问它解决了什么问题。

---

## 一、为什么需要 UVM

裸写 SystemVerilog 验证环境时，每个测试都需要手动定义信号、手动构造每一笔数据、手动编写复制 / 比较 / 打印等功能。数据量少时尚可应对，当需要发送成百上千笔数据时，这种方式完全不可维护。

UVM 框架的做法是：把验证环境中反复出现的通用需求（打印、复制、比较、消息报告、生命周期管理、组件替换等）抽象成标准类，验证工程师通过继承这些类直接获得对应能力，把精力集中在真正需要定制的部分——数据字段的定义和 DUT 相关的驱动逻辑。

---

## 二、类树全景

```
uvm_void                       空壳，纯类型占位，不能实例化
  └─ uvm_object                增加打印/复制/比较/打包等操作
       │
       ├─ uvm_report_object    增加消息报告能力（info/error/warning/fatal）
       │    └─ uvm_component   增加 phase 生命周期
       │         ├─ driver      将 item 拆成 pin 级信号驱动 DUT
       │         ├─ monitor     从 pin 级信号采样并拼回 item
       │         ├─ scoreboard  比对发送和接收的 item
       │         ├─ sequencer   sequence 与 driver 之间的调度管道
       │         ├─ agent       打包 driver、monitor、sequencer
       │         ├─ env         打包所有 agent 和 scoreboard
       │         └─ test       最顶层，负责 override 和启动 sequence
       │
       └─ uvm_transaction     增加随机化支持和事务ID
            └─ uvm_sequence_item 增加溯源信息
                 ├─ 自定义transaction   具体的数据包
                 └─ uvm_sequence_base
                      └─ uvm_sequence   item 的批量生成器
```

从 uvm_object 往下，整个类树分成两条支线。之所以要分开，是因为两类对象需要的职责完全不同：数据对象不需要生命周期也不需要发消息，而环境组件不需要随机化。各取所需，互不引入冗余能力。

---

## 三、逐层说明

### uvm_void

`uvm_void` 是一个空的 virtual class，内部没有定义任何方法或属性，也不能直接实例化。它的作用只是为整个 UVM 类树提供一个统一的根节点，所有 UVM 类归根结底都派生自它，这样在需要传递任意类型对象的地方（例如 `uvm_config_db` 的默认类型参数）就有了一个通用的类型占位符。日常编写验证环境时不会直接用到它。

### uvm_object

`uvm_object` 在 `uvm_void` 的基础上增加了一组操作型方法：`print()` / `sprint()` 用于打印对象内容，`copy()` / `clone()` 用于复制对象，`compare()` 用于比较两个对象的字段是否一致，`pack()` / `unpack()` 用于将对象打包成字节流或从字节流还原，`record()` 用于将对象字段记录到仿真波形中，`get_type_name()` 用于获取类名。这些方法全部是 virtual 的——框架只定义了函数签名和调用流程，具体字段的打印 / 复制 / 比较逻辑由派生类通过 `do_print()` / `do_copy()` / `do_compare()` 等钩子方法来实现。

uvm_object 本身不包含任何数据属性。它提供的是操作对象的通用工具箱，继承它就免去了重复编写这些样板代码。

### 分支一：transaction 支线

**uvm_transaction** 在 `uvm_object` 的基础上增加了随机化支持和事务 ID。它引入了一个 `rand` 修饰的 `m_transaction_id` 字段，每笔 transaction 拥有一个唯一的 32 位编号。同时它重写了 `do_copy()`、`do_compare()`、`do_pack()`、`do_record()` 等方法，确保复制或打包时事务 ID 也被一并处理。

transaction 本质上是"对 DUT 完成一次操作所需信息的抽象"——比如一次寄存器读写操作需要地址、数据和读写标志，这三个字段就组成一个 transaction。不同 DUT 需要的字段不同，由验证工程师在自定义 transaction 类中添加。

**uvm_sequence_item** 在 `uvm_transaction` 的基础上增加了溯源能力：`get_sequence_id()` 返回是哪个 sequence 生成了这笔 item，`set_item_context()` 记录当前所处的 sequence 上下文，`get_depth()` 返回嵌套层数（一个 sequence 内部调用另一个 sequence 时的层级）。当仿真中同时运行多条 sequence、发出数万笔 item 时，如果某一笔数据出错，通过这些溯源信息可以立刻定位到具体是哪条 sequence 产生的。实际编写 transaction 时，直接继承 `uvm_sequence_item` 即可。

**uvm_sequence_base** 同样派生自 `uvm_sequence_item`。这意味着 sequence 本身也是 `uvm_sequence_item` 的派生类——设计原因在于 sequence 支持嵌套：一条父 sequence 可以将子 sequence 当作一笔 item 发送给 sequencer。sequence 不是 component，没有 phase 机制；它的唯一职责是在 `body()` 任务中循环创建 item、调用 `randomize()` 随机化字段、通过 `start_item()` / `finish_item()` 将 item 发送出去。sequence 是 item 的批量生产者。

**sequencer** 是 sequence 和 driver 之间的调度管道。sequence 调用 `finish_item()` 时，item 进入 sequencer 的内部队列；driver 通过 `get_next_item()` 从队列中取出 item。这种设计将数据生产（sequence）和信号驱动（driver）完全解耦——更换 driver 实现时，sequence 代码无需任何改动，反之亦然。

流水线全景：sequence 生产 item → sequencer 排队缓冲 → driver 取出 item 并拆解为 pin 级信号 → DUT 处理 → monitor 从 pin 级信号采样并拼回 item → scoreboard 比对发送端与接收端的 item。

### 分支二：report_object 与 component 支线

**uvm_report_object** 在 `uvm_object` 的基础上增加了消息报告能力：`` `uvm_info()``（普通信息）、`` `uvm_warning()``（警告）、`` `uvm_error()``（错误但不停止仿真）、`` `uvm_fatal()``（致命错误并终止仿真）。这一层存在的理由在于：消息报告是环境组件的需求，而 transaction 作为数据载体不需要也不应该具备发送消息的能力。将消息功能独立为一层，transaction 支线就不必背负它用不到的方法。

**uvm_component** 在 `uvm_report_object` 的基础上增加了 phase 机制，为所有环境组件提供了统一的生命周期调度。

Phase 解决的核心问题是：验证环境中存在大量组件（driver、monitor、scoreboard、sequencer 等），这些组件的创建、互联和运行有严格的先后顺序——必须先创建所有组件，再建立它们之间的连接，最后才能启动仿真逻辑。如果没有统一的调度机制，每个验证工程师需要自行协调这些顺序，极易出错。

UVM 将组件的生命周期划分为以下 phase：

- `build_phase`：创建子组件。每个 component 在此阶段通过 factory 的 `type_id::create()` 创建其下层的子组件实例。此阶段为 function，不消耗仿真时间。
- `connect_phase`：建立组件之间的连接。driver 与 sequencer 之间、monitor 与 scoreboard 之间的 port 连接均在此阶段完成。此阶段为 function。
- `run_phase`：执行主要的仿真逻辑。所有 component 的 run_phase 并行启动，各自执行自身任务。此阶段为 task，可以消耗仿真时间。
- `report_phase`：输出测试结果。各组件在此阶段输出统计信息和 PASS / FAIL 结论。此阶段为 function。

Phase 的执行顺序遵循固定规则：build_phase 自顶层向底层依次执行（父组件先创建，子组件后创建），connect_phase 和 report_phase 自底层向顶层依次执行（叶子组件先完成连接和汇报），run_phase 则全体组件并行启动。

driver、monitor、scoreboard、sequencer、agent、env、test 均派生自 `uvm_component`，因此它们都遵循这套统一的生命周期。编写一个 component 的典型模式是：在 build_phase 中创建子组件，在 connect_phase 中连接 port，在 run_phase 中编写核心逻辑，在 report_phase 中输出结果。

---

## 四、Factory 机制

### 解决的问题

在传统的 `new()` 创建方式下，组件类型被硬编码在代码中。如果需要将某个组件替换为另一个版本（例如将普通 driver 替换为错误注入 driver），必须逐个修改所有创建该组件的文件。当验证环境规模扩大，组件数量增多，这种方式的维护成本急剧上升。

Factory 提供了一种"在不修改已有代码的前提下替换组件实现"的机制。

### 工作机制

Factory 本质是一个全局的映射表（key 为类名，value 为创建该类型实例的函数）。每个需要被工厂管理的类通过宏注册到该表中，后续创建实例时不直接调用 `new()`，而是通过 `type_id::create()` 查询该表并创建实例。当需要替换某个组件时，只需要在测试顶层调用 `set_type_override(原类型, 替代类型)`，之后所有对该类型的 `create()` 调用都会返回替代类型的实例。

### 使用步骤

**第一步：注册类。** 在每个需要通过 factory 创建的类中使用注册宏。component 类使用 `` `uvm_component_utils``，object 类使用 `` `uvm_object_utils``。宏展开后会在类内自动生成一个名为 `type_id` 的静态注册表对象，以及 `get_type()` 和 `get_type_name()` 等辅助方法。

**第二步：用 create 替代 new。** 在 build_phase 中创建子组件时，使用 `类名::type_id::create("实例名", this)` 替代 `new("实例名", this)`。对于 object 类型（如 transaction），create 调用不需要 parent 参数。

**第三步（按需）：override 替换。** 当需要替换某个组件时，在 test 的 build_phase 中写入一行 `原类型::type_id::set_type_override(替代类型::get_type())`。此后环境中所有对该类型的 create 调用都会自动返回替代类型的实例。

### 具体示例

同一个 DUT，需要两种测试——正常发包的测试 A 和乱序发包的测试 B。

不用 factory 的做法：编写两套 env，normal_env 内部写死 normal_driver，random_env 内部写死 random_driver。添加第三种测试时又需要复制一份 env。代码翻倍，维护困难。

用 factory 的做法：只编写一套 env，内部通过 `my_driver::type_id::create()` 创建 driver（不写死具体子类）。然后编写 normal_driver 和 random_driver 两个版本（均继承自 my_driver）。测试 A 不做任何额外操作，默认获得 normal_driver；测试 B 加一行 `my_driver::type_id::set_type_override(random_driver::get_type())`，所有 create 调用自动返回 random_driver。env 代码始终不变，新增测试只需要写新的 driver 子类并添加一行 override。

### 关键语法

`::` 是 SystemVerilog 的作用域解析符，`A::B` 表示去 A 类中查找静态成员 B。`type_id` 是注册宏在类内自动生成的静态对象，类型为 `uvm_component_registry`（component 类）或 `uvm_object_registry`（object 类），其 `create()` 方法负责查询 factory 表并创建实例。factory 底层是一张哈希表，override 的本质就是将表中某个 key 对应的创建函数替换为另一个。

---

## 五、交易流水线全貌

```
sequence → sequencer → driver → DUT → monitor → scoreboard
```

| 角色 | 扮演什么 | 具体职责 |
|---|---|---|
| transaction | 数据包 | 一次 DUT 操作的信息载体，字段由验证工程师自定义 |
| sequence | 数据包生产器 | 循环创建 item，随机化，发送到 sequencer |
| sequencer | 调度管道 | 维护 FIFO 队列，sequence 往里放，driver 往外取 |
| driver | 信号驱动 | 从 sequencer 取 item，按 DUT 时序拆成 pin 级信号 |
| monitor | 信号采集 | 从 pin 级信号采样，拼回 item，发给 scoreboard |
| scoreboard | 结果比对 | 比较发送端和接收端的 item，判断测试通过或失败 |

---

## 六、Phase 执行顺序

```
时间方向 →

build_phase:           test → env → agent → driver/monitor/sequencer
                       （自顶向下，父先于子创建）

connect_phase:         driver/monitor/sequencer → agent → env → test
                       （自底向上，叶子先完成连接）

run_phase:             所有 component 并行执行

report_phase:          driver/monitor/sequencer → agent → env → test
                       （自底向上，叶子先汇报）

final_phase:           收尾清理
```
