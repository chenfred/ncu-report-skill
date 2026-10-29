**调研报告与资料索引**

更新日期：2026-10-29。本文链接均相对于 `.codex/`，迁移仓库位置后仍可使用。

**1. 已落盘调研报告：共 3 份**

| 编号 | 报告 | 日期 | 内容范围 | 验证与边界 |
|---|---|---|---|---|
| R01 | [SUMMARY.zh-CN.md](../SUMMARY.zh-CN.md) | 2026-10-28 | 仓库定位、目录分工、完整工作流、脚本能力、六维分析、15 类诊断模式、输出结构和限制 | 阅读仓库文档与实现；未运行 GPU 性能采集 |
| R02 | [METRICS.zh-CN.md](../METRICS.zh-CN.md) | 2026-10-28 | 全部 108 项默认精选指标的分类与含义；源码热点、负载均衡、PM 时间线、NCU 规则及按需指标 | 核对默认清单覆盖；通过本机 NCU 离线查询核对部分定义；未验证真实报告解析 |
| R03 | [ASCEND950_MIGRATION_RESEARCH.zh-CN.md](../ASCEND950_MIGRATION_RESEARCH.zh-CN.md) | 2026-10-28 | msopprof 真机、msopprof simulator、npusim CA 的能力并集；950 流水、SIMT、逐核分析、后端差异和迁移建议 | 本次整理读取的已有报告；报告记载历史源码核对及 3 项合成 DB 测试，未运行真实 NPU kernel 或 CA 模型，未实施移植 |

R01、R02 来自本对话的阅读和整理。R03 已存在于当前仓库；其调研过程和测试结果以正文为来源，不能据本次索引操作声称重新完成了那些验证。

**2. 按问题查报告**

| 想了解的问题 | 主要入口 |
|---|---|
| 这个 skill 做什么、结构如何、最终交付什么？ | R01 第 1—4、8 节 |
| 是否自动写出优化后的 kernel？ | R01 第 1、9 节；工作记录中的优化循环说明 |
| 输入是否必须是源码、是否必须给真实数据？ | R01 第 3—4 节；[WORK_LOG.md](WORK_LOG.md) 的输入边界记录 |
| 是否为静态分析、line info 是什么？ | [WORK_LOG.md](WORK_LOG.md) 的关键结论 |
| 默认分析哪些 NCU 指标？ | R02 第 1—12、18 节 |
| 源码热点、时间线和 NCU 规则怎么处理？ | R02 第 13—16 节 |
| Atomic、bank conflict、FP64、分支要看什么？ | R02 第 17 节 |
| 能否展示逐 warp、逐指令执行流水？ | [WORK_LOG.md](WORK_LOG.md) 的时间线调研记录；R02 第 15 节 |
| Ascend 950 是否有更直接的计算/搬运流水材料？ | R03 第 2—5 节 |
| 只分析 core0 是否充分？ | R03 第 7 节 |
| npusim 字段是否能直接视为真机性能？ | R03 第 8 节 |
| 移植这个 skill 需要改哪些模块？ | R03 第 9—10 节 |

部分问答之前仅在聊天中展开，现已汇总到 WORK_LOG；没有把它们误列为此前已经存在的独立报告。

**3. 原始仓库文档：报告的主要依据**

以下为项目原有说明与知识库，不是此次新增的调研报告。

