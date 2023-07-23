---
title: windows驱动-键鼠过滤和键鼠模拟
date: 2023-07-23
categories: 
  - windows驱动编程
author: oxygen
---

# 键盘过滤

与串口过滤相似,键盘的过滤也是通过Device_object来进行的附加原设备;

一般来说,键盘过滤的设备对象也是有名字的,叫做"kbdcalsss0",当然,如果键盘更多的话,会有更多;

## 如何Attch Device

对于键盘的过滤设备附加,有两个函数

- IoAttchDevice

- IoAttchDeviceToDeviceStack

第一个是要知道原始设备对象的名字,返回的是==lower dev==,第二个需要原来的设备对象;二者各有好处,因此这里也可以用两种方法;

### IoAttchDevice

```c++
	//接下来使用缓冲区 其实kbd一般就是用systembuffer 也就是BUFFERED
	ftl_dev->Flags |= DO_BUFFERED_IO;
	//代表着开始
	ftl_dev->Flags &= ~DO_DEVICE_INITIALIZING;
	RtlSecureZeroMemory(ftl_dev->DeviceExtension, sizeof(KBD_EXT_INFO));
	//ftl_dev->DeviceType = next_dev->DeviceType;
	//因为有名字 直接用IoAttchDevice
	PDEVICE_OBJECT next_dev{ 0 };
	status = IoAttachDevice(ftl_dev, &kbd_name, &next_dev);
```

这里需要注意的就是flags其实只需要设置两个即可,一个就是DO_BUFFERED_IO;第二个是`ftl_dev->Flags &= ~DO_DEVICE_INITIALIZING;`这个代表着设备正式开始执行;

事实上,如果填DO_DIRECT_IO,则直接在UserBuffer中找,DO_POWER_PAGABLE则是MdlAddress映射;

### IoAttchDeviceToDeviceStack

用这个附加设备更麻烦些,需要先找到键盘设备对象;一般找方法是通过

```c++
EXTERN_C NTSTATUS ObReferenceObjectByName(PUNICODE_STRING ObjectName,
	ULONG Attributes,
	PACCESS_STATE AccessState,
	ACCESS_MASK DesiredAccess,
	POBJECT_TYPE ObjectType,
	KPROCESSOR_MODE AccessMode,
	PVOID ParseContext,
	PVOID* Object);
```

而ObjectType填`EXTERN_C POBJECT_TYPE* IoDriverObjectType;`这些都是导出但是wdm中没有定义;

而名字填`#define KBD_DRIVER_NAME  L"\\Driver\\Kbdclass"`;

可以看到,这里找到的是驱动对象,而找到键盘设备对象的方法就是遍历这个驱动对象的Device链表;

```c++
auto cur_dev = kbd_drv_obj->DeviceObject;
	PDEVICE_OBJECT ftl_dev{ 0 };
	while (cur_dev) {
		status = IoCreateDevice(drv_obj,
			sizeof(DEV_EXT_INFO),//用于附加信息,比如可以把设备栈的下一个给附加上
			0,
			cur_dev->DeviceType,
			cur_dev->Characteristics,
			0,
			&ftl_dev
		);

		//注意,返回的dev和target dev有时候不一样,返回的一定是之前位于最顶端的
		//第二个参数你可以认为只要在设备栈上,就可以
		auto next_dev = IoAttachDeviceToDeviceStack(ftl_dev, cur_dev);
```

## next_dev存在哪里?

在派遣函数处理中,需要IoCallDriver,第一个就是设备栈的下一个设备,而上面调用的那两个函数都是可以返回之前顶层的设备栈;

这里可以使用DeviceObject->DeviceExtension来保存,只需要在CreateDevice加上这个大小就可以了;

## Read派遣函数

Read派遣被调用就代表csrss的系统线程发送了一个请求,需要读取键盘端口上面的扫描码;

但是不一定键盘按下,因此存在一个==异步==现象,这个时候如何获取当前Read的IRP最终获取到的键盘按键呢?

这里需要一个叫做这个的函数

```c++
VOID
IoSetCompletionRoutine(
    _In_ PIRP Irp,
    _In_opt_ PIO_COMPLETION_ROUTINE CompletionRoutine,
    _In_opt_ __drv_aliasesMem PVOID Context,
    _In_ BOOLEAN InvokeOnSuccess,
    _In_ BOOLEAN InvokeOnError,
    _In_ BOOLEAN InvokeOnCancel
    );
```

