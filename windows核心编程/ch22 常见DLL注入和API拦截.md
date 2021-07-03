**目录：**

一 使用注册表来注入DLL

**二 挂起进程注入**

**三 调试器注入(调试API的使用)**

**四 挂起线程注入**

**五 钩子注入**

**六 APC注入**

**七 远程线程注入DLL**

**八 DLL劫持**

**九 导入表注入**

**十 输入法注入**

**十一 LSP注入**

十二 **使用LdrLoadDll和_NtCreateThreadEx实现注入**

# **一 使用注册表来注入DLL**

**Appinit_Dll/AppCertDlls/IFEO（映像劫持）可以用于注入和持久化。完整的路径如下：**

```bat
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows\Appinit_Dll
HKLM\Software\Wow6432Node\Microsoft\Windows\NT\CurrentVersion\Windows\Appinit_Dlls
HKLM\System\CurrentControlSet\Control\Session Manager\AppCertDlls
HKLM\Software\Microsoft\Windows NT\CurrentVersion\image file execution options
```

**1 使用AppInit_Dll**恶意软件能在AppInit_DLLs键下插入恶意的DLL的路径，以便其他进程加载。该键下每个DLL会被加载到所有的加载User32.dll的进程中。User32.dll是常见的Windows基础库。因此，当恶意软件修改这个子键时，大量进程将加载恶意的DLL。

其中的**AppInit_Dlls**键的值可能包含一个Dll的文件名或一组Dll的文件名（通过空格或逗号分隔）。同时，为了让系统使用这个注册表值，还必须创建一个名为**LoadAppInit_Dlls**类型位DWORD的注册表项，并将它的值设为1.**缺点：**1.DLL只会被映射到使用了User32.dll的进程中，而非指定的进程.实际上很多进程并没有使用User32.dll2.映射的进程越多，“容器”进程奔溃的可能性就越大3.UEFI安全启动禁止Appinit_Dlls的相关内容（解决方法：在BIOS界面关闭相关UEFI安全启动功能即可）

**2 AppCertDlls**

```c
void reg_inject_dll(LPCTSTR lpszExeName,LPCTSTR lpszDllName)
{
    HKEY hKey;
    int dwFlag = 0x100;
    TCHAR szKey[MAX_PATH] = {};
    StringCbPrintf(szKey,sizeof(szKey),
        _T("Software\\Microsoft\\Windows NT\\CurrentVersion\\Image File Execution Options\\%s"),
        lpszExeName);
    RegCreateKeyEx(HKEY_LOCAL_MACHINE, 
        szKey, 
        0, NULL,
        0, KEY_ALL_ACCESS, 
        NULL, &hKey, NULL);
    RegSetValueEx(hKey, _T("VerifierDlls"), 0, REG_SZ, (BYTE *)lpszDllName, (lstrlen(lpszDllName) + 1) * sizeof(char));
    RegSetValueEx(hKey, _T("GlobalFlag"), 0, REG_DWORD, (BYTE *)&dwFlag, sizeof(dwFlag));        
    RegCloseKey(hKey);
}
```


3 IFEO


# 二 挂起进程注入

