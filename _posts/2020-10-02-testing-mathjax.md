---
layout: post
title: HiLink-76x8 OpenWRT
tags: Openwrt 14.07
math: true
date: 2020-10-02 15:32 +0800
---

## Openwrt 14.07

### `MTK提供的Openwrt版本`

> 版本名称是Barrier Breaker 版本号是14.07

### `Openwrt编译环境`

> 编译虚拟机在百度网盘（使用vmware虚拟机）：

> 链接: https://pan.baidu.com/s/1fE5BrAIC8I3tjR1i6tv6cw 提取码: n86v

### `虚拟机的构建方法：`

> 这里使用开源的VirtualBox作为演示的虚拟机方案

> 下载地址：https://www.virtualbox.org/

> 下载的虚拟机硬盘文件是一个压缩包 拆分成两个文件，解压时 只需要解压ubuntu.7z.001既可

> 下载完成后解压

> 安装virtualbox虚拟机软件

> 软件下载地址：https://www.virtualbox.org/

### 新建一个虚拟机

> 类型选择Linux

> 版本选择 Ubuntu(64-bit)

> 内存建议2G以上

> 使用已有的虚拟硬盘文件

> 注册虚拟硬盘：

### 设置CPU 网络等参数

> 网卡1设置为 主机网络 用来 本机访问虚拟机

> 网卡2 设置为桥接网卡

> 桥接网卡 选为 可以联网的网口 用作虚拟机连接外网

> 创建完成后 启动虚拟机

> 用户名：luke  密码： luke

> 启动进到系统后

> 查询网口

> Eth0 ： 既主机HOST 网络 用来和 本地通信 （IP地址 每个系统可能不同）

> Eth1 ： 既桥接网卡网络 用于虚拟机连接外网 （IP地址 由桥接的网络分配）

> SDK源码目录

> 打开终端 进入到MtkOpenwrt目录下

```
make menuconfig
#可以先使用默认配置进行编译

make V=99 -j4 #(-j取决于 虚拟机使用了几核CPU)

#编译完成，编译出的固件在路径bin/ramips/下

#名称为：openwrt-ramips-mt7628-mt7628-squashfs-sysupgrade.bin

#如果是一个新安装的ubuntu系统 编译一个openwrt一般需要安装的软件大概有：

sudo apt-get update

sudo apt-get install build-essential gawk git vim subversion zlib1g-dev libncurses5-dev quilt texinfo cmake make python2 samba openssh-server
```

> `#如果定制 添加自己的程序 会引起固件变大 SDK默认的 固件大小最大只能到8M，如果超过8M 默认是编译不出固件的 #需要修改：target/linux/ramips/image/Makefile中 的Default8M 改为16 或32`

> `虚拟机的用户名luke  密码：luke`

> 虚拟机中自带了Mtk Openwrt的源码，/home/luke/MtkOpenwrt,  已经有默认配置， 可以直接编译。

> 编译时可能会遇到问题

```
make kernel_menuconfig->
Ralink Module  --->

[*] WiFi Driver Support  --->

[]   WiFi packet forwarding
#把WiFi packet forwarding  的*去掉 重新编译
```

> 虚拟机共享目录到主机：

```
sudo apt-get install samba

#修改/etc/samba/smb.conf,在文件末尾添加：
[share]
    comment = Shared Folder with username and password
    path = /home/luke
    public = yes
    writable = yes
    valid users = luke
    available = yes
    browseable = yes
    guest ok = no
#添加samba的用户名和密码
sudo smbpasswd -a luke
```

### `Openwrt配置编译`

> SDK中已经存在一个默认配置，满足路由的基本功能，客户也可以根据自己的需求，进行自定义配置

> 命令：

`make menuconfig`

> WIFI驱动配置在 MTK Properties-> Drivers --->kmod-mt7628 下 如需要STA功能可以选择：AP-Client Support

```
使用命令 
make V=99
#编译结果保存在bin/ramips/目录下
#生成固件名：openwrt-ramips-mt7628-mt7628-squashfs-sysupgrade.bin
```

### `使用reg 命令控制7688/7628的寄存器`

> MtkOpenwrt可以使用reg命令对7688/7628的寄存器进行配置

> 在需要在配置固件时加入reg命令：

