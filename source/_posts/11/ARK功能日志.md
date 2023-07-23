---
title: 我的第二个博客
date: 2023-07-22 16:10:22
categories: TEST
password: mysecretpassword
encrypt: false
---
## 简介

> 此笔记记录了我写windows平台ARK工具的各种功能实现的原理以及日志

### 项目介绍

Oxygen Anti Rootkit tools,是一款内核态反病毒的工具,有进程 驱动模块 应用层钩子 内核钩子 文件系统 监控模块 系统信息模块;

## 进程模块

### 模块介绍

进程作为ARK 工具的最基础的模块,功能主要如下

### 进程线程展示

#### R3部分

进程展示我才用的是部分R3+部分R0询问信息的方式;

在R3部分，采用的是`ZwQuerySystemInformation`来获取`SystemProcessInformation`即OS的所有进程信息;

代码如下

```C++
		PSYSTEM_PROCESS_INFORMATION psp = NULL;
		DWORD dwNeedSize = 0;
		status = ZwQuerySystemInformation(SystemProcessInformation, NULL, 0, &dwNeedSize);
		if (status == STATUS_INFO_LENGTH_MISMATCH)
		{
			BYTE* pBuffer = new BYTE[dwNeedSize+PAGE_SIZE];

			status = ZwQuerySystemInformation(SystemProcessInformation, (PVOID)pBuffer, dwNeedSize+PAGE_SIZE, &dwNeedSize);
			if (status == 0)
			{
				psp = (PSYSTEM_PROCESS_INFORMATION)pBuffer;
				do {

					p_info info{ 0 };
					 
					info.pid = (DWORD)psp->UniqueProcessId;
					//转换成多字节
					sprintf(info.name, "%S", psp->ImageName.Buffer);
					info.sid = psp->SessionId;
					info.stime=trans_time(psp->CreateTime);
					info.ppid = get_parent_pid(info.pid);

					//询问信息只能用驱动
					query_process_info(&info);


					//printf("fullPath->%ws\r\n", info.fpath);
					info_list.append(info);
				
					//下一个
					psp = (PSYSTEM_PROCESS_INFORMATION)((UINT_PTR)psp + psp->NextEntryOffset);
				} while (psp->NextEntryOffset != 0);
			}
			delete[] pBuffer;


			pBuffer = NULL;


		}

```

第一次判断是否是`STATUS_INFO_LENGTH_MISMATCH`,这样说明缓冲区太小了，接着再次询问一次;

其中询问到的结构是一个`_SYSTEM_PROCESS_INFORMATION`的不定数组,结构如下

```C++
	typedef struct _SYSTEM_PROCESS_INFORMATION {
		ULONG NextEntryOffset;
		ULONG NumberOfThreads;
		LARGE_INTEGER SpareLi1;
		LARGE_INTEGER SpareLi2;
		LARGE_INTEGER SpareLi3;
		LARGE_INTEGER CreateTime;
		LARGE_INTEGER UserTime;
		LARGE_INTEGER KernelTime;
		UNICODE_STRING ImageName;
		KPRIORITY BasePriority;
		HANDLE UniqueProcessId;
		HANDLE InheritedFromUniqueProcessId;
		ULONG HandleCount;
		ULONG SessionId;
		ULONG_PTR PageDirectoryBase;
		SIZE_T PeakVirtualSize;
		SIZE_T VirtualSize;
		ULONG PageFaultCount;
		SIZE_T PeakWorkingSetSize;
		SIZE_T WorkingSetSize;
		SIZE_T QuotaPeakPagedPoolUsage;
		SIZE_T QuotaPagedPoolUsage;
		SIZE_T QuotaPeakNonPagedPoolUsage;
		SIZE_T QuotaNonPagedPoolUsage;
		SIZE_T PagefileUsage;
		SIZE_T PeakPagefileUsage;
		SIZE_T PrivatePageCount;
		LARGE_INTEGER ReadOperationCount;
		LARGE_INTEGER WriteOperationCount;
		LARGE_INTEGER OtherOperationCount;
		LARGE_INTEGER ReadTransferCount;
		LARGE_INTEGER WriteTransferCount;
		LARGE_INTEGER OtherTransferCount;
	} SYSTEM_PROCESS_INFORMATION, * PSYSTEM_PROCESS_INFORMATION;
```

之所以说是不定数组,因为他找到数组的下一个元素方法是`psp = (PSYSTEM_PROCESS_INFORMATION)((UINT_PTR)psp + psp->NextEntryOffset);`之所以会这样,是因为每个`PSYSTEM_PROCESS_INFORMATION`结构后面跟随的是如下这个结构

```C++
typedef struct _SYSTEM_THREADS
	{
		LARGE_INTEGER KernelTime;
		LARGE_INTEGER UserTime;
		LARGE_INTEGER CreateTime;
		ULONG         WaitTime;
		PVOID         StartAddress;
		CLIENT_ID     ClientId;
		KPRIORITY     Priority;
		KPRIORITY     BasePriority;
		ULONG         ContextSwitchCount;
		THREAD_STATE  State;
		KWAIT_REASON  WaitReason;
	}SYSTEM_THREADS, * PSYSTEM_THREADS;
```

包含了所有的线程信息,因为枚举进程的所有线程就是通过这个方法;

此外,进程的信息远不止上面的那些,因此还需要额外获取,首先就是获取进程的公司名

```C++
auto get_file_companyname(const char* full_path) -> char*
	{

		DWORD dwDummy;
		char* pCompanyName;
		DWORD dwSize = GetFileVersionInfoSizeA(full_path, &dwDummy);
		UINT cbTranslate;

		if (dwSize == 0) {
			return nullptr;
		}



		BYTE* pVersionInfo = new BYTE[dwSize];
		if (!GetFileVersionInfoA(full_path, 0, dwSize, pVersionInfo)) {
			delete[] pVersionInfo;
			return nullptr;
		}


		struct LANGANDCODEPAGE {
			WORD wLanguage;
			WORD wCodePage;
		} *lpTranslate;

		// Read the list of languages and code pages.
		VerQueryValueA(pVersionInfo,
			("\\VarFileInfo\\Translation"),
			(LPVOID*)&lpTranslate,
			&cbTranslate);

		//获取页码 只有这样才能读到
		for (int i = 0; i < (cbTranslate / sizeof(struct LANGANDCODEPAGE)); i++)
		{
			char SubBlock[50] = { 0 };
			sprintf(SubBlock, "\\StringFileInfo\\%04x%04x\\CompanyName", lpTranslate[i].wLanguage,
				lpTranslate[i].wCodePage);
			UINT uLen;

			// Retrieve file description for language and code page "i". 
			if (VerQueryValueA(pVersionInfo,
				SubBlock,
				(LPVOID*)&pCompanyName,
				&uLen)) {

				return pCompanyName;
			}


		}

		//没成功 没读到 有可能没有公司名称
		return nullptr;
	}
```