```c
#include <windows.h>
#include <stddef.h>
#include <cstring>

//结构必须字节对齐1
#pragma pack(1)  
typedef struct _INJECT_CODE
{
    BYTE  byPUSHFD;
    BYTE  byPUSHAD;
    BYTE  byMOV_EAX;          //mov eax, addr szDllpath
    DWORD dwMOV_EAX_VALUE;
    BYTE  byPUSH_EAX;         //push eax
    BYTE  byMOV_ECX;          //mov ecx, LoadLibrary
    DWORD dwMOV_ECX_VALUE;
    WORD  wCALL_ECX;          //call ecx
    BYTE  byPOPAD;
    BYTE  byPOPFD;
    BYTE  byJUMP;
    DWORD dwJUMP_VALUE;
    CHAR  szDllPath[MAX_PATH];
}INJECT_CODE, *PINJECT_CODE;
#pragma pack()  

int main()
{
    //ShellCode();
    CONTEXT context = {0};
    STARTUPINFO si = {0};
    si.cb = sizeof(STARTUPINFO);
    PROCESS_INFORMATION pi = {0};
    if (CreateProcess(TEXT("D:\\HelloWorld.exe"),
                      NULL, NULL, NULL, 
                      FALSE, CREATE_SUSPENDED, NULL, NULL, &si, &pi)){
        DWORD dwWrite = 0;
        LPVOID pAlloc = VirtualAllocEx(pi.hProcess, 
                                       NULL, 
                                       0x1000, 
                                       MEM_COMMIT, 
                                       PAGE_EXECUTE_READWRITE);
        if (pAlloc != NULL){
            // 这个是直接注入DLL
            char * szDllPath = "D:\\TestDll.dll";
            //给ShellCode结构体赋值
            INJECT_CODE ic;
            ZeroMemory(&ic, sizeof(INJECT_CODE));
            
            context.ContextFlags = CONTEXT_FULL;
            GetThreadContext(pi.hThread, &context);
            ic.byPUSHFD = 0x9C;
            ic.byPUSHAD = 0x60;
            ic.byMOV_EAX = 0xB8;
            ic.dwMOV_EAX_VALUE = (DWORD)pAlloc + offsetof(INJECT_CODE, szDllPath);
            ic.byPUSH_EAX = 0x50;
            ic.byMOV_ECX = 0xB9;
            ic.dwMOV_ECX_VALUE = (DWORD)&LoadLibrary;
            ic.wCALL_ECX = 0xD1FF;
            ic.byPOPAD = 0x61;
            ic.byPOPFD = 0x9D;
            ic.byJUMP = 0xE9;
            
            // 注意jump返回地址是相对偏移的
            ic.dwJUMP_VALUE = context.Eip - (DWORD)((BYTE*)pAlloc + offsetof(INJECT_CODE, byJUMP) ) - 5;
            strncpy(ic.szDllPath, szDllPath, strlen(szDllPath));
            
            if (WriteProcessMemory(pi.hProcess, pAlloc, &ic, sizeof(INJECT_CODE), &dwWrite)){
                context.Eip = (DWORD)pAlloc;
                context.ContextFlags = CONTEXT_FULL;
                if (SetThreadContext(pi.hThread, &context)){
                    if (ResumeThread(pi.hThread) != -1){
                        OutputDebugString("恢复成功！");
                    }
                }
            }
        }
    }
}
```

## 三 调试器注入

