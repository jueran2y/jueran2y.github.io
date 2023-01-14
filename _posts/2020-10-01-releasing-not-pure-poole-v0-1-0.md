---
layout: post
title: OpenWRT18.06
author: Songzi Vong
tags:
- jekyll theme
- jekyll
date: 2020-10-01 13:56 +0800
---
## Openwrt18.06

* TOC
{:toc}
## Openwrt18.06

## Openwrt18 是 GIT社区上的开源版本

Openwrt18 是 GIT社区上的开源版本

`源码下载`

`固件编译配置`

修改默认网络参数

修改默认管理IP

VLAN配置介绍

修改适配内存：

修改适配文件系统大小

配置GPIO/功能脚

恢复出厂

添加按键功能

WIFI使用

防火墙配置

支持WAN口访问ssh

端口转发

DMZ

允许Ping WAN口IP

允许通过WAN口访问web

支持硬件看门狗的一种方法

串口使用介绍

UART0

UART1：

UART2：

UCI语法介绍

`匿名配置添加方法：`

`有名配置添加方法：`

调试串口添加登陆密码

添加4G拨号功能

添加拨号代码quectel-cm

添加内核补丁

添加编译选项

SPI使用

配合uboot 实现固件备份功能

华邦系列Flash芯片读取FlashId

Flash 的3/4byte问题

编译时可能会遇到的一些问题

开发

hello world示例

mem命令

读配置寄存器：

设置寄存器：

WIFI定频测试

CE自适应测试

### `源码下载`

> 使用的编译环境是 Ubuntu16.04 64位

> 目前有一些客户再使用这个版本，以 github社区的 openwrt18.06.5为例：

> git下载：

> git clone --branch v18.06.5 https://github.com/openwrt/openwrt.git openwrt18.06

> （18.06.5 vocore2 有对这个版本做了适配，也可以通过vocore2的适配闭源的WiFi驱动，具体方法参见：中的介绍）

### `固件编译配置`

```
cd openwrt06
./scripts/feeds update -a;./scripts/feeds install -a
make menuconfig
#选择
#Target System(MediaTek Ralink MIPS)
#Subtarget (MT76x8 based boards)
#Target Profile (MediaTek MT7628 EVB)
```

```
#编译前 需要修改target/linux/ramips/image/mt76xmk
#第87行：
/*
	IMAGE_SIZE := $(ralink_default_fw_size_4M) -> IMAGE_SIZE := $(ralink_default_fw_size_16M)
*/

#编译 
make V=99 -j4 
#编译完成后 生成固件位置在bin/target
cd bin/targets/ramips/mt76x8/

#固件文件名：openwrt-ramips-mt76x8-mt7628-squashfs-sysupgrade.bin
```

### 修改默认网络参数

### 修改默认管理IP

> 默认参数生成文件package/base-files/files/bin/config_generate

> 修改103行，默认管理IP 192.168.1.1

> 默认子网掩码是255.255.255.0 根据自己的需要修改

### VLAN配置介绍

```
#关于switch vlan的配置
在/etc/config/network 启动后默认的配置如下：
config switch
        option name 'switch0'
        option reset '1'
        option enable_vlan '1'

config switch_vlan
        option device 'switch0'
        option vlan '1'
        option ports '0 1 2 3 6t'

config switch_vlan
        option device 'switch0'
        option vlan '2'
        option ports '4 6t'
#默认配置了两个VLAN vlan1 vlan2
vlan1中 加入了5个端口0，1，2，3，6 其中6属于trunk口 （6t），可以接收多个vlan的包

vlan2中 加入了2个端口4，6 其中6属于trunk口（6t）
vlan1 在系统中属于LAN口  VLAN2 属于WAN口 

在模块上划分多个互不想通的网段
可以通过设置VLAN 既划分vlan端口的形式将端口进行vlan隔离同时
比如：
config switch_vlan
        option device 'switch0'
        option vlan '1'
        option ports '0 1 6t'

config switch_vlan
        option device 'switch0'
        option vlan '3'
        option ports '2 3 6t'

config switch_vlan
        option device 'switch0'
        option vlan '2'
        option ports '4 6t'

vlan1 vlan2 对应ifconfig中的eth1 eth2 此时加了一个vlan3 也需要生成一个eth3 对应vlan3的逻辑网口
方法：
在/etc/config/network中 添加lan1配置
config interface 'lan1'
        option type 'bridge'
        option ifname 'eth3'
        option proto 'static'
        option ipaddr '11254'
        option netmask '2220'
        option ip6assign '60'
在/etc/config/firewall中添加lan1的防火墙配置
config zone
        option input 'ACCEPT'
        option forward 'REJECT'
        option output 'ACCEPT'
        option name 'lan1'
        option network 'lan1'

config forwarding
        option dest 'wan'
        option src 'lan1'

修改/etc/config/dhcp 添加lan1的dhcp server：
config dhcp 'lan1'
        option start '100'
        option leasetime '12h'
        option limit '150'
        option interface 'lan1'
```

> 其在页面上的显示：

### 修改适配内存：

> MT7628.dts中的 默认内存配置是0x2000000 32M内存 7688A 内存规格是128M 可以把内存配置改为 0x8000000

> 修改完成后 内存显示大小

### 修改适配文件系统大小

> 7628N/7688A 使用的W25Q256JVEQ，是32M的Flash 可以修改 为0x1fb0000 设置固件的 分区为7688A/7628N的最大允许值

> 其他比如使用16MFlash的7628NS/7688AS 可以配置为0xfb0000

```
partition@50000 {
    label = "firmware";
    reg = <0x50000 0x1fb0000>;
};
```

