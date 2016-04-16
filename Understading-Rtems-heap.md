
##Understading-Rtems-Heap
不是linux那种的虚存管理，而是实存管理，即采用线性编制的方式，逻辑地址和物理地址一一对应的平面模式【这也是和linux内核的一大不同】
Linux采用的段页式管理方式能够将应用程序使用的地址和实际的物理内存大小脱离开，
如果采用实存管理方式，应用程序可以使用的逻辑地址和实际的内存大小是息息相关的
实存少了安全性，但是增加了分配内存的确定性和时效性

####静态内存分配和动态内存分配
* 静态内存分配是指在编译或者链接时将应用程序所需的内存空间分配好，采用这种分配方案的RTEMS内核和应用所占的内存空间在编译时候就能够确定，中断向量表等其它区域所占用的内存空间大小是个定值， 这样采用静态的内存分配机制，在编译时就可以确定 RTEMS 所需内存的大小。
* 动态内存分配 是指在系统运行时候根据应用需要动态分配内存，动态分配会产生内存碎片和时间不确定的问题。出于实时性可靠性和成本的需求，硬实时操作系统不允许动态分配，而对实时性要求不高的系统，允许进行动态内存分配，只要保证其尽快执行完成就好

目前嵌入式实时操作系统基本上是硬实时和软实时的结合体。rtems也是，所以rtems中既有静态分配又有动态分配
rtems中通过配置头文件的形式完成了操作系统需求内存的静态分配工作，对于动态内存分配主要提供了两种分配方式：
1. 分区管理：是将内存划分为大小相等的缓存块,以队列的形成将空闲缓存块组织在一起,并以缓存块（定长内存块）为单位动态地分配。
2. 区域管理：以可变大小的内存段进行分配,用双向链表来管理空闲内存段,采用首次适应算法对内存（变长内存块）进行分配。

Rtems中动态内存管理（workspace、partition和region都是基于heap管理的）
先看heap，再看workspace，这两个是核心抽象层的，partition和region是服务层的。
####//heap
heap主要是对内存空间进行管理提供分配和释放的接口
```
bootcard
{
     Configuration.work_space_start = (void*)0x800000;
     Configuration.work_space_size=0x36000;
epos_initialize_data_structures
{
     _Workspace_Handler_initialization
     {
          //是否进行初始化
          _Heap_Initialize
          {
               //根据设定的work_space_start和work_space_size初始化_Workspace_Area          
               _Workspace_Area是一个 Heap_Control全局变量，它来统一管理这个系统中的所有的
          }
     }
}
}
```
_Workspace_Area(Heap_Control类型)
```
typedef struct {
Heap_Block free_list;      //空闲的堆数据块环形链表的头和尾
uint32_t page_size;        //分配单位和对齐大小
uint32_t min_block_size;   //按 page_size 对齐的最小 block 大小
void *begin;               //堆的起始地址
void *end;                 //堆的结束地址
Heap_Block *start;         //堆中第一个合法的堆数据块地址
Heap_Block *final;         //堆中最后一个合法的堆数据块地址
Heap_Statistics stats;     //对堆使用的运行时统计
} Heap_Control;
```
其中堆数据块这个结构就是把整个堆空间分成若干内存块，初始化的时候整个堆空间只有一个大的可用堆数据块（还有一个只是标志结束的 dummy block），以后每分配一块内存就产生一个堆数据块（ block）。堆数据块是通过 Heap_Block 结构来描述的。
```
typedef struct Heap_Block_struct Heap_Block;
struct Heap_Block_struct {
uint32_t prev_size; // 如果前一堆数据块是空闲的，则表示前一堆数据块的大小
uint32_t size;      // 表示了本块的大小和前一块的状态
Heap_Block *next;   // 指向下一空闲堆数据块的指针 只有在本身为free时有效
Heap_Block *prev;   // 指向上一空闲堆数据块的指针 只有在本身为free时有效
};
```
####//操作
1. _Heap_Initialize 函数
相当于按照指定的地址分配了内存空间，组织成了堆的形式，比如说为work_space分配的空间
这个堆初始化之后只是一个大的block，相当于环形链表上只有一个节点，就是整个堆的大小的内存空间
![](http://i.imgur.com/nR8cUOM.png)
2. _Heap_Allocate分配堆
_Heap_Allocate可能被多个地方调用，各个管理组件的数据结构的初始化都需要分配workspace的空间，此时需要调用_Heap_Allocate
完成在指定的堆中分配指定大小的堆空间
rtems对堆的动态分配采用的是首次适配（first-fit）算法：这种算法的主要缺点是使内存的前端出现很多内存碎片（rtems为此设置一个阈值避免每次分配的时候搜索这些小的碎片内存块）
具体来说，就是在分配的时候，逐个遍历堆中的空闲块链表freelist，如果找到比当前要分配大小还大的内存块大小，就利用这块空闲块来分配所要分配的空间，在分配好之后，如果发现这个空闲块剩下的空间可以构成一个比所允许的空闲块大的空间，就freelist上这个空闲块的这个节点替换成这个剩下空闲块，如果比所允许的最小空闲块还要小，就直接删除其中的数据。

![](http://i.imgur.com/pVo9kXO.png)
3. _Heap_Extend扩展堆
rtems中的堆基本上就只有两个：一个是workspace一个是rtems_region
堆扩展基本上只用在rtems region中，而且唯一一种的方式就是在堆的末尾地址上继续增加空间，即将堆（堆从低地址到高地址扩展）的最高地址增加

4. _Heap_Resize_block改变堆内存块大小
主要是当前堆内存块和下一个堆内存块之间的合作来达到扩大或者缩小堆内存块的目的的
如果resize的内存块的大小大于当前内存块，则向下一个内存块要一部分空间进行填充，如果下一个内存块中不空闲则不能resize，不够就下一个
如果resize的内存块的大小小于当前内存块，则把多出的内存块分给下一个内存块或者直接分割成一个新的节点

5. _Heap_Free释放堆内存块
给的地址合法的情况下，看当前释放的内存块是否能够和前面或者后面的内存块合并，如果能则合并，如果不能，改变block属性，链起来

