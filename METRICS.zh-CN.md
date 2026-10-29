**ncu-report-skill：分析类别与指标说明**

依据：当前仓库 `helpers/ncu_utils.py` 的 `KEY_METRICS`、三个报告分析脚本，以及 `reference/05`、`06`、`08`。整理日期：2026-10-28。

**范围与标记**

- **默认**：`analyze_reports.py` 从报告中尝试读取的 108 项精选指标；缺失或读取异常时返回 `None`。这不表示每种 GPU、每份报告都有这些值，也不表示脚本自动完成了诊断。
- **专项**：源码热点、PM 时间线脚本读取的实例数据、API 数据，以及规则结果。
- **按需**：诊断文档要求结合问题继续检查的指标，不在 108 项默认清单中。已采集时可以从原始报告读取，未采集时可能要追加采集。
- `metrics_all_*.json` 另外保存报告中所有指标的汇总值。因此“不在精选清单”不等于“没有导出”，但逐实例深入分析仍需要原始报告。
- 下文的 `{a,b}` 是列举多个实际指标的缩写；尖括号表示需要替换的变量，不是完整指标名。

**0. 先理解命名和计量口径**

| 组成 | 简短解释 |
|---|---|
| `launch__` | 当前 launch 的形状和资源配置 |
| `device__` | GPU 的硬件属性 |
| `gpu__` | GPU 级别指标或组合指标 |
| `sm__` / `smsp__` | SM / SM 子分区级别指标；不要把两者的每周期数值直接混用 |
| `l1tex__` / `lts__` / `dram__` | L1/TEX、L2、显存相关指标 |
| `.sum` / `.avg` / `.min` / `.max` | 对指标实例的汇总方式；通常涉及硬件单元，不应一概解释为时间轴上的总和/平均/峰值 |
| `.per_second` | 每秒速率，如 bytes/s、cycles/s |
| `.per_cycle_active` | 按相关单元活跃周期归一化的速率 |
| `.pct_of_peak_sustained_active` | 活跃周期内，相对持续峰值能力的百分比 |
| `.pct_of_peak_sustained_elapsed` | 按整个测量区间归一化，相对持续峰值能力的百分比 |
| `.pct` / `.ratio` | 百分比 / 比值；具体分母需要看该指标定义 |

例如 `sm__inst_executed_pipe_fma.avg.pct_of_peak_sustained_active` 反映 FMA 管线指令吞吐相对峰值的水平，不是“所有指令里 FMA 指令占多少”。吞吐类组合指标也不能直接解释为 GPU 的忙碌时间占比或 FLOPS 利用率。