> 添加独立的Flash配置分区

### 配置GPIO/功能脚

> openwrt18配置功能脚作为GPIO的方法：

> 比如如果使用 EPHY0~EPHY4 作为GPIO

> 修改对应的DTS配置文件

> 默认MT7628.dts中

```
&pinctrl {
    state_default: pinctrl0 {
        gpio {
            ralink,group = "i2c", "p0led_an", "p1led_an", "p2led_an", "p3led_an", "p4led_an";    //配置P0到P4的LED灯为GPIO模式
            ralink,function = "gpio";
        };
    };
};
```

> 导出GPIO目录到 /sys/class/gpio/目录下，通知设置GPIO为输出模式

```
gpio-export {
        compatible = "gpio-export";
        #size-cells = <0>;

        gpio39 {
            gpio-export,name = "gpio39";
            gpio-export,direction_may_change = <1>;
            gpio-export,output = <1>;
            gpios = <&gpio1 7 GPIO_ACTIVE_LOW>;
        };
        gpio40 {
            gpio-export,name = "gpio40";
            gpio-export,direction_may_change = <1>;
            gpio-export,output = <1>;
            gpios = <&gpio1 8 GPIO_ACTIVE_LOW>;
        };
    };
```

> pin脚在固件中分组名称 可以通过 /sys/kernel/debug/pinctrl/pinctrl/pinmux-functions文件查看

```
function: sdxc d6, groups = [ pwm1 ]
function: utif, groups = [ pwm1 ]
function: gpio, groups = [ pwm1 ]
function: pwm1, groups = [ pwm1 ]
function: sdxc d7, groups = [ pwm0 ]
function: utif, groups = [ pwm0 ]
function: gpio, groups = [ pwm0 ]
function: pwm0, groups = [ pwm0 ]
function: sdxc d5 d4, groups = [ uart2 ]
function: pwm, groups = [ uart2 ]
function: gpio, groups = [ uart2 ]
function: uart2, groups = [ uart2 ]
function: sw_r, groups = [ uart1 ]
function: pwm, groups = [ uart1 ]
function: gpio, groups = [ uart1 ]
function: uart1, groups = [ uart1 ]
function: -, groups = [ i2c ]
function: debug, groups = [ i2c ]
function: gpio, groups = [ i2c ]
function: i2c, groups = [ i2c ]
function: refclk, groups = [ refclk ]
function: perst, groups = [ perst ]
function: wdt, groups = [ wdt ]
function: spi, groups = [ spi ]
function: jtag, groups = [ sdmode ]
function: utif, groups = [ sdmode ]
function: gpio, groups = [ sdmode ]
function: sdxc, groups = [ sdmode ]
function: -, groups = [ uart0 ]
function: -, groups = [ uart0 ]
function: gpio, groups = [ uart0 ]
function: uart0, groups = [ uart0 ]
function: antenna, groups = [ i2s ]
function: pcm, groups = [ i2s ]
function: gpio, groups = [ i2s ]
function: i2s, groups = [ i2s ]
function: -, groups = [ spi cs1 ]
function: refclk, groups = [ spi cs1 ]
function: gpio, groups = [ spi cs1 ]
function: spi cs1, groups = [ spi cs1 ]
function: pwm_uart2, groups = [ spis ]
function: utif, groups = [ spis ]
function: gpio, groups = [ spis ]
function: spis, groups = [ spis ]
function: pcie, groups = [ gpio ]
function: refclk, groups = [ gpio ]
function: gpio, groups = [ gpio ]
function: gpio, groups = [ gpio ]
function: rsvd, groups = [ wled_an ]
function: rsvd, groups = [ wled_an ]
function: gpio, groups = [ wled_an ]
function: wled_an, groups = [ wled_an ]
function: jtag, groups = [ p0led_an ]
function: rsvd, groups = [ p0led_an ]
function: gpio, groups = [ p0led_an ]
function: p0led_an, groups = [ p0led_an ]
function: jtag, groups = [ p1led_an ]
function: utif, groups = [ p1led_an ]
function: gpio, groups = [ p1led_an ]
function: p1led_an, groups = [ p1led_an ]
function: jtag, groups = [ p2led_an ]
function: utif, groups = [ p2led_an ]
function: gpio, groups = [ p2led_an ]
function: p2led_an, groups = [ p2led_an ]
function: jtag, groups = [ p3led_an ]
function: utif, groups = [ p3led_an ]
function: gpio, groups = [ p3led_an ]
function: p3led_an, groups = [ p3led_an ]
function: jtag, groups = [ p4led_an ]
function: utif, groups = [ p4led_an ]
function: gpio, groups = [ p4led_an ]
function: p4led_an, groups = [ p4led_an ]
function: rsvd, groups = [ wled_kn ]
function: rsvd, groups = [ wled_kn ]
function: gpio, groups = [ wled_kn ]
function: wled_kn, groups = [ wled_kn ]
function: jtag, groups = [ p0led_kn ]
function: rsvd, groups = [ p0led_kn ]
function: gpio, groups = [ p0led_kn ]
function: p0led_kn, groups = [ p0led_kn ]
function: jtag, groups = [ p1led_kn ]
function: utif, groups = [ p1led_kn ]
function: gpio, groups = [ p1led_kn ]
function: p1led_kn, groups = [ p1led_kn ]
function: jtag, groups = [ p2led_kn ]
function: utif, groups = [ p2led_kn ]
function: gpio, groups = [ p2led_kn ]
function: p2led_kn, groups = [ p2led_kn ]
function: jtag, groups = [ p3led_kn ]
function: utif, groups = [ p3led_kn ]
function: gpio, groups = [ p3led_kn ]
function: p3led_kn, groups = [ p3led_kn ]
function: jtag, groups = [ p4led_kn ]
function: utif, groups = [ p4led_kn ]
function: gpio, groups = [ p4led_kn ]
function: p4led_kn, groups = [ p4led_kn ]
```

