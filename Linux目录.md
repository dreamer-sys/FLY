Linux目录

/*完成ls显示指定目录下文件列表的功能，如“ls 目录”、“ls  –l  目录”、“ls  -l”*/
一.打开目录
#include <dirent.h>
#include <sys/types.h>
DIR *opendir(const char *path);
参数：path－目录名
返回值：成功返回DIR指针，否则返回NULL(置errno)
举例
    DIR *d1;
    d1=opendir(“/home”);
    if(d1==NULL)
	perror(“opendir”);
	.
二.读目录
struct dirent *readdir(DIR *dirp);
参数：dirp－opendir返回的DIR指针
返回值：成功回dirent结构体指针，返回NULL表明目录结尾或错误（错误则置errno）
说明：
	struct dirent
	{
	ino_t  d_ino;//i节号
	char d_name[ ];//目录下文件/子目录名
	};
每次readdir读出一行记录，即一组文件名字及i节点号。
如果要读出目录文件中所有文件的名字，则要使用循环方式调用readdir，直到目录文件结尾。
.
三.关闭目录
int closedir(DIR *dirp);
参数
	dirp：opendir返回的DIR指针
返回值：
	成功返回0，否则返回-1（置errno）
举例：
     DIR *d1;
     d1=opendir(“/home”);
     if(d1==NULL)
	    perror(“opendir”);
		closedir(d1);
.
四.改变当前目录
#include <unistd.h>
int chdir(const char *path);
参数：
	path：目录名
返回值：
	成功返回0，否则返回－1（置errno）
注意：
	chdir切换的是本进程的当前目录，不是shell的当前目录，进程切换对shell的当前目录没影响。
.
五.创建目录
#include <sys/stat.h>
#include <sys/types.h>
int mkdir(const char *pathname, mode_t mode);
参数：
	pathname：要创建的新目录的路径名。
	mode：新建目录的权限。
返回值：
	执行成功时返回0，失败返回-1。
.
六.删除目录
#include <unistd.h>
int rmdir(const char *pathname);
参数：
	pathname：要删除的目录的路径名
返回值：
	成功时返回0，失败返回-1
