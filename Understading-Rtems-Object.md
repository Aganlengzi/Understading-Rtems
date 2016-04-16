##Understading-Rtems-Object

RTEMS 设计了一种对象模型，以便为核心层和系统服务层的各种组件提供一致和安全的访问手段。这
种对象模型的具体实现是位于核心层的 object handler 组件，它可以完成如下目标：
* 提供一种统一的机制来表示系统资源;
* 提供了一种对象命名方法，可以方便地实现对不同类型资源的扩展命名；
* 提供了一种统一的访问支撑机制来使用系统资源；
* 提供一种统一的机制来审计对象的使用情况，为系统管理对象的使用提供帮助。

对象的命名方法、对象的组织方法、基于类的对象管理方法和具体的实现进行更深入分析

对象的命令是由应用程序命名的，在RTEMS中存储管理和使用的是对象的ID
```
31 30 29 28 27|26 25 24|23 22 21 20 19 18 17 16|15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 0 
class         |API     |node                   |index
```
class比如message watchdog...
API比如classic api...
node处理器节点，没想到可以有这么多
index前面三个决定的对象种类的对象个数

对象的组织
```
struct Chain_Node_struct {
Chain_Node *next;      //链上的下一个节点
Chain_Node *previous;  //链上的前一个节点
};
```
```
typedef struct {
Chain_Node *first;           //链上的第一个节点
Chain_Node *permanent_null;  //链上的空节点
Chain_Node *last;            //链上的最后一个节点
} Chain_Control;
```
Chain_Control中first指向第一个节点，last指向最后一个节点
链表中第一个节点的previous指向Chain_Control的first，
链表中最后一个节点的next指向Chain_Control的NULL

在使用时，RTEMS充分利用了在linux中也广泛被采用的C语言实现继承的方法
即将基本的数据类型内嵌在派生出来的结构体中，并且将其放在派生结构体的最开始部分，
在使用的时候，可以方便通过类型转换来利用基础结构体的操作方法进行整个派生结构体和对象的操作
```
Objects_Control结构体
typedef struct {
Chain_Node Node;    // 一个对象的链节点
Objects_Id id;        //对象的 ID
Objects_Name name;    //对象的名字
} Objects_Control;
```
系统中通过Objects_Information对系统中所有的对象进行管理
在系统初始化的时候，定义了一张能够管理系统中可预见的所有类型对象的Objects_Information类型二维数组_Objects_Information_table[ the_api ][ the_class ]，在这个数组中，the_api和the_class相当于是下标，指示在系统创建某种类型的服务的时候（比如说timer，系统相当于用这张表将所有的对象资源管理起来）OBJECTS_CLASSIC_API为2，TIMER也为2
那么classic的timer服务的管理信息就是_Objects_Information_table[2][2]中保存的指针，而这个指针指向的是Objects_Information类型的_Timer_Information，其中保存了用户配置和系统配置的信息。其结构如下：
```
typedef struct {
Objects_APIs the_api;           //对象所属的 API 类
uint16_t the_class;             //对象在所属 API 类下的属性类
Objects_Control **local_table;  //符合此类的本地对象列表
Objects_Name *name_table;       //本地对象名字的列表
Chain_Control Inactive;         //没有分配的对象列表
void **object_blocks;           //同类的对象列表所处内存块
#if defined(RTEMS_MULTIPROCESSING)
Chain_Control *global_table;    //属于此类的全局对象列表
#endif
......
} Objects_Information;
```
Objects_Information中包含的信息用于某类对象的管理，其中包含了很多信息。
所有属于同一类的对象保存在一个连续地址空间的内存block中（object_blocks）。且此内存块的最后部分还保存了对象的名字列表。
所有未被分配（即空闲的）对象由Inactive 链表管理，
所有已经被使用的对象可以由 local_table 列表（基于对象的 ID 中的 index）来查询。

大致的过程：
首先通过_Objects_Information_table可找到对应类（API类+属性类）的Objects_Information，而通过Objects_Information的local_table 和 global_table 这两个Chain_Control 就可以找到属于这个类的已经分配了的所有对象，通过object_blocks可以访问属于这个类的没有分配的对象。
在Rtems中上述的过程是通过其对象管理定义的各种接口函数实现的。


