### 恢复出厂

```
umount /dev/mtdblock6;firstboot -y #重启生效
```

### 添加按键功能

> 修改DTS 添加按键功能

```
#设置GPIO 38 作为reset按键 
#这里 reset linux的kmod-gpio-button-hotplug 驱动中定义了几个功能
#在/etc/rc.button/中 可以看到 
#其中reset 短按1秒以内 是reset 长按5秒以上是恢复出厂

keys {
        compatible = "gpio-keys-polled";
        poll-interval = <20>;

        reset {
            label = "reset";
            gpios = <&gpio1 6 GPIO_ACTIVE_LOW>;
            linux,code = <KEY_RESTART>;
        };
    };
#gpio-button-hotplug 内核模块 定义的按键功能名称映射
#可以在DTS中配置相应的key  并在/etc/rc.button/创建相应的文件实现需要的按键功能
static struct bh_map button_map[] = {
    BH_MAP(BTN_0,           "BTN_0"),
    BH_MAP(BTN_1,           "BTN_1"),
    BH_MAP(BTN_2,           "BTN_2"),
    BH_MAP(BTN_3,           "BTN_3"),
    BH_MAP(BTN_4,           "BTN_4"),
    BH_MAP(BTN_5,           "BTN_5"),
    BH_MAP(BTN_6,           "BTN_6"),
    BH_MAP(BTN_7,           "BTN_7"),
    BH_MAP(BTN_8,           "BTN_8"),
    BH_MAP(BTN_9,           "BTN_9"),
    BH_MAP(KEY_BRIGHTNESS_ZERO, "brightness_zero"),
    BH_MAP(KEY_CONFIG,      "config"),
    BH_MAP(KEY_COPY,        "copy"),
    BH_MAP(KEY_EJECTCD,     "eject"),
    BH_MAP(KEY_HELP,        "help"),
    BH_MAP(KEY_LIGHTS_TOGGLE,   "lights_toggle"),
    BH_MAP(KEY_PHONE,       "phone"),
    BH_MAP(KEY_POWER,       "power"),
    BH_MAP(KEY_RESTART,     "reset"),
    BH_MAP(KEY_RFKILL,      "rfkill"),
    BH_MAP(KEY_VIDEO,       "video"),
    BH_MAP(KEY_WIMAX,       "wwan"),
    BH_MAP(KEY_WLAN,        "wlan"),
    BH_MAP(KEY_WPS_BUTTON,      "wps"),
};
```

### WIFI使用

> hostapd

> 开源的WIFI驱动 AP功能是通过hostapd 做配置下发 认证管理功能

> 在页面上使能AP

```
#运行启动hostapd 
/usr/sbin/hostapd -s -P /var/run/wifi-phypid -B /var/run/hostapd-phyconf
#配置文件：
driver=nl80211
logger_syslog=127
logger_syslog_level=2
logger_stdout=127
logger_stdout_level=2
hw_mode=g
beacon_int=100
channel=11
ieee80211n=1
ht_coex=0
ht_capab=[SHORT-GI-20][SHORT-GI-40][TX-STBC][RX-STBC1]
interface=wlan0
ctrl_interface=/var/run/hostapd
ap_isolate=1
bss_load_update_period=60
chan_util_avg_period=600
disassoc_low_ack=1
preamble=1
wmm_enabled=1
ignore_broadcast_ssid=0
uapsd_advertisement_enabled=1
auth_algs=1
wpa=0
ssid=OpenWrt
bridge=br-lan
bssid=20:05:05:00:03:fc
```

> `wpa_supplicant`

> wpad

```
#iw命令的一些用法
root@Openwrt:~# iw dev wlan0 info   #显示AP的信息
Interface wlan0
        ifindex 8
        wdev 0x2
        addr 40:d6:3c:63:ab:1c
        ssid OpenWrt
        type AP
        wiphy 0
        channel 11 (2462 MHz), width: 20 MHz, center1: 2462 MHz
        txpower 00 dBm


root@Openwrt:~# iw dev wlan0 station dump   #显示连接的STA的信息
Station 48:2c:a0:63:12:4e (on wlan0)
        inactive time:  150 ms
        rx bytes:       45111
        rx packets:     803
        tx bytes:       9689
        tx packets:     104
        tx retries:     61
        tx failed:      1
        rx drop misc:   0
        signal:         -49 [-55, -49] dBm
        signal avg:     -50 [-54, -50] dBm
        tx bitrate:     8 MBit/s MCS 5 short GI
        rx bitrate:     0 MBit/s
        expected throughput:    21Mbps
        authorized:     yes
        authenticated:  yes
        associated:     yes
        preamble:       short
        WMM/WME:        yes
        MFP:            no
        TDLS peer:      no
        DTIM period:    2
        beacon interval:100
        CTS protection: yes
        short preamble: yes
        short slot time:yes
        connected time: 43 seconds
iw dev   #查看WIFI设备信息
phy#0
        Interface wlan0
                ifindex 8
                wdev 0x2
                addr 40:d6:3c:63:ab:1c
                ssid OpenWrt
                type AP
                channel 11 (2462 MHz), width: 20 MHz, center1: 2462 MHz
                txpower 00 dBm
iw phy phy0 interface add mon0 type monitor # 创建监听接口
iw dev mon0 set freq 2437 #指定捕获频率
iw dev mon0 del #删除监听接口
```

