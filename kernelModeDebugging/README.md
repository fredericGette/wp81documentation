Add the following files in the target phone (coming from C:\Program Files (x86)\Windows Phone Kits\8.1\MSPackages\Merged\arm\fre\Microsoft.MS_KDNETUSB_ON.MSN.MainOS.spkg):
```
	\windows\System32\drivers\kdnic.sys
	\windows\System32\kd_02_5143.dll
	\windows\System32\kdcom.dll
	\windows\System32\kdnet.dll
	\windows\System32\kdstub.dll
	\windows\System32\kdusb.dll
```

Get the IP of the host computer (Example 192.168.1.16).

Change the BCD configuration of the target phone:
```
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD /dbgsettings net HOSTIP:192.168.1.16 PORT:50000 KEY:1.2.3.4
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD -set {default} debug on
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD -set {default} dbgtransport kdnet.dll
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD -set {dbgsettings} busparams 1
```

In the host computer start C:\Program Files (x86)\Microsoft Windows Phone 8 KDBG Connectivity\bin\VirtEth.exe  
Warning: this version of VirtEth requires "Virtual Machine Network Services" which is not available in Windows 10+. Please use Windows 8.1

In case of error, check that "Virtual Machine Network Services" is enable only in one network connection:

![Network](network.png)
![Network2](network2.png)

Reboot the target phone.

VirtEth should display the following messages:
![virteth](virteth.jpg)

You can now start the debugger client in the host computer:

C:\Program Files (x86)\Windows Kits\8.1\Debuggers\x64\kd.exe -k net:port=50000,key=1.2.3.4


