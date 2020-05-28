# Linux下的函数调用

#### 一. open函数头文件（man 2 open查看）对文件的操作前提都是先打开文件

#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

1. int open(const char*pathname,int flags,mode_t perms);//已存在文件则可以将mode省略
                      路径    O_RDONLY: 只读打开|
							  O_WRONLY: 只写打开|这三个常量，必须制定一个且只能指定一个，
							  O_RDWR: 读，写打开|
.							  
							  O_CREAT: 若文件不存在，则创建它，需要使用mode选项。来指明新文件的访问权限
							  O_APPEND: 追加写，如果文件已经有内容，这次打开文件所写的数据附加到文件的末尾而不覆盖原来的内容
							  O_TRUNC:如果已经存在文件，并且打开的方式是可写的(如O_RDWR或O_WRONLY)，就将文件的长度截为0，也就是将内容清除
.							  
返回值：成功打开返回打开文件的文件描述符，是int类型，失败返回-1.
.
2. int open(const char*pathname,int flags,mode_t mode);//创建并打开一个不存在的文件，三个参数都要写
 第三个参数是设定该文件的权限，具体参数如下（权限：用户所有者  组   其他人）
S_IRUSR : 文件所有者有读（r）权限
S_IWUSR : 文件所有者有写（w）权限
S_IRGRP : 文件所属组有读（r）权限
S_IWGRP : 文件所属组有写（w）权限
S_IROTH : 文件所属other有读（r）权限
S_IWOTH : 文件所属other有写（w）权限，等
一般情况下文件描述符应该从3开始，因为标准输入0(键盘),标准输出1(显示器),标准错误输出2已经被占用了。012是每个进程默认打开的.
.
每open一次文件，就能得到一个新的打开文件表项，因此每次open都可以得到一个独立的文件偏移量。
一个进程多次open同一个文件，由于得到不同的fd和打开文件表项，因此每次fd都拥有各自独立的文件偏移量。
.
3. fd=open(   );//fd是描述符，可以通过fd来判断是否成功
关闭文件close(fd)   返回值：成功返回0，否则返回-1
man 3 exit查看退出函数

例：
int fd;
fd=open(“f1”,O_RDWR);
if(fd==-1) {  
perror(“open”);   
exit(1);  }
int fd;
fd=open(“/home/f2”,O_RDWR|O_TRUNC|O_CREAT,0644);
if(fd==-1) {  
perror(“open”);   
exit(1);  }

#### 二. write函数头文件（man 2 write）

1. 写文件

  #include <unistd.h>

ssize_t write(int fd, const void *buf, size_t count);//ssize_t相当于int
	fd：文件描述符
	buf：要写入文件的数据区首地址
	nbytes：要写入文件的数据字节个数
返回值：成功返回已写字节数，失败返回-1

ps： 写常规文件时，write的返回值通常等于请求写的字节数count，而向终端设备或者网络写时则不一定
使用write 1 函数将刚刚读出的内容打印到屏幕上
使用write fd则是将内容写进某个文件
例：
int fd;
fd=open(“f1”,O_RDWR);
if(fd==-1) {  
perror(“open”);   
exit(1);  }
write(fd,”hello”,strlen(“hello”));
close(fd);
.

2. 读文件(man 2 read)
ssize_t read(int fd,void *buf,size_t nbytes);
参数：
	fd：文件描述符
	buf：保存读取数据的缓冲区
	nbytes：要读取数据的字节数
返回值：成功返回已读取字符数，失败返回-1
read返回0表明文件到达结尾。
如果希望能读完文件所有的数据，最好通过循环方式调用read，直到读到文件尾为止
例：
int fd;
char  buf[100];
fd=open(“f1”,O_RDWR);
if(fd==-1) {  
perror(“open”);   
exit(1);  }
read(fd,buf,sizeof(buf)-1);
close(fd);
.
3. fread函数read函数的区别
  fread函数是封装好的库函数，而read函数是系统函数，一般来说，fread效率更高； 
  读取文件的差别：fread函数功能更强大，可以读取结构体的二进制文件，但是如果是最底层的操作，用到文件描述符的话，用read会更好。
  .

  #### 三. lseek文件定位函数(man lseek)

  #include <sys/types.h>
  #include <unistd.h>
  .
