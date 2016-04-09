##Understading-Rtems-Watchdog-TOD

Rtems中的时间管理主要是通过以下两种方式进行的，一种是watchdog，另一种是TOD。下面对其主要
###//核心抽象层watchdog初始化
实际上就是完成了watchdog所用的变量的初始化，主要是两个链表
```
RTEMS_initialize_data_structures()
{
     _Watchdog_Handler_initialization
     {
        _Watchdog_Sync_count = 0;
		_Watchdog_Sync_level = 0;
		_Watchdog_Ticks_since_boot = 0;
		_Watchdog_Ticks_chain       //这条链上的watch_dog每次tick(时钟中断)都会被处理
		_Watchdog_Seconds_chain     //这条链上的watch_dog每second才会被处理
     }
}
```
####//watchdog操作
_Watchdog_Initialize:创建一个watch_dog并将其状态设置成 WATCHDOG_INACTIVE
_Watchdog_Insert_ticks：将创建的watch_dog插入到_Watchdog_Ticks_chain中
注意插入的时候要选择正确的插入点，即保证这个链表还是根据到期的先后顺序排列的，同时需要更新链表上受影响的节点的delta_interval
_Watchdog_Tickle（_Watchdog_Tickle_ticks或者_Watchdog_Tickle_seconds）：每个时钟间隔或者每秒更新链上的所有watchdog
_Watchdog_Adjust：根据指定的方向和间隔值更新指定的链表，
需要注意的是，链表的时间值的更新如果向前更新，即让watchdog定时边长比较方便，即将链表头中的delta_interval加上一定值就可以了
但是如果是模拟时间的增长的话，因为时间的增长可能会造成一些watchdog到期，那么就要处理到期watchdog的routine

###//核心抽象层TOD
提供了对时间的基本支持，主要完成了对系统启动后时间的更新维护，
```
rtems_initialize_data_structures
{
     _TOD_Handler_initialization
     {
          _TOD_Now          //这个值为当前时间
          _TOD_Uptime       //这个值为0   
          _TOD_Is_set       //为false
     }
}
```
####//TOD操作
```
rtems_clock_set             //由用户程序调用设置
{
     _TOD_Set
     {
          _Watchdog_Adjust_seconds   //前面说过这个函数是对精度为second的链表进行时间值调整的函数，如果让时间向后流的话可能引起watchdog的routine执行
          设置_TOD_Now为用户指定的时间，用户要传递参数进来
          设置_TOD_Is_set为true
     }
}
```
_TOD_Tickle_ticks     //就是下面讲到的在时钟中断到来的时候TOD的改变

###//watchdog和TOD在时钟到来时的更新操作
```
硬件时钟中断
{
     Clock_isr
     {
          rtems_clock_tick
          {
                _TOD_Tickle_ticks();
                {
                    更新_TOD_Uptime
                    _Watchdog_Ticks_since_boot += 1;
                    更新_TOD_Now
                    如果到秒级了，每一秒都要调用_Watchdog_Tickle_seconds函数一次
                    {
                         _Watchdog_Tickle( &_Watchdog_Seconds_chain ); //更新这个链表上的watchdog
                    }
                }
                _Watchdog_Tickle_ticks();
                {
                    _Watchdog_Tickle( &_Watchdog_Ticks_chain );//_Watchdog_Ticks_chain 在每个时钟到来的时候都会进行处理
                    此函数完成对watch_dog的delta_interval的减一操作
                    并且负责到期的watch_dog的移除操作和相应的watch_dog的routine执行
                    而此routine执行是有条件的，即只有在chain上的并且允许触发的watch_dog才能够被触发
                    因为watchdog 总是按照其事件发生时间的顺序，从早到晚排成一条链，所以在检查的时候只要从头开始检查，检查到未到时的结束就可以了
                    链中后面的节点记录相对于前一个节点发生的delta_interval，链首记录还有多少个tick发生
                }        
                _Thread_Tickle_timeslice();
                检查调度
          }
     }
}
```