| 文件 | 内容 |
|---|---|
| [README.md](../README.md) | 定位、安装、依赖、直接运行 helpers |
| [SKILL.md](../SKILL.md) | 技能入口、触发条件、采集与诊断主流程 |
| [helpers/README.md](../helpers/README.md) | 工具调用方法 |
| [reference/00-directory-layout.md](../reference/00-directory-layout.md) | 每次实验的目录与命名 |
| [reference/01-workflow.md](../reference/01-workflow.md) | 端到端工作流 |
| [reference/02-harness-guide.md](../reference/02-harness-guide.md) | 原程序与独立 harness、真实输入与合成输入 |
| [reference/03-collection.md](../reference/03-collection.md) | NCU 采集、范围控制、重放、时钟与比较 |
| [reference/04-python-api.md](../reference/04-python-api.md) | `ncu_report` 数据访问 |
| [reference/05-analysis-dimensions.md](../reference/05-analysis-dimensions.md) | 六维分析 |
| [reference/06-diagnosis-playbook.md](../reference/06-diagnosis-playbook.md) | 15 类模式与修复方向 |
| [reference/07-report-template.md](../reference/07-report-template.md) | 最终 REPORT.md 模板 |
| [reference/08-metric-names.md](../reference/08-metric-names.md) | 指标名称及架构/版本差异 |
| [reference/09-common-issues.md](../reference/09-common-issues.md) | 采集、源码归因、JIT、权限等问题 |
| [blackwell-cuda-programming.md](../blackwell-cuda-programming.md) | Blackwell 特性、16 条编程准则与例外 |

**4. 核心实现定位**

| 文件 | 查阅重点 |
|---|---|
| [helpers/ncu_utils.py](../helpers/ncu_utils.py) | `KEY_METRICS`、`dump_all_metrics`、`rule_speedups`、`pm_series`、`warp_state_timeline`、`inline_chains` |
| [helpers/analyze_reports.py](../helpers/analyze_reports.py) | `collect` 导出指标与规则，`compare` 并列比较 |
| [helpers/extract_stall_hotspots.py](../helpers/extract_stall_hotspots.py) | 17 类 PC 状态样本、源码聚合、`inst_executed`、内联归因 |
| [helpers/plot_timeline.py](../helpers/plot_timeline.py) | 默认序列匹配、replay 分组、ASCII 图、warp 状态时间桶 |
| [helpers/list_flashinfer_workloads.py](../helpers/list_flashinfer_workloads.py) | workload 定义、shape 分布、轴值筛选、真实数据路径 |
| [helpers/harness_template.cu](../helpers/harness_template.cu) | 输入准备和 launch 骨架；需要填写 TODO |
| [helpers/safetensors_loader.h](../helpers/safetensors_loader.h) | 真实 tensor 字节读取及限制 |

**5. 已查阅的 NVIDIA 官方资料入口**

这些链接用于本对话中的时间线、依赖与指标释义核对，不意味着本次索引操作重新验证了在线文档。在线版本可能变化。

| 来源 | 用途 |
|---|---|
| [Nsight Compute Profiling Guide](https://docs.nvidia.com/nsight-compute/ProfilingGuide/) | 指标、采样、重放、管线、warp 状态定义 |
| [Nsight Compute UI Guide](https://docs.nvidia.com/nsight-compute/NsightCompute/) | PM timeline、Function Stats、Source/SASS、寄存器和 scoreboard 依赖 |
| [Compute Triage Guide](https://docs.nvidia.com/nsight-compute/ComputeTriage/) | 架构相关的管线分析，Tensor 指标解释 |
| [Nsight Systems User Guide](https://docs.nvidia.com/nsight-systems/UserGuide/index.html) | kernel、stream、CUDA memory operation、GPU metrics 时间线 |
| [NVBit 官方研究仓库](https://github.com/NVlabs/NVBit) | 动态 SASS 插桩、指令/访存 tracer 能力和开销边界 |

Ascend/msopprof/npusim 的具体源码链接与 commit 固定引用集中在 R03 文末，优先沿用该报告的版本限定，避免把当前最新版混入历史判断。

**6. 版本基线**

- 原 skill 调研基线：`c7980a43cb88a8734d113b22b57ff66746cf5719`。
- 本次建立索引前 HEAD：`5749757f68a59c81d04573a6bbc8fc7ce72b9db8`，提交标题 `[WIP] investigate`，日期 2026-10-29；3 份报告均已被 Git 跟踪。
- R03 记录的 msopprof 基线：`a52c8694979016be703006ec242901519bef31d8`。
- R03 记录的 npusim 基线：`f2baa33edb594b51b97e528eb749a07fa9ae02ea`。
- 本对话实际查询到的本机 NCU：2026.1.1.0；不能把仓库文档中的 2026.3.1 当作本机已安装版本。
