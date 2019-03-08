---
layout:     post
title:      "Linux的进程、线程以及调度（二）"
subtitle:   " \"fork，vfork, thread, 进程0, wait_queue, SUBREAPER\""
date:       2019-03-02
author:     "Songze Lee"
header-img: "img/post-bg-2019.jpg"
tags:
    - Linux的进程、线程以及调度
---
> Linux的进程、线程以及调度（二）

# Linux的进程、线程以及调度（二）

## 1. fork vfork

### 1.1 fork
fork是创建的一个进程，当有1个进程p1，一旦进入了fork，fork体内就出现了p1和p2，p1和p2
都会从这个fork体内返回，所以fork一进去就会变成两个进程。如何变成2个呢？
p1是一个task_struct,p2也是一个task_struct,在内核里面是2个task_struct,在内核的调度
算法层面上，只要看到一个task_struct它就可以调度，linux内核的调度器认得的就是task_struct,
看到一个task_struct就可以去调度，去执行。当p1把p2 fork出来的时刻，p1（task_struct）
是一个资源的封装单位，在p1里面有mm struct,fs struct,files struct,signal描述资源，
p2一旦创建出来它也需要有mm struct,fs struct,files struct，signal来描述p2的各种资源。
当p1把p2创建出来的时刻，p2是p1的new born，linux认为p1和p2具有某种内在的联系，p1就将
描述自己的资源结构体对拷给p2。p1的fs是来描述它的root和cwd（当前工作路径），p2创建出来
和p1的root和当前工作路径是一样的。新生的进程和父进程刚开始的资源是完全一样的，但是既然
是两个进程，区分进程的标志是资源不同，刚开始父进程和子进程资源一样，只有谁动资源(任何
修改)，就要分裂。举个例子，p1的files有自己打开的文件列表，如打开了1，2，3，刚fork出来
p2，也能看到文件列表1，2，3，但是p2后面打开文件4，那么p1是没有打开文件4的，这样p1和p2
的文件资源就不一样了。所以，当一个进程刚fork出来会执行一个资源的copy，但是任何修改都会
造成分裂，如chroot改变了根路径，open了新文件，写过内存memory，执行了新的mmap，做了信
号的绑定sigaction等等。

![](/img/process/fork_copy.jpg)

### 1.2 写时拷贝技术
资源都好分裂，唯一不好分裂的是内存资源，因为内存资源需要做写时分裂，需要有个办法来探测
谁在写哪一片内存。下面介绍linux著名的技术Copy-on-write（写时拷贝）,简称COW。

先看一个例子 cow.c
``` c
#include <sched.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int data = 10;

int child_process()
{
	printf("Child process %d, data %d\n",getpid(),data);
	data = 20;
	printf("Child process %d, data %d\n",getpid(),data);
	_exit(0);
}

int main(int argc, char* argv[])
{
	int pid;
	pid = fork();

	if(pid==0) {
		child_process();
	}
	else{
		sleep(1);
		printf("Parent process %d, data %d\n",getpid(), data);
		exit(0);
	}
}
```
--------------------------------------------------------------------------------
分析:全局变量data初始为10，fork后父子进程资源一样，所以子进程打印data为10，然后对
data进行修改为20，此时子进程资源因为修改而分裂，打印为值为20，所以父进程休眠1s后打印
data的值10。
结果为:
```
Child process 82655, data 10
Child process 82655, data 20
Parent process 82654, data 10
```
分裂的原理如下图：
![](/img/process/cow.jpg)

--------------------------------------------------------------------------------
- 最开始只有一个进程p1，假设data的虚拟地址virt1，物理地址为phy1，virt1通过mmc找到物理
地址phy1，原则上data是数据段权限是R+W；
- 一旦fork出p2后，p1和p2的虚拟地址和物理地址还是一样的为vir1和phy1，看到的data是同一份
data，这时候把data对应的页表权限改为RD-ONLY，
- 一旦页表所对应的权限改为RD_ONLY,cpu去写，无论是p1或p2去写RD-ONLY这块，cpu都会收到
page fault（缺页中断），上面例子中时子进程p2先去写，是写不成功的，地址权限是只读的，但
cpu会收到缺页中断后给写的进程p2从内存条里面申请新的内存，p2就会得到新的物理地址phy2，
linux会把老的内存phy1拷贝到新的phy2，然后修改进程的页表，把子进程p2的virt1指向phy2，
这时父子进程看到的虚拟地址是一样的，但是看到的data物理地址已经不一样了，之后linux会把两
个进程的页表权限都改成R+W，父子进程就可以再去写data了。

copy-on-write依赖于MMU，如果cpu中没有MMU，fork是不能工作的。如uclinux里面，自从linux2.6
以后不需要uclinux，本身就支持uclinux。

### 1.3 vfork
没有mmu的cpu是无copy-on-write，也就没有fork只有使用vfork。vfork和fork区别是:vfork父进程
阻塞直到子进程发生exit或者exec。fork 是子进程p2的mm struct,fs struct,files struct资源会
对拷重新来一份，vfork有很大的不一样，p1的mmc struct不再对拷给p2，p2 的task_struct的mm指
针直接指向（等于）p2 mm指针，mmc指针来描述内存资源，p1和p2的mm指针都指向同一个内存资源，
p1的mm指针是来描述p1的内存资源,p2的mm指针是来描述p2的内存资源，但现在指向同一个mm struct,
就证明p1的内存资源就是p2的内存资源。所以vfork是会clone vm的。

![](/img/process/vfork_copy.jpg)
上面fork代码改为vfork，结果就是10，20，20

