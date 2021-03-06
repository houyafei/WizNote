操作系统所使用的内存体系结构是理解操作系统如何运行的关键。

# 一 进程的虚拟地址空间

每个进程都有自己的**虚拟内存空间**，对32位进程来说，这个地址空间的大小位4G，范围从0x00000000~0xFFFFFFFF。

对64位进程来说，大小位16EB。

注意，这里只是**虚拟内存空间**——不是**物理存储器**，需要把物理存储器分配或映射到相应的地址空间，否则会导致范围违规。

# 二 虚拟地址空间的分区

虚拟地址空间被划分位4个区。每个区的大小以来于操作系统的底层实现。

| **空指针赋值分区** | 这一分区地址空间为**[0x00000000,0x0000FFFF]**，保留该分区的目的地是为了帮助程序员捕获对空指针的赋值。任何访问该区间的操作都会引发内存访问违规。 |
| ------------------ | ------------------------------------------------------------ |
| **用户模式分区**   | 这一分区是进程地址空间的驻地，可用的地址区间和用户模式分区的大小取决于CPU体系结构 |
| **64KB禁入分区**   | 用于分隔用户模式和内核模式。                                 |
| **内核模式分区**   | 操作系统代码的驻地，与线程管理、内存管理、文件系统支持、网络支持以及设备驱动程序相关的代码都载入这一分区，应用程序访问这一区间会导致访问违规 |

# 三 地址空间中的区域

当系统创建一个进程并赋予它**地址空间**时，可用地址大部分都是**闲置的（free）**或**尚未分配的（unallocated）**。为了使用这部分地址空间，必须调用**VirtualAlloc**来分配其中的区域，分配区域的操作被称为**预订(参数是MEM_RESERVE)**。

预定地址空间时，系统会确保**区域的起始地址**正好是**分配粒度**的整数倍。目前所有CPU平台使用相同的**分配粒度，大小为64KB**。

系统会确保**区域的大小**正好是**系统页面**（页面是一个内存单元，系统通过它管理内存）大小的整数倍。页面大小根据CPU二有所不同，x86和x64页面大小为4KB，IA-64则为8KB。

# 四 给区域调拨物理存储器

为了使用所预定的地址空间区域，还必须分配物理存储器，并将存储器映射到所预订的区域。这个过程叫做**调拨物理存储器**。物理存储器始终都**以页面为单位来调拨**。通过调用**VirtualAlloc**函数(参数是**MEM_COMMIT**)来将物理存储器调拨给所预订的区域。

当不需要访问所预订的区域中已调拨的物理存储器时，应该释放物理存储器。这个过程称为**撤销调拨****物理存储器**，通过**VirtualFree**函数来完成（参数**MEM_DECOMMIT**）。

# 五 物理存储器和页交换文件

操作系统能让磁盘空间看起来像内存一样，磁盘上的文件一般被称为**页交换文件**，其中**包含虚拟内存**，可供任何进程使用。

从应用程序的角度看，页交换文件以一种透明的方式增大了应用程序可用内存的总量。如果一台机器装备了1GB的内存，硬盘上还有1GB的页交换文件，那么应用程序会认为可用内存的总量为2GB。

不在页交换文件中维护的物理存储器：

当用户执行一个应用程序时，系统会预订一块地址空间，并注明该区域相关联的物理存储器时exe文件本身，而不是从页交换文件中分配空间。这个文件映像称为**内存映射文件**。

# 六 页面保护属性

| 宏名                                             | 说明                                   |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| PAGE_NOACCESS                                             | 任何访问操作都引发访问违规                                   |
| PAGE_READONLY                                             | 试图写入或执行，会引发访问规矩                               |
| PAGE_READWRITE                                            | 执行页面中的代码会引发访问违规                               |
| PAGE_EXECUTE                                              | 读写页面都将引发违规                                         |
| PAGE_EXECUTE_READ                                         | 写入页面将引发违规                                           |
| PAGE_EXECUTE_READWRITE                                    | 任何操作都不会引发违规                                       |
| PAGE_WRITECOPY (系统在映射exe或dll映像文件时使用)         | 试图执行将引发访问违规,试图写入将使系统为进程单独创建一份该页面的私有副本(以页交换页面为后被存储器) |
| PAGE_EXECUTE_WRITECOPY (系统在映射exe或dll映像文件时使用) | 任何操作都不会引发违规,试图写入将使系统为进程单独创建一份该页面的私有副本(以页交换页面为后被存储器) |
| PAGE_NOCACHE                                              | 禁止对已调拨的页面进行缓存,该标志存在的主要目的是为了让需要需要操控的内存缓冲区的驱动程序人员使用,不建议将该标志用于除此以外的其他用途 |
| PAGE_WRITECOMBINE                                         | 给驱动程序开发人员使用,允许堆单个设备的多次写操作组合在一起,以提高性能 |
| PAGE_GUARD                                                | 使应用程序能够在页面中的任何一个字节被写入时得到通知         |



**内存区域类型**

| 类型              | 描述                                                 |
| ----------------- | ---------------------------------------------------- |
| 闲置（Free）      | 该地址空间尚未预订(**RESERVE**)                      |
| 私有（Private）   | 区域的虚拟地址以**系统的页交换文件**为后备存储器     |
| 映像（Image）     | 开始以**映像文件**为后备存储器，但此后不一定是       |
| 已映射（Mapping） | 开始以**内存映射文件**为后备存储器，但此后不一定是。 |

