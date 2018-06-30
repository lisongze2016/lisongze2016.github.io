---
layout:     post
title:      "Linux的进程、线程以及调度（一）"
subtitle:   " \"task_struct, 进程的生命周期, 僵尸, fork\""
date:       2018-06-30
author:     "Songze Lee"
header-img: "img/post-bg-2018.jpg"
tags:
    - Linux的进程、线程以及调度
---
> Linux的进程、线程以及调度（一）

# Linux的进程、线程以及调度（一）

## 1.1  task_struct

进程是一个资源分配单位。
linux下进程是用什么样的数据结构来描述它？
进程控制块 PCB（process control block），linux下的进程控制块是task_struct结构体。

```cpp
struct task_struct {
	pid_t pid;
	...
	struct mm_struct *mm;	/* 描述内存资源 */
	/* filesystem information */
	struct fs_struct *fs;	/* 描述文件系统资源 */
	/* open file information */
	struct files_struct *files;	/* 描述文件资源 */
	struct signal_struct *signal;
	...
}

```
![](/img/process/PCB.jpg)

### pid
pid的数量是有限的，32位系统查看pid最大值，cat /proc/sys/kernel/pid_max, 值为32768。

### fork 炸弹
```cpp
：(){:|:&};:
/* 函数名为冒号：，{}函数体里面调用自己，再创建一个管道创建一个新的进程调用自己并且后台执行，
分号；为函数结束，再调用自己，这个程序不断的创建新的进程导致pid最终用完。 */

```
### task_struct 被管理的数据结构
task_struct通过形成链表,树,哈希来被管理，这里task_strcuct 只是一个但是用多个数据结构从多个不同的侧面来描述它，自然在各种场景下访问都比较快，如遍历所有的进程就用链表，想知道进程的父就用树，想通过pid快速的检索一个进程就用哈希表，这是典型的以空间换时间的思路。
![](/img/process/task_struct.jpg)

## 1.2 Linux 进程生命周期(就绪、运行、睡眠、停止、僵死)

- 就绪态 <----> 运行态

进程一旦被fork出来就处于就绪态，一旦拿到的cpu就处于运行态，实际上在内核里面就绪和运行的状态标志是一样的。但是进程不能老是一直占有cpu，也可能让给别人，所以运行态也可能切回就绪态。有两种可能性，(1)时间片用完（分时调度），(2)时间片没用完但是被更紧急的事件抢占。

- 运行态 -----> 睡眠态

进程处于运行态经常需要等待资源，如串口，网络发包，等资源不可能一直占有cpu死等，必须把进程切换睡眠态。

睡眠分为深度睡眠和浅度睡眠，深度睡眠指必须等到资源才会醒，浅度睡眠指资源来了也会醒，信号(singal)来了也会醒。

- 睡眠态 ----> 就绪态

一旦等到了资源，切换就绪态

- 运行态 ----> 僵尸

进程刚死时会变成僵尸，在linux下任何一个进程死时，不是人间蒸发，都会变成僵尸。

- 运行态 ----> 停止态

停止态指进程在运行的过程中不去睡眠人为的让它停止，一般是发stop信号(ctrl +z ),或者是gdb attach debug调试。

- 停止态 ----> 就绪态

给停止态发送continue信号即可到就绪态。

![](/img/process/lifecycle.jpg)

## 1.3 僵尸是什么？
僵尸是什么？指的是用于描述它的数据结构task_struct还没有消失，但是进程所依附的资源都已经消失了，留下来的目的是让它的父进程wait4的api去等待它死，父进程通过wait4来等它时，僵尸才会消失，僵尸是非常短的临界状态，一个进程死了但父进程还没有来到及调它的wait4得情况，它就是僵尸，一旦父进程wait4他就直接人间蒸发掉了。
资源已经释放：无内存泄露等，task_struct还在：父进程可以查到子进程的死因(父进程wait 它的时候可以得到子进程的退出码)。
![](/img/process/zombie.jpg)

