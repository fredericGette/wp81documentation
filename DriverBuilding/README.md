# How to build a Windows Phone kernel driver

## Requirements

- [Install a telnet server on the phone](../telnetOverUsb/README.md), in order to configure and start/stop the kernel driver.
- Visual Studio 2015
- Windows Driver Kit.
- Windows Phone Driver Kit. 

## WPF or WDM driver ?

WPF are the "modern" way to code a "Plug'n'Play aware" kernel driver. But it can only be started by the Plug'n'Play Manager when the device related to the driver is detected.    

WDM are the "old" way to code a kernel driver. This kind of driver is called a "legacy" driver because it is coded in the "NT V4 style".  
But the big advantage of a WDM driver is that it can be started and stopped like a service and it doesn't require to be attached to device.  

## Building on the desktop

Open a Command Prompt in the folder containing your sources.  

Update the PATH environment variable:  
```
SET PATH=C:\Program Files (x86)\Microsoft Visual Studio 12.0\Common7\IDE\;C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\bin\x86_arm;C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\bin;%PATH%
```

Compile the sources:  
```
CL.exe /c /I"C:\Program Files (x86)\Windows Phone Kits\8.1\Include\km" /I"C:\Program Files (x86)\Windows Kits\8.1\Include\Shared" /I"C:\Program Files (x86)\Windows Kits\8.1\Include\km" /I"C:\Program Files (x86)\Windows Kits\8.1\Include\wdf\kmdf\1.11" /I"C:\Program Files (x86)\Windows Kits\8.1\Include\km\crt" /Zi /W4 /Od /D _ARM_ /D ARM /D _USE_DECLSPECS_FOR_SAL=1 /D STD_CALL /D DEPRECATE_DDK_FUNCTIONS=1 /D MSC_NOOPT /D _WIN32_WINNT=0x0602 /D WINVER=0x0602 /D WINNT=1 /D NTDDI_VERSION=0x06020000 /D DBG=1 /D _ARM_WINAPI_PARTITION_DESKTOP_SDK_AVAILABLE=1 /D KMDF_VERSION_MAJOR=1 /D KMDF_VERSION_MINOR=11 /Zp8 /Gy /Zc:wchar_t- /Zc:forScope- /GR- /FI"C:\Program Files (x86)\Windows Kits\8.1\Include\Shared\warning.h" /kernel /GF -cbstring /d1import_no_registry /d2AllowCompatibleILVersions /d2Zi+ driver.c
```

Generate the .sys:  
```
link.exe  /VERSION:"6.3" /INCREMENTAL:NO /LIBPATH:"C:\Program Files (x86)\Windows Phone Kits\8.1\lib\win8\km\ARM" /WX "C:\Program Files (x86)\Windows Kits\8.1\lib\winv6.3\UM\ARM\armrt.lib" "C:\Program Files (x86)\Windows Kits\8.1\lib\win8\KM\arm\BufferOverflowFastFailK.lib" "C:\Program Files (x86)\Windows Kits\8.1\lib\win8\KM\arm\ntoskrnl.lib" "C:\Program Files (x86)\Windows Kits\8.1\lib\win8\KM\arm\hal.lib" "C:\Program Files (x86)\Windows Kits\8.1\lib\win8\KM\arm\wmilib.lib" "C:\Program Files (x86)\Windows Kits\8.1\lib\wdf\kmdf\arm\1.11\WdfLdr.lib" "C:\Program Files (x86)\Windows Kits\8.1\lib\wdf\kmdf\arm\1.11\WdfDriverEntry.lib" "C:\Program Files (x86)\Windows Kits\8.1\Lib\winv6.3\km\arm\wdmsec.lib" /NODEFAULTLIB /NODEFAULTLIB:oldnames.lib /MANIFEST:NO /DEBUG /SUBSYSTEM:NATIVE,"6.02" /STACK:"0x40000","0x2000" /Driver /OPT:REF /OPT:ICF /ENTRY:"FxDriverEntry" /RELEASE  /MERGE:"_TEXT=.text;_PAGE=PAGE" /MACHINE:ARM /PROFILE /kernel /IGNORE:4078,4221,4198 /osversion:6.3 /pdbcompress /debugtype:pdata driver.obj
```

## Signing on the desktop

Even with "testsigning" activated by bcdedit.exe or with [CMD.Injector](https://github.com/fadilfadz01/CMD.Injector_WP8), you must sign your driver. Otherwise Windows Phone refuses to start it.  

> [!NOTE]
> The phone must be unlocked by a version 2.9+ of [WPinternals](https://github.com/ReneLergner/WPinternals) to be able to activate the "testsigning" in the BCD registry.  

_The following process is copied from a [StackOverflow](https://stackoverflow.com/a/201277)._  

Open a Command Prompt in the folder containing your unsigned .sys  

Create a self-signed Certificate Authority (CA):  
```
"C:\Program Files (x86)\Windows Kits\8.1\bin\x64\makecert.exe" -r -pe -n "CN=My CA" -ss CA -sr CurrentUser -a 
```

Import the CA certificate:  
```
certutil -user -addstore Root MyCA.cer
```

Create a code-signing certificate (SPC):  
```
"C:\Program Files (x86)\Windows Kits\8.1\bin\x64\makecert.exe" -pe -n "CN=My SPC" -a sha256 -cy end -sky signature -ic MyCA.cer -iv MyCA.pvk -sv MySPC.pvk MySPC.cer
"C:\Program Files (x86)\Windows Kits\8.1\bin\x64\pvk2pfx.exe" -pvk MySPC.pvk -spc MySPC.cer -pfx MySPC.pfx
```

Sign the driver:  
```
"C:\Program Files (x86)\Windows Kits\8.1\bin\x86\signtool.exe" sign /ph /fd "sha256" /f MySPC.pfx driver.sys
```

## Deploy and start the driver on the Phone

Copy the .sys file in the shared folder of the phone: `C:\Data\USERS\Public\Documents`  
> [!NOTE]
> When you connect your phone with a USB cable, this folder is visible in the Explorer of your computer. And in the phone, this folder is mounted in `C:\Data\USERS\Public\Documents` 

_The rest of the process must be executed on the phone within a telnet session._  

Configure the driver in the registry by using the sc.exe command:  
```
sc create myDriver type= kernel binPath= C:\Data\USERS\Public\Documents\driver.sys
```

### WDM driver

You can directly start the driver:  
```
sc start myDriver
```

...and stop it when you want:  
```
sc stop myDriver
```

### WDF driver

> [!WARNING]
> This section is a "work in progress".

You have to change the _start type_ of the driver (_demand\_start_ by default) to _system\_start_:  
```
sc config myDriver start= system
```

The driver will be started at the boot of the system:  
```
powertool -reboot
```

> [!NOTE]
> You can also activate a log of the drivers started at boot:  
> `bcdedit /set {current} bootlog yes`
> And after the boot, look at the content of the file `C:\Windows\ntbtlog.txt`  
