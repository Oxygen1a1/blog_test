---
title: windows驱动-文件过滤(minifilter)
date: 2023-07-23
categories: 
  - windows驱动编程
author: oxygen
---

# 文件过滤系统

## 文件过滤系统的类型

​	在文件过滤系统中,有两种过滤系统,一种就算`legacy`老的过滤系统,这种还是老样子，就像之前学得WDM一样,需要附加文件系统设备,但是发送`IRP`请求

​	而另一种则是`minifilter`,这个是最新的文件过滤系统,但是他需要设置`.inf`文件来自启动;此外,他还需要注册自身,以便开始过滤;而`minifilter`有两个回调,一个就算`preCallback`,一个是`postCallback`;**这里可以把`preCallback`就像我们之前键鼠过滤的`dispatchFunc`派遣函数一样,而`endCallback`则理解为我们键鼠驱动设置的`completeRoutine`即可**;

​	此外,使用`Minifilter`,也称为`ftlmgr`,有下面的好处

优点:

1.增加开发速度

2.不用关心IRP处理工作,这些交给 Filter Manager处理即可.

## minifilter的创建

总的来说,小文件过滤系统创建非常麻烦,类似于进程线程回调一样,需要预先定义相关的结构体;

而且,对于minifilter的驱动,是不需要自己在写`drvUnload`了,直接在注册minifilter的时候指定即可;

### filterObj的注册

这个代表的minifilter这个对象,注册(创建)方法如下

```C++
auto freg = FLT_REGISTRATION{
		sizeof FLT_REGISTRATION,
		FLT_REGISTRATION_VERSION,
		0,
		nullptr,
		callbacks,//回调
		unloadFunc,//minifilter 卸载回调
		nullptr,
		nullptr,
		nullptr,
		nullptr,
		nullptr,
		nullptr,
		nullptr,
		nullptr
	};
status=FltRegisterFilter(drv, &freg, &fltFilter);
```

创建完成之后,使用`FltUnregisterFilter(fltFilter);`来进行取消注册;而注册的结构便是`FLT_REGISTRATION`;

这里比较关注的是`callBack`以及`unloadFunc`,其中callback就类似`WDM`框架中的`IRP_MJ_XX`的回调,只不过这个callbacks是一个数组,他的定义如下

```C++
const FLT_OPERATION_REGISTRATION callbacks[] = {
	{IRP_MJ_CREATE,0,preCallbackCreate/*precallback*/,postCallbackCreate/*post callabck*/},
	{IRP_MJ_WRITE,0,preCallbackWrite/*precallback*/,nullptr/*post callabck*/},
	//结束的结构 其实就是0
	{IRP_MJ_OPERATION_END}
};
```

可以看到,这里我们把Callbacks里面只定义了`Create`和`Write`,也就是对文件进行`CreateFile`和`WriteFile`的时候会被拦截,而最后一个`{IRP_MJ_OPERATION_END}`用于告诉系统这个是结束的标志;

此外,每一个`FLT_OPERATION_REGISTRATION`都有两个回调函数,分别是`preCallback`和`postCallback`,一般来说,`postCallback`不太用到;

### IRP 回调函数

下面给出pre和postCallback的Create声明,因为在注册的时候,填写的是`IRP_MJ_WRITE`,所以只有在调用WriteFile的时候才会进入这个函数;

```C++
auto postCallbackCreate(PFLT_CALLBACK_DATA data, PCFLT_RELATED_OBJECTS fltObjs, void* completeContext,FLT_POST_OPERATION_FLAGS flags)->FLT_POSTOP_CALLBACK_STATUS{

	KdPrint(("post create callback!\r\n"));
	return FLT_POSTOP_FINISHED_PROCESSING;
}
auto preCallbackCreate(PFLT_CALLBACK_DATA data,PCFLT_RELATED_OBJECTS fltObjs,void** completeContext)->FLT_PREOP_CALLBACK_STATUS {

	auto ffni = PFLT_FILE_NAME_INFORMATION{ 0 };
	WCHAR fileName[260]={ 0 };
	auto status=FltGetFileNameInformation(data, 
		FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT, 
		&ffni);
	if (NT_SUCCESS(status)) {

		status=FltParseFileNameInformation(ffni);
		//KdPrint(("name info"))
		if (NT_SUCCESS(status)) {
			//获取filename
			if (ffni->Name.MaximumLength < 260) {

				memcpy(fileName, ffni->Name.Buffer, ffni->Name.Length);
				//DbgPrintEx(77,0,"create file name : %ws\r\n", fileName);
			}
		}

		FltReleaseFileNameInformation(ffni);
	}

	return FLT_PREOP_SUCCESS_WITH_CALLBACK;//FLT_PREOP_SUCCESS_no_CALLBACK 如果返回这个 postcallback不会被调用
}

```