实际名称和单位以报告的 `metric_names()`、`metric.unit()`、`metric.description()` 为准。官方定义参考：[Nsight Compute Profiling Guide](https://docs.nvidia.com/nsight-compute/ProfilingGuide/)。

**1. 时间与频率：3 项默认指标**

回答：kernel 实际运行多久？优化前后是否在可比的时钟条件下测量？

| 指标 | 介绍与用途 |
|---|---|
| `gpu__time_duration.sum` | 目标 GPU 工作的持续时间；kernel 报告中用于核心性能比较，通常是 ns，需检查单位 |
| `sm__cycles_elapsed.avg.per_second` | 由 SM 周期与时间得到的平均频率，用来检查两次采集的时钟条件 |
| `smsp__cycles_active.avg` | SMSP 活跃周期数的平均值；反映活动规模，本身不是利用率百分比 |

常用衍生量：`speedup = baseline_duration / new_duration`。该值是被测 kernel 的加速比，不自动等于应用端到端加速比。

**2. Launch 几何、资源与硬件容量：16 项默认指标**

回答：启动了多少工作？每个 block 消耗多少资源？硬件最多能容纳多少工作？

| 指标 | 介绍与用途 |
|---|---|
| `launch__grid_size` | 整个 grid 的 block 数量 |
| `launch__block_size` | 每个 block 的线程数量 |
| `launch__grid_dim_{x,y,z}` | grid 三个维度，帮助识别 batch/head/tile/split 等并行映射 |
| `launch__block_dim_x` | block 的 x 维度；默认清单没有 y、z 两项 |
| `launch__thread_count` | 本次 launch 的线程总数 |
| `launch__waves_per_multiprocessor` | 基于驻留容量估算的工作波次数，用来判断工作是否充分及是否有部分尾波；不是实际执行轨迹 |
| `launch__registers_per_thread` | 每线程寄存器数量，用来分析寄存器资源压力 |
| `launch__shared_mem_per_block` | 每 block 的共享内存资源用量，用来分析驻留限制 |
| `launch__occupancy_limit_blocks` | 受硬件 block 数上限约束时，每 SM 可驻留的 block 数上限 |
| `launch__occupancy_limit_registers` | 由寄存器容量导致的驻留 block 上限 |
| `launch__occupancy_limit_shared_mem` | 由共享内存容量导致的驻留 block 上限 |
| `launch__occupancy_limit_warps` | 由 warp/线程容量导致的驻留 block 上限 |
| `device__attribute_multiprocessor_count` | 目标设备的 SM 数量 |
| `device__attribute_max_warps_per_multiprocessor` | 每 SM 最大驻留 warp 数量 |

四个 `occupancy_limit_*` 是容量限制，不是瓶颈百分比。`waves < 1` 表示未用满估算的并发容量，不等于必然有部分 SM 完全没工作；要结合 grid 与 SM 数判断。

文档还列出 `launch__block_dim_{y,z}`、`launch__shared_mem_per_block_{static,dynamic}`，这些属于按需查看。

**3. Occupancy 与就绪 warp：8 项默认指标**

回答：有多少 warp 驻留？其中有多少能够发射下一条指令？

| 指标 | 介绍与用途 |
|---|---|
| `sm__maximum_warps_per_active_cycle_pct` | 仓库用作理论 occupancy，描述当前配置的驻留上限 |
| `sm__warps_active.avg.pct_of_peak_sustained_active` | 实际 occupancy，即活跃/驻留 warp 数相对容量的水平 |
| `sm__warps_active.{avg,max,min}.per_cycle_active` | 每活跃周期驻留 warp 数的平均、最大、最小汇总，辅助观察单元间差异 |
| `smsp__warps_active.avg.per_cycle_active` | 每 SMSP 的平均驻留 warp 数 |
| `smsp__warps_eligible.avg.per_cycle_active` | 每 SMSP 平均就绪 warp 数，即可参与指令发射竞争的 warp |
| `smsp__warps_eligible.max.per_cycle_active` | 就绪 warp 数的最大实例汇总，用于对照不同调度子分区 |

驻留、就绪和实际发射是不同状态。停顿 warp 仍可能驻留，因此不能把低 achieved occupancy 直接归因于“stall 导致驻留槽位腾空”。也不能把按 active cycles 计算的 occupancy 当成全芯片忙碌率。

典型组合：驻留 warp 多、就绪 warp 很少，说明需要继续检查依赖、同步和内存等待，而不只是继续增加线程。

**4. 指令发射与 IPC：4 项默认指标**

回答：调度器能否持续发射？执行了多少指令？

| 指标 | 介绍与用途 |
|---|---|
| `sm__inst_executed.avg.per_cycle_active` | SM 活跃周期内的平均执行指令数，即该口径下的 IPC |
| `smsp__issue_active.avg.per_cycle_active` | SMSP 活跃周期中发生指令发射的比例/速率 |
| `smsp__issue_active.avg.pct_of_peak_sustained_active` | 指令发射活动相对峰值的百分比 |
| `smsp__inst_executed.avg` | 各 SMSP 执行指令数的平均汇总，不是 FLOP 数 |

发射率低时结合 eligible warps 和 stall 看为什么发不出去；发射率高但仍慢时，可能要关注指令总量和执行管线压力。指令少不保证快，减少指令却增加关键依赖也可能退化。

**5. 整体吞吐 SOL：4 项默认指标**

回答：计算系统和内存系统的哪些资源接近能力上限？

| 指标 | 介绍与用途 |
|---|---|
| `sm__throughput.avg.pct_of_peak_sustained_elapsed` | SM 吞吐的组合指标，用于初步判断计算侧资源压力 |
| `gpu__compute_memory_throughput.avg.pct_of_peak_sustained_elapsed` | 计算内存通路的组合吞吐，包括多个内存相关资源；不能等同于 HBM 带宽 |
| `gpu__compute_memory_access_throughput.avg.pct_of_peak_sustained_elapsed` | cache 和 DRAM 内部访问活动的组合吞吐，用来辅助区分内存内部忙碌程度 |
| `l1tex__throughput.avg.pct_of_peak_sustained_active` | L1/TEX 单元在活跃周期内的吞吐水平 |

判断显存带宽是否接近上限，应继续看第 8 类 DRAM 指标；判断哪条计算管线繁忙，应继续看下面两类。Throughput 型组合指标通常取组成项的最高归一化百分比，不能当作所有资源的平均利用率。

**6. 普通计算、访存发射与控制管线：7 项默认指标**

回答：SM 内部是哪类执行管线承担主要压力？

下表列出完整指标名的公共格式：

```text
sm__inst_executed_pipe_<pipe>.avg.pct_of_peak_sustained_<basis>
```

| `pipe` | 默认 `basis` | 介绍 |
|---|---|---|
| `fma` | `active`、`elapsed` | FMA 管线相关指令吞吐，覆盖浮点算术等；具体支持的整数/浮点指令随架构变化 |
| `alu` | `active` | 算术逻辑管线，主要涉及逻辑、位操作和部分整数计算 |
| `lsu` | `active`、`elapsed` | Load/Store 管线指令吞吐，反映访存指令处理压力，不等同于搬运的 bytes/s |
| `xu` | `active` | 特殊函数及类型转换相关管线 |
| `adu` | `active` | 地址分歧处理相关管线，也涉及部分常量加载和屏障支持；不要简单等同于所有地址计算 |

同一管线的 active 高而 elapsed 低，说明活跃时较忙，但全测量区间平均较低，需要再检查不均衡、空闲和时间线。LSU 高而 DRAM 不高，可能更值得分析指令数量、访问粒度和缓存层级。

管线与指令的对应关系随架构变化。参考：[NVIDIA 管线定义](https://docs.nvidia.com/nsight-compute/ProfilingGuide/#metrics-reference)。

**7. Tensor Core 与 TMEM：6 项默认指标**

回答：使用了哪条 MMA 路径？Tensor 执行与 TMEM 活动如何？

| 指标 | 介绍与用途 |
|---|---|
| `sm__pipe_tensor_cycles_active.avg.pct_of_peak_sustained_active` | Tensor pipe 在 SM 活跃区间内的活动水平 |
| `sm__pipe_tensor_cycles_active.avg.pct_of_peak_sustained_elapsed` | Tensor pipe 按整个测量区间归一化的活动水平 |
| `sm__ops_path_tensor_op_hmma_src_bf16_dst_fp32_sparsity_off.avg` | BF16 输入、FP32 结果、非稀疏 HMMA 路径的数学操作量平均汇总 |
| `sm__ops_path_tensor_op_utchmma_src_bf16_dst_fp32_sparsity_off.avg.pct_of_peak_sustained_elapsed` | 对应 BF16→FP32 非稀疏 UTC/第五代 MMA 路径的操作吞吐相对峰值 |
| `sm__mem_tensor_cycles_active.avg.pct_of_peak_sustained_elapsed` | Tensor Memory 活动周期水平，不是 TMEM 已占用容量百分比 |
| `smsp__sass_inst_executed_op_utcmma.sum` | 执行的 UTCMMA SASS 指令数量；一条 MMA 指令代表的数学工作量不等于一个 FLOP |

这两个 BF16 专用指标不是所有 dtype 的统一 Tensor 利用率。分析 FP16、FP8、FP4 或 INT 运算，需要查找对应路径。不同架构的 Tensor 发射管线与实际计算引擎指标也可能不同，不能仅凭某个 `pipe_tensor` 指标为零就判定所有 Tensor Core 路径都没使用。

文档另外讨论 `sm__pipe_tensor_subpipe_{hmma,imma,dmma}_cycles_active...` 和更广义 tensor ops，属于按需指标。参考：[NVIDIA Compute Pipeline Throughput](https://docs.nvidia.com/nsight-compute/ComputeTriage/#compute-pipeline-throughput)。

**8. DRAM/HBM 流量和带宽：8 项默认指标**

回答：实际访问了多少显存？读写带宽是否成为限制？

| 指标 | 介绍与用途 |
|---|---|
| `gpu__dram_throughput.avg.pct_of_peak_sustained_elapsed` | 同时考虑读写的 DRAM 吞吐水平 |
| `dram__bytes_read.sum` | 显存读取字节总量 |
| `dram__bytes_write.sum` | 显存写入字节总量 |
| `dram__bytes_read.sum.per_second` | 实际显存读取速率，通常为 bytes/s |
| `dram__bytes_read.sum.pct_of_peak_sustained_elapsed` | 显存读取速率相对峰值的水平 |
| `dram__bytes_write.sum.pct_of_peak_sustained_elapsed` | 显存写入速率相对峰值的水平 |
| `dram__sectors_read.sum` | 显存读取 sector 数量 |
| `dram__sectors_write.sum` | 显存写入 sector 数量 |

可对照理论最低字节量判断是否存在额外搬运。写密集 kernel 不能只看读取百分比。DRAM 吞吐低也不能排除 L1/L2、LSU 或内存延迟瓶颈。

**9. 缓存命中：4 项默认指标**

回答：访问在哪一级命中？不同类型访问的局部性如何？

| 指标 | 介绍与用途 |
|---|---|
| `l1tex__t_sector_hit_rate.pct` | L1/TEX 的总体 sector 命中率 |
| `lts__t_sector_hit_rate.pct` | L2 的 sector 命中率 |
| `l1tex__t_sector_pipe_lsu_mem_global_op_ld_hit_rate.pct` | LSU global load 路径的 L1/TEX sector 命中率 |
| `l1tex__t_sector_pipe_lsu_mem_global_op_st_hit_rate.pct` | LSU global store 路径的 L1/TEX sector 命中率 |

命中率要结合流量、缓存策略和算法复用看。只访问一次的流式数据命中率低可以是正常现象；命中率很高也可能同时存在大量冗余 load 指令。

**10. 不同地址空间的访存指令：7 项默认指标**

回答：执行了多少 global、shared、local 访存指令？

| 指标 | 介绍与用途 |
|---|---|
| `smsp__sass_inst_executed_op_global_ld.sum` | global load 指令执行次数 |
| `smsp__sass_inst_executed_op_global_st.sum` | global store 指令执行次数 |
| `smsp__sass_inst_executed_op_local_ld.sum` | local memory load 指令执行次数 |
| `smsp__sass_inst_executed_op_local_st.sum` | local memory store 指令执行次数 |
| `smsp__sass_inst_executed_op_shared.sum` | shared memory 相关指令执行总数 |
| `smsp__sass_inst_executed_op_shared_ld.sum` | shared memory load 指令执行次数 |
| `smsp__sass_inst_executed_op_shared_st.sum` | shared memory store 指令执行次数 |

这是指令计数，不是字节量，也不等于所有异步/TMA 路径的完整流量。global load 多但 DRAM 流量小，可能是缓存复用；相同字节量下指令特别多，可以检查向量化机会。

local load/store 非零是检查寄存器 spill 的线索，但也可能来自显式局部数组、栈等。应结合编译器资源报告和 SASS 确认。限制寄存器配额不保证减少 spill，过度限制反而可能增加 spill。

**11. 合并访存与 sector 效率：7 项默认指标**

回答：一次请求触及多少 sector？传输的数据中有多少真正有用？

| 指标 | 介绍与用途 |
|---|---|
| `l1tex__t_sectors_pipe_lsu_mem_global_op_ld.sum` | global load 对应的 sector 总数 |
| `l1tex__t_sectors_pipe_lsu_mem_global_op_ld_lookup_hit.sum` | 其中在 L1/TEX 命中的 sector 数 |
| `l1tex__t_sectors_pipe_lsu_mem_global_op_ld_lookup_miss.sum` | 其中在 L1/TEX 未命中的 sector 数 |
| `l1tex__t_sectors_pipe_lsu_mem_global_op_st.sum` | global store 对应的 sector 总数 |
| `l1tex__t_requests_pipe_lsu_mem_global_op_ld.sum` | global load 请求数 |
| `l1tex__t_requests_pipe_lsu_mem_global_op_st.sum` | global store 请求数 |
| `smsp__sass_average_data_bytes_per_sector_mem_global_op_st.ratio` | store 每个 sector 的平均有效数据字节数，32B sector 的充分使用对应约 32B |

常用衍生量：`load sectors/request = load sectors / load requests`。在 32 个 lane 全部活跃、地址连续且对齐、每 lane 单次访问 4/8/16 字节的条件下，参考值分别是 4/8/16。广播、部分 lane 活跃、不同访问宽度混合时不能直接套用。

该比值描述请求展开后的 sector 数，不是 DRAM 字节数；缓存命中与否还会改变后续流量。

**12. Warp 停顿与调度状态：34 项默认指标**

回答：warp 为什么不能继续发射？哪些原因更值得排查？

这里有两组不同口径：

```text
16 项汇总比值：
smsp__average_warps_issue_stalled_<reason>_per_issue_active.ratio

17 项逐 PC 状态样本：
smsp__pcsamp_warps_issue_stalled_<reason>

1 项采样总数：
smsp__pcsamp_sample_count
```

`ratio` 是以发射活动为分母的归一化等待量，不是 0—100% 的运行时间占比，也不能与 PC 样本计数直接相加。PC 样本占比可以用相应状态样本数除以同口径的总样本数，但它仍不等于可消除的 kernel 时间比例。

| `<reason>` | 简短介绍 | 排查方向 |
|---|---|---|
| `long_scoreboard` | 等待 L1TEX 路径操作产生的数据依赖，包括 global/local 等访问 | 访问局部性、依赖链、在途请求和延迟隐藏 |
| `short_scoreboard` | 等待非 L1TEX 的 MIO 操作依赖，例如 shared memory、部分特殊函数 | shared 访问、bank conflict、特殊函数和依赖链 |
| `wait` | 等待固定延迟执行依赖 | 串行算术链、独立指令不足 |
| `barrier` | 等待同步屏障条件满足 | 同步范围、到达不均、warp 分工 |
| `membar` | 等待内存屏障/内存顺序相关操作 | fence 的必要性及频度 |
| `math_pipe_throttle` | 所需数学执行管线暂时不可用 | 管线压力和指令组合；不能只解释为 FMA 饱和 |
| `mio_throttle` | MIO 指令队列/通路拥堵 | shared、特殊函数等指令频度及访问展开 |
| `lg_throttle` | local/global 指令队列拥堵 | 标量访存指令数量、向量化、访问粒度 |
| `tex_throttle` | TEX 相关指令队列拥堵 | 纹理/表面及相关通路压力 |
| `not_selected` | warp 已就绪，但调度器选择了另一个 warp | 这是调度状态，不应直接当作有害等待 |
| `selected` | warp 被选中并发射指令 | 生产性状态，不是停顿；仅在默认 PC 样本组中 |
| `branch_resolving` | 等待分支目标等控制流信息解析 | 分支密度、间接跳转 |
| `dispatch_stall` | 指令因分派端条件暂不能送入执行通路 | 结合架构及具体指令分析 |
| `drain` | warp 退出时等待未完成操作排空 | 尾部操作及未完成访存 |
| `no_instruction` / `no_instructions` | 等待取指或指令缓存未命中处理 | 代码体积、指令局部性、短 kernel 启动阶段 |
| `sleeping` | 处于 sleep，例如显式 nanosleep | 是否为 producer/consumer 的有意等待 |
| `misc` | 其他硬件等待原因 | 不能从名称独立推断根因 |

名称差异：汇总组是 `no_instruction`，PC 组是 `no_instructions`。汇总组不含 `selected`，因此为 16 项；PC 组包含它，为 17 项。

最值得联合看的组合包括：long scoreboard + 低 DRAM 吞吐；short scoreboard + shared wavefront excess；wait + 低 eligible；barrier + 角色和输入不均衡。以上都只是诊断方向，不是单指标判决。

状态定义参照：[NVIDIA Warp Stall Reasons](https://docs.nvidia.com/nsight-compute/ProfilingGuide/#metrics-reference)。

**13. 源码行与 SASS 热点：专项分析**

回答：上述指令和等待具体集中在哪些代码位置？

| 数据/API | 介绍与用途 |
|---|---|
| PC 样本指标的 `num_instances()` 与 `correlation_ids()` | 取得每个指令地址对应的状态样本数量 |
| `action.source_info(pc)` | 把 PC 对应到源码文件和行号，需要 line info |
| `inst_executed` | 逐 PC 的 warp 级执行指令计数，热点脚本另外读取；不属于 108 项默认清单 |
| `action.sass_by_pc(pc)` | 查看对应 SASS，内联处理时也用来比较主操作码 |
| `launch__function_pcs` | 内联映射时获取函数起始地址，将反汇编偏移转换为 PC |
| `--binary` 与 `nvdisasm -gi` | 读取内联链，将库头文件中的热点重新归到 kernel 自身的源码行 |

脚本输出总样本热点、各状态热点、执行指令数热点。采样位置可能是等待数据的消费者，产生依赖的 load 在前面；热点总数还包含 selected/not_selected，因此不能直接当作可节省时间排序。

**14. Block 负载均衡与长尾：按需指标 + 专项时间线**

回答：是否少数 SM、CTA 或长输入拖住了最后一段执行？

| 数据 | 介绍与用途 |
|---|---|
| `sm__cycles_active.{avg,max,min,sum}` | SM 活跃周期的实例汇总，比较工作持续时间差异；不在精选 108 项中 |
| `launch__waves_per_multiprocessor` | 结合驻留容量检查是否存在部分尾波；属于默认项 |
| SM activity PM 时间线 | 查看后段是否只剩少量活动，而不是只看全程平均值 |
| 输入决定的 `work_per_cta`、长度分布 | 从数据和 launcher 推导，不是 NCU 计数器 |

文档要求按需计算 max/avg、max/min、末波容量利用等量。不能仅凭某一个不均衡比值确认长尾，必须结合任务启动顺序和时间线。脚本没有自动理解每个 CTA 的算法工作量。

**15. PM Sampling 时间序列：专项分析**

回答：计算、内存、停顿在 kernel 生命周期中怎样变化？

| 序列/API | 介绍与默认行为 |
|---|---|
| `TPC.TriageCompute.sm__cycles_active.avg.per_cycle_elapsed` | SM 活动随时间变化；默认绘制匹配项 |
| `TPC.TriageCompute.sm__inst_executed_realtime.avg.per_cycle_active` | 实时执行指令速率；默认绘制匹配项 |
| `TPC.TriageCompute.sm__pipe_tensor_cycles_active_realtime.avg.pct_of_peak_sustained_elapsed` | Tensor pipe 活动时间线；默认绘制匹配项 |
| `FBSP.TriageCompute.dram__throughput.avg.pct_of_peak_sustained_elapsed` | DRAM 吞吐时间线；默认绘制匹配项 |
| `LTS.TriageCompute.lts__throughput.avg.pct_of_peak_sustained_elapsed` | L2 吞吐时间线；默认绘制匹配项 |
| `SM_A.TriageCompute.l1tex__throughput.avg.pct_of_peak_sustained_elapsed` | L1/TEX 时间线；在报告提供对应序列时默认匹配，版本相关 |
| `action.timed_warp_samples()` | 含 timestamp、PC、stall_reason、not_issued 等字段的采样；脚本按时间桶统计状态占比 |
| `pmsampling:smsp__warps_issue_stalled_<reason>.avg` | 某些旧版报告的 warp 状态时间序列；脚本匹配其中一组常用状态 |

FMA、TMEM 的实时序列在文档中也有列举，但默认绘图正则没有选中它们；可通过 `--metric` 指定报告中实际存在的名称。该参数替换默认选择列表。

脚本可以发现渐长的尾部、周期性利用率低谷等形状，不生成逐 warp、逐指令起止事件。不同 replay 的指标也不能严格逐点对应。

**16. NCU 规则结果：专项提取**

回答：NCU 自带规则发现了哪些问题？它估计的改善空间是什么？

| 字段 | 简短介绍 |
|---|---|
| `section`、`rule` | 规则所属分析 section 和规则标识 |
| `severity` | 信息、正常、优化建议、警告或错误 |
| `title`、`message` | 规则标题和解释 |
| `speedup` | 规则给出的预计改善百分比，可能缺失 |
| `speedup_type` | local/global，指出估计作用于局部还是整体 |

`analyze_reports.py` 按估计值排序输出，但优先级仍需人工核对。多个规则可能描述同一个根因，不能相加得到总收益；不同范围的收益也不应直接等价比较。

**17. 诊断手册中的补充指标：按需查看**

| 主题 | 指标 | 简短介绍 |
|---|---|---|
| Atomic 竞争 | `lts__t_sectors_op_atom.sum` | L2 atomic 操作相关 sector 数量 |
| Reduction 原子操作 | `lts__t_sectors_op_red.sum` | L2 reduction 操作相关 sector 数量 |
| L2 压力 | `lts__throughput.avg.pct_of_peak_sustained_elapsed` | 全程 L2 吞吐，结合 atomic、缓存和 DRAM 判断 |
| Shared wavefront 压力 | `l1tex__data_pipe_lsu_wavefronts.avg.pct_of_peak_sustained_elapsed` | LSU wavefront 处理压力，需要结合访问类型判断 |
| 逐指令 shared 实际开销 | `memory_l1_wavefronts_shared` | shared 操作实际需要的 wavefront 数 |
| 逐指令 shared 理想开销 | `memory_l1_wavefronts_shared_ideal` | 无额外冲突情况下的参考 wavefront 数 |
| Shared 额外开销 | `derived__memory_l1_wavefronts_shared_excessive` | 实际与理想相比的额外 wavefront，用来核对 bank conflict |
| FP64 活动 | `sm__pipe_fp64_cycles_active.avg.pct_of_peak_sustained_active` | 检查非预期双精度路径；名称与硬件支持需确认 |
| FP64 指令吞吐 | `sm__inst_executed_pipe_fp64.avg.pct_of_peak_sustained_active` | 从执行指令吞吐角度检查 FP64 管线 |
| 线程参与程度 | `smsp__thread_inst_executed_per_inst_executed.ratio` | 每条 warp 指令对应的线程指令数量，用作参与程度的线索，不能单独证明分支发散 |

这些不属于精选 108 项，但文档会要求针对性查看。不同地址的流量多不能单独证明 atomic 竞争；shared wavefront 多也不自动等于 bank conflict，应与理想值比较。

**18. 默认指标数量核对与分析顺序**

| 默认类别 | 数量 |
|---|---:|
| 时间与频率 | 3 |
| Launch、资源与硬件容量 | 16 |
| Occupancy 与就绪 warp | 8 |
| IPC 与发射 | 4 |
| 整体吞吐 | 4 |
| 普通管线 | 7 |
| Tensor/TMEM | 6 |
| DRAM | 8 |
| Cache | 4 |
| 访存指令数 | 7 |
| Sector/request 与写效率 | 7 |
| 状态汇总比值 | 16 |
| PC 状态样本与总数 | 18 |
| **合计** | **108** |

这些数据服务于仓库的六个分析维度，不是 108 个独立优化目标。合理阅读顺序是：

1. 时间和频率：确认性能变化真实、条件可比。
2. Launch、occupancy、eligible 和发射：确认工作是否充分，调度是否持续。
3. SOL、计算管线、Tensor、DRAM：判断资源压力在哪一侧。
4. Cache、访存指令、sectors、stall：解释压力或等待的原因。
5. 源码热点和依赖：定位到具体代码。
6. PM 时间线和输入分布：检查平均数是否掩盖阶段性问题。
7. 核对 NCU 规则，形成少量可验证的优化优先项。

来源：[精选指标实现](helpers/ncu_utils.py)、[热点脚本](helpers/extract_stall_hotspots.py)、[时间线脚本](helpers/plot_timeline.py)、[分析维度](reference/05-analysis-dimensions.md)、[诊断手册](reference/06-diagnosis-playbook.md)、[指标名称参考](reference/08-metric-names.md)。本次另用本机 NCU 2026.1.1 的离线 `--query-metrics --chip gb100` 核对了部分定义，未执行任何 kernel 或性能采集。