它可以设置一个回调,让IRP结束后,可以走回调,这个时候可以在IRP中读取相关信息;

```c++
NTSTATUS dis_func_read(PDEVICE_OBJECT dev, PIRP irp) {
	auto ext = (PKBD_EXT_INFO)dev->DeviceExtension;
	auto next_dev = ext->next_dev;
	IoCopyCurrentIrpStackLocationToNext(irp);

	//安装return回调
	IoSetCompletionRoutine(irp, read_routine, 0, 1, 1, 1);
	g_pending_count++;
	//IoSkipCurrentIrpStackLocation(irp);
	return IoCallDriver(next_dev, irp);
}
```

## 如何结束

考虑如下情况,当Read IRP被执行,安装了CompletionRoutine,这个时候如果卸载了驱动,等到下次按键盘时,会直接产生BSOD;

因为这个时候需要确保所有的READ IRP的完成回调已经执行完;

这里采取一个非常简单的解决方法,那就是设置一个全局变量;叫做pending_count;

一旦IRP READ进入,+1;而IRP READ 完成回调完成-1;

因此结束进程可以这样写

IoDetachDevice(next_dev);
```C++
while (g_pending_count) {
	LARGE_INTEGER interval = { 0 };
	interval.QuadPart = -10 * 1000 * 1000;
	KeDelayExecutionThread(KernelMode, FALSE, &interval);
}

IoDeleteDevice(ftl_dev);
DbgPrintEx(77, 0, "[+]drv unload success\r\n");
```

## Read完成时的回调写法

在Read完成时,会进入之前设置的完成回调,这个时候如果想要读取正确的键盘,需要按照一定的结构读取;

