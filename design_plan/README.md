# SoC 事务级仿真原型：设计计划

状态：规划阶段，未开始实现。更新日期：2026-11-12。

## 目录

- [1. 阅读顺序](#1-阅读顺序)
- [2. 当前共识](#2-当前共识)
- [3. 文档状态与变更规则](#3-文档状态与变更规则)
- [4. 来源与边界](#4-来源与边界)

## 1. 阅读顺序

| 文档 | 作用 |
|---|---|
| [01 · 原型总体计划](./01_prototype_plan.md) | 目标、架构、模块范围、性能路线与里程碑 |
| [02 · 设计规范草案](./02_design_contracts.md) | 组件、时间、事件、资源、错误和扩展接口的约束 |
| [03 · Checkpoint 与恢复](./03_checkpoint_and_restore.md) | 保存切面、内容、文件生命周期、恢复和兼容性 |
| [04 · 验证与性能计划](./04_validation_and_performance.md) | 正确性、恢复等价、负向测试与性能度量 |
| [05 · 决策与待确认项](./05_decisions_and_open_questions.md) | 已确认决策、建议方案、未决问题与讨论顺序 |
| [06 · 读写事务与内存可见性](./06_memory_transaction_contract.md) | 已确认 Q2：生效点、冲突顺序、大小/地址、响应、非法访问与恢复要求 |
| [07 · 时间、时钟、CDC 与超时](./07_time_clock_cdc_and_timeout.md) | Q3：1 ps 有理数时钟、双域、同刻可见性、固定宽限期与恢复 |
| [08 · 互连与内存服务](./08_interconnect_and_memory_service.md) | Q4：双向链路、消息 RR、credit、冲突门控、响应背压及双域手算 |
| [09 · 纯 C++ 离散事件内核](./09_discrete_event_kernel.md) | 替代 SystemC 的已确认路线；组件状态、逻辑句柄、调度与恢复的待审方案 |

建议先读 01，再用 05 确认范围；09 解释当前内核路线，06–08 依次解释数据、时间、资源语义，02–04 用于审查统一规范与恢复验收。

## 2. 当前共识

第一版是纯 C++ 确定性离散事件内核驱动的独立 SoC 事务级时序仿真原型，研究请求源、互连/credit 与内存服务之间的延迟、吞吐和拥塞。不执行真实 CPU 指令，不启动 OS。2026-11-12 确认 C36，取代原 C++ + SystemC 技术选择，首版移除 SystemC 依赖。

首版采用受限读写与真实 payload。Q2 已确认单目标连续 RAM（默认 1 MiB、冷启动全零）、1–64 B 指定大小的自然对齐访问、目标执行时生效、冲突按目标接纳顺序及允许乱序响应，详见 [06](./06_memory_transaction_contract.md)。

Q3/Q4 主要行为已确认：首版即支持 1 ps 网格上的有理数时钟、源/互连与内存双域、事务级 CDC；同拍新消息/credit/释放槽位不旁路，预定链路结束允许无缝接续。两源共享独立双向链路，按消息 RR 仲裁；目标启动前检查冲突，响应满则占服务槽等待，不预留响应槽、不提前执行读写。Timeout 采用一次固定源域周期宽限，最终时刻完成优先。

本轮进一步确认：响应使用共享总池，整条发送结束才释放槽；每源消费宽度为可配置正整数、默认 1。CDC 第一个严格更晚边沿算第 1 级，第 S 级可见；同刻到期消息按发送序号批量入队。服务 L 与启动间隔 II 均为正整数，按实际启动边沿计数。

默认频率、容量、延迟数值、时钟表示范围及初始化等仍待冻结。详见 [07](./07_time_clock_cdc_and_timeout.md) 和 [08](./08_interconnect_and_memory_service.md)，不能将其余教学参数视为已批准默认值。

Checkpoint 是核心能力：在明确的模型提交边界保存，保留队列、在途事务和未来事件；新进程恢复后，要求规定的后续模型可观察轨迹与不中断执行一致。

所有参与恢复的组件必须遵守显式状态、可序列化动作的统一契约。不以挂起线程/协程帧作为可恢复状态，不承诺直接接入现有 n_project 模块；未来 SystemC 适配需另行设计，首版不承诺双后端。

进一步确认 C37–C43：单宿主线程、全局可索引最小堆；仅 Queued 可取消/改期，Selected 起锁定；真实改期保持 ID 并更新版本/排序。每组件只维护最早下一次激活，业务事实分别保存。统一周期不延后业务，默认诊断快照看提交前，checkpoint/完成结算保持提交后。身份编码、批次阶段与恢复表示等余项见 [09](./09_discrete_event_kernel.md)，尚未全部冻结。

## 3. 文档状态与变更规则

- **已确认 C**：用户已经明确选择的目标或边界。
- **建议 P**：为了实现目标提出的架构或规范；尚未逐项获得用户确认。
- **待确认 Q**：存在多种合理方案，需要继续讨论。
- **后续范围 F**：首版暂不承诺实现的能力。

02–04 中除明确关联已确认决策的规则外，“必须”表示**对应草案被批准后**的实现要求，不表示用户已经逐条确认。06–09 区分确认行为、推导和待审细节。事件日历与主动移除路线已确认，但句柄生命周期、具体 API、C++ 标准和依赖格式仍未冻结。

变更时同步相关契约、验收用例与决策记录。模型语义变化、快照 schema 变化和性能优化应分别说明；不能把恢复不一致当作优化的默认代价。

本目录只是计划和规范，不是已实现功能、测试结果或性能承诺。当前没有新增仿真器代码。

## 4. 来源与边界

设计参考已有知识库，不复制内部产品协议、配置或实现代码：

- [仿真器设计蓝图](../../../doc_skills/doc/03_emulator/03_design_blueprint.md)
- [QEMU、gem5 等模拟器比较](../../../doc_skills/doc/03_emulator/02_simulator_comparison.md)
- [模型契约与验证](../../../doc_skills/doc/03_emulator/08_model_contracts_and_validation.md)
- [事件生命周期、checkpoint 与重放](../../../doc_skills/doc/03_emulator/09_event_lifecycle_checkpoint_and_replay.md)
- [时钟同步与校准](../../../doc_skills/doc/03_emulator/07_clock_synchronization_and_calibration.md)
- [ROI 与统计口径](../../../doc_skills/doc/03_emulator/10_experimental_design_roi_and_uncertainty.md)
- [SoC 贯穿案例](../../../doc_skills/doc/01_soc_design/04_capstone_and_learning_path.md)
- [n_project 通用设计分析](../../../doc_skills/doc/04_n_project/n_project_simulation_design_comparison.md)

当前项目位于 `dy_simulator/dy_soc_simulator_core`；这些链接按该布局指向 Documents 下的知识库，迁移仓库后需调整。原资料的访问和保密边界不因改名而改变。本文描述独立通用原型，不代表对 n_project 当前完整 checkpoint 能力的代码审计。
