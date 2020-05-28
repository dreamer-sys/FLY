Linux进程间通信

一.信号
1. 信号：软中断，被发送给一个正在执行的进程，通知该进程有某一特定事件发生。
通常用于异步事件（如：用户击键这样进程外的事件）处理。
每个信号都有一个名字，以SIG开头
2. kill –l查看信号值
man 7 signal查看信号详细信息
3. 常见信号：在/usr/include/asm/signum.h中每个信号都被定义为一个正整数。

	0信号被认为是空信号，kill可以向进程发送0信号，以此判断进程是否存在。
	进程收到信号后，大部分情况都是要终止
	信号名       值         说明                        缺省动作
	SIGINT       2      中断Ctrl+C                      终止
	SIGKILL      9      终止进程，不可阻塞、捕捉、忽略     	终止
	SIGTERM      15     终止进程，可阻塞、捕捉、忽略        终止
	SIGALRM      14     超时                          	终止     SIGALRM: 定时器到期，可用alarm函数来设置定时器。默认动作终止进程。
	SIGUSR2      12     用户自定义信号                   终止
4. 信号来自内核，但生成信号的请求来自3处：
	1、用户：用户通过Ctrl－C或Ctrl－\等按键来请求内核产生信号。
	2、内核：当进程执行出错时，内核给进程发送一个信号，例如：非法段存取，浮点数溢出，或非法的机器指令。
	3、进程：一个进程可以通过系统调用kill给另一个进程发送信号，这也是进程间通信的一种方式。 
只有具有root权限的进程才能向其他任一进程发送信号，非root权限的进程只能向属于同一个组或同一个用户的进程发送信号。 
5. 信号的传递过程（内核产生信号后要把它传递给某个或某些进程）
（1）生成（generating）。内核要更新目标进程的数据结构，表示一个新的信号已经被发送给此进程。此时，进程并没有对信号做出任何响应。
（2）传递（delivery）。强迫目标进程对信号做出响应。
信号生成后到传递前的那段时间，信号处于未决状态。
6. 信号做什么？
视情况而定，很多信号都会杀死进程。某时刻进程还在运行，下一秒就消亡了，从内存中被删除，相应的所有文件描述符被关闭，并从进程表中被删除。但是进程也有办法保护自己不被杀死。
7. 当进程接收到SIGINT时，并不一定要消亡，进程能够通过系统调用signal告诉内核，它要如何处理信号。进程有3种选择：
	1、接收默认处理（通常终止进程）
	     Linux对每个信号都有默认处理。此时进程不需要显式的调用signal。但进程可以通过以下调用来恢复默认处理：signal（SIGINT，SIG_DFL）;
	2、忽略信号
	     signal（SIGINT，SIG_IGN）；
	3、调用一个函数
	     程序告诉内核当信号到来时应该调用哪个函数。信号到来时被调用的函数称为信号处理函数。程序调用：
	     signal（signum，functionname）；
SIGKILL和SIGSTOP这两种信号不能被忽略的原因是：它们向超级用户提供一种使进程终止或停止的可靠方法
8. Signal --安装信号，设置信号处理办法
	#include <signal.h>
	void (*signal(int signum, void ((*handler)(int))))(int);
功能：指定signum信号的处理函数为handler。
参数：
	signum：信号值
	handler：
	执行系统默认动作：SIG_DFL
	忽略：SIG_IGN 执行忽略
	捕捉：参数为int返回值为void的信号处理函数。
	返回值：
		成功上次对该信号的处理函数地址，失败返回SIG_ERR
	注意：SIGKILL与SIGSTOP不可忽略与捕捉
9. 生成信号的两个函数：kill和raise
	int kill(pid_t pid, int signum);
功能：向pid进程发送信号signum
参数：
	pid>0：发给pid进程   (掌握)
	pid==0：发给与发送进程同一进程组的所有进程。
	pid<0：发给进程组号为|pid|的所有进程，且发送进程对该进程具有发送信号权限。
	pid==-1：发给所有进程，且发送进程对该进程具有发送信号权限。（各种内核版本说法不一）
	返回值：成功返回0，错误返回-1。

生成信号的两个函数：kill和raise
#include <signal.h>
int raise(int sig);
功能：给当前进程发送信号
参数：sig：要发送的信号
返回值：
	成功是返回0；失败时返回非0值
raise(sig);等价于kill(getpid(), sig);

10. 设置定时器：alarm
	#include <unistd.h>
	unsigned int alarm(unsigned int seconds);