### 防火墙配置

### 支持WAN口访问ssh

```
config rule
        option dest_port '22'
        option src 'wan'
        option name 'Allow-SSH'
        option target 'ACCEPT'
        list proto 'tcp'
```

### 端口转发

> 来自internet的使用tcp协议访问路由80端口的请求映射到内网192.168.1.10的 80端口

```
config redirect
        option src wan
        option src_dport 80
        option proto tcp
        option dest_ip 1110
```

### DMZ

> 例如把局域网的192.168.1.2的服务器映射到公网中。需要增加option target DNAT这一项，才能在openwrt中实现DMZ的功能。

> 修改/etc/config/firewall

```
config redirect
        option src              wan
        option proto            all
        option dest_ip          112
        option target           DNA
```

### 允许Ping WAN口IP

```
config rule
        option name 'Allow-Ping'
        option src 'wan'
        option proto 'icmp'
        option icmp_type 'echo-request'
        option family 'ipv4'
        option target 'ACCEPT'
```

### 允许通过WAN口访问web

```
config rule
        option name 'Allow-Web'
        option src 'wan'
        option proto 'tcp'
        option dest_port '80'
        option target 'ACCEPT'
```

### 支持硬件看门狗的一种方法

> 用I2S_CLK作为看门狗的喂狗pin脚

> I2S_CLK（GPIO3）默认作为I2S功能pin脚

> 使用xxx看门狗芯片 中说明 如果 喂狗pin脚处于高阻态，则看门狗会自动喂狗 ，

> I2S_CLK作为 I2S功能脚或配置为GPIO的输入脚 都是高阻态，这样就可以确保，固件启动后 再设置为

> GPIO输出，控制喂狗

> 修改DTS配置 把 I2S配置为GPIO功能脚

> 编译启动固件， 固件默认并没有导出到/sys/class

> 可以参考下面脚本的写法控制喂狗

```
cd /sys/class/gpio; echo 3 > export ;cd gpio3 ;echo out > direction;a=true;while $a;do echo 1 > value;sleep 1;echo 0 > value;sleep 1;done
```

### 串口使用介绍

> 7688/7628 共有3个UART串口

### UART0

> 默认作为调试串口使用

> 如果需要使用串口 需要屏蔽掉串口的调试打印

> 修改DTS：

```
#把console配置成一个不存在的串口 既可
chosen {
        bootargs = "console=ttyS3,57600";
};
```

### UART1：

> 配置DTS开启UART1

```
#修改target/linux/ramips/dts/MT76dts
#添加：
&uart1 {
    status = "okay";
};
```

### UART2：

> 修改DTS开启UART2

> openwrt18 默认无法直接打开UART2 还需要同时开启SD

```
make menuconfig->Kernel modules  --->Other modules  --->
<*> kmod-sdhci
<*> kmod-sdhci-mt7620
#同时修改DTS文件添加：
&uart2 {
    status = "okay";
};

&sdhci {
    status = "okay";
    mediatek,cd-high;
};
```

> 另外也可以通过mem命令打开，但需要在系统启动后

### UCI语法介绍

> https://openwrt.org/docs/guide-user/base-system/uci openwrt社区对于UCI命令的帮助文档

> UCI配置分为两种 有名和匿名

### `匿名配置添加方法：`

> 比如wifi接口的配置就属于匿名配置：

> uci add wireless wifi-iface

> uci set wireless.@wifi-iface[-1].device=mt7628

> uci set wireless.@wifi-iface[-1].ifname=ra1

> uci set wireless.@wifi-iface[-1].network=lan1

> uci set wireless.@wifi-iface[-1].mode=ap

> uci set wireless.@wifi-iface[-1].ssid=HLK-A905

> uci set wireless.@wifi-iface[-1].encryption=psk2

> uci set wireless.@wifi-iface[-1].key=12345678

### `有名配置添加方法：`

> 比如network中的网络接口配置就属于有名配置：

> uci set network.lan1=interface

> uci set network.lan1.ifname=eth0.1

> uci set network.lan1.force_link=1

> uci set network.lan1.type=bridge

> uci set network.lan1.proto=static

> uci set network.lan1.ipaddr=192.168.17.254

> uci set network.lan1.netmask=255.255.255.0

### 调试串口添加登陆密码

> 为了安全性考虑，进行串口登入的时候也希望像ssh那样要求输入用户名和密码才能进入控制台。

```
#修改target/linux/ramips/base-files/etc/inittab
#改为：
::sysinit:/etc/init.d/rcS S boot
::shutdown:/etc/init.d/rcS K shutdown
::askconsole:/bin/login

make menuconfig

Base system --->
  <*> busybox ......
         [*] Customize busybox option
               Login/Password Management Utilities --->
                     [*] login (NEW)
```

### 添加4G拨号功能

> 我们目前只使用过EC20 和EC200 其他模块需要自行添加

### 添加拨号代码quectel-cm

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/374967d3-8b73-4572-b527-7f6db5d3958b/quectel-cm.tar.gz

```
#把quectel-cm.tar.gz 解包后 放到package/目录下
luke@ub:~/openwrt06/package$ ls
base-files  boot  devel  feeds  fibocom_fm150  firmware  kernel  libs  Makefile  network  quectel-cm  system  utils
```

> 添加移远内核模块

> 把quectel_usb.tar.gz 解压放到package/kernel目录下

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7ff34ca3-462e-4674-ab60-aa6f2a2d15bb/quectel_usb.tar.gz