获取公司名需要这个进程的全路径**,因此首先需要`GetModuleNameEx`这个函数对应的句柄权限,由于害怕卡句柄权限,我选择在在R0获取，获取PEB,然后查进程路径,返回R3,这样就不怕句柄权限受阻;**

在获取到进程的全路径,只需要按照上面的代码进行询问就行了,步骤就是

- `GetFileVersionInfoA`获取这个文件的信息
- `VerQueryValueA`获取页码,这个是很重要的一步,在询问文件信息,有点像`路径`,这个就像是密码,通向下一级的密码,比如`"\\StringFileInfo\\%04x%04x\\CompanyName"`这个字符串，从StringFileInfo->CompanyName,中间这个代码就是靠`VerQueryValueA`询问出来的
- 事实上,上面的这个路径除了页码之外其他都是固定的,因此需要询问；
- `VerQueryValueA`继续,询问正确的;

总的来说,R3这个部分主要用于获取进程的所有PID,而众所周知,有些进程的PID是被句柄权限卡死的.因此想要在R3用`ZwQueryInfomationThread/Process`来询问具体进程的信息是不可行的.因此询问进程信息我才用的是PID传入内核,内核来询问,这就是R0部分

#### R0部分

代码如下

```C++
auto query_process_info(pp_info info,PEPROCESS process)->bool {
		KAPC_STATE apc{ 0 };
		
		KeStackAttachProcess(process, &apc);

		__try {

			auto peb=PsGetProcessPeb(process);
		

			if (MmIsAddressValid(peb)) {
				
				//开始复制
				memcpy(info->cmdline, peb->ProcessParameters->CommandLine.Buffer, MAX_PATH);
				memcpy(info->fpath, peb->ProcessParameters->ImagePathName.Buffer, MAX_PATH);
				auto uAccess = user_cannot_access(process);
				info->uaccess = !uAccess;
			}
		}
		__except (1) {

			KeUnstackDetachProcess(&apc);
			return false;
		}

		KeUnstackDetachProcess(&apc);
		return true;
	}
```

实际上很简单,只需要从 Peb找出cmdLine和`fullPath`全路径,有了全路径,就可以获取改进程的图标,

而至于用户是否可访问,我才用了如下的方式进行判断;

````C++
auto user_cannot_access(PEPROCESS process) -> bool {
		
		UNICODE_STRING funcName{ 0 };
		RtlInitUnicodeString(&funcName, L"PsIsProtectedProcess");
		bool nonAccess = false;

		bool (*func)(PEPROCESS) = (bool(*)(PEPROCESS))MmGetSystemRoutineAddress(&funcName);
		//先用PsIsProtectedProcess判断
		if (func !=nullptr) {

			nonAccess = func(process);

		}

		auto header = (POBJECT_HEADER)((UINT64)process - 0x30);
		//用ObjectHeader->KernelOnlyAccess
		nonAccess |= header->KernelOnlyAccess;
		return nonAccess;

	}
````

首先就算通过一个导出函数`PsIsProtectedProcess`来判断是不是保护进程,事实上,这个函数很简单.只是判断了EPROCESS的一个位

```assembly
; _BOOL8 __fastcall PsIsProtectedProcess(__int64)
test    byte ptr [rcx+87Ah], 7
mov     eax, 0
setnbe  al
retn
```

而在EPROCESS+87A 7的位置是

```C++
struct _PS_PROTECTION Protection;
//0x1 bytes (sizeof)
struct _PS_PROTECTION
{
    union
    {
        UCHAR Level;                                                        //0x0
        struct
        {
            UCHAR Type:3;                                                   //0x0
            UCHAR Audit:1;                                                  //0x0
            UCHAR Signer:4;                                                 //0x0
        };
    };
}; 
```

此外,再通过判断改EPROCESS的ObjectHeader里面的`KernelOnlyAccess`

```C++
		auto header = (POBJECT_HEADER)((UINT64)process - 0x30);
		//用ObjectHeader->KernelOnlyAccess
		nonAccess |= header->KernelOnlyAccess;
```

这个也代表着用户不可访问;

### 枚举模块

这个功能是纯R0实现,思路就是遍历PEB的Ldr这个链表;

Ldr位于PEB的0x18位置

```C++
struct _PEB
{
    UCHAR InheritedAddressSpace;                                            //0x0
    UCHAR ReadImageFileExecOptions;                                         //0x1
    UCHAR BeingDebugged;                                                    //0x2
    union
    {
        UCHAR BitField;                                                     //0x3
        struct
        {
            UCHAR ImageUsesLargePages:1;                                    //0x3
            UCHAR IsProtectedProcess:1;                                     //0x3
            UCHAR IsImageDynamicallyRelocated:1;                            //0x3
            UCHAR SkipPatchingUser32Forwarders:1;                           //0x3
            UCHAR IsPackagedProcess:1;                                      //0x3
            UCHAR IsAppContainer:1;                                         //0x3
            UCHAR IsProtectedProcessLight:1;                                //0x3
            UCHAR IsLongPathAwareProcess:1;                                 //0x3
        };
    };
    UCHAR Padding0[4];                                                      //0x4
    VOID* Mutant;                                                           //0x8
    VOID* ImageBaseAddress;                                                 //0x10
    struct _PEB_LDR_DATA* Ldr;                                              //0x18
    XXX;
}
```

Ldr是一个这样的结构