```c
#include <windows.h>
#include <stddef.h>

#pragma  pack(1)
typedef struct _INJECT_CODE{
    BYTE  byMOV_EAX;          //mov eax, addr szDllpath
    DWORD dwMOV_EAX_VALUE;
    BYTE  byPUSH_EAX;         //push eax
    BYTE  byMOV_ECX;          //mov ecx, LoadLibrary
    DWORD dwMOV_ECX_VALUE;
    WORD  wCALL_ECX;          //call ecx
    BYTE  byINT3;             //int 3
    CHAR  szDllPath[MAX_PATH];
}INJECT_CODE, *PINJECT_CODE;
#pragma  pack()

int main()
{
    BOOL bIsSystemBreak = TRUE;
    CHAR * szDllPath = "D:\\TestDll.dll";
    PBYTE pBaseAddr = NULL;
    CONTEXT ctxNew = { 0 }, ctxOld = { 0 };
    STARTUPINFO si = { 0 };
    si.cb = sizeof(STARTUPINFO);
    PROCESS_INFORMATION pi = { 0 };
    DEBUG_EVENT dbgEvent = { 0 };
    INJECT_CODE ic = { 0 };
    DWORD dwWrite = 0;
    HANDLE hProcess = NULL;
    HANDLE hThread = NULL;
    DWORD  dwOffset = 0;
    if (CreateProcess(TEXT("D:\\HelloWorld.exe"), NULL, NULL, NULL,
        FALSE, DEBUG_ONLY_THIS_PROCESS, NULL, NULL, &si, &pi))
    {
        // 防止被调试进程和调试器一起关闭
        DebugSetProcessKillOnExit(FALSE);
        while (WaitForDebugEvent(&dbgEvent, INFINITE))
        {
            switch (dbgEvent.dwDebugEventCode)
            {
            case CREATE_PROCESS_DEBUG_EVENT:
                hProcess = dbgEvent.u.CreateProcessInfo.hProcess;
                hThread = dbgEvent.u.CreateProcessInfo.hThread;
                pBaseAddr = (PBYTE)VirtualAllocEx(hProcess, NULL, 0x1000, MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);
                if (pBaseAddr == NULL)
                {
                    OutputDebugString(TEXT("申请虚拟空间失败"));
                    return 0;
                }
                // 这段ShellCode影响很大，注意编写
                // ShellCode的编写问题：注意部分操作符对不同字节的情况对应的十六进制码不同
                // 顾最好现将数据mov到某个寄存器，然后再对寄存器进行操作
                ic.byMOV_EAX = 0xB8;
                ic.dwMOV_EAX_VALUE = (DWORD)pBaseAddr + offsetof(INJECT_CODE, szDllPath);
                ic.byPUSH_EAX = 0x50;
                ic.byMOV_ECX = 0xB9;
                ic.dwMOV_ECX_VALUE = (DWORD)&LoadLibrary;
                ic.wCALL_ECX = 0xD1FF;
                ic.byINT3 = 0xCC;
                memcpy(ic.szDllPath, szDllPath, strlen(szDllPath));
                if (!WriteProcessMemory(hProcess, pBaseAddr, &ic, sizeof(ic), &dwWrite))
                {
                    OutputDebugString(TEXT("写入shellCode失败"));
                    break;
                }
                ctxOld.ContextFlags = CONTEXT_FULL;
                GetThreadContext(hThread, &ctxOld);
                // 修改context的Eip
                ctxNew = ctxOld;
                ctxNew.ContextFlags = CONTEXT_FULL;
                ctxNew.Eip = (DWORD)pBaseAddr;
                if (!SetThreadContext(hThread, &ctxNew))
                {
                    OutputDebugString(TEXT("SetThreadContext 失败"));
                    return -1;
                }
                break;
            case EXCEPTION_DEBUG_EVENT:
                if (dbgEvent.u.Exception.ExceptionRecord.ExceptionCode == EXCEPTION_BREAKPOINT)
                {
                    if (bIsSystemBreak)
                    {
                        bIsSystemBreak = FALSE;
                        break;
                    }
                    ctxOld.ContextFlags = CONTEXT_FULL;
                    GetThreadContext(hThread, &ctxOld);
                    if (!SetThreadContext(hThread, &ctxOld))
                    {
                        OutputDebugString(TEXT("SetThreadContext失败"));
                        return -1;
                    }
                    if (!ContinueDebugEvent(dbgEvent.dwProcessId, dbgEvent.dwThreadId, DBG_CONTINUE))
                    {
                        OutputDebugString(TEXT("忽略该调试问题"));
                        return -1;
                    }
                    // ExitProcess(0);
                    return 0;
                }
                break;
            }
            if (!ContinueDebugEvent(dbgEvent.dwProcessId, dbgEvent.dwThreadId, DBG_EXCEPTION_NOT_HANDLED))
            {
                OutputDebugString(TEXT("继续线程事件失败"));
                return 0;
            }
        }//while
    }
    return 0;
}
```

# 四 挂起线程注入

# 五 钩子注入

```c
#include <windows.h>
#define DLL_EXPORT _declspec(dllexport) 

#ifdef __cplusplus
extern "C"{
#endif // __cplusplus
    
DLL_EXPORT BOOL SetHook(DWORD dwThreadId);
DLL_EXPORT BOOL UnHook();

#ifdef __cplusplus
}
#endif // __cplusplus

// 共享段（注意：必须在声明同时初始化，否则共享段创建失败）
#pragma  data_seg(".SHARE") 
HMODULE g_hDllModule = NULL;
HHOOK  g_hHook = NULL;
#pragma  data_seg()
#pragma  comment(linker,"/section:.SHARE,rws") 

LRESULT CALLBACK KeyBoardProc(int nCode, WPARAM wParam, LPARAM lParam);

BOOL WINAPI DllMain(HMODULE hDllModule, DWORD dwReason, LPVOID lpreserved){
    switch (dwReason){
    case DLL_PROCESS_ATTACH:
        g_hDllModule = hDllModule;
        break;
    case DLL_PROCESS_DETACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
        break;
    }
    return TRUE;
}

// 在系统范围内装载钩子函数
BOOL SetHook(DWORD dwThreadId){
    if (dwThreadId > 0){
        g_hHook = SetWindowsHookEx(WH_KEYBOARD, KeyBoardProc, g_hDllModule, dwThreadId);
        if (g_hHook != NULL){
            OutputDebugString("Hook Success.");
            return TRUE;
        }
    }
    return FALSE;
}

// 下载钩子函数
BOOL UnHook(){
    if (g_hHook != NULL){
        if (UnhookWindowsHookEx(g_hHook)){
            g_hHook = NULL;
            return TRUE;
        }
    }
    return FALSE;
}

LRESULT CALLBACK KeyBoardProc(int nCode, WPARAM wParam, LPARAM lParam)
{
    MessageBox(NULL, "KEY PRESS", "钩子注入成功", MB_OK);
    if (nCode > 0){
        switch (wParam){
        case HC_ACTION:
            break;
        case HC_NOREMOVE:
            break;
        }
    }
    return CallNextHookEx(g_hHook, nCode, wParam, lParam);
}
```