### 添加内核补丁

```
#把补丁999-4G.patch 放到target/linux/ramips/patches-14/目录下
```

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/db111de5-2ced-4a92-847f-d9824c0effce/999-4G.patch

### 添加编译选项

```
make menuconfig->HLK Custom  ---><*> quectel.......................................................
Kernel modules  --->USB Support  ---><*> kmod-usb-net...............
																		 <*>   kmod-usb-net-cdc-eem.....................
                                     <*>   kmod-usb-net-cdc-mbim.....................
                                     <*>   kmod-usb-net-rndis.........................
                                     <*> kmod-usb-serial.....................
                                     <*>   kmod-usb-serial-option...................
                                     <*>   kmod-usb-serial-qualcomm.................
```

> 编译到固件里 启动时使用quectel-cm命令进行拨号

> 内网穿透

> 目前有几种开源软件可以使用N2N 和FRP， 都必须有公网的服务器

> N2N的使用方法

> FRP的使用方法

### SPI使用

> 需要后续补充

> 另一路SPI使用介绍

> 7688/7628 硬件上有两路SPI，且只支持半双工模式，只能接Flash SPI屏这种 半双工设备

> 其中一路片选已经接了 SPI Flash

> 可通过另外一路SPI 片选扩展 Flash 也可以外接SPI屏 等半双工设备

### 配合uboot 实现固件备份功能

> 实现思路

> 以7688A/7628N 使用32M Flash的模块举例：

> 限制固件的firmware分区大小 这里假设限制为12M （最大不能超过15M kernel+rootfs的固定占用+jffs2分区的最小也要有1M）

> 新增一个备份的Flash分区backup_f大小也为12M

> 修改sysupgrade脚本 添加-z选项 可以把升级固件升级到备份分区

> 在uboot env参数中添加firmware_startok字段

1.  Uboot中添加

> 启动流程 启动时 判断firmware_startok是否为1

> 如果是1 将boot env的 firmware_startok标记为 0 待固件启动成功后 把firmware_startok设置为1

> 如果是0 判断固件启动出现问题，校验firmware_backup分区

> 确认需求的几个点

1.  Flash划分：

> Flash分区的划分

1.  升级过程：

> 触发主分区备份到备份分区 sysupgrade添加命令支持 （可以支持整体备份），

> 单独备份overlay分区，置标志位 表示备份分区需要拷贝到主分区

> 升级固件成功后 清除标志位

1.  Uboot恢复过程：

> 判断标志位如果置位，则把备份分区拷贝到主分区中

> 这个过程需要两个工具

> fw_setenv (用来写入修改 uboot环境变量的工具)

> sysupgrade新增一个接口 可以保存主分区以及配置文件到备份分区中

> 问题：

> 这是升级加备份的过程 正常情况下 升级是没有问题的

> 这里没有考虑到 如果主分区无法启动 此时备份分区标志位也没有置位 这时候怎么恢复启动（新升级固件有bug 或者Flash出现跳变导致的无法启动）

### 华邦系列Flash芯片读取FlashId

> 海凌科7688/7628模块使用的华邦系列的Flash芯片带有UniqueId

> 如果需要通过硬件的唯一性 确保软件不被拷贝复制，可以通过使用FlashId进行加解密来实现

> 读取FlashId的内核补丁：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/55a3e92c-d8a2-495c-9f79-30e43f3a466b/999-flashid.patch

### Flash 的3/4byte问题

> SPI Flash 的地址模式有两种 3byte/4byte

> 3byte模式下 寻址空间只有16M

> 4byte模式下 寻址空间大于16M

> 海凌科的 7688/7628 模块 默认是3byte启动模式 所以出厂的Flash也是3byte模式

> 3byte启动模式 在一些固件中可能会存在一个问题

> reboot 或 拉低CPU PORST 无法启动

> 解决方法：修改7688的启动模式为4byte

> 需要修改SPI CS1的启动电平为高电平（拉高3.3V）

> 需要把Flash通过配置寄存器的方式设置为4byte模式

> 如何确认是否是3/4 Byte造成的无法启动：

> reboot 重启不了 不要断电，

> 拉高SPI_CS1 pin脚 再按CPU复位按键 查看是否可以重启

### 编译时可能会遇到的一些问题

1.  WARNING: Makefile 'package/kernel/utils/busybox/Makefile' has a dependency on 'libpam', which does not exist

> 可以使用命令 重新索引并强制安装可能已移动的包。如果在包移入和移出核心包的分支之间切换，就可能会发生这种情况

> ```
> ./scripts/feeds update -i -f;./scripts/feeds install -a -f

> VPN的使用

> 我们这里选择了L2TP PPTP openvpn GRE IPSec N2N 在openwrt18.06中的配置方法，其他版本的openwrt类似。

### 开发

#### hello world示例

在package目录下创建hello_world目录

创建Makefile文件，文件内容：

```
include $(TOPDIR)/rules.mk

#包名，版本和发布版本号
PKG_NAME:=hello_world
PKG_RELEASE:=1

#源码路径
#定制变量，下面会使用
SOURCE_DIR:=/home/luke/helloworld

include $(INCLUDE_DIR)/package.mk

PKG_BUILD_DIR:=$(BUILD_DIR)/hello_world

#包定义，说明包将如何出现在整体配置菜单中的位置
define Package/hello_world
    SECTION:= HLK Custom
    CATEGORY:=HLK Custom
    TITLE:= HiLink Ser2Net
    DEPENDS:=+libpcre
endef

