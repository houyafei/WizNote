当线程发出一个同步设备I/O请求的时候，它会被临时挂起，直到设备完成I/O请求为止。此类挂起会损害性能，这是因为线程无法进行有用的工作，比如开始对另一个请求进行处理。
因此，简而言之，我们希望线程不会被阻塞住，这样它们就能始终进行有用的工作。

# 一 打开和关闭设备

windows的优势之一就是它所支持的设备数量。我们把设备定义为能够与之进行通信的任何东西。

| **设备**       | **打开方式**                                                 | **常见用途**                                                 |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 文件 | CreateFile |   |
| 目录           | CreateFile，需要指定**FILE_FLAG_BACKUP_SEMANTICS**标志       | 打开目录是我们能够改变目录的属性和它的时间戳                 |
| 逻辑磁盘驱动器 | CreateFile（pszName为"**\\.\x:**"），其中x是驱动器的盘符     | 格式化驱动器或者检测驱动器媒介的大小                         |
| 物理磁盘驱动器 | CreateFile（pszName为"**\\.\PHYSICALDRIVEx**"）.其中x是物理驱动器号。例如，为了读写用户的第一物理驱动器的扇区，我们应该指定"\\.\PHYSICALDRIVE0" | 访问分区表,注意，错误地写入设备可能会导致操作系统的文件系统无法访问磁盘的内容 |
| 串口           | CreateFile（pszName为“**COMx**”）                            | 通过电话线传输数据                                           |
| 并口           | CreateFile（pszName为“**LPTx**”）                            | 将数据传输给打印机                                           |
| 邮件槽         | 服务端：**CreateMailslot**（pszName为"**\\.\mailslot\mailslotname**"） 客户端：**CreateFile**（pszName为"**\\servename\mailslotname**"） | **一对多数据传输**                                           |
| 命名管道       | 服务端：**CreateNamepipe**(pszName为"**\\.\pipe\pipename**") 客户端：CreateFile(pszName为"**\\servername\pipe\pipename**") | **一对一数据传输**                                           |
| 匿名管道       | CreatePipe用来打开服务端和客户端                             |                                                              |
| 套接字         | Socket，accept或者AcceptEx                                   | 保温或数据流的传输                                           |
| 控制台         | **CreateConsoleScreenBuffer**或**GetStdHandle**              | 文件窗口的屏幕缓存                                           |

```c
// 一般使用CloseHandle关闭
BOOL CloseHandle(HANDLE hDevice);
// 套接字则使用closesocket关闭
int closesocket(SOCKET s);
// 查询设备的类型
DWORD GetFileType(HANDLE hDevice);
// FILE_TYPE_UNKNOWN 文件为未知类型
// FILE_TYPE_DISK    文件时一个磁盘文件
// FILE_TYPE_CHAR    是一个并口设备或控制台
// FILE_TYPE_PIPE    命名管道或匿名管道
```

# 二 使用文件设备

## 2.1 取得文件的大小

```c
// 获取文件的逻辑大小
DWORD WINAPI GetFileSize( HANDLE  hFile, _Out_opt_ LPDWORD lpFileSizeHigh);

// lpFileSizeHigh和返回值确定文件大小 
BOOL GetFileSizeEx(HANDLE hFile, PLARGE_INTEGER pliFileSize);

// PLARGE_INTEGER是一个联合类型
// 返回文件的物理大小
DWORD GetCompressedFileSize(LPCTSTR lpFileName, _Out_opt_ LPDWORD lpFileSizeHigh);
```

## 2.2 设置文件指针的位置

```c
// 注意：一个内核对象存在一个指针。
BOOL WINAPI SetFilePointerEx(
  _In_      HANDLE         hFile,                // 设备句柄
  _In_      LARGE_INTEGER  liDistanceToMove,     // 移动距离
  _Out_opt_ PLARGE_INTEGER lpNewFilePointer,     // 返回移动后的新位置，可为NULL
  _In_      DWORD          dwMoveMethod          // 移动基址,FILE_BEGIN\FILE_END\FILE_CURRENT
);
```

