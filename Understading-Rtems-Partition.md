##Understading-Rtems-Partition

Rtems中Partition为动态定长内存分配机制，即固定大小内存块的申请和释放的内存使用方式
一个分区就是把物理上邻接的内存区域划分成固定大小的缓冲区
以链表的形式进行管理，使用时，请求和释放都是以缓冲区为单位的。
####Partition manager的初始化过程：
```
_RTEMS_API_Initialize
{
     _Partition_Manager_initialization
     {
          //和前面的_Timer_Information的对象管理方式相类似
          这里将对象变成了全局的_Partition_Information
          所以要找到Partition相关的操作的话，应该是在提供给用户使用的应用程序接口了
     }
}
```
###分区控制块
```
typedef struct {
Objects_Control Object;
void *          starting_address;      //物理地址
uint32_t        length;                //分区大小
uint32_t        buffer_size;           //每个缓冲区大小
rtems_attribute attribute_set;         //属性
uint32_t        number_of_used_blocks; //已使用的缓冲区个数
Chain_Control   Memory;                //缓冲区链表
}
```
###//操作
1. _Partition_Manager_initialization：分区管理器初始化
上面已经说到，这个只是Rtems系统中对所有的功能性的全局变量（视为对象）的一种初始化方式，主要是建立其数据机构和在系统全局中的信息等。

2. rtems_partition_create：创建分区，由应用程序调用创建自己要使用的分区
创建一个用户命名的缓冲区定长的内存分区
```
{
     //检查参数合法性
     _Partition_Allocate     //分配分区控制块，按照用户指定的参数，起始地址，长度，缓冲块大小等对其进行赋值
     _Chain_Initialize       // 缓冲区链表进行初始化,此时Partition链表建立，节点个数为length/buffer_size
}
```
我看到的测试用例中是在应用的配置文件即system.h中首先申请可一个数组空间，数组空间的申请是指定标志让编译器按照字节对齐的方式申请的，然后传递给create的参数是数组名
这样就满足传递进去的是一块连续的物理地址空间，然后按照用户指定的方式进行分割

3. rtems_partition_ident：根据分区的name得到系统组织对象的id，然后后面的操作通过id来进行

4. rtems_partition_delete：需要注意的是分区中如果有缓冲区被使用是不能删除的

5. rtems_partition_get_buffer：按照指定的partition的id（通过rtems_partition_ident得到）分配一个空闲的buffer
就是从链表中找到一个空闲的节点返回，partition默认返回第一个空闲的节点，用户释放节点放在表尾。
```
rtems_partition_get_buffer
{
     _Partition_Get                      //得到partition的控制块指针，同时得到对象管理中的局部性
     _Partition_Allocate_buffer          //分配这个partition中的buffer块 
     {
          _Chain_Get
          {
               _Chain_Get_first_unprotected
          }
     }
}
```
6.  rtems_partition_return_buffer：这个操作和上面的get_buffer的过程十分相似，只是如上面所说，被释放的节点是被append到链表的表尾的。
