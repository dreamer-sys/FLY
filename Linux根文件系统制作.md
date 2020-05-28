一.BusyBox(软件)
BusyBox 是很多标准 Linux工具的一个单个可执行实现。BusyBox 包含了一些简单的工具，例如 cat 和 echo，还包含了一些更大、更复杂的工具，例如 
grep、find、mount 以及 telnet（不过它的选项比传统的版本要少）；有些人将BusyBox 称为 Linux 工具里的瑞士军刀。
从http://www.busybox.net/downloads/ 下 载busybox ，这里下载的是busybox-1.13.3.tar.gz，这和当前mini2440 开发板使用的版本是一致的。

二.步骤
1. 创建一个建立根文件系统目录的脚本文件create_rootfs_bash：
进入到/opt/studyarm目录，新建建立根文件系统目录的脚本文件create_rootfs_bash，内容如下：
	#!/bin/sh
	echo "------Create rootfs directons start...--------"
	mkdir rootfs
	cd rootfs
	echo "--------Create root,dev....----------"
	mkdir root dev etc boot tmp var sys proc lib mnt home
	mkdir etc/init.d etc/rc.d etc/sysconfig
	mkdir usr/sbin usr/bin usr/lib usr/modules
	echo "make node in dev/console dev/null"
	mknod -m 600 dev/console c 5 1
	mknod -m 600 dev/null c 1 3
	mkdir mnt/etc mnt/jffs2 mnt/yaffs mnt/data mnt/temp
	mkdir var/lib var/lock var/run var/tmp
	chmod 1777 tmp
	chmod 1777 var/tmp
	echo "-------make direction done---------"

	chmod +x create_rootfs_bash    //改变文件的可执行权限
	./create_rootfs_bash           //运行脚本，就完成了根文件系统目录的创建
2. 动态链接库直接用友善之臂的，先解压友善之臂的根文件包，拷贝lib的内容到新建的根文件目录lib内。
	cd /mnt/hgfs/share
	tar –zxvf root_qtopia.tgz –C /opt/studyarm
	cp –rfd /opt/studyarm/root_qtopia/lib/* /opt/studyarm/rootfs/lib/*
3. 交叉编译Bosybox
使用Busybox可以自动生成根文件系统所需的bin、sbin、usr目录和linuxrc文件
1）解压busybox
	cd /mnt/hgfs/share
	tar –zxvf busybox-1.13.3.tar.tgz –C /opt/studyarm
2）进入源码，修改Makefile文件：
	cd /opt/studyarm/busybox-1.13.3
	修改：
	CROSS_COMPILE ?=arm-linux- //第 164 行
	ARCH ?=arm //第 189 行
3）配置busybox
提示：友善之臂已经在光盘中提供了busybox的源代码包，在光盘\linux目录中，文件名为：busybox-1.13.3-mini2440.tgz（用户手册5.4章节介绍了解压安
装的方法），解压后里面包含了友善之臂提供的缺省配置文件：fa_config(输入命令“cp fa.config .config”可以调用该配置)，一般用户直接使用缺省文件
就可以了，这样生成的busybox和root_qtopia中的是完全一致的。但为了对它的配置了解更多一些，可以参考原文作者的如下步骤：
	输入 make menuconfig 进行配置
	(1) 、Busybox Settings--->
		General Configuration--->
		[*] Show verbose applet usage messages
		[*] Store applet usage messages in compressed form
		[*] Support –install [-s] to install applet links at runtime
		[*] Enable locale support(system needs locale for this to work)
		[*] Support for –long-options
		[*] Use the devpts filesystem for unix98 PTYs
		[*] Support writing pidfiles
		[*] Runtime SUID/SGID configuration via /etc/busybox.config
		[*] Suppress warning message if /etc/busybox.conf is not readable
		Build Options--->
		[*] Build BusyBox as a static binary(no shared libs)
		[*] Build with Large File Support(for accessing files>2GB)
		Installation Options->
		[]Don’t use /usr
		Applets links (as soft-links) --->
		(./_install) BusyBox installation prefix
		Busybox Library Tuning --->
		(6)Minimum password legth
		(2)MD5:Trade Bytes for Speed
		[*]Fsater /proc scanning code(+100bytes)
		[*]Command line editing
		(1024)Maximum length of input
		[*] vi-style line editing commands
		(15) History size
		[*] History saving
		[*] Tab completion
		[*]Fancy shell prompts
		(4) Copy buffer size ,in kilobytes
		[*]Use ioctl names rather than hex values in error messages
		[*]Support infiniband HW
	(2) 、Linux Module Utilities--->
		(/lib/modules)Default directory containing modules
		(modules.dep)Default name of modules.dep
		[*] insmod
		[*] rmmod
		[*] lsmod
		[*] modprobe
		-----options common to multiple modutils
		[ ] support version 2.2/2.4 Linux kernels
		[*]Support tainted module checking with new kernels
		[*]Support for module .aliases file
		[*] support for modules.symbols file
	(3) 、在busybox中配置对dev下设备类型的支持
		Linux System Utilities --->
		[*]Support /etc/mdev.conf
		[*]Support command execution at device addition/removal
4. 编译busybox(在rootfs目录下会生成目录bin、sbin、usr和文件linuxrc的内容)
	make TARGET_ARCH=arm PREFIX=/opt/rootfs all install
5. 使用mknod指令来添加设备
	mknod /dev/fb0 c 29 0 /*建立显示器设备文件*/
	mknod /dev/ts c 254 0 /*建立触摸屏设备文件*/
6. 动态添加内核模块
	modprobe 8139too #挂载8139too模块
	modprobe vfat #挂载vfat模块
	insmod / lib/modules/2.6.9-11.EL/Kernel/drivers/pci/hotplug/capiphp.ko