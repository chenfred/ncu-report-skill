**ncu-report-skill 迁移 Ascend 950：msopprof 与 npusim 能力并集调研**

调研日期：2026-10-28。范围：性能分析流程迁移可行性、数据与瓶颈覆盖、NCU 对照及实施建议；未实施 skill 移植，未在 Ascend 950 真机上运行采集。

本次由三个 subagent 分别调查 msopprof、npusim、迁移语义与 NCU 对照，再交叉核对关键源码。GitCode 网页读取受限时，msopprof 使用官方 Ascend GitHub 镜像；npusim 从用户给出的 GitCode 仓库直接克隆读取。

| 对象 | 本次检查版本 |
|---|---|
| 当前 ncu-report-skill | `c7980a43cb88a8734d113b22b57ff66746cf5719` |
| Ascend/msopprof | `a52c8694979016be703006ec242901519bef31d8`，2026-10-21 |
| cann/npu-simulator | `f2baa33edb594b51b97e528eb749a07fa9ae02ea`，2026-10-27 |
| NCU 比较依据 | 调研当日 NVIDIA 在线 Profiling Guide、UI、Python Report Interface、NvRules 文档 |

**1. 核心判断**

取 **msopprof 真机 + msopprof simulator + npusim CA 仿真** 的能力并集，可以覆盖原 skill 的主要诊断目标，并增加更直接的 NPU 指令执行区间、同步事件和软件流水分析。推荐保留原来的“测量 → 诊断 → 优化计划 → 验证”工作方法，重写采集适配层、执行模型、指标定义和诊断规则。

这不是只把 `ncu` 替换为 `msopprof`：CUDA 的 warp occupancy、scoreboard 和 sectors/request 阈值，不能直接对应 NPU 的核间负载、Cube/Vector 执行和 GM/L1/L0/UB 搬运。950 还有 SIMT 路径，应将 Cube、SIMD/VF、SIMT 分开分析。

两个重要补充：

- 950 的 msopprof **真机模式已经支持 PipeTimeline、InstrTimeline 和 SIMT 相关分析**，流水可视化不只存在于仿真模式。[真机指南][m1]
- npusim 能提供模型中的指令起止时间，但其上层指南与当前开源报告后端存在差异。不能把文档里每一种轨道都当成安装版本必然输出的事实。[npusim 指南][n1]、[当前后端说明][n4]

**2. 能力并集：能采什么、能诊断什么**

下表的“可诊断”表示具备相关证据，最终原因仍需结合源码、输入和验证实验判断。

| 分析目标 | 主要来源 | 可获得的信息 | 能支持的诊断 |
|---|---|---|---|
| 总耗时、执行配置 | msopprof 真机 | Task Duration、block/mix block 配置、频率、每核/子核时间与 cycles | 性能回归、切分不当、启动/执行时间差异 |
| 计算单元利用 | msopprof ArithmeticUtilization/PipeUtilization | Cube、Vector、VF、SFU、SIMT VF、Scalar 的周期占比和部分指令计数 | 计算侧主导、算力闲置、标量控制开销、执行模式选择 |
| 搬运路径 | msopprof Memory/MemoryL0/MemoryUB | GM、L1、L0A/B/C、UB 间的搬运量、带宽、通路占比 | 复用不足、重复搬运、特定通路受限、tile 不合适 |
| 缓存 | msopprof L2Cache | 950 的 close/far、read/write hit/miss/victim 及命中率 | 局部性、驱逐与缓存压力 |
| 等待与冲突 | msopprof ResourceConflictRatio | Cube/MTE/Vector 等待；950 Vector 的 STU/LDU/SFU 冲突指标 | 资源争用、计算等搬运、执行单元冲突 |
| 核间负载 | msopprof per-core CSV/Occupancy；npusim 多核汇总 | 各核/子核时间、吞吐、起止跨度、最慢核差异 | 长尾、工作切分不均、不同角色失衡 |
| 真机流水 | msopprof PipeTimeline/InstrTimeline | 指定计算/搬运 pipe 的实际活动和采样指令区间；可有 PC/调用栈 | 部分实际计算/搬运重叠、气泡、阶段耗时 |
| 仿真流水与同步 | msopprof simulator；npusim CA | core/subcore/pipe、指令起止、部分 flag/buffer 同步连线 | 预取距离、双缓冲是否有效、等待链、首尾阶段开销 |
| 源码/汇编归因 | msopprof Source、simulator 热点 | PC、源码/调用栈、指令、执行次数、部分 GPR 状态/搬运字节 | 哪段代码产生等待、哪段负责冗余搬运或寄存器压力 |
| SIMT 调度问题 | msopprof PcSampling/Warp 流水 | scoreboard、同步、寄存器 bank、资源、指令缓冲等 stall；warp entry 耗时 | SIMT 依赖、发射受阻、分支/同步问题 |
| VF 执行效率 | npusim VF/summary，受实际后端限制 | exec/ldst/scalar 或 branch IPC、调用数、执行波动、部分 VF 指令时间线 | VF 内计算与访存配比、依赖链、低执行效率 |
| 性能上限与提示 | msopprof Roofline、Occupancy 建议；npusim 派生汇总 | 存储/计算上限模型、核间均衡提示、pipe/overlap 指标 | 判断优先优化计算、数据搬运还是任务分布 |

