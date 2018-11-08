---
title: Kernel sync and async
date: 2018-11-08 08:47:55
tags: KernelAPI
categories: Kernel
---

# 0.介绍
kernel有很多的同步和异步机制，做简单整理，力求能够熟练使用。

<--more-->
# 1.同步机制

* `并发`：多个执行单元同时被执行
* `竞态`:并发的执行单元对共享资源(硬件资源和软件上的全局变量等)的访问导致竞争状态。
并发与竞态。

假设有2个进程试图同时向一个设备的相同位置写入数据，就会造成数据混乱。处理并发常用的技术:`加锁`或者`互斥`，即确保在任何时间只有一个执行单元可以操作共享资源。在Linux内核中主要通过`semaphore`机制和`spin_lock`机制实现。

## 1.1 信号量
Linux内核信号量在概念和原理上与用户信号量一样的，但是它不能在内核之外使用，`它是一种睡眠锁`.

如果有一个任务想要获得已经被占用的信号量时，信号量会将这个`进程放入一个等待队列`，然后让其睡眠当持有信号量的进程将其释放后，处与等待队列中任务被唤醒，并让其获得信号量。

* 信号量在创建时需要`设置一个初始值`，表示**允许几个任务同时访问该信号量**保护的共享资源。初始值为1就变成互斥锁(Mutex)，即同时只能有一个任务可以访问信号量保护的共享资源。
* 当任务访问完被信号量保护的共享资源后，必须释放信号量。释放信号量通常把信号量的值加1实现，如果释放后信号量的值为非正数，表明有任务任务等待当前信号量，因此要唤醒信号量的任务。

信号量的实现也是与体系结构相关的，定义在`<asm/semaphore.h>`中，struct semaphore类型用类表示信号量。

1.定义信号量

```c
struct semaphore sem;
```

2.初始化信号量

```c
void sema_init(struct semaphore*sem,int val) 该函数用于初始化信号量设置信号量的初值，它设置信号量sem的值为val;
```

**互斥锁**

```c
void init_MUTEX(struct semaphore*sem);
```
该函数用于初始化一个互斥锁，即它把信号量sem的值设置为1.

```c
void init_MUTEX_LOCKED(struct semaphore*sem);
```
该函数也用于初始化一个互斥锁，但它把信号量sem的值设置为0,即一开始就处于已锁状态。

定义与初始化工作可由如下宏完成： 

* `DECLARE_MUTEX(name)`定义一个信号量name，并初始话它的值为1. 
* `DECLARE_MUTEXT_LOCKED(name)`定义一个信号量name，但把它的初始值设置为0,即创建时就处于已锁的状态。

3.获取信号量

```c
void down(struct semaphore*sem);
```

获取信号量sem,**可能会导致进程睡眠**，**因此不能在中断上下文使用该函数**。 该函数将把sem的值减1:

* 如果信号量的sem值为非负，就直接返回.
* 否则调用者将被挂起。**直到别的任务释放该信号量才能继续运行**

```c
int down_interruptible(struct semaphore*sem);
```
获取信号量sem.如果信号量不可用，进程将被置为`TASK_INTERRUPTIBLE`类型的睡眠状态。该函数返回值来区分正常返回还是被信号中断返回:

* 如果返回0，表示获得信号量正常返回
* 如果被信号打断，返回`-EINTR`.

```c
int dow_killable(struct semaphore*sem);
```
获取信号量sem,如果信号量不可用，进程将被设置为`TASK_KILLABLE`类型的睡眠状态. 注：`down()`函数已经不建议继续使用。建议使用`down_killable()`或d`own_interruptible()`函数。

4.释放信号量

```c
void up(struct semaphore*sem);
```
该函数释放信号量sem,即把sem的值加1,如果sem的值为非正数，表明有任务等待该信号量，因此唤醒这些等待者。

## 1.2 自旋锁
**自旋锁最多只能被一个可执行单元持有**。自旋锁不会引起调用者睡眠,如果一个执行难线程试图获得一个已经持有的自旋锁，那么线程就会一直进行忙循环，一直等待下去在那里看是否该自旋锁的保持者已经释放了锁，“自旋”就是这个意思。

1.初始化
```c
spin_lock_init(x);
```
该宏用于初始化自旋锁x，自旋锁在使用前必须先初始化。

2.获取锁

```c
spin_lock(x)
```
获取自旋锁lock，如果成功，立即获得锁，并马上返回，否则它将一直自旋在那里，直到该自旋锁的保持者释放。

```c
spin_trylock(x)
```
试图获取自旋锁lock，如果能立即获得锁，并返回真，否则立即返回假。它不会一直等待释放.

3.释放锁

```c
spin_unlock(x)
```
释放自旋锁lock,它与`spin_lock`或`spin_trylock`配对。`锁用完要进行释放`

## 1.3 信号量与自旋锁比较

* 信号量可能允许有多个持有者，而自旋锁任何时候只能允许一个持有者.当然也有信号量叫互斥信号量(只能一个持有者)，允许有多个持有者的信号量叫计数信号量
* 信号量适合保持较长时间，而自旋锁适合于保持时间非常短的情况，在实际应用中自旋锁控制代码只有几行，而持有自旋锁的时间也不会超过两次上下文切换的时间，因此线程一旦要进行切换，就至少花费出人两次，自旋锁的占用时间如果远远长于两次上下文切换，我们就应该选择信号量。

# 2.异步
主要使用队列来形成一种类似"缓冲"，从而产生异步。这个缓冲可以是数据，可以是"函数"。
## 2.1 等待队列wait_queue
可以使用等待队列来实现进程阻塞，在阻塞进程时，将进程放入等待队列，当唤醒进程时，从等待队列中取出进程。

