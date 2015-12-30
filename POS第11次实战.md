# POS 第11次实战

## 实战题
1. Bash脚本模拟插拔USB设备500次(1分)
    - 脚本要有一个输入参数，用于标识USB设备
2. 实现一个虚拟的USB存储设备(4分)
    - lsusb命令能够看到该设备
    - /dev下有设备描述符
    - 支持读写操作。能够对/dev下的设备描述符，进行读写操作


## 实战一

参考 <https://lwn.net/Articles/143397/>

运行情况：

```
ivan@ubuntu-Ihost:~/kernel/usbweek$ tree /sys/bus/usb/drivers/usb
/sys/bus/usb/drivers/usb
├── 1-1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1/1-1
├── bind
├── uevent
├── unbind
└── usb1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1

2 directories, 3 files
ivan@ubuntu-Ihost:~/kernel/usbweek$ sudo ./a.sh 1-1
The 1 round:
/sys/bus/usb/drivers/usb
├── bind
├── uevent
├── unbind
└── usb1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1

1 directory, 3 files
Device unbinded.
/sys/bus/usb/drivers/usb
├── 1-1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1/1-1
├── bind
├── uevent
├── unbind
└── usb1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1

2 directories, 3 files
Device binded.
The 2 round:
/sys/bus/usb/drivers/usb
├── bind
├── uevent
├── unbind
└── usb1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1

1 directory, 3 files
Device unbinded.
/sys/bus/usb/drivers/usb
├── 1-1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1/1-1
├── bind
├── uevent
├── unbind
└── usb1 -> ../../../../devices/pci0000:00/0000:00:06.0/usb1

2 directories, 3 files
Device binded.

```

附脚本代码：

```bash
#!/bin/bash
if [ $# -eq 1 ]; then
	for i in `seq 1 5`;
	do
		echo The $i round:

		echo -n "$1" > /sys/bus/usb/drivers/usb/unbind
		tree /sys/bus/usb/drivers/usb
		echo Device unbinded.

		echo -n "$1" > /sys/bus/usb/drivers/usb/bind
		tree /sys/bus/usb/drivers/usb
		echo Device binded.
	done
else
	echo Usage: $0 BusID
fi
```

## 实战二

> 上节课的实战改为如下内容:
>
> 1. 把 usb virtual host controller跑起来，总线下要挂一个设备，设备号为0xfcfc:0xdeba。
> 2. 该设备要匹配skeleton驱动，lsusb -t中这个设备driver为skeleton

### vhci_hcd

<http://sourceforge.net/p/usb-vhci/wiki/Home/>

```bash
git clone git://git.code.sf.net/p/usb-vhci/vhci_hcd
insmod usb-vhci-hcd.ko
insmod usb-vhci-iocifc.ko
chmod 666 /dev/usb-vhci
```

### 虚拟驱动 skeleton

`drivers/usb/usb-skeleton.c` 拷贝出来编译为内核模块。

```c
/* table of devices that work with this driver */
static const struct usb_device_id skel_table[] = {
        { USB_DEVICE(USB_SKEL_VENDOR_ID, USB_SKEL_PRODUCT_ID) },
        { USB_DEVICE(0xfcfc, 0xdeba)},      /* 增加这一行 */
        { }                                     /* Terminating entry */
};
MODULE_DEVICE_TABLE(usb, skel_table);
```

### 虚拟设备

下载 `libusb-vhci` 并 `make`.

进入`libusb_vhci-0.7/examples/`，修改`virtual_device2.c`开头部分的描述信息。

```c
const uint8_t dev_desc[] = {
	18,     // descriptor length
	1,      // type: device descriptor
	0x00,   // bcd usb release number
	0x02,   //  "
	0,      // device class: per interface
	0,      // device sub class
	0,      // device protocol
	64,     // max packet size
	0xfc,   // vendor id
	0xfc,   //  "
	0xba,   // product id
	0xde,   //  "
	0x38,   // bcd device release number
	0x11,   //  "
	0,      // manufacturer string
	1,      // product string
	0,      // serial number string
	1       // number of configurations
};

const uint8_t conf_desc[] = {
	9,      // descriptor length
	2,      // type: configuration descriptor
	32,     // total descriptor length (configuration+interface)
	0,      //  "
	1,      // number of interfaces
	1,      // configuration index
	0,      // configuration string
	0x80,   // attributes: none
	0,      // max power

	9,      // descriptor length
	4,      // type: interface
	0,      // interface number
	0,      // alternate setting
	2,      // number of endpoints
	0,      // interface class
	0,      // interface sub class
	0,      // interface protocol
	0,       // interface string

	7,
	5,
	0x01,
	2,
	0,
	0x10,
	0,

	7,
	5,
	0x82,
	2,
	0,
	0x10,
	0
};
```

### 运行

```bash
$ sudo insmod vhci_hcd/usb-vhci-hcd.ko
$ sudo insmod vhci_hcd/usb-vhci-iocifc.ko
$ sudo insmod skeleton.ko
$ (cd libusb_vhci-0.7/examples/; sudo ./virtual_device2)
created usb_vhci_hcd.0 (bus# 2)
got port stat work
status: 0x0100
change: 0x0000
flags:  0x00
port is powered on -> connecting device
got port stat work
status: 0x0101
change: 0x0001
flags:  0x00
CONNECTION state changed -> invalidating address
got port stat work
status: 0x0101
change: 0x0000
flags:  0x00
got port stat work
status: 0x0111
change: 0x0000
flags:  0x00
port is resetting
-> completing reset
got port stat work
status: 0x0103
change: 0x0010
flags:  0x00
RESET successfull -> use default address
got port stat work
status: 0x0103
change: 0x0000
flags:  0x00
got process urb work
GET_DESCRIPTOR
DEVICE_DESCRIPTOR
got port stat work
status: 0x0111
change: 0x0000
flags:  0x00
port is resetting
-> completing reset
port is disabled
got port stat work
status: 0x0103
change: 0x0010
flags:  0x00
RESET successfull -> use default address
got port stat work
status: 0x0103
change: 0x0000
flags:  0x00
got process urb work
SET_ADDRESS (adr=2)
got process urb work
GET_DESCRIPTOR
DEVICE_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
CONFIGURATION_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
CONFIGURATION_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
STRING_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
STRING_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
STRING_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
STRING_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
STRING_DESCRIPTOR
got process urb work
GET_DESCRIPTOR
STRING_DESCRIPTOR
got process urb work
SET_CONFIGURATION

```

然后再查看就可以看到

```bash
$ lsusb -t
/:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=usb_vhci_hcd/1p, 480M
    |__ Port 1: Dev 2, If 0, Class=(Defined at Interface level), Driver=skeleton, 12M
/:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=ohci-pci/12p, 12M
    |__ Port 1: Dev 2, If 0, Class=Human Interface Device, Driver=usbhid, 12M
```