msopprof 950 具体字段见 [性能数据说明][m2]；源码、流水和扩展模式见 [真机指南][m1]。仿真输出结构见 [simulator 输出说明][m4]；npusim 的 VF 与汇总见 [VF 指南][n3]和[当前后端说明][n4]。

字段不是一律真机测量：例如 simulator 的 cycles 是模型结果，npusim 的 bandwidth 是指令字节和仿真时间推导，msopprof 某些模式会混合真机与模拟结果。每项观察都应保存证据来源。

**3. 三个采集入口如何分工**

| 入口 | 最适合承担的工作 | 主要输出 |
|---|---|---|
| msopprof 真机 | 确认目标硬件上的瓶颈、核间表现、真实收益；补充采样流水/SIMT stall | 各指标 CSV、`trace.json`、`visualize_data.bin` |
| msopprof simulator | 展开每核指令、源码行、调用栈、同步关系 | `*_code_exe.csv`、`*_instr_exe.csv`、每核和汇总 `trace.json`、可视化文件 |
| npusim CA | 在无 NPU 时运行模型，分析执行区间、同步 flow、VF、pipe 利用和重叠 | DB、trace JSON、`summary.json`、`aicore_utilization.json`、可选 VF JSON/归档 |

输入仍然需要可运行 app/driver、编译好的 kernel、实际 shape/dtype/布局以及有关数据。工具不会自动发现生产热点；选 workload 的职责继续由 skill/Agent 通过 benchmark、调用代码和数据集完成。npusim 支持 kernel 直调；PyTorch 场景被官方标为 best-effort。[npusim 使用场景][n2]

npusim 默认 CA 用于时序仿真；`--pv` 只有功能仿真，没有可用于上述性能比较的精确时序。官方说明 kernel 与板上二进制兼容，但支持范围、模型库、执行路径仍需匹配目标 SoC。[npusim README][n0]、[用户指南][n1]

两条仿真入口都依赖 CANN 提供的模拟环境。npusim 当前代码选择 CAModel 库并编排执行、处理报告；不能据此声称它与 msopprof simulator 所有模式具有完全相同的模型、采集格式和性能结论，也不能把它们视为可互换的 CLI。[npusim 模型路径][n12]、[msopprof simulator 指南][m3]

**4. 950 真机流水的实际边界**

msopprof 提供比当前 ncu-report-skill 更直接的执行通路时序，但其官方限制必须保留在新 skill 中：

- `PipeTimeline` 与 `InstrTimeline` 基于采样；全部核开启也只展示 **6 个核**。
- `InstrTimeline` 支持 cube/fixp/vector/mte1/mte2/mte3，每 pipe 最多 **1024 个 PC**。
- 指令密集时可能丢数据；SIMD VF、SIMT VF 内部指令在该真机指令视图中不可见。
- `PipeTimeline` 与 `InstrTimeline` 不能同时使能；range replay 也与若干深入分析模式互斥。
- PC/调用栈关联需要按文档编译 `-g`。这属于 Ascend 编译流程要求，不应照搬 CUDA 对 `-G` 的解释。
- SIMT Warp 流水通过 entry 首尾插桩获得，独立重放，会影响耗时；官方不建议与其他类别数据比较。

