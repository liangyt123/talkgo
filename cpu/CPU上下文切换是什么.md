# CPU上下文

多进程竞争会导致平均负载增加？增高的原因是什么？

答案就是cpu上下文切换。linux是多任务系统，它支持远大于cpu数量的任务同时运行，实际是cpu流片分配给不同的进程，造成多任务的错觉。

在运行每个任务的时候，cpu都要知道程序在入口点在哪，即是从哪里加载以及在哪里运行。这需要cpu寄存器以及程序计数器。这两个是cpu运行任何任务前的必要条件。所以也叫cpu上下文。

# CPU上下文切换

实际就是把前一个任务的寄存器以及程序计数器存储起来，并进行新的任务的载入寄存器与pc计数器。然后跳转到程序计数器所指的位置，再加以执行。

# CPU上下文切换的种类

上下文切换都会让cpu平均负载增加，那具体的CPU上下文切换的种类是什么呢？

## 系统调用

linux按照特权等级把进程的运行区间分为内核态与用户态。

Ring0(内核)具有最高权限

Ring3(用户)只能访问有限资源

内核态和用户态的转变是通过系统调用来实现的，如查看文件的内容的时候，就需要多次系统调用来完成，open文件，read()读取文件，write()将内容输出。

系统调用过程中发生了上下文切换的，因为程序要保存用户态信息然后跳到内核态信息进行处理，内核态处理完，把用户态信息重新载入，所以发生了用户态->内核态，以及内核态->用户态两次cpu上下文切换

但系统调用的时候不存在虚拟内存等用户态资源，也不会切换进程。所以系统调用被称为特权模式切换，而非上下文切换，但实际上cpu上下文切换避免不了

## 进程上下文切换

进程是由内核来管理和调度的，进程的切换只能发生在内核态，所以，进程的上下文不仅包括了虚拟内存，栈，全局变量等用户空间资源，还包括了内核堆栈，及存储器等内核空间状态

进程1：进程1上下文-保存--->进程2：进程2上下文加载

进程上下文切换次数过多，会导致CPU将大量时间耗费在寄存器，内核栈，虚拟内存等资源的保存还有恢复上，进而大大缩短进程运行时间。

linux通过TLB(translation lookaside buffer)来管理虚拟内存到物理内存的映射关系，虚拟内存更新后，物理内存也要更新，那么内存访问会变慢，多处理器中，缓存是被共享的，刷新缓存不知影响现在处理器进度还影响共享缓存的其他处理器的进程。

### 什么时候会切换进程上下文

linux给每一个CPU都维护了一个就绪队列，将活跃进程按优先级以及等待CPU时间来排序，然后优先级最后和等待CPU时间最长的程序进行调度。那么什么时候进程会被调度到CPU上呢？

**** 该进程执行完，就绪队列弹出一个新的元素

**** 该进程时间片耗尽

**** 系统资源不足时，进程挂起

**** 进程通过sleep函数主动挂起

**** 当有优先级更高的程序，当前程序会被挂起

**** 硬件中断，转向中断服务程序

## 线程上下文切换

线程是调度的基本单位，而进程是资源拥有的基本单位，内核的调度对象是线程，进程为线程提供虚拟内存，全局变量等。那么线程的上下文切换分为两种，两线程不属于相同进程，两线程不属于同一进程。

**** 线程切换不属于相同进程，那么其切换和进程切换一样

**** 前后线程属于相同进程，此时虚拟内存是共享的，只需要切换线程的私有数据，寄存器等不共享的数据

## 中断上下文切换

为了响应硬件中断事件，需要中断上下文切换来打断正常程序的处理。但中断不涉及用户态的上下文切换，中断程序都是很简短的，以便很快结束，并且比进程有更高的优先级。

# 怎么查看系统的上下文情况

通过vmstat工具能查看系统的上下文情况。关注有4个项cs,in,r,b：

1.cs 分别是每秒上下文切换次数

2.in则是每秒中断次数

3.r是就绪队列长度

4.b是处于不可中断睡眠状态的进程数

```shell
root@iZm5e78t3hp8rw0t2g09ikZ:~# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 183856 182112 1178604    0    0     0     2    1    2  1  1 99  0  0

```

但vmstat只给出了系统的上下文的大体情况。通过pidstat -w 可以查看每个进程的上下文切换情况。

```shell
root@iZm5e78t3hp8rw0t2g09ikZ:~# pidstat -w
Linux 4.4.0-151-generic (iZm5e78t3hp8rw0t2g09ikZ) 	05/31/2020 	_x86_64_	(1 CPU)

08:48:03 AM   UID       PID   cswch/s nvcswch/s  Command
08:48:03 AM     0         1      0.03      0.00  systemd
08:48:03 AM     0         2      0.00      0.00  kthreadd
```

其中cswch/s是每秒自愿上下文切换次数，nvcswch是非自愿上下文切换次数

****自愿上下文切换，是指程序无法获取自愿 导致的上下文切换，I/O 内存不足

**** 非自愿上下文切换，是进程由于时间片已到等原因，被系统强制调度，进而上下文切换

案例情况：

使用sysbench 模拟多线程并发的瓶颈

```shell
root@iZm5e78t3hp8rw0t2g09ikZ:~# sysbench  --num-threads=10 --max-time=300 --test=threads run
sysbench 0.4.12:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 10

Doing thread subsystem performance test
Thread yields per test: 1000 Locks used: 8
Threads started!

```

同样通过vm查看，上下文交换情况，就绪队列情况等信息

