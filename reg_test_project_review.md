# reg_test 项目评价（发帖用）

## 优点

**分层完整**
transaction、driver、monitor、scoreboard、agent、env、test 全部落齐，严格遵循了 UVM 组件树结构。

**设计决策有思考**
注意到 DUT 的 rvalid 始终为高导致 monitor 无法区分读状态，采用 monitor 只采样写、driver 主动汇报读结果的双路径设计，不拘泥于模板。

**细节到位**
fork...join_none 后台跑 FIFO、UVM_NO_DPI 配置、config_db 通配符，说明真的跑通过、踩过坑。

## 不足

**DUT 太简单**
256×32 寄存器，时序两个周期走完，没有 pipeline、没有背压、没有 burst，跟真实协议差距大。

**没有 RAL**
标准做法用 uvm_reg + uvm_reg_block + adapter，这里用 sequence 手动发读写，不是正经寄存器验证方法学。

**没有覆盖率收集**
没有 covergroup / coverpoint，验证进度无法量化。

**sequence 复用性弱**
没有 virtual sequence，多 sequence 同时运行的场景扩展困难。

## 后续

用 AES-128 项目把这些不足全部补上：RAL、覆盖率、标准总线、virtual sequence。
