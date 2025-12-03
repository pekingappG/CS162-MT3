# MapReduce 知识点复习笔记

## 1. 核心概念与动机 (Motivation)
MapReduce 是由 Google 设计的一种分布式计算模型，旨在解决大规模数据处理的问题。
* **目标**：让程序员在不了解分布式底层细节（如网络通信、负载均衡、容错）的情况下，编写能在数千台机器上运行的程序。
* **核心思想**：将计算抽象为两个阶段：Map（映射）和 Reduce（归约）。

## 2. 编程模型 (Programming Model)
数据以 **Key-Value (键值对)** 的形式流动。

1.  **Map 函数**: `(K_in, V_in) -> list(K_inter, V_inter)`
    * 读取输入数据，进行处理，输出中间键值对。
    * *例如*：Word Count 中，将文本行分解为 `(word, 1)`。

2.  **Shuffle (洗牌)**: 
    * **隐式阶段**：框架自动按 Key 对中间数据进行分组。所有具有相同 Key 的数据会被发送到同一个 Reduce 任务。

3.  **Reduce 函数**: `(K_inter, list(V_inter)) -> list(K_out, V_out)`
    * 对同一个 Key 的一组值进行聚合处理。
    * *例如*：Word Count 中，将 `(hello, [1, 1, 1])` 求和为 `(hello, 3)`。

## 3. 系统架构 (Architecture)
* **Coordinator (协调者/Master)**: 
    * 负责管理整个作业。
    * 分配任务（Map 或 Reduce）给空闲的 Worker。
    * 监控 Worker 状态。
* **Worker (工作节点)**: 
    * 执行实际的 Map 或 Reduce 任务。
* **数据流**:
    * Input -> **Map** -> Intermediate Files (写入 Worker 本地磁盘) -> **Shuffle** (通过网络传输) -> **Reduce** -> Output Files (写入 GFS/HDFS 分布式文件系统)。

## 4. 容错机制 (Fault Tolerance)
MapReduce 最强大的特性是能够处理由廉价硬件组成的集群中频繁发生的故障。

### 4.1 Worker 故障处理
当 Coordinator 检测到 Worker 宕机（通过心跳超时）：
1.  **重置正在进行的任务**：该节点上所有未完成的任务重置为 Idle，重新分配给其他节点。
2.  **重做已完成的 Map 任务**：**关键点！** 即使 Map 任务已完成，但因为其中间结果存储在故障节点的**本地磁盘**上，数据已丢失，必须在其他节点重新执行该 Map 任务。
3.  **Reduce 任务无需重做**：如果 Reduce 已完成，结果已写入全局文件系统（如 HDFS），数据是安全的。

### 4.2 Stragglers (落后者)
Straggler 指的是没有死机但运行极慢的节点（可能因为磁盘坏道、CPU 竞争等）。
* **推测执行 (Speculative Execution)**：当作业接近尾声时，Coordinator 会为剩余的慢任务在其他节点上启动**备份副本**。
* **竞速机制**：原任务和备份任务谁先完成，就采用谁的结果，并终止另一个。

### 4.3 幂等性 (Idempotence)
为了让“重试”和“推测执行”安全工作，用户的 Map 和 Reduce 函数必须是**确定性**的。即多次执行相同的输入，必须产生相同的输出。

## 5. MapReduce vs. Spark
MapReduce 的主要局限在于**中间结果落盘**。
* **MapReduce**: Map 输出写磁盘 -> Reduce 读磁盘。对于迭代算法（如机器学习），每一轮都要读写磁盘，效率低。
* **Spark**: 引入了 RDD (弹性分布式数据集)，允许将中间结果**缓存在内存中**。对于迭代应用，性能可提升 10-100 倍。