```C++
//0x58 bytes (sizeof)
struct _PEB_LDR_DATA
{
    ULONG Length;                                                           //0x0
    UCHAR Initialized;                                                      //0x4
    VOID* SsHandle;                                                         //0x8
    struct _LIST_ENTRY InLoadOrderModuleList;                               //0x10
    struct _LIST_ENTRY InMemoryOrderModuleList;                             //0x20
    struct _LIST_ENTRY InInitializationOrderModuleList;                     //0x30
    VOID* EntryInProgress;                                                  //0x40
    UCHAR ShutdownInProgress;                                               //0x48
    VOID* ShutdownThreadId;                                                 //0x50
}; 
```

其中中间的那三个链表都是链接着一个`_LDR_DATA_TABLE_ENTRY`的结构,里面记录了模块的信息

```C++
//0x120 bytes (sizeof)
struct _LDR_DATA_TABLE_ENTRY
{
    struct _LIST_ENTRY InLoadOrderLinks;                                    //0x0
    struct _LIST_ENTRY InMemoryOrderLinks;                                  //0x10
    struct _LIST_ENTRY InInitializationOrderLinks;                          //0x20
    VOID* DllBase;                                                          //0x30
    VOID* EntryPoint;                                                       //0x38
    ULONG SizeOfImage;                                                      //0x40
    struct _UNICODE_STRING FullDllName;                                     //0x48
    struct _UNICODE_STRING BaseDllName;                                     //0x58
    union
    {
        UCHAR FlagGroup[4];                                                 //0x68
        ULONG Flags;                                                        //0x68
        struct
        {
            ULONG PackagedBinary:1;                                         //0x68
            ULONG MarkedForRemoval:1;                                       //0x68
            ULONG ImageDll:1;                                               //0x68
            ULONG LoadNotificationsSent:1;                                  //0x68
            ULONG TelemetryEntryProcessed:1;                                //0x68
            ULONG ProcessStaticImport:1;                                    //0x68
            ULONG InLegacyLists:1;                                          //0x68
            ULONG InIndexes:1;                                              //0x68
            ULONG ShimDll:1;                                                //0x68
            ULONG InExceptionTable:1;                                       //0x68
            ULONG ReservedFlags1:2;                                         //0x68
            ULONG LoadInProgress:1;                                         //0x68
            ULONG LoadConfigProcessed:1;                                    //0x68
            ULONG EntryProcessed:1;                                         //0x68
            ULONG ProtectDelayLoad:1;                                       //0x68
            ULONG ReservedFlags3:2;                                         //0x68
            ULONG DontCallForThreads:1;                                     //0x68
            ULONG ProcessAttachCalled:1;                                    //0x68
            ULONG ProcessAttachFailed:1;                                    //0x68
            ULONG CorDeferredValidate:1;                                    //0x68
            ULONG CorImage:1;                                               //0x68
            ULONG DontRelocate:1;                                           //0x68
            ULONG CorILOnly:1;                                              //0x68
            ULONG ChpeImage:1;                                              //0x68
            ULONG ReservedFlags5:2;                                         //0x68
            ULONG Redirected:1;                                             //0x68
            ULONG ReservedFlags6:2;                                         //0x68
            ULONG CompatDatabaseProcessed:1;                                //0x68
        };
    };
    USHORT ObsoleteLoadCount;                                               //0x6c
    USHORT TlsIndex;                                                        //0x6e
    struct _LIST_ENTRY HashLinks;                                           //0x70
    ULONG TimeDateStamp;                                                    //0x80
    struct _ACTIVATION_CONTEXT* EntryPointActivationContext;                //0x88
    VOID* Lock;                                                             //0x90
    struct _LDR_DDAG_NODE* DdagNode;                                        //0x98
    struct _LIST_ENTRY NodeModuleLink;                                      //0xa0
    struct _LDRP_LOAD_CONTEXT* LoadContext;                                 //0xb0
    VOID* ParentDllBase;                                                    //0xb8
    VOID* SwitchBackContext;                                                //0xc0
    struct _RTL_BALANCED_NODE BaseAddressIndexNode;                         //0xc8
    struct _RTL_BALANCED_NODE MappingInfoIndexNode;                         //0xe0
    ULONGLONG OriginalBase;                                                 //0xf8
    union _LARGE_INTEGER LoadTime;                                          //0x100
    ULONG BaseNameHashValue;                                                //0x108
    enum _LDR_DLL_LOAD_REASON LoadReason;                                   //0x10c
    ULONG ImplicitPathOptions;                                              //0x110
    ULONG ReferenceCount;                                                   //0x114
    ULONG DependentLoadFlags;                                               //0x118
    UCHAR SigningLevel;                                                     //0x11c
}; 
```

因此遍历这个链表,就可以得到该进程的所有模块了;

### 枚举句柄表

我们知道,Win的私有句柄表是一个3级指针;

```C++
struct _HANDLE_TABLE* ObjectTable;                                      //0x570
//0x80 bytes (sizeof)
struct _HANDLE_TABLE
{
    ULONG NextHandleNeedingPool;                                            //0x0
    LONG ExtraInfoPages;                                                    //0x4
    volatile ULONGLONG TableCode;                                           //0x8
    struct _EPROCESS* QuotaProcess;                                         //0x10
    struct _LIST_ENTRY HandleTableList;                                     //0x18
    ULONG UniqueProcessId;                                                  //0x28
    union
    {
        ULONG Flags;                                                        //0x2c
        struct
        {
            UCHAR StrictFIFO:1;                                             //0x2c
            UCHAR EnableHandleExceptions:1;                                 //0x2c
            UCHAR Rundown:1;                                                //0x2c
            UCHAR Duplicated:1;                                             //0x2c
            UCHAR RaiseUMExceptionOnInvalidHandleClose:1;                   //0x2c
        };
    };
    struct _EX_PUSH_LOCK HandleContentionEvent;                             //0x30
    struct _EX_PUSH_LOCK HandleTableLock;                                   //0x38
    union
    {
        struct _HANDLE_TABLE_FREE_LIST FreeLists[1];                        //0x40
        struct
        {
            UCHAR ActualEntry[32];                                          //0x40
            struct _HANDLE_TRACE_DEBUG_INFO* DebugInfo;                     //0x60
        };
    };
}; 
```