**注意：**  
1.可以将文件指针设置超过文件当前大小。除非在该位置向文件写入数据或者调用**SetEndOfFile**，否则不会增加文件中在磁盘的实际大小  
2.如果SetFilePointerEx操作的文件是用**FILE_FLAG_NO_BUFFERING**标志打开，那么文件指针只能被设置为扇区大小的整数倍。  
3.windows没有提供**GetFilePointerEx**函数，可通过**SetFilePointerEx**获取。  

```c
SetFilePointerEx(hFile,0,&liCurrentPointer,FILE_CURRENT)
```

## 2.3 设置文件尾

通常，在关闭文件的时候，系统会负责设置文件尾。

**SetEndOfFile将当前文件指针设定文件截断文件的大小或增加文件大小。**

```c
BOOL WINAPI SetEndOfFile(_In_ HANDLE hFile);
```

# 三 执行同步设备I/O

windwos允许我们执行同步设备的I/O。记住，设备可以是文件，也可以是邮件曹、管道、套接字等等。无论使用的是何种类型的设备，我们都可以使用相同的函数来执行I/O操作。

注意：
1.执行同步设备的I/O时，CreateFile时一定不能指定参数**FILE_FLAG_OVERLAPPED**标志。
2.同步执行ReadFile和WriteFile，如果成功返回TRUE，错误则返回FALSE，具体错误通过GetLastError返回。  
```c
BOOL WINAPI WriteFile(
	HANDLE hFile, 
	LPCVOID lpBuffer,
    DWORD nNumberOfBytesToWrite, 
    LPDWORD lpNumberOfBytesWritten, 
    LPOVERLAPPED lpOverlapped);
    
BOOL WINAPI ReadFile(
	HANDLE hFile, 
	LPVOID lpBuffer, 
	DWORD nNumberOfBytesToRead, 
	LPDWORD lpNumberOfBytesRead, 
    LPOVERLAPPED lpOverlapped);
// lpOverlapped：使用同步I/O时，该参数必须为NULL
```

## 3.1 将数据刷新至设备

```c
BOOL FlushFileBuffers(HANDLE hFile);    // 将缓冲区数据写入设备
```

## 3.2 同步I/O的取消

windows vista中，下面函数允许我们**将一个给定线程尚未完成的同步I/O请求取消**:

```c
BOOL CancelSynchronousIO(HANDLE hThread); // hThread具有THREAD_THERMINATE权限
```

# 四 异步设备I/O基础

要以异步的方式来访问设备，必须在调用CreateFile，并在dwFlagsAndAttributes参数中指定**FILE_FLAG_OVERLAPPED**标志打开设备。读写操作还是使用**ReadFile**和**WriteFile**。

```c
BOOL WINAPI WriteFile(_In HANDLE hFile, 
                      _In_out LPCVOID lpBuffer,
                      _In DWORD nNumberOfBytesToWrite, 
                      _In_out LPDWORD lpNumberOfBytesWritten,  /*无意义*/
                      _In_out LPOVERLAPPED lpOverlapped);
                      
BOOL WINAPI ReadFile(_In HANDLE hFile, 
                     _In_out LPVOID lpBuffer, 
                     _In DWORD nNumberOfBytesToRead, 
                     _In_out LPDWORD lpNumberOfBytesRead, /*无意义*/
                     _In_out LPOVERLAPPED lpOverlapped);
// lpNumberOfBytesWritten和lpNumberOfBytesRead：无意义，设为NULL。
// lpOverlapped:需要指定一个OVERLAPPED结构对象。
// 使用可提醒IO进行异步I/O时，必须使用ReadFileEx和ReadFileEx替换ReadFile和WriteFile。（请参考5.3）
```

## 4.1 OVERLAPPED结构

```c
typedef struct _OVERLAPPED {
  _Out  ULONG_PTR Internal;      // 错误码
  _Out  ULONG_PTR InternalHigh;  // 实际读写的字节数
  _In   DWORD Offset; // 读写起始位置(非文件设备忽略该字段,此时必须初始化为0,异步文件IO时必须指定)
  _In   DWORD OffsetHigh; 
  _In_out HANDLE hEvent; // 事件对象
} OVERLAPPED, *LPOVERLAPPED;
```

## 4.2 异步设备I/O的注意事项

在执行异步I/O操作的时候，设备驱动程序不一定以先入先出的方式来处理队列中的I/O请求。
另外，如果请求的I/O是**异步方式执行**，那么正常情况下**ReadFile**和**WriteFile**会返回**FALSE**，这时候调用**GetLastError**返回**ERROR_IO_PENDING**，那么I/O请求已经被成功地加入了I/O队列，会在晚些时候完成。

