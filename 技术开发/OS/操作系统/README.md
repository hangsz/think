## 微内核

1. 采用了面向对象设计
2. 提高了灵活性和可拓展性：OS 许多功能由独立软件实现，根据不同硬件开发独立软件即可，且可以修改、拓展旧功能。
3. 增强了系统的可靠性和可移植性：定义了规范的接口，模块化，更容易开发、验证。与硬件耦合的代码都独立在硬件隐藏层，其他部分与硬件无关，因为移植成本比较小
4. 提供了分布式系统的支持：微内核的模块之间通过消息通信。

## 嵌入式操作系统

1. 嵌入式实时操作系统，简称 RTOS
2. 常用评价指标：系统调用平均运行时间、任务切换时间、线程切换时间、信号量混洗时间
3. 目标：要及时调度可利用资源进行实时控制，要及时响应实时事件和中断
4. 可根据硬件变化/环境要求对内核进行裁剪
5. 调度：大多数是抢占式

## 内存管理

### 物理内存

PFN: 物理页帧号，SPARSEMEM将PFN差分成了三个level，每个level分别对应：

- ROOT NUMBER：ROOT编号，找到 root
- SECTIONS_PER_ROOT: root内的section偏移
- PAGE_SHIFT: section内的page偏移

vmemmap区域：vmemmap区域是一块起始地址是VMEMMAP_START，范围是2TB的虚拟地址区域，位于kernel space

数据结构：

- **mem_secton: 二级指针，指向一级指针数组
- *mem_section: 一级指针，指向 mem_section 数组
- mem_section: struct，有一个 secton_mem_map 成员
- secton_mem_map: 指针，指向 page 数组, 数组每项内容为一个page，一段连续内存
- page: struct, 内容为物理页的基本信息

![](技术开发/OS/操作系统/1.png)

SPARSEMEM中__pfn_to_page和__page_to_pfn的实现如下：

```
#define __pfn_to_page(pfn)      (vmemmap + (pfn))
#define __page_to_pfn(page)     (unsigned long)((page) - vmemmap)      
#define vmemmap        ((struct page *)VMEMMAP_START - (memstart_addr >> PAGE_SHIFT
```

### 虚拟内存

数据结构

- task_struct 中的 mm_struct

![](技术开发/OS/操作系统/2.png)

### 分页管理

固定页大小，且逻辑页大小 == 物理页大小

1. 逻辑地址：（页号，页内偏移）
2. 物理地址：（块号，页内偏移）
3. 逻辑地址与物理地址映射关系：

- 逻辑地址转换为二进制
- 页号和页内偏移：

- 物理页大小(字节 KB) *1024，再转换为二进制，**0 的个数为页内偏移，剩下的地址位数为页号**。1K=2^10，10 位。
- 逻辑地址(10 进制)//物理页大小(字节)

- 块号：查询页表
- 物理地址：快号*物理页大小+页内偏移

**段式管理**

1. 逻辑地址为（段号，偏移），偏移需要在段长约束范围内
2. 物理地址为：段基地址 - 偏移

段页式管理

1. 程序按逻辑分为多段，每段进行分页

## 磁盘管理

## 文件系统

### 文件系统管理

文件系统可以挂栽

![](技术开发/OS/操作系统/3.png)

### 文件

在 Linux 中共有 7 种类型的文件，分为 3 大类：

1. 普通文件，包括文本文件和二进制文件
2. 目录文件（文件夹文件）
3. **特殊文件**

- [链接文件](https://gnu-linux.readthedocs.io/zh/latest/Chapter03/00_link.html)
- 字符设备文件
- 套接字（Socket）文件，用于网络通讯，一般由应用程序创建
- 命名管道文件
- 块文件

[https://gnu-linux.readthedocs.io/zh/latest/Chapter03/00_inode.html](https://gnu-linux.readthedocs.io/zh/latest/Chapter03/00_inode.html)

**磁盘扇区大小：**文件储存在硬盘上，硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存512字节（相当于0.5KB）。

**块：**由多个扇区组成，是文件存取的最小单位。最常见的是4KB，即连续八个 sector组成一个块(block)。

**inode:**

- 存储文件的元信息，如创建者、大小等
- `stat filename`查看文件元信息
- inode 由于也占用磁盘空间，所以要提前划分一块区域存储 inode，一般按照 2KB 存储对应一个 inode 提前划分出存储 inode 的空间。
- inode_in_use：正在使用的 inode 链表(通过i_list域链接)
- inode_unused：未使用的 inode 链表

文件：

- 一个文件对应一个 inode
- 硬链接：linux 允许多个文件指向一个 inode，删除一个文件不影响其他
- 软链接：linux 允许一个文件指向另一个文件，其文件内容为另一个文件名

目录：也是一个文件，对应一个 inode。只是目录文件内容为 文件名：文件 inode 的映射表

文件索引节点法，一个文件包含地址项、索引块(存储地址项)和数据块

1. 直接地址项(4B)，指向数据块(1K)
2. 一级间接地址(4B)，指向一级索引块(1k), 索引块指向数据块，数据块数 1K/4B
3. 二级间接地址(4B)，指向二级索引块(1k), 二级索引块指向一级索引块，一级索引块数 1K/4B

## 进程通信

### 同步

1. PV 操作, P(S): S-1, V(S): S+1

### 外设通信

1. DMA 是在外设与内存之间建立直接数据通路。

## 终端

![](技术开发/OS/操作系统/4.png)

