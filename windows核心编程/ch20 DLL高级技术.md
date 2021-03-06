# 三 延迟装载DLL 

一个延迟载入的DLL是隐式链接的**(通过LoadLibrary+getProcAddress的方式)**，系统一开始不会将该DLL载入，只有当我们的代码试图去引用DLL中包含的一个符号时，系统才会实际载入该DLL。

**缺点：**
- **含有导出字段的DLL是无法延迟载入**  
- kernel32.dll模块是无法延迟载入的，这是因为必须载入该模块才能调用LoadLibrary和GetProcAddress  
- **不应该在DllMain入口点函数中调用一个延迟载入的函数，因为这样可能会导致程序崩溃**  

但是，如果采用显示装载DLL的话，大量的DLL在初始化阶段被加载，会导致程序加载缓慢，影响用户体验。因此，延迟装载被提了出来：

**使用：**

在链接可执行文件的时候，修改一些链接器的开关，注意：**不能在**源代码中通过**#pragma comment(linker,"")**来设置。

**延迟载入DLL: 链接器->输入->延迟装载(添加需要延迟装载的DLL名字)**

**卸载延迟加载的DLL:链接器->高级->卸载延迟记载的DLL->选择（是）**

通过上面两个开关告诉链接器下列事情：

1. 将DLL从可执行模块的导入表中去除

2. 在可执行模块中嵌入一个新的延迟载入段（即Delay Import Section，称为**.didata**) 来标识要从simple.Dll中导入哪些函数。

3. 通过让对延迟载入函数的调用跳转到__delayLoadHelper2函数，来完成对延迟载入函数的解析。

# 四 函数转发器 

函数转发器是DLL输出段中的一个条目，用来将一个函数调用转发到另一个DLL中的另一个函数。

在自己的DLL模块中使用函数转发器。最简单的方法是使用pragma指示器，如下面所示：

```c 
#pragma comment(linker,"/export:SomeFunc=DllWork.SomeOhterFunc")

// 例如转发User32的MessageBox
#pragma comment(linker, "/export:MSGBOXW=User32.MessageBoxW")

// 使用部分
#include <windows.h>

typedef int (*fMessageBoxW)(
    HWND    hWnd,
    LPCTSTR lpText,
    LPCTSTR lpCaption,
    UINT    uType
);

int main()
{
    HMODULE hDll = LoadLibrary(L"secondDll.Dll");
    if (hDll == NULL)
    {
        MessageBox(NULL, L"载入DLL失败", L"", MB_OK);
        return -1;
    }
    fMessageBoxW f = (fMessageBoxW)GetProcAddress(hDll, "MSGBOXW");
    f(NULL, TEXT("测试"), TEXT("msg"), MB_OK);
    FreeLibrary(hDll);
    return 0;
}
```

意思是，编译DLL时输出一个SomeFunc函数，实际实现的是DllWork.dll模块的SomeOtherFunc函数。

# 五 已知的DLL

系统对操作系统提供的某些DLL进行了特殊处理，这些DLL被称为已知的DLL。

在注册表中有这个一个注册表项：

```c
HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs
```

![img](images/bfec7893-50b9-4270-97db-6bb5b3c248d7.png)

当**LoadLibrary**或**LoadLibraryEx**被调用的时候，系统会先在注册表中寻找是否存在的项，如果找到，则从这个注册表项的**DllDirectory**值（默认为%SystemRoot%\System32）所表示的目录中开始搜索DLL。如果没有找到，则使用正常的搜索规则。

```c
LoadLibrary(TEXT("SomeLib"));
LoadLibrary(TEXT("SomeLib.dll"));
// 如果传入的名字带.dll后缀,会先将扩展名去掉，然后在KnownDLLs注册表项中搜索，看其中是否有之符合的值名。、
// 如果没有则用正常的搜索规则。但是，如果找到，那么系统会查看与值名相对应的数据，并试图用该数据载入DLL。
// 系统还会从这个注册表项的DllDirectory值所表示的目录中开始搜索DLL。DllDirectory默认值为%SystemRoot%\System32
```

# 六 DLL重定向

DLL重定向：强制操作系统的加载程序首先从应用程序的目录中载入模块，只有当加载程序无法找到要找的文件时，才会在其他的目录中寻找。

2种方法：

1. 创建一个文件

&ensp;&ensp;在应用程序（假设为app.exe）的目录创建一个名为**app.exe.local文件**，LoadLibrary(Ex)在内部做了修改，来检查这个文件是否存在。如果存在，则加载这个目录中的模块。

2. 创建一个目录

&ensp;&ensp;或者创建一个名为**app.exe.local目录**，把希望应用加载的DLL放在这个目录中。

```c
// 默认情况下禁用太特性，以下时打开方法
HKLM\Software\Microsoft\WindowsNT\CurrentVersion\Image File Execution Options
增加一个条目 DWORD DevOverrideEnable=1
```

![img](images/4f96dd35-2db8-48e8-828a-b8b0db452054.png)  

# 七 模块的基地址重定位

# 八 模块的绑定