其中TableCode的最后是0,1,2分别代表着1 2 3级指针,指向

```C++
//0x10 bytes (sizeof)
union _HANDLE_TABLE_ENTRY
{
    volatile LONGLONG VolatileLowValue;                                     //0x0
    LONGLONG LowValue;                                                      //0x0
    struct
    {
        struct _HANDLE_TABLE_ENTRY_INFO* volatile InfoTable;                //0x0
    LONGLONG HighValue;                                                     //0x8
    union _HANDLE_TABLE_ENTRY* NextFreeHandleEntry;                         //0x8
        struct _EXHANDLE LeafHandleValue;                                   //0x8
    };
    LONGLONG RefCountField;                                                 //0x0
    ULONGLONG Unlocked:1;                                                   //0x0
    ULONGLONG RefCnt:16;                                                    //0x0
    ULONGLONG Attributes:3;                                                 //0x0
    struct
    {
        ULONGLONG ObjectPointerBits:44;                                     //0x0
    ULONG GrantedAccessBits:25;                                             //0x8
    ULONG NoRightsUpgrade:1;                                                //0x8
        ULONG Spare1:6;                                                     //0x8
    };
    ULONG Spare2;                                                           //0xc
}; 

```

实际上,找到这个三级指针的方法很简单,因为Handle事实上是index*4,所以先要&3ff

```C++
if (level == 1) {

			return (PHANDLE_TABLE_ENTRY)(*(uint64_t*)(table_code - 1 + 8 * (u_handle >> 10)) + 4 * (u_handle & 0X3FF));
		}
		else if (level == 2) {

			return (PHANDLE_TABLE_ENTRY)(*(uint64_t*)(*(uint64_t*)(table_code - 2 + 8 * (u_handle >> 19)) + 8 * (u_handle >> 10 & 0X1FF)) + 4 * (u_handle & 0x3ff));

		}
		else {

			return (PHANDLE_TABLE_ENTRY)(table_code + 4 * (u_handle & 0x3ff));

		}
```

这样找到`HANDLE_TABLE_ENTRY`之后，然后找到各种信息;

这种方法可行但是麻烦,事实上windows提供了更好的方法;

首先,询问系统的所有句柄信息,然后判断是不是要比较的进程,如果是进行下一步操作;

```C++
auto status = ZwQuerySystemInformation(SystemHandleInformation, tmp, PAGE_SIZE, &needSize);
		ExFreePool(tmp);


		if (status != STATUS_INFO_LENGTH_MISMATCH) {
			return 0;
		}

		//多分配点
		auto buf = (PSYSTEM_HANDLE_INFORMATION)ExAllocatePoolWithTag(PagedPool, needSize+PAGE_SIZE, 'tmp');

		status = ZwQuerySystemInformation(SystemHandleInformation, buf, needSize + PAGE_SIZE, &needSize);
		if (!NT_SUCCESS(status)) {
			ExFreePool(buf);
			return 0;
		}

		//开始bianli 
		for (auto i = 0ul; i < buf->NumberOfHandles; i++) {

			auto item = buf->Handles[i];

			if (pid == (HANDLE)item.pid) {
                //do something
            }
        }
}
```

上面的结构如下

```C++
	typedef struct _SYSTEM_HANDLE_TABLE_ENTRY_INFO {
		ULONG pid;
		UCHAR ObjectTypeIndex;
		UCHAR HandleAttributes;
		USHORT HandleValue;
		PVOID Object;
		ULONG GrantedAccess;
	} SYSTEM_HANDLE_TABLE_ENTRY_INFO, * PSYSTEM_HANDLE_TABLE_ENTRY_INFO;

	typedef struct _SYSTEM_HANDLE_INFORMATION {
		ULONG NumberOfHandles;
		SYSTEM_HANDLE_TABLE_ENTRY_INFO Handles[1];
	} SYSTEM_HANDLE_INFORMATION, * PSYSTEM_HANDLE_INFORMATION;
```

是一个边长数组,而仅通过这些信息,远远不够,因此还需要`ZwQueryObject`来进行询问;

```C++
if ((HANDLE)item.pid == pid) {

				PsLookupProcessByProcessId((HANDLE)item.pid, &process);
				KeStackAttachProcess(process, &apc);

				OBJECT_BASIC_INFORMATION obi{ 0 };
				OBJECT_NAME_INFORMATION* oni{ 0 };
				OBJECT_TYPE_INFORMATION* oti{ 0 };
				OBJECT_HANDLE_FLAG_INFORMATION ohf{ 0 };

				status = ZwQueryObject((HANDLE)item.HandleValue, (OBJECT_INFORMATION_CLASS)0,
					&obi, sizeof obi, 0);


				oni = (OBJECT_NAME_INFORMATION*)ExAllocatePoolWithTag(PagedPool, sizeof OBJECT_NAME_INFORMATION, 'tmp');
				memset(oni, 0, sizeof OBJECT_NAME_INFORMATION);
				status = ZwQueryObject(((HANDLE)item.HandleValue), (OBJECT_INFORMATION_CLASS)undoc::ObjectNameInformation
					, oni, sizeof oni, &needSize
				);
				if (oni == nullptr) { KeUnstackDetachProcess(&apc); return false; }
				if (status == STATUS_INFO_LENGTH_MISMATCH) {
					ExFreePool(oni);
					oni = nullptr;
					oni = (OBJECT_NAME_INFORMATION*)ExAllocatePoolWithTag(PagedPool, needSize, 'tmp');
					if (oni == nullptr) { KeUnstackDetachProcess(&apc); return false; }
					status = ZwQueryObject(((HANDLE)item.HandleValue), (OBJECT_INFORMATION_CLASS)undoc::ObjectNameInformation
						, oni, needSize, &needSize
					);
				}


				oti = (OBJECT_TYPE_INFORMATION*)ExAllocatePoolWithTag(PagedPool, sizeof OBJECT_TYPE_INFORMATION, 'tmp');
				if (oti == nullptr) { KeUnstackDetachProcess(&apc); return false; }

				status = ZwQueryObject(((HANDLE)item.HandleValue), (OBJECT_INFORMATION_CLASS)undoc::ObjectTypeInformation
					, oti, sizeof OBJECT_TYPE_INFORMATION, &needSize
				);

				if (status == STATUS_INFO_LENGTH_MISMATCH) {
					ExFreePool(oti);
					oti = nullptr;
					oti = (OBJECT_TYPE_INFORMATION*)ExAllocatePoolWithTag(PagedPool, needSize, 'tmp');
					if (oti == nullptr) { KeUnstackDetachProcess(&apc); return false; }


					memset(oti, 0, needSize);
					status = ZwQueryObject(((HANDLE)item.HandleValue), (OBJECT_INFORMATION_CLASS)undoc::ObjectTypeInformation
						, oti, needSize, &needSize
					);

				}

				status = ZwQueryObject(((HANDLE)item.HandleValue), (OBJECT_INFORMATION_CLASS)undoc::ObjectHandleFlagInformation
					, &ohf, sizeof ohf, 0
				);

				
				KeUnstackDetachProcess(&apc);

				//开始设置 一定要取消挂靠 不然蓝屏
				auto cur_index = infos->count;
				infos->infos[cur_index] = { .access = item.GrantedAccess,
				.handleType = 0,.handleName = 0,.handle = (HANDLE)item.HandleValue,
				.handleObject = (UINT_PTR)item.Object,.ptrRef = obi.PointerCount,
				.handleRef = obi.HandleCount,.closeProtect = ohf.ProtectFromClose
				};

				//开始复制TypeName和Name
				_Utils::_wtochar(infos->infos[cur_index].handleName, oni->Name.Buffer);
				_Utils::_wtochar(infos->infos[cur_index].handleType, oti->TypeName.Buffer);
				infos->count++;
				ExFreePool(oti);
				ExFreePool(oni);
			}
```