其中`data`至关重要,可以调用一系列`ftl`的函数,来获取这个文件的一些信息,比如上面就是用`FltGetFileNameInformation(data,FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,&ffni);`来获取的文件的基础信息,并据此,调用`FltParseFileNameInformation`并获取文件的名字;最后释放,`FltReleaseFileNameInformation(ffni);`

## minifilter的通信

我们知道,对于`WDM`模型的驱动,一般采用的是IOCTL进行通信,而基于`minifilter `驱动也有自己独特的通信方法;

叫做创建端口,端口顾名思义,至少需要两个才能通信,

### 通信端口的创建

```C++
RtlInitUnicodeString(&name, L"\\mf");
status |= FltBuildDefaultSecurityDescriptor(&sd, FLT_PORT_ALL_ACCESS);
InitializeObjectAttributes(&oa, &name, OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE, NULL, sd);
status = FltCreateCommunicationPort(fltFilter, &port, &oa, nullptr, connectCallback, disconnectCallback, messageCallback, 1);
FltFreeSecurityDescriptor(sd);
```

在`FltCreateCommunicationPort`的时候,可以发现,有三个回调,其实就是分别是端口链接(用户R3端口链接)的时候,会执行这个,而通信的时候,一般执行`messageCallback`;

值得一提的是,在`connect`回调里面,我们需要先保存`clientPort`,这样以便于`disConnect`的时候关闭关闭这个客户端;

```C++
//通信用的三个回调
auto connectCallback(PFLT_PORT clientPort, PVOID portCookie, PVOID context, ULONG size, PVOID* connectCookie) {
	
	__debugbreak();
	g_clientPort = clientPort;//是这个 而不是我们当时创建的
	DbgPrintEx(77, 0, "connect!\r\n");
	return STATUS_SUCCESS;
}



auto disconnectCallback(void* connect_cookie) -> void {

	DbgPrintEx(77, 0, "disconnect!\r\n");

	//用这个关闭 关clientPort
	FltCloseClientPort(fltFilter, &g_clientPort);
	DbgPrintEx(77, 0, "send Message!\r\n");
}


auto messageCallback(void* port_cookie,void* input_buf,ULONG input_len,void* out_buf,ULONG out_len,PULONG ret_outbuf_len) {

	__debugbreak();
	PRINTK("message Callback\r\n");

	PRINTK("user msg is :%s\r\n", (PCHAR)input_buf);

	//修改用户传进来的缓冲区
	strcpy((char*)out_buf, "haha!");
	return STATUS_SUCCESS;
}
```

如上述代码,使用`FltCloseClientPort(fltFilter, &g_clientPort);`来进行关闭;

### R3通信

R3通信比较简单,但是需要注意一些细节,需要包含`flt`头文件;

```C++
#include <iostream>
#include <windows.h>
#include <fltUser.h>

#pragma comment(lib,"user32.lib")
#pragma comment(lib,"kernel32.lib")
#pragma comment(lib,"fltlib.lib")
#pragma comment(lib,"fltMgr.lib")
FilterConnectCommunicationPort(L"\\mf", 0, 0, 0, 0, &port);

    if (port == NULL || port == INVALID_HANDLE_VALUE) {

        cout << "failed to get port!" << endl;

        system("pause");
    }
auto ret=FilterSendMessage(port, buf, sizeof buf, recBuf, sizeof recBuf, 0);
```

上述代码`FilterConnectCommunicationPort`就是链接端口用的,而`FilterSendMessage`就是用于通信;

至此,一个文件系统过滤驱动完成,从而过滤拦截文件的操作;