如果返回地是**ERROR_IO_PENDING**以外的值，那么表示I/O请求无法被添加到设备驱动程序队列。

| ERROR_INVALID_USER_BUFFER |      |
| ------------------------- | ---- |
| RROR_NOT_ENOUHT_MEMORY    |      |
| ERROR_NOT_ENOUHT_QUOTA    |      |

如何处理上述操作：
这些错误地发生基本上是因为还有一定数量地待处理I/O请求未完成，因此我们需要等带一些I/O处理完成之后再次调用ReadFile和WriteFile。

注意：
在异步I/O完成之之前，一定不能移动或是销毁在发出I/O请求时所使用地OVERLAPPED结果（所以，异步I/O应该给每一个I/O搭配一个OVERLAPPED类型参数）。

## 4.3取消队列中的设备I/O请求

在设备驱动程序对一个已经加入队列的设备I/O请求进行处理之前将其取消。  
```c
// 取消指定文件句柄内的所有IO请求
BOOL WINAPI CancelIo( _In_ HANDLE hFile);
// 取消指定文件句柄的特定IO请求
BOOL WINAPI CancelIoEx(_In_ HANDLE hFile, _In_opt_ LPOVERLAPPED lpOverlapped);
```

# 五 接收I/O请求完成通知

windows提供了4中不同方法来接收I/O请求已经完成地通知。

| **技术**         | **摘要**                                                     |
| ---------------- | ------------------------------------------------------------ |
| 触发设备内核对象 | 当向一个设备同时发出多个I/O请求的时候，这种方式没什么用 它允许一个线程发出I/O请求，另一个线程对结果进行处理 |
| 触发事件内核对象 | 这种方法允许我们向一个设备同时发出多个I/O请求 **它允许一个线程发出I/O请求，另一个线程对结果进行处理** |
| 使用可提醒I/O    | 这种方法允许我们向一个设备同时发出多个I/O请求。 **发出I/O请求的线程必须对结果进行处理。** |
| 使用I/O完成端口  | 这种方法允许我们向一个设备同时发出多个I/O请求。 它允许一个线程发出I/O请求，另一个线程对结果进行处理，这项技术具有高度的伸缩性和最佳的灵活性 |

## 5.1 触发设备内核对象

ReadFile和WriteFile将I/O请求添加到队列之前，会将内核对象设为未触发状态，当设备驱动完成了请求之后，驱动程序会将设备内核对象设为触发状态。

**缺点：**不能处理多个I/O请求，例如同时对同一个文件的头尾进行读操作(这里指同一个句柄)，我们不能通过等待设备内核对象来对线程进行同步，因为任何一个操作完成都会使该设备内核对象被触发。

```c
// 注意，以下代码不值得参考，和同步读写性能一样，仅作讲解。
HANDLE hFile = CreateFile(...,FILE_FLAG_OVERLAPPED,...);
BYTE bBuffer[100];
OVERLAPPED o= {0};
o.Offset = 345;
BOOL bReadDown = ReadFile(hFile, bBuffer, 100, NULL, &o);
DWORD dwError = GetLastError();
if(!bReadDown && dwError == ERROR_IO_PENDING)){
    WaitForSingleObject(hFile,INFINITE);
    bReadDone = TRUE;
}
if(bReadDone){  
}else{
}
```

## 5.2 触发事件内核对象

当一个异步IO完成时，设备驱动程序会检查**OVERLAPPED**结构的**hEvent**成员是否为NULL，如果不为NULL，那么驱动程序会调用**SetEvent**来触发事件。当然，驱动程序仍然也会将设备内核对象设为触发状态。  