因此可以说“真机支持受覆盖限制的指令时序”，不能说“全芯片所有指令的完整无损硬件 trace”。[真机流水指南][m1]

950 的 PcSampling 提供 `Scoreboard_Not_Ready`、`Warp_Level_Sync`、`Register_bank_conflict`、`Resource_conflict`、`IBuf_Empty`、`Divergence_Stack_Spill` 等分类。应建立 NPU 自己的 stall 解释，`active` 不能作为浪费时间。[源码与 Stall 支持表][m1]

**5. 相对 NCU 的优势与限制**

| 比较项 | msopprof + npusim 并集 | NCU 与现有 skill | 判断 |
|---|---|---|---|
| 无硬件开展工作 | CA/PV 可在 CPU 环境执行模型 | 报告可离线分析，实际采集要 GPU | NPU 组合能在无设备时推进模型内验证和时序分析 |
| 核内指令执行区间 | 仿真有逐指令时序；950 真机有受限采样流水 | 普通报告以计数、PC/warp 状态采样、PM 序列为主 | NPU 组合更直接支持显式流水的区间分析 |
| 同步和 buffer 分析 | 部分 set/wait、释放/获取事件有 flow | NCU 有寄存器/scoreboard 依赖与 stall 归因 | NPU 的事件材料更贴近 Ascend 软件流水改造；不是普遍精度更高 |
| 真实硬件依据 | 真机计数器可用于缓存、搬运、计算等；仿真补充解释 | 指标和采样主要以真实 GPU 执行为基础 | 两侧均应以独立 benchmark 验证收益 |
| 规则与 Agent 接口 | CSV/JSON/SQLite 易处理，但格式和语义分散 | `ncu_report`、NvRules、规则收益估计接口较集中 | NPU 移植需要新的标准化层和规则库 |
| 架构相关性 | 可见 Cube、Vector、MTE、FIXP、VF、SIMT 等 | 可见 warp scheduler、计算 pipe、cache 等 | 关注的问题可迁移，硬件模型与阈值不能逐项改名 |
| 数据完整性 | 真机采样覆盖有限；模型与后端也可能缺事件 | PM/PC 是采样，多 pass 可能错位 | 图形更细不等于完整，也不等于更真实 |
| 版本一致性 | 本次发现明确的中英文/实现差异及占位字段 | 也有架构和版本差异，但现有 skill 已有适配层 | NPU 端必须探测实际能力，不能只照文档生成命令 |

NCU 并非只有平均指标：官方提供 PM 时间线、带时间的 warp 样本、源码寄存器与 scoreboard 依赖、规则系统；普通源码 PC 采样则不等于逐指令的发射/完成日志。比较应以这些实际能力为基线。[NCU 采样与重放][v1]、[NCU UI 与依赖分析][v2]

NCU 的报告 Python API 和 NvRules 可以直接访问大量分析结果；本次没有确认到 NPU 组合中与它完全等价的统一规则结果 API及逐条可比较的收益估计。已有 Roofline/负载建议仍然有价值。[NCU Python API][v3]、[NvRules API][v4]

双方 profiler 都可能通过重放、缓存处理、采样或插桩改变条件。msopprof 文档说明 kernel/range replay 清理 L2，而 application replay 不清理；NCU 也有缓存与重放控制。因此不能只质疑仿真，却把所有 profiler 数值都当成无扰动应用真值。[msopprof 重放说明][m1]、[NCU 可复现性][v1]

**6. 新的诊断手册应该如何写**

以下是迁移设计建议，不是工具已经实现的自动结论。