```
make menuconfig ->
	MTK Properties->
		Applications-> 
			<*> reg #勾选上reg， 升级固件后重启。
```

1.  reg s 0 设置寄存器地址的偏移量 0 表示 寄存器地址设置为MT7688的datasheet 第五章中描述的sysctl base address。`在使用其他命令前必须先使用一次`

2.  reg r 60 读取GPIO1_MODE 寄存器的值

3.  reg w 60 xx 设置GPIO1_MODE 寄存器的值

> 示例：默认代码编译完成后 GPIO1_MODE的寄存器默认设置UART1为GPIO模式

> 可以进行如下操作把UART1_MODE设置为UART1-Lite模式

```
reg s 0
reg r 60   #读取GPIO1_MODE的寄存器值为0x55044410
```

> 通过读取7688的datasheet可以确认GPIO1_MODE的24：25bit 是用于控制UART1的引脚的模式，0表示把UART1引脚配置为uart1-lite，1表示把UART1引脚配置为GPIO模式，使用命令

```
reg w 60 0x54044410 #（把24：25bit置为0）
```

> 验证UART1是否是串口：使用echo aaaaaaa >/dev/ttyS0(UART1 在系统中的设备)

> 其他命令选项，可以通过reg  --help 学习用法

> 设置功能脚的引脚作为GPIO也是类似用法

> 使用reg 命令控制作为GPIO引脚的输入输出功能

> 通过reg命令可以控制寄存器使相应引脚进入到GPIO模式

> reg命令 还用来控制GPIO引脚作为输入还是输出

> 示例：GPIO0引脚默认既作为GPIO引脚，通过reg命令控制寄存器设置GPIO0引脚作为GPIO的输出模式并输出高低点评：(GPIO0的gpio索引号是11)

> 参看MT7688_Datasheet_v1_4.pdf手册的5.8一节，描述了7688的GPIO寄存器

> GPIO_CTRL_0寄存器用以控制0~31号gpio的方向，设置为0表示输入，设置1表示输出

> reg r 600 读取GPIO_CTRL_0寄存器的值为0

> 把gpio11设置为输出模式 可以使用命令 :

`reg w 600 0x800`

> 设置gpio11为高电平：写入GPIO_DSET_0寄存器的bit11控制gpio11为高电平

`reg w 630 800`

> 设置gpio11为低电平：写入GPIO_DSET_0寄存器的bit11 控制gpio11为低电平

`reg w 640 800`

> 设置引脚为GPIO的输入类似， 对于GPIO11 只需要控制GPIO_CTRL_0的bit11为0即可

### `配置WIFI的STA功能`

> 在这个版本的openwrt的WIFI驱动是一个AP+STA共存的apclient功能，源码对于APClient的支持目前仅限于命令行 和 配置文件，不支持在luci页面中进行配置，只能通过命令行对周围的AP进行扫描并配置。

> 扫描AP命令：

```
iwpriv ra0 set SiteSurvey=1;sleep 3;iwpriv ra0 get_site_survey
```

> 配置STA

1.  通过修改/etc/config/wireless配置文件

```
config wifi-iface
option device   mt7628
option ifname   ra0
option network  lan
option mode     ap
option ssid     mt7628-1912
option encryption psk2
option key      12345678
```

> 添加下面的配置参数

```
option ApCliEnable '1'
option ApCliSsid 'zhongxing'
option ApCliAuthMode 'WPA2PSK'
option ApCliEncrypType 'AES'
option ApCliWPAPSK '12345678'
```

> 完成后 配置WAN口到apcli0

> 修改/etc/config/network

```
config interface 'wan'
option proto 'dhcp'
option ifname 'apcli0'
```

```
#重启网络
/etc/init.d/network restart
```

### `Openwrt的出厂配置恢复方法`

> 在命令行输入:

```
umount  /dev/mtdblock6; firstboot -y
```

> firstboot 输入Y 确认恢复默认

> Openwrt将会被清除已有的配置信息，恢复为默认出厂配置

### `Openwrt中的网口配置`

> 出厂固件默认的网口配置对照底板示意：

