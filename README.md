1.   Introduction
In recent years, ARMv8-based many-core CPUs have been emerging as a compelling alternative for building high-performance computing (HPC) systems[1-3]. Examples of such use cases include the A64FX chip for the Fugaku supercomputer[4, 5], and ThunderX2 in the Astra supercomputer[6]. Studies suggest that ARMv8-based HPC systems can achieve comparable performance as the traditional HPC hardware and are thus strong contenders in the market of next-generation HPC servers[7]. 

In an era where the CPU hits the memory wall[8], the cache is a key CPU component for achieving high performance. Cache performance is important for HPC many-cores because workloads running on such systems often incur frequent inter-core communication that can dominate the program execution time. To unlock the potential hardware performance, an important task of software optimization is to match the memory access pattern to the underlying cache architecture and coherence protocol. Unfortunately, doing so is non-trivial as the cache typically works as a “black box” with many implementation details unavailable to the software developers. To support code and performance optimization for many-core systems, it is highly attractive to have a way to help developers evaluate, characterize and understand the cache behavior of the underlying hardware to adapt their code accordingly. 

Micro-benchmarks are an effective way of revealing the hardware implementation to allow software developers to obtain hardware insights. Indeed, micro-benchmarks have been widely used to characterize and evaluate the memory hierarchy system on the conventional 
 multi-cores. Examples of such benchmarks include the 
 benchmark suite, which focuses on measuring the memory throughput, i.e., data accessing bandwidth with multi-cores[9]. The 
 suite quantifies the performance of various computer components[10]. It uses a pointer-chasing approach to measure the overhead of moving data across cache levels on a single core. However, 
 ignores the communication overhead of transferring cachelines across different hardware cores, which is essential to optimize parallel programs concerning shared memory accesses. For this, Molka et al. provided a set of micro-benchmarks (
) to characterize such performance behaviors[11]. This tool has proven to be extremely valuable for quantifying core-to-core communication[12, 13]. While memory performance characterization is a heavily studied field for the 
 CPUs, there is little work for understanding the memory hierarchy design for ARMv8 high-performance many-core systems. As ARMv8 is emerging as an important class of CPUs in the HPC domain, it is desired to have a dedicated benchmark suite designed for characterizing the memory hierarchy of ARMv8 many-cores. 

This work aims to close the gap of lacking ARMv8 memory characterization benchmarks. To this end, we have extended the 
 benchmark suite to adapt it to ARMv8 systems in terms of obtaining architecture parameters, setting cacheline states, enabling the clock-wise timing, and using the cache-related instructions (Section 3). Our porting leads to a new, open-source benchmark suite, namely 
 1. 

We demonstrate the benefit of 
 by applying it to three representative ARMv8 many-core systems: Phytium 2000+, ThunderX2, and Kunpeng 920 (KP920). We showcase that 
 is useful in characterizing the underlying memory hierarchy of ARMv8 systems. With 
, we measure the core-to-core communication performance of moving cachelines between distinct cores in terms of latency and bandwidth (Section 3). We obtain undisclosed performance data and reveal many micro-architecture details of the many-core systems on both latency (Section 4) and bandwidth (Section 5). With the extensive, quantified results in place, we compare different cache architecture designs of the three ARMv8 processors. We then give quantitative guidelines for optimizing software memory accesses on ARMv8 many-core systems (Section 6). 

Our evaluation results provide a quantitative reference for analyzing, modeling, and optimizing parallel programs on ARMv8 many-core systems. To the best of our knowledge, this is the first effort of systematically dissecting the memory hierarchy of ARMv8 many-core systems. 

2.   System Architectures
This section describes the three ARMv8 many-core CPUs in this work. Table 1 and Table 2 summarize their hardware configuration parameters and software environment, respectively. 

Table  1.  Hardware Configuration Parameters of the Three ARMv8 CPUs
CPU	Core
Frequency (GHz)	Processor
Interconnect	Number of
Cores per Chip	L1 Cache (I/D)
(per Core)	L2
Cache	L3 Cache
(per Chip)	DRAM
Support
Phytium 2000+	2.2	/	64	32 KB/32 KB	2 MB (per core group)	/	8x DDR4-2400
2x ThunderX2 99x	2.5	CCPI2	32	32 KB/32 KB	256 KB (per core)	32 MB	8x DDR4-2666
2x KP920-6426	2.6	HCCS	64	64 KB/64 KB	512 KB (per core)	64 MB	8x DDR4-2933
下载: 导出CSV | 显示表格
Table  2.  Software Environment on the Three ARMv8 CPUs
CPU	Operating System	Compiler
Phytium 2000+	Linux kernel version 4.19.46	gcc 9.3.0
2x ThunderX2 99xx	Linux kernel version 4.19.46	gcc 8.2.1
2x KP920-6426	Linux kernel version 4.19.46	gcc 8.2.1
下载: 导出CSV | 显示表格
2.1   Phytium 2000+ Architecture
Fig.1 gives a high-level view of Phytium 2000+ based on the 
 architecture. It features 64 ARMv8 compatible processing cores, which are organized into eight panels (panels 0–7). Note that each panel connects a memory control unit (MC). 


Figure  1.  High-level view of the Phytium 2000+ architecture. The 64 processor cores are divided into eight panels, where each panel contains eight Xiaomi cores based ARMv8[14].
下载: 全尺寸图片 幻灯片
Each panel has eight Xiaomi cores, and each core has a private L1 cache of 32 KB for data and instructions, respectively. Every four cores form a core group and share a 2 MB L2 cache. Given that the L1 read port is 128 bits in width and runs at 2.2 GHz, we calculate that the theoretical L1 read bandwidth is 35.2 GB/s. 

