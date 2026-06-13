## 第一条线 组件的创建。即`build_phase`，方向top-down。主要任务是create 下级组件 + 用 config_db **分发 cfg**；
### 1 tb 层
  1 对外暴露接口（也就是声明信号 并初始化时钟信号）aclk，arestn，mosi，cs，sck。内部把暴露的接口和interface里的信号连接起来。
  2 实例化interface，并传virtual interface进uvm (uvm_test_top)
  3 实例化 sva 模块
  4 新加miso model （特殊情况）
  5 启动uvm + 波形

### 2 test_base(uvm_test_top)层
  主要是为了后续test创建基础环境
  1 通过factory creat env
  2 创建config；创建env的config，创建axi和spi的config
  3 把top层传进uvm的vif 分发(get)到具体的 vif；把env的config 分发下去。
  4 打印仿真树

### 3 env层
  1 创建 agent（axi_acitve,spi_passive）、scb、coverage组件。、
  2 拿env_cfg， 传 axi_cfg以及spi_cfg。
  
### 4 agent
#### axi
 1 拿config然后分发下去。
 2 创建monitor，driver，sequencer；
  
## 第二条线是 connect_phase，组件之间的连接。




