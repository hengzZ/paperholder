## Quantifying the Performance Impact of Memory Latency and Bandwidth for Big Data Workloads

##### Abstract
* In recent years, DRAM technology improvements have scaled at a much slower pace than processors. <br>
  ***While server processor core counts grow from 33% to 50% on a yearly cadence, DDR 3/4 memory channel bandwidth has grown at a slower rate,
  and memory latency has remained relatively flat for some time.***
* Combined with new computing paradigms such as big data analytics, which involves analyzing massive volumes of data in real time, there is a
  trend of increasing pressure on the memory subsystem. <br>
  ***This makes it important for computer architects to understand the sensitivity of the performance of big data workloads to memory bandwidth and latency,
  and how these workloads compare to more conventional workloads.***

**Values:**
* present analytic equations to quantify the impact of memory bandwidth and latency.
* classify different workloads according to their inherent bandwidth requirement and latency sensitivity.

##### Related Work
1. discussing memory system characterization of conventional enterprise workloads. （传统 workloads 的 memory 系统特性）
2. performance analysis of large-scale internet data center workloads. （互联网数据中心）
    * 处理器核心数相关特性
    * 调度策略特性
    * 平台级相关设计特性
3. characterized big data workloads. （Hadoop 集群）
    * 特殊 case 的特性
    * 功耗-性能表现
    * page-table 和 virtual memory 相关优化
    * L1/L2/L3 cache misses 特征
    * 内存使用特征
4. performance modeling. （）
    * data and instruction cache miss rates
    * mask the cost by performing ops in parallel
    * instruction level parallelism
    * estimate performance using CPI
    * memory-level parallelism (MLP)

##### Workloads
* Big Data
    * In-Memory Column Stores
    * Needle(针) In The HayStack(干草堆) (NITS) Search
    * Proximity Search
    * Spark In-Memory Computing
* Enterprise Workloads
    * Online Transaction Processing (OLTP)
    * Java processing (JVM)
    * Virtualized processing (virtualization)
    * Web tier caching (web caching)
* High Performance Computing Workloads (HPC)
    * SPEC CPU2006 SPECfp benchmark suite <br>
      —— The SPEC CPU2006 benchmarks represent real-world problems of R&D environments or a highly specialized field such as weather forecasting.
* Memory Traffic Characterization
    * Intel Memory Latency Checker

##### METHODOLOGY
##### 1.1 Cycles per Instruction and Pathlength
* number of clock *cycles per instruction* (CPI)
* the required number of instructions to complete a unit of work. (*Pathlength*)   

##### 笔记
***Pathlength*** 取决于软件代码路径，即当前 workload 完整执行需要的指令执行数目。 <br>
论文中的研究针对 fixed workload (fixed pathlength) 和固定核心数量。

##### 1.2 Relating CPI to Memory Behavior
假设前提
* instruction throughput is sensitive to and limited by memory latency. (***memory bound***)

其他可能的极端
* ***core bound*** —— performance is solely limited by the processor issue width and functional units.
* ***bandwidth bound*** —— performance is limited by the maximum amount of memory bandwidth the system can deliver.

简写符号表
* ***CPI*** —— Cycles per instruction.
* ***PL*** —— Pathlength.
* ***CPI_eff*** —— Effective CPI.
* ***CPI_cache*** —— CPI with an infinite cache. （Cache 足够用不存在 Miss。）
* ***MPI*** —— Misses per instruction.
* ***MPKI*** —— Misses per 1000 instructions.
* ***MP*** —— Miss penalty(惩罚), in cycles.
* ***BF*** —— Blocking factor. （percentage of the miss penalty. 用于调整惩罚力度的大小。）
* ***MLP*** —— Memory-level parallelism. （*multiple simultaneous outstanding long latency cache misses.*）
* ***BW*** —— Bandwidth.
* ***WBR*** —— Writeback rate @ cache line evictions(收回).
* ***LS*** —— Line size.
* ***IOPI*** —— I/O events per instruction.
* ***IOSZ*** —— Average size of memory read per I/O event.
* ***CPS*** —— Cycles per second (Core Speed, Hz)
* ***HPC*** —— High performance computing.
* ***NITS*** —— Needle in the haystack queries.

###### single-thread （two-level memory hierarchy） 条件下的 CPI 模型
<div align="center"><img src="pics/simplest-CPIeff-form.jpg" width="35%"></div>

* ***If an application is truly core bound, it will have no sensitivity to memory latency, and the BF will be 0.***
* ***If the workload is bandwidth bound, the equation should still hold(成立), as queuing theory informs us that latency increases dramatically(显著地) as available bandwidth becomes saturated(饱和的).***

综上，
* MP 即数据从 Memory 到 Cache 的时钟消耗。
* 当 ***core bound*** 时， ***BF*** 设置为 0。
* 当 ***bandwidth bound*** 时， ***BF*** 需要根据实际情况调整，以符合 latency 特性。

##### 笔记
* 单线程、两级内存，即 Memory 之下只有一级 Cache。
* ***CPI_eff*** 即不存在 Cache Miss 理想情况下的 CPI 加上 MPI 乘以 MP，同时乘以一个惩罚力度调整因子 BF。

###### 其他论文中的 CPI 模型
<div align="center"><img src="pics/another-CPIeff-form.jpg" width="35%"></div>

将两个 ***CPI_eff*** 公式联立，可推导出 ***BF*** 的求解公式
<div align="center"><img src="pics/BF-solving-form.jpg" width="30%"></div>

