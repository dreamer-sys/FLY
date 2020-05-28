Linux IO口的c编程

一.Glibc：是GUN发布的C语言标准库，即C语言运行库
1. 动态库(.so)和静态库(.a)都位于/lib和/usr/lib目录中
libpthread.so   libthread.a      数学计算的c程序:libm.so    多线程的c程序:libpthread.so
2. 函数头文件
/usr/include
3. 函数库说明软件
/usr/man     /usr/share/man
4. ldd命令用于判断一个程序必须使用的动态库
参数说明：
	--version打印ldd的版本号
	-v --verbose打印所有信息，例如包括符号的版本信息
	-d --data-relocs执行符号部署，并报告缺少的目标对象（只对ELF格式适用）
	-r --function-relocs对目标对象和函数执行重新部署，病报告缺少的目标对象和函数（只对ELF格式适用）
5.ldconfig是一个动态链接库管理命令，其目的为了让动态链接库为系统所共享
ldconfig的主要用途: 
	1.默认搜寻/lilb和/usr/lib ,以及配置文件/etc/ld.so.conf内所列的目录下的库文件。
	2.搜索出可共享的动态链接库,库文件的格式为: lib***.so.** ,进而创建出动态装入程序(Id.so)所需的连接和缓存文件。
	3.缓存文件默认为/etc/ld.so.cache ,该文件保存已排好序的动态链接库名字列表。
	4.ldconfig通常在系统启动时运行,而当用户安装了一个新的动态链接库时,就需要手工运行这个命令。
	.
二.GCC编译器动态库的搜索路径搜索的先后顺序
1. 编译目标代码时指定的动态库搜索路径；
2. 环境变量LD_LIBRARY_PATH指定的动态库搜索路径；
3. 配置文件/etc/ld.so.conf中指定的动态库搜索路径；
4. 默认的动态库搜索路径/lib；
5. 默认的动态库搜索路径/usr/lib；
.
三.文件的打开、读写、定位偏移（函数见前一章笔记）
.
四.并行与串行通信（Linux中要访问串口，只要打开相关的文件即可）
1. Linux下串口文件是位于/dev下的：
	COM1串口一位/dev/ttyS0
	COM2串口二位/dev/ttyS1
2. 串口设置（波特率、数据位、校验位、停止位等）
struct termios
{unsigned short c_iflag; /* 输入模式标志*/
unsigned short c_oflag; /* 输出模式标志*/
unsigned short c_cflag; /* 控制模式标志*/
unsigned short c_lflag; /*区域模式标志或本地模式标志或局部模式*/
unsigned char c_line; /*线路规程 */
unsigned char c_cc[NCC]; /* 控制字符特性*/
speed_t c_ispeed;/*输入速度*/
speed_t c_ospeed;/*输出速度*/
};
c_cflag常量名称：
INPCK 奇偶校验使能
IGNPAR忽略奇偶枚验槽误
PARMRK 奇偶校验错误掩码 
ISTRIP裁减掉第8位比特
IXON启动输出软件流控
IXOFF 启动输入软件流控
IXANY输入任意字符可以重新启动输出(默认为输入起始字符才重启输出)
IGNBRK忽略输入终止条件
BRKINT当检测到输入终止条件时发送SIOINT信号。
INLCR将接收到的NL(换行符)转换为CR(回车符)
IGNCR忽略接收到的CR(回车符)
ICRNL将接收到的CR(回车符)转换为NL(换行符)
IUCLC将接收到的大写字符映射为小写字符
IMAXBEL当输入队列满时响铃
1.设置波特率：
	tcgetattr(fd,&options);//设置波特率为19200...
	cfsetispeed(&options,B19200);
	cfsetospeed(&options,B19200);//将本地模式(CLOCAL)和串行数据接收(CREAD)设置为有效...
	options.c_cflag|=(CLOCAL|CREAD); //设置串口...
	tcsetattr(fd,TCSANOW,&options);
2.设置数据位：
	options.c_flag &=~CSIZE; /*用数据位掩码清空数据位设置*/
	options.c_cflag|=CS8;//使用8位数据位
	//options.c_cflag|=CS7; //使用7位 数据位
	//options.c_cflag|=CS6;//使用6位数据位
	//options.c_cflag|=CS5;//使用5位数据位
3.设置奇偶校验位：
	//使能奇校验
	options.c_cflag|=(PARODD|PARENB);
	options.c_iflag|=INPCK;
	//使能偶校验
	options.c_cflag|=PARENB;
	options.c_cflag&=~PARODD;/*清除偶校验标志，则配置为奇校验*
	options.c_iflag|=INPCK;
4.设置停止位：
	options.c_flag&=~CSTOPB;/*将停止位设置为一个比特*/
	options.c_cflag|=CSTOPB;/将停止位设置为两个比特*/
5.激活配置：
	#include <termios.h>
	#include <unistd.h>
	int tcsetattr(int fd,int optional_actions,const struct termios *termios_p);
6 打开串口：
	int fd;//串口的文件描述符
	fd=open("/dev/ttyS0",O_RDWR|O_NOCTTY|O_NDELAY);
	if(fd==-1){
		perror("open_port:Unable to open /dev/ttyS0 -");
	}
7 读写串口：
	//串口发送数据
	char buffer[]="Hello Linux";
	int len;
	int nByte;
	len=strlen(buffer);
	nByte=write(fd,buffer,len);
	if(nByte < len)
	{
		perror("writer fault");
	}
	//申口读取数据
	char buf[1000];
	int nByte;
	int len=900;
	nByte=read(fd,buffer,len);
	if(nByte <0)
	{
		perror("read fault");
	}
