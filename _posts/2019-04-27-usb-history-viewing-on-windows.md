---
layout: post
title: "Windows查看USB插拔记录"
excerpt: "这篇文章将会教大家如何从取证的角度在Windows中查看USB插拔记录"
tags:
    - Windows
    - USB
categories:
    - 取证分析
last_modified_at: 2019-04-27T21:59:38+08:00
---

## Setup API Logs

Windows XP

```
\Windows\setupapi.log
\Windows\setupapi.log.old
```

[Windows 7](https://technet.microsoft.com/en-us/library/ee851579(v=ws.10).aspx) and up include

```
\Windows\inf\setupapi.app.log
\Windows\inf\setupapi.dev.log
\Windows\inf\setupapi.offline.log
```

Additional Windows 10

```
\Windows\inf\setupapi.upgrade.log
```

在这些log中记录了USB设备的首次插入的时间，利用grep命令进行搜索，如下：

```bash
$ grep 'Device Install.*USBSTOR' setupapi.dev.log -A 1
>>>  [Device Install (Hardware initiated) - SWD\WPDBUSENUM\_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}]
>>>  Section start 2019/04/20 21:46:53.550
```

那么我们能够得到的信息如下：

序列号：AA00000000011178&0 (如果第二个字符是&那么表示是系统生成的并非设备序列号，此处是A，所以表示这串表示的就是设备序列号)

设备类ID：53f56307-b6bf-11d0-94f2-00a0c91efb8b

初次插入USB时间(本地时间，也就是北京时间)：2019/04/20 21:46:53.550



根据微软资料显示，这个类ID表示的就是存储设备：

> The GUID_DEVINTERFACE_DISK [device interface class](https://msdn.microsoft.com/library/windows/hardware/ff541339) is defined for hard disk [storage devices](https://msdn.microsoft.com/library/windows/hardware/ff566969).
>
> | Attribute  | Setting                                |
> | :--------- | :------------------------------------- |
> | Identifier | GUID_DEVINTERFACE_DISK                 |
> | Class GUID | {53F56307-B6BF-11D0-94F2-00A0C91EFB8B} |

还有一个表示卷设备的类ID`53F5630D-B6BF-11D0-94F2-00A0C91EFB8B`，后面要用到：

> The GUID_DEVINTERFACE_VOLUME [device interface class](https://msdn.microsoft.com/library/windows/hardware/ff541339) is defined for volume devices.
>
> | Attribute  | Setting                                |
> | :--------- | :------------------------------------- |
> | Identifier | GUID_DEVINTERFACE_VOLUME               |
> | Class GUID | {53F5630D-B6BF-11D0-94F2-00A0C91EFB8B} |



## 注册表

### Fred自动生成报告

注册表里面的信息可以使用[Fred](https://www.pinguin.lu/fred)自动生成报告，简单方便，用Fred打开注册表后Reports -> Generate Report就能生成报告，并且还会附带其它一些有用的信息(图中展示的只是一部分，还有很多信息)：

![fred-report](/assets/images/fred-report.png)

但是有的时候我们可能不能不手工分析，比如注册表文件损坏，只能看到一部分的时候，就只能自己手工分析(比如我碰到过Fred解析不了但是可以用RegRipper解析的情况)



### HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\USBSTOR\

这个键也保存的有USB设备的信息，如下图所示：

![usb-history-viewing](/assets/images/usb-history-viewing.png)

但是Windows自带的注册表看不见时间记录信息，需要借助三方工具

* Windows在线分析的时候可以使用工具[RegistryExplorer](https://f001.backblazeb2.com/file/EricZimmermanTools/RegistryExplorer_RECmd.zip)(在Win7上使用该工具需要安装.NET框架)，这个工具可能会出现字体太小的问题，如图方式可以解决：

  ![registry-explorer](/assets/images/registry-explorer.png)

* fred：<https://www.pinguin.lu/fred>

以下演示都用fred：根据维基百科参考资料显示，我们需要用fred(Windows版本太老，亦无法显示时间，这里使用Linux版本作演示)打开`C:\Windows\System32\config\SYSTEM`

> Windows NT systems store the registry in a binary file format which can be exported, loaded and unloaded by the Registry Editor in these operating systems. The following system registry files are stored in `%SystemRoot%\System32\Config\`:
>
> - `Sam` – HKEY_LOCAL_MACHINE\SAM
> - `Security` – HKEY_LOCAL_MACHINE\SECURITY
> - `Software` – HKEY_LOCAL_MACHINE\SOFTWARE
> - `System` – HKEY_LOCAL_MACHINE\SYSTEM
> - `Default` – HKEY_USERS\.DEFAULT
> - `Userdiff` – Not associated with a hive. Used only when upgrading operating systems
>
> The following file is stored in each user's profile folder:
>
> - `%USERPROFILE%\Ntuser.dat` – HKEY_USERS\<[User SID](https://en.wikipedia.org/wiki/Security_Identifier)> (linked to by HKEY_CURRENT_USER)



然而此处会出现一个问题，这些注册表文件是属于Windows系统独占的文件，除了Windows自身别的程序无法打开或者复制这些文件，因此需要先将注册表导出，以下提供几种方法，实测第一种方法导出的hive文件会略小于实际大小，因为没有作文件对齐(末尾未补零)：

* 以管理员身份打开cmd输入命令：reg save HKLM\System system.hiv。之后会在当前目录生成system.hiv文件
* 利用FTKImager可以导出，因为它是通过类似挂载硬盘的方式，而不是通过操作系统去操作，因此Windows无法阻止它，提供下载链接：
  * 最新版但是需要安装：<http://ad-exe.s3.amazonaws.com/AccessData_FTK_Imager--4.2.1.exe>
  * 解压直接使用版(推荐)：<https://ad-zip.s3.amazonaws.com/Imager_Lite_3.1.1.zip>
* 关机，然后启动Live USB将文件copy出来，但是这样做的坏处是需要关机



接下来利用fred打开注册表文件：

![fred-demo](/assets/images/fred-demo.png)

可以看到多出了一列：`Last mod. time`。这一列记录了各个键最后一个被修改的时间(UTC时间)

最后修改时间：2019/04/20 21:46:52



### HKEY_LOCAL_MACHINE\SYSTEM\MountedDevices

这个键下面记录了设备的挂载记录，包括USB和硬盘记录，通过查看MountedDevices可以看出USB挂载的时候所使用的盘符

![fred-mounted-devices](/assets/images/fred-mounted-devices.png)

根据对应的值可以看出USB挂载在了F盘，如果想要准确的Unicode字符串可以利用一些工具，例如010editor，将值复制到剪贴板后，打开010editor切换到hex模式然后使用快捷键`CTRL+SHIFT+V`就能将剪贴板数据当做二进制数据粘贴，然后转换到Unicode模式就能得到完整字符串：

`_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}`

可以看到和上面setupapi.dev.log看到的数据完全一致，代表这个USB就是setupapi里面记录的USB，并且MountedDevices记录的最后修改时间也是：2019/04/20 21:46:52。这说明系统最后插入的USB就是此设备



### HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceClasses

用fred打开两个键的内容(上面提到的两个键)：

![fred-device-classes](/assets/images/fred-device-classes.png)

为了方便查看很长的数据，我们还可以用`reglookup`来查看，命令如下：

`reglookup -p 'ControlSet001/Control/DeviceClasses/{53f56307-b6bf-11d0-94f2-00a0c91efb8b}' SYSTEM | grep USBSTOR`

输出如下：

```
/ControlSet001/Control/DeviceClasses/{53f56307-b6bf-11d0-94f2-00a0c91efb8b}/##?#USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b},KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f56307-b6bf-11d0-94f2-00a0c91efb8b}/##?#USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}/DeviceInstance,SZ,USBSTOR\Disk&Ven_SMI&Prod_USB_DISK&Rev_1100\AA00000000011178&0,
/ControlSet001/Control/DeviceClasses/{53f56307-b6bf-11d0-94f2-00a0c91efb8b}/##?#USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}/#,KEY,,2019-04-20 13:46:52
```

这里的时间戳表示的是在Windows最后一次启动过程中首次连接USB的时间是：21:46:52

再查看另外一个键：

`reglookup -p 'ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}' SYSTEM 2> /dev/null | grep USBSTOR`

输出如下：

```
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b},KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/DeviceInstance,SZ,STORAGE\Volume\_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b},
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9},KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0002,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0002/,0xFFFF0011,%FF,
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0003,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0003/,0xFFFF0011,%FF,
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0004,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0004/,0xFFFF0011,%00,
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0005,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0005/,0xFFFF0007,%01%00%00%00,
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0006,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0006/,0xFFFF0007,%01%00%00%00,
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0007,KEY,,2019-04-20 13:46:52
/ControlSet001/Control/DeviceClasses/{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/##?#STORAGE#Volume#_??_USBSTOR#Disk&Ven_SMI&Prod_USB_DISK&Rev_1100#AA00000000011178&0#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}#{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}/#/Properties/{4d1ebee8-0803-4774-9842-b77db50265e9}/0007/,0xFFFF0003,%0C,
```

记录的时间同样是：21:46:52



### HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\USB

这个键下面的内容也可能会随着插拔USB而更新，而且经测试Win7在某些情况插入USB的时候仅有此键会记录插入时间！在该例中对应的键为：

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\USB\VID_090C&PID_1000\AA00000000011178`



### 监控注册表验证

利用[Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite)套件自带的[Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon)对注册表事件进行监控，并利用[USBDeview](https://www.nirsoft.net/utils/usb_devices_view.html)对USB插拔事件监控，再结合命令行搜索时间字符串(`reg save HKLM\SYSTEM SYSTEM.reg /y && reglookup.exe SYSTEM.reg 2>ERROR | findstr 14:43:14 2>ERROR2`)，得出以下结论：

* 经验证，在Win7/Win10上仅插入USB会刷新注册表项的时间，因此可以根据时间推断最后一次插入USB的时间但是无法知道USB拔出的时间

* 电脑重启后时间戳可能会变成重启时刻的时间戳，验证了USBDeview官方的一句话：

  > **Notice:** According to user reports, On some systems the 'Last Plug/Unplug Date' and the 'Created Date' values are initialized after reboot. This means that these columns may display the reboot time instead of the correct date/time.

附上用来验证用的ProcMon.exe的配置文件，可以导入自行验证：

[usb.pmc](https://cl.ly/c883fe4c02b7)


## 安全日志

参考自维基百科，在安全日志中可以审计如下记录：

> [6416](https://docs.microsoft.com/en-us/windows/device-security/auditing/event-6416): A new external device was recognized by the System.
>
> 6419: A request was made to disable a device.
>
> 6420: A device was disabled.
>
> 6421: A request was made to enable a device.
>
> 6422: A device was enabled.
>
> 6423: The installation of this device is forbidden by system policy.
>
> 6424: The installation of this device was allowed, after having previously been forbidden by policy.
>
> Microsoft-Windows-Partition/Diagnostic
>
> [1006](https://twitter.com/mattifestation/status/916338889840721920?s=03): May contain Manufacturer, Model, Serial, and raw Partition Table, MFT, and VBR data.

打开Windows的事件查看器，用事件ID筛选日志：

`1006,6416,6419-6424`

然后进行审计即可，如图所示：

![secpol-events](/assets/images/secpol-events.png)

但是Windows 10默认不开启PnP日志，若要开启，需要做如下操作：



先启用本地安全策略(可以用secpol.msc命令打开)的蓝色光标处的项：

![secpol-config](/assets/images/secpol-config.png)

然后打开PnP安全策略：

![secpol-PnP](/assets/images/secpol-PnP.png)

最后需要重启Windows，就能启用PnP安全策略了

## 参考文档

<https://www.forensicswiki.org/wiki/Setup_API_Logs>

<https://www.forensicswiki.org/wiki/USB_History_Viewing>

<https://en.wikipedia.org/wiki/Windows_Registry>

<https://docs.microsoft.com/en-us/windows-hardware/drivers/install/guid-devinterface-disk>

<https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/audit-pnp-activity>