```shell
root@iZm5e78t3hp8rw0t2g09ikZ:~# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 176576 182124 1179700    0    0     0     2    1    0  1  1 99  0  0
 0  0      0 176608 182124 1179700    0    0     0     0  295  963  0  1 99  0  0
 4  0      0 176608 182124 1179700    0    0     0     0  274  919  1  0 99  0  0
 0  0      0 176608 182124 1179700    0    0     0     0  245  864  1  0 99  0  0
 8  0      0 175832 182124 1179700    0    0     0     0  314 144724  3 14 83  0  0
 7  0      0 175832 182124 1179700    0    0     0     0  471 959787 19 81  0  0  0
 9  0      0 175832 182124 1179700    0    0     0    12  421 806898 16 84  0  0  0
 8  0      0 175832 182124 1179700    0    0     0     0  494 962662 14 86  0  0  0

```

上面可以看出多线程模拟后cs明显增加，就绪队列变成了8  多余cpu个数1，有了大量的竞争。us和sy使用了达到了100%,sy占大头，表示CPU主要被内核占用，in也上升了，说明中断也是个占用cpu的问题，之后还是用pidstat进行查看

```shell
root@iZm5e78t3hp8rw0t2g09ikZ:~# pidstat -w -u 1
Linux 4.4.0-151-generic (iZm5e78t3hp8rw0t2g09ikZ) 	05/31/2020 	_x86_64_	(1 CPU)

09:48:38 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
09:48:39 AM     0     20342   21.69  101.20    0.00  122.89     0  sysbench

09:48:38 AM   UID       PID   cswch/s nvcswch/s  Command
09:48:39 AM     0         3      1.20      0.00  ksoftirqd/0
09:48:39 AM     0         7     30.12      0.00  rcu_sched
09:48:39 AM   110       973      2.41      0.00  chronyd
09:48:39 AM     0      1430      1.20      0.00  aliyun-service
09:48:39 AM     0     15888     12.05      0.00  AliSecGuard
09:48:39 AM     0     19746      1.20      0.00  kworker/0:2
09:48:39 AM     0     19879      1.20      0.00  kworker/u2:0
09:48:39 AM     0     20353      1.20      1.20  pidstat
09:48:39 AM     0     21400     12.05      0.00  AliYunDun


```

可以看到cpu的上升是由sysbench决定的，因为cpu有超频的 所以上面的显示是122.89%

看pidstat的cswch与nvcswch相加也就几十，远远未到vm的cs的几十万，这是怎么回事呢？

而实际是上面的命令是进程的上下文切换情况，pidstat加上-t 参数后 就可以看到线程的cs情况了：

```shell
Average:        0         -     20357      1.04    554.83  |__pidstat
Average:        0     20358         -      0.09      1.61  sysbench
Average:        0         -     20358      0.09      1.61  |__sysbench
Average:        0         -     20359   7717.42  51596.40  |__sysbench
Average:        0         -     20360   7972.73  56406.44  |__sysbench
Average:        0         -     20361   7593.47  55318.09  |__sysbench
Average:        0         -     20362   6582.48  56550.38  |__sysbench
Average:        0         -     20363   7682.95  57191.86  |__sysbench
Average:        0         -     20364   5561.93  35829.92  |__sysbench
Average:        0         -     20365   7822.73  54132.39  |__sysbench
Average:        0         -     20366   6885.98  56087.88  |__sysbench
Average:        0         -     20367   9286.55  52572.82  |__sysbench
Average:        0         -     20368   6429.64  55901.42  |__sysbench

```

上面看出sysbench的主线程（进程）切换次数不多，但线程切换的次数很多，所以上面案例的情况是 sysbench线程过多。

上面提到的中断也会使负载增加，因为中断是在内核态的，pidstat只是进程的性能分析工具，所以要查看内核态的信息，通过观察watch -d cat /proc/interrupts来看到rescheduling interrupts的变化，这里的中断过高还是多任务调度的问题。


# 用户态到内核态的切换情况
系统调用
这是用户态进程主动要求切换到内核态的一种方式，用户态进程通过系统调用申请使用操作系统提供的服务程序完成工作。比如前例中fork()实际上就是执行了一个创建新进程的系统调用。

而系统调用的机制其核心还是使用了操作系统为用户特别开放的一个中断来实现，例如Linux的int 80h中断。

用户程序通常调用库函数，由库函数再调用系统调用，因此有的库函数会使用户程序进入内核态（只要库函数中某处调用了系统调用），有的则不会。

异常
当CPU在执行运行在用户态下的程序时，发生了某些事先不可知的异常，这时会触发由当前运行进程切换到处理此异常的内核相关程序中，也就转到了内核态，比如缺页异常。

外围设备的中断
当外围设备完成用户请求的操作后，会向CPU发出相应的中断信号，这时CPU会暂停执行下一条即将要执行的指令转而去执行与中断信号对应的处理程序，

如果先前执行的指令是用户态下的程序，那么这个转换的过程自然也就发生了由用户态到内核态的切换。比如硬盘读写操作完成，系统会切换到硬盘读写的中断处理程序中执行后续操作等。

这3种方式是系统在运行时由用户态转到内核态的最主要方式，其中系统调用可以认为是用户进程主动发起的，异常和外围设备中断则是被动的。

#### 结论

自愿上下文切换过多是指进程在等待

非自愿上下文切换过多是进程在强制调度，说明在争抢CPU，CPU的确成了瓶颈。