功能：设置一个定时器
参数：seconds：定时器的时间
返回值：
	返回上一个定时器剩余时间；如果没有其他定时器则返回0
定时器的作用：
     1、在执行流中加入时延。
     2、调度一个将来要做的任务。
11. 挂起进程：pause
	#include <unistd.h>
	int pause(void);
功能：暂停进程直到信号到达。任何信号都可以唤醒进程，而非仅仅等待SIGALRM。当捕获到信号时返回 -1
	Alarm举例：
	#include <stdio.h>
	#include <signal.h>
	main()
	{
	 void wakeup(int);
	 printf("about to sleep for 4 seconds\n");
	 signal(SIGALRM,wakeup);
	 alarm(4);
	 pause();
	 printf("morning so soon?\n");
	}
	void wakeup(int signum)
	{
	  printf("Alarm received from kernel\n");
	}
注：系统中每个进程都有一个私有的闹钟，就像一个定时器，可设置在一定秒数后闹铃。时间一到，时钟就发送一个信号SIGALARM到进程。除非进程为SIGALARM设置了处理函数（handler），否则信号到达将杀死这个进程。
新闹钟将覆盖上一个闹钟。

12. 添加延时：sleep
	#include <unistd.h>
	unsigned int sleep(unsigned int seconds);
功能：当前进程睡眠指定的秒数
参数：seconds：要睡眠的秒数
返回值：
	睡眠结束时返回0；否则返回睡眠剩余时间。
Sleep函数是如何实现？要做3件事情：
	为SIGALRM设置处理函数；
	调用alarm（）；
	调用pause（）；
注意sleep与alarm比较：
	sleep阻塞等待，其后语句暂不执行
	alarm非阻塞等待，其后语句继续执行
13. 信号集与屏蔽信号
有时既不希望进程接到信号立即中断当前执行，也不希望信号被忽略掉，而是延迟一段时间去调用信号处理函数。这就可以通过阻塞信号的方法实现。
每个进程都有一个信号掩码（signal umask），用来保存即将被阻塞的信号。
信号忽略：系统仍然传递该信号，只是相应进程对该信号不作任何处理而已。
信号阻塞：系统不传递该信号，显示该进程无法接收到该信号直到进程的信号集发生改变。
信号集通常用于进程对信号的阻塞操作。
使用方法：准备好进程的信号集，调用sigprocmask设置进程的信号掩码。

14. 信号集：
代表由多个信号所组成集合的一种数据类型
sigset_t类型，按位代表最多1024个信号
	<signal.h>
	typedef __sigset_t sigset_t;
	<bits/sigset.h>
	# define _SIGSET_NWORDS (1024 / (8 * sizeof (unsigned long int)))
	typedef struct
	 {
	unsigned long int __val[_SIGSET_NWORDS];
	} __sigset_t;
	.
信号集是按位进行设置相应信号的，不方便，所以提供一套信号集操作函数设置信号集的值：
	清空信号集，不包含任何信号 int sigemptyset(sigset_t *set)；
	设满信号集，包含所有信号 int sigfillset(sigset_t *set)； 
	将signum信号加入信号集 int sigaddset(sigset_t *set, int signum) ；
	将signum信号从信号集中删除 int sigdelset(sigset_t *set, int signum)；
	判断signum是否属于set信号集 int sigismember(const sigset_t *set, int signum)；
15. 设置信号掩码来完成信号的阻塞（了解）
	int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);  
参数
	How：做什么操作
	SIG_BLOCK //添加信号掩码，set与oldset逻辑或SIG_UNBLOCK //去掉部分信号，set与oldset逻辑减
	SIG_SETMASK //替换信号掩码，set对oldset进行赋值
	set：新信号集的指针，为NULL表明仅想读取当前信号掩码
	oldset：存放原来信号集的指针
返回值：成功返回0。失败返回-1（置errno）
16. sigaction函数
在POSIX中用sigaction函数代替signal.参数非常类似。指定什么信号将被如何处理。如果愿意还能够得到这个信号上一次被处理的设置。
函数原型
    #include <signal.h>
    int sigaction(int signum,const struct sigaction *act,struct sigaction *oldact)); 
     第一个参数：要处理的消息。
     第二个参数：描述如何响应信号的结构体。
     第三个参数：如果不是NULL，就指向描述被替换的处理结构。
sigaction与signal的比较：
	在sigaction 中的struct sigaction结构：在老式的信号处理方式基础上增加了新的处理方式。
	sigaction较signal更可靠。