Each panel contains two directory control units (DCUs) and one routing cell. The DCUs on each panel act as dictionary nodes of the entire on-chip network. With these function modules, 
 conducts a hierarchical on-chip network. Phytium 2000+ uses a home-grown 
 cache coherency protocol to implement a distributed directory-based global cache coherency. 2 

2.2   ThunderX2 Architecture
ThunderX2 is built based on the 
 microarchitecture. Fig.2 shows the architecture of a two-socket ThunderX2 processor. There are 32 cores per socket operating at 2.5 GHz, each with a 32 KB data cache, a 32 KB instruction cache, and a 256 KB L2 cache. All the cores within a socket share a 32 MB last level cache (L3), arranged as 2 MB slices via a dual-ring on-chip bus. The L3 cache is exclusive, filling up with evicted L2 cachelines. This ring bus is connected to the 2nd-generation Cavium’s Coherent Processor Interconnect (CCPI2). There are two load-store units, each capable of moving 128-bit data per core. We calculate that the theoretical peak L1 read bandwidth is 80 GB/s. 


Figure  2.  ThunderX2 architecture 2.
下载: 全尺寸图片 幻灯片
2.3   KP920 Architecture
The KP920 system has two 64-bit ARMv8 processors designed by HiSilicon based on the 
 microarchitecture[15]. The two processors are connected with Huawei Cache-Coherent System (HCCS). Each processor has two CPU-compute dies (CCDs) and one Compute-IO die, connected with a dual-ring Network on Chip (NoC). There are eight CPU clusters (CCLs) within a CCD, and each CCL has four cores running at 2.6 GHz. Besides, each CCD has its memory controllers and an L3 cache slice, working as a NUMA node. That is, the two-socket KP920 can be seen as four NUMA nodes. 

The overview of the architecture of KP920[15] is shown in Fig.3. Each core features 64 KB private L1 instruction and data caches as well as 512 KB private L2. All the 64 cores in one CCD share 64 MB the last level cache. Four cores within a CCL are accompanied with the same L3 cache tag. 


Figure  3.  KP920 architecture[15]. L1D: level-1 data; L1I: level-1 index.
下载: 全尺寸图片 幻灯片
3.   Benchmarking Methodology
Nowadays, many-core CPUs feature a memory system hierarchy to hide memory latencies and improve memory bandwidths. But these architectural features are transparent for programmers, and only limited software control is available. It is challenging to design micro-benchmarks that can reveal the detailed performance properties of a given cache architecture. Therefore, we carefully design a suite of memory micro-benchmarks (
) to characterize and compare the cache architectures of representative ARMv8 many-core systems. 

3.1   Benchmark Design
This benchmark is extended based on the work of [11, 16], mainly targeting the 
 architectures. Due to the architecture and ISA differences between 
 and ARMv8, we heavily extend this memory benchmark to support the ARMv8 systems, aiming to be a versatile cross-architecture modeling tool for cache-coherent many-core architectures. 

Overview. Fig.4 shows that 
 has six measurement steps (S1–S6). We use three threads (T0, Tn, and Tx), each pinned to a distinct hardware core (
, 
n, and 
x). S1 ensures that all the required TLB entries for the current measurement are present in 
. We synchronize the threads at S2 and S4. S3 prepares data in the specified cache level of 
n in a well-defined coherency state (modified, exclusive, or shared). We have to flush the caches at S5, because the memory benchmarks often show a mixture of effects from different cache levels rather than just a single one. To separate these effects, we explicitly place data in certain cache levels[16]. S6 is the latency/bandwidth measurement step, which always runs on 
. 


Figure  4.  Measurement steps with three threads (or cores). Note that T0 denotes a thread running core 0 (
) and Tn denotes a thread running on core n (
n). S1–S6 represent the six measurement steps, respectively.
下载: 全尺寸图片 幻灯片
We use pointer-chasing to measure the latency of moving a cacheline by randomly accessing discontinuous data elements (Fig.5(a)). Each cacheline is accessed only once to avoid data reuse. No consecutive cachelines are accessed to eliminate the influence of the adjacent line prefetcher. By contrast, we measure the sustainable bandwidth by continuously accessing a chunk of data elements (Fig.5(b)). 


Figure  5.  Different memory access patterns supported by 
. (a) Accessing randomly linked data elements to measure the latency. (b) Accessing contiguous data elements to measure the bandwidth. Here, “ran” represents arbitrary data contents.
下载: 全尺寸图片 幻灯片
Setting Cacheline States. Tn places data in the caches in a well-defined coherency state at S3. The coherency state is generated as follows: 1) modified state: Tn writes the data, and invalidates all copies in other cores; 2) exclusive state: Tn first writes to the memory to invalidate copies in other caches, then invalidates its cache, and then reads the data; 3) shared state: Tn caches data in the exclusive state, and then reads the data from 
x. 

Enabling the Clock-Wise Timing. For each measurement, we need a high-resolution timer to measure durations. We can enable the clock-wise timing with the Performance Monitors Cycle Count Register (
) on the ARMv8-based architecture. But the register 
 is only accessible in the kernel mode. Thus, we use a kernel module to activate the performance monitoring unit. The critical steps of this kernel module are summarized as follows: 1) reading the contents of the control register 
; 2) activating the user mode by writing 
; 3) resetting all hardware counters by writing 
, and 4) enabling the performance counter by writing 
. With this kernel module, the 
 register is accessible via the 
 instruction in the user mode. 