## 3. linux线程的实现本质
同样把上面方式放大，pthread_create去创建线程时本质上是去调linux clone，p2的资源不会重新来
一份，都指向p1的资源。完成共享了资源。但是p1和p2在内部都是task_struct,都可以被调度。
![](/img/process/thread_copy.jpg)

### 3.1 进程、线程和“人妖”
我们可以让p1和p2只clone一部分资源，如果是进程要求p1和p2不共享资源，线程要求p1和p2共享所有
资源，这种情况就处于进程和线程的零界态。所以在linux task struct 和task struct之间有三种
状态：进程，线程，“人妖”。
![](/img/process/clone.jpg)

### 3.2 PID和TGID
linux 一个多线程的程序在内核里面每个都是一个独立的tast struct，所以在内核里面必然有各自的
pid，但是POSIX标准要求一个进程有多个线程必须向上面看起来是一个整体，也就是每个线程getpid都
应是同一个pid，linux就有个技巧，用假的TGID，如下图，p1通过pthread_create创建出p2，p3，p4,
让P1的TGID为PID1，P2，p3，p4都为PID1。其实在内核空间里面PID都不相等，但是在用户空间getpid()
时实际是拿的TGID。但是还是有办法拿到真正的PID。如下代码例子thread.c

``` c
#include <stdio.h>
#include <pthread.h>
#include <stdio.h>
#include <linux/unistd.h>
#include <sys/syscall.h>

static pid_t gettid( void )
{
	return syscall(__NR_gettid);
}

static void *thread_fun(void *param)
{
	printf("thread pid:%d, tid:%d pthread_self:%lu\n", getpid(), gettid(),pthread_self());
	while(1);
	return NULL;
}

int main(void)
{
	pthread_t tid1, tid2;
	int ret;

	printf("thread pid:%d, tid:%d pthread_self:%lu\n", getpid(), gettid(),pthread_self());

	ret = pthread_create(&tid1, NULL, thread_fun, NULL);
	if (ret == -1) {
		perror("cannot create new thread");
		return -1;
	}

	ret = pthread_create(&tid2, NULL, thread_fun, NULL);
	if (ret == -1) {
		perror("cannot create new thread");
		return -1;
	}

	if (pthread_join(tid1, NULL) != 0) {
		perror("call pthread_join function fail");
		return -1;
	}

	if (pthread_join(tid2, NULL) != 0) {
		perror("call pthread_join function fail");
		return -1;
	}

	return 0;
}

```
结果如下:
```
thread pid:159429, tid:159429 pthread_self:140219652781888
thread pid:159429, tid:159430 pthread_self:140219644491520
thread pid:159429, tid:159431 pthread_self:140219636098816
```
top命令（进程视角）看到的是TGID
![](/img/process/top.jpg)
159429为主线程的PID

top命令（线程视角）
![](/img/process/top-H.jpg)
这里159430和159431是在内核中真正的PID

## 4. 进程0和进程1
--------------------------------------------------------------------------------
既然每一个进程都是被创建出来的，那么第一个进程是怎么来的？init进程是被0进程创建出来的。
0进程可以认为是linux开机时的一个线索。
![](/img/process/process0.jpg)
在pstree中看不到0进程，因为0进程在内核把1进程fork出来后就退化成IDLE进程，IDLE进程是
linux内核特殊的调度类，所有的进程都停止或睡眠后，就去调度0进程跑。进程0一跑就把cpu进
入睡眠态，cpu 这时就处在等中断的状态，除非来一个中断才会被唤醒。

## 5.进程的睡眠和等待队列

睡眠分为深度睡眠和浅度睡眠，深度睡眠指必须等到资源才会醒，浅度睡眠指资源来了也会醒，信号
(singal)来了也会醒。

睡眠在linux主要借助等待队列数据结构，等待队列非常类似设计模式中发布者-订阅者模式，在linux
中进程都要等一个资源，多个进程等待资源可以将它们挂在一个等待队列上，资源来了，唤醒等待队列，
进程就自己醒。
![](/img/process/wait_queue.jpg)
## 6.6.孤儿进程的托孤，SUBREAPER

PR_SET_CHILD_SUBREAPER是linux3.4加入的新特性。把它设置为非零值，当前进程就会变成subreaper，
就会像1号进程那样收养孤儿进程。
![](/img/process/subreaper.jpg)
linux托孤有两种可能性。要么托付给init进行，要么托付给subreaper。如下图
![](/img/process/subreaper2.jpg)
p3是p2的子进程，p2死了，p3会往上找subreaper，找不到直接托付给init。p4死了，p5向上找到
subreaper，就托付给subreaper。如下例子：
```c
#include <stdio.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
	pid_t pid,wait_pid;
	int status;

	pid = fork();

	if (pid==-1)	{
		perror("Cannot create new process");
		exit(1);
	} else 	if (pid==0) {
		printf("child process id: %ld\n", (long) getpid());
		pause();
		_exit(0);
	} else {
		printf("parent process id: %ld\n", (long) getpid());
		wait_pid=waitpid(pid, &status, WUNTRACED | WCONTINUED);
		if (wait_pid == -1) {
			perror("cannot using waitpid function");
			exit(1);
		}

		if(WIFSIGNALED(status))
			printf("child process is killed by signal %d\n", WTERMSIG(status));

		exit(0);
	}
}

```
结果:
```
parent process id: 164187
child process id: 164188
```
![](/img/process/pstree.jpg)
![](/img/process/pstree2.jpg)

