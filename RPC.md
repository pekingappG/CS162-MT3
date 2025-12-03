# Remote Procedure Call (RPC) 核心总结

基于 CS162 Lecture 21 & 22  
RPCs convert everything to/from some canonical form which describes how
to encode/decode entries. Thus, the only thing that needs to be shared is
this canonical representation.

## 1. 核心概念 (Core Concept)

RPC 的核心目标是让分布式系统中的通信对程序员透明。

* **定义**: RPC 允许程序调用位于不同地址空间（通常是远程机器）上的过程（函数）。
* **目标 (Goal)**: 让远程调用看起来就像本地函数调用一样 (**"Make communication look like an ordinary function call"**)。
* **抽象 (Abstraction)**: 它隐藏了底层的数据转换和网络传输复杂性 (**"Automate the complexity of translating & transmitting data"**)。

---

## 2. 架构组件 (Architecture Components)

RPC 的实现依赖于客户端和服务器端的 **"Stub" (存根)** 代码。

### Client Stub (客户端存根)
* 负责 **Marshalling (序列化/编组)** 参数：将函数参数打包成网络消息。
* 发送消息给服务器，并等待回复。
* 接收回复后，负责 **Unmarshalling (反序列化)** 返回值。

### Server Stub (服务端存根)
* 接收网络消息。
* 负责 **Unmarshalling** 参数：将消息解包还原成函数参数。
* 调用实际的服务器过程。
* 将结果 **Marshalling** 后发回给客户端。

---

## 3. 关键机制 (Key Mechanisms)

### Marshalling & Unmarshalling (序列化与反序列化)
这是 RPC 中最昂贵的操作之一。
* **挑战**: 不同的机器可能有不同的数据表示方式（例如 **Endianness** 大小端问题，或者指针在远程机器无意义）。
* **解决方案**: 将数据转换为一种 **"Canonical form" (标准形式)** 或序列化为字节流。
* **IDL (Interface Definition Language)**: 使用接口定义语言来描述函数和类型，通过 **Stub Generator** 自动生成 Stub 代码。

### Binding (绑定)
客户端如何知道要把请求发给哪台机器的哪个端口？
* **定义**: 将用户可见的服务名称转换为网络端点 (**"converting a user-visible name into a network endpoint"**)。
* **Dynamic Binding (动态绑定)**: 通常使用 **Name Service (命名服务)** 在运行时查找。这支持了 **Fail-over (故障转移)**，如果一台服务器挂了，可以绑定到另一台。

---

## 4. RPC 的问题与挑战 (Challenges)

这是考试中判断题和简答题的常考点，特别是关于“RPC 和本地调用是否一样”的问题。

### A. 故障处理 (Failure Handling)
RPC 和本地调用最大的区别在于故障模式。
* **Non-Atomic Failures (非原子性故障)**: 分布式系统中可能出现部分故障。
* **Partial Failures**: 客户端可能不知道服务器是崩溃了，还是仅仅是网络慢。
* **关键区别**: 本地调用要么成功，要么导致整个程序崩溃；而 RPC 可能出现 **Inconsistent view of the world** (例如：请求已在服务器执行，但 Ack 丢失，客户端以为失败了)。

### B. 性能开销 (Performance Overhead)
RPC **不是** 性能透明的 (**"RPC is not performance transparent"**)。

* **结论**: RPC 比本地调用昂贵得多 (**"RPC is more expensive than a local procedure"**)。
* **原因 (Explain)**:
    1.  **Marshalling/Unmarshalling overhead**: 序列化和反序列化参数需要消耗 CPU。
    2.  **Network Latency**: 网络传输比内存访问慢几个数量级。
    3.  **Kernel-Crossing**: 需要系统调用进行网络 I/O。
* **答题句式**: "RPCs introduce significant overheads including **marshalling arguments** and **network latency**, making them orders of magnitude slower than local calls."

---

## 5. 考试常见题型速查

* **Q: Is RPC transparent to the programmer?**
    * **A:** Syntactically yes (looks like a function call), but semantically and performance-wise **NO** (due to failures and latency).
* **Q: Does RPC handle pointers?**
    * **A:** No, pointers usually gain meaning only within a local address space. Data pointed to must be copied/serialized (**Pass by value**).
* **Q: UDP vs TCP overhead?** (Ref: Problem 4e)
    * **A:** **UDP has lower overhead**. TCP requires connection setup (3-way handshake) and maintains state for reliability, creating higher overhead.