Linux进程

一.shell:Shell是用户与内核之间的接口，shell本身也是一个程序，用户登录后便作为进程运行。
用户登录的shell:
	/etc/passwd
	echo $SHELL

二.进程
1. 进程标识 PID：
每个进程对应一个非负整数, pid_t(有符号整型)
最大的PID为32767，每次分配进程id从小开始分配，依次递增，直到最大值后开始使用空闲的最小pid。
三个特殊的进程：0，1，2
系统第一个进程为init进程，pid为1，以后所有的进程都是通过fork产生的子/孙进程。pstree命令
父进程标识 PPID：
除init进程外，每个进程都有由另一个进程创建出来的，该进程称为被创建进程的父进程，而且每个进程都有唯一的父进程。
2. 获取进程标识
	#include <sys/types.h>
	#include <unistd.h>
	pid_t getpid(void);
获取父进程标识：
	pid_t getppid(void);
3. 用户标识
通常，每个用户有一个唯一的用户ID，创建用户时由系统确定，0为root用户的ID。
/etc/passwd
	帐号:密码:UID:GID:真实姓名:家目录:shell
	root:x:0:0:root:/root:/bin/bash
每个用户还属于用户组，一个用户可以属于多个用户组，一个用户组中可以有多个用户。
/etc/group
	组名:密码:GID:用户列表
	bin:x:1:root,bin,daemon
4. 获取用户标识：
#include   <unistd.h>
#include   <sys/types.h>
获取实际用户ID     uid_t getuid(void );
获取有效用户ID     uid_t geteuid(void );
获取实际组ID        gid_t getgid(void );
获取有效组ID        gid_t getegid(void );
实际上，uid_t和gid_t都是unsigned short类型
5. 切换用户身份使用“su  –  <用户名>”命令。使用exit退出当前切换。
root调用”chmod 4755 getids”
	查看权限，root与u1分别运行getids比较结果，4代表root模式下可读
表示附加权限的把八进制数字，abc与之前一致，分别是对应User、Group、及Other（拥有者、群组、其他组）的权限。因为SUID对应八进制数字是4，SGID对于八进制数字是2，则“4755”表示设置SUID权限，“6755”表示同时设置SUID、SGID权限。

用户标识小结：
	实际ID：创建进程的用户ID
	有效ID：进程访问系统资源的身份ID
	一般情况，实际用户ID与有效用户ID相同，当执行程序具有setuid权限时，进程的有效用户ID变为程序的拥有者

三.创建进程（fork）
1. #include <unistd.h>
	pid_t  fork( void );
fork被调用一次，产生一个新的子进程，即存在了2个进程（一次调用，两次返回）
返回值：父进程中返回子进程id，子进程中返回0，错误返回-1。
子进程id，执行的进程的程序getpid ()获取的就是父进程的ID, id就是自己成的ID；
id=0执行的是子进程程序；id<0; 创建失败，退出

getpid()!=id

