Linux线程

一.概念
1. 一个程序如何才能同时（宏观）完成多个任务？
	使用fork和exec同时运行多个程序。
	使用线程在同一个程序中同时运行多个程序。
2. 一个进程至少包括一个线程，还可以创建其他线程，各个线程之间平等，通常称原来的线程为主线程。
3. 进程是资源分配的最小单位，而线程是计算机中独立运行、CPU调度的最小单元同一进程中的线程共享整个进程空间
4. 进程与线程比较：
	创建进程比创建线程时空开销大；
	多进程间通信开销大；
	多线程间通信开销小，但是线程太多也容易导致其他同步的问题；
	进程稳定性比线程好；
	一个进程出问题，其它进程不受影响；
	一个线程出问题，其他线程容易受影响

	简单总结：
	节约资源，速度快，选择多线程
	稳定性好，选择多进程
5. 线程共享资源:
	文件描述符表
	每种信号的处理方式
	当前工作目录
	用户ID和组ID
	内存地址空间 (.text/.data/.bss/heap/共享库)

6. 线程非共享资源
	线程id
	处理器现场和栈指针(内核栈)
	独立的栈空间(用户空间栈)
	errno变量
	信号屏蔽字
	调度优先级
7. 线程与共享  线程间共享全局变量！
注：线程默认共享数据段、代码段等地址空间，常用的是全局变量。而进程不共享全局变量，只能借助mmap

二.创建线程
1. pthread函数(其作用，对应进程中fork() 函数)
 
	int pthread_create (pthread_t *thread, const pthread_attr_t *attr,void *(*start_routine) (void *), void *arg); 
	参数：
		thread：指向线程标识符的指针
		attr：设置线程属性，可设为NULL表示默认属性
		start_routine：线程运行函数的起始地址，注意原型
		arg：传给运行函数的参数，不传则设为NULL，若传多个用结构体“打包”。 
		返回值：成功返回0，失败返回errno，不置errno。
	例：void *function(void *arg)
	{	//线程运行函数操作 }
	main()
	{
	     pthread_t  tid;           
	     int ret;
	     ret = pthread_create ( &tid , NULL, function, NULL);//或者&function
	     if(ret!=0) 
	     {   printf ("Create pthread error!\n");
		    return  1;
		}
	   pthread_join(tid,NULL);//等待线程tid结束
	}
2. 获取当前线程id的函数
#include <pthread.h>
pthread_t pthread_self(void);
.
注意：编译链接时，一定要加上-lpthread或者-pthread选项
.
三.进程等待
1. 线程等待的原因 
有时一个线程为了等待某个线程执行结束，需要使用pthread_join挂起当前线程等待另一个线程结束。
默认线程（可联合）退出后资源不主动释放，需要调用pthread_join等待并释放资源。
2. pthread_join函数，阻塞等待线程退出，获取线程退出状态（其作用，对应进程中waitpid()函数）
int pthread_join (pthread_t th, void **thread_return);
参数：
	th：被等待的线程标识符，是线程ID 【注意】：不是指针
	thread_return：自定义指针，存储被等待线程的返回值。通常不使用，直接置为NULL即可，thread_return：存储线程结束状态。
	返回值：成功返回0，错误返回-1（置errno）
例：
void *function(void *arg)
{   
     //线程运行函数操作  
 }
main
{
     pthread_t  tid           
     int ret;
     ret=pthread_create(&tid, NULL, function, NULL);
	 //其他操作
     pthread_join(tid,NULL);//等待线程tid结束
     //继续其他操作
}
.
四.线程结束方式
1. 	正常结束
	中途退出：pthread_exit
	被其他线程强制退出：pthread_cancel
2. 线程可隐式退出（执行函数结束），也可显式调用pthread_exit()
void pthread_exit(void *value_ptr);
	参数：指向返回状态的指针，可置为NULL。
线程内调用pthread_exit与exit的区别：
	pthread_exit结束当前线程
	exit结束当前进程，当前进程内的其他线程也被结束
4. 多线程环境中，应尽量少用，或者不使用exit函数，取而代之使用pthread_exit函数，将单个线程退出。任何线程里exit导致进程退出，其他线程未工作结束，主控线程退出时不能return或exit。
另注意，pthread_exit或者return返回的指针所指向的内存单元必须是全局的或者是用malloc分配的，不能在线程函数的栈上分配，因为当其它线程得到这个返回指针时线程函数已经退出了。

五.线程的取消(一个线程可以自己把自己杀死，也可以被别人杀死)
1. 线程的终止可能由于被其他线程调用pthread_cancel取消，线程有是否可被取消的属性，默认是可被取消的
一个线程可以使用pthread_cancel取消另一个线程：
      int pthread_cancel (pthread_t thread_id);
	参数：被取消线程的id
	返回值：成功返回0，否则返回errno。

若要取消一个线程首先必须知道该线程id，可将要被取消的线程id设为全局变量，如何获取线程的id：
	在pthread_create时返回给调用者
	在线程中调用pthread_self系统调用
	pthread_t pthread_self(void);
例：pthread_cancel(pthread_self());//杀死自己

六.互斥锁
1. 初始化互斥量有两种方法：
	方法1，静态分配:
		pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER

	方法2，动态分配:
		int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread mutexattr _t *restrict attr);
	参数：mutex：要初始化的互斥量
	attr：NULL