Using the Vector Instructions. We use the vector instructions to read/write data from/to the memory system. The ARMv8-based architecture extends NEON with 32 128-bit vector registers while keeping the same mnemonics as general registers 3. In assembly instructions, the register can identify the vector format including Vn (128-bit scalar), Vn (.2D, .4S, .8H, .16B) (128-bit vector), and Vn (.1D, .2S, .4H, .8B) (64-bit vector). We use the 
 instruction on the ARMv8 architecture when moving data between registers and memory. The selected vector format is four single-precision floating-point words (.4S). 

Using Special Instructions. Besides the general instructions, we use special ARMv8 instructions. 
 is used to invalidate specified cachelines. It is useful when controlling the initial coherency state of cachelines. To put target data into the right cache level, we use 
 to ensure that the ARMv8 processors perform no optimization on the execution order of the fetch instructions. In addition, we use the 
 instruction to avoid unaligned memory accesses. 

3.2   Benchmark Portability
Our work targets the widely used ARMv8 many-core processors. This architecture is used by several high-performance computing systems, including Astra, Isambard, and Fugaku. Recently, ARM has announced the release of the ARMv9 architecture but there are currently no commercial off-the-shelf ARMv9 processors available. We believe 
 can be easily ported to ARMv9. Doing so would require using ARMv9-specific assembly instructions for loads and stores as well as providing routines for obtaining system parameters on frequency and cache organization. Other than these, most of 
 can remain unchanged. Our future work will look into the memory characterization of ARMv9. 

4.   Latency Results
In this section, we analyze and compare the latency results of the three ARMv8 architectures. We measure the latency of core 0 (
) loading cachelines which are exclusive, modified, or shared in different cores (
x) and different cache levels. The dataset size is set from half of the L1 cache (16 KB or 32 KB) to 200 MB to cover each memory level. We find that the latency results show a visible phase change as the size of the dataset increases. And this phase change is consistent with the capacity of each cache level. 

4.1   On Phytium 2000+
The cores on Phytium 2000+ are organized into eight panels. We measure the latency when 
 is accessing cores on panel 1 to panel 7, respectively. For each panel, we choose to use the first core. Besides, the “Local” latency means accessing data that has been prepared in 
 locally. The “Same Core Group” means the accessed core and caches are located on the same core group with 
, sharing the same L2 cache slice. The results are shown in Fig.6 and Table 3. 


Figure  6.  Latency (cycle) for accessing different locations on Phytium 2000+. (a) Accessing exclusive or modified cachelines. (b) Accessing shared (with a core in another panel) cachelines. (c) Accessing shared (with 
) cachelines.
下载: 全尺寸图片 幻灯片
Table  3.  Latency on Phytium 2000+
Accessed Core Location	Exclusive/Modified		Shared (A Core in Another Panel)	RAM
L1	L2		L1	L2
Local	1.8(4)	9.1(20)		1.8(4)	9.1(20)	122.3(269)
Same core group	18.6(41)	9.1(20)	9.1(20)	9.1(20)	122.3(269)
Panel 0	45.0(99)–49.1(108)	42.3(93)	37.3(82)–39.5(87)	42.3(93)	122.3(269)
Panel 1	53.6(118)–59.5(131)	54.1(119)	44.5(98)–50.9(112)	54.1(119)	138.2(304)
Panel 2	75.5(166)–80.5(177)	76.3(168)	68.2(150)–72.3(159)	76.3(168)	158.2(348)
Panel 3	65.5(144)–70.5(155)	65.5(144)	57.7(127)–61.8(136)	65.5(144)	154.6(340)
Panel 4	62.7(138)–67.3(148)	61.4(135)	53.6(118)–58.2(128)	61.4(135)	140.0(308)
Panel 5	70.9(156)–77.3(170)	72.7(160)	60.5(133)–88.8(147)	72.7(160)	162.7(358)
Panel 6	92.3(203)–99.1(218)	95.5(210)	80.9(178)–87.3(192)	95.5(210)	174.5(384)
Panel 7	82.7(182)–88.6(195)	84.5(186)	74.5(164)–80(176)	84.5(186)	167.3(368)
Note: The latency data in this table uses two metric units: nanosecond (ns) and clock cycle (cycle). For example, 1.8(4) means 1.8 ns (4 cycles). All the latency tables in Section 4 use the same setting.
下载: 导出CSV | 显示表格
Local Accesses. From Fig.6, we see that the local accessing latency is independent of the coherency state of the accessed data. The local latency changes twice during the whole process, i.e., at 32 KB (the size of L1 cache) and 2 MB (the size of L2 cache). The latencies of accessing the local L1 and L2 cache are 1.8 ns (4 cycles) and 9.1 ns (20 cycles), respectively. The specification of 
 describes that accessing the local L1 and L2 takes 2 ns and 8 ns, respectively, which is identical to our measured results[3]. 

Within a Core Group. Every four cores on Phytium 2000+ share a local L2 cache slice and form a core group. Thus, the accessing latency to the L2 cache is the same as the local one (9.1 ns). For the remote L1 cache, we observe the latency reduces from 18.6 ns to 9.1 ns when the cacheline is shared. This change shows that 
 can directly obtain data from the L2 cache. It can be inferred that the L2 cache is inclusive. Cachelines can be modified in the L1 cache without being written back to the L2 cache because of the write-back policy adopted on Phytium 2000+. This feature leads to a larger overhead (18.6 ns vs 9.1 ns) when accessing the modified data located in the remote L1. 