ZwQueryObject可以询问这个句柄的各种信息,包括名字 权限,类型名字 是否可以关闭等等...

### 枚举进程窗口

这个功能完全通过R3来实现;用到了内核/R3 未文档化但是导出的函数,通过逆向EnumProcessWindow就可以知道

```C++
auto enum_windows(HANDLE pid) -> pwindows_info {

		auto hWin32u = LoadLibrary(_T("win32u.dll"));

		std::function<NTSTATUS(HDESK, HWND, BOOL, BOOL, DWORD, UINT, HWND*, PUINT)> enumWnd =
			(NtUserBuildHwndList_t)GetProcAddress(hWin32u, "NtUserBuildHwndList");

		auto hwndArry = new HWND[10];
		UINT needCount = 0;
		auto status = enumWnd(0, 0, true, 0, 0, 10, hwndArry, &needCount);
		

		if (status != 0xC0000023) {

			return nullptr;

		}

		delete[] hwndArry;

		//多申请点
		hwndArry = new HWND[needCount + 100];
		status = enumWnd(0, 0, true, 0, 0, needCount + 100, hwndArry, &needCount);
		if (!NT_SUCCESS(status)) {

			return nullptr;

		}
		auto winInfos = new windows_info_t;
		winInfos->count = 0;
		winInfos->infos = new window_info_t[needCount];
		memset(winInfos->infos, 0, needCount * sizeof window_info_t);


		for (int i = 0; i < needCount; i++) {
			auto _tid = 0ull, _pid = 0ull;
			_tid = GetWindowThreadProcessId(hwndArry[i], (PDWORD) &_pid);
			if (pid == (HANDLE)_pid) {

	 			winInfos->infos[winInfos->count].hwnd = hwndArry[i];
				winInfos->infos[winInfos->count].pid = pid;
				winInfos->infos[winInfos->count].tid = (HANDLE)_tid;
				GetWindowTextA(hwndArry[i], winInfos->infos[winInfos->count].titile, MAX_PATH);
				winInfos->infos[winInfos->count].isVisible=IsWindowVisible(hwndArry[i]);

				winInfos->count++;
			}
		
		}


		delete[] hwndArry;

		

		return winInfos;

	}
```

通过找`win32u.dll`的`NtUserBuildHwndList`来进行枚举,这个函数的参数如下

```C++
typedef NTSTATUS(WINAPI* NtUserBuildHwndList_t)(
		HDESK hdesk,
		HWND hwndNext,
		BOOL fEnumChildren,
		BOOL RemoveImmersive,//移除沉浸式窗口
		DWORD idThread,
		UINT cHwndMax,
		HWND* phwndFirst,
		PUINT pcHwndNeeded);
```

值得一提的是`GetWindowThreadProcessId`这个函数,它可以根据HWND来获取进程,但是其实还是可以获取这个窗口的线程的,这个函数返回的是就是线程ID;

### 枚举定时器

这个是整个进程模块最耗时的时间,因为涉及到逆向的过程;

这里的定时器是Windows进程所有的定时器,并不是DPC的定时器;

逆向过程是首先找到user32.dll的`SetTimer`这个函数;

```C++
UINT_PTR __stdcall SetTimer(HWND hWnd, UINT_PTR nIDEvent, UINT uElapse, TIMERPROC lpTimerFunc)
{
  return NtUserSetTimer(hWnd, nIDEvent, uElapse, lpTimerFunc, 0);
}
```

可以看出,他调用了`win32u.dll`的导出函数`NtUserSetTimer`,注意,这个系统版本是19044,在低版本win10甚至更低版本,是没有`win32u.dll`的,是直接通过`user32.dll`syscall进入内核;

这样,直接去`win32kfull.sys`查看这个内核函数,IDA F5

