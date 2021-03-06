---
title: 北航操作系统Lab4挑战性任务
date: 2022-07-09 20:00:00 +/-TTTT
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [操作系统]     # TAG names should always be lowercase
author: lyh
---

## 题目概括

- 依照POSIX标准，在MOS中实现线程功能
  - 实现4个POSIX标准规定的函数：`pthread_create`,`pthread_exit`,`pthread_cancel`,`pthread_join`
  - 向用户空间提供接口
  - 线程间实现全地址空间共享
- 依照POSIX标准，实现信号量功能
  - 实现6个POSIX标准规定的函数：`sem_init`,`sem_destroy`,`sem_try`,`sem_wait`,`sem_trywait`,`sem_getval`
  - 完整支持无名信号量
- 编写对功能的测试程序，给出测试结果

## 实现情况

完成了题目要求，并具有几个特色：

- 支持一个进程开辟任意多个线程
- 额外实现了fork功能对线程的适配
- 信号量等待队列为先进先出

## 线程实现介绍

### 线程功能相关定义

- 定义TCB结构体，作为线程控制块，结构体内容为
	```c
	struct TCB {
		LIST_ENTRY(TCB) tcb_link; // 线程链表域
		u_int env_id;        // 所属进程编号
		struct Trapframe tf; // 保存寄存器堆
		u_int thread_status; // 线程状态，有FREE、RUNNABLE、NOT_RUNNABLE三种状态
		int joined_on;       // join：记录对方的线程号
		void **p_retval;     // join：自身要接收结果的地址
		void *retval;        // exit：结果存到这里
	};
	```
- Env结构体删除条目：
	```c
	struct Trapframe env_tf;        // trapframe放到TCB中
	u_int env_ipc_receiving;		// ipc接收状态标志
	```
- Env结构体增加条目：
	```c
	struct TCB_list tcb_list;  // 线程调度链表
	struct TCB *now;	// 当前运行的线程
	struct TCB *env_ipc_recving_thread;	// 处于ipc接收状态的线程
	```
- 内核中分配1024个TCB的空间，将其用tcb_free_list组织起来，并提供五个操纵TCB的函数：
	- tcb_init：将全部tcb插入tcb_free_list
	- tcb_alloc：分配一个tcb
	- tcb_number：获取tcb序号
	- tcb_get：根据序号获取tcb
	- tcb_free：释放tcb
- 新增4个系统调用：
	- sys_pthread_create：创建线程
	- sys_pthread_exit：结束本线程
	- sys_pthread_cancel：结束另一个线程
	- sys_pthread_join：等待至另一个线程结束


### 线程创建

- 创建进程的时候，给进程分配首个TCB
- 之后创建每个线程的时候，每次都再申请一个TCB
- 每个线程有自己的运行栈，0号线程的栈顶是USTACKTOP
- 每个运行栈占据1个页面，i号线程的栈顶是USTACKTOP - i×BY2PAGE
- 线程的其它设置包括pc、status等

### 线程运行

调度中维护env->now，以便得知当前运行的是哪个线程

##### 调度

- 依然使用进程调度链表，带优先级的时间片轮转算法
- 每个进程内部有线程调度链表，无优先级的时间片轮转算法
- 对于runnable状态的进程，查看其内部有没有runnable的线程
	- 如果有，就运行之
	- 如果没有，就将此进程放到队尾

##### join与exit

这俩是一对，为方便起见，称进行join的线程为A线程，被join的线程称为B线程，执行exit的也是B线程

从先后关系来看，存在两种可能的情况：

- 如果先join后exit：
	- A线程将接收返回值的地址保存到TCB中，然后进入阻塞
	- B线程在exit的时候需要把返回值填入A线程的接收地址，并唤醒A线程
- 如果先exit后join：
	- B线程只需把返回值填入TCB
	- A线程发现B线程已经结束，就不需要阻塞，直接从对方TCB中获得返回值

##### ipc

每个进程中只允许一个线程处于recv等待状态，额外记录等待线程的编号，send过来的时候唤醒这个线程即可

##### fork

分析的结果是，只改sys_env_alloc就可以，fork.c的内容完全不用改，cow机制挺完善的

复制env的时候也同步复制tcb，微调当前tcb的复制品

保证在子进程下次被调度的时候一定会选中复制品线程，而不是其它线程

### 线程结束

线程结束时，如果有其它线程在join，就把它们的p_retval中填上retval，并把它们唤醒

仅当进程的所有线程都结束后，进程才会结束，这是通过修改libos.c实现的

## 信号量实现介绍

