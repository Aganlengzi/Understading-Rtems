##Understading-Rtems-Heap
区域管理比分区管理灵活。
把一个物理上邻接的内存空间通过用户自定义的边界划分成大小不同的段（ Segment），这些段可以动态的被分配和回收
而且内存的控制结构（区域控制块）在回收已使用的段（ segment）时，控制块会通过相邻数据块合并的方式来尽量维护大的空闲空间。这使
得区域管理的系统开销降低到最小。 而当应用程序发现一个区域的内存空间不够时，可以调用rtems_region_extend 函数来扩展该区域的大小。当任务要求从某个区域获取空闲段而未成功时，可以立即返回，也可以采取多种等待策略。等待策略包括优先级等待、 FIFO 等待。在 FIFO 等待策略中又可分为有限等待和无限等待。这样可以最大限度地满足不同实时应用的内存管理需求。

区域管理的主要缺点就是会产生较多的内存碎片。

区域控制块
```
typedef struct {
Objects_Control Object;
Thread_queue_Control Wait_queue; 	//等待得到内存空间的线程等待队列
void *starting_address;              //开始地址
uint32_t length;                     //物理地址大小
uint32_t page_size;                  //页大小
uint32_t maximum_segment_size;       //最大段大小
rtems_attribute attribute_set;       //属性
uint32_t number_of_used_blocks;      //已分配了的块数
Heap_Control Memory;                 //内存堆
} Region_Control;
```
其中的wait_queue是一个非常重要的成员变量，region允许系统中暂时得不到内存的任务等待

###//操作
1. _Region_Manager_initialization：初始化，这个和之前讲过的各种partition的相类似，都是统一在系统的object管理中的。

2. rtems_region_create：用给定起始地址、长度、名字等创建一个区域
创建过程和partition的创建过程十分类似，但是因为region的控制块中的线程等待队列wait_queue的存在，在其数据结构初始化的时候需要对wait_queue进行初始化，并设定其是按照优先级还是FIFO进行排队等待。其中wait_queue的数据结构如下：
```
typedef struct {
  union {
    Chain_Control Fifo;          
    Chain_Control Priority[TASK_QUEUE_DATA_NUMBER_OF_PRIORITY_HEADERS];
  } Queues;
  Thread_blocking_operation_States sync_state;
  Thread_queue_Disciplines discipline;
  States_Control           state;
  uint32_t                 timeout_status;
}Thread_queue_Control;
```
关于等待队列的初始化结果是创建一个Fifo类型的链表（比较简单调用_Chain_Initialize_empty就可以了）
还是一个按照优先级的链表，可以看到上面这是一个数组，所以要对代表每个优先级的数组项都建立一个链表，每个数组项都调用_Chain_Initialize_empty完成初始化。

3. rtems_region_get_segment：从给定区域中分配段
从指定的区域中region获取满足请求大小（可能比请求的要大）的空闲内存块，如果有足够的内存，分配成功
如果没有
1）调用任务将会永远等待空闲内存块
2）如果指定RTEMS_NO_WAIT就不等待，直接返回错误码
3）任务返回前指定等待时间
注意等待的任务在没有设置RTEMS_NO_WAIT的情况下是要放到之前的wait_queue中的
```
rtems_region_get_segment
{
     _Region_Allocate_segment
     {
          _Region_Allocate //因为这里的region的memory本身就是一个heap_control，本身就是一个堆，那么就是调用堆的分配函数来进行的
          {                          //只不过Rtems对这个heap进行了封装，封装之后的region支持在请求不到的时候进行等待（应该是最主要的封装功能吧）
               _Heap_Allocate_aligned_with_boundary
          }
     }
}
```
如果内存区被删除，等待任务将会得到错误码。通过_Region_Process_queue来完成，在每次rtems_region_return_segment 成功的时候
都会调用这个函数来检查等待队列中的任务的情况。如果满则条件，为其分配内存。

4. rtems_region_return_segment：将段释放回区域
和rtems_region_get_segment函数主要流程是相似的，只不过这里通过调用_Region_Free_segment实际调用的是_Heap_Free函数完成堆的内存释放操作
与前面的任务加入等待队列相对应，这里需要给等待队列中任务获得内存的激励
每次rtems_region_return_segment 成功的时候
都会调用这个函数来检查等待队列中的任务的情况。如果满则条件，为其分配内存。

5. rtems_region_delete：对于没有段在被分配的区域，实施删除操作
判断当前的这个region是没有segment被使用的，即判断the_region->number_of_used_blocks是否为0
_Objects_Close和 _Region_Free（最终是调用_Objects_Free）函数完成从系统的对象管理中删除相关的信息和释放相应的内存

 rtems_region_extend：扩展段长可变的区域
 rtems_region_ident：得到与给定区域名对应的 ID
 rtems_region_get_information：得到给定区域的信息
 rtems_region_get_free_information：得到区域中空闲块的信息
 rtems_region_get_segment_size：得到段长
 rtems_region_resize_segment：改变给定段的大小到给定值

