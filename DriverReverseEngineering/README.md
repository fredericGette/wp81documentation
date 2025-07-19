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

## Print formated GUID

```
import uuid
import idaapi
import idc

def format_guid_from_address(ea):
    """
    Reads 16 bytes from the given address (ea) in IDA Pro,
    assumes they represent a GUID, and formats them into
    the string {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}.

    Args:
        ea: The effective address (segment address) in IDA Pro
            where the 16 bytes of the GUID are located.

    Returns:
        A string representing the GUID in the specified format,
        or None if 16 bytes cannot be read from the address.
    """
    if not idaapi.is_loaded(ea):
        print(f"Error: Address 0x{ea:X} is not loaded in the database.")
        return None

    # Read 16 bytes from the specified address
    guid_bytes = idc.get_bytes(ea, 16)

    if guid_bytes is None or len(guid_bytes) != 16:
        print(f"Error: Could not read 16 bytes from address 0x{ea:X}.")
        return None

    try:
        # Create a UUID object from the bytes
        # The uuid.UUID constructor expects bytes in network byte order (big-endian).
        # Many GUIDs in Windows (and often elsewhere) are stored little-endian.
        # If your GUIDs appear reversed, you might need to reverse the bytes:
        # new_uuid = uuid.UUID(bytes_le=guid_bytes)
        # or new_uuid = uuid.UUID(bytes=guid_bytes[::-1]) if they're little-endian in memory.
        # For this example, we assume standard network byte order or that the bytes
        # are already in the correct order for uuid.UUID.
        new_uuid = uuid.UUID(bytes_le=guid_bytes)

        # Format and return the GUID string
        return f"{{{new_uuid}}}"
    except ValueError as e:
        print(f"Error creating UUID from bytes at 0x{ea:X}: {e}")
        return None

# --- Example Usage in IDA Pro Python console ---
# Replace this with the actual address where your GUID bytes are located.
# For example, if your GUID starts at 0x401000:
# guid_address = 0x401000
#
# Let's use a dummy address for demonstration if you don't have one handy.
# You MUST change this to a valid address in your IDB containing 16 bytes.
# For example, let's assume a sample GUID '00112233-4455-6677-8899-AABBCCDDEEFF'
# stored at 0x140010000 in little-endian in memory:
#
# If the bytes at 0x140010000 are:
# 33 22 11 00 55 44 77 66 88 99 AA BB CC DD EE FF
#
# You would need to read them and potentially reorder.
# The `uuid.UUID` constructor expects the bytes in big-endian for `bytes=`.
# If your GUID is stored little-endian in memory (common on Windows),
# you might need to use `uuid.UUID(bytes_le=guid_bytes)` or reverse the
# first three parts for `bytes=`.
#
# Let's assume for simplicity you have a section of 16 bytes
# that, when interpreted directly, forms a big-endian GUID.
# If they are little-endian (very common), you would need to adjust.

# Let's define a sample address. YOU NEED TO CHANGE THIS to your actual address.
# Make sure this address contains 16 consecutive bytes you expect to be a GUID.
target_address = idc.here() # Use current cursor address as an example

# Get the GUID string
guid_string = format_guid_from_address(target_address)

if guid_string:
    print(f"GUID found at 0x{target_address:X}: {guid_string}")
else:
    print(f"Failed to retrieve GUID from 0x{target_address:X}")
```