起初打算在用户态实现，后来发现这样无法保证原子性，例如sem_wait()如果这样做:
```c
int sem_wait(Sem *s) {
    if (sem_not_valid(s))
        return EINVAL;
    if (s->val==0)
        syscall_yield();
    // !! 此处必须有原子性，否则信号量可能小于零 !!
    s->val--;
    return 0;
}
```

所以还是使用系统调用的方式。

- 定义环形等待队列Queue，以及相关的函数
	- 结构体定义：
		```c
		struct Queue_ {
			int list[10];
			u_int first;
			u_int last;
    	};
		```
	- 相关函数：
		- queueInit
		- queuePeek
		- queueEmpt
		- queueFull
		- queuePush
		- queuePop
- 定义Sem结构体：
  ```c
  struct Sem_ {
    u_int magic;  // 魔数
    u_int val;    // 信号量的值
    Queue q;      // 环形等待队列，自行实现
  };
  ```
- sem_init() sem_destroy()在用户态实现
- sem_wait() sem_trywait() sem_post()需要原子性，通过系统调用实现
- 信号量的等待队列是先进先出的
- 只要等待队列和线程的阻塞唤醒实现完毕，信号量实现起来就比较简单

## 测试

设计了5个测试，测试目的分别是：
1. **线程基本操作**：create、exit、join、cancel
2. **信号量基本操作**：init、wait、post
3. **哲学家进餐问题**：线程与信号量综合，检查不同调度策略的死锁存在性
4. **质数筛**：如果开启大量线程（50个以上），还能否正确分配、调度
5. **线程与fork综合**：是否实现fork对线程功能的适配

### 1 线程基本操作

本测试使用3个线程

- 主线程创建thread1，然后等待thread1结束，查看它的返回值，然后创建thread2
- thread1打印0-9999这10000个数，然后返回10086这个数
- thread2是一个死循环

测试结果显示：
- thread1能够正常运行，证明create是有效的
- 当thread1执行完毕，主线程才会恢复执行，也拿到了正确的返回值，证明join和exit是有效的
- 仅当对thread2执行cancel操作，进程才能结束，证明cancel是有效的


### 2 信号量基本操作

本测试使用3个线程与2个信号量

- 主线程：
  - 创建出2个线程，thread1和thread2
  - 初始化两个信号量s1=1和s2=0
- thread1是死循环，内容为
  - 先wait(s1)
  - 打印5行"thread111111"
  - post(s2)
- thread2也是死循环，内容为
  - 先wait(s2)
  - 打印3行"thread2"
  - post(s1)

预期效果为先打印5行"thread111111"，然后交替打印3行"thread2"与5行"thread111111"，测试结果符合预期

### 3 哲学家进餐

本测试使用6个线程与5个信号量

这是一个经典的PV操作问题：

> 一张圆桌上坐着5名哲学家，每两个哲学家之间的桌上摆一根筷子，桌子的中间是一碗米饭。哲学家们倾注毕生的精力用于思考和进餐，哲学家在思考时，并不影响他人。只有当哲学家饥饿时，才试图拿起左、右两根筷子（一根一根地拿起）。如果筷子已在他人手上，则需等待。饥饿的哲学家只有同时拿起两根筷子才可以开始进餐，当进餐完毕后，放下筷子继续思考。

如果每个哲学家都选择先拿左手的筷子，后拿右手的筷子，就可能每个哲学家都只拿起了左手的筷子，导致死锁

此问题有一个避免死锁的策略，就是偶数编号的哲学家先拿左手边的筷子，奇数编号的哲学家先拿右手边的筷子

3号测试模拟了这个问题，并尝试了上述的两种策略

为了更容易触发死锁，在问题中加一个设定，就是在哲学家拿起第一根筷子后，会思考一会，再尝试拿第二根筷子；另外设定每个哲学家都只会进餐100次，之后线程就结束

实际测试中，两种调度策略的效果都符合预期

### 4 质数筛

哇哈哈，从6.S081抄来的创意，从多进程改成了多线程

![](/assets/img/2022-07-09-MOS_Lab4_Challenge/%E8%B4%A8%E6%95%B0%E7%AD%9B%E7%A4%BA%E6%84%8F%E5%9B%BE.PNG)

本测试使用50+个线程，100+个信号量

- 主线程：
  - 创建1号线程，向其发送2-200的所有数 
- i号线程：
  - 收到的第一个数，认为是质数，打印出来
  - 接下来收到的所有数，用第一个数去除，如果不能整除，就发送到i+1号线程

测试结果表明，单个进程开100以下的线程一般不会出问题

### 5 线程与fork综合测试

就很简单地测了一下，主线程先pthread_create出一个新线程，再fork，新线程也确实执行了两次，说明问题不大，但在多次fork的时候存在问题，最后没时间了就没再管这块
