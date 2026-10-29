**ncu-report-skill 仓库阅读总结**

阅读日期：2026-10-28。对应提交：`c7980a43cb88a8734d113b22b57ff66746cf5719`。

本文基于当前仓库的 README、SKILL、全部 10 份 reference 文档、全部 helpers 实现和 Blackwell 编程参考。涉及工具版本、架构适配的描述按仓库声明整理，未独立验证其硬件参数或所有版本的兼容性；本次未实际采集 GPU 性能报告。

**1. 仓库主要做什么**

这是一个面向 CUDA kernel 性能诊断的 **Agent Skill + 分析脚本 + 优化知识库**。核心目标是把 Nsight Compute（NCU）的原始采样报告，转化为可复核、能定位到源码、按影响排序的优化计划。

其主线是：

```text
确定真实工作负载与 kernel 分支
    → 准备可分析的运行目标
    → NCU 采集原始报告
    → 脚本提取指标、规则、源码热点、时间线
    → 按六个维度诊断
    → 匹配瓶颈模式与优化方向
    → 输出带证据和复现步骤的 REPORT.md
```

README 将其定位为 Claude Code skill，并说明它是 Kernel Design Agents（KDA）的独立子模块，采用 MIT 许可证。仓库声明主要针对 Blackwell B200/B300，围绕 Nsight Compute 2026.3.1 编写，也考虑 2026.1.1 的差异。手写 CUDA、Triton、CUTLASS/CuTe 都是其预期分析对象。

**自动化边界：脚本负责数据整理；模型或工程师负责因果判断和优化计划。** 当前没有自动改写 kernel、搜索最优参数、验证数值正确性或自动完成优化闭环的程序。

来源：[README.md](README.md)、[SKILL.md](SKILL.md)。

**2. 仓库结构与各层职责**

| 层次 | 文件/目录 | 主要作用 |
|---|---|---|
| 技能入口 | `SKILL.md` | 定义触发场景、基本原则、工作顺序和阅读索引，要求先测量、再诊断、最后规划优化 |
| 操作流程 | `reference/00`—`04` | 实验目录、端到端步骤、harness 构造、采集命令、Python API 使用方法 |
| 分析方法 | `reference/05`—`06` | 六个分析维度与 15 类“现象 → 原因 → 修复方向” |
| 交付与兼容性 | `reference/07`—`09` | 报告模板、指标名称差异、权限/源码/JIT/采样等常见问题 |
| 可执行工具 | `helpers/` | 5 个 Python 文件、CUDA harness 模板、C++ safetensors 读取头文件及说明 |
| 编程知识 | `blackwell-cuda-programming.md` | Blackwell 特性、16 条通用性能准则、不同 kernel 类型的例外和检查清单 |

它采用按需展开的组织方式：入口文件指导下一步，reference 提供细节，helpers 完成重复的数据处理工作。既可以通过 Agent 使用，也可以直接运行 Python 工具分析已有报告。

**3. 一次分析如何进行**

| 步骤 | 具体做法 | 目的 |
|---|---|---|
| 建立独立实验目录 | 每次实验保存自己的 harness、原始报告、分析结果、最终报告 | 避免不同 kernel 版本、输入和采集结果混淆 |
| 明确分析对象 | 确认 kernel 名称、模板实例、输入 shape、实际 dispatch 分支、基线和问题 | 保证测到的是用户真正关心的运行路径 |
| 选择工作负载 | 优先覆盖真实热路径、典型大输入、小 grid 场景、负载最不均衡场景 | 防止均匀随机输入掩盖长尾问题 |
| 准备运行目标 | 已有 line info 就运行原程序或小 driver；否则构造独立 harness，并用 `-lineinfo` 编译 | 建立机器指令与源码行之间的映射 |
| 采集 | 首次通常使用 `--set full`，筛选 kernel 和 launch，记录时钟与版本 | 一份报告供多个分析工具复用 |
| 提取与解释 | 运行三个报告脚本，再结合六个维度和诊断手册判断 | 将大量计数器组织成可解释证据 |
| 交付 | 写入配置、关键数值、瓶颈、3—5 项优化优先级、置信度和复现命令 | 让建议能够被检查、实施和再次测量 |