| 性能现象 | 需要的证据 | 候选改造方向 |
|---|---|---|
| Cube 不忙，MTE 活跃或等待占比高 | 真机路径带宽/流量 + CA 中 load→compute 依赖 | 改 tiling、增加复用、提前搬运、double/multi-buffer；先辨别带宽还是延迟 |
| 计算与搬运串行 | 同一时间基准下的 pipe 区间、set/wait/buffer flow | 调整预取距离、缓冲生命周期、同步位置和软件流水 |
| Vector/VF 执行效率低 | VF exec/ldst/scalar/branch IPC、指令组成、调用时间和次数 | 增加独立工作、改数据流、减少冗余 load/store、批量处理；不能只追高 IPC |
| Scalar 占比高 | 真机 scalar 指标、源码/指令热点、模型时间线 | 简化索引和控制逻辑、减少重复配置、适当展开；防止代码膨胀/取指退化 |
| 等待 buffer/flag 的时间长 | 对应 producer/consumer 的事件和执行区间 | 调整事件位置、缓冲数、AIC/AIV 分工；同步必须保持正确性 |
| 核间长尾 | 全核逐核时间、工作量、最慢核与尾块输入 | 均衡切分、重新组织任务、修正边界 tile 或特殊核负担 |
| 缓存效率差或搬运过多 | 真机 L2、GM/L1/L0/UB 路径流量、具体 load/store | 改数据布局和访问顺序、提升 tile 复用；用目标场景确认缓存变化 |
| SIMT 发射受阻 | 真机 PcSampling + GPR/源码/warp 材料 | 按950 stall分类处理依赖、寄存器bank、同步、分支和指令供给 |
| 重叠率提高但端到端没改善 | 真机总耗时 + 各阶段关键路径、额外搬运/同步成本 | 判断改动是否只是移动开销，或优化了非关键路径 |

原六维可以保留其提问方式，但应调整为：

1. **核映射、切分与资源配置**：取代对所有 NPU 算子统一套用 GPU warp occupancy。
2. **核间/子核负载与长尾**。
3. **等待、依赖和源码热点**：SIMD/Cube 与 SIMT 分支处理。
4. **Cube/Vector/VF 计算效率**。
5. **计算、搬运、同步时间线**。
6. **存储层次、数据复用与搬运效率**。

建议增加显式输出：**调度与同步关键路径**。不能仅报告“某 pipe 利用率低”，要解释“哪个阶段等哪个 producer、哪个 buffer 复用点限制重叠、修改后预计缩短哪一段”。没有完整依赖事实时应称“候选关键路径”，并用实验确认。

**7. 只看 core0：可以作为深挖入口，不能替代全核检查**

core0 有代表性取决于工作分配，而不取决于核心数量少。

在各核同构、tiling/迭代数接近、无特殊角色，并已确认全核耗时分布相近时，core0 适合检查稳态软件流水。变长、MoE、稀疏、尾块、边界 mask、分阶段归约、通信等场景不应默认满足此前提。

建议策略：

```text
全核轻量概览
    → 典型核（可以是 core0）
    → 最慢核
    → 尾块/特殊角色核（与前面重复时合并）
    → 分别检查相应 AIC / AIV0 / AIV1
```

“只记录 core0”和“保留全核记录、仅显示 core0”是不同的采集范围。msopprof simulator 的 core 选择文档本身以负载均匀为适用前提；npusim 也提示单核日志无法暴露跨核问题。[msopprof simulator 指南][m3]、[npusim 指南][n1]

另外，950 的 SIMT 路径仍涉及 warp，不能因改用 NPU 而删除全部 warp 层分析。

**8. 已核实的版本差异与指标口径**

**8.1 msopprof：950 simulator 不支持所有总览能力**

- 当前中文文档和实现都表明：950 simulator 不支持 `PMSampling`、`Overhead`。英文页面的相反表述不能采用。CLI 帮助实现、支持矩阵和针对 950 的单元测试能相互印证。[支持矩阵][m5]、[CLI 实现][m6]、[单元测试][m7]
- 950 simulator 热点表支持源码、PC、pipe、cycles/次数，但不支持其表中的 GPR Count、UB Bank Conflict、Vector Unit Utilization、Process Bytes。950 真机 Source 则可提供其中一些数据，两者应组合使用。[simulator 指南][m3]、[真机指南][m1]
- 950 不支持源码行/指令级模拟 L2 命中率列；真机核级 L2 计数仍可用。不要将“缺少逐行值”解释为“没有缓存分析”。
- simulator 的 `--timeout` 会留下运行前缀；截断数据不能证明完整 kernel 没有尾部问题。

**8.2 npusim：当前报告后端与上层指南漂移**

当前源码优先尝试 BiProfRunner，然后退回 `prof_demo`；安装的 CANN 是否提供另一个完整后端需要现场确定。[后端选择实现][n5]

本次直接检查的 `prof_demo` 支持 `KernelInfo` / `ExecutedInstructions` 两表合同的特定版本。其 README 明确：