```C++
UINT_PTR __fastcall NtUserSetTimer(HWND hWnd, UINT_PTR id, unsigned int elapse, UINT_PTR proc, unsigned int unk_flags)
{
  UINT_PTR timerId; // rbx
  __int64 v10; // rax
  __int64 CurrentThreadWin32Thread; // rax
  UINT64 window_instance; // rbp
  unsigned int _elapse; // edi
  unsigned int _unk_flags; // esi
  __int64 CurrentProcessWin32Process; // rax
  __int64 CurrentProcessWin32Process_1; // r8
  __int64 v17; // rax
  __int64 errcode; // rcx

  EnterCrit(0i64, 0i64);
  timerId = 0i64;
  if ( !*(_QWORD *)(SGDGetUserSessionState() + 8)
    || (v10 = SGDGetUserSessionState(), !ExIsResourceAcquiredSharedLite(*(PERESOURCE *)(v10 + 8))) )
  {
    if ( (gdwExtraInstrumentations & 1) != 0 )
      KeBugCheckEx(0x164u, 0x2Aui64, 0i64, 0i64, 0i64);
    DbgkWerCaptureLiveKernelDump(aNtuser, 400i64, 42i64, 0i64, 0i64, 0i64, 0i64, 0i64, 0);
  }
  CurrentThreadWin32Thread = PsGetCurrentThreadWin32Thread();
  ++*(_DWORD *)(CurrentThreadWin32Thread + 48);
  if ( !hWnd )
  {
    window_instance = 0i64;
hwnd_valid:
    _elapse = 10;
    if ( elapse >= 10 )                         // 如果间隔小于10ms,那就赋值10,因为时钟中断
      _elapse = elapse;
    _unk_flags = unk_flags;
    if ( _elapse > 0x7FFFFFFF )
      _elapse = 0x7FFFFFFF;
    if ( unk_flags == 0x7FFFFFF5 )              // 正常调用是0
    {
      _unk_flags = 0x7FFFFFFF - _elapse;
    }
    else if ( unk_flags != -1 && (_elapse + unk_flags < _elapse || _elapse + unk_flags > 0x7FFFFFFF) )
    {
      errcode = 87i64;
      goto error;
    }
    if ( !window_instance )
      goto driectly_set;                        // hwnd是一 直接设置
    CurrentProcessWin32Process = PsGetCurrentProcessWin32Process(0x7FFFFFFFi64);
    CurrentProcessWin32Process_1 = CurrentProcessWin32Process;
    if ( CurrentProcessWin32Process )
      CurrentProcessWin32Process_1 = -(__int64)(*(_QWORD *)CurrentProcessWin32Process != 0i64) & CurrentProcessWin32Process;
    if ( CurrentProcessWin32Process_1 == *(_QWORD *)(*(_QWORD *)(window_instance + 0x10) + 0x1A0i64) )// 不能跨进程设置  tagWND* spwndParent;
    {
driectly_set:
      timerId = InternalSetTimer((void *)window_instance, id, _elapse, (void *)proc, _unk_flags, 0);
      goto LABEL_18;
    }
    errcode = 5i64;
error:
    UserSetLastError(errcode);
    goto LABEL_18;
  }
  window_instance = ValidateHwnd(hWnd);         // 把hwnd转换成指针 tagWND
  if ( window_instance )
    goto hwnd_valid;
LABEL_18:
  v17 = PsGetCurrentThreadWin32Thread();
  --*(_DWORD *)(v17 + 48);
  UserSessionSwitchLeaveCrit();
  return timerId;
}
```

可以看到,如果HWND有效,最终会转换成`PWND`这个结构,而如果HWND是NULL,那么则是插入全局的Timer,不针对窗口,那么接下来看`InternalSetTimer`函数

事实上,这个函数中出现了两个链表,之前猜测定时器可能放置在链表中;

其中一个链表是` timer = (UINT64 *)((char *)&gTimerHashTable + 0x10 * ((BYTE1(pwnd) + (unsigned __int8)nIDEvent) & 0x3F));// Timer基于窗口?`

也就是`gTimerHashTable`,这个链表如其名字,实际上他就是一个哈希表,哈希函数是

`hash(x,y)=0x10*( ()(Byte)(pwnd>>8)+(Byte)nIdEvent)) & 0x3F)`,不难看出,&0x3F 说明了这个哈希表大小是64个链表,有点像哈希桶的感觉,而0x10就是LIST_ENTRY的大小,

事实上,Windows就算通过这个根据PWND+ID(时钟的)+Hash函数快速定位到对应的链表上,`FindTimer`这个函数就可以看到上述代码;

上述的hash表挂在`PTIMER+0x70`的位置

```C++
v30 = (char *)(newAllocTimer + 0x70),
                v31 = (char *)&gTimerHashTable
                    + 16 * (((unsigned __int8)v14 + (unsigned __int8)*(_DWORD *)(newAllocTimer + 96)) & 0x3F),
                v32 = (char **)*((_QWORD *)v31 + 1),
```

而另一个表叫做`gtmrListHead`则较为暴力,就是一个双向链表,连着所有PTIMER结构,挂在`0x48`的位置

```C++
  *(_QWORD *)(newAllocTimer + 96) = nIDEvent;
          if ( *(_QWORD *)(gtmrListHead + 8i64) != gtmrListHead
            || (*(_QWORD *)(newAllocTimer + 0x50) = gtmrListHead,
                *v29 = gtmrListHead,
                *(_QWORD *)(gtmrListHead + 8i64) = v29,
                gtmrListHead = newAllocTimer + 0x48,
```

接着看下面的代码

```C++
HMAssignmentLock(v39, 0i64);
  *(_DWORD *)(header + 0x28) = dwElapse_1_1;
  *(_DWORD *)(header + 0x34) = dwElapse_1_1;
  *(_QWORD *)(header + 0x20) = pTimerFunc;
  *(_QWORD *)(header + 0x68) = 0i64;
  if ( (v13 & 0x200) != 0 )
    *(_DWORD *)(header + 44) = flags;
  *(_DWORD *)(header + 0x80) = (MEMORY[0xFFFFF78000000320] * (unsigned __int64)MEMORY[0xFFFFF78000000004]) >> 24;
  if ( (v13 & 0x80u) != 0 )
  {
    v13 &= ~0x80u;
  }
  else if ( (v13 & 0x100) != 0 )
  {
    *(_QWORD *)(header + 0x68) = thread;
  }
  *(_DWORD *)(header + 48) = v13 | 8;
  *(_QWORD *)(header + 24) = threadInfo;
```

由此可以推断出,PTIMER的结构应该是下面的结构