仓库的典型采集命令如下，实际选项需以所装 NCU 版本为准：

```bash
ncu --set full --import-source yes --clock-control boost \
    -k "regex:KERNEL" -c 1 \
    -o "$PROFILE_RUN_DIR/reports/full_<tag>.ncu-rep" \
    <command> [args]
```

`--set full` 用于收集整体指标、SourceCounters 和 PM Sampling；`--import-source yes` 尝试把源码嵌入报告；`-k` 与 `-c 1` 限定采集对象。**一次采集命令仍可能重放 kernel 多次**，不能把它理解成一次物理执行。

对于有 JIT、autotuning 或预热的程序，文档提供 profiler start/stop 范围或跳过前几次 launch 的方式。已有报告则可以直接离线分析；只有实际采集及目标程序执行需要 GPU，Python 解析不需要。

来源：[工作流](reference/01-workflow.md)、[harness 指南](reference/02-harness-guide.md)、[采集方法](reference/03-collection.md)。

**4. 辅助脚本具体实现了什么**

| 文件 | 输入与实现 | 输出/用途 |
|---|---|---|
| `ncu_utils.py` | 自动寻找 NCU 自带的 `ncu_report`；遍历 report → range → action；兼容指标缺失及不同值类型 | 共享基础层；包含 108 项精选指标、规则解析、PM 序列和内联链处理 |
| `analyze_reports.py` | 读取一个或多个报告，逐 kernel 提取汇总值、精选指标和 NCU 自带规则 | 指标 JSON/TXT、按预估收益排序的规则文本、多对象并列表格 |
| `extract_stall_hotspots.py` | 根据 correlation ID 获取 PC，将不同 warp 状态样本汇总到源码行；另统计 `inst_executed` | 热点行排名、各状态的热点行、执行指令数排名 |
| `plot_timeline.py` | 提取 PM 时间序列、裁剪活跃区间、分桶绘图；对 timed warp samples 按时间统计原因占比 | ASCII 时间线，辅助观察长尾、利用率低谷和流水线气泡 |
| `list_flashinfer_workloads.py` | 读取 flashinfer-trace 的定义 JSON 和工作负载 JSONL，筛选轴值、统计 shape、查 UUID | 挑选代表输入，定位 safetensors 文件与标量参数 |
| `harness_template.cu` | 提供参数解析、随机填充、设备分配和 CUDA 错误检查等骨架 | 用户仍须填入 kernel、shape、输入输出分配和实际 launch |
| `safetensors_loader.h` | 用轻量 JSON 扫描器读取 header、shape、dtype、数据偏移，再加载 tensor payload | 独立 C++ harness 可读取真实输入字节，无需引入 Python 框架 |

三个报告脚本接受 `--run-dir`、可重复的 `--report`/`--tag`，输出到 `analysis/`。它们会处理报告中的多个 kernel launch，也支持 `--kernel REGEX` 过滤；不是只分析报告中的第一个 kernel。

实现中比较有价值的细节有三处：

- **内联代码归因。** NCU 默认给出的源码位置常是 CUTLASS/CuTe 的最内层头文件。传入 `--binary` 后，工具使用 `cuobjdump` 提取 cubin、`nvdisasm -gi` 读取内联链，再将样本聚合到最外层的 kernel 源码行，同时保留原始归因结果。它还比较采样 PC 对应的 SASS 主操作码，提示二进制是否可能不匹配；这不是完整二进制一致性校验。
- **时间线区分 replay。** 工具按时间戳范围和样本数分组绘制 PM 序列，在 API 支持时调用 `aligned_metric` 对齐。不同 replay 的序列不能逐点比较，只能谨慎比较形状；warp 状态图也使用自己的时间轴。
- **复用 NCU 规则。** 工具读取 `action.rule_results()`，保留规则来源、严重程度、说明及 local/global 收益范围，按数值排序。仓库没有在 Python 中重新实现一套自动瓶颈判定引擎。

来源：[helpers 说明](helpers/README.md)、[公共实现](helpers/ncu_utils.py)、[热点分析](helpers/extract_stall_hotspots.py)、[时间线实现](helpers/plot_timeline.py)。

**5. 六个分析维度分别回答什么问题**

