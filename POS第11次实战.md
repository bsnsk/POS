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

参考 <http://www.linux-usb.org/gadget/>
