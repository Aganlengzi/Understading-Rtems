##Understading-Rtems-Timer

###// 服务层Timer
Timer就是一个定时器，或者直观理解为闹钟，帮助任务定制在未来某一时刻执行某个操作（routine）
Timer server是一个任务级的定时器，实际上就是专门生成了一个Timer server任务来完成定时器的工作
这就要求这个任务的优先级要比其它所有的任务优先级都要高，并且不能被抢占

Timer和Timer server是上层服务，它们建立在底层提供的时间管理功能TOD和Watchdog的基础之上
Watchdog实际上提供的是一种相对的时序关系，通过链表中的位置决定，TOD提供的是一种绝对的度量

先看Timer，系统在初始化的时候，创建了_Timer_Information这个全局变量用于管理timer其中将所有的timer以Timer_Control组织成一个链表
####//Timer结构的初始化
```
RTEMS_initialize_data_structures()
{
     _RTEMS_API_Initialize()
     {
          _Timer_Manager_initialization()     //初始化Timer相关的所有数据结构
          {
               //初始化全局的_Timer_Information并放到_Objects_Information_table对应的位置
               //相当于建立了timer的基本数据结构，后面可以通过timer相关的函数进行定时器的创建、使用、和删除
          }
     }
}
```
####//Timer的操作
1. rtems_timer_create：创建定时器，由用户程序调用，这里创建的定时器只是创建相关的结构体并没有和时间服务程序（定时器到期后应该执行的程序绑定）
```
{
     _Timer_Allocate             //为定时器分配资源
     {
          _Objects_Allocate     //从_Timer_Information->inactive中分配节点，这里有个点，如果其中没有足够节点，要扩展这块内存
     }
     设置allocate的这个timer状态为TIMER_DORMANT即不再使用状态
     _Watchdog_Initialize      //初始化这个timer中的watchdog，这个watchdog被设置为WATCHDOG_INACTIVE，即它不在前面讲到的watchdog tick链和second链上
     _Objects_Open              //调用对象管理程序_Objects_Open，将timer对象注册到系统的对象管理中
     //这样这个timer就算是创建好了，等着被用吧（并没有开始使用）
}
```
2. rtems_timer_ident：根据timer的名字，名字是用户程序中用户取的，系统中是通过id来进行注册和管理的，后面需要用到

3. rtems_timer_fire_after：设置触发的相对时间。根据用户指定的timer的id设置ticks和到期执行的routine以及传递给routine的数据
```
{
     _Watchdog_Initialize（）//根据用户指定的timer的id设置ticks和到期执行的routine以及传递给routine的数据
     _Watchdog_Insert_ticks
     {
          _Watchdog_Insert     //将初始化好的watchdog插入到_Watchdog_Ticks_chain链中
     }
}
```
4. rtems_timer_fire_when：设置触发的绝对时间。根据用户指定的timer的id设置wall time和到期执行的routine以及传递给routine的数据
```
{
     _Watchdog_Initialize               //和上面的过程差不多
     _Watchdog_Insert_seconds     //由于墙上时间精度是秒所以这里放到了watchdog的second链上
}
```
5. rtems_timer_cancel：取消定时器，最终执行的是将timer的watchdog从链上接触并将其状态设置成非活跃状态
6. rtems_timer_reset：重置定时器，重置已有的定时器，interval和routine和之前的一样，只是将已经配置好的timer先停掉，然后再执行一次插入操作
7. rtems_timer_delete：把之前注册在对象_Timer_Information管理中的对象给删除掉，将timer的watchdog从相应链表上删除，利用_Timer_free将timer对象的内存收回

