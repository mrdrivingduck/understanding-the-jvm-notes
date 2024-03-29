# Chapter 3.6 - 低延迟垃圾收集器

Created by : Mr Dk.

2020 / 01 / 26 19:01 🧨🧧

Ningbo, Zhejiang, China

---

## 3.6 低延迟垃圾收集器 (Low-Latency Garbage Collector)

衡量垃圾收集器的三项最重要的指标：

- 内存占用
- 吞吐量
- 延迟

其中，延迟的重要性日益凸显，因此另外两者可以通过升级硬件设备来提高。低延迟垃圾收集器，在整个 GC 过程中基本上都是并发的，只有初始标记、最终标记阶段有短暂的停顿，且停顿时间基本上是固定的，与堆大小无关。

### 3.6.1 Shenandoah 收集器

第一款不由 _Oracle_ 开发的 HotSpot 垃圾收集器。目标：在任何堆内存大小下，都能把 GC 的停顿时间限制在 10 ms 内：

1. G1 的回收阶段是多线程并行，但无法与用户线程并发；Shenandoah 支持并发的整理算法
2. Shenandoah 目前不使用分代收集
3. Shenandoah 摒弃了记忆集，改用全局数据结构 _连接矩阵_ (Connection Matrix) 记录跨 Region 的引用关系 - 一个简单的二维表格，记录引用关系

Shenandoah 收集器的工作过程：

1. 初始标记 (Initial Marking)
   - 标记与 GC Roots 直接关联的对象
   - **Stop The World**，但停顿时间 **与堆大小无关**
2. 并发标记 (Concurrent Marking)
   - 遍历对象图
   - 与用户线程并发
3. 最终标记 (Final Marking)
   - 处理剩余的 SATB
   - 统计回收价值最高的 Region 构成回收集
   - **短暂停顿**
4. 并发清理 (Concurrent Cleanup)
   - 清理 Region 内一个存活对象也没有的 Immediate Garbage Region
5. 并发回收 (Concurrent Evacuation)
   - 把回收集中的存活对象复制到未被使用的 Region 中
   - 并发复制
6. 初始引用更新 (Initial Update Reference)
   - **短暂停顿**
7. 并发引用更新 (Concurrent Update Reference)
   - 将堆中所有指向旧对象的引用修正到复制后的新地址
8. 最终引用更新 (Final Update Reference)
   - 修正存在于 GC Roots 中的引用
   - **停顿**
9. 并发清理 (Concurrent Cleanup)
   - 回收集中的所有 Region 已再无存活对象
   - 清理这些 Immediate Garbage Region，回收这些 Region

_Brooks Pointer_：实现对象移动与用户程序并发的一种解决方案。

> 此前的解决方案：在需要被复制的内存上设置自陷中断，在中断处理程序中，将访问转发到复制后的新对象上。导致用户态与核心态的频繁切换，代价很大，频繁使用不值当。

Brooks 提出在原有对象布局前统一加一个新的引用字段，在非并发复制的情况下，该字段指向对象自己。每次对象访问会带来一次额外的转向开销 (被优化到只有一条汇编指令)。在并发复制阶段，只需将旧对象的指针指向新对象，就能转移访问。这种设计的多线程竞争问题：

- 如果是并发读取，那么访问旧对象和新对象都一样
- 如果是并发写入，就必须保证写操作只能发生在新复制的对象上

Shenandoah 使用 CAS 操作保证了并发访问的正确性。

### 3.6.2 ZGC (Z Garbage Collector) 收集器

目标也是在保证吞吐量的前提下，把 GC 停顿时间限制在 10ms 以下。与 Shenandoah 和 G1 一样，ZGC 也采用基于 Region 的内存布局。但 ZGC 中的 Region 具有动态性 - 动态创建和销毁，分为小型、中型、大型 Region，用于存放不同大小的对象。

ZGC 并发整理算法的实现 - 染色指针技术 (Colored Pointer)。之前，为了给对象存储一些额外的信息，必须在对象头中增加额外的字段，而染色指针直接将标记信息记录在引用对象的指针上。比如 Linux 的 64-bit 指针，高 18-bit 不能用于寻址，余下的 46-bit，将高 4 位提取用于存储标志信息。这种设计也导致 ZGC 能管理的内存不能超过 `2^42` Bytes；显然，染色指针也不支持 32-bit 平台。

染色指针优势：

1. 染色指针使得某个 Region 的存活对象被移走后，Region 可以被立刻释放、重用，而不必等待整个堆中所有指向该 Region 的引用都被修正后才能被清理
2. 染色指针大幅减少了内存屏障的使用数量：因为通过在指针内保存信息，就不需要通过内存屏障维护这些信息了
3. 染色指针可扩展：Linux 的 64 位指针中还有 18 位可以被开发

ZGC 的运作过程：

1. 并发标记 (Concurrent Mark)
   - 遍历对象图，可达性分析
   - 该阶段前后也要经过初始标记 + 最终标记的短暂停顿
   - 标记不在对象上进行，而是在指针上进行
2. 并发预备重分配 (Concurrent Prepare for Relocate)
   - 扫描所有的 Region，统计本次 GC 的 Region，组成重分配集 (Relocation Set)
3. 并发重分配 (Concurrent Relocate)
   - 把重分配集中的存活对象复制到新的 Region 上
   - 为重分配集中的每个 Region 维护一个转发表，记录旧对象到新对象的引用关系
   - ZGC 可以通过染色指针得出一个对象是否处于重分配集中
   - 用户线程访问重分配集中的对象，将会被内存屏障截获，根据转发表中的记录将访问转发到新对象上，并修正更新该引用的值到新对象上 - 指针的 **自愈** (Self-Healing)
     - 只有第一次访问旧对象会陷入转发，触发内存屏障
     - 省去了之后每次访问该对象的开销 (对比 Brooks 每次都要转发)
   - 一旦重分配集中某 Region 中的存活对象都复制完毕后，该 Region 可以被立即释放
     - 但转发表还得留着，因为某些对旧对象的引用可能一直没有被触发修正
     - 一旦被使用，就会自愈
4. 并发重映射 (Concurrent Remap)
   - 修正整个堆中指向重分配集中旧对象的所有引用
   - 该阶段不是很迫切 (因为可以通过自愈的方式保证)
   - 在实现上，将该阶段的工作与下一次 GC 的并发标记阶段合并
   - 所有指针被修正后，转发表就可以释放掉

ZGC 收集器在几乎整个收集过程中都可并发

- 吞吐率方面 (ZGC 的弱项)，已经达到了以高吞吐量为目标的 Parallel Scavenge 的 **99%**，直接超越了 G1
- 停顿时间上 (ZGC 的强项)，与 G1 拉开了两个数量级的差距

## 垃圾收集器的选择

- 应用程序的主要关注点是什么？吞吐率？停顿时间？选择不同特性的垃圾收集器
- 运行应用的基础设施如何？系统架构？CPU 核心数？根据垃圾收集器的实现特性
- 使用 JDK 的发行商？版本号？不同版本的 JDK 允许使用不同的垃圾收集器