```
#网口从左到右分别对应7688/7628的 P0 P1 P2 P3 P4引脚
#Openwrt下的所谓的LAN WAN口是通过脚本进行配置的
#./build_dir/target-mipsel_24kec+dsp_uClibc-2/switch/ipkg-ramips_24kec/switch/lib/network/switch.sh中
setup_switch()
{
	#configEsw LLLLW  配置P4为WAN口
	configEsw WLLLL     配置P0为WAN口
}
#LLLLW  （P4为WAN口）在底板上的位置如图：
```

> WLLLL (P0为WAN口)在底板上的位置如图：

```
#如临时需将WAN口更换为LAN口 ， 可以单板上修改/lib/network/switch.sh文件，setup_switch()
{
	configEsw WLLLL #如果W在左边 则改到右侧， 如果在右侧则改到左侧
}
```

### `在openwrt中添加自己的应用并编译到固件中`

> 下面用helloworld工程示例如何在openwrt中添加一个应用

> 在package下创建一个目录 helloworld

> 在helloworld目录下创建Makefile文件：文件见附件：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d07b6e01-faf4-4dc4-9a47-7e952ac9086a/Makefile.txt

> Makefile的规则描述，详细文档可以参考：https://openwrt.org/docs/guide-developer/packages

> 示例代码打包见附件：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a207b826-d6f0-4750-9a3f-8237f6bb04a0/helloworld.tar.gz

> 直接解压到openwrt的package目录下

> Helloworld目录下的src目录是存放源代码的地方：

> 示例中src目录下的Makefile才是真正用来对代码进行编译的Makefile。

> 客户也可以根据自己的需求编写makefile ，需要注意的几个地方 a. openwrt编译工具的名字是：mipsel-openwrt-linux-uclibc-开头的交叉编译器

> 不能直接使用mipsel-openwrt-linux-uclibc-ld进行链接。

> 如果使用示例中带的Makefile，对于简单的工程是可以的，它可以遍历src目录所有的.c文件进行编译

> Makefile的描述：

> 文件创建完成后，使用make menuconfig对openwrt进行配置：

> 选中后 进入到工程目录下进行make 编译完成后将固件升级后

> 编译时库的依赖问题：

> 如果使用了pthread多线程库

> 可以在外层Makefile中添加如下：

> 还有一种方法，可以在编译时欺骗openwrt的编译过程

> 在Makefile中添加：

```
define Package/helloworld/extra_provides
echo "libpthread.so.0"
```

> 这样在编译时，就不会出现因为缺少某个库而报错了。但应用实际是运行不了的，需要把相应的库copy到系统库目录下才可以。

> 上电启动脚本：

> Openwrt的上电执行/etc/init.d下的脚本 并按照脚本中的START变量大小依次执行。

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/84a3d64e-e9c9-49a8-af0b-8e7f5d89a9bf/hello.init

> 启动脚本的写法参见：

### `使用C语言控制GPIO`

> 代码示例：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/69e0e71b-daf0-4475-a0e7-45cc754d30e1/gpio.tar.gz

> 直接替换package/ramips/applications/gpio下的文件即可，替换完成后使用make menuconfig 添加gpio：

> MTK Properties  --->Applications  ---> <*> gpio.............................................. Command to config gpio

> 内核的补丁：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/07b40f71-ddd0-4ffe-988d-ef2f881be4a1/072-mtkgpio.patch

> 把补丁放到target/linux/ramips/patches/目录下

```
gpio getdir [gpio]
gpio setdir [gpio] [dir]     dir----0:IN 1:OUT
gpio read   [gpio]
gpio write  [gpio] [val]      val----0/1
gpio notify [gpio] [pid]     notify pid and proc signal
gpio led    [gpio] [on] [off] [blink] [reset] [times]
```

### MTK openwrt的GPIO驱动中实现了三个功能

1.  设置GPIO方向

> 操作示例：

```
#设置GPIO 11方向：
gpio setdir 11 1 #(0: IN  1: OUT)
#获取GPIO 11方向：
gpio getdir 11
#读取GPIO 11值：
gpio read 11
#写入GPIO 11为 1 ：需要把GPIO 11的方向设置为OUT
gpio setdir 11 1
gpio write 11 1
```

2.  中断通知

> 代码示例：

> GPIO驱动实现了一套中断通知机制，设置给定GPIO触发中断，绑定GPIO到某个进程上，进而使用信号通知进程，用于按键处理，根据按键按住的时间长短有三种信号