.      
二.管道(如果用管道，那么管道要创建于子进程之前)
1. 单向传输：管道是最早期的IPC方式，进程间可通过管道传递数据。发送数据的进程通过管道的写端写入数据，接收进城通过管道的读端读取数据。
管道分类：
	无名管道pipe：进程本身或父子进程间使用
	有名管道fifo：无亲缘关系进程间也可使用
单独构成一种独立的文件系统
管道对于管道两端的进程而言，就是一个文件，但它不是普通的文件，它不属于某种文件系统，而是自立门户，单独构成一种文件系统，并且只存在于内存中 
----------->===========--------->
发送进程    字符先进先出  接收进程
2. 无名管道
	#include <unistd.h>
 	int pipe(int pipedes[2]);
参数：pipedes：文件描述符数组，0读1写(一般先写再读)
返回值：成功返回0，错误返回－1（置errno）
说明：
	创建一个管道，这个管道是由pipedes [0]，pipedes [1]两个文件描述符表示的一个通信信道，大小4096。
	pipedes[1]为写文件描述符，pipedes [0]为读文件描述符。
	对管道读写用read，write
3. 命名管道
	#include <sys/stat.h>
	int mkfifo(const char *path, mode_t perms);
参数：
	path：有名管道的路径名
	perms：管道的访问权限，同open参数。
返回值：成功返回0，错误返回-1（置errno）
说明：
	对有名管道的访问：open、read、write、close
	注意：如果某进程以只读打开管道后，将会阻塞直到另一个进程以只写方式打开管道后才能继续进行，反之亦然。
区别：
无名管道只有具有亲缘关系的进程可以使用，如父子进程通信。
有名管道有名字，能够获得其名字的进程都可以使用。
有名管道的名字可以在目录树中找到，但占用的是内存空间，不拥有磁盘块。
1有名管道先进先出，占用内存，特殊文件
2单向，双向通信要两个管道
	无名管道：亲缘进程使用，如父子进程
	有名管道：任意进程都可使用
3无名管道创建同时就已打开，满写或空读都将阻塞。
4命名管道需要显式调用open打开，在目录树中可看到名字。
	读/写方式打开管道将会阻塞到写/读方式打开。
	以读写方式打开将不会阻塞。
	满写空读也会阻塞。
5读关闭，写错误；写关闭，读返回0
.
三.进程间通信（IPC）
1. SystemV IPC
1）在本机中通信，不能跨网络；
2)生存期与内核相同，系统重启则消失；
3)生存期内，任何进程可使用整型标识符访问。
4)没有i节点和路径名，所以不能像文件/目录一样访问
5)可使用ipcs显示状态、ipcrm删除对象。
6)访问权限：读、写，没有执行
7)系统调用：
	Xget、Xctl
		（X：sem-信号量，shm-共享内存，msg-消息队列）
	semop，shmat、shmdt，msgsnd、msgrcv
8)SystemV可将路径名使用ftok函数生成关键字，Xget的参数为关键方式来打开对象返回整型标识符。
2. 信号量（semaphore）
1）一种比较特殊的IPC，它只是一个计数器（counter）。一般情况，多个进程在访问共享对象时使用信号量实现同步操作，如经典的生产者/消费者问题。
实质是一个整数计数器，记录可同时访问共享资源的单元个数。
当进程要求使用某资源，先对信号量减1
	>=0：进程可以用该资源
	<0：进程休眠，直至信号量值大于或等于0时被唤醒。
进程对资源访问结束时，信号量值加1
	<=0：证明有其他进程等待，唤醒队列第一个进程。

2）信号量操作基本过程
POSIX信号量只针对一个信号量操作，SystemV信号量可以对一个信号量集合操作。
基本步骤：
	通过路径名设置信号量集合的关键字：ftok
	建立信号量集合并得到标识符：semget
	设置信号量集合：semctl
	对信号量集合中任意个信号量操作（加减1）：semop
	删除信号量集合操作使用semctl完成。
3）semget
	#include <sys/types.h>
	#include <sys/ipc.h>
	int semget(key_t key, int nsems, int semflg);
功能：创建信号量
参数：
	key：信号量的键值
	nsems：信号量的个数
	semflg：创建标志，可以是IPC_CREAT或IPC_CREAT | IPC_EXCL
返回值：成功返回共享内存ID；失败返回-1，并设置errno
4）semctl
	int semctl(int semid, int semnum, int cmd, ...);
功能：信号量控制函数
参数：
	semid：信号量ID
	cmd：控制命令
	semnum：信号量集中信号量的序号