- 没有 cache/dispatch/region/debug/warp 事件生产者，不能承诺对应丰富泳道。
- cache 的 `0/null` 和某些 VF 零值是兼容占位，不代表测得零。
- SIMT 叶指令展示缺失，保留 `VF_SIMT` 调用，不提供其内部完整 warp 流水。
- VF JSON 列表与 timeline 覆盖不同：timeline 只包含 `VF`，不包含 `VFI/VF_SIMT` 内部轨迹。
- 没有旧版本的一些 HTML/SVG、warp utilization 等报告。

这些是 **当前特定后端限制**，不能推导为 msopprof + npusim 并集完全缺少相关能力；SIMT stall、核级缓存等可从 msopprof 真机补充。但两条路线均缺少的精细事件，仍需标记不可观测。[当前后端说明][n4]

**8.3 相同名字不保证相同统计意义**

| 字段 | 本次核实的意义 | 不能作的解释 |
|---|---|---|
| msopprof Occupancy | 核间耗时/吞吐/cache 的均衡分析 | CUDA resident warp occupancy |
| npusim aicore_utilization | 子核首条到末条指令的跨度 / DB 全局跨度 | 持续忙碌率；中间可能有大量空泡 |
| npusim pipe utilization | 非等待执行区间的并集 / kernel 总周期，按选中核统计 | 单条指令耗时简单求和，或活跃周期归一化指标 |
| npusim imbalance_ratio | 当前实现是各核执行跨度 max/min | 文档措辞中的一般 pipe 不均衡 |
| npusim pipeline_overlap | 逐核两pipe区间交集之和 / 并集之和 | 端到端收益、相对kernel总时间的隐藏百分比 |
| npusim bandwidth.total_gbps | 解码字节 / 活跃核数 / kernel时间 / 1024³ | 整芯片实测GB/s；当前数值实为每活跃核平均GiB/s口径 |
| source行 cycles / pipe time | 可跨并发操作累加的统计 | 可直接相加得到kernel延迟 |

`pipeline_overlap` 低也不必然代表隐藏失败：如果短搬运完全包含在很长的计算区间内，交并比仍可能低。迁移时应同时计算相对较短阶段的覆盖率及相对 kernel 的 exposed time，并明确这是新增派生分析。

npusim 带宽利用率还使用固定峰值常量，不能未经目标 SoC 核对就用于断言片外内存已饱和。[pipe 实现][n6]、[summary/overlap 实现][n7]、[bandwidth 实现][n8]、[occupancy 实现][n9]

**8.4 核范围与时间基准必须单独保存**

当前 npusim `report --core-id` 不是全局过滤器：trace、pipe、scalar IPC 等按选择核过滤，occupancy、VF 与部分 kernel 汇总仍可能覆盖 DB 中全部记录核。`record` 的选择又影响实际日志覆盖。第一次分析宜显式选择全核，再生成代表核详情，不要依赖默认值。

各工具中的硬件计数、硬件采样、插桩重放、CA 仿真不能拼接成一张声称“真实同次执行”的时间线。计算重叠关系时，应要求相同 capture/replay、相同时钟/时间基准、明确的 core/subcore 与事件覆盖范围。

**9. 移植实现建议**

以下是拟议结构，尚未创建代码：

| 当前文件/模块 | 迁移建议 |
|---|---|
| `SKILL.md`、目录约定、报告模板 | 保留方法与可复现性约定；增加真机/仿真分工、代表核选择 |
| `ncu_utils.py` | 拆成 msopprof/npusim 适配器与共同指标模型；不沿用 CUDA 指标表 |
| `analyze_reports.py` | 解析 CSV/JSON/DB，输出全核概览、真实与模拟指标对照 |
| `extract_stall_hotspots.py` | 增加 Ascend PC/源码热点、SIMT stall 分类、VF 局部分析 |
| `plot_timeline.py` | 利用已有 trace；新增 pipe 区间、overlap、等待/同步 flow 分析，保留数据来源 |
| Blackwell/诊断参考 | 用950存储层次、Cube/Vector/VF/SIMT执行和同步机制重写 |
| 输入准备 | 替换为 Ascend kernel driver/tiling 输入，同时保留真实 workload 选择原则 |

标准化观测至少记录：