| 维度 | 核心问题 | 主要证据 |
|---|---|---|
| SM 占用率与 launch 配置 | 工作量够不够填满 GPU？驻留数量受什么限制？ | grid/block、waves/SM、寄存器、shared memory、理论/实际 occupancy |
| Thread-block 负载均衡 | 是否少数长任务拖住整个 kernel？ | 输入工作量分布、各 SM 活跃周期差异、PM 长尾 |
| 指令停顿与源码热点 | warp 在等什么？等待集中在哪些源码位置？ | `long_scoreboard`、`short_scoreboard`、`wait`、`barrier` 等及逐 PC 样本 |
| Tensor Core 使用 | 矩阵计算是否使用 Tensor Core？利用率为什么不足？ | Tensor pipe、MMA 指令、Blackwell UTCMMA/TMEM 相关指标 |
| SM 利用率时间线 | 性能是全程低，还是某些阶段低？ | 活跃度、发射率、Tensor/内存吞吐、warp 状态随时间的变化 |
| 访存与缓存效率 | 是带宽、延迟、访问布局还是数据搬运量问题？ | DRAM 读写吞吐、L1/L2 命中、sectors/request、store 效率、local memory 指令 |

该方法的重点是交叉验证。例如，低 DRAM 吞吐结合大量 `long_scoreboard`，可以引导分析访存依赖和延迟隐藏问题；仍需结合 cache、指令和代码判断具体层级与原因，不能仅靠这两个数值断言问题一定在 L1。

源码样本也不是直接的耗时账单：采样记录的是 warp 等待发射的指令位置，真正产生依赖的指令可能在它之前。对于 producer/consumer 分工的 kernel，设计上就需要等待的 warp 不一定处在关键路径上。

来源：[六个分析维度](reference/05-analysis-dimensions.md)。

**6. 如何从证据得到优化方向**

`reference/06-diagnosis-playbook.md` 包含 A—O 共 15 类模式。每类组织信号、原因、优先尝试的修复、深入改造方向，并在适用处说明例外。

| 模式 | 文档给出的主要优化方向 |
|---|---|
| A：grid 太小 | 拆分更多可并行维度，例如 split-K |
| B：不均衡与长尾 | 按长度组织工作、拆分长任务、chunk 化、分类 dispatch |
| C：global load 不合并 | 调整线程与数据映射、改变布局、共享内存转置 |
| D：写入稀疏 | 聚合输出、连续写入、适当向量化 |
| E：访存延迟主导 | 增加独立在途请求、预取、提高可用并行度 |
| F：矩阵计算走标量路径 | 考虑适配 MMA/Tensor Core 或现成库 |
| G：atomic 竞争 | warp/block 分层归约、分桶再合并 |
| H：shared memory bank conflict | 根据逐指令证据调整 padding、swizzle 和布局 |
| I：同步开销 | 缩小同步范围、减少屏障、调整 warp 分工 |
| J：理论与实际 occupancy 差距 | 联合排查负载不均、执行阶段和其他等待现象 |
| K：寄存器溢出 | 调整活跃变量、资源预算、数据存放或 kernel 拆分 |
| L：非预期 FP64 | 检查字面量与数学运算的精度选择 |
| M：计算与访存不重叠 | 双缓冲、异步搬运、多级流水线 |
| N：warp divergence | 调整数据分组、分支粒度或低成本条件计算 |
| O：串行依赖链 | 引入独立工作、减少关键路径上的状态搬运或串行步数 |

建议最终只保留最重要的 3—5 项，逐项给出指标值、源码位置、预期影响、适用工作负载和实施成本。NCU 的 `Est. Speedup` 是线索，需要先核对；不同规则可能重叠，local/global 的范围也不同，不能直接相加当作总收益。

来源：[诊断手册](reference/06-diagnosis-playbook.md)、[报告模板](reference/07-report-template.md)、[解释注意事项](reference/09-common-issues.md)。

**7. Blackwell 编程文档起什么作用**

`blackwell-cuda-programming.md` 是配套设计参考，覆盖 `tcgen05`、TMEM、2CTA、TMA 相关流水线、低精度与 block scaling、cluster/DSMEM 等主题，以及并行度、合并访存、共享内存、寄存器、精度、Tensor Core、atomic、同步等 16 条准则。

