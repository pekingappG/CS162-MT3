# Windows NTFS (New Technology File System) 总结

基于 CS162 Lecture 19

## 1. 概述
* **背景**：NTFS 是现代 Windows 系统（自 1993 年起）的默认文件系统。
* **设计理念**：相比于 FAT 或早期的 Unix 文件系统，NTFS 采用了更灵活的结构，特别是使用了 **变长范围 (Variable length extents)** 和高度统一的元数据管理。

---

## 2. 核心结构：主文件表 (Master File Table, MFT)
NTFS 的核心数据结构是 **MFT**。在 NTFS 中，几乎“一切皆文件”，甚至 MFT 本身也是一个文件。

* **结构**：MFT 中的每一个条目（Entry）包含文件的元数据（Metadata）。
* **属性化存储**：NTFS 中的所有信息（包括文件名、权限、甚至文件内容）都被视为 **<属性 : 值> (<attribute:value>)** 的序列。

---

## 3. 文件存储策略 (File Representation)
NTFS 根据文件的大小，采用不同的方式在 MFT 中存储数据，以优化性能和空间利用率。

### A. 小文件 (Small Files) - 驻留数据 (Resident)
* **机制**：对于非常小的文件，其数据直接存储在 MFT 记录内部，紧挨着元数据。
* **术语**：这种直接存放在 MFT 里的数据称为 **"Resident Data"**。
* **优点**：读取元数据时也就同时读到了文件内容，无需额外的磁盘 I/O。

### B. 中等文件 (Medium Files) - 范围/区段 (Extents)
* **机制**：当文件变大，无法塞进 MFT 记录时，数据被存储在磁盘的其他数据块中。
* **索引方式**：NTFS 使用 **Extents (范围)** 来记录数据位置。
    * 一个 Extent 记录为：`[Start Block, Size]` (起始块，长度)。
    * 这与 Unix 的块指针（Block Pointers）不同，Unix 记录每个块的地址，而 Extent 记录一段连续的区域，对大且连续的文件非常高效。
* **术语**：这种存储在 MFT 之外的数据称为 **"Non-resident Data"**。

### C. 大文件/碎片化文件 (Large/Fragmented Files)
* **机制**：如果文件非常大或者碎片化严重，导致 Extents 列表太长，无法放入一个 MFT 记录中。
* **解决方案**：主 MFT 记录会包含指向 **其他 MFT 记录** 的指针。这些额外的 MFT 记录专门用来存储剩余的属性列表或 Extents。

---

## 4. 目录结构 (Directories)
* **实现**：NTFS 的目录结构是作为 **B-Trees (B树)** 实现的，而不是像 Unix FFS 那样的简单线性列表。
* **优势**：B-Tree 结构使得在包含大量文件的目录中查找特定文件的速度更快（对数时间复杂度）。
* **内容**：每个 MFT 条目都包含一个文件名属性，其中包含了人类可读的文件名以及父目录的文件号。

## 5. 其他特性
* **硬链接 (Hard Links)**：NTFS 支持硬链接。这是通过在 MFT 条目中包含 **多个文件名属性 (Multiple file name attributes)** 来实现的，每个属性对应一个链接名。