Across Core Group. The hardware cores on a different core group from 
 will not share the same L2 cache slice. Accessing data across these cores must be forwarded by the routing cell. As a result, the latency numbers will be larger. The specific latency numbers are determined by the distance of these core groups to 
. The latency difference between 
 accessing the two core groups on a remote panel is around 3 ns. We choose to use the core group with a smaller latency to represent the entire panel in this context. It is worth noting that Phytium 2000+ adopts a unique strategy when accessing shared cachelines. 
 obtains data neither from the most recently visited copy (like the MESIF protocol) nor from the nearest copy (the strategy used by ThunderX2). If a third copy is in the same core group with 
, it can be obtained directly from the shared local L2 cache. In this situation, when the size of the dataset is smaller than that of the L2 cache, the latencies to access data are equal to the local L2 latency (9.1 ns). The data beyond the L2 cache size can only be obtained from the remote memory module. Thus, the latency shows a leap at 2 MB, displayed in Fig.6(c). Otherwise, data can be obtained only from the first copy rather than a closer copy (Fig.6(b)). The latencies of accessing the shared cachelines in the remote L2 caches are consistent with the exclusive. Besides, the cores on the same panel are connected directly to the same memory module and incur a similar latency. The latencies to other panels increase over the panel distance. 

4.2   On ThunderX2
The latency measurement results on ThunderX2 are shown in Fig.7 and Table 4. The “Local” has the same meaning as that in Subsection 4.1. The “Same Socket” refers to loading data from cores that share the same L3 cache with 
. And here, we choose to use 
. The results labeled as “Another Socket” denote access to data that is located in the core-caches of another socket. And here we choose to use 
. 


Figure  7.  Latency (cycle) for accessing different locations on ThunderX2. (a) Accessing exclusive or modified or shared (with 
) cachelines. (b) Accessing shared (with 
) cachelines.
下载: 全尺寸图片 幻灯片
Table  4.  Latency on ThunderX2
Accessed Core Location	Exclusive/Modified/Shared (
)		Shared (
)	RAM
L1	L2	L3		L1	L2	L3
Local	1.2(3)	4.00(10)	24.00(60)		1.2(3)	4.00(10)	24.00(60)	95.6(239)
Same socket	18.0(45)–26.0(65)	31.20(78)	24.00(60)		18.0(45)–26.0(65)	31.20(78)	24.00(60)	95.6(239)
Another socket	78.0(195)–112.4(281)	140.70(352)	140.70(352)		18.0(45)–26.0(65)	31.20(78)	24.00(60)	212.3(531)
下载: 导出CSV | 显示表格
Local Accesses. The three turning points of the local latency are consistent with the sizes of the three cache levels of ThunderX2. The latencies are 1.2 ns (3 cycles), 4 ns (10 cycles), and 24 ns (60 cycles), respectively. These results are consistent with those we measured with 
 (1.6 ns, 4.4 ns and 25.8 ns). 

Within a Socket. As 
 shares the same L3 slice with 
, the data located in the L3 cache of 
 can be accessed directly while accessing the local L3 cache (24 ns). Since L3 in ThunderX2 is exclusive, it does not contain data placed in the higher-level caches. Therefore, when 
 accesses data in the remote L1 or L2 caches, it must first load data from the higher-level caches. This operation is independent of the coherency states (7.2 ns). 

Another Socket. The access to another socket is through the CCPI2 link. Transferring data from the L3 cache of 
 takes around 140.7 ns (352 cycles). We obtain that the latency of walking through this link is 116.7 ns by comparing the latencies of accessing 
 and 
. When the cachelines are shared with 
 (see Fig.7(b)), the latencies of loading them from caches of 
 become the same as those from 
. When the second copy is placed on 
 (see Fig.7(a)), transferring the shared cachelines has no difference from the exclusive state. These indicate that the memory controller is able to fetch the nearest copy. 

4.3   On KP920
As we have shown in Subsection 2.3, KP920 has four NUMA nodes. To measure the latency across NUMA nodes, we choose to use the first core of each remote node. We also measure the latencies of 
 accessing 
 (Local), 
 (Same CCL), and 
 (Same CCD) within a NUMA node. The results are shown in Fig.8 and Table 5. It should be noted that the L3 columns in the table only list stable values. 


Figure  8.  Latency (cycle) for accessing different locations on KP920. (a) Accessing exclusive or modified cachelines. (b) Accessing shared (with a core in node 0) cacheline.
下载: 全尺寸图片 幻灯片
Table  5.  Latency on KP920
Accessed Core	Exclusive/Modified		Shared (A Core in Node 0)	RAM
Location	L1	L2	L3		L1	L2	L3
Local	1.15(3)	2.7(7)	14.2(37)		1.15(3)	2.7(7)	14.2(37)	91.5(238)
Same CCL	11.90(31)	14.2(37)	14.2(37)		11.90(31)	14.2(37)	14.2(37)	91.5(238)
Same CCD	39.2(102)–45(122)	45.0(122)	44.2(115)		39.2(102)–45.0(122)	45.0(122)	44.2(115)	91.5(238)
Node 1	68.1(177)–75(195)	75.0(195)	75.0(195)		43.8(144)–61.5(160)	61.5(160)	61.5(160)–75.0(195)	102.3(264)
Node 2	146.9(382)–158.1(411)	164.2(427)	164.2(427)		28.1(73)–30.4(79)	31.2(81)	31.2(81)	189.2(492)
Node 3	161.2(419)–176.5(459)	183.5(477)	183.5(477)		28.1(73)–30.4(79)	31.2(81)	31.2(81)	208.5(542)
下载: 导出CSV | 显示表格
Within a NUMA Node. The first two turning points of the local latency occur at 64 KB and 512 KB, i.e., the private L1 and L2 cache size per core. The last change is at 64 MB, which is the LLC size on a processor. The accessing latencies of the local L1 and L2 caches are 1.15 ns (3 cycles) and 2.7 ns (7 cycles), respectively. The 
 measurement results are 1.5 ns and 3.1 ns, which are basically consistent with ours. For the remote L1 and L2 caches, we observe that the latencies are close to those of accessing the corresponding L3 caches. This observation indicates that the L3 cache of KP920 is inclusive. As shown in Fig.8, the latency of accessing L3 varies a lot. The specific changing process is shown in Table 6. 
 is suited in the same CCL with 