1. off_t lseek(int fd, off_t pos, int whence);
参数:
	fd：文件描述符
	whence：取下面其中一个值
		SEEK_SET：代表文件开始，即设为pos
		SEEK_CUR：代表当前位置，即设为当前值+pos
		EEK_END：代表文件结尾，即设为文件长度＋pos（常用此项）
	pos：可正、负、0，代表相对whence的偏移值。
返回值:
成功返回设置后的当前文件偏移量，off_t相当于long。失败返回-1(置errno)

2. lseek常用的几种方式:
设置文件的某个绝对位置:lseek(fd, 值, SEEK_SET);
查找文件偏移量的当前位置:
     off_t where
     where=lseek(fd, 0, SEEK_CUR);
查找文件结尾:
     off_t where
     where=lseek (fd, 0, SEEK_END);





#### 四. time(man 2 time)

#include <time.h>

time_t time(time_t *tloc);
返回值：成功返回已读取时间，失败返回-1

#### 五. 获取文件属性stat(man 2 stat) 常见的文件属性数据类型为struct stat，可使用stat系列函数获取

1. int lstat(cosnt char *path,  struct stat *buf); 重点
      int stat(const char *path,  struct stat *buf);
      int fstat(int fd,  struct stat *buf);
功能：获取文件属性
stat：通过文件路径获取
fstat：通过文件描述符获取
lstat：通过文件路径获取，若查询符号链接文件，仅查其本身属性而不是所链接文件属性。
参数：
	path：文件路径
	fd：文件描述符
	buf：返回的文件属性（结构体）
返回值：成功返回0，否则返回－1（置errno）
2. struct stat 
      {
      dev_t	st_dev;		//文件所在的设备ID
      ino_t	st_ino; 		//i节点号
      mode_t	st_mode; 	//类型权限
      nlink_t	st_nlink; 	//硬链接个数
      uid_t	st_uid; 		//uid
      gid_t	st_gid; 		//gid
      dev_t	st_rdev; 	//若是特殊文件，设备ID
      off_t	            st_size; 	//文件大小（字节数）
      time_t	st_atime; 	//上次访问时间
      time_t  	st_mtime; 	//上次修改内容时间
      time_t	st_ctime; 	//上次文件信息修改时间
      blksize_t	st_blksize; 	//块大小（字节）
      blkcnt_t	st_blocks;	//文件块个数
      };

lstat举例(man 2 lstat)
struct stat  buf;
int rt;
rt=lstat(“/etc/passwd”,  &buf);
if(rt==-1)
	perror(“lstat”);

3. 判断文件类型——方法一  ----test1.c
例：
struct stat  buf;
lstat(“filename”,&buf);
//判断buf.st_mode & S_IFMT的值是否与各个文件类型宏相等，相等即为某类型。
switch(buf.st_mode & S_IFMT)    {
	case S_IFDIR:  printf("d");break;
	case S_IFLNK:  printf("l");break;
	case S_IFREG:  printf("-");break;
	default:printf("?");
}


判断文件类型——方法二
struct stat  buf;
lstat(“filename”,&buf);
宏操作返回真代表是该类型文件例:S_ISREG(buf.st_mode)//常规文件

三组用户九种访问权限
S_IRUSR  S_IWUSR  S_IXUSR
S_IRGRP  S_IWGRP  S_IXGRP
S_IROTH   S_IWOTH  S_IXOTH
三个权限修饰位：
S_ISUID   S_ISGID   S_ISVTX
文件访问权限判断
根据st_mode低9位进行判断，如
st_mode & S_IRUSR：非0用户可读，0用户不可读

