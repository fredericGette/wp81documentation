See below how to bypass the "developer license" check in Visual Studio 2015 and Windows 8.1  

Tested with Visual Studio Community 2015 (Version 14.0.25431.01 Update 3) and Windows 8.1 Pro 64-bit Operating System, x64-based processor.  

# Case #1: Your already have a "developer license", but it is expired and you get error 0x80245004 when you try to renew it.  
The "developer license check" occurs when you open your project 
![openProject](openProject.JPG)

or when you debug it on a windows phone device.  
![debugProject](debugProject.JPG)

![renewLicense](renewLicense.JPG)

And after having failed to contact the license server, you get this error message:  
![Error80245004](devLicenseError80245004.JPG)

![deployProject](deployProject.JPG)

## Solution

> [!CAUTION]
> __You may lost the usage of your computer !__  
> Make a copy of your files on another support before doing the following actions 
> and never do this if your disk contains important data !!

Replace file C:\Windows\SysWOW64\WSClient.dll (SHA256 Hash 8922E3DCD3EF74FA071A097DBC46CB6608D64EC0D4FAF362DBFD83E82EC01E1E)
![Wow64WSClient](Wow64WSClient.JPG)

by this [one](SysWOW64_expired_license/WSClient.dll) (SHA256 Hash 35ABD0312844D715EA96AEEDE3396DCB7B5C7C9008C0FAE072F6340DFD135C10)  

