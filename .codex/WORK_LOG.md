**工作记录与上下文交接**

整理日期：2026-10-29（Asia/Shanghai）。项目目录：`/home/chenfred/repos/ncu-report-skill`。

记录范围：当前可见对话中完成的阅读、调研、答疑和文档工作，以及本次发现并读取的已有迁移报告。本文为工作摘要，不是逐字聊天备份。

**1. 用户目标与关注点**

最初任务：仔细阅读仓库，结构化解释主要做了什么、怎么做；总结打印在屏幕并保存为 Markdown。

后续关注的问题：

1. skill 最终输出是什么；是否只是静态分析。
2. 是否存在“采集 → 分析 → 开发 Agent 修改 → 正确性/性能验证 → 再采集”的循环。
3. 输入需要哪些源码、运行入口、shape、dtype、布局、分布或完整数据。
4. line info 如何连接源码和 SASS。
5. 是否能获得计算/访存、不同 warp 之间的指令执行重叠材料；NVIDIA 官方工具支持到什么程度。
6. 具体分析哪些大类与指标，并附简短含义。
7. 当前任务：把工作记录及全部已落盘调研报告索引写入 `.codex/`。

用户曾因 Ctrl+L 清屏要求重新输出总结，因此重要结论与文件导航应可从本目录恢复。

**2. 已完成工作**

| 工作 | 结果 | 落盘位置 |
|---|---|---|
| 阅读 README、SKILL、全部 10 份 reference、全部 helpers、Blackwell 参考 | 梳理流程、实现与能力边界 | [SUMMARY.zh-CN.md](../SUMMARY.zh-CN.md) |
| 输入/输出、动态采集、line info、优化循环答疑 | 区分 skill 和外层开发 Agent 的责任 | 本记录第 3 节及 SUMMARY |
| 核对时间线实现并查 NVIDIA 官方文档 | 区分采样曲线、指令依赖、完整动态指令轨迹 | 本记录第 4 节及 METRICS |
| 逐项核对指标、分类与含义 | 覆盖全部 108 项默认指标，区分专项和按需分析 | [METRICS.zh-CN.md](../METRICS.zh-CN.md) |
| 纳入已有 Ascend 950 迁移报告 | 读取全文、整理主题与验证边界；未复跑其研究过程 | [ASCEND950_MIGRATION_RESEARCH.zh-CN.md](../ASCEND950_MIGRATION_RESEARCH.zh-CN.md) |
| 建立交接目录 | 入口、工作记录、报告与来源索引 | [.codex/README.md](README.md)、[RESEARCH_INDEX.md](RESEARCH_INDEX.md) |

**3. 对话中已明确的关键结论**

**3.1 定位和输出**

该仓库是 Agent Skill、辅助脚本和诊断知识库。完整流程为：确定真实 workload/dispatch → 准备运行目标 → NCU 采集 → 提取指标/规则/热点/时间线 → 六维诊断 → 按影响排序的优化建议。

核心交付物是 `$PROFILE_RUN_DIR/REPORT.md`，包含配置、关键数值、证据、优化优先级、置信度和复现方法；并保留 `harness/`、`reports/`、`analysis/`。

脚本负责数据提取整理，最终因果判断和 REPORT.md 由使用 skill 的 Agent/工程师完成。当前仓库没有完整自动优化器，没有自动生成并验证优化 kernel 的闭环程序。

**3.2 动态采集与离线分析**

它不是完全静态分析。NCU 实际执行并可能多次重放 kernel，采集硬件计数、PC/warp 状态和 PM 序列；Python helpers 可以在无 GPU 环境读取已有报告。

离线读取动态采集数据，不等于静态源码分析。结论受输入、分支、硬件、时钟、采集/重放设置影响。

**3.3 输入要求和 workload 选择**

- 可以从源码、可执行程序、库入口/driver 开始，也可以直接提供已有 `.ncu-rep/.ncu-repz`。
- 准备实际执行时，需要确定 shape、dtype，以及影响行为的布局、stride、tiling/dispatch 和相关输入。
- 没有重要数据相关行为时，可使用真实 shape/dtype 与合理、可复现的合成数值。
- 变长长度、稀疏索引、路由等控制工作量或访问模式的数据，不能随意用均匀随机输入代替；必要时提供完整真实 tensor。
- Agent 应阅读实际 benchmark、输入生成器、launcher 和数据集选择代表输入，不是仅凭核函数源码猜生产热点。
- FlashInfer 工具可以统计 shape、筛选轴值和定位 safetensors；它不会自动识别生产耗时热点，也不会自动理解项目完整 dispatch。

