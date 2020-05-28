内核配置make menuconfig(或xconfig等)时，从Kconfig中读出菜单，用户选择后保存到.config的内核配置文件中。在内核编译时，主Makefile调用这个.config，就知道了用户的选择。
*上面的内容说明了，Kconfig就是对应着内核的配置菜单。如果要想添加新的驱动到内核的源码中，可以修改Kconfig,这样就可以选择这个驱动，如果想使这个驱动被编译，要修改Makefile
————————————————
.
1.建立.c文件，末尾有模块加载和卸载函数
module_init(dev_init);/*模块初始化，仅当使用insmod/podprobe 命令加载时有用，如果设备不是通过模块方式加载，此处将不会被调用*/
module_exit(dev_exit);/*卸载模块，当该设备通过模块方式加载后，可以通过rmmod 命令卸载，将调用此函数*/
MODULE_LICENSE("GPL"); /*版权信息*/
MODULE_AUTHOR("ubuntu."); /*开发者信息*/
.
2.打开 linux-2.6.32.2/drivers/char/Kconfig 文件
Kconfig
作用：Kconfig用来配置内核，它就是各种配置界面的源文件，内核的配置工具读取各个Kconfig文件，生成配置界面供开发人员配置内核，最后生成配置文件.config
语法：
例1：
config BUTTONS_LEDS_ZHAO     //定义的模块名称
        tristate "Mini2440 button  and leds sample" //图形化界面显示的名称
        depends on MACH_MINI2440 //基于mini2440
        default m if MACH_MINI2440 //默认编译入内核稍后会更改成编译为module形式
        help                                     //帮助信息
          Mini2440  button and leds  module sample.
例2：
menu条目用于生成菜单，其格式如下：
menu "Unicode Trans Support"
config SUPPORT_CHARSETDET
            bool "Support Match Character Set Codepage"
            default n
.            
config SUPPORT_ISO88591_CP28591
            bool "Codepage ISO8859-1 Latin 1"
            default y
config SUPPORT_ISO88592_CP28592
            bool "Codepage ISO8859-2 Central European"
            default y
config SUPPORT_ISO88593_CP28593
            bool "Codepage ISO8859-3 Latin 3"
            default y
config SUPPORT_ISO88594_CP28594
            bool "Codepage ISO8859-4 Baltic"
            default y
config SUPPORT_ISO88595_CP28595
            bool "Codepage ISO8859-5 Cyrillic"
            default y
    endmenu
 menu之后的“Unicode Trans Support”是菜单名，menu和endmenu间有很多config条目，在配置界面中如下所示：
         Unicode Trans Support--->
                       [ ] Support Match Character Set Codepage 
                       [*] Codepage ISO8859-1 Latin 1
			[ ] Codepage ISO8859-2 Central European
config的类型有5种:
1.string(字符串)：表示该CONFIG宏可以设为一串字符,比如#define CONFIG_XXX "config test"
2.hex(十六进制)：表示该CONFIG宏可以设为一个十六进制,比如#define CONFIG_XXX 0x1234
3.integer(整数)：表示该CONFIG宏可以设为一个整数,比如#define CONFIG_XXX 1234
4.tristate表示三态：编译进内核（y)，编译成模块(m)，不编译(n)
5.boolean 主要有两种y或n
depend/requires则表示依赖项,比如depends on XXX 表示当前宏需要CONFIG_ XXX宏打开的前提下,才能设置它 (注意依赖项的config参数只有bool或tristate才有效)
default y:表示默认是勾上的,当然也可以写为default m或者default n
default缺省的编译选项 m表示默认该文件表示以模块方式编译
select：反向依赖,如果当前项选中，那么也选中select后的选项，和depends on刚好相反,比如 selecton XXX表示当前宏如果是y或者m,则会自动设置XXX=y或者m(注意参数只有bool或tristate才有效)
range：范围，用于hex和integer；range A B表示当前值不小于A，不大于B；设置用户输入的数据范围,比如range 0 100表示数据只能位于0~100
• prompt:     提示信息,如果对于choice而言,则会用来当做一个单选框入口点的标签
comment：注释
choice/endchoice
choice会生成一个单选框,里面通过多选一方式选择config,需要注意choice中的config参数只能bool或tristate
help是帮助信息
.
3.Makefile
obj-$(CONFIG_BUTTONS_LEDS_ZHAO) += buttons_leds_zhao.o
注：CONFIG_BUTTONS_LEDS_ZHAO应与config BUTTONS_LEDS_ZHAO相同
.
4..config
make mini2440_defconfig ARCH=arm生成.o文件
make menuconfig
按空格键使该行最前面的符号变为<M>（表示为编译成Module形式，默认<*>为编译入内核）
5.举个例子：
假设想把自己写的一个flash的驱动程序加载到工程中，而且能够通过menuconfig配置内核时选择该驱动该怎么办呢？可以分三步：
第一：将你写的flashtest.c 文件添加到/driver/mtd/maps/ 目录下。
第二：修改/driver/mtd/maps目录下的kconfig文件：
config MTD_flashtest     //定义的模块名称
tristate “ap71 flash"      //图形化界面显示的名称
这样当make menuconfig时 ，将会出现 ap71 flash选项。
第三：修改该目录下makefile文件。
添加如下内容：obj-$(CONFIG_flashtest) += flashtest.o
这样，当你运行make menucofnig时，你将发现ap71 flash选项，如果你选择了此项。该选择就会保存在.config文件中。当你编译内核时，将会读取.config文件，当发现ap71 flash 选项为yes 时，系统在调用/driver/mtd/maps/下的makefile 时，将会把 flashtest.o 加入到内核中。即可达到你的目的
————————————————
回到linux-2.6.32.2 源代码根目录位置，执行make modules，就可以生成我们所需要的内核模块文件 mini2440_hello_module.ko 了，如图：至此，我们已经完成了模块驱动的编译。