```text
source_kind = hardware_counter / hardware_sample /
              instrumented_trace / simulation_CA / simulation_PV
tool_version、CANN_version、report_backend、schema_version、exact_soc
kernel_binary_hash、shape/dtype/layout、tiling、相关输入分布/数据标识
capture_id / replay_id、clock、time_unit、time_base
executed_core_count、recorded_core_ids、analyzed_core_ids、subcore/pipe
missing / unsupported / placeholder / sampled / dropped / truncated
```

零值、未采集、占位值、不支持应区分表示。未出现的字段不能自动填零并触发“没有该瓶颈”的结论。

推荐工作流：

```text
真实输入与正确性基线
    → msopprof 真机概览、逐核检查
    → 选择典型核/最慢核/特殊角色
    → 真机流水或SIMT采样 + CA仿真补充指令/同步证据
    → Agent输出有证据的候选修改
    → 开发、正确性验证、仿真比较
    → 真实950 benchmark与必要的msopprof复测
```

没有真机时，可以先进行 CA 模型分析和功能验证，但结论应标注“模型预测，待真机确认”。本次没有发现针对所有950算子、缓存与并发场景均成立的量化仿真误差保证，不能由“精确时序仿真”名称推导绝对性能完全等于芯片实测。

**10. 调研验证范围与待现场确认事项**

- 已阅读两个官方仓库的文档，并检查相关参数支持、后端选择、指标计算实现。
- subagent 执行了 npusim 的三项官方合成 DB 报告测试，均通过，用于核实报告范围和字段语义；没有运行真实 NPU kernel 或 CA 模型。
- 实际移植开始时，需要锁定950的具体型号、CANN/msopprof/npusim/Insight版本、模型库与报告后端，并用一份真实产物验证字段和单位。
- 能力并集应通过“可用后端选择”实现，不要求每个 kernel 每轮都跑三个工具；优先真机概览，按发现的问题追加深入采集。

[m1]: https://github.com/Ascend/msopprof/blob/a52c8694979016be703006ec242901519bef31d8/docs/zh/user_guide/msopprof_user_guide.md
[m2]: https://github.com/Ascend/msopprof/blob/a52c8694979016be703006ec242901519bef31d8/docs/zh/user_guide/msopprof_performance_data.md
[m3]: https://github.com/Ascend/msopprof/blob/a52c8694979016be703006ec242901519bef31d8/docs/zh/user_guide/msopprof_simulator_user_guide.md
[m4]: https://github.com/Ascend/msopprof/blob/a52c8694979016be703006ec242901519bef31d8/docs/zh/user_guide/msopprof_simulator_performance_data.md
[m5]: https://github.com/Ascend/msopprof/blob/a52c8694979016be703006ec242901519bef31d8/csrc/op_profiling/argparser/arg_checker.cpp
[m6]: https://github.com/Ascend/msopprof/blob/a52c8694979016be703006ec242901519bef31d8/csrc/op_profiling/interface/ms_op_prof.cpp
[m7]: https://github.com/Ascend/msopprof/blob/a52c8694979016be703006ec242901519bef31d8/test/ut/op_profiling/argparser/test_arg_checker.cpp
[n0]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/README.md
[n1]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/docs/zh/user_guide/01_user_guide.md
[n2]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/docs/zh/user_guide/02_user_scene.md
[n3]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/docs/zh/user_guide/03_vf_report_guide.md
[n4]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/npusim/prof_demo/README.md
[n5]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/npusim/core/report/report_strategy_impl.py
[n6]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/npusim/prof_demo/analysis/pipes.py
[n7]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/npusim/prof_demo/reports/summary.py
[n8]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/npusim/prof_demo/analysis/bandwidth.py
[n9]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/npusim/prof_demo/reports/occupancy.py
[n12]: https://gitcode.com/cann/npu-simulator/blob/f2baa33edb594b51b97e528eb749a07fa9ae02ea/npusim/core/record/paths.py
[v1]: https://docs.nvidia.com/nsight-compute/ProfilingGuide/
[v2]: https://docs.nvidia.com/nsight-compute/NsightCompute/
[v3]: https://docs.nvidia.com/nsight-compute/PythonReportInterface/index.html
[v4]: https://docs.nvidia.com/nsight-compute/NvRulesAPI/index.html