**3.4 line info、源码文本和内联映射**

- `-lineinfo` 在目标代码中保留 PC 到源码位置的映射；没有它仍可进行整体指标分析，但源码定位受限。
- `--import-source yes` 在采集时嵌入能够找到的源码文本，和 line info 不是同一件事。
- 源码行与 SASS 不是一一对应；优化、循环、模板、内联使关系更复杂。
- `extract_stall_hotspots.py --binary` 使用 `cuobjdump`、`nvdisasm -gi` 读取内联链，把头文件采样归回 kernel 自身行。
- 内联处理会比较 SASS 主操作码作为一致性线索，不是完整二进制一致性证明。

**3.5 优化循环**

当前 skill 支持重复采集、针对性指标复测和 A/B 比较，因此可以被外层 Agent 放入循环：

```text
基线实现
    → skill 采集与诊断
    → 开发 Agent 选择候选改动
    → 正确性验证与独立 benchmark
    → 修复/回退或接受版本
    → skill 再采集、比较与分析剩余瓶颈
    → 达到停止条件后结束
```

正确性、版本管理、回退、优化预算和停止条件，需要外层工作流负责。README 提到 KDA 主仓库，但当前可见对话没有检查它是否已实现完整自动循环。

**3.6 指标和诊断的解释边界**

- 默认 `KEY_METRICS` 共 108 项，不代表每份报告都有全部值；None 不是零。
- `dump_all_metrics()` 保存指标汇总值与单位，不是原始报告的无损完整导出。
- Occupancy 反映驻留；停顿 warp 仍可驻留。active、eligible、issued 要分开解释。
- Stall ratio、PC 样本占比与 kernel 时间占比是不同口径。
- 热点列表含 selected/not_selected，不能把总样本视为可消除的停顿时间。
- local memory 指令可能来自 spill、局部数组或栈，不可只凭非零断定 spill。
- sectors/request 的参考值依赖每 lane 宽度、活跃 lane、对齐和广播模式。
- Tensor/TMEM 指标需匹配架构和 dtype，不能只凭一个旧管线指标为零断言完全没有 Tensor Core 活动。
- NCU 规则收益可能重叠且 local/global 范围不同，不能简单相加。
- 仓库参考文档有简化和内部不一致，SUMMARY 与 METRICS 已明确相关限制。

**4. 指令流水和官方工具调研记录**

用户希望看到横轴为时间、纵向为指令类别或不同 warp、可表现计算与访存重叠的材料。

当前 skill 的 `plot_timeline.py` 输出 PM 指标曲线及分桶后的 warp 状态占比，可以辅助观察长尾、气泡、计算/内存活动阶段；没有逐 warp 的动态泳道，也没有每条指令的发射与完成日志。

line info、SASS 静态顺序、累计执行次数和 PC 样本不能唯一还原每个 warp 的每次循环执行过程。多个 replay 的曲线也不能直接视为同次执行的逐点重叠证据。

已查阅的官方能力：

| 工具/视图 | 已确认的用途 | 边界 |
|---|---|---|
| NCU PM Sampling timeline | 观察计算/内存相关采样指标随时间变化 | 采样，不是完整指令执行轨迹 |
| NCU Function Stats | 按时间范围观察函数/源码行的采样和主要停顿 | 依赖版本与采集内容 |
| NCU Source/SASS、Instructions & Dependencies | 查看指令类别、寄存器和 scoreboard 依赖 | 依赖图不是实际起止时间图 |
| Nsight Systems | kernel、stream、CUDA memory operation、CPU API 和 GPU metrics 时间线 | CUDA 拷贝事件不能当成 kernel 内每条 LDG/STG 的区间 |
| NVBit | NVIDIA Research 动态 SASS 插桩框架，可用于开发 tracer | 不是现成无扰动流水图；插桩会改变执行时序 |

本次查阅未发现可直接读取普通 NCU 报告并完整恢复所有 warp、所有指令发射/完成区间的标准视图；这是调研范围内的结论，不是对所有未来工具能力的保证。

