# Network File System (NFS) 核心总结

基于 CS162 Lecture 22 & 23

## 1. 架构设计 (Architecture)

NFS 的目标是提供对远程文件的 **透明访问 (Transparent Access)**，让用户感觉就像访问本地磁盘一样。

### VFS (Virtual Filesystem Switch)
为了让 NFS 能无缝集成到 Unix/Linux 中，操作系统使用了 VFS 抽象层。
* **定义**: VFS 是一个通用的文件系统接口，它定义了标准的系统调用（如 `open`, `read`, `write`）。
* **作用**: 区分本地文件和远程文件。
    * 如果是本地文件 -> 调用本地文件系统（如 ext4）。
    * 如果是远程文件 -> 调用 **NFS Client**。
* **核心对象**: VFS 包含 `superblock` (文件系统信息), `inode` (文件元数据), `dentry` (目录项), `file` (打开的文件) 等对象。

### Client-Server 模型
* **NFS Client**: 运行在用户机器的内核中，将 VFS 请求转换为 **RPC** 调用。
* **NFS Server**: 接收 RPC 请求，将其转换回服务器本地的 VFS 操作，访问物理磁盘。
* **通信**: 使用 **RPC (Remote Procedure Call)** 和 **XDR (External Data Representation)** 进行数据传输和格式转换。

---

## 2. 协议特性 (Protocol Characteristics)

这是 NFS 最重要的考点，特别是关于“状态”和“幂等性”。

### A. Stateless (无状态)
NFS 服务器 **不维护** 客户端的任何状态（如打开的文件、文件指针位置等）。
* **实现方式**: 每个请求必须包含完成操作所需的所有信息 (Self-contained)。
    * 例如：读操作必须是 `read(fh, offset, count)`，显式指定偏移量，而不是依赖服务器记录的“当前位置”。
* **优势 (Advantage)**: **Crash Recovery (崩溃恢复)** 非常简单。
    * 如果服务器崩溃重启，它不需要恢复之前的连接状态，因为根本没有状态。
    * 客户端只需要重发请求即可。

### B. Idempotent Operations (幂等操作)
为了支持无状态架构下的“重试”机制，NFS 操作被设计为幂等的。
* **定义**: 执行一次和执行多次的效果是一样的。
* **例子**:
    * `Write(Data, Offset)` 是幂等的（写同样的数据到同样的绝对偏移量，结果不变）。
    * `Append(Data)` **不是** 幂等的（重试会导致数据重复追加）。
    * **解决方案**: NFS 不直接支持 Append 操作，客户端必须先计算文件末尾的 Offset，然后将其转换为标准的 Write 操作。

---

## 3. 缓存与一致性 (Caching & Consistency)

为了解决网络性能瓶颈，NFS 引入了缓存，但也带来了一致性挑战。

### 缓存策略
* **Client Caching**: 客户端会缓存读取的数据块和元数据，以减少网络 RPC 次数。
* **Write-through / Write-behind**:
    * NFS 通常要求修改的数据在返回“成功”给客户端之前，必须 **Commit to server's disk** (v2) 或者至少进入稳定存储。这是为了防止服务器崩溃导致数据丢失。

### 一致性模型 (Consistency Model)
NFS 提供的是 **Weak Consistency (弱一致性)**，而不是严格的 Sequential Consistency。

1.  **Polling (轮询)**: 客户端定期（例如每 3-30 秒）向服务器查询文件属性（如修改时间），检查缓存是否过期。
2.  **后果**:
    * 如果 Client A 修改了文件，Client B 可能在几十秒内仍看到旧数据（直到它的缓存超时去轮询服务器）。
    * **"Close-to-Open" Consistency**: 当 Client A **关闭 (Close)** 文件时，强制刷写数据；当 Client B **打开 (Open)** 文件时，强制重新检查。但在文件打开期间，一致性无法保证。
3.  **并发写**: 如果多个客户端同时写同一个文件，结果是 **不可预测的 (Arbitrary)**，可能会出现数据混合。

---

## 4. 文件句柄 (File Handles)

* NFS 不使用 Unix 的文件描述符 (File Descriptor, fd)，因为 fd 是进程私有的且包含状态（如 offset）。
* NFS 使用 **File Handle (FH)**：
    * 一个不透明的标识符（Opaque Identifier）。
    * 唯一标识服务器上的一个文件。
    * 通常包含：Volume ID, Inode Number, Generation Number (用于检测文件是否被删除并重建)。
* **Lookup 操作**: `open("/foo/bar")` 在 NFS 中不会产生一个“打开”状态，而是产生一系列查找 RPC：
    1.  `Lookup(RootFH, "foo")` -> 返回 `fooFH`
    2.  `Lookup(fooFH, "bar")` -> 返回 `barFH`
    3.  之后的读写操作都直接使用 `barFH`。

---

## 5. 优缺点总结

* **Pros (优点)**:
    * **Simple**: 协议简单，易于实现故障恢复（因为无状态）。
    * **Transparent**: 应用程序不需要修改即可读写远程文件。
    * **Portable**: 基于 RPC/XDR，跨平台兼容性好。
* **Cons (缺点)**:
    * **Performance**: 经过网络和 RPC 开销大，且为了可靠性经常需要同步写磁盘。
    * **Consistency**: 弱一致性可能导致应用看到陈旧数据。
    * **Security**: 早期 NFS 缺乏强安全性，依赖 IP 信任。