例子:
```cpp
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
#if 1
		/* define 1 to make child process always a zomie */
		printf("ppid:%d\n", getpid());
		while(1);
#endif
		do {
			/* 子进程死 waitpid就返回，status保存子进程退出原因 */
			wait_pid=waitpid(pid, &status, WUNTRACED | WCONTINUED);

			if (wait_pid == -1) {
				perror("cannot using waitpid function");
				exit(1);
			}

			if (WIFEXITED(status))
				printf("child process exites, status=%d\n", WEXITSTATUS(status));

			if(WIFSIGNALED(status))
				printf("child process is killed by signal %d\n", WTERMSIG(status));

			if (WIFSTOPPED(status))
				printf("child process is stopped by signal %d\n", WSTOPSIG(status));

			if (WIFCONTINUED(status))
				printf("child process resume running....\n");

		} while (!WIFEXITED(status) && !WIFSIGNALED(status));

		exit(0);
	}
}

```

调试:
```cpp
将if 1 改为if 0
窗口1：
$ ./a.out
$ child process id: 113494
$ child process is killed by signal 2
窗口2：
$ kill -2 113494
```
将if 0 改为if 1 父进程一直不能wait子进程
```cpp
窗口1：
$ ./a.out
$ ppid:130064
$ child process id: 130065

窗口2：
lisongze@svr04:~$ ps -aux | grep a.out
lisongze 130064  100  0.0   4200   788 pts/8    R+   14:08   0:09 ./a.out 父进程
lisongze 130065  0.0  0.0   4200    84 pts/8    S+   14:08   0:00 ./a.out 子进程
lisongze 130240  0.0  0.0  15944  2508 pts/11   S+   14:08   0:00 grep --color=auto a.out
lisongze@svr04:~$ kill -2 130065 	杀掉子进程
lisongze@svr04:~$ ps -aux | grep a.out
lisongze 130064  101  0.0   4200   788 pts/8    R+   14:08   0:22 ./a.out  父进程
lisongze 130065  0.0  0.0      0     0 pts/8    Z+   14:08   0:00 [a.out] <defunct> 子进程变成僵尸态
lisongze 131485  0.0  0.0  15944  2548 pts/11   S+   14:08   0:00 grep --color=auto a.out
lisongze@svr04:~$ kill -9 130065
lisongze@svr04:~$ ps -aux | grep a.out
lisongze 130064  101  0.0   4200   788 pts/8    R+   14:08   0:30 ./a.out
lisongze 130065  0.0  0.0      0     0 pts/8    Z+   14:08   0:00 [a.out] <defunct> 任何信号是干不掉僵尸，但僵尸不会耗费资源
lisongze 132025  0.0  0.0  15944  2488 pts/11   S+   14:09   0:00 grep --color=auto a.out
lisongze@svr04:~$ kill -2 130064  唯一可以清理子进程僵尸方法是杀掉父进程
lisongze@svr04:~$ ps -aux | grep a.out
lisongze 132987  0.0  0.0  15944  2552 pts/11   S+   14:09   0:00 grep --color=auto a.out

```
## 1.4 内存泄漏的真实含义
不是进程死了，内存没释放，而是进程活着，运行越久，耗费内存越多
![](/img/process/mem.jpg)

## 停止状态与作业控制，cpulimit
ctrl + z，fg/bg 例子：
```cpp
int main()
{
	int i = 0;

	while(1) {
		volatile int j,k;
		for(i = 0; i < 1000000; i++);
		printf("hello %d\n",j++);
		printf("world %d\n",j++);
	}
	return 0;
}
```
调试：
通过ctrl+z可让进程暂停，fg可让进程继续执行。

cpulimit 控制一个进程的cpu消耗。
cpulimit -l 20 -p 10111
限制pid为10111程序的cpu使用率不超过20
cpulimit的原理是不断进程在停止和运行态之间来回切换，以此来降低cpu利用率。
![](/img/process/cpulimit.jpg)

## 1.5 fork
打印几个hello？

```cpp
main()
{
	fork();
	printf("hello\n");
	fork();
	printf("hello\n");
	while(1);
}

```
一个fork执行如下图：
![](/img/process/fork.jpg)
上面程序fork之后产生4个进程，打印6个hello

##### 参考学习宋宝华《打通linux脉络：Linux的进程、线程以及调度》后总结。

