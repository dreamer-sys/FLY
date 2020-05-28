uboot移植

一.  修改Makefile顶层
定义交叉编译工具链和开发板配置选项：
diff -aurNp u-boot-2009.11/Makefile u-boot-2009.11_tekkaman/Makefile
--- u-boot-2009.11/Makefile 2009-12-16 06:20:54.000000000 +0800
+++ u-boot-2009.11_tekkaman/Makefile  2010-03-28 17:16:12.000000000 +0800
@@ -157,7 +157,7 @@ sinclude $(obj)include/autoconf.mk
# load ARCH, BOARD, and CPU configuration
include $(obj)include/config.mk
export  ARCH CPU BOARD VENDOR SOC
-
+CROSS_COMPILE = arm-tekkaman-linux-gnueabi-//修改处
# set default to nothing for native builds
ifeq ($(HOSTARCH),$(ARCH))
CROSS_COMPILE ?=
@@ -3046,6 +3046,9 @@ smdk2400_config  :  unconfig
smdk2410_config :  unconfig
@$(MKCONFIG) $(@:_config=) arm arm920t smdk2410 samsung s3c24x0
+
+mini2440_config :  unconfig											//修改处
+ @$(MKCONFIG) $(@:_config=) arm arm920t mini2440 tekkamanninja s3c24x0 //修改处
SX1_stdout_serial_config \
SX1_config:  unconfig
二. 在/board中建立mini2440目录和文件
1. 由于上一步板子的 vender 中填了 tekkamanninja ，所以开发板 mini2440 目录一定要建在/board子目录中的 tekkamanninja 目录下 ，否则编译出错。
cd board
mkdir -p tekkamanninja/mini2440
cp -arf sbc2410x/* tekkamanninja/mini2440/
cd tekkamanninja/mini2440/
mv sbc2410x.c mini2440.c
2. 修改mini2440 目录下的Makefile文件:
@@ -25,7 +25,7 @@ include $(TOPDIR)/config.mk
LIB  = $(obj)lib$(BOARD).a
-COBJS  := sbc2410x.o flash.o
+COBJS  := mini2440.o flash.o//修改处
SOBJS := lowlevel_init.o
SRCS := $(SOBJS:.o=.S) $(COBJS:.o=.c)
三.在include/configs/中建立开发板配置文件
cp include/configs/sbc2410x.h include/configs/mini2440.h
四.测试编译环境
在 U-boot 源码的根目录下：
make mini2440_config
Configuring for mini2440 board...
make
问题：
1. 配置出错：
make mini2440_config
Makefile:????: *** 遗漏分隔符 。 停止。
请在 U-boot 的根目录下 Makefile 的
“@$(MKCONFIG) $(@:_config=) arm arm920t mini2440 tekkamanninja s3c24x0 ”
前加上“Tab”键，这是 Makefile 的规则：所有命令都必须以“Tab”开头。
2. 如果编译时出现以下错误（这是编译器的问题，没出错就不要修改）：
uses hardware FP, whereas u-boot uses software FP
	修正的方法：
diff -aurNp u-boot-2009.11/cpu/arm920t/config.mk u-boot-2009.11_tekkaman/cpu/arm920t/config.mk
--- u-boot-2009.11/cpu/arm920t/config.mk  2009-12-16 06:20:54.000000000 +0800
+++ u-boot-2009.11_tekkaman/cpu/arm920t/config.mk  2010-03-28 17:16:12.000000000 +0800
@@ -21,7 +21,8 @@
# MA 02111-1307 USA
#
-PLATFORM_RELFLAGS += -fno-common -ffixed-r8 -msoft-float
+PLATFORM_RELFLAGS += -fno-common -ffixed-r8
+#-msoft-float
PLATFORM_CPPFLAGS += -march=armv4
# =========================================================================