1.定义和初始化

```c
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
```

可以使用宏来完成，定义和初始化过程

```c
DECLARE_WAIT_QUEUE_HEAD(my_queue);
```

2.睡眠

**a.有条件睡眠**

* `wait_event(queue,condition)`:当condition为真时，立即返回；否则让进程进入`TASK_UNINTERRUPTIBLE`模式的睡眠，并挂在queue参数所指定的等待队列上。
* `wait_event_interruptible(queue,conditon)`:当condition为真时，立即返回；否则让进程`TASK_INTERRUPTIBLE`的睡眠，并挂起queue参数所指定的等待队列
* `int wait_event_killable(wait_queue_t queue,condition)`:当condition为真时，立即返回；否则让进程进入`TASK_KILLABLE`的睡眠，并挂在queue参数所指定的等待队列上。

**b.无条件睡眠(老版本，建议不再使用)**

* `sleep_on(wait_queue_head_t *q)`:让进程进入不可中断的睡眠，并把它放入等待队列q.
* `interruptible_sleep_on(wait_queue_head_t *q)`:让进程进入可中断的睡眠，并把它放入等待队列q。

3.唤醒

* `wake_up(wait_queue_t *q)`:从等待队列q中唤醒状态为`TASK_UNINTERRUPTIBLE`,`TASK_INTERRUPTIBLE`,`TASK_KILLABLE`的所有进程。
* `wake_up_interruptible(wait_queue_t*q)`:从等待队列q中唤醒状态为`TASK_INTERRUPTIBLE`的进程。

## 2.2completion
内核编程中常见的一种模式是，**在当前线程之外初始化某个活动**，**然后等待该活动的结束**。内核中提供了另外一种机制——completion接口。**Completion是一种轻量级的机制**，他允许一个线程告诉另一个线程某个工作已经完成。实现基于等待队列。

1.结构与初始化

```c
struct completion {
	unsigned int done;     /*用于同步的原子量*/
	wait_queue_head_t wait;/*等待事件队列*/
} x;
```
函数
```c
void init_completion(x);
```

宏实现

```c
DECLARE_COMPLETION(work)
```

2.等待

```c
wait_for_completion(work);
```
3.完成

```c
completion(work);
```

## 2.3 工作队列work_struct
工作队列一般用来做滞后的工作，比如在中断里面要做很多事，但是比较耗时，**这时就可以把耗时的工作放到工作队列**。说白了就是`系统延时调度`的一个自定义函数。

工作队列的使用分两种情况:

* `利用系统`共享的工作队列来添加自己的工作，这种情况处理函数`不能消耗太多时间`,这样会影响共享队列中其他任务的处理;
* 另外一种是创建自己的工作队列并添加工作。

### 2.3.1 使用系统

1.声明工作函数

```c
void my_func();
```
2.创建一个工作结构体变量，并将处理函数和参数的入口地址赋给这个工作结构体变量

```c
//编译时创建名为my_work的结构体变量
//并把 函数入口地址 和 参数地址 赋给它;
DECLARE_WORK(my_work,my_func,&data); 
```

如果不想要在编译时就用`DECLARE_WORK()`创建并初始化工作结构体变量，也可以在程序运行时再用`INIT_WORK()`创建：

```c
//创建一个名为my_work的结构体变量，创建后才能使用INIT_WORK()
struct work_struct my_work; 

//初始化已经创建的my_work，其实就是往这个结构体变量中添加处理函数的入口地址和data的地址
// 通常在驱动的open函数中完成
INIT_WORK(&my_work,my_func,&data);
```

3.将工作结构体变量添加入系统的共享工作队列

```c
schedule_work(&my_work); //添加入队列的工作完成后会自动从队列中删除
```

### 2.3.2 创建自己的工作队列来添加工作

1.声明工作处理函数和一个指向工作队列的指针

```c
void my_func();
struct workqueue_struct *p_queue;
```

2.创建自己的工作队列和工作结构体变量(通常在`open`函数中完成)

```c
//创建一个名为my_queue的工作队列并把工作队列的入口地址赋给声明的指针
p_queue=create_workqueue("my_queue"); 

//创建一个工作结构体变量并初始化，和第一种情况的方法一样
struct work_struct my_work;
INIT_WORK(&my_work, my_func, &data); 
```

3.将工作添加入自己创建的工作队列等待执行

```c
//作用与schedule_work()类似
//不同的是将工作添加入p_queue指针指向的工作队列而不是系统共享的工作队列
queue_work(p_queue, &my_work);
```

4.第四步：删除自己的工作队列

```c
destroy_workqueue(p_queue); //一般是在close函数中删除
```
## 2.4 tasklet
小进程，主要用于执行一些小任务，对这些任务使用全功能进程比较浪费。也称为中断下半部，**在处理软中断时执行**。

1.初始化和结构体

```c
struct tasklet_struct{
	struct tasklet_struct* next;
	unsigned long state;
	atomic_t count;
	void (*func)(unsigned long);
	unsigned long data;
};
```
函数：`tasklet_init(t,func,data)`

宏：
```c
DECLARE_TASKLET(name,func,data);//tasklet is scheduled for executation
//两种状态：TASKLET_STATE_SCHED
//TASKLET_STATE_RUN//tasklet is running(SMP only)
```
2.执行:`tasklet_schedule(t)`
3.销毁

函数：
宏：
## 2.5 工作队列和tasklet区别
Linux2.6内核使用了不少工作队列来处理任务，他在使用上和tasklet最大的不同是**工作队列的函数可以使用休眠**，而tasklet的函数是不允许使用休眠的。