2. 销毁互斥量（销毁互斥量需要注意）：
	使用PTHREAD_MUTEX_INITIALIZER初始化的互斥量不需要销毁
	不要销毁一个已经加锁的互斥量 已经销毁的互斥量，要确保后面不会有线程再尝试加锁
	int pthread_mutex_destroy(pthread_mutex_t *mutex)；
3. 互斥量加锁和解锁
int pthread_mutex_lock(pthread mutex t *mutex); int pthread_mutex_unlock(pthread mutex t *mutex);
返回值:成功返回0,失败返回错误号
4. 进程有优先级，线程也有优先级。当某个线程优先级很高，只要这个优先级很高的线程一到，其他线程都必须排到后面挂起等待
5. 例：
/*线程互斥锁*/
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
pthread_mutex_t mutex=PTHREAD_MUTEX_INITIALIZER;
int count=0;

void *thread_fun();
int main (int argc,char **argv[])
{
    int rtn1,rtn2;
    pthread_t thread_id1;
    pthread_t thread_id2;
    char *message1="new_thread1";
    char *message2="new_thread1";
    rtn1=pthread_create(&thread_id1,NULL,&thread_fun,(void*)message1);
    rtn2=pthread_create(&thread_id2,NULL,&thread_fun,(void*)message2);
    pthread_join(thread_id1,NULL);
    pthread_join(thread_id2,NULL);
    pthread_exit(0);
}
void *thread_fun()
{
    pthread_mutex_lock(&mutex);
    count++;
    sleep(1);
    printf("count=%d\n",count);
    pthread_mutex_unlock(&mutex);
    printf("----end----\n");
}
.
七.分离线程
1. 为什么要分离线程？
如果新线程创建后，不用pthread_join()等待回收新线程，那么就会造成内存泄漏，但是当等待新线程时，主线程就会一直阻塞，影响主线程处理其他链接要求，这时候就需要一种办法让新线程退出后，自己释放所有资源，因此产生了线程分离
2. 线程自己退出后释放自己资源int pthread_detach(pthread_self())

	线程组内其他线程对目标线程进行分离：int pthread_detach(pthread_t thread)
	返回值：成功返回0，失败返回错误码
如果一个线程进行了分离，那么就不需要pthread_join操作了

八.两个方向的参数传递:
1. 主线程向子线程传递参数:
通过函数 int pthread_create(pthread_t *thread, const pthread_attr_t *attr,void *(*start_routine) (void *), void *arg);
在创建线程时，利用参数arg传递参数给子线程.
2. 子线程向主线程传递参数:
通过函数 int pthread_join(pthread_t thread, void **retval);
主线程等待子线程结束，从参数retval读取子线程的返回值.
在需要传递多个简单结构参数的时候，通常将线程间传递的参数定义为一个结构体。