```c
BOOL WINAPI SetFileCompletionNotificationModes( _In_ HANDLE FileHandle, _In_ UCHAR  Flags);
// Flags传入FILE_SKIP_SET_EVENT_ON_HANDLE，IO完成时不会触发文件句柄。

BOOL WINAPI GetOverlappedResult(
  _In_  HANDLE       hFile,            // 设备句柄
  _In_  LPOVERLAPPED lpOverlapped,    // VOERLAPPED结构结果
  _Out_ LPDWORD      lpNumberOfBytesTransferred,    // 传输的字节数
  _In_  BOOL         bWait    // TRUE：等到IO完成才返回，FALSE：返回FALSE，GetLastError()返回ERROR_IO_INCOMPLETE.
);


// 例子：
HANDLE hFile = CreateFile(..., FILE_FLAG_OVERLAPPED,...);
BYTE bReadBuff[10];    // 读操作
OVERLAPPED oRead = {0};
oRead.hEvent = CreateEvent(..);
// 提交一个读操作给读写驱动
ReadFile(hFile, bReadBuff, 10, NULL,&oRead);
// ReadFileEx(hFile, bReadBuff,10, &oRead, NULL);
BYTE bWriteBuff[10] = {0,1,2,3,4,5,6,7,8,9};
OVERLAPPED oWrite = {0};
oWrite.hEvent = CreateEvent(...);
// 提交一个写操作给底层驱动
WriteFile(hFile,bWriteBuff, _countof(bWriteBuff), NULL, &oWrite);
// WriteFileEx(hFile, bWriteBuff, _countof(bWriteBuff), &oWrite, NULL);
// 干点别的事情。。。。。
HANDLE h[2] = {oRead.hEvent, oWrite.hEvent);
// 获取读写结果               
DOWRD dw = waitforMultipleObjects(2,h, FALSE, INFINITE);
switch(dw)
{
    case WAIT_TIMEOUT:
    case WAIT_FATLE:
    case WAIT_OBJECT_0 + 0;
    case WAIT_OBJECT_0 + 1:
}
```

## 5.3 使用可提醒I/O

当系统创建一个线程的时候，会同时创建一个与线程相关联的队列。这个队列被称为**异步过程调用（APC）队列**。

为了将IO完成通知添加到线程的APC队列中，我们应该调用**ReadFileEx**和**WriteFileEx。**

**当ReadFileEx和WriteFileEx**发出一个I/O请求的时候，这两个函数会**将I/O操作**以及**回调函数的地址(lpCompletionRoutine的值)传给设备驱动程序**。

当设备驱动程序完成I/O请求的时候，会在发出I/O请求的线程的APC队列中添加一项。该项包含了回调函数的地址，以及在发出I/O请求时所使用的OVERLAPPED结构的地址。

当线程处于**可提醒状态**的时候，系统会检查的它APC队列，对队列中的每一项，系统会调用完成函数，并传入I/O错误码，已传输字节数以及OVERLAPPED结果的地址。

注意，如果回调函数执行完毕后，会从APC队列中释放，如果需要对继续读写，则需要在对调函数返回前，重新调用ReadFileEx或WriteFileEx进行设置。

```c
// 异步读取文件
BOOL WINAPI ReadFileEx(
	HANDLE hFile, 
    LPVOID lpBuffer, 
    DWORD nNumberOfBytesToRead,
    _Inout_ LPOVERLAPPED lpOverlapped, 
    /*回调函数*/
    _In_ LPOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine); 

// 异步写入文件
BOOL WINAPI WriteFileEx( 
	HANDLE hFile, 
    LPVOID lpBuffer, 
    DWORD nNumberOfBytesToWrite,
    _Inout_  LPOVERLAPPED lpOverlapped, 
    /*回调函数*/
    _In_ LPOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine); 
// lpCompletionRoutine：回调函数。

// 下面是完成函数的格式：
VOID CALLBACK FileIOCompletionRoutine(
  _In_    DWORD        dwErrorCode,                // 错误码
  _In_    DWORD        dwNumberOfBytesTransfered,  // 实际读写数据字节数
  _Inout_ LPOVERLAPPED lpOverlapped     // 在异步读写，可提醒I/O方式下，系统没有使用该参数，用户可自行发挥（注意，该参数是输入输出的）
);
```

**注意：**

1.当一个可提醒I/O完成时，驱动程序不会触发**事件对象**。事实上，设备根本没有使用到OVERLAPPED的hEvent成员。**(此时，我们可以利用hEvent占用的空间)**

2.这个APC由队列是由系统在内部维护的

3.当IO请求完成的时候，系统会将他们添加到线程的APC队列中 ------ 回掉函数并不会立即被调用。

4.为了对线程APC队列中的项进行处理，线程必须将自己置为**可提醒状态**。

