# How to build a Windows Phone 8.1 driver with VSCode, LLVM, CMake and Ninja

This guide describes how to set up a kernel-mode **KMDF 1.11** driver project for Windows Phone 8.1 (ARM),
using the same toolchain as the [console application](https://github.com/fredericGette/wp81documentation/blob/main/ConsoleApplicationBuilding/vscode/README.md): `clang-cl` + `lld-link` driven by CMake and Ninja.  

## Requirements

- [Install a telnet server on the phone](https://github.com/fredericGette/wp81documentation/tree/main/telnetOverUsb#readme), in order to install and start the driver.
- [LLVM](https://releases.llvm.org/) (tested with LLVM 22), installed in `C:\Program Files\LLVM\`.  
  Make sure `clang-cl.exe` and `lld-link.exe` are present in `C:\Program Files\LLVM\bin\`.
- [CMake](https://cmake.org/download/) 3.20 or later.
- [Ninja](https://ninja-build.org/) build system, accessible from the PATH.
- [Visual Studio Code](https://code.visualstudio.com/) with the following extensions:
  - **C/C++** (Microsoft) — for IntelliSense
  - **CMake Tools** (Microsoft) — for CMake integration
- **Windows Driver Kit 8.1** (Windows Kits 8.1), providing:
  - the KMDF 1.11 headers: `C:\Program Files (x86)\Windows Kits\8.1\Include\wdf\kmdf\1.11\`
  - the kernel CRT headers: `C:\Program Files (x86)\Windows Kits\8.1\Include\km\crt\`
  - the ARM kernel import libraries: `C:\Program Files (x86)\Windows Kits\8.1\Lib\winv6.3\km\arm\`
  - the ARM KMDF libraries: `C:\Program Files (x86)\Windows Kits\8.1\Lib\wdf\kmdf\arm\1.11\`
- **Windows Phone 8.1 SDK**, providing `wdm.h` and the shared headers it includes, in:
  - `C:\Program Files (x86)\Windows Phone Kits\8.1\Include\um\`

## Project structure

```
wp81BmsFilter/
├── .vscode/
│   └── c_cpp_properties.json   ← IntelliSense configuration
├── src/
│   ├── Driver.h                ← includes <wdm.h> and <wdf.h>
│   ├── Driver.c                ← DriverEntry, EvtDeviceAdd, queues...
│   └── ...                     ← other .c files
├── CMakeLists.txt              ← build definition
└── CMakePresets.json           ← ARM32 toolchain preset
```

Drivers are written in C: the project is declared as `C` only, and every `src/*.c` file is compiled.

## CMakePresets.json

The preset selects `clang-cl` as the C compiler, targets `armv7-pc-windows-msvc`, uses Ninja as the generator, builds in `Release` and outputs build artifacts to the `build/` subdirectory.

Unlike the console application, the linker flags are not in the preset: all the driver-specific options are in `CMakeLists.txt`.

```json
{
    "version": 3,
    "configurePresets": [
        {
            "name": "arm32-kmdf",
            "displayName": "ARM32 KMDF 1.11 driver (Windows Phone 8.1)",
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build",
            "toolset": {
                "value": "host=x64",
                "strategy": "external"
            },
            "architecture": {
                "value": "arm",
                "strategy": "external"
            },
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_C_COMPILER": "C:/Program Files/LLVM/bin/clang-cl.exe",
                "CMAKE_LINKER": "C:/Program Files/LLVM/bin/lld-link.exe",
                "CMAKE_C_FLAGS": "--target=armv7-pc-windows-msvc",
                "CMAKE_TRY_COMPILE_TARGET_TYPE": "STATIC_LIBRARY"
            }
        }
    ],
    "buildPresets": [
        {
            "name": "arm32-kmdf",
            "configurePreset": "arm32-kmdf"
        }
    ]
}
```

> `CMAKE_TRY_COMPILE_TARGET_TYPE` is set to `STATIC_LIBRARY` so that CMake's compiler detection step does not attempt to link an executable for the cross-compilation target.

## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)

# Use the name of the current source directory as the project/target name
get_filename_component(APP_NAME ${CMAKE_CURRENT_SOURCE_DIR} NAME)

project(${APP_NAME} C)

# Kernel-mode KMDF 1.11 driver for Windows Phone 8.1 (ARM), built with LLVM
# (clang-cl + lld-link, see CMakePresets.json).
#   - KMDF 1.11 headers/libs and ARM kernel import libs: Windows Kits 8.1 (WDK)
#   - wdm.h and the shared headers it includes: Windows Phone Kits 8.1 (um)
set(WDK_ROOT "C:/Program Files (x86)/Windows Kits/8.1" CACHE PATH "Windows Kits 8.1 root")
set(PHONE_KIT_ROOT "C:/Program Files (x86)/Windows Phone Kits/8.1" CACHE PATH "Windows Phone Kits 8.1 root")
set(KMDF_MINOR 11)

# Drop CMake's user-mode defaults: /RTC1 needs a runtime, kernel32/user32 do not
# exist for kernel code, and a driver is not a console app with a manifest.
set(CMAKE_C_FLAGS_DEBUG "/Od")
set(CMAKE_C_FLAGS_RELEASE "/O2 /DNDEBUG")
set(CMAKE_C_STANDARD_LIBRARIES "")
set(CMAKE_CREATE_CONSOLE_EXE "")

file(GLOB SOURCES CONFIGURE_DEPENDS "src/*.c")
add_executable(${APP_NAME} ${SOURCES})

set_target_properties(${APP_NAME} PROPERTIES
    SUFFIX ".sys"
    MSVC_RUNTIME_LIBRARY "MultiThreaded"
)

target_compile_definitions(${APP_NAME} PRIVATE
    _ARM_=1
    _KERNEL_MODE=1
    WINNT=1
    NTDDI_VERSION=0x06030000
    _WIN32_WINNT=0x0603
    WINVER=0x0603
    KMDF_VERSION_MAJOR=1
    KMDF_VERSION_MINOR=${KMDF_MINOR}
    DEPRECATE_DDK_FUNCTIONS=1
    WDF_DEVICE_NO_WDMSEC_H   # wdmsec.h is not in the installed kits; no SDDL strings are used
)

# SYSTEM -> passed as /imsvc, so warnings inside the kit headers are not reported.
# um\minwin is only needed for apiset.h; it comes after um so um's ntdef.h wins.
target_include_directories(${APP_NAME} SYSTEM PRIVATE
    "${WDK_ROOT}/Include/wdf/kmdf/1.${KMDF_MINOR}"
    "${WDK_ROOT}/Include/km/crt"
    "${PHONE_KIT_ROOT}/Include/um"
    "${PHONE_KIT_ROOT}/Include/um/minwin"
)

target_compile_options(${APP_NAME} PRIVATE
    /W4
    /Zl                 # no default CRT library references
    /GS-                # no stack cookies (FxDriverEntry does not initialise them)
    /Gy
    /Z7
    /clang:-mno-implicit-float   # no compiler-generated VFP/NEON use in kernel code
)

target_link_directories(${APP_NAME} PRIVATE
    "${WDK_ROOT}/Lib/winv6.3/km/arm"
    "${WDK_ROOT}/Lib/wdf/kmdf/arm/1.${KMDF_MINOR}"
)

target_link_options(${APP_NAME} PRIVATE
    /DRIVER
    /SUBSYSTEM:NATIVE,6.03
    /ENTRY:FxDriverEntry
    /NODEFAULTLIB
    /MACHINE:ARM
    /DEBUG
    /OPT:REF
    /OPT:ICF
    /INCREMENTAL:NO
    /MANIFEST:NO
    /TSAWARE:NO
)

target_link_libraries(${APP_NAME} PRIVATE
    ntoskrnl.lib
    hal.lib
    BufferOverflowFastFailK.lib
    wdfldr.lib
    wdfdriverentry.lib
)
```

Key points:

**Target and output**
- The driver is built as an executable with the `.sys` suffix. `MultiThreaded` (static) is selected so that CMake does not add `/MD`, which would reference the user-mode DLL runtime.
- CMake's default user-mode settings are removed: `/RTC1` (needs a runtime), the default `kernel32.lib user32.lib ...` list, and the console subsystem.

**Preprocessor definitions**
- `_KERNEL_MODE`, `_ARM_` and `WINNT` select the kernel-mode, ARM variant of the WDK headers.
- `NTDDI_VERSION=0x06030000`, `_WIN32_WINNT=0x0603` and `WINVER=0x0603` target Windows 8.1 (Windows Phone 8.1).
- `KMDF_VERSION_MAJOR=1` / `KMDF_VERSION_MINOR=11` select KMDF 1.11, the framework version shipped with Windows Phone 8.1.
- `WDF_DEVICE_NO_WDMSEC_H` stops `wdf.h` from including `wdmsec.h`, which is not present in the installed kits. It is only needed for SDDL strings.

**Include directories**
- The WDK 8.1 installation used here has no `Include\km\wdm.h`, so `wdm.h` is taken from the Windows Phone Kit (`Include\um`). KMDF headers and the kernel CRT headers come from the WDK.
- `um\minwin` is only needed for `apiset.h`. It must come **after** `um`, so that `um\ntdef.h` is the one used.
- `SYSTEM` makes CMake pass these folders with `/imsvc`, so warnings in the kit headers are not reported (the equivalent of `/external:I` + `/external:W0` in the user-mode example).

**Compiler options**
- `/Zl`: no default CRT library is referenced in the object files.
- `/GS-`: no stack cookies. The security cookie is not initialised by `FxDriverEntry`.
- `/clang:-mno-implicit-float`: prevents clang from using VFP/NEON registers on its own (for example to copy structures), which is not allowed in kernel code without saving the floating-point state.
- `/Z7`: debug information in the object files, so that a `.pdb` is produced with `/DEBUG`.

**Linker options and libraries**
- `/DRIVER` and `/SUBSYSTEM:NATIVE,6.03` produce a kernel driver image for Windows 8.1.
- `/ENTRY:FxDriverEntry`: the entry point is provided by `wdfdriverentry.lib`. It binds the driver to the KMDF runtime (`WDFLDR.SYS`) and then calls your `DriverEntry`.
- `/NODEFAULTLIB`: only the libraries listed explicitly are linked.
- `ntoskrnl.lib` and `hal.lib` are the kernel import libraries, `BufferOverflowFastFailK.lib` provides the kernel fast-fail helper, `wdfldr.lib` and `wdfdriverentry.lib` are the KMDF stub libraries.
- The resulting `.sys` imports only `ntoskrnl.exe` and `WDFLDR.SYS`.

## IntelliSense configuration (.vscode/c_cpp_properties.json)

The include paths and defines must match the ones in `CMakeLists.txt`.

```json
{
  "configurations": [
    {
      "name": "WP8.1-ARM32-KMDF",
      "includePath": [
        "${workspaceFolder}/src",
        "C:/Program Files (x86)/Windows Kits/8.1/Include/wdf/kmdf/1.11",
        "C:/Program Files (x86)/Windows Kits/8.1/Include/km/crt",
        "C:/Program Files (x86)/Windows Phone Kits/8.1/Include/um",
        "C:/Program Files (x86)/Windows Phone Kits/8.1/Include/um/minwin"
      ],
      "defines": [
        "_WIN32",
        "_ARM_=1",
        "_KERNEL_MODE=1",
        "WINNT=1",
        "NTDDI_VERSION=0x06030000",
        "_WIN32_WINNT=0x0603",
        "WINVER=0x0603",
        "KMDF_VERSION_MAJOR=1",
        "KMDF_VERSION_MINOR=11",
        "DEPRECATE_DDK_FUNCTIONS=1",
        "WDF_DEVICE_NO_WDMSEC_H"
      ],
      "compilerPath": "C:/Program Files/LLVM/bin/clang-cl.exe",
      "compilerArgs": [
        "--target=armv7-pc-windows-msvc"
      ],
      "cStandard": "c11",
      "intelliSenseMode": "windows-clang-arm"
    }
  ],
  "version": 4
}
```

> Unlike the console application example, the include paths are listed one by one (no `/**`): a recursive search would mix user-mode and kernel-mode headers.

## Source files

The main header includes the WDM and KMDF headers, in this order:

```c
#include <wdm.h>
#include <wdf.h>
```

Your `DriverEntry` is a normal KMDF entry point (`WdfDriverCreate`, `EvtDriverDeviceAdd`...). Do not name it as the linker entry point: `FxDriverEntry` calls it.

Keep in mind that nothing from the user-mode runtime is available: no `printf`, no `malloc`, no C++. Use the kernel routines (`ExAllocatePoolWithTag`, `RtlCopyMemory`...) and ETW (`EtwRegister` / `EtwWriteString`) or `DbgPrint` for logging.

### Building from VSCode

Open the folder in VSCode. The CMake Tools extension detects `CMakePresets.json` automatically.

1. Open the Command Palette (`Ctrl+Shift+P`) and run **CMake: Select Configure Preset**, then choose **ARM32 KMDF 1.11 driver (Windows Phone 8.1)**.
2. Run **CMake: Configure** (or click the CMake status-bar button).
3. Run **CMake: Build** (`F7` or `Ctrl+Shift+B`).

### Building from the command line

```
cmake --preset arm32-kmdf
cmake --build build
```

The build produces `build\MyDriver.sys` (ARM Thumb-2, native subsystem 6.3) and `build\MyDriver.pdb`.

## Deployment

Copy `build\MyDriver.sys` to the shared folder of the phone: `C:\Data\USERS\Public\Documents`

> When you connect your phone with a USB cable, this folder is visible in Windows Explorer on your computer.

Then, from a telnet session on the phone:

1. Copy the driver to the drivers folder:
   ```
   copy C:\Data\USERS\Public\Documents\MyDriver.sys C:\Windows\System32\drivers\
   ```
2. Create the service (`Type=1` kernel driver, `Start=3` demand start):
   ```
   reg add HKLM\SYSTEM\CurrentControlSet\Services\MyDriver /v Type /t REG_DWORD /d 1
   reg add HKLM\SYSTEM\CurrentControlSet\Services\MyDriver /v Start /t REG_DWORD /d 3
   reg add HKLM\SYSTEM\CurrentControlSet\Services\MyDriver /v ErrorControl /t REG_DWORD /d 1
   reg add HKLM\SYSTEM\CurrentControlSet\Services\MyDriver /v ImagePath /t REG_EXPAND_SZ /d \SystemRoot\System32\drivers\MyDriver.sys
   ```
3. Attach the driver to its device. For a filter driver, add its service name to the `UpperFilters` (or `LowerFilters`) value (`REG_MULTI_SZ`) of the device instance under `HKLM\SYSTEM\CurrentControlSet\Enum\...`. If the value already exists, append to it; don't replace it.
4. Reboot the phone (or restart the device) so that the device stack is rebuilt with the driver.

The driver must be signed in a way the device accepts (or test signing enabled).  
See the [Signing on the computer section](https://github.com/fredericGette/wp81documentation/blob/main/DriverBuilding/README.md#signing-on-the-computer).

## Troubleshoot

### `wdmsec.h` not found

`wdf.h` includes `wdmsec.h`, which is not in the installed kits. Define `WDF_DEVICE_NO_WDMSEC_H` (only SDDL-based device security is lost).

### Conflicting definitions from `ntdef.h`

The `um\minwin` folder contains its own version of some headers. Put it **after** `um` in the include directories so that `um\ntdef.h` is found first.

### Definitions missing from the kit headers

Some definitions are not visible to kernel code in the Windows Phone kit headers, for example the `TRACE_LEVEL_*` constants of `evntrace.h`. Define your own equivalents in your header:

```c
/* Standard ETW levels (TRACE_LEVEL_* is not visible to kernel code in this kit's evntrace.h). */
#define MY_LEVEL_ERROR         2
#define MY_LEVEL_WARNING       3
#define MY_LEVEL_INFORMATION   4
```

### Unresolved `__security_cookie` / `__security_check_cookie`

The compiler generated stack cookies. Build with `/GS-`.

### Unresolved `__RTC_*`, or `/RTC1` errors

CMake added its default Debug flags. Override `CMAKE_C_FLAGS_DEBUG` (and `CMAKE_C_FLAGS_RELEASE`) as shown above.

### The linker looks for `kernel32.lib`, `libcmt.lib` or `msvcrt.lib`

Check that `CMAKE_C_STANDARD_LIBRARIES` is empty, and that `/Zl` and `/NODEFAULTLIB` are present.