, sharing the same L3 cache tag. Thus, its latency should be the same as “Local” in the L3 stage. We mainly compare “Local” and “Same CCD” latency (Fig.8(a)). We see that both of them keep steady before the data size is 8MB. After that, the former increases (from 14.2 ns to 38.5 ns) as the dataset grows while the latter decreases (from 44.2 ns to 38.5 ns). Finally, they meet at 32 MB and continue to increase till 64 MB. The difference before the data size is 32 MB illustrates the LLC of KP920 has an affinity to the cores. That is, different CCLs correspond to different L3 cache slices. The accessed data is prioritized to be placed in their corresponding local L3 slice, which differs from ThunderX2. The latency of accessing the L3 cache on ThunderX2 does not vary from core to core, while the LLC is also divided into slices. The latencies end up being the same when the entire 32 MB LLC is filled. Thereafter, the data will be placed in another L3 cache on node 1. The greater the dataset is than 32 MB, the larger the overhead the remote access incurs. The latency increases from 38.5 ns to 53.5 ns. 

Table  6.  Latency for Accessing LLC on KP920
Accessed Core Location	Dataset Size
512 KB–8 MB	8 MB–32 MB	32 MB–64 MB
Local	14.2(37)	14.2(37)–38.5(100)	38.5(100)–53.5(139)
Same CCL	14.2(37)	14.2(37)–38.5(100)	38.5(100)–53.5(139)
Same CCD	44.2(115)	44.2(115)–38.5(100)	38.5(100)–53.5(139)
Node 1	75.0(195)	75.0(195)	75.0(195)–85.0(221)
Node 2	164.2(427)	164.2(427)	164.2(427)–171.2(445)
Node 3	183.5(477)	183.5(477)	189.2(492)–183.5(477)
下载: 导出CSV | 显示表格
Across NUMA Nodes. Accessing exclusive or modified cachelines on the remote nodes requires walking through the ring bus (node 1) or the HCCS (node 2 and node 3). As a result, the latencies are larger compared with “Same CCD”. From Table 5, we see that the overhead across CCDs and processors is approximately 10.8 ns and 86.9 ns, respectively. 

When the cachelines are shared (suppose that the second copy is in node 0), the latencies across the NUMA node show a significant difference (see Fig.8(b)). The latencies of accessing cores in another processor (node 2 and node 3) decrease significantly to the same value (31.2 ns). It is even smaller than the latency of “Same CCD” (local node, 44.2 ns). It is possible because 
 accesses the copy in node 0 rather than nodes of another processor. For a core on the same processor but in another CCD (node 1), the latency also decreases but with a very small difference. It does not reach the value of accessing the local node, 44.2 ns. We argue that the data is still transferred through the interchip bus. 

5.   Bandwidth Results
In this section, we focus on the read bandwidth. The experimental settings are the same as those for the latency measurement. Our following analysis will show that the read bandwidth results are consistent with the latency results, with only several exceptions. 

5.1   On Phytium 2000+
As shown in Fig.9, the read bandwidth of the local L1 cache is 33.6 GB/s, which is close to its theoretical peak (35.2 GB/s). Meanwhile, reading data from the local L2 cache can reach a bandwidth of 18.2 GB/s. The read bandwidth to the L2 cache of 
 is the same as that to the local L2 cache of 
. It is because the two cores share the same L2 cache slices. But the bandwidth is reduced to be around 13.3 GB/s when 
 loads exclusive or modified cachelines suited in 
's L1 cache. This is because a check operation is required. The bandwidths of accessing the L1 and L2 caches of the remote cores have a similar trend. The specific bandwidth can be found in Table 7. If the cachelines are shared, it is unnecessary to perform this check step. Thus, the remote L1 bandwidth is the same as that of accessing the corresponding L2 cache. 


Figure  9.  Read bandwidth (GB/s) for accessing different locations on Phytium 2000+. (a) Accessing exclusive or modified cachelines. (b) Accessing shared (with a core in another panel) cachelines.
下载: 全尺寸图片 幻灯片
Table  7.  Bandwidth (GB/s) on Phytium 2000+
Accessed Core Location	Exclusive/Modified		Shared	RAM
L1	L2		L1	L2
Local	33.6	18.2		33.6	18.2	6.5
Same core group	13.3	18.2		18.2	18.2	6.5
Panel 0	11.9	12.5		12.5	11.8	6.5
Panel 1	10.5	11.5		11.4	11.0	5.6
Panel 2	8.3	9.0		8.8	9.0	5.0
Panel 3	9.0	10.0		9.8	10.0	5.0
Panel 4	9.4	10.5		10.8	10.5	5.3
Panel 5	8.5	9.5		9.6	9.5	4.5
Panel 6	6.9	7.8		7.7	7.8	4.0
Panel 7	7.6	8.4		8.3	8.4	4.3
下载: 导出CSV | 显示表格
Because 
, 
, and 
 are located on the same panel, they are connected directly to the same memory module. When accessing data in the local memory module, the bandwidth can reach around 6.5 GB/s. The bandwidth of accessing the remote memory modules varies from panel to panel. The farther the panel is located from panel 0, the smaller the bandwidth we will have. 