它还按 decode attention、element-wise、reduction、GEMM、gather/scatter、通信等类型说明哪些现象可能是合理取舍。例如，逐元素运算没有数据复用时不必强行加入 shared memory；非矩阵运算没有 Tensor Core 活动也不自动构成问题。

它服务于“知道瓶颈后如何重新设计 kernel”，主技能服务于“先确认现有 kernel 的瓶颈”。其中的 PTX 和流水线代码是说明材料，仓库没有提供相应完整、高性能 kernel 实现或实测结果集。

**8. 最终产物是什么**

仓库要求一次实验保留完整的证据目录：

```text
<profile_root>/<run_name>/
├── REPORT.md                    # 模型或工程师撰写的结论、优化优先级与复现方法
├── harness/                     # 当次 driver、源码、二进制、编译信息
├── reports/
│   └── full_<tag>.ncu-rep        # 原始 NCU 报告，也支持 .ncu-repz
└── analysis/
    ├── metrics_all_<tag>.json    # 全部指标的汇总值及可获得的单位
    ├── metrics_key_<tag>.json
    ├── metrics_key_<tag>.txt     # 108 项精选指标，未采集的项可能为空
    ├── rules_<tag>.txt          # NCU 规则结果
    ├── compare_*.txt            # 多个对象的关键指标并列比较
    ├── stall_hotspots_<tag>.txt # 源码热点与执行指令数
    └── pm_timeline_<tag>.txt    # ASCII 时间线
```

`REPORT.md` 包含采集配置、关键数值表、六维分析、综合诊断、排序后的优化方向、置信度及复现步骤。优化前后可以创建独立的比较目录，读取两边已有报告，无需为了比较再次采集。

来源：[目录约定](reference/00-directory-layout.md)、[最终报告模板](reference/07-report-template.md)。

**9. 阅读实现后需要明确的边界**

- **知识库比自动诊断程序更完整。** 六维分析、15 类模式和报告模板主要是文档。脚本不自动计算所有衍生量，也不自动选择修复方案；atomics、分支或 bank conflict 的深入分析可能需要继续读取原始报告、追加指标或编写一次性脚本。
- **汇总 JSON 不是原始报告的完整替代。** `dump_all_metrics()` 保存 `m.value()` 与单位，没有导出每个实例的样本、correlation IDs、源码内容和完整时间线；深入复核仍应保留 `.ncu-rep/.ncu-repz`。
- **热点总数不能等同于浪费时间。** 热点脚本的汇总列表包含 `selected` 和 `not_selected`；它们分别涉及成功发射、就绪但未被选中。排名需结合状态明细、执行指令数和关键路径解释，不能直接当成“可消除停顿”排名。
- **真实输入和二进制仍要由使用者保证。** harness 中有大量 `TODO(you)`，默认 launch 为空；safetensors 读取器整文件加载，不支持分片，dtype 需要调用者检查。FlashInfer 工具帮助浏览数据，不能自动理解项目全部 dispatch 逻辑。
- **兼容性有明确前提。** `ncu_report` 来自 NCU 安装；源码归因要求 line info，内联链还需要匹配二进制和反汇编工具。指标缺失会显示为空，不能当成零。短 kernel 的 PM 样本可能不足，旧版本和其他 GPU 的指标/API 需要检查。
- **参考材料存在简化和不一致。** 例如 Blackwell 文档一处写寄存器容量为 `64K × 32-bit`，另一处表格写成 `64 KB`；旧的 coalescing 说明固定以 4 sectors/request 为理想值，而较新的分析文档按每线程 4/8/16 字节访问区分为 4/8/16。使用时应核对具体指令和工具指标，不能机械套用阈值或示例。
- **单 kernel 诊断不能代表全部应用性能。** 配套文档明确把 Host–Device 同步、kernel 之间的空档等问题交给 Nsight Systems 等系统级工具；最终收益也需要独立正确性验证和真实 benchmark 确认。

当前跟踪文件中没有测试套件、CI 配置、实际 NCU 报告或完整优化案例。本次已对 5 个 Python 文件执行 AST 语法检查，均通过；这只验证语法，不代表完成 NCU API、报告解析或 GPU 运行验证。
