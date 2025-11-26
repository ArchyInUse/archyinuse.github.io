---
title: "Hacking Forager — Part 1: DLL Injection (A Fun, Technical Deep Dive)"
date: 2025-11-25
categories: [gamehacking, low-level]
tags: [dllinjection]     # TAG names should always be lowercase
---

Hey everyone!  
Welcome to the first entry of my **multi-part series on hacking _Forager_** — a wonderfully addictive 2D open-world crafting game by HopFrog.  
(_And if you haven’t already played it, you can find it on steam._ → [Forager on Steam](https://store.steampowered.com/app/751780/Forager/))

![Forager Preview](/assets/image/2025-11-25/Pasted image 20251122151551.png)

Forager throws you into a fast-paced loop of gathering, crafting, building and fighting. It’s simple on the surface but has a surprising amount of depth when you get into it.

And today, we’re going to completely trivialize _all_ of that with a cheat menu of my own creation, I'm very excited to be writing about this project as it contains some of my favorite subjects such as DLL Injections, low level code patching and messing with process memory for advantages in game!

This series will come in four parts:

```
1 — DLL Injection  ← you are here
2 - ImGui Implementation of a simple overlay
3 — Accessing Process Memory via Static Pointers
4 — Patching Game Behavior Manually (Code Injection)
```

Alright, enough intro — let’s talk DLLs.

---

### What’s a DLL, Really?
A **DLL (Dynamic-Link Library)** is essentially a collection of functions and code that can be easily accessed, regardless of language. 
Programs rely on DLLs for extra functions, shared code, and system-level features.

For example:
- `kernel32.dll` handles things like `CreateProcess`
- `user32.dll` handles things like keyboard input & cursor position

While many programming languages provide APIs that abstract these calls, under the hood they call DLLs that are windows / kernel managed to provide us with basic features such as opening a process or getting user input.

Just like EXEs, DLLs have an entry point — **`DllMain`** — which runs when the DLL is loaded into a process.

We won’t be exporting custom functions in this series, but we _will_ be using DLLs to run code inside the game.

---

### What Is DLL Injection?
DLL injection is the act of forcing another process to load your DLL, even though it normally never would.

The key idea:
> If your DLL loads inside the game’s memory, your code runs with the game’s permissions and can read, write, and execute anything the game can.

That’s why DLL injection has alot of uses in:
- cheat menus
- internal overlays
- game modding
- debugging tools
- trainers

And yes — any decent anti-cheat will detect this method easily.  
It’s also important to note that Forager is a single-player game with a modding API, I won’t be ruining anyone else’s experience with this project.

---

### The Game Plan
Here’s the exact workflow we’ll be implementing:
1. Write a test DLL to confirm the injection
2. Find the target process (`Forager.exe`) and get a handle to it
3. Allocate memory _inside the game_ for the path of our DLL
4. Write our DLL path into that memory
5. Call `LoadLibraryA` from within the game using `CreateRemoteThread`

This might look a tad scary for those who’ve not worked with DLLs, but once you’ve seen it once, you’ll realize it’s pretty straightforward. The only scary part about this is going to be the Windows API Naming Scheme, legacy code has it’s weaknesses and that’s the most glaring one.

---
#### Step 1 — Building a Test DLL

```cpp
#include <windows.h>

DWORD WINAPI MainThread(LPVOID lpParam) {
    MessageBoxA(NULL, "DLL Injection!", "DLL Injection!", NULL);
    return 0;
}

BOOL APIENTRY DllMain(HMODULE hModule,
    DWORD ul_reason_for_call,
    LPVOID lpReserved) {

    switch (ul_reason_for_call) {
    case DLL_PROCESS_ATTACH:
        CreateThread(NULL, NULL, MainThread, NULL, NULL, NULL);
        break;
    }
    return TRUE;
}
```

If you’ve never seen a DLL before, this probably looks like alien syntax, but here’s the TL;DR:

`DllMain(hModule, reason, lpReserved)`
- **`hModule`** → the base address of our DLL in the target process
- **`reason`** → tells you why the DLL is being called (we only care about `DLL_PROCESS_ATTACH`)
- **`lpReserved`** → we can safely ignore this argument.

if you're wondering why we're creating a new thread - because our DLL will eventually run a GUI and some logic loops, we don’t want to block the game’s main thread, otherwise Forager freezes every time you do something as our actions will block main thread activity.

Creating a separate thread is standard practice for cheat menus.

---
#### Step 2 — Preparing the Injector

Full source [here](https://github.com/ArchyInUse/ForagerCheatMenu/blob/master/Injector/Injector.cpp).

Before we do anything fancy, we enable debug privileges:
```cpp
bool enableDebugPrivilege() {
    HANDLE hToken = nullptr;
    if (!OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &hToken))
        return false;

    TOKEN_PRIVILEGES tp;
    LUID luid;
    if (!LookupPrivilegeValue(nullptr, SE_DEBUG_NAME, &luid)) {
        CloseHandle(hToken);
        return false;
    }

    tp.PrivilegeCount = 1;
    tp.Privileges[0].Luid = luid;
    tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;

    BOOL ok = AdjustTokenPrivileges(hToken, FALSE, &tp, sizeof(tp), nullptr, nullptr);
    CloseHandle(hToken);
    return ok && GetLastError() == ERROR_SUCCESS;
}
```

**Short Windows Internals Tangent**
Without this privilege enabled on our process, some processes simply won’t allow remote thread creation or memory operations from testing.

`SeDebugPrivilege` allows a process to inspect and modify other processes regardless of ownership of that process. Windows stores privileges inside access tokens, every  process has one. Similarly to usage of JWTs in modern web apps, it contains information of who you are and what you're allowed to do.

Even when running as administrator you must explicitly turn this privilege on. When requesting permission from the OS to get this privilege, it will check our access token to determine if an administrator ran the program, which is why we have to run the injector as administrator to get access to this privilege.

While this knowledge is not absolutely neccessary to understand the injector, I think it's important to understand this part as these topics become very relevant very quickly when dealing with an anti-cheat or any defensive system such as Anti Virus programs that we bypass in the pentesting world.

---
#### Step 3 — Finding the Forager Process
Simple process enumeration:
```cpp
vector<PROCESSENTRY32> listProcesses()
{
    vector<PROCESSENTRY32> list;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snapshot == INVALID_HANDLE_VALUE) return list;

    PROCESSENTRY32 pe;
    pe.dwSize = sizeof(pe);
    if (Process32First(snapshot, &pe))
    {
        do {
            list.push_back(pe);
        } while (Process32Next(snapshot, &pe));
    }

    CloseHandle(snapshot);
    return list;
}
```

Then filter:

```cpp
PROCESSENTRY32W foragerProcEntry{0};
bool found = false;

for (auto& proc : processes)
{
    string currExe = WCharToStr(proc.szExeFile);
    if (currExe == TARGETEXE) // TARGETEXE = "Forager.exe"
    {
        foragerProcEntry = proc;
        found = true;
    }
}
```

---
#### Step 4 — Opening the Process

```cpp
HANDLE hForager = OpenProcess(
    PROCESS_CREATE_THREAD |
    PROCESS_QUERY_INFORMATION |
    PROCESS_VM_OPERATION |
    PROCESS_VM_WRITE |
    PROCESS_VM_READ,
    FALSE,
    foragerProcEntry.th32ProcessID
);
```

We need:
- VM read/write
- ability to allocate memory
- ability to create a thread inside the game

---
#### Step 5 — Allocating Memory for the DLL Path

This is the part that’s most prone to confusion:

```cpp
auto strLen = MENUDLL.size() + 1; // +1 for null terminator
auto memRegion = VirtualAllocEx(hForager, NULL, strLen, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
WriteProcessMemory(hForager, memRegion, MENUDLL.c_str(), strLen, NULL);
```

**Adding 1 to the MENUDLL size**
`MENUDLL` is a `const std::string`, `LoadLibraryA` expects a c-style string, which means a null terminated string. `std::string` does not include null termination in it's size and we must make sure the null termination is there otherwise the program could be reading any data that happens to be in that memory region until a null byte appears, thereby making our DLL path corrupted with random data.

When allocating adding space for the null byte insures that when we call `MENUDLL.c_str()`, the entire string, null byte included gets written in the game memory.

**Why are we allocating memory inside the game?**
When we call `LoadLibraryA` remotely, it runs inside _Forager’s_ memory space — not our injector process.  
So the string pointer we pass must refer to memory that **exists inside Forager**.

The injector and the game do **not** share address spaces.

This also means that the string we pass it in my case is an absolute path and not a relative one, if you want to pass in a relative path you must make sure it’s within the same folder as the game executable or within one of the standard search locations for windows, you can check the standard search order [here](https://www.okta.com/identity-101/dll-hijacking/).


---
#### Step 6 — Calling LoadLibraryA Inside the Game

```cpp
// grab the LoadLibraryA address
auto loadLibraryAddr =
    GetProcAddress(GetModuleHandleA("kernel32.dll"), "LoadLibraryA");

// execute LoadLibraryA with our custom DLL as the argument
auto injectedThreadHandle = CreateRemoteThread(
    hForager,
    NULL,
    0,
    (LPTHREAD_START_ROUTINE)loadLibraryAddr,
    memRegion,
    0,
    NULL
);
```

And just like that:
Forager loads the DLL and our `DllMain` executes.

---
#### Step 7 — Checking Injection Success

```cpp
WaitForSingleObject(injectedThreadHandle, 5000);
DWORD exitcode = 0;
GetExitCodeThread(injectedThreadHandle, &exitcode);

if (exitcode != 0) {
    cout << "Thread handle: 0x" << hex << injectedThreadHandle << endl;
    cout << "Allocated memory at: 0x" << hex << memRegion << endl;
    cout << "Successfully injected the DLL." << endl;

    VirtualFreeEx(hForager, memRegion, 0, MEM_RELEASE);
    CloseHandle(injectedThreadHandle);
    CloseHandle(hForager);
}
else {
    cout << "Failed to inject DLL, exit code: {" << exitcode << "}." << endl;
}
```

This part of the code is a fail check that checks if our DLL was injected correctly, while we can rely on a message box which (spoiler alert!) does pop up after the injection happens, when developing our GUI, we don't want to rely on that to confirm injection.

We can grab the exit code using `GetExitCodeThread` to make sure our DLL was loaded. It's return value is going to be the `HMODULE` of the injected DLL, or in non-windows terms, we get an address (a pointer) to the base of the DLL that was just loaded relative to Forager's memory space.

If the injection fails, it will return `NULL`, a.k.a the value `0`.

---
### Important Project Settings

#### Subsystem → **Windows (/SUBSYSTEM:WINDOWS)**

![Windows Subsystem Setting](/assets/image/2025-11-25/Pasted image 20251124191718.png)

#### Configuration Type → **Dynamic Library (.dll)**

![Dynamic Library Setting](/assets/image/2025-11-25/Pasted image 20251124191803.png)

---

### Injection Success 🎉

![Injection Success](/assets/image/2025-11-25/Pasted image 20251124192044.png)

The DLL message box confirms it:  
Our code is officially running _inside_ Forager.
From here, we’ll be able to:
- read game memory
- write to game memory
- hook functions
- build GUI overlays
- patch game logic manually

This sets us up perfectly for Part 2: ImGui Implementation of a simple overlay.

Thanks for reading — Part 2 drops soon!