#### 六. 用户/组ID与名字的转换

1. 通过用户ID得到用户信息
#include <pwd.h>
struct passwd *getpwuid(uid_t uid);
返回值：成功返passwd结构体指针，否则返NULL(置errno)
struct passwd{
char *pw_ name;//用户名.
char *pw_passwd; //用户密码
uid_t pw_uid;//用户ID
gid_t pw_gid;//用户所在组ID
char *pw_gecos; //实际用户名
char *pw_dir;//用户主目录
char *pw_shell;//用户所用Shell
};
.
2. 通过用户组ID得到用户组信息
#include <grp.h>
struct group *getgrgid(gid_t gid);
返回值：成功返回结构体指针，否则返回NULL(置errno)
struct group{
char *gr name;//组名
char *gr_passwd;//组密码
gid_t gr_gid;.//组ID
char **gr_mem;//组成员列表
};
.
3. 硬链接与符号链接  ---test4.c
硬链接
创建  
#include <unistd.h>
int link(char *pathname1, char *pathname2);
参数：
	pathname1表示已存在文件
	pthname2表示硬链接文件
返回值：成功返回0，失败返回-1（置errno）
删除：可以删除任何文件，不专指删硬链接
int unlink(char *pathname);
参数： pathname要删除的链接文件名
返回值：成功返回0，失败返回-1（置errno）
注意：删除文件时如果已经打开则要延迟到文件关闭后才真正删除
.
4. 符号链接(软连接)指向另一个已经存在的链接
创建#include <unistd.h>
	int symlink(char *actualpath, char *sympath);
参数：
	actualpath表示真实存在的文件或目录
	sympath表示符号链接文件
返回值：成功返回0，失败返回-1（置errno）
读取符号链接所指原文件名：
int readlink(char *pathname，char *buf, int bufsize);
参数：
	pathname：符号链接文件名
	buf：存放被链接文件名的缓冲区
	bufsize：缓冲区大小
返回值：成功返回实际写入缓冲区的字节数，失败返回-1
setuid，设置用户id(setuid位能让普通用户以root用户的角色运行只有root帐号才能运行的程序或命令)
setgid，设置组id
sticky，粘附位(可以理解为防删除位,如果没有写权限则这个目录下的所有文件都不能被删除,同时也不能添加新的文件)
.
5. 文件权限修饰位 ——setuid位
chmod   u+s   文件名
如果文件的拥有者权限原本有x则显示s，否则显示S。

6. 文件权限修饰位——setgid位 
setgid对目录设置后，任何用户在该目录下创建文件的属组都是该目录的属组
chmod  g+s  目录名
举例：
#groupadd  test           创建组test
#mkdir  /home/dir        创建目录
#chmod o+w  /home/dir   使其他用户可创建
#chgrp test  /home/dir     修改目录属组为test
#chmod g+s   /home/dir   修改目录setgid位
#ls –ld  /home/dir           查看目录权限
以后在/home/dir下新建文件和目录的属组都是/home/dir的属组。

7. chmod ［who］ ［+|-|=］ ［mode］ 文件名

u 表示用户（user），即文件或目录的所有者。
g 表示同组（group）用户，即与文件属主有相同组ID的所有用户。 
o 表示其他（others）用户。 
a 表示所有（all）用户。它是系统默认值。 
操作符号可以是： 
+ 添加某个权限。 
- 取消某个权限。 
= 赋予给定权限并取消其他所有权限（如果有的话）。 
数字设定法的一般形式为： 
chmod ［mode］ 文件名

我们必须首先了解用数字表示的属性的含义：4表示可读权限，2表示可写权限，1表示可执行权限，0表示没有权限，然后将其相加。所以 数字属性的格式应为3个从0到7的八进制数，其顺序是（u）（g）（o）。 

例如，如果想让某个文件的属主有“读/写”二种权限，需要把4（可读）+2（可写）＝6（读/写）