5.2   On ThunderX2
Fig.10 and Table 8 give a high-level view of thebandwidths on ThunderX2. 


Figure  10.  Read bandwidth (GB/s) for accessing different locations on ThunderX2. (a) Accessing exclusive cachelines. (b) Accessing modified cachelines. (c) Accessing shared (with 
) cachelines. (d) Accessing shared (with 
) cachelines.
下载: 全尺寸图片 幻灯片
Table  8.  Bandwidth (GB/s) on ThunderX2
Accessed Core Location	Exclusive		Modified		Shared (
)		Shared (
)	RAM
L1	L2	L3		L1	L2	L3		L1	L2	L3		L1	L2	L3
Local	78.3	35.5	24.3		78.3	35.5	24.3		73.3	35.5	24.3		67.8	51.7	24.3	9.0
Same socket	19.5	19.5	24.3		18.5	19.5	24.3		19.5	19.5	24.3		19.5	19.5	24.3	9.0
Another socket	5.8	5.8	6.3		5.8	5.8	6.3		16.2–17.7	19.0	22.4		5.8	5.8	6.3	4.2
下载: 导出CSV | 显示表格
The read bandwidth to the local L1 cache can reach 78.3 GB/s. The measurement result is basically consistent with the theoretical value (80 GB/s). Reading data from the local L2 cache and L3 cache can reach a bandwidth of 35.5 GB/s and 24.3 GB/s, respectively. However, the local L1 read bandwidth drops when accessing the shared cachelines (73.3 GB/s on 
 and 67.8 GB/s on 
). And it fluctuates significantly while another copy is suited on the remote socket. 

As we have mentioned above, only when the accessed data is located in the L3 cache, 
 can load data from the L3 cache directly. In such a case, the read bandwidth to 
 is the same as that of accessing the local L3 cache slice, reaching 24.3 GB/s. Otherwise, the data must be loaded from the remote L1 or L2 caches. When the cacheline is exclusive or shared, it can be loaded directly from the remote L2 caches (19.5 GB/s). When the cacheline is modified, it has to be loaded from the remote higher level cache (18.5 GB/s). 

 is located on another socket, not sharing a common L3 cache slice with 
. As a result, the read bandwidth of accessing the L1 or L2 cache of 
 is 5.8 GB/s, and accessing L3 yields a larger bandwidth, staying around 6.3 GB/s. The bandwidth of accessing data in the local memory module can reach 9 GB/s, whereas accessing data from another memory module stays around 4.2 GB/s. 

5.3   On KP920
The specific bandwidths are listed in Table 9. The local L1 bandwidth is 81.2 GB/s, and the L2 bandwidth is 51.8 GB/s. It can be seen from Fig.11 that the L3 bandwidth in node 0 exhibits a complicated trend. For “Local” and “Same CCL” cores (
 and 
), the bandwidth first stays at 21.4 GB/s and then decreases to 16.5 GB/s at 32 MB. On the contrary, the bandwidth of “Same CCL” (
) first stabilizes at 14.7 GB/s and then increases to 16.5 GB/s. From 32 MB to 64 MB, both of them decrease to the memory bandwidth (11.6 GB/s). The variation of bandwidth is due to the affinity of the L3 cache slices, as we have analyzed in Subsection 4.3. 

Table  9.  Bandwidth (GB/s) on KP920
Accessed Core Location	Exclusive/Modified		Shared (A Core in Node 0)	RAM
L1	L2	L3		L1	L2	L3
Local	81.2	51.8	21.4–16.5		81.2	51.8	21.4–16.5	11.6
Same CCL	28.0	51.8	21.4–16.5		81.2	51.8	21.4–16.5	11.6
Same CCD	15.3	14.7	14.7–16.5		17.5	16.8	15.2–16.5	11.6
Node 1	15.3	14.7	13.3–14.0		14.1	13.3	13.2–12.8	10.3
Node 2	7.0	7.0	7.0–7.8		16.4	16.8	15.2–16.5	6.4
Node 3	6.7	6.7	6.4–7.3		16.4	16.8	15.2–16.5	6.1
下载: 导出CSV | 显示表格

Figure  11.  Read bandwidth (GB/s) for accessing different locations on KP920. (a) Accessing exclusive or modified cachelines. (b) Accessing shared (with a core in node 0) cachelines
下载: 全尺寸图片 幻灯片
The “node 1” core (
) is located on another CCD with 
 but still within the same processor. Therefore, the bandwidth loading data from the remote caches is almost the same as that from 
. Similarly, the bandwidth of “node 2” is close to that of “node 3”, while they are also in the same processor. These explain that the interchip connections within a processor do not affect the bandwidth performance. 

When cachelines are shared (see Fig.11(b)), 
 can load data from a copy in a local node rather than from a different processor. Therefore the bandwidth of the remote caches in node 2 or node 3 can reach the same as “Same CCL”. While the accessed node is in the same processor with 
 (node 1), the bandwidth result shows that 
 still uses the copy in the remote node rather than in the local node. 

