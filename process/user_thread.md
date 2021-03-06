用户级线程实现原理
=========================================


定义
------------------------------------
* 用户级调度,
* 可抢占
* 有优先级别
* 完全隔离



讲一下轻量级进程模型的实现原理
这个是蛮多人还是比较关注的。我之前比较少谈这个
但是我今天我们详细的谈一谈轻量级进程到底是怎么回事。先谈谈进程
所谓的进程到底是什么样的东西？其实进程本质上无非就是一个栈加上寄存器器的状态。
进程的切换怎么做呢？就是保存当前进程的寄存器
然后把寄存器修改为另外一个新进程的寄存器状态 这样相当于同时也切换了栈
因为栈的位置其实也寄存器维持的（ESP/EBP）。这个就是进程的概念
哪怕操作系统的内核帮你做的本质上也是这样。所以这些事情是在用户态一样可以做到
而不是不能做到。本质上来讲和函数的那个调用你可以认为也是差不多
因为函数的调用也是保存寄存器 只是相对少一些
至少不会切换栈。所以本质上讲其实是这个切换的成本是和函数调用是基本上差不多的
我自己测过 大概就是函数调用的10倍左右
基本还是在同样的数量级范畴。那么在这样一个轻量级进程的概念引入以后
实际上整个轻量级进程的程序物理上是怎么样的？底层其实还是线程池加异步IO
你可以把这个线程池中的每个线程想象成虚拟CPU（VCPU）。逻辑的轻量级进程（routine）的个数通常是远大于物理的线程数的
每个物理的线程同一个时刻肯定只有一个routine在跑
更多的routine是在等待当中的。但是这个等待中的routine有两种 一种是等IO的
就是说我把CPU交给他也干不了活 还有一种是IO操作已经完成
或者是自己本身并没有等任何前置条件
总之是可以参与调度的。如果某一个物理的线程（VCPU）它的routine主动的或者是因为IO触发了一个调度
把线程（VCPU）让出来 这个时候就可以让一个新routine跑在上面
也就是从等待当中并且可以满足调度的routine参与调度
按照某种优先级算法选择一个routine。所以轻量级进程调度原理是这样的 它是用户态的线程
然后有一个非强占式的调度机制 调度时机主要由IO操作触发。发生IO操作的时候
IO操作的函数是这样实现的：首先发起一个异步的IO请求
发起后把这个routine状态设置为等待IO完成 然后再出让CPU
这个时候也就触发调度器的调度 这个时候调度器就会看看有没有人等着调度
有它就可以切换过去。然后再IO事件完成的时候
IO完成后通常会有一个回调函数作为IO完成的事件通知
这个会被调度器接管 具体做什么呢？很简单就是把这个IO操作所属的routine设为Ready
可以参与调度了。因为刚刚它的状态是在等IO
就算调度到它也没有办法做事情。而 Ready 的话就是让这个routine可以参与调度。还有一种情况就是routine主动出让CPU
这种情况下routine的状态在切换的时候仍然是Ready的
任何的时间都可以切到它。以上几个基本上是非强占式的调度里面最基础的几个调度器触发的条件：IO操作、IO完成事件、主动出让CPU。但是其实在用户态的线程也可以实现强占式的调度
做法也是非常简单的 调度器起来一个定时器
这个定时器定时出发一个定时任务 这个定时任务检查每个正在执行当中的routine的状态
发现占CPU时间比较长就可以让它主动地让出CPU
这就就可以实现强占式的调度。所以哪怕在用户态
它可以完全实现操作系统进程调度所有做的事情。这就是轻量级进程的实现原理。