结论，
* ***BF*** 与 ***1/MLP*** 成比例。
* ***BF*** 又由 ***Overlap_CM*** 抵消一部分。
* ***Overlap_CM*** —— represents the overlap of core execution with cache misses, and is computed as a fraction of the average miss penalty. (core 执行与 cache miss 的重叠程度)

特别说明
* ***Overlap_CM*** 和 ***MLP*** 不容易直接监测获取。

因此，
* 后面需要考虑如何计算获得 ***BF*** 的值。
* 论文中假设 ***BF*** 是一个常量，因为希望 ***BF*** 计算公式的第二项部分相比第一项 ***1/MLP*** 很小，同时希望当 ***MP*** 增大时该项趋近于 0。

##### 笔记
***CPI_cache*** 不代表没有 ***core stall***。 （*It is possible for data dependencies and serialization events to cause core stalls even when executing with an infinite cache.*）

另外，
* 论文假设 infinite third-level(last-level) cache （LLC），即 L1/L2 的未命中将导致访问 LLC 而导致 stall cycles。
* ***CPI_cache*** 的值通过对 core 进行带有指令跟踪的仿真获得。

##### 1.3 Modeling Bandwidth
###### Bandwidth 需求模型 （bandwidth demand in bytes per second on the memory subsystem）
<div align="center"><img src="pics/bandwidth-demand-form.jpg" width="35%"></div>

* Miss 导致的读写影响
    * ***MPI*** misses per instruction at the processor cache(either demand or prefetch).
    * ***WBR*** percentage of cache misses that require writeback of a dirty victim.
    * ***LS*** line size in bytes.
* ***I/O references also result in memory reads and writes.*** （I/O 事件）
    * ***IOPI*** rate of I/O events per instruction.
    * ***IOSZ*** average size of a memory read or write per I/O event.
* ***CPS*** —— Core speed in cycles per second. (MHz/GHz)

###### ***MP*** cache miss penalty 的计算
***Using BW value, we can estimate the average MP using queuing theory or characterization data that provides a relationship between queuing delay and utilization.***

##### 1.4 Application of the Model
先验信息 <br>
***In many cases, workloads will exhibit(展现) distinct phases of execution, and these phases will have different values for CPI_cache and MPI, which in turn affect bandwidth demand and observed MP.*** 
（很多 workload 表现出明显的不同执行阶段，而这些不同执行阶段的 ***CPI_cache, MPI, MP*** 不同。）

* 确定可靠的最小采样时长 <br>
  *In cases where bandwidth demand does not reach capacity when measured at 100ms to 200ms intervals, we find that this steady-state periodic behavior can be represented accurately with our model.*
* 分段独立监测 <br>
  *We can apply our model to multiple program phases independently.*

模型失效的情况
* If a workload's performance is limited by synchronization bottlenecks or available bandwidth from I/O devices.

##### BUILDING THE MODEL
##### 2.1 统计 ***CPI_cache BF***

先验信息 <br>
我们通过 emon 检测的 CPI 数值实际就是 ***CPI_eff***。 另外还可以监测出 ***MPI MP***。

观测不同的 miss penalties 下 ***CPI_eff*** 的值
* When we reduce the freq at which the core executes(called freq scaling), we slow down the core speed while keeping the memory speed the same.
  (保持 memory speed 不变，降低 core freq。)
* When we reduce the speed at which the memory subsystem operates, the penalty for a miss measured in core cycles is increased due to memory speed being reduced.
  (保持 core frq 不变，降低 memory speed。)

* 通过以上的控制变量测试，我们可以测出不同 ***（MPI * MP）*** 下的 ***CPI_eff***。
* 然后，带入 CPI 模型公式计算 ***CPI_cache BF***。（方程组求解法）

##### APPLYING THE MODEL
##### 3.1 Enterprise and HPC Workloads (Workloads 的结果进行分析)
应用途径 <br>
*We compare CPI_cache and the BF obtained using our methodology with various enterprise and high performance computing workloads.*
***This helps us position how big data, as a workload class, compares to the other classes.***

结论
* As shown in Tab.4 for the enterprise workloads, on average, it can be observed that the blocking factor(***BF***) is relatively high, likely due to the ineffectiveness of ***prefetching*** and ***branch prediction***.
* The workload parameters for the HPC workloads are shown in Tab.5. For high performance computing workloads, on average, the blocking factor(***BF***) is observed to be low.
  These usages typically involve numerical simulations and analysis that involve ***large amounts of data*** and ***instruction-level parallelism***.
  The data access is also regular, making ***prefreching*** highly effective.

指标 ***BF***
* ***The low blocking factor(BF) indicates that the CPI is not heavily dependent on the latency(延迟) of memory references, and these workloads have sufficient parallelism to mask(覆盖隐藏) the same.***

<div align="center"><img src="pics/workload-parameters.jpg" width="45%"></div>

##### 3.2 Creating Workload Classes (分析固有的 memory 读写需求和 latency 敏感度)
<div align="center"><img src="pics/bandwidth-demand-vs-latency-sensitivity.jpg" width="45%"></div>

由上图分析，
* workloads 可划分为多个聚类中心（K-means），每个不同的聚类中心对 latency 和 bandwidth 具有不同的敏感度。
* ***At this point, we omit(忽略) the core-bound Proximity query workload from our analysis, as it will not show any sensitivity to memory latency or bandwidth.***

##### 3.3 Impact of Memory Subsystem (研究 Memory 子系统的变化对性能的影响)
* Queuing Delay and Effective Utilization
* Impact of Bandwidth
* Impact of Compulsory Latency

意义，为未来的内存系统及相关技术提供研究方向。