返回值：成功返回0；失败返回-1

3. System V共享内存（share memory）
1）概括：通过两个或多个进程共享同一块内存区域实现进程间通信。通常是一个进程创建共享内存区域，然后多个进程可以对其进行访问。
由于数据是由内存直接映射到进程空间，所以速度快。但涉及同步问题，可用信号量来实现同步。
2）共享内存基本操作过程：
	创建将被共享的内存空间（确定大小）；
	将该空间映射到本进程中；
	进行正常的读写操作；
	解除映射；
	如果需要，删除被共享的内存空间。
3）#include <sys/shm.h>
创建并得到共享内存段：int shmget(key_t key, size_t size, int flags);
返回值：成功返回共享内存标识符，错误返回-1(置errno)
.
控制共享内存段(包含删除)：int shmctl(int shmid, int cmd, struct shmid_ds *data);
返回值：成功返回0，错误返回-1（置errno）
.
连接共享内存段：void *shmat(int shmid, const void *shmaddr, int flags);
返回值：成功返回进程中该内存段的地址，错误返回-1(置errno)
.
断开共享内存段：int shmdt(const void *shmaddr);
返回值：成功返回0，错误返回-1（置errno）
.
4）shmget
	#include <sys/ipc.h>
	#include <sys/shm.h>
	int shmget (key_t key, size_t size, int shmflg);
功能：创建共享内存
参数：
	key：共享内存的键值
	size：共享内存大小
	shmflg：创建标志
返回值：成功返回共享内存ID；失败返回-1
5）shmat说明
	void *shmat(int shmid, const void *shmaddr, int flags);
参数：
	shmid：共享内存标识符。
	shmaddr：进程映射内存段的地址，可以指定，但一般设置为NULL，表示由系统安排，安排好的地址由返回值返回。
	flags：对该内存段是否设置只读，默认设置读写。
返回值：成功返回进程中该内存段的地址，错误返回-1(置errno)
6）共享内存与管道的区别：
共享内存与管道都是利用内存空间进行进程间通信。
进程server从一个文件传输到进程client的过程看两者的区别，管道需要4次复制，共享内存需要2次复制，速度快。
4. System V消息队列
1）概念：也叫报文队列，克服信号最多在接收信号进程的生命周期中有效（随进程持续）的缺点。消息队列在系统重启前一直有效（随内核持续）
2）消息队列，即消息的一个链表，一系列连续排列的消息，保存在内核中，通过消息队列的引用标识符访问。
每个消息都包括两部分：类型和正文。
任何进程都可以向消息队列中发送指定类型消息，其他进程都可以从消息队列中根据类型获取相应的消息。
3）消息队列操作基本过程：
	创建消息队列；
	根据收/发消息类型进行收/发消息；
	如有必要，则删除消息队列；
关于消息队列说明：
	队列，先进先出
	如果类型相同，先发送的消息将先被读出
	消息被读出则消息队列就减少一个结点。
	当消息队列满时的写操作或消息队列空时的读操作都会阻塞。
4）msgget
	#include <sys/types.h>
	#include <sys/ipc.h>
	#include <sys/msg.h>
	int msgget(key_t key, int msgflg);
功能：创建、打开消息队列
参数：
	key：消息队列键值
	msgflg：创建方式
返回值：成功返回消息队列ID；失败返回-1
5）msgsnd
	int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg)
功能：发送消息
参数：
	msgid：消息队列ID
	msgp：指向消息指针
	msgsz：消息长度
	msgflg：发送方式
返回值：成功返回消息队列ID；失败返回-1
6）msgrcv
	ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
功能：接收消息
参数：
	msgid：消息队列ID
	msgp：指向消息的指针
	msgsz：消息长度
	msgflg：接收方式
返回值：成功返回消息正文字节数；失败返回-1
7）msgctl
	int msgctl(int msqid, int cmd, struct msqid_ds *buf);
功能：消息队列控制函数
参数：
	msgid：消息队列ID
	cmd：控制命令
返回值：成功返回0；失败返回-1
四.总结
1）管道：
连接一个进程的输出至另一个进程的输入的一种方法。
单向，无名( 进程本身或亲缘进程)、有名(FIFO无关进程)
掌握无名管道父子进程操作
2）信号：
用于通知接收进程有某种事件发生
signal,kill
3）信号量：
一个计数器，用来记录对某个资源的使用情况
4）共享内存:
允许多个进程访问同一块内存空间
5）消息队列：
消息的链表，链表中每个节点包括消息类型和正文两部分。