# 六 APC注入

```c
#include <windows.h>

int WINAPI WinMain(_In_ HINSTANCE hInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ LPSTR lpCmdLine, _In_ int nShowCmd)
{
    BOOL bRet = FALSE;
    STARTUPINFO si = { 0 };
    si.cb = sizeof(STARTUPINFO);
    PROCESS_INFORMATION pi = { 0 };
    PBYTE pAlloc = NULL;
    DWORD dwWrite = 0;
    CHAR * szDllPath = "D:\\TestDll.dll";
    bRet = CreateProcess(TEXT("D:\\HelloWorld.exe"), NULL, NULL, NULL,
        FALSE, CREATE_SUSPENDED, NULL, NULL, &si, &pi);
    if (bRet)
    {
        pAlloc = (PBYTE)VirtualAllocEx(pi.hProcess, NULL, 0x100, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
        if (pAlloc != NULL)
        {
            bRet = WriteProcessMemory(pi.hProcess, pAlloc, szDllPath, strlen(szDllPath) + 1, &dwWrite);
            if (bRet)
            {
                if (QueueUserAPC((PAPCFUNC)LoadLibraryA,
                    pi.hThread,
                    (ULONG_PTR)pAlloc))
                {
                    OutputDebugString("QueueUserAPC成功!");
                }
                else
                {
                    OutputDebugString("QueueUserAPC失败!");
                }
            }
        }
        ResumeThread(pi.hThread);
    }
    return 0;
}
```



# 七 远程线程来注入DLL

```c
const char * pDllPath = "E:\\CodeFile\\VC++\\MyInjectTools\\Release\\TestDll.dll";
DWORD dwSize = strlen(pDllPath) + 1;
DWORD dwWrite = 0;
FARPROC fLoadLibraryA = GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
HANDLE hProcess = OpenProcess(PROCESS_CREATE_THREAD | PROCESS_QUERY_INFORMATION |
                              PROCESS_VM_OPERATION | PROCESS_VM_WRITE |
                              PROCESS_VM_READ, FALSE, dwProcessID);
if (hProcess != NULL){
    LPVOID p = VirtualAllocEx(hProcess, NULL, dwSize, 
                              MEM_REVERSE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);
    if (p != NULL){
        // 写入到远程进程
        if (WriteProcessMemory(hProcess, p, pDllPath, dwSize, &dwWrite)){
            HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, 
                           (LPTHREAD_START_ROUTINE)fLoadLibraryA, p, 0, NULL);
            WaitForSingleObject(hThread, INFINITE);
            VirtualFreeEx(hProcess, p, 0, MEM_RELEASE);
            return true;
        }
    }
}
```

# 八 DLL劫持
把我们知道进程必然会载入的一个DLL替换掉。
缺点:不能自动适应版本变化。
优点：利用白加黑把自己的dll运行起来且不会被杀毒软件报毒。
九 导入表注入
十 输入法注入
十一 网络LSP注入


**八 API拦截**

**1 Inline钩子**

***\*通过覆盖代码来拦截API\**
**

**2 IAT钩子**

***\*通过修改模块的导入段来拦截API\**
**

**3 EAT钩子**