windows提供了6个函数，可以将线程设置为**可提醒状态**。前5个函数的最后一个参数**bAlertable**表示调用线程是否应该将自己置为可提醒状态。

对**MsgWaitForMultipleObjectsEx**来说，必须使用**MWMO_ALERTABLE**标志来让线程进入可提醒状态。

需要注意，当调用这些函数中的任何一个时，只要线程的APC队列中至少有一项，线程就不会进入睡眠状态。

**注意，线程是否处在可提醒状态非常重要，这决定完成函数能否被正确地执行。**

```c
DWORD WINAPI SleepEx(DWORD dwMilliseconds, BOOL  bAlertable);
DWORD WINAPI WaitForSingleObjectEx( HANDLE hHandle, DWORD dwMilliseconds, BOOL bAlertable);
DWORD WINAPI WaitForMultipleObjectsEx(DWORD nCount,const HANDLE *lpHandles, BOOL bWaitAll,DWORD dwMilliseconds, BOOL bAlertable);
DWORD WINAPI SignalObjectAndWait(HANDLE hObjectToSignal, HANDLE hObjectToWaitOn, DWORD dwMilliseconds, BOOL bAlertable);
BOOL WINAPI GetQueuedCompletionStatusEx(
  _In_  HANDLE             CompletionPort,  
  _Out_ LPOVERLAPPED_ENTRY lpCompletionPortEntries,
  _In_  ULONG              ulCount,
  _Out_ PULONG             ulNumEntriesRemoved,
  _In_  DWORD              dwMilliseconds,
  _In_  BOOL               fAlertable
);
DWORD WINAPI MsgWaitForMultipleObjectsEx(
  _In_       DWORD  nCount,
  _In_ const HANDLE *pHandles,
  _In_       DWORD  dwMilliseconds,
  _In_       DWORD  dwWakeMask,
  _In_       DWORD  dwFlags
);
```

**缺点：**
1.回调函数 —— 可提醒函数要求我们创建一个回调函数，这使得代码的**实现变得更加复杂**。
2.线程问题 —— **负载不均**，发出IO请求的线程必须同时对完成通知进行处理。如果一个线程发出多个请求，即使其他线程处于空闲状态，该线程也必须对请求的完成通知做出响应。
3.
Windows提供了一个函数，允许我们手动地将一项添加到APC队列中：

```c
DWORD WINAPI QueueUserAPC(PAPCFUNC pfnAPC, // 函数指针
                          HANDLE hThread,  // 插入目标APC队列的线程
                          ULONG_PTR dwData); // 完成函数的参数

// 可提醒IO的APC完成函数的原型
VOID WINAPI APCFunc(ULONG_PTR dwParam); 
```

## 5.4 使用I/O完成端口

IO完成端口地设计初衷就是与线程池配合使用。

### 5.4.1 创建IO完成端口

前三个参数只在把完成端口同设备相关联的时候才有用。如果不关联设备，只创建完成端口，那么前三个参数可以为：INVALID_HANDLE_VALUE，NULL，0。最后一个参数指示I/O完成端口同时能运行的最多线程数。如果为0，那么默认为机器上的CPU数。不过你可以用几个不同的值做实验来确定哪个值有最佳的性能。顺便说一句，这个函数是唯一一个创建了内核对象，而没有 LPSECURITY_ATTRIBUTES 参数的 Win32 函数。这是因为**完成端口只应用于一个进程内**。

　　当你创建一个I/O完成端口时，内核实际上创建了5个不同的数据结构。

注意这里的几个结构：
1）设备列表 
2）IO完成队列（先入先出） 
3）等待线程队列（后入先出）【注意不是常规上的】 
4）已释放现成队列
5）已暂停现成列表

