# Remote debug WindowsPhone 8.1 kernel from Windows 8.1

Add the following files in the target phone (coming from C:\Program Files (x86)\Windows Phone Kits\8.1\MSPackages\Merged\arm\fre\Microsoft.MS_KDNETUSB_ON.MSN.MainOS.spkg):
```
	\windows\System32\kd_02_5143.dll
	\windows\System32\kdcom.dll
	\windows\System32\kdnet.dll
	\windows\System32\kdstub.dll
	\windows\System32\kdusb.dll
```

Get the IP of the host computer (Example 192.168.1.16).

Reboot the phone in *Mass storage* mode, see [Key combinations of Windows Phone 8.1](https://github.com/fredericGette/Lumia520/blob/main/content/windows_keys/README.md).  
Then open an Admin Command Prompt.  
And change the BCD configuration of the target phone:
```
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD /dbgsettings net HOSTIP:192.168.1.16 PORT:50000 KEY:1.2.3.4
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD -set {default} debug on
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD -set {default} dbgtransport kdnet.dll
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD -set {dbgsettings} busparams 1
```

>[!NOTE]
>In this example *F:* is the letter of the *MainOS* disk drive.  You must change it to match the correct letter.  
>And you must also change the *HOSTIP* to match the IP address of the host computer.  

In the host computer start C:\Program Files (x86)\Microsoft Windows Phone 8 KDBG Connectivity\bin\VirtEth.exe  
Warning: this version of VirtEth requires "Virtual Machine Network Services" which is not available in Windows 10+. Please use Windows 8.1  
For Windows 10 use VirthEth_RS1 as indicated [below](#Debug-with-windows-10).

In case of error, check that "Virtual Machine Network Services" is enable only in one network connection:

![Network](network.png)
![Network2](network2.png)

Reboot the target phone.

VirtEth should display the following messages:
![virteth](virteth.jpg)

You can now start the debugger client in the host computer:

C:\Program Files (x86)\Windows Kits\8.1\Debuggers\x64\kd.exe -y C:\Symbols -k net:port=50000,key=1.2.3.4

![kd](kd.jpg)

>[!NOTE]
>You can also use windbg.exe with the same option as kd.exe  

# Others BCD configuration:

To display boot menu:
```
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD /set {bootloadersettings} bootmenupolicy legacy
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD /set {bootmgr} displaybootmenu on
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD /set {bootmgr} timeout 60
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD /displayorder {default}
```

To display errors:
```
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD -set {globalsettings} booterrorux Standard
```

# Debug with windows 10
Thanks to [Leway213](https://github.com/Leeway213/BSP-aw1689/blob/master/doc/Dev%20Guide.md#2-debug-with-a-virtual-net-over-usb)   
Create a virtual switch in Hyper-V  
![vitualSwitch](HyperV.png)
Then start VirthEth_RS1.exe

# Usefull kd/windbg command

`.sympath C:\Symbols`  
`.reload`  

See the stack trace  
`k`  

See the list of loaded modules  
`lm`  
See only loaded module matching a pattern  
`lm m oempanel`  
See the list of loaded modules sorted by name  
`lm sm`  

Let the windows phone running  
`g`  

Break  
`ctrl+c`  

Create an "unresolved" break point (you can set the break even before the loading of the module)    
`bu wp81wiimote!EvtIoDeviceControl`  

List break point  
`bl`  

Display the address and name of the symbol "poDebug" of the module "nt"  
`x /2 nt!poDebug`

Enter into memory (0x821d3628) the double-word values (4 bytes) specified (0xFFFFFFFF)    
`ed 821d3628 0xFFFFFFFF`

Activate kernel debug messages  
`ed nt!poDebug 0x800`  
`ed nt!poDebug 0xFFFFFFFE`
`ed nt!Kd_DEFAULT_Mask 0xFFFFFFFF`  
`ed nt!Kd_WIN2000_Mask 0xFFFFFFFF`  

Set the r4 register for the current thread to 0xFFFFFFFF  
`r r4=FFFFFFFF`

Step over   
`p`

Step into  
`t`

Display memory  
`dc addr`  

Display the address of a module added to a value   
`?qci2c8930+0x2376`

Force load modules  
`!analyze`

Reboot the phone (use with the command line `-d` to break as soon as a kernel module is loaded)   
`.reboot`

# Notes

When a Windows phone is configured to use KDNET over USB, Media Transport Protocol (MTP) is disabled. On the host computer, in File Explorer, you will not see the usual phone folders (Documents, Music, Pictures, and the like).  

If you want to use MTP, turn off kernel-mode debugging for the phone.  

In mass storage mode:  

```
bcdedit /store F:\efiesp\efi\Microsoft\Boot\BCD /deletevalue {default} debug
``` 

