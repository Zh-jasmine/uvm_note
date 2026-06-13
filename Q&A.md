`AXI4 Full` 的核心是两件事：

1. **支持 burst**  
    一次地址握手后，可以连续传多拍数据，这才是它相比 `AXI4-Lite` 最本质的增强点。
    
2. **支持更高并发**  
    可以有多笔 transaction 同时在路上，也就是常说的 `outstanding`。为了区分这些事务，Full 一般会配合 `ID` 使用。
    

“乱序”只是这个高并发能力**可能带来的结果**


| 参数      | set 端                                      | get 端                          | 作用           |
| ------- | ------------------------------------------ | ------------------------------ | ------------ |
| 作用域     | `null` + `"uvm_test_top"` → `uvm_test_top` | `this` + `""` → `uvm_test_top` | 两边拼出的路径一致才匹配 |
| field 名 | `"axi_vif"`                                | `"axi_vif"`                    | 字典的 key，要对上  |
| 值       | 存入 `axi_vif` 句柄                            | 取出写进 `env_cfg.axi_cfg.vif`     | vif 完成交接     |
## 为什么 set 在 `initial` 里、`run_test()` 之前
必须先 set 再 `run_test`。因为 `run_test` 之后才进 build_phase，test 的 build_phase 一执行就要 get vif。**先存好，get 才取得到**——顺序反了会 get 失败、`uvm_fatal`。

uvm_config_db#(virtual axi_interface)::set(null, "uvm_test_top", "axi_vif", axi_vif);

**`axi_interface` 不是 module，是 interface**。`virtual axi_interface` 是「指向 interface 实例的句柄」

`axi_interface axi_vif();` 这行**不是声明句柄**，它是**例化了一个实体**（跟 `AXI_slave_top DUT(...)` 例化 module 一个道理）。`axi_vif` 是实体，不是指向谁的引用。真正的「句柄」是 driver 里那个 `virtual axi_interface vif;`——它才是个引用，要靠 `vif = axi_cfg.vif;` 才指向实体。

## 那为什么 set 的值能传 `axi_vif`（实例），而类型参数是句柄？

```
uvm_config_db#(virtual axi_interface)::set(null, "uvm_test_top", "axi_vif", axi_vif);//             └─ 类型参数 T = 句柄类型                                        └─ 传的是实例
```

因为 SV 有**隐式转换**：当你把一个 interface 实例（`axi_vif`）传给一个 `virtual axi_interface` 句柄类型的形参时，SV 自动生成一个「指向 `axi_vif` 的句柄」存进去。


|         | 第一条线                                              | 第二条线                              |
| ------- | ------------------------------------------------- | --------------------------------- |
| phase   | `build_phase`                                     | `connect_phase`                   |
| 方向      | **top-down**(自顶向下)                                | **bottom-up**(自底向上)               |
| 做什么     | create 下级组件 + 用 config_db **分发 cfg**              | 连接 TLM 端口(port ↔ export/imp)      |
| 为什么这个方向 | 父必须先存在,才能决定建哪些子、给什么配置(如 `is_active` 决定建不建 driver) | 连接要求**双方组件都已存在**,所以必须等 build 全部完成 |