8. 文件权限修饰位——sticky位 
chmod  o+t  目录名//o是other
举例:
#useradd  u1               创建u1用户
#touch /home/dir/aa     当前用户创建aa
#touch /home/dir/bb     当前用户创建bb
#su -  u1 后   rm  /home/dir/aa    u1删除aa
exit   退出u1
#chmod o+t  /home/dir      设置sticky位
#su - u1  后   rm  /home/dir/bb    u1无法删bb
exit      退出u1

#### 七. dup/dup2

1. 重定向:将一个文件描述符（新）复制到另一个现有的文件描述符（旧）上，即指向现有的文件表项
两个函数：
	dup：新描述符是默认当前最小可用的文件描述符
	dup2：新描述符可以通过参数任意指定，如果指定的已经被占用，则自动将其关闭并重新使用
2. dup
#include <unistd.h>
int dup(int fd);
参数：现有文件描述符
返回值：成功返回新文件描述符（最小可用的文件描述符），错误返回-1（置errno）
说明：将最小可用文件描述符指向fd对应的打开文件描述，如dup(3)
3. 文件属性的修改
修改文件权限操作：
#include <sys/types.h>
#include <unistd.h>
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
两个函数的区别类似stat系列函数
说明：除非设置整个权限，否则需要首先调用stat获得mode，然后设置mode，再利用chmod进行修改
4. 修改系统umask值
文件对三组用户有功九种存取权限
umask屏蔽字：影响新建文件/目录的权限
新建文件或目录权限：
	使用命令方式：文件权限 = 666 & 屏蔽字反码
	目录权限 = 777 & 屏蔽字反码
	使用系统调用方式：文件或目录权限＝系统调用中权限 & 屏蔽字反码
命令umask和系统调用umask可以查看或设置屏蔽字
5. 超级用户能修改用户/组，文件拥有者能修改组，其他用户不能修改。
#include <sys/types.h>
#include <unistd.h>

int chown(const char *pathname,uid_t owner,gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int lchown(const char *path,uid_t uid,gid_t gid);
三个函数区别类似stat系列函数
.

#### 八. 目录的基本操作

1. 打开目录
#include <dirent.h>
#include <sys/types.h>
DIR *opendir(const char *path);
参数：path－目录名
返回值：成功返回DIR指针，否则返回NULL(置errno)
举例：
    DIR *d1;
    d1=opendir(“/home”);
    if(d1==NULL)
	perror(“opendir”);
2. 读目录
struct dirent *readdir(DIR *dirp);
参数：dirp－opendir返回的DIR指针
返回值：成功回dirent结构体指针，返回NULL表明目录结尾或错误（错误则置errno）
说明：
struct dirent{
	ino_t d_ino;//i节点号
	char d_name[];//目录下文件、子目录名
}
每次readdir读出一行记录，即一组文件名字及i节点号。
如果要读出目录文件中所有文件的名字，则要使用循环方式调用readdir，直到目录文件结尾
3. 关闭目录
int closedir(DIR *dirp);
参数:
	dirp：opendir返回的DIR指针
返回值：成功返回0，否则返回-1（置errno）
举例:
     DIR *d1;
     d1=opendir(“/home”);
     if(d1==NULL)
	    perror(“opendir”);
		closedir(d1);
4. 改变当前目录
#include <unistd.h>
int chdir(const char *path);
参数：
	path：目录名
返回值：成功返回0，否则返回－1（置errno）
注意：chdir切换的是本进程的当前目录，不是shell的当前目录，进程切换对shell的当前目录没影响
5. 创建目录
#include <sys/stat.h>
#include <sys/types.h>
int mkdir(const char *pathname, mode_t mode);
参数：
	pathname：要创建的新目录的路径名。
	mode：新建目录的权限。
返回值：执行成功时返回0，失败返回-1
6. 删除目录
#include <unistd.h>
int rmdir(const char *pathname);
参数:
	pathname：要删除的目录的路径名
返回值：成功时返回0，失败返回-1

九 