```C++
typedef struct timer_t{

		HEAD head;//0
		void* pfn;//20
		DWORD32 elapse;//28
		DWORD32 flags;//2c
		DWORD32 unkFlags;//30
		DWORD32 elapse1;//34
		char padding[0x10];//38
		LIST_ENTRY list1;//链接的是gtmrListHead //48
		void* spwnd;//58
		UINT64 id;//60
		void* threadObject;//68
		LIST_ENTRY list2;//Hash链接gTimerHashTable

	}*ptimer_t;
```

其中HEAD是我参考xp源码得到的,结构如下

```C++
	typedef struct _HEAD{
	void* unk[3];
	void* threadInfo;
	}HEAD;
```

这个ThreadInfo结构是这样

```C++
typedef struct threadInfo{
    WIN32THREAD w32thread;
    UNK;
}
```

`WIN32THREAD`的第一个成员就算改定时器所属的线程的`EPROCESS`

从`HEAD`到`threadInfo`均参考XP源码,经过验证发现没有问题;

而至于如何遍历进程的所有定时器呢?遍历hash链表实际上是画蛇添足了;因为根本不知道Timer的ID,所以0-63试还是很慢,不如直接遍历`gtmrListHead`来;

接下来,就是获取`gtmrListHead`,这里比较方便的是,无论是`gtmrListHead`还是`gTimerHashTable`都是`win32kfull.sys`从`win32kbase.sys`导出的,这个就比较方便；直接写一个遍历内核模块导出表的进行判断即可;

```C++
auto find_module_export(void* base,const char* name) -> void* {
		
		//为了确保查得到,需要进行附加explorer.exe 才能有地址空间 比如win32kxx.sys
		//要求调用这个函数的人必须是GUI线程 否则无法查找
		
		//explorer不一定能获取到?
		if (!MmIsAddressValid(base) || name == nullptr) return nullptr;
		__try {

			if (*((unsigned short*)base) != 0x5A4D) {
				
				return 0;//不是有效的PE文件
			}

			auto dosHeaders = (PIMAGE_DOS_HEADER)base;

			auto ntHeaders = (PIMAGE_NT_HEADERS)(dosHeaders->e_lfanew + (UINT_PTR)base);

			auto exportDirectory = (PIMAGE_EXPORT_DIRECTORY)((PUCHAR)base + ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress);

			//nameTable存的是函数名的RVA
			auto nameTable = (PULONG)(exportDirectory->AddressOfNames + (PUCHAR)base);
			//索引到funcTable索引转换需要这个
			auto ordinalTable = (PSHORT)(exportDirectory->AddressOfNameOrdinals + (PUCHAR)base);
			auto funcTable = (PULONG)(exportDirectory->AddressOfFunctions + (PUCHAR)base);

			for (auto i = 0ul; i < exportDirectory->NumberOfNames; i++) {

				if (strcmp((char*)base + nameTable[i], name) == 0) {

					//find
					auto index = ordinalTable[i];
					auto ret = (UINT_PTR)base + funcTable[index];
					return (void*)ret;

				}

			}

			return 0;

		}
		__except (1) {
			
			return nullptr;
		}
		

		

	}
```

所以通过遍历进程的所有线程来得到PEPROCESS,再通过`threadInfo`来判断是不是该进程所属的定时器,如果是,那么就添加;

```C++
//给入一个链表头 插入这里面 插入改进程的所有定时器
	auto query_process_timer(__inout pfind_list_t head,HANDLE pid) -> void {
	
		//先询问进程的所有线程
		auto tidArry = kprocess::query_threads_tid(pid);
		if (tidArry == nullptr) return;

		
		for (int i=0;tidArry[i];i++) {

			query_timer_count((PLIST_ENTRY)head, tidArry[i]);

		}

		ExFreePool(tidArry);

		return;

	}
```

```C++
auto query_timer_count(PLIST_ENTRY head,HANDLE tid) -> unsigned int {

		unsigned int count = 0;
		PETHREAD thread{ 0 };
		auto status = PsLookupThreadByThreadId(tid, &thread);
		if (!NT_SUCCESS(status)) {
			return 0;
		}
		ObDereferenceObject(thread);


		auto volatile gtmrListHead = (PLIST_ENTRY)
			_Utils::find_module_export(_Utils::find_module_base("win32kbase.sys"),
				"gtmrListHead"
			);
		if (gtmrListHead == nullptr) return 0;//这个不是hash链表

		for (auto entry = gtmrListHead->Flink;
				entry != gtmrListHead; entry = entry->Flink) {

				auto item = CONTAINING_RECORD(entry, timer_t, list1);
				//这个地方疑似不能解引用 有时候PageFault
				//注意 这里的定时器有可能属于hwnd==0的 因此最好判断threadInfo
				if ((*(PETHREAD*)(item->head.threadInfo)) == thread
					) {

					if (find(head, item)) {
						continue;
					}
					else {
						auto _item = (pfind_list_t)ExAllocatePoolWithTag(PagedPool, sizeof find_list_t,
							'list');
						if (item == nullptr) {

							ONLY_DEBUG_BREAK;
						}
						_item->timer = item;
						InsertHeadList(head, (PLIST_ENTRY)(_item));
						count++;
					}

				}

		}

		return count;

	}
```

`(*(PETHREAD*)(item->head.threadInfo)`来判断线程,如此遍历;即可遍历到进程的所有定时器;

### 注入DLL

这里为了兼容,ARK的注入DLL我才用的是内核开辟线程+`LdrLoadDll`的方式进行

这里为了方便,我使用了在内核层获取某dll的导出函数;

总的来说工具性,也是遍历LDR,找DLL Base,然后找导出表;

当然,还需要进行设置shellcode,来开辟线程注入,x64 x86shellcode分别是