```c
// 1.创建I/O完成端口对象
// 2.将设备与I/O完成端口关联起来
HANDLE WINAPI CreateIoCompletionPort(
  _In_     HANDLE    FileHandle,                
  _In_opt_ HANDLE    ExistingCompletionPort,
  _In_     ULONG_PTR CompletionKey,            // 完成键，操作系统不关心这个传入什么值，一般是我们用来识别操作类型
  _In_     DWORD     NumberOfConcurrentThreads
);


// 1. 创建一个新的I/O完成端口对象
HANDLE hCompletionPort = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, N); 
// N是同一时间最多能有线程线程处在可运行状态，传入0，则使用默认值（等于主机CPU数量）
// 2. 将设备与I/O完成端口关联起来（这个很重要）
CreateIoCompletionPort(hDevice, hCompletionPort, CK_READ【自定义】, 0);
//
//////////// 思路上可以将它拆成两个函数
// 
// 创建新的IO完成端口对象
HANDLE CreateNewCompletionPort(DWORD dwNumberOfConcurrentThread){
    return CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, dwNumberOfConcurrentThread);
}
//  关联IO完成端口对象和设备对象
// 所有发往hDevice的IO请求在完成的时候都有一个dwCompletionKey。
BOOL AssociateDeviceWithCompletionPort(
    HANDLE hCompletionPort,
    HANDLE hDevice,
    DWORD dwCompletionKey){
    HANDLE h = CreateCompletionPort(hDevice, hCompletionPort, dwCompletionKey, 0);
    return h == hCompletionPort;
}
```

### 5.4.2 IO完成端口地周边框架

线程池的线程数量一般是CPU数量的两倍。  
线程池中的所有线程应该执行同一个函数。一般来说，线程完成初始化工作，会进入一个循环。在循环内部，线程将自己切换到睡眠状态，来等待设备IO请求完成并进入完成端口。  
调用**GetQueuedCompletionStatus**可以达到这一目的：

```c
BOOL WINAPI GetQueuedCompletionStatus(
  _In_  HANDLE       CompletionPort,    // 要监控的完成端口句柄
  _Out_ LPDWORD      lpNumberOfBytes,   // 完成读写字节数
  _Out_ PULONG_PTR   lpCompletionKey,   // 完成键（用户自定义），框架会根据实际执行的线程填写该标识
  _Out_ LPOVERLAPPED *lpOverlapped,     // 返回Overlapped结构
  _In_  DWORD        dwMilliseconds     // 等待时间
);
// 如果预计会不断收到大量的IO请求，那么我们可以调用GetQueuedCompletionStatusEx同时取得多个IO请求
// 而不必让许多线程等待完成端口，减少线程上下文切换的时间
BOOL WINAPI GetQueuedCompletionStatusEx(
  _In_  HANDLE             CompletionPort,
  _Out_ LPOVERLAPPED_ENTRY lpCompletionPortEntries,
  _In_  ULONG              ulCount,
  _Out_ PULONG             ulNumEntriesRemoved,
  _In_  DWORD              dwMilliseconds,
  _In_  BOOL               fAlertable
);
/////////////////////////////////////////////////////////////////
// 检查队列情况
/////////////////////////////////////////////////////////////////
 
DWORD dwNumBytes;
ULONG_PTR CompletionKey;
OVERLAPPED* pOverlapped;
// 绑定设备句柄和IO完成端口句柄
BOOL bOk = GetQueuedCompletionStatus(hIOCP,&dwNumBytes, &CompletionKey, &pOverlapped, 1000);
DWORD dwError = GetLastError();
if(bOk){
    // 处理成功的IO请求
    switch(CompletionKey)
    {
        case CK_READ:
            // 进行读操作
            break;
        case CK_WRITE:
            // 进行写操作
            break;
    }
}else{
    if(pOverlapped != NULL){
        // 处理失败的IO请求,dwError包含失败原因
    }else{
        if(dwError == WAIT_TIMEOUT){
            // 等待I/O完成端口对象超时
        }else{
            // 调用 GetQueuedCompletionStatus 函数失败，dwError包含失败原因
        }
    }
}
```

### 5.4.3 IO完成端口如何管理线程池

### 5.4.4 线程中有多少线程？

## 5.5 模拟已完成地IO请求

IO完成端口并不一定用于设备的IO。**PostQueuedCompletionStatus** 用来将一个已完成的IO通知追加到IO完成端口的队列中。  
```c
BOOL WINAPI PostQueuedCompletionStatus(
  _In_     HANDLE       CompletionPort,    // 完成端口对象句柄
  _In_     DWORD        dwNumberOfBytesTransferred,    // 传输字节数
  _In_     ULONG_PTR    dwCompletionKey,    // 完成键
  _In_opt_ LPOVERLAPPED lpOverlapped    // 
);
```