2. 关于子进程对父进程资源的处理：
进程映像：PCB、程序段、数据段、堆、栈
PCB：子进程获得父进程的副本，并修改PCB中部分属性。
程序段：父子进程共享，因为程序段只读属性。
数据段、堆、栈：写时复制技术，一般进程大部分资源在虚拟内存中，为节省全部复制浪费的时间空间，创建子进程时这些区域不复制，只有在对这些区域进行修改时才复制部分内存页。
3. fork举例：			|左边例子中，执行次序：
pid_t  id;				|父进程：1赋值id   			2对if-else-else语句选择执行  3其他代码
id=fork();             |子进程：1fork成功后，赋值id  2对if-else-else语句选择执行  3其他代码
if(id<0) 			   |期间遇到exit或main函数return都会各自退出进程
{    perror(“fork”);    |
}   				   |
else if(id==0)		    |
{   //子进程代码   }		|
else					|
{    //父进程代码   }	|
其他代码;

说明：fork之前都是父进程代码，调用fork成功后，父子进程都将继续执行调用fork以后的所有代码，直到exit或main函数return。

	forktest.c
假设父进程id为1273，子进程id为1274。
	pid_t id;
	id=fork();
	if(id<0)
	{   perror("fork");    exit(1);}
	else if(id==0)
	 {    printf(“child pid=  %d\n",getpid());  }
	      else
	  {    printf(“parent pid=  %d\n",getpid());  }
	printf("%d print this sentence\n",getpid());	
注：父进程先打开文件，向文件写内容hello，创建子进程子进程写内容world，父进程再写welcome。结果文件内容是helloworldwelcome。父子进程共享打开文件表项。但是子进程操作不会影响父进程。

四.vfork
1. 定义：vfork与fork的区别在于vfork是借用而不复制父进程空间，之所以提出vfork就是由于fork写时复制的策略固然能减少全部拷贝浪费的时间空间
所以如果创建子进程目的是要执行其他程序，那么这部分空间也不需要拷贝。因此使用vfork创建子进程，目的是子进程直接调用exec族函数去执行其他程序。
2. fork与vfork的区别：
#include <unistd.h>
pid_t  vfork( void );
vfork:
	子进程与父进程共享数据段和堆栈
	父进程被挂起，子进程运行结束或调用exec族、exit时父进程才能继续执行。

五.exec族函数----执行一个新程序
1. 一般父进程创建子进程后,子进程要执行其他的程序，这也是Linux内核执行程序的方法。
一个进程如何来启动另一个程序的执行？ －exec族函数
2. exec族函数：以参数所指程序替换正在执行的程序
execl、execlp、execle
execv、execvp、execve
形式：execAB（A：l、v；B：空、p、e）
3. exec族函数原型
exec*
int execl (const char *path, const char *arg, ...);
	int execlp(const char *file,  const char *arg, ...);
	int execle(const char *path, const char *arg,… , const char *envp[ ]);
	int execv (const char *path, const char *argv[ ]);
	int execvp(const char *file,  const char *argv[ ]);
	int execve(const char *path, const char *argv[ ], const char *envp[ ];
.
path/file：可执行文件名
arg/argv：传给path/file可执行文件的命令行参数
envp：环境变量
exec族函数的返回值：
	成功则没有返回值，失败返回-1，最常见的错误就是参数所指定的可执行文件不存在。
.
1.l、v的区别：命令行参数形式不同
2.p和非p的区别：可执行文件名是否需要指定路径。
3.e和非e的区别：是否需要指定环境变量列表
4. 定义函数
    #include<unistd.h>
    int execl(const char * path,const char * arg,...., NULL);
函数说明:
   execl()用来执行参数path字符串所代表的文件路径，接下来的参数代表执行该文件时传递过去的argv[0]、argv[1]……，最后一个参数必须用空指针(NULL)作结束。
返回值: 
   执行成功，则函数不会返回,执行失败，则直接返回-1，失败原因存于errno
参数说明：
	path是一个绝对路径,argv[?]是传给新进程的参数,很有意思的是argv[0]不必是正确的文件名, 实际上可以是任何字符串。就是为了伪装argv[0],让它不能反映程序的真是位置而已.
5. 例：
#include <unistd.h>
main( )
{
    execl(“/bin/ls”, “ls”, “-al”, “/etc/passwd”, NULL);
}
执行结果:
-rw-r--r-- 1  root  root  705  Sep 3 13 :52 /etc/passwd
6. 应用举例
#include <unistd.h>
#include <stdio.h>
int main( )
{
    printf(“Running  ps  with  execlp\n”);
    execlp(“ps”,  “ps”,  “-af”,  0);
    printf(“Done!\n”);//此时done不会输出，因为上条指令去执行了其他程序
    exit(0);
}
exec族函数说明：
	1.使用参数指定程序的程序段、用户数据段、堆栈替换原进程的程序段、用户数据段、堆栈。
	2.原进程的系统数据段（包括PCB）几乎没有被破坏，所以进程属性基本上被保留了，比如PID。
	3.也就是说，对系统而言，还是那个进程，不过已经开始执行另一个程序了（旧瓶装新酒）。 
六.wait和waitpid：父进程等待子进程（创建子进程、运行一个程序、等待子进程结束）
1. 作用：
进程之间的继承关系可使用pstree查看，除init外所有进程都有父进程。父进程负责处理僵死的子进程（通过wait系列函数）。每个结束的子进程都有结束状态，可由父进程通过wait系列函数获得。
这里的“等待”包括几个过程：
(1)父进程挂起等待子进程结束
(2)获得子进程结束状态
(3)处理僵死子进程。
2. wait和waitpid
wait：
	等待任意一个子进程
	属于waitpid的特例
waitpid：
	等待指定的子进程，如指定某个子进程，指定属于某个组的任意子进程。
3. wait
#include <sys/types.h>
#include <sys/wait.h>
 pid_t wait(int *status);
参数：
	status：子进程的返回状态指针，可置NULL表明父进程不获取子进程返回状态。
返回值：
	成功返回终止子进程id，返回0表示没有子进程，错误返回-1置errno。
说明：
	当调用wait时，父进程被挂起直至该进程的任一个子进程结束wait调用才返回。
4. waitpid
pid_t waitpid(pid_t pid, int *status,int options);
参数:
	pid:
		pid  >  0 : 等待进程id为pid的子进程
		pid == 0 : 等待与自己同组的任意子进程
		pid == -1: 等待任意一个子进程 wait()
		pid  <  -1: 等待进程组号为|pid|的子进程
	status：子进程退出状态，不需要可置为NULL
	options：可为0，或者WNOHANG：若指定子进程未结束立即返回0
返回值：
	成功返回终止子进程id，返回0表示没有子进程，错误返回-1置errno。
wait(&stat) 等价于 waitpid(-1, &stat, 0)
.
.
多进程程序的一般结构:
      if ((pid = fork())> 0)
      {
           parent’s code;
           wait( );    
       }
       else if (pid == 0)
            child’s code;
       else
            error handling;
       return xxx;
.
七.退出进程
1. 进程的终止：
正常：
	main函数结束
	main函数中调用return
	exit
	_exit
异常：
	接收到某种信号
	调用abort：产生SIGABRT信号，类似接收信号
2. 进程的终止状态：
整型，由父进程接收。
可以通过exit、_exit、main中的return返回进程终止状态。
若main的返回类型是整型，main函数执行到最后一条语句时（隐式返回），则进程终止状态是0。
3. exit
#include <stdlib.h> 
 void exit( int status);
参数：status代表进程退出状态，可不加。
说明:
	1.正常结束当前进程
	2.无返回值，传递SIGCHLD信号给父进程，父进程可以由wait函数取得子进程结束状态。 
	3.刷新进程缓冲区，关闭未关闭的文件。
4. _exit
#include <unistd.h> 
 void _exit( int status);
参数：status代表进程退出状态。
说明 :
	1.立即结束当前进程
	2.无返回值，传递SIGCHLD信号给父进程，父进程可以由wait函数取得子进程结束状态。 
	3.不刷新进程缓冲区，关闭未关闭的文件。
	4.进程结束是通过显式或隐式(通过调用exit)调用_exit结束
5. exit和_exit区别
1.exit是标准C库函数，_exit是unix库函数
2.exit最终会调用_exit实现
3.都没有返回值，都会向父进程传递SIGCHLD信号。
4.exit可以不带终止状态，_exit必须带
5.exit正常结束进程，在进程终止之前要刷新缓冲区
6._exit立即结束进程，不刷新缓冲区。
.
6. 进程的一生(从进程本身角度观察)
1.随着一句fork，一个新进程呱呱落地，但这时它只是老进程的一个克隆。
2.然后，随着exec，新进程脱胎换骨，离家独立，开始了独立工作的职业生涯。
3.人有生老病死，进程也一样
自然死亡：运行到main最后一个“}”，从容离去；
中途退场：2种方式，一种是调用exit函数，一种是在main函数内使用return。无论哪一种方式，它都可以留下留言，放在返回值里保留下来；
强制结束：被其它进程结束它的生命。
4.进程死掉以后，会留下一个空壳，父进程调用wait打扫战场，使其最终归于无形。

八.进程控制相关函数（在一个程序内通过system函数来运行另一个程序并由此创建一个新进程，这是在一个程序内执行另一个程序的最简单的方法。）
1. system
#include <stdlib.h>
int system(const char *string);
	string：要加载执行的命令
命令执行失败时返回-1
参数string为要执行的命令字符串，它将被送给Unix的命令解释程序shell，由shell来执行该命令。
实际上system是通过fork，exec，waitpid来实现的。
2. Linux特殊进程
1.因父亲进程先退出而导致一个子进程被init进程收养的进程为孤儿进程。
例：
int main()
{
	pid_t pid;
	if((pid=fork())==-1)
		perror("fork");
	else if(pid==0)
	{
		printf("pid=%d,ppid=%d\n",getpid(),getppid());
		sleep(2);
		printf("pid=%d,ppid=%d\n",getpid(),getppid());
	}
	else
	exit(0);
}

2.而已经退出但还没有回收资源的进程为僵死进程。 
例：
int main()
{
	pid_t pid;
	if((pid=fork())==-1)
		perror("fork");
	else if(pid==0)
	{
		printf("child_pid pid=%d\n",getpid());
		exit(0);
	}
	sleep(3);
	system("ps");
	exit(0);
}