```C++
//48:83EC 30 | sub rsp, 30 |
	//B9 00000000 | mov ecx, 0 |
	//BA 00000000 | mov edx, 0 |
	//49 : B8 78563412785634 | mov r8, 1234567812345678 | r8 : &"吚x\n€|$@"
	//4C : 8D4C24 28 | lea r9, qword ptr ss : [rsp + 28] |
	//FF15 07000000 | call qword ptr ds : [7FFF0F30CECF] |
	//48 : 83C4 1E | add rsp, 1E |
	//33C0 | xor eax, eax |
	//C3 | ret |
	//0000000000000000 | dq 0 |

	inline char x64_shellcode[] = {
	0x48,0x83,0xEC,0x38,
	0xB9,0x00,0x00,0x00,0x00,
	0xBA,0x00,0x00,0x00,0x00,
	0x49,0xB8,0x78,0x56,0x34,0x12,0x78,0x56,0x34,0x12,
	0x4C,0x8D,0x4C,0x24,0x28,
	0xFF,0x15,0x07,0x00,0x00,0x00,
	0x48,0x83,0xC4,0x38,
	0x33,0xC0,
	0xC3,
	0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00
	};

	//| 83EC 10 | sub esp, 50 |
	//| 6A 00 | push 0 |
	//| 8D4424 08 | lea eax, dword ptr ss : [esp + 20] |
	//| 890424 | mov dword ptr ss : [esp] , eax |
	//| 68 78563412 | push 12345678 |
	//| 6A 00 | push 0 |
	//| 6A 00 | push 0 |
	//| B8 78563412 | mov eax, 12345678 |
	//| FFD0 | call eax |
	//| 83C4 0A | add esp, 50 |
	//| 33C0 | xor eax, eax |
	//| C3 | ret |
	inline char x86_shellcode[] = {
		0x83,0xEC,0x50,
		0x6A,0x00,
		0x8D,0x44,0x24,0x20,
		0x89,0x04,0x24,
		0x68,0x78,0x56,0x34,0x12, //UNICODE_STRING*
		0x6A,0x00,
		0x6A,0x00,
		0xB8,0x78,0x56,0x34,0x12,
		0xFF,0xD0,
		0x83,0xC4,0x50,
		0x33,0xC0,
		0xC3
	};
#pragma pack(push)
#pragma pack(1)
	typedef struct x64_shellcode_t {
		char padding[16];
		UNICODE_STRING* dllPath;
		char padding02[18];
		UINT_PTR LdrLoadDll;
	}*px64_shellcode_t;

	typedef struct x86_shellcode_t {
		char padding[13];
		ULONG dllPath;//x86指针
		char paddin02[5];
		ULONG LdrLoadDll;//
		char padding02[8];
	}*px86_shellcode_t;
#pragma pack(pop)


	//0x8 bytes (sizeof) x86的这个要手动弄
	typedef struct UNICODE_STRING_x86
	{
		USHORT Length;                                                          //0x0
		USHORT MaximumLength;                                                   //0x2
		ULONG Buffer;                                                          //0x4
	}*PUNICODE_STRING_x86;
```

x86的API函数均是stdcall,因此不用手动平衡堆栈;

注入的函数大同小异

```C++
	auto inject_x86(PEPROCESS process,const wchar_t* dll_path) -> bool {
		wchar_t dllPath[MAX_PATH] = { 0 };
		wcscpy(dllPath, dll_path);
		KAPC_STATE apc{ 0 };
		//先分配内存
		bool ret = false;
		KeStackAttachProcess(process, &apc);

		do {
			__try {
				auto procDll = _Utils::_ualloc(PAGE_SIZE, PAGE_READWRITE);
				if (procDll == nullptr) break;

				//复制一下
				memset(procDll, 0, PAGE_SIZE);
				memcpy(procDll, dllPath, MAX_PATH);

				//然后+0x500位置就是UNICODE_STRING*
				auto usprocDll = (PUNICODE_STRING_x86)((ULONG)procDll + 0x500);
				
				//初始化x86的UNICODE_STRING
				usprocDll->Length = (USHORT)wcslen((wchar_t*)procDll)*2;
				usprocDll->MaximumLength = usprocDll->Length+2;
				usprocDll->Buffer = (ULONG)(procDll);

				//获取x86的LdrLoadDll
				auto LdrLoadDll=_Utils::find_module_export_wow64(process, 
					_Utils::find_module_base_wow64(process,L"ntdll.dll"), 
					"LdrLoadDll");
				//分配shellcode
				auto shellcode=_Utils::_ualloc(PAGE_SIZE, PAGE_EXECUTE_READWRITE);

				//写入shellcode
				memcpy(shellcode, x86_shellcode, sizeof x86_shellcode);

				//按照结构解析
				((px86_shellcode_t)(shellcode))->dllPath = (ULONG)usprocDll;
				((px86_shellcode_t)(shellcode))->LdrLoadDll = LdrLoadDll;

				//获取NtCreateThreadEx函数
				fnNtCreateThreadEx NtCreateThreadEx = 
					(fnNtCreateThreadEx)_Utils::get_nt_func("NtCreateThreadEx");
				//需要修改PreviousMode

				HANDLE hThread = 0;
				auto oMode=_Utils::change_pre_mode(KernelMode);
				auto status = NtCreateThreadEx(&hThread, THREAD_ALL_ACCESS, NULL, NtCurrentProcess(), 
					(PVOID)shellcode, NULL, 0, 0, 0, 0, 0);
				//恢复PreviousMode
				_Utils::resume_pre_mode(oMode);
				if (!NT_SUCCESS(status)) break;

				//成功 去执行shellcode了
				ret = true;
				break;
			
			}
			__except (1) { break; }
			

			//填入
		} while (false);


		KeUnstackDetachProcess(&apc);
		return ret;
	}
```

### 强制结束进程

强制结束进程很简单,只要有句柄权限就行,因此可以按照下面方式进行写

```C++
	auto terminate_process(HANDLE pid) -> NTSTATUS {
		HANDLE hProcess = 0;
		NTSTATUS status;
		OBJECT_ATTRIBUTES oa{ 0 };
		PEPROCESS process{ 0 };

		status = PsLookupProcessByProcessId(pid,&process);
		if (!NT_SUCCESS(status)) return status;
		ObDereferenceObject(process);

		::CLIENT_ID id{ id.UniqueProcess = PsGetProcessId(process),0 };
		ZwOpenProcess(&hProcess, PROCESS_ALL_ACCESS, &oa, &id);

		status =ZwTerminateProcess(hProcess, 0);
		ZwClose(hProcess);
		return status;

	}
```

通过ZwOpenProcess来进行获取需要的句柄权限,然后ZwTerminateProcess进行结束;