> SIGUSR1    按键按住大于6S

> SIGUSR2    按键大于6S

> SIG42 按住时间上小于250ms

> 使用方法：

```
gpio notify 38 `pidof gpio_notify` #绑定GPIO38 到 gpio_notify进程上
```

> Gpio_notify的源码附件：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/448a1589-58d1-42f2-b853-6653daf8ffb1/notify.tar.gz

3.  LED控制

> 由内核定时器控制LED的闪烁，控制代码见：

```
build_dir/target-mipsel_24kec+dsp_uClibc-2/linux-ramips_mt7628/linux-14/drivers/char/ralink_gpio.c：
ralink_gpio_led_do_timer函数，实际控制代码宏#if defined (RALINK_GPIO_HAS_9532)的范围内
```

> 驱动需要 6个参数：

> GPIO： gpio号

> On：

> Off：   On+off表示闪烁比例，具体闪烁时间和 多个参数有关。

> on off的大致时间单位是100ms 既：on=1 off=1表示亮100ms，灭100ms

> Blinks：Blinks表示闪烁的次数（一个亮灭为一次）

> Rests：  blinks/rests 这两个参数表示闪烁/熄灭次数

> 比如 on=1 off=1 blink=10 rests=10表示 闪烁10/2次。

> Times：控制次数 1

> 例如：on = 1 off = 1 blink = 10 rests = 10 times =1

> 表示亮100ms 灭100ms 一共循环10/2 共5个周期 然后灭10/2 5个周期，循环一次

> 这个驱动的LED控制比较复杂，如果需求较简单的话 ，可以参照下面的设置：

> 长亮：on >= 4000

> 长灭：off >= 4000  或 rests >= 4000

> 快闪：on = 1 off =1 blink=4000

> 慢闪：on = 5 off = 5 blink = 4000

### `VLAN设置`

> MTK openwrt的默认vlan配置

> 这里大致介绍一下交换芯片的VLAN管理：

### `Port和vlan相关属性`

> `PORT属性：`

> PVID：端口的缺省vlan ID，当收到的数据包不带vlan tag的時候，芯片会给数据包打上PVID，然后进行转发

> `Vlan属性：`

> vlan有三個重要的属性：VID，member port 和 untag port

> VID：唯一标识一个vlan

> Member Port：Vlan的成员端口

> 当端口收到一個带vlan的数据包的時候，芯片会首先判断该端口是否是数据包所属vlan的成员端口，如果不是，直接丢弃，反之通过，当芯片要转发一个数据包的時候，只会把数据包转发到所属vlan的成员端口。

> untag port：需要去除vlan tag的端口

> 当端口要发出某一数据包的時候，芯片会判断该数据包所属vlan在本端口是否是untag的，如果是，就去掉vlan tag，反之保留。

> `数据收发流程`

> `芯片配置`

> `配置vlan的VID`

> 7628/7688一共有8个寄存器，用来记录VID，每个寄存器可以记录两个VID，一共可记录16个VID

> 每个VID 12位，VID的范围是0 - 4095

> Mtkopenwrt的默认vlan设置命令：switch reg w 50 2001

> 既 VID0设置为VLAN1，VID1设置为VLAN2

> `配置vlan的member port`

> 为1则对应port是该vlan的成员端口

> 当设置P0为WAN口时，是这么设置MEM PORT的：

> switch reg w 70 ffff417e

> Vlan0 mem port： 7e的二进制形式：0111 1110 表示把端口P1 P2 P3 P4 P5 P6都加入到VLAN1中，

> Vlan1 mem port： 41的二进制形式：0100 0001 表示把端口 P0 P6加入到VLAN2中

> P6是交换芯片连接CPU的端口

> `配置vlan的untag port`

> 为1则该vlan在对应port是untag的

> `配置端口的PVID`

> 每个寄存器控制两个端口的PVID，默认PVID为1

> 配置Port 0的PVID为2 可以使用命令switch reg w 40 1002 (1002是16进制，高位的1是Port 1的PVID设置为1)

### `支持SD卡`

> 需要配置openwrt