提出但尚未实施的扩展方向：以 NCU 定位问题，再选择特定 block/warp/角色，在 tile/stage 边界做有选择的时间戳插桩，输出阶段级泳道图。异步 TMA/MMA 的发起与完成必须区分，不能把发起指令前后的短区间误认为操作完整执行区间。

来源入口集中在 [RESEARCH_INDEX.md](RESEARCH_INDEX.md) 第 5 节。

**5. 已有 Ascend 迁移报告的交接摘要**

本次发现根目录已有 R03，读取后纳入索引。以下内容来自该报告，未在本次整理中重新做外部验证：

- 研究 msopprof 真机、msopprof simulator、npusim CA 的能力并集，建议保留测量→诊断的方法，重写架构模型、指标适配和规则。
- 报告说明 950 真机支持受采样/覆盖限制的 PipeTimeline、InstrTimeline 和 SIMT 分析，不能称为全芯片无损 trace。
- npusim CA 提供模型时序；PV 为功能仿真。当前某些开源报告后端与指南存在字段、轨道和占位值差异。
- core0 可作为代表核的深挖入口，但不能替代全核概览、最慢核与特殊角色检查。
- CUDA occupancy 与 msopprof 同名字段不能直接等价；重叠率、带宽、利用率须保留各自定义。
- 真机计数、硬件采样、插桩重放、CA 模拟不得拼成声称同次实际执行的时间线。
- 报告记载历史执行过 3 项 npusim 合成 DB 报告测试；没有运行真实 NPU kernel 或 CA 模型，没有实施 skill 移植。

具体工具 commit、源码引用和待现场验证项见 [ASCEND950_MIGRATION_RESEARCH.zh-CN.md](../ASCEND950_MIGRATION_RESEARCH.zh-CN.md)，不将报告记载的历史测试计为本对话实测。

**6. 已执行的检查与未验证事项**

本对话此前实际完成：

- 对 5 个 Python 文件执行 AST 语法解析，均通过；不是运行时/API 兼容性测试。
- 通过 AST 读取 `KEY_METRICS`，确认 108 项。
- 检查 METRICS 文档中分组缩写和状态/管线模板展开后覆盖全部默认指标。
- 检查 SUMMARY、METRICS 的本地链接和代码围栏。
- 查询本机 `ncu --version`，得到 2026.1.1.0。
- 用 `ncu --query-metrics --chip gb100` 离线核对部分定义；这不需要执行 kernel，也不证明本机实际 GPU 为 GB100。
- 文档修改后执行过 `git diff --check`。

没有完成或没有验证：

- 没有为用户实际采集任何 `.ncu-rep`，没有指定待优化 kernel。
- 没有对真实报告运行完整三脚本链路，没有验证所有架构/版本兼容性。
- 没有修改或优化算子，没有加速收益、正确性或数值误差实测。
- 没有实现自动优化循环、NVBit tracer 或 warp/stage 插桩。
- 没有检查 KDA 主仓库的调度实现。
- 没有实施 Ascend 移植或进行目标 950 真机验证。

**7. 2026-10-29 本次归档动作**

- 操作前 `.codex/` 不存在，工作区状态干净。
- 操作前 HEAD 为 `5749757f68a59c81d04573a6bbc8fc7ce72b9db8`（`[WIP] investigate`）；3 份中文报告均已跟踪。该提交是本次整理前已有状态，不是本次执行了 commit。
- 新建 `.codex/README.md`、`.codex/RESEARCH_INDEX.md`、`.codex/WORK_LOG.md`。
- 报告正文保留原路径，原有源码和报告内容未修改。
- 本次任务是记录与索引，不触发采集、移植或开发工作。

**8. 后续可选方向（等待用户指定，不是已启动任务）**

| 方向 | 先决信息/动作 |
|---|---|
| 实际体验当前 skill | 选择一个可运行 kernel、代表输入与基线，确认 GPU 和性能计数器权限 |
| 核对 KDA 自动优化闭环 | 阅读 KDA 主仓库的开发 Agent、调度和验证流程 |
| 扩展计算/访存重叠可视化 | 明确需要采样统计还是阶段级实测；取得相同时间基准的事件 |
| 实施 Ascend 950 移植 | 锁定 SoC、CANN、工具版本和报告后端，获取最小真实样例，先验证字段/单位 |

恢复工作时，先用 `git status --short` 确认是否有新的并行修改，再查报告索引。不要把历史分析结论当作当前环境实测，也不要从报告中的实施建议推断用户已经授权开始开发。
