# Tips to decompile a Windows Phone 8.1 kernel driver

1. Execute plugin [Driver Buddy Reloaded](https://github.com/fredericGette/wp81DriverBuddyReloaded) to find the WDF functions
2. Find the function calling `WdfDriverCreate`: this is the function `DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)`
3. Find the function calling `WdfDeviceCreate`: this is the function `EvtDriverDeviceAdd(WDFDRIVER Driver, PWDFDEVICE_INIT DeviceInit)`
4. Find a call to `WdfIoQueueCreate(WDFDEVICE Device,PWDF_IO_QUEUE_CONFIG Config,[optional] PWDF_OBJECT_ATTRIBUTES QueueAttributes,[out, optional] WDFQUEUE *Queue)`  
   - The 2nd parameter (not including the `WdfFdoInitQueryPropertyEx`) is a pointer to a `WDF_IO_QUEUE_CONFIG`  
     - The 7th DWORD is a pointer to `EvtIoDeviceControl(WDFQUEUE Queue, WDFREQUEST Request, size_t OutputBufferLength, size_t InputBufferLength, ULONG IoControlCode)`
5. Find a call to `WdfObjectGetTypedContextWorker(WDFOBJECT Handle, PCWDF_OBJECT_CONTEXT_TYPE_INFO TypeInfo)`
   - The 2nd parameter (not including the `WdfFdoInitQueryPropertyEx`) is a pointer to a `WDF_OBJECT_CONTEXT_TYPE_INFO`
     - The second DWORD of this structure is a pointer to a null terminated ASCII string of the name of the context.
     - The third DWORD is the size of the context in bytes.
   - Create a structure named as the context and having the correct size.
6. Find the function calling `MmGetSystemRoutineAddress` with the unicode string "EtwRegisterClassicProvider"
   - In the DriverEntry the 7th memory address before the call of this function is the `ETW_provider_GUID`:
```
      log_offset_02 = 0;
      pETW_provider_GUID = (int)&byte_4070B0 + 984;// ETW_provider_GUID
      dword_40BF28 = 0;
      dword_40BF38 = 0;
      byte_40BF3C = 1;
      byte_40BF3D = 0;
      word_40BF3E = 0;
      dword_40BF40 = 0;
      init_tracing_WMI_or_ETW(result, v5, v6, 0);
```
7. Find a call to `WdfDeviceAddQueryInterface(WDFDEVICE Device, PWDF_QUERY_INTERFACE_CONFIG InterfaceConfig)`
   - The structure before the call is a `WDF_QUERY_INTERFACE_CONFIG`
     - The second DWORD of this structure is a pointer to a exported direct-call INTERFACE
     - The third DWORD of this structure is a pointer to the GUID of the exported INTERFACE
       - The INTERFACE contains pointer to some functions that another driver can direct-call.
8. Find a call to `WdfIoTargetQueryForInterface(WDFIOTARGET IoTarget, LPCGUID InterfaceType,[out] PINTERFACE Interface, USHORT Size, USHORT Version,[optional] PVOID InterfaceSpecificData)`	
   - The 2nd parameter is the GUID of a INTERFACE exported by another driver.
   - The 3rd parameter is the a pointer to a structure containing the exported interface (an array of functions)
9. Execute plugin [FakePDB/Generate .PDB file (with function labels)](https://github.com/Mixaill/FakePDB) to generate a .PDB file for [kernel debugging with windbg](/kernelModeDebugging/README.md).

## Exporting All Pseudocode to a Text File

Go to File -> Script command... or press Alt+F7.  
In the "Script command" window, paste the following Python code, then click "run":

```
import ida_hexrays
import ida_funcs
import ida_kernwin
import os

def export_all_pseudocode():
    # Get the output file path, default to the same directory as the IDB
    output_file_path = ida_kernwin.ask_file(True, "*.c", "Save all pseudocode to...")

    if not output_file_path:
        print("[-] Operation cancelled by user.")
        return

    # Ensure the Hex-Rays decompiler is available
    if not ida_hexrays.init_hexrays_plugin():
        print("[-] Hex-Rays decompiler is not available.")
        return

    print("[+] Starting pseudocode export to: {}".format(output_file_path))

    with open(output_file_path, "w", encoding="utf-8") as output_file:
        # Iterate through all functions in the database
        for func_ea in Functions():
            try:
                # Get the function object
                func = ida_funcs.get_func(func_ea)
                if not func:
                    continue

                # Decompile the function
                cfunc = ida_hexrays.decompile(func)

                if cfunc:
                    # Write the function prototype and the pseudocode to the file
                    output_file.write("// Function: {}\n".format(ida_funcs.get_func_name(func_ea)))
                    output_file.write(str(cfunc))
                    output_file.write("\n\n")
                else:
                    print("[-] Failed to decompile function at 0x{:X}".format(func_ea))

            except ida_hexrays.DecompilationFailure as e:
                print("[-] Decompilation failed for function at 0x{:X}: {}".format(func_ea, e))
            except Exception as e:
                print("[-] An unexpected error occurred for function at 0x{:X}: {}".format(func_ea, e))

    print("[+] Successfully exported all pseudocode to: {}".format(output_file_path))

# Run the export function
export_all_pseudocode()
```