参考[此链接](https://learn.microsoft.com/zh-cn/windows/win32/api/ntddkbd/ns-ntddkbd-keyboard_input_data?redirectedfrom=MSDN)查看结构

直接设置了BUFFERED_IO,所以在SystemBuffer中查找;

长度同样在IRP中查找,来确定这个结构的连续的长度;

```c
typedef struct _KEYBOARD_INPUT_DATA {
	USHORT UnitId;
	USHORT MakeCode;//scan code
	USHORT Flags;
	USHORT Reserved;
	ULONG  ExtraInformation;
} KEYBOARD_INPUT_DATA, * PKEYBOARD_INPUT_DATA;
```

而flags代表如下

```c++
char* keyflag[4] = { "keydown","keyup","e0","e1" };
```

因此遍历即可

```c++
for (int i = 0; i < buf_len; i += sizeof(KEYBOARD_INPUT_DATA));
		DbgPrintEx(77, 0, "[+]scan code->0x%x and state is %s\r\n", kbd_str->MakeCode, keyflag[kbd_str->Flags]);
```

这样即可达到遍历按键过滤的目的;当然,可以改变这个缓冲区的相关信息,从而达到过滤;

# 鼠标过滤

鼠标过滤和键盘过滤很像,唯一区别就是,鼠标过滤的驱动名称和键盘过滤的命名规则不太一样,键盘是`kdbclass0`,而鼠标则是`PointerClass0`

![image-20230528160307477](./assets/image-20230528160307477-1685260988803-1.png)

一般`PointerClass1`才是用于USB的;

## 鼠标附加的设备对象

但是经过测试,发现如果是直接`IoAttchDevice`,通过`DeviceName`来进行附加,并不能拦截键鼠标,原因暂时未知,因此这里采用的是`IoAttchDeviceToDeviceStack`来进行设备附加，方式就是先定位到鼠标的驱动，然后使用`ObReferenceObjectByName`来获取`DriverObject`,然后遍历整个DeviceObj，使用上述函数进行附加

```C++
auto attchDev(PDRIVER_OBJECT drv) -> NTSTATUS {
	UNICODE_STRING drvName = { 0 };
	RtlInitUnicodeString(&drvName, L"\\Driver\\mouclass");
	PVOID fileObj{ nullptr };
	//获取鼠标驱动的驱动对象
	NTSTATUS status=ObReferenceObjectByName(&drvName, 0, 0, FILE_ALL_ACCESS, *IoDriverObjectType, KernelMode, 0, &fileObj);

	if (!NT_SUCCESS(status)) return status;
	const auto drvObj = (PDRIVER_OBJECT)(fileObj);

	auto cur_dev = drvObj->DeviceObject;
	while (cur_dev) {
		//注意,返回的dev和target dev有时候不一样,返回的一定是之前位于最顶端的
		//第二个参数你可以认为只要在设备栈上,就可以
		PDEVICE_OBJECT ftl_dev{ 0 };
		status = IoCreateDevice(drv,
			sizeof dev_ext_t,//用于附加信息,比如可以把设备栈的下一个给附加上
			0,
			cur_dev->DeviceType,
			cur_dev->Characteristics,
			0,
			&ftl_dev
		);

		ftl_dev->Flags |= DO_BUFFERED_IO;
		ftl_dev->Flags &= ~DO_DEVICE_INITIALIZING;

		auto next_dev = IoAttachDeviceToDeviceStack(ftl_dev, cur_dev);
		reinterpret_cast<pdev_ext_t>(ftl_dev->DeviceExtension)->lowerDev = next_dev;
		cur_dev = cur_dev->NextDevice;
		
	}

	return status;
}
```

如上,附加完毕之后,至于其他,则和键盘过滤完全类似,也就是还是现需要在`readDis`里面设置回调,在回调里面进行读取鼠标的相关信息,唯一不同的就是原来的`INPUT_KEYBORD_DATA`变成了`INPUT_MOUSE_DATA`,通过这个可以确定鼠标到底执行了什么操作

```C++
auto readComplete(PDEVICE_OBJECT dev, PIRP irp, PVOID context) -> NTSTATUS {
	UNREFERENCED_PARAMETER(dev);
	UNREFERENCED_PARAMETER(context);
	auto sysBuf = reinterpret_cast<pmouse_input_t>(irp->AssociatedIrp.SystemBuffer);


	//一共有多少个input结构
	const auto num= irp->IoStatus.Information / sizeof mouse_input_t;

	//把信息打印出来
	if (irp->IoStatus.Status == STATUS_SUCCESS) {
		for (int i = 0; i < num; i++) {
			
			const auto state = sysBuf[i].ButtonFlags;
			//MOUSE_LEFT_BUTTON_DOWN	鼠标左键已更改为向下。
			//MOUSE_LEFT_BUTTON_UP	鼠标左键已更改为向上。
			//MOUSE_RIGHT_BUTTON_DOWN	鼠标右键已更改为向下。
			//MOUSE_RIGHT_BUTTON_UP	鼠标右键已更改为向上。
			//MOUSE_MIDDLE_BUTTON_DOWN	鼠标中间按钮已更改为向下。
			//MOUSE_MIDDLE_BUTTON_UP	鼠标中间按钮已更改为向上。
			switch (state)
			{
			case MOUSE_LEFT_BUTTON_DOWN:
				DbgPrintEx(77, 0, "鼠标左键已更改为向下\r\n");
				break;
			case MOUSE_LEFT_BUTTON_UP:
				DbgPrintEx(77, 0, "鼠标左键已更改为向上\r\n");
				break;
			case MOUSE_RIGHT_BUTTON_DOWN:
				DbgPrintEx(77, 0, "鼠标右键已更改为向下\r\n");
				break;
			case MOUSE_RIGHT_BUTTON_UP:
				DbgPrintEx(77, 0, "鼠标右键已更改为向上\r\n");
				break;
			case MOUSE_MIDDLE_BUTTON_DOWN:
				DbgPrintEx(77, 0, "鼠标中间按钮已更改为向下\r\n");
				break;
			case MOUSE_MIDDLE_BUTTON_UP:
				DbgPrintEx(77, 0, "鼠标中间按钮已更改为向上\r\n");
				break;
			default:
				DbgPrintEx(77, 0, "其他事件\r\n");
				break;
			}

		}
	}

	gPending--;

	return irp->IoStatus.Status;
}
```

而这个结构可以在MSDN的官网找到;这样就可以完成鼠标的过滤了

# 键鼠模拟

一直以来,键鼠模拟是FPS游戏所需要的一个功能,下面的代码将讲述和理解驱动模拟键鼠的原理;

## 键鼠类驱动服务程序

看名字可以知道,这个就是键盘鼠标过滤驱动的一部分,但是他必须作为过滤驱动的一部分,也就是他要么属于功能驱动,要么是过滤驱动,但是无论属于哪一个,他都是需要先附加到总线驱动的设备对象;

> 在键鼠设备驱动中，`kbclass.sys`和`mouclass.sys`是功能驱动程序，它们处理键盘和鼠标的主要功能。而与这些设备直接通信的是底层的硬件驱动，这可能是PS/2驱动（`i8042prt.sys`）或USB驱动（`usbhub.sys`等），这些通常被视为总线驱动程序。这两类驱动程序之间可能还有过滤驱动程序，**例如键盘过滤驱动`kbdfilter.sys`**，它可以修改或观察从键盘设备到功能驱动程序的数据流。

值得注意的是,在新版本的WIndows中,kdbfilter.sys这个过滤驱动已经没了;但是,从上面的话不难看到,无论是键鼠类哪一个驱动程序,都是需要先附加到总线驱动程序才能用的;

比如键盘鼠标就是需要先附加到`i8042prt.sys`这个端口总线驱动上面;

![image-20230529095259572](./assets/image-20230529095259572.png)

![image-20230529095333885](./assets/image-20230529095333885.png)

其中i8042prt这个是PS/2端口的键盘和鼠标,而kdbhid则是`USB`的键盘; mouhid则是USB链接的鼠标

### 类驱动程序附加

再根据已有的资料,windows的类驱动程序附加实际上是通过这个CTL_CODE来与`kbfilter`通信使得类驱动程序和端口程序进行链接的

```C++
#define IOCTL_INTERNAL_KEYBOARD_CONNECT CTL_CODE(FILE_DEVICE_KEYBOARD, 0x0080, METHOD_NEITHER, FILE_ANY_ACCESS)
```

> IOCTL_INTERNAL_KEYBOARD_CONNECT 请求将 Kbdclass 服务连接到键盘设备。Kbdclass 在打开键盘设备之前将此请求发送到键盘设备堆栈。
>
> Kbfiltr收到键盘连接请求后，Kbfiltr通过以下方式过滤连接请求：
>
> - 保存由Kbdclass 传递给过滤驱动程序的 Kbdclass 的[CONNECT_DATA (Kbdclass)](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/kbdmou/ns-kbdmou-_connect_data)结构 的副本
> - 用它自己的连接信息代替类驱动程序连接信息
> - 将 IOCTL_INTERNAL_KEYBOARD_CONNECT 请求发送到设备堆栈

但是kdfilt已经消失,猜测可能在`i8042`或者是`kbdhid`中;在`kdbhid.sys`可以找到处理`IOCTL_INTERNAL_KEYBOARD_CONNECT `但是代码过于复杂,没能看懂,

![image-20230529102146149](./assets/image-20230529102146149.png)

总之,根据已有资料,就在这一步,kdclss提供了一个类服务回调函数,函数定义如下

> KeyboardClassServiceCallback例程是 Kbdclass 提供的类服务回调例程**。**功能驱动程序在其 ISR 调度完成例程中调用类服务回调。类服务回调将输入数据从设备的输入数据缓冲区传输到类数据队列。

```C++
VOID KeyboardClassServiceCallback(
  _In_    PDEVICE_OBJECT       DeviceObject,
  _In_    PKEYBOARD_INPUT_DATA InputDataStart,
  _In_    PKEYBOARD_INPUT_DATA InputDataEnd,
  _Inout_ PULONG               InputDataConsumed
);
```

> **KeyboardClassServiceCallback**将输入数据从设备的输入缓冲区传输到类数据队列。该例程由功能驱动程序的 ISR 调度完成例程调用。
>
> **KeyboardClassServiceCallback**可以由上层键盘过滤器驱动程序提供的过滤器服务回调进行补充。过滤器服务回调过滤传输到类数据队列的键盘数据。例如，过滤器服务回调可以删除、转换或插入数据。[Kbfiltr](https://go.microsoft.com/fwlink/p/?linkid=256125)是 MSDN 代码库中的示例筛选器驱动程序，包括[**KbFilter_ServiceCallback**](https://learn.microsoft.com/en-us/previous-versions/ff542297(v=vs.85))，它是键盘筛选器服务回调的模板。

可以看到,这个回调函数是关键,他一旦被调用,就可以==输入数据从设备的输入缓冲区传输到类数据队列==,因此驱动的键盘模拟就是通过调用这个回调函数,来向`kbdclass`这个驱动插入假数据,让他误以为有键鼠信号;

事实上,这个`IoCtl`所传入的缓冲区是一个`CONNECT_DATA`的结构

```C++
typedef struct _CONNECT_DATA {
  IN PDEVICE_OBJECT ClassDeviceObject;
  IN PVOID          ClassService;
} CONNECT_DATA, *PCONNECT_DATA;
```

> ## 成员
>
> ```
> ClassDeviceObject
> ```
>
> 指向上层类[过滤器设备对象](https://learn.microsoft.com/en-us/windows-hardware/drivers/)（过滤器 DO）的指针。
>
> ```
> ClassService
> ```
>
> 指定类服务例程。请参阅 [PSERVICE_CALLBACK_ROUTINE](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/kbdmou/nc-kbdmou-pservice_callback_routine)。
>
> 
>
> ## 评论
>
> 键盘类驱动程序将此结构与[IOCTL_INTERNAL_KEYBOARD_CONNECT](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/kbdmou/ni-kbdmou-ioctl_internal_keyboard_connect)请求一起使用；鼠标类驱动程序使用[IOCTL_INTERNAL_MOUSE_CONNECT](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/kbdmou/ni-kbdmou-ioctl_internal_mouse_connect)。

可以看到,classService就是这个回调;**所以现在的问题在于如何找到`kdbclass`这个驱动在附加端口驱动时候的这个回调了;**

经过查看xp泄露源码的`kbfilter.c`这个文件,不难发现

```C++
case IOCTL_INTERNAL_KEYBOARD_CONNECT:
        //
        // Only allow one connection.
        //
        if (devExt->UpperConnectData.ClassService != NULL) {
            status = STATUS_SHARING_VIOLATION;
            break;
        }
        else if (irpStack->Parameters.DeviceIoControl.InputBufferLength <
                sizeof(CONNECT_DATA)) {
            //
            // invalid buffer
            //
            status = STATUS_INVALID_PARAMETER;
            break;
        }

        //
        // Copy the connection parameters to the device extension.
        //
        connectData = ((PCONNECT_DATA)
            (irpStack->Parameters.DeviceIoControl.Type3InputBuffer));

        devExt->UpperConnectData = *connectData;

        //
        // Hook into the report chain.  Everytime a keyboard packet is reported
        // to the system, KbFilter_ServiceCallback will be called
        //
        connectData->ClassDeviceObject = devExt->Self;
        connectData->ClassService = KbFilter_ServiceCallback;

        break;
```

在devExt(注意,这个Device到底是谁的设备,总之这个设备对象被附加的一定是`kbdclass`这个驱动,他有可能是i8042.sys,有可能是`mouhid.sys` 如果有`kbdfilter.sys`,那么就是这个驱动的设备对象,没有的话,则是后两者);

可以看到,他得到了connectData后,直接放到了`DevExt`的UpperConnect这个位置;也就是设备栈的更高层的设备相关的信息放在这了,毫无疑问,回调也是放在这的;

同时,在最后也可以看到,他把`conntectData`给修改了,这也印证了微软文档的那句话

- 保存由Kbdclass 传递给过滤驱动程序的 Kbdclass 的[CONNECT_DATA (Kbdclass)](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/kbdmou/ns-kbdmou-_connect_data)结构 的副本
- 用它自己的连接信息代替类驱动程序连接信息
- 将 IOCTL_INTERNAL_KEYBOARD_CONNECT 请求发送到设备堆栈

当然，如果没有`kbfiltl.sys`这个驱动,端口键鼠驱动同样是有这个IOCTL的请求的

```C++
case IOCTL_INTERNAL_KEYBOARD_CONNECT:
        //
        // This really isn't something to worry about overall, but it is worthy
        // enough to be noted and recorded.  The multiple starts will be handled in
        // I8xPnp and I8xKeyboardStartDevice routines
        //
        if (KEYBOARD_PRESENT()) {
            Print(DBG_ALWAYS, ("Received 1+ kb connects!\n"));
            SET_HW_FLAGS(DUP_KEYBOARD_HARDWARE_PRESENT);
        }

        InterlockedIncrement(&Globals.AddedKeyboards);

        kbExtension->IsKeyboard = TRUE;

        SET_HW_FLAGS(KEYBOARD_HARDWARE_PRESENT);

        Print(DBG_IOCTL_INFO, ("IOCTL: keyboard connect\n"));

        //
        // Only allow a connection if the keyboard hardware is present.
        // Also, only allow one connection.
        //
        if (kbExtension->ConnectData.ClassService != NULL) {

            Print(DBG_IOCTL_ERROR, ("IOCTL: error - already connected\n"));
            status = STATUS_SHARING_VIOLATION;
            break;
        }
        else if (irpSp->Parameters.DeviceIoControl.InputBufferLength <
                sizeof(CONNECT_DATA)) {

            Print(DBG_IOCTL_ERROR, ("IOCTL: error - invalid buffer length\n"));
            status = STATUS_INVALID_PARAMETER;
            break;
        }

        //
        // Copy the connection parameters to the device extension.
        //

        kbExtension->ConnectData =
            *((PCONNECT_DATA) (irpSp->Parameters.DeviceIoControl.Type3InputBuffer));

        hookKeyboard = ExAllocatePool(PagedPool,
                                      sizeof(INTERNAL_I8042_HOOK_KEYBOARD)
                                      );
        if (hookKeyboard) {
            topOfStack = IoGetAttachedDeviceReference(kbExtension->Self);

            RtlZeroMemory(hookKeyboard,
                          sizeof(INTERNAL_I8042_HOOK_KEYBOARD)
                          );

            hookKeyboard->CallContext = (PVOID) DeviceObject;

            hookKeyboard->QueueKeyboardPacket = (PI8042_QUEUE_PACKET)
                I8xQueueCurrentKeyboardInput;

            hookKeyboard->IsrWritePort = (PI8042_ISR_WRITE_PORT)
                I8xKeyboardIsrWritePort;

            I8xSendIoctl(topOfStack,
                         IOCTL_INTERNAL_I8042_HOOK_KEYBOARD,
                         (PVOID) hookKeyboard,
                         sizeof(INTERNAL_I8042_HOOK_KEYBOARD)
                         );

            ObDereferenceObject(topOfStack);
            ExFreePool(hookKeyboard);
        }

        status = STATUS_SUCCESS;
        break;
```

可以看到,它实现的方法是类似的,也是把devExt的`connectData`给换成传进来的数据;

由此,只要无脑遍历Ext,可以定位到`ConnectData`;并定位到这个回调;就可以调用这个回调,然后模拟键鼠了;

但是需要注意的是,由于不确定什么时候能找到类驱动程序(kbdclass,mouclass)的LowerDev,所以得从端口驱动开始遍历,一直查他的`AttachedDevice`,直到`AttachedDevice==kdbclass.Deivce`的时候,可以算是找到了相应的设备对象,然后,只需要遍历这个设备对象的`Ext`即可,**找到一个地址,它位于类驱动程序的范围内(最好是text节)**,然后模拟即可;

下面是源码

```C++
//kdbDriver是kbdclass的驱动 pPortDev是 最前i8042prt或者是kdhid的设备(应该是属于键鼠最底层的设备了)
NTSTATUS SearchServiceFromKdbExt(PDRIVER_OBJECT KbdDriverObject,PDEVICE_OBJECT pPortDev)
{
  PDEVICE_OBJECT pTargetDeviceObject = NULL;
  UCHAR *DeviceExt;
  int i=0;
  NTSTATUS status;
  PVOID KbdDriverStart;
  ULONG KbdDriverSize = 0;  
  PDEVICE_OBJECT  pTmpDev;
  UNICODE_STRING  kbdDriName;

  KbdDriverStart = KbdDriverObject->DriverStart;
  KbdDriverSize = KbdDriverObject->DriverSize;

  status = STATUS_UNSUCCESSFUL;

  RtlInitUnicodeString(&kbdDriName,L"\\Driver\\kbdclass");
  pTmpDev = pPortDev;
  while(pTmpDev->AttachedDevice != NULL)
  {
    KdPrint(("Att:  0x%x",pTmpDev->AttachedDevice));
    KdPrint(("Dri Name : %wZ",&pTmpDev->AttachedDevice->DriverObject->DriverName));
    if(RtlCompareUnicodeString(&pTmpDev->AttachedDevice->DriverObject->DriverName,
      &kbdDriName,TRUE) == 0)
    {
      break;
    }
    pTmpDev = pTmpDev->AttachedDevice;
  }
  if(pTmpDev->AttachedDevice == NULL)
  {
    return status;
  }

  //这个时候 pTmpDev->AttachedDevice一定是kdbDriverObject所属的某个设备
  pTargetDeviceObject = KbdDriverObject->DeviceObject;  
  while(pTargetDeviceObject)  
  {
     //一直找 找到相同为止
    if(pTmpDev->AttachedDevice != pTargetDeviceObject)
    {
      pTargetDeviceObject = pTargetDeviceObject->NextDevice; 
      continue;
    }
    //找到了 目前不确定为啥要这样 而不是直接遍历kdbDriverObject的设备,难道说kdbDriverObject的
    //设备不全是附加到下一级过滤驱动的设备吗?
    //这个意思就是找到下一个(UpperDev)是kdbDriver的设备里面的DeviceExt
    DeviceExt = (UCHAR *)pTmpDev->DeviceExtension;
    g_KoMCallBack.KdbDeviceObject = NULL;
    //遍历我们先找到的端口驱动的设备扩展的每一个指针  
    for (i=0;i<4096;i++, DeviceExt++)  
    {  
      PVOID tmp;  
      if (!MmIsAddressValid(DeviceExt))  
      {  
        break;  
      }  
      //找到后会填写到这个全局变量中，这里检查是否已经填好了  
      //如果已经填好了就不用继续找了，可以直接退出  
      if (g_KoMCallBack.KdbDeviceObject && g_KoMCallBack.KeyboardClassServiceCallback)  
      {  
        status = STATUS_SUCCESS;  
        break;  
      }

      //在端口驱动的设备扩展里，找到了类驱动设备对象，填好类驱动设备对象后继续  
      tmp = *(PVOID*)DeviceExt;  
      if (tmp == pTargetDeviceObject)  
      {  
        g_KoMCallBack.KdbDeviceObject = pTargetDeviceObject;  
        continue;  
      }  

      //如果在设备扩展中找到一个地址位于KbdClass这个驱动中，就可以认为，这就是我们要找的回调函数  
      if ((tmp > KbdDriverStart) &&  (tmp < (UCHAR*)KbdDriverStart+KbdDriverSize) &&  
        (MmIsAddressValid(tmp)))  
      {  
        //将这个回调函数记录下来  
        g_KoMCallBack.KeyboardClassServiceCallback = (MY_KEYBOARDCALLBACK)tmp;  
      }  
    }  
    if(status == STATUS_SUCCESS)
    {
      break;
    }
    //换成下一个设备，继续遍历  
    pTargetDeviceObject = pTargetDeviceObject->NextDevice;  
  }  
  return status;
}
```

这也即可遍历出回调以及相应的设备;

## 调用类服务驱动的回调完成模拟

这一步很简单,在学完键鼠过滤,我们知道,再附加设备之后,经常设置一个回调,正是这个回调,可以看到键鼠的相关信息,而解析这个信息的,用到了一些结构体,分别是

`KEYBOARD_INPUT_DATA`和`MOUSE_INPUT_DATA`,而这个回调就是如此,模拟键鼠消息的时候,需要先分配两个结构体,第一个就是要模拟的数值,第二个作为结尾用,清理;

代码如下

```C++
MouseInputDataStart = mid;
MouseInputDataEnd = MouseInputDataStart + 1;
MouseInputDataStart = mid;
	MouseInputDataEnd = MouseInputDataStart + 1;
	if (g_KoMCallBack.MouseClassServiceCallback)
	{

		KeEnterGuardedRegion();
		KeRaiseIrql(DISPATCH_LEVEL, &irql);
		if (MmIsAddressValid(g_KoMCallBack.MouDeviceObject->DeviceExtension))
		{
			PULONG64 spin = (PULONG64)(((PUCHAR)g_KoMCallBack.MouDeviceObject->DeviceExtension) + 0x90);
			if (MmIsAddressValid(spin))
			{
				g_KoMCallBack.MouseClassServiceCallback(g_KoMCallBack.MouDeviceObject,
					MouseInputDataStart,
					MouseInputDataEnd,
					&InputDataConsumed);
			}
		}
		
	
		KeLowerIrql(irql);
		KeLeaveGuardedRegion();
	}
```

此外,还需要注意的是,他把IRQL提升到了DPC_LEVEL,而且做了一些检查;