> [!NOTE]
> You have to change the owner and update the permissions on the existing WSClient.dll file before being able to rename or delete it. See [here](https://techcult.com/delete-files-protected-by-trustedinstaller/) for an example.

or use a disk editor (like [HxD](https://mh-nexus.de/en/hxd/)) to search and replace the following bytes in your disk (you should find them only in a WSClient.dll file):  
| Action | Value |
| --- | --- |
| search | `11 48 74 0a 48 75 24 bf 17 03 09 80 eb 0b 33 ff` |
| replace by | `11 48 74 0a 48 75 24 bf 00 00 00 00 eb 0b 33 ff` |

# Case #2: You don't have any "developer license".

You get almost the same error than when you have an expired license.

## Solution

> [!NOTE]
> This solution works in both case: with an expired license or without license.


> [!CAUTION]
> __You may lost the usage of your computer !__  
> Make a copy of your files on another support before doing the following actions 
> and never do this if your disk contains important data !!

Replace file C:\Windows\SysWOW64\WSClient.dll (SHA256 Hash 8922E3DCD3EF74FA071A097DBC46CB6608D64EC0D4FAF362DBFD83E82EC01E1E)
![Wow64WSClient](Wow64WSClient.JPG)

by this [one](SysWOW64_no_license/WSClient.dll) (SHA256 Hash 92DBDDB766E13DF6A4961EFE2E9FB2A2BF1CACE22452EB787F6C30BC1BFAC7AB)  

> [!NOTE]
> You have to change the owner and update the permissions on the existing WSClient.dll file before being able to rename or delete it. See [here](https://techcult.com/delete-files-protected-by-trustedinstaller/) for an example.

or use a disk editor (like [HxD](https://mh-nexus.de/en/hxd/)) to search and replace the following bytes in your disk (you should find them only in a WSClient.dll file):  
| Action | Value |
| --- | --- |
| search | `ff e8 f0 7a ff ff 8b 4d fc 8b c7 5f 33 cd 5b e8 8e` |
| replace by | `ff e8 f0 7a ff ff 8b 4d fc 31 c0 5f 33 cd 5b e8 8e` |


# Explanations

Visual Studio calls the function CheckDeveloperLicense of C:\Windows\SysWOW64\WSClient.dll and then - in case of failure of the check - it calls the function AcquireDeveloperLicense of this same library to renew the "developer license".
![wsclientFunctions](wsclientFunctions.JPG)

This is visible with windbg:  
- Open an Administrator Command Prompt.
- Go to `C:\Program Files (x86)\Windows Kits\8.1\Debuggers\x64` or where your windbg is located.
- Run the `tasklist` command to list the running processes and get the PID of Visual Studio.
![tasklist](tasklist.jpg)
- Start windbg with the PID of the process of Visual Studio in parameter: `windbg -p 5608`
- In the "Command" window of windbg, list the modules with the command `lm`. You should see WSClient in the result.
![windbgWsClient01](windbgWsClient01.JPG)
- Still in the "Command" window of windbg, list the exported functions of WSClient with `x WSClient!*`. You should see the function CheckDeveloperLicense.  
![windbgWsClient02](windbgWsClient02.JPG)
- Still in the "Command" window of windbg, enter the command `bu WSClient!CheckDeveloperLicense` to put a breakpoint at the beginning of the function CheckDeveloperLicense. You could check this in the "Breakpoints" window:
![windbgWsClient03](windbgWsClient03.JPG)
- Hit F5 (Go) to let Visual Studio runs freely.
![windbgWsClient04](windbgWsClient04.JPG)
- In Visual Studio, trigger the "developer license check" by trying to debug your project on a WindowsPhone device for example.
- Now you can see that the function CheckDeveloperLicense of WSClient is called (you can browse the different threads to find the one calling this function)  
![windbgWsClient05](windbgWsClient05.JPG)
- You can also hit F10 (Step Over) to follow step-by-step the execution of the function (beware! this is a very long function).

At the end, the function CheckDeveloperLicense returns an error code:
- ErrorCanceledHR 0x800704C7 
- ErrorNotFoundHR 0x80070490
- ErrorExpiredHR 0x80090317

You can get this information thanks to the Powershell Cmdlet `Get-WindowsDeveloperLicense`.  
This cmdlet displays the validity of the developer license and its expiration date.
![Get-WindowsDeveloperLicense01](Get-WindowsDeveloperLicense01.JPG)

You can find some information about this cmdlet in the following folders:  
```
C:\Windows\System32\WindowsPowerShell\v1.0\Modules\WindowsDeveloperLicense
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.Windows.DeveloperLicense.Commands
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.Windows.DeveloperLicense.Commands.Resources
```

In particular, `C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.Windows.DeveloperLicense.Commands` contains `Microsoft.Windows.DeveloperLicense.Commands.dll` which we can easily read with a .NET decompiler (like JustDecompile for example).

The decompiled version of the cmdlet is using some native functions exported by WSClient.dll:  
![NativeMethods](NativeMethods.JPG)

We can also take a look inside the GetWindowsDeveloperLicenseCommand to see how the native function is used.
![GetWindowsDeveloperLicenseCommand](GetWindowsDeveloperLicenseCommand.JPG)

And look also at the native constants to find the different return codes of the native functions.
![NativeConstants](NativeConstants.JPG)

> [!NOTE]
> The Powershell Cmdlet Get-WindowsDeveloperLicense is using C:\Windows\System32\WSClient.dll whereas Visual Studio is calling the Wow64 version of this library, but there is very little differences between the two. 

With all these information, we can now open the WSClient.dll in a disassembler (like [Ghidra](https://ghidra-sre.org/)) to find a way to change the return code of the function CheckDeveloperLicense.  
Fortunately, there's only one place - near the end of the function - where the return code "ErrorExpiredHR 0x80090317" is set.
![Ghidra01](Ghidra01.JPG)

By changing this value to 0x00000000 we can transform it into a "success":
![Ghidra02](Ghidra02.JPG)

The modification for the second case (no developer license) is a little more complicated.  
After several unsuccessful tests, I figured out how to always return 0x00000000 instead of an error code.  
As the return code is stored in the `eax` register, we have to set it to zero near the end of the function CheckDeveloperLicense.

See below the last disassembled portion of the function before modification (to obtain this with windbg, use the command `pt` to go to the "return" of the function):  
```
58aac587 8bc7            mov     eax,edi
5881c589 5f              pop     edi
5881c58a 33cd            xor     ecx,ebp
5881c58c 5b              pop     ebx
5881c58d e88ee5feff      call    WSClient!WSpTLRW+0x9b0 (5880ab20)
5881c592 8be5            mov     esp,ebp
5881c594 5d              pop     ebp
5881c595 c20400          ret     4
```
The instruction `mov eax,edi` copy the content of the register `edi` into `eax`.
And when we arrive here without a license, the registers have the following values:  
```
58aac587 8bc7            mov     eax,edi
0:000:x86> r
eax=00000001 ebx=00000000 ecx=fa5d048f edx=00c50000 esi=0071d968 edi=80070490
eip=58aac587 esp=0071d760 ebp=0071d924 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
```
We see that edi=80070490 corresponds to the value of the error code "ErrorNotFoundHR".

To force `eax` to zero, we can use the instruction `xor eax,eax`:  
```
58aac587 31c0            xor     eax,eax
5881c589 5f              pop     edi
5881c58a 33cd            xor     ecx,ebp
5881c58c 5b              pop     ebx
5881c58d e88ee5feff      call    WSClient!WSpTLRW+0x9b0 (5880ab20)
5881c592 8be5            mov     esp,ebp
5881c594 5d              pop     ebp
5881c595 c20400          ret     4
```

## We can do the same modification in the "System32" version of WSClient.dll
![Get-WindowsDeveloperLicense02](Get-WindowsDeveloperLicense02.JPG)

> [!CAUTION]
> __You may lost the usage of your computer !__  
> Make a copy of your files on another support before doing the following actions 
> and never do this if your disk contains important data !!

Replace file C:\Windows\System32\WSClient.dll (SHA256 Hash 10EBA5B72A25FFE1D184967050F6DC2233DE4EEA4A1B5D554DED8EC7885040B2)
![System32WSClient](System32WSClient.JPG)

by this [one](System32/WSClient.dll) (SHA256 Hash 32CB2F1E08FC3542CC0AC0C76EBA8B4CB4815D01999605BA3CCA7CFDF1991385)  
or use a disk editor (like [HxD](https://mh-nexus.de/en/hxd/)) to search and replace the following bytes in your disk (you should find them only in a WSClient.dll file):  
| Action | Value |
| --- | --- |
| search | `c9 74 0e ff c9 75 37 be 17 03 09 80 e9 8f d3 ff` |
| replace by | `c9 74 0e ff c9 75 37 be 00 00 00 00 e9 8f d3 ff` |