#包描述；更详细地描述包的用途
define Package/hello_world/description
    HLK Custom Application
endef

#包准备；创建构建目录并复制源代码
#最后一个命令是必要的，以确保我们的准备说明与补丁系统保持兼容
define Build/Prepare
    mkdir -p $(PKG_BUILD_DIR)
    cp -rfd ./src/* $(PKG_BUILD_DIR)/
		$(Build/Patch)
endef


#包安装说明；在包内创建一个目录来保存我们的可执行文件，然后将我们之前构建的可执行文件复制到该文件夹中
define Package/hello_world/install
    $(INSTALL_DIR) $(1)/usr/bin
    $(INSTALL_DIR) $(1)/etc
    $(INSTALL_BIN) $(PKG_BUILD_DIR)/hello_world $(1)/usr/bin/
endef

# 这个命令总是最后一个，它使用我们上面给出的定义和变量来完成工作
$(eval $(call BuildPackage,hello_world))
```

> 创建src目录，在src目录下创建Makefile文件

```
#创建Makefile文件
#内容如下：
###############################################################################
#
# A smart Makefile template for GNU/LINUX programming
#
# Date:   2012/10/24
#
# Usage:
#   $ make           Compile and link (or archive)
#   $ make clean     Clean the objectives and target.
###############################################################################

OPTIMIZE := #-O2
WARNINGS := #-Wall -Wno-unused -Wno-format -Wno-strict-aliasing
DEFS     :=
EXTRA_CFLAGS := #-DDEBUG #-m32

#定义一些变量，编译链接时使用
INC_DIR   = ./include           #include目录
SRC_DIR   = .   #源代码目录，表示编译时去当前SRC_DIR目录下查找源文件

OBJ_DIR   = ./out	#目标文件目录， 表示编译后生成的.o文件存放目录， 用以链接使用
EXTRA_SRC =     #一些额外增加或者文件比较分散的源文件
EXCLUDE_FILES =	#排除的文件列表

SUFFIX       = c 	#源文件后缀，表示源代码可以支持的源文件后缀
TARGET       := hello_world #生成的目标名称
#TARGET_TYPE  := ar		#目标类型 表示是静态链接目标文件
#TARGET_TYPE  := so		#表示是动态链接目标文件
TARGET_TYPE  := app		#表示是生成应用程序


LDFLAGS += -lpcre    #需要链接的库

#####################################################################################
#  Do not change any part of them unless you have understood this script very well  #
#  This is a kind remind.                                                           #
#####################################################################################

#FUNC#  Add a new line to the input stream.
#定义一个变量
define add_newline
$1

endef

#Makefile 函数
#反过滤函数——filter-out。
#$(filter-out <pattern...>,<text> )

# foreach函数
#foreach 函数和别的函数非常的不一样。因为这个函数是用来做循环用的，Makefile 中的 
#foreach 函数几乎是仿照于 Unix 标准 Shell（/bin/sh）中的 for 语句，或是 C-Shell 
#（/bin/csh）中的foreach语句而构建的 语法：
#$(foreach <var>,<list>,<text> )
##$(foreach <var>,<list>,<test>) 把函数<list>中的单词逐一取出放到参数<var>所指定的变量中，然后再执行<text>所包含的表达式。每一次<text>会返回一个字符串，循环过程中，<text>的所#
#返回的每个字符串会以空格分割，最后当整个循环结束时，<text>所返回的每个字符串所组成的整个字符串将会是foreach函数的返回值
#举例：$(foreach d,$(sort $(INC_DIR) $(SRC_DIR)),-I$d)
# INC_DIR=../include 
# SRC_DIR=./src 

# wildcard函数
#语法： $(wildcard PATTERN)
#列出当前目录下所有符合模式“PATTERN”格式的文件名
#举例：$(wildcard *.c) 返回值为当前目录下所有.c源文件列表

# 模式字符串替换函数——patsubst
#功能：查找<text>中的单词（单词以“空格”、“Tab”或“回车”“换行”分隔）是否 
#符合模式<pattern>，如果匹配的话，则以<replacement>替换。这里，<pattern>可以包##括通配符“%”，表示任意长度的字串。如果<replacement>中也包含“%”，那么， 
##<replacement>中的这个“%”将是<pattern>中的那个“%”所代表的字串。（可以######用“\\” 来转义，以“\\%”来表示真实含义的“%”字符） 
#返回：函数返回被替换过后的字符串。
#$(patsubst %.c,%.o, x.c.c bar.c)
#把字串“x.c.cbar.c”符合模式[%.c]的单词替换成[%.o]，返回结果是“x.c.obar.o”

#使用：SRC = $(notdir wildcard)
#去除所有的目录信息，SRC里的文件名列表将只有文件名。

#6 eval函数详解http://bbs.chinaunix.net/thread-2321462-3-html
#FUNC# set the variable `src-x' according to the input $1
define set_src_x
src-$1 = $(filter-out $4,$(foreach d,$2,$(wildcard $d/*.$1)) $(filter %.$1,$3))
endef 
#这里定义一个变量，后面会用到(会在这里用到$(eval $(foreach i,$(SUFFIX),$(call set_src_x,$i,$(SRC_DIR),$(EXTRA_SRC),$(EXCLUDE_FILES)))) )
#src-$1的意思是 在call调用之后， 这里会变成：
#src-$i = $(filter-out $(EXCLUDE_FILES), $(foreach d, $(SRC_DIR), $(wildcard $d/*.$i)) $(filter #%.$1, $(EXTRA_SRC) )
#这里$i 是SUFFIX中的c cpp cxx $(prefix_objdir) = out, $(SRC_DIR)=src, 
#这句话意思是遍历src_dir 找到src_dir里所有的.c文件和EXTRA_SRC里的所有文件，通过排除掉#EXCLUDE_FILES文件夹里的文件。

#FUNC# set the variable `obj-x' according to the input $1
define set_obj_x	
obj-$1 = $(patsubst %.$1,$3%.o,$(notdir $2))
endef
#同样的后面也会用到,
#这里在call之后会变成 obj-c = $(patsubst %.c,$(prefix_objdir)%.o, $(notdir $(src-c)))
#这里意思就是obj-c = 把.c名字替换成.o名字，且这个.o有$(prefix_objdir)前缀 即out/*.o这种形式#（同时排除src-c里面的目录文件）

#VAR# Get the uniform representation of the object directory path name
ifneq ($(OBJ_DIR),)	#如果OBJ_DIR不为空，
prefix_objdir := $(filter-out /,$(prefix_objdir)/)	
endif
#prefix_objdir  = $(shell echo $(OBJ_DIR)|sed 's:\\(\./*\\)*::')	#prefix_objdir = (后面这个函数执行#后， 这里OBJ_DIR=out, 这里是把./过滤掉， out没有./ prefix_objdir = out)
##/反过滤函数， 过滤掉匹配的模式，这里是吧匹配/的过滤掉，即 /不能是out的输出

GCC      := $(CROSS_COMPILE)gcc
G++      := $(CROSS_COMPILE)gcc
SRC_DIR := $(sort . $(SRC_DIR))
inc_dir = $(foreach d,$(sort $(INC_DIR) $(SRC_DIR)),-I$d)
#include的目录， 执行完成makefile的函数后， inc_dir = -lsrc/

#--# Do smart deduction automatically
$(eval $(foreach i,$(SUFFIX),$(call set_src_x,$i,$(SRC_DIR),$(EXTRA_SRC),$(EXCLUDE_FILES)))) 
#这个makefile函数 call set_src_x 
$(eval $(foreach i,$(SUFFIX),$(call set_obj_x,$i,$(src-$i),$(prefix_objdir))))
#这个makefile函数 调用 set_obj_x ，这个意思就是 把src-c里的.c名字变为out/*.o名字
$(eval $(foreach f,$(EXTRA_SRC),$(call add_newline,vpath $(notdir $f) $(dir $f))))
#这个makefile函数 调用add_newline 只有$1，那么这句话的意思是设置vpath变量
#执行完成后vpath (exter_src)文件 目录 vpath的意思是到指定目录下去找extra_src文件。
$(eval $(foreach d,$(SRC_DIR),$(foreach i,$(SUFFIX),$(call add_newline,vpath %.$i $d))))

all_objs = $(foreach i,$(SUFFIX),$(obj-$i))
#这里 把obj-c obj-cc obj-cxx obj-cxx定义的.o文件汇集
all_srcs = $(foreach i,$(SUFFIX),$(src-$i))
#同理all_objs

CFLAGS       += $(EXTRA_CFLAGS) $(WARNINGS) $(OPTIMIZE) $(DEFS)
TARGET_TYPE := $(strip $(TARGET_TYPE))

ifeq ($(filter $(TARGET_TYPE),so ar app),)
	$(error Unexpected TARGET_TYPE `$(TARGET_TYPE)`)
endif

#定义伪目标
#如果编写一个规则，并不产生目标文件，则其命令在每次make 该目标时都执行
#
PHONY = all romfs .mkdir clean

all: .mkdir $(TARGET)

define cmd_o
$$(obj-$1): $2%.o: %.$1  $(MAKEFILE_LIST)
	$(GCC) $(inc_dir) -Wp,-MT,$$@ -Wp,-MMD,$$@.d $(CFLAGS) -c -o $$@ $$<
endef
$(eval $(foreach i,$(SUFFIX),$(call cmd_o,$i,$(prefix_objdir))))
#这个函数是执行cmd_o , 
#执行完之后cmd_o变成：
# $(obj-c):out/%.o: %.c %(MAKEFILE_LIST
#	$(GCC) $(inc_dir) -Wp, -MT, $$@ -Wp,-MMD,$$@.d $(CFLGAS) -c -o $$@ $$<
#这个是典型makefile的静态模式规则。
#这里$(obj-c)里的.o 和$2%.o里的.o都依赖各自对应的.c
#会出现相应$(SUFFIX)中的obj-c obj-cxx obj-cpp
#这句意思就是(obj-c)里的.o和$(prefix_objdir)%.o都依赖于各自对应的.c
#它其实和%.o:%.c类似，只不过是指定了具体哪个目录下的.c对应与哪个目录下的.o
#动作就是 执行gcc 编译成对应的.o文件
#$@--目标文件，$^--所有的依赖文件，$<--第一个依赖文件。
#这个目标，执行次数就是.c文件的个数， $@生成目标， $$< 目标依赖的第一个文件。

#判断目标类型，根据目标类型决定链接链接参数

ifeq ($(TARGET_TYPE),so)
CFLAGS  += -fpic -shared
LDFLAGS += -shared
endif

ifeq ($(TARGET_TYPE),ar)
$(TARGET): AR := $(CROSS_COMPILE)ar
$(TARGET): $(all_objs)
	rm -f $@
	$(AR) rcvs $@ $(all_objs)
else
$(TARGET): LD = $(if $(strip $(src-cpp) $(src-cc) $(src-cxx)),$(G++),$(GCC))
$(TARGET): $(all_objs)
	$(GCC) $(all_objs) $(LDFLAGS) -o $@
endif

.mkdir:
	@if [ ! -d $(OBJ_DIR) ]; then mkdir -p $(OBJ_DIR); fi
clean:
	rm -f $(prefix_objdir)*.o $(prefix_objdir)*.d $(TARGET)
	rm -rf out/
install:
	cp *.so /lib
#这里 include 所有all_objs里所有匹配%.o.d文件，意思是如果c文件包含的include文件更新了那么make就会更新.o文件，重新编译链接。但第一次编译时并没有任何文件，因为-include是在命令执行之前执行的操作。
-include $(patsubst %.o,%.o.d,$(all_objs))

.PHONY: $(PHONY)
```

> 创建C文件

```
#include <stdio.h>

int main(int argc, char **argv)
{
    printf("helloworld!.\\r\\n");

    return 0;
}
```

> 执行make menuconfig 选中

```
make menuconfig->HLK Custom  --->*> hello_world
#编译：
make package/hello_world/compile V=99
```

### mem命令

> 除了配置DTS预先设置pin脚的功能 还可以通过直接写内存控制寄存器的方式 启动后设置pin脚功能

### 读配置寄存器：

```
#读取
mem 0x10000060
00000000: 44 44 15 50 55 05 00 00  00 00 00 00 00 00 00 00    DD.PU...........
00000010: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    ................
00000020: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    ................
00000030: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    ................
#具体定义请参考7688的寄存器手册
```

> MT7688_Datasheet_v1_4.pdf

### 设置寄存器：

> mem 0x10000064 0x555

> 控制ETH_PHY LED灯的一个示例：

```
#控制EPHY_LED3 EPHY_LED4
while [ 1 ]; do mem 0x10000634 0x180; sleep 1; mem 0x10000644 0x180; sleep 1; done
```

> mem的代码包：

https://s3-us-west-2.amazonaws.com/secure.notion-static.com/765fa02c-f9c0-4dcf-a074-3e1136ae52e1/mem.tar.gz

> 将mem.tar.gz 解压到package目录下

```
#勾选上mem
make menuconfig->HLK Custom  ---><*> mem
```

### WIFI定频测试

> 定频测试需要烧录定频测试固件 ，我们提供了一个固件供客户对WIFI进行定频 认证

> 固件名称：`first_7688.bin`

> 定频工具：MT7688_7628_7603_7636_QA tool.zip

> 连接模块的方法

1.  如果引出了P1~P4中的其中一个 (默认为LAN口) 可以把电脑连接到模块网口上，电脑IP设置为自动获取即可，待模块启动后 使用telnet 工具登录10.10.10.254

2.  如果只引出了P0口（默认作为WAN口），需要通过路由器给模块分配IP

    > 将模块的P0口连接到上级路由器，通过路由器给模块分配到一个IP，电脑也连接到路由器上

    > 在路由器上查询到模块获取的IP，记住这个IP。

> 有的windows系统自带telnet 可以通过CMD 命令行 输入telnet 查看是否存在telnet命令

> 如果系统没有自带，需要使用其他工具 比如 putty mobaxterm等终端工具

> 连接模块的telnet ：

> 出现系统提示符后 输入ated 回车确认

> 打开QA工具

> 并选择APSOC

> 点击OK，并选择电脑的网卡

> 打开QATool

### CE自适应测试

> 定频测试需要烧录定频测试固件 ，我们提供了一个固件供客户对WIFI进行定频 认证

> 固件名称：`first_7688.bin`

> 连接模块的方法

1.  如果引出了P1~P4中的其中一个 (默认为LAN口) 可以把电脑连接到模块网口上，电脑IP设置为自动获取即可，待模块启动后 使用telnet 工具登录10.10.10.254

2.  如果只引出了P0口（默认作为WAN口），需要通过路由器给模块分配IP

    > 将模块的P0口连接到上级路由器，通过路由器给模块分配到一个IP，电脑也连接到路由器上

    > 在路由器上查询到模块获取的IP，记住这个IP。

> 有的windows系统自带telnet 可以通过CMD 命令行 输入telnet 查看是否存在telnet命令

> 如果系统没有自带，需要使用其他工具 比如 putty mobaxterm等终端工具

> 连接模块的telnet ：

> 出现系统提示符后 输入ated 回车确认

> 设置区域国家码：nvram_set CountryCode EU 重启

> 测试信道1 b模

> iwpriv ra0 set Channel=1

> iwpriv ra0 set FixedTxMode=CCK iwpriv ra0 set WirelessMode=1 iwpriv ra0 set BasicRate=3

> 测试信道13 b模

> iwpriv ra0 set Channel=13

> iwpriv ra0 set FixedTxMode=CCK iwpriv ra0 set WirelessMode=1 iwpriv ra0 set BasicRate=3

> 测试g模

> iwpriv ra0 set FixedTxMode=OFDM iwpriv ra0 set WirelessMode=4 iwpriv ra0 set BasicRate=351

> 测试n模

> iwpriv ra0 set FixedTxMode=HT iwpriv ra0 set WirelessMode=9 iwpriv ra0 set BasicRate=15

> iwpriv ra0 set HtBw=1(打开N40) iwpriv ra0 set HtBw=0(打开N20)

> CE自适应标准：EN 300 328 v1.8.1

> 如果测试不过，可以尝试以下操作：

1.  iwpriv ra0 set ed_chk=1 开启edcca功能

2.  尝试调低edcca的阀值，让DUT 更容易听到干扰 可以先读一下 mac 60200618 iwpriv ra0 mac 60200618 iwpriv ra0 mac 60200618=0xd7e87910