```
make menuconfig
#驱动支持

MTK Properties  --->

	Drivers  --->

		<*> kmod-mtk-mmc......................................... MMC/SD card support

#配置文件系统支持：

	Kernel modules  --->

		Filesystems  --->

#配置内核

make kernel_menuconfig:

	Ralink Module  --->

		[*] One Port Only
```

### `支持网络共享samba`

```
对openwrt做配置
make menuconfig
	Kernel modules  --->
		Filesystems  --->
			<*> kmod-fs-cifs................................................ CIFS support

#配置LUCI页面：

LuCI  --->
	Applications  --->
		<*> luci-app-samba.................... Network Shares - Samba SMB/CIFS module

#配置samba

MTK Properties  --->
	Applications  --->
		<*> samba-server................................................ Samba Server

#修改配置文件：package/network/services/samba36/files/samba.config

config samba
option 'name'           'OpenWrt'
option 'workgroup'      'WORKGROUP'
option 'description'        'OpenWrt'
option 'homes'          '1'
option 'interface' 'wan wwan lan'  #允许访问网口
config sambashare
option browseable 'yes'
option name 'Share'
option path '/mnt/sda1'
option users 'root,nobody'  #允许用户
option read_only 'no'
option guest_ok 'yes'
option create_mask '0700'
option dir_mask '0700'
```

### `设置STA功能`

> MT7628 Mtkopenwrt的WIFI驱动是ApCli驱动，既可以作为AP 同时也可以作为STA连接其他无线路由器

> 需要配置驱动：

```
make menuconfig
	MTK Properties  --->
		Drivers  --->
			<*> kmod-mt76................................... MTK MT7628 wifi AP driver  --->
				WiFi Operation Modes  --->
					[*] AP-Client Support
```

> 选中AP-Client Support 后 退出 编译固件

> 这里介绍WIFI的配置命令：

### 扫描附近AP：

```
iwpriv ra0 set SiteSurvey=1; sleep 3; iwpriv ra0 get_site_survey
```

### 设置STA：

### `直接修改/etc/config的配置文件`

> 示例：

```
config wifi-iface
option device   mt7628
option ifname   ra0
option network  lan
option mode     ap
option ssid     mt7628-1912
option encryption psk2
option key      12345678
```

> 添加下面的配置参数

```
option ApCliEnable '1'
option ApCliSsid 'Hi-Link_PLC'
option ApCliAuthMode 'WPA2PSK'
option ApCliEncrypType 'AES'
option ApCliWPAPSK '12345678'

#修改option channel 为实际连接的AP的信道

config wifi-device      mt7628
option type     mt7628
option vendor   ralink
option band     4G
option channel  6
option auotch   2

#修改后保存。

#再对/etc/config/network进行修改

#添加：

config interface 'wwan'
option proto 'dhcp'
option ifname 'apcli0' #修改网口为apcli0

#修改/etc/config/firewall:

#在wan域中添加list network 'wwan'

config zone
option name 'wan'
option input 'REJECT'
option output 'ACCEPT'
option forward 'REJECT'
option masq '1'
option mtu_fix '1'
list network 'wan'
list network 'wan6'
list network 'wwan'

#重启网络 
/etc/init.d/network restart
```

1.  当然也可以通过UCI命令来设置：

```
#设置STA参数：
uci set wireless.@wifi-iface[-1].ApCliEnable=1
uci set wireless.@wifi-iface[-1].ApCliSsid=JDCwifi_0661
uci set wireless.@wifi-iface[-1].ApCliAuthMode=WPA2PSK
uci set wireless.@wifi-iface[-1].ApCliEncrypType=AES
uci set wireless.@wifi-iface[-1].ApCliWPAPSK=sycdpi191
uci set wireless.mt76channel=1
uci commit

#设置网络：

uci set network.wwan=interface
uci set network.wwan.ifname=apcli0
uci set network.wwan.proto=dhcp
uci commit

#设置防火墙：

uci add_list [firewall.@zone[1].network=wwan](mailto:firewall.@zone[1].network=wwan)
uci commit

#设置完成后重启网络

#iwpriv apcli0 set ApCliAutoConnect=1
/etc/init.d/network restart
```

> 也可以使用纯命令进行设置：