The bandwidth of accessing the local memory module is 11.6 GB/s. When accessing the remote memory module on other nodes, the bandwidth will decrease significantly. The difference in the memory bandwidth within a processor is much smaller than the across-processor one. The former is as low as 1.3 GB/s (11.6 GB/s vs 10.3 GB/s), while the latter can reach 5.2 GB/s (11.6 GB/s vs 6.4 GB/s). 

6.   Discussion
Our results have revealed significant differences in intra-core and inter-core communication performance of the three ARMv8 many-core systems. Their performance is compared in terms of the cache organization and the coherency protocols in Subsection 6.1. We then summarize optimization guidelines based on the comparison and analysis (in Subsection 6.2). 

6.1   Comparing Communication Performance
Intra-Core Cache Organization. Each core of Phytium 2000+ or ThunderX2 has a private 32 KB L1 data cache. KP920 owns a larger private L1 data cache per core, twice as large as the former. Accessing the local L1 cache on the three platforms is very fast, taking around 3 cycles. But in terms of the read bandwidth, Phytium 2000+ can achieve around half of that on the other two platforms (33.6 GB/s on Phytium 2000+ vs 78.3 GB/s on ThunderX2 and 81.2 GB/s on KP920). It is because Phytium 2000+ can load 32 bytes per cycle, while ThunderX2 and KP920 can load 64 bytes per cycle. ThunderX2 and KP920 have private L2 cache slices. By contrast, the L2 cache of Phytium 2000+ is a shared last-level cache, which will be discussed in Subsection 6.2. The local L2 latencies of both ThunderX2 and KP920 are small, 4 ns and 2.7 ns, respectively. But the L2 size of ThunderX2 is only half of that of KP920. Moreover, the L2 read bandwidth of KP920 is much larger than that of Thunder (51.8 GB/s vs 35.5 GB/s). Thus, in general, the performance of KP920’s L2 cache is better than that of ThunderX2's. 

Inter-Core Cache Organization. We compare the shared LLC cache organization and analyze the inter-core LLC latencies. Table 10 summarizes the LLC latencies for 
 accessing other cores on the three ARMv8 platforms. In addition, we measure the inter-core LLC latency between 64 cores. We use heat maps to visualize the measurement results in Fig.12, where the three platforms use a uniform color band for the intuitive comparison. It is easy to see that the LLC latency between cores is symmetrical. The LLC size of Phytium 2000+ is the smallest (i.e., 2 MB sharing among four cores and thus 32 MB in total), while the latency is minimal (9.1 ns). The latency of the local LLC on Phytium 2000+ is minimal (9.1 ns) because there is only one private cache between the core and the local LLC. But its size is too small, with 2 MB shared by four cores. When a core accesses other LLCs, the latency varies from 42.3 ns to 95.5 ns according to the distance between panels. On ThunderX2, each core can access another core sharing with the LLC with the same latency (24 ns). It is because the 32 cores on a single chip are connected through a uniform ring bus. The latency number increases to 140.7 ns when accessing the LLC on another socket, which is the largest among the three platforms. Contrary to ThunderX2, the LLC of KP920 is partitioned and has an affinity for hardware cores, resulting in nonuniform latency. In general, the latency can be divided into three levels according to the core layout of CCL, CCD, and socket. KP920's advantage compared with ThunderX2 is that the 64 cores are located on the same socket; and thus the maximum latency is smaller. But due to the affinity, its LLC latency is unstable as the data size grows, which has been analyzed in Subsection 4.3. 

Table  10.  LLC Latency (ns) for 
 Accessing Other Cores on the Three Platforms
Accessed Core	Phytium 2000+	ThunderX2	KP920
–
9.1	24.0	14.2–38.5–53.5
–
42.3	24.0	44.2–38.5–53.5
–
54.1	24.0	44.2–38.5–53.5
–
76.3	24.0	44.2–38.5–53.5
–
65.5	24.0	44.2–38.5–53.5
–
61.4	140.7	75–85
–
72.7	140.7	75–85
–
95.5	140.7	75–85
–
84.5	140.7	75–85
Note: Different values connected with “–” mean the LLC lantency changes of KP920.
下载: 导出CSV | 显示表格

Figure  12.  Core-to-core LLC latency (ns) for 64 cores on the three platforms. A lighter color represents a smaller latency. (a) Phytium 2000+. (b) ThunderX2. (c) KP920.
下载: 全尺寸图片 幻灯片
Cache-Coherency Protocols. We compare the three systems in terms of coherency protocols. We observe that Phytium 2000+ and KP920 show no difference between the exclusive and the modified states. It is probably because they both use directory-based protocols. The most noticeable difference between them is how they handle the shared data. When there are multiple shared copies on distinct cores, ThunderX2 adopts a straightforward policy—the accessing core can obtain data from the nearest copy. Meanwhile, Phytium 2000+ will fetch data from the first copy if no copy is in the same core group with the accessing core. It may lead to a large latency because the first copy can have the farthest distance. Besides, ThunderX2 uses an exclusive LLC policy when managing the multi-level caches. In this case, cachelines from the remote higher-level caches must be fetched from the remote cores or main memory. It is because no copy exists in the exclusive L3 cache. From the measurement results (see Fig.7(a)), we observe that it chooses to use the remote cores. Compared with the inclusive policy adopted by KP920, ThunderX2 shows no performance loss but increases the effective capacity of the relevant cache slices. 

6.2   Optimization Suggestions
Based on our measurement results and the analysis, we summarize three optimization suggestions (OSs) for programmers on the three ARMv8 many-core systems. 