```
iwpriv apcli0 set ApCliEnable=0
ifconfig apcli0 0
iwpriv apcli0 set ApCliSsid=HUAWEI-3Q6QQW
iwpriv apcli0 set ApCliWPAPSK=sycdpi191
iwpriv apcli0 set ApCliAuthMode=WPA2PSK
iwpriv apcli0 set ApCliEncrypType=AES
iwpriv apcli0 set ApCliAuthMode=WPA2PSK
iwpriv apcli0 set ApCliEncrypType=AES
iwpriv apcli0 set Channel=8
iwpriv apcli0 set ApCliAutoConnect=1
iwpriv apcli0 set ApCliEnable=1

#查询是否连接到AP上 可以使用命令 iwconfig

iwpriv apcli0 set ApCliAutoConnect=1
#这个命令是用来 当AP信道切换了 ，sta会主动搜索信道连接到新的信道上
#可以用来防止有些ap会切换信道 导致连不上的问题
#不过连上之后最好关掉 否则可能会一直切换信道 导致其他设备连不上模块的AP

iwpriv apcli0 set ApCliAutoConnect=0
```

### `隐藏SSID`

```
uci set wireless.@wifi-iface[-1].hidden=1
uci commit
/etc/init.d/network restart
```

### `串口使用`

> Mt7628/7688共有3个串口，mtkopenwrt使用了串口UART0作为调试串口，默认波特率是57600

> 设置Uart1为串口（串口下标从0开始）

> 1) 修改源码：build_dir/target-mipsel_24kec+dsp_uClibc-0.9.33.2/linux-ramips_mt7628/linux-3.10.14/drivers/char/ralink_gpio.h

> 的514行，

> #define RALINK_GPIOMODE_DFT     (RALINK_GPIOMODE_UART2 | RALINK_GPIOMODE_UART3) | (RALINK_GPIOMODE_SPI_CS1) | (RALINK_GPIOMODE_WDT)

> 7688的默认把UART2  UART3  SPI_CS1  WDT功能脚作为GPIO，将RALINK_GPIOMODE_UART2 去掉，编译完成后升级固件即可 （注意 代码中宏 串口是从1开始， RALINK_GPIOMODE_UART2 表示寄存器中的UART1串口 ）

> UART2默认的波特率是9600 ，固件上对应的串口设备名称： /dev/ttyS0

> 2) 启动后通过命令配置为串口：

> 设置GPIO1_MODE寄存器，设置25:24bit为 0

> 临时设置方法：

> 把串口2复用脚 设置为串口功能的命令：

```
reg s 0
reg r 60 #读取GPIO1_MODE寄存器的值，读取到GPIO1_MODE的寄存器值为0x55444410，把25:24 bit设置为0
reg w 60 0x54444410
```

> 设置UART3 为串口：

> Mtkopenwrt的串口驱动默认只初始化了uart1 uart2两个串口需要修改代码

> 1）添加代码补丁：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/155fd203-d2ee-471a-b66f-3cf6689aca01/071-uart2-enable.patch

> 放到target/linux/ramips/patches/目录下

```
#配置内核
make kernel_menuconfig
	Device Drivers  --->
		Character devices  --->
			Serial drivers  --->
				(3) Maximum number of 8250/16550 serial ports   #把串口改为3个
```

> 2）源码同uart1 lite的修改

> 通过命令设置：

```
reg s 0
reg r 60 #读取GPIO1_MODE寄存器的值，读取到GPIO1_MODE的寄存器值为0x55444410，把27:26 bit设置为0

reg w 60 0x51444410
```

### `mtkopenwrt自带命令`

> mii_mgr:用来设置查询phy寄存器

> 获取port的端口状态

> 可以查看https://blog.csdn.net/subfate/article/details/44958597介绍的PHY寄存器

> port4的phy 编号为4 读取寄存器1

> `mii_mgr -g -p 4 -r 1`

> 其中 bit2 表示端口链接状态Link Status，bit5表示Auto-Negotiation Complete 自动协商是否完成

> 一般插拔网线 只是这两个bit在改变

> 它与reg命令获取ESW Base address+0x80 既读取POA寄存器的29bit是相同的

> `reg s 0;reg r 110080;`

> 也可以通过`switch phy 4`读取到 端口4的所有寄存器的值，其中01：7849和`mii_mgr -g -p 4 -r 1``读取到的值一致`

### `硬件限制及注意事项`

1.  硬件SPI 不支持全双工模式