OS1. Our performance data can be used to identify the communication bottleneck of the parallel algorithms. The efficiency of inter-core communication on many-core processors is an important factor restricting the performance of parallel programs. Identifying the communication bottleneck is the basis of optimizing parallel algorithms. Through abstracting inter-core communication into the read and write operations on shared variables, we can build a communication model for the ARMv8 platforms with the latencies measured from 
. For example, the synchronization barrier is a typical parallel algorithm restricted by inter-core communication. Different barrier algorithms such as dissemination and tournament algorithms have different communication patterns. We can use the communication model to determine the bottleneck of each algorithm on the ARMv8 platforms for the following optimization. 

OS2. We recommend that programmers pin a group of threads that access the same shared variables to cores that share the same LLC slices. Our experimental results show that the cost of obtaining data from cores sharing the same LLC slices is generally smaller than that from other cores. Taking the tree-based barrier as an example, a group of threads share a shared integer variable to achieve synchronization. By controlling each group of threads to share the same LLC slices as many as possible, expensive remote accesses can be avoided. The same idea can also be used to optimize other collective communication algorithms such as broadcast and reduction. In particular, some processors like KP920 may have non-uniform LLCs. Therefore, it is better to place threads in four cores in a CCL. And the size of the data shared by multiple threads should be controlled within 8 MB for fast local accesses. 

OS3. Programmers should keep an eye on the panel distances. If there is a need for access across NUMA nodes, it is crucial to select the right nodes to minimize the cross-node overhead. Taking running SpMV (sparse matrix-vector multiplications) on Phytium 2000+ as an example, programmers should pay attention to the distance between panels. It is because using multiple threads for SpMV involves core-to-core communication to achieve the sharing of the dense input vector. Many hypergraph-based algorithms[17] have been proposed to minimize the inter-thread communication. 

7.   Related Work
Although the effective use of memory systems is essential to obtain the best performance, vendors seldom provide the details of the memory hierarchy or the achieved performance. For this reason, researchers have to obtain such performance results and implementation details through measurements. 

Babka and Tuma[18] proposed experiments that investigate detailed parameters of the 
 processors. The experiment is built on a general benchmark framework and obtains the required memory parameters by performing one benchmark or a combination of multiple open-source benchmarks. It focuses on detailed parameters, including the address translation miss penalties, the parameters of the additional translation caches, the size of cacheline, and the cache miss penalties, etc. 

McCalpin[9] presented four benchmark kernels (Copy, Scale, Add, and Triad), 
, to access the memory bandwidth for current computers, including uniprocessors, vector processors, shared-memory systems, and distributed-memory systems. 
 is one of the most commonly used memory bandwidth measurement tools in Fortran and C. But it focuses on throughput measurement without considering the latency metric. 

Molka et al.[11] proposed a set of benchmarks, including studying the performance details of the 
 architecture. Based on these benchmarks, they obtained undocumented performance data and architectural properties. It is the first to measure the core-to-core communication overhead, but their work is only applicable to the 
 architectures. Fang et al. extended the microkernels to Intel Xeon Phi[13]. Ramos and Hoefler[12] proposed a state-based modeling approach for memory communication, allowing algorithm designers to abstract away from the architecture and the detailed cache coherency protocols. The model is built based on the measurement results of the cache-coherent memory hierarchy. 

Besides the 
 processors, researchers have designed microbenchmarks for other many-core processors to demystify their microarchitectures and memory hierarchies. Wong et al.[19] developed a set of CUDA microbenchmarks and measured the architectural characteristics of the NVIDIA GT200 (GTX280) GPU. Mei and Chu[20] proposed a fine-grained pointer chasing microbenchmark to investigate the throughput and access latency of GPU's global memory and shared memory. They investigated the GPU memory hierarchy of three recent NVIDIA GPUs: Fermi, Kepler, and Maxwell. Lin et al.[21] presented a microbenchmark suite called 
 to evaluate the key micro-architectural features of the SW26010 many-core processor. This evaluation work targets specialized accelerators, e.g., GPGPUs or SW26010, rather than the cache-coherent many-core architectures. 

There exist some performance analysis work on the ARMv8-based HPC systems. Mantovani et al.[7] analyzed the performance and energy consumption of Dibona, a system powered by ThunderX2. McIntosh-Smith et al.[22] presented the performance results of Isambard, which combines ThunderX2 CPUs with Cray's Aries interconnect. These studies focus on the performance behaviors of the entire system rather than the cache architectures. 

8.   Conclusions
In this paper, we extended and refined a set of benchmarks (
) to measure the intra-core and inter-core communication performance of the ARMv8 systems. We chose three representative ARMv8 systems as our experimental platforms to demonstrate the potential of 
, including Phytium 2000+, ThunderX2, and KP920. Experimental results showed that our 
 can provide inter-core latency and bandwidth in different cache levels and coherency states. By comparing and analyzing these detailed and quantitative performance descriptions, we found that the three ARMv8 processors have their own strengths and weaknesses in cache organization and coherency protocol. For example, ThunderX2 has a uniform and exclusive LLC, which is good for data communication within a socket. While the overhead of accessing data across CCPI2 is too high. Performance data cannot help us determine which microarchitecture design is optimal, but it can help us identify the communication bottleneck of the parallel algorithms. We also provided guidelines on improving the performance of parallel programs by optimizing memory accesses based on the communication performance. 

For future work, we will extend our 
 to ARMv9 machines once they are available. Besides, we will combine the memory access patterns of different applications with the communication performance of different architectures to determine the optimal correspondence. We want to find the most versatile microarchitecture design. 
