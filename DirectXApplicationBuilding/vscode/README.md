# How to build a Windows Phone DirectX 11 application with VSCode, LLVM, CMake and Ninja

This page explains how to build a Windows Phone 8.1 DirectX 11 application (an `.appx` package with a `CoreWindow`, Direct3D 11, Direct2D and DirectWrite) without Visual Studio.  

Compared to a [console application](../../ConsoleApplicationBuilding/vscode/README.md), a DirectX application needs four more things:
- the Visual Studio template is written in C++/CX, which clang doesn't support,
- the HLSL shaders must be compiled,
- the application must be packaged in an `.appx`,
- the `.appx` must be installed on the phone.

## Requirements

- Everything listed for the [console application with vscode](../../ConsoleApplicationBuilding/vscode/README.md#requirements): LLVM, CMake, Ninja, VSCode, the Windows Phone 8.1 SDK and the Visual Studio 11.0 (VS 2012) WPSDK headers and libraries.
- [Python 3](https://www.python.org/), to create the `.appx`.
- `fxc.exe` (the Direct3D shader compiler of the Windows SDK), for example in `C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0\x64\` (fxc version 6.3.9600 with its `d3dcompiler_47.dll`). It's optional: the project can use precompiled shaders instead.
- To install the application: an [SSH server on the phone](https://github.com/fredericGette/wp81Sshd) (or the [telnet server](../../telnetOverUsb/README.md) + [FTP](../../ftpOverUsb/README.md)) and `TH.exe` (Microsoft test tool copied into `C:\Windows\System32` by CMD Injector).

## Project structure

```
wp81LlvmDxTest/
├── .vscode/
│   └── c_cpp_properties.json      ← IntelliSense configuration (same as a console application)
├── src/                           ← C++ sources: plain C++ and WRL, no C++/CX
│   ├── pch.h
│   ├── App.cpp, App.h             ← main(), IFrameworkView, IFrameworkViewSource
│   ├── wp81LlvmDxTestMain.cpp/.h  ← rendering loop
│   ├── Common/
│   │   ├── DeviceResources.cpp/.h ← D3D11 device, swap chain, D2D context
│   │   ├── DirectXHelper.cpp/.h   ← log file, fail-fast, synchronous file read
│   │   └── StepTimer.h
│   └── Content/
│       ├── Sample3DSceneRenderer.cpp/.h
│       ├── SampleFpsTextRenderer.cpp/.h
│       └── ShaderStructures.h
├── shaders/
│   ├── SampleVertexShader.hlsl
│   ├── SamplePixelShader.hlsl
│   └── prebuilt/                  ← .cso used when fxc.exe isn't found
├── package/
│   ├── AppxManifest.xml           ← final manifest of the package
│   └── Assets/                    ← Logo.png, SmallLogo.png, SplashScreen.png, ...
├── tools/
│   ├── makeappx.py                ← creates the .appx (replaces MakeAppx.exe)
│   └── checkappx.py               ← validates the .appx with the Windows APPX API
├── CMakeLists.txt
└── CMakePresets.json
```

## From C++/CX to plain C++

The Visual Studio template uses C++/CX (`ref class`, `^`, `ref new`, `Platform::String`, PPL tasks). Clang can't compile it, so every Windows Runtime call goes through the ABI headers of the SDK (`Windows Phone Kits\8.1\Include\abi\windows.*.h`, namespace `ABI::Windows::...`) and the Windows Runtime C++ Template Library (WRL: `wrl.h`, `wrl/event.h`, `wrl/wrappers/corewrappers.h`).  
Direct3D, Direct2D, DirectWrite and WIC are plain COM: this part of the template (renderers, `DeviceResources`) keeps nearly the same code.

| C++/CX | Plain C++ / WRL |
|---|---|
| `CoreWindow^ window` | `Microsoft::WRL::ComPtr<ABI::Windows::UI::Core::ICoreWindow> window` |
| `ref class App sealed : IFrameworkView` | `class App : public RuntimeClass<ABI::Windows::ApplicationModel::Core::IFrameworkView>` + `InspectableClass(L"wp81LlvmDxTest.App", BaseTrust)` |
| `ref new App()` | `Make<App>()` |
| `window->Bounds` | `window->get_Bounds(&rect)` |
| `DisplayInformation::GetForCurrentView()` | activation factory `IDisplayInformationStatics` then `GetForCurrentView(&info)` |
| `x->Event += ref new TypedEventHandler<...>(this, &App::OnEvent)` | `x->add_Event(Callback<__FITypedEventHandler_2_...>(this, &App::OnEvent).Get(), &token)` |
| `DisplayOrientations::Landscape` | `DisplayOrientations_Landscape` |
| `Platform::String^` | `HSTRING` (`HString`, `HStringReference`) |
| `create_task(...).then(...)` | synchronous code |
| `throw Platform::Exception` | log + `ExitProcess` (see [Errors](#errors-without-c-exceptions)) |

### main

C++/CX `[Platform::MTAThread] int main(Platform::Array<Platform::String^>^)` becomes:

```cpp
int main()
{
	RoInitializeWrapper roInit(RO_INIT_MULTITHREADED);
	DX::ThrowIfFailed(roInit);

	ComPtr<ICoreApplication> coreApplication;
	DX::ThrowIfFailed(GetActivationFactory(HStringReference(RuntimeClass_Windows_ApplicationModel_Core_CoreApplication).Get(), &coreApplication));

	ComPtr<Direct3DApplicationSource> direct3DApplicationSource = Make<Direct3DApplicationSource>();
	DX::ThrowIfFailed(coreApplication->Run(direct3DApplicationSource.Get()));
	return 0;
}

IFACEMETHODIMP Direct3DApplicationSource::CreateView(IFrameworkView** view)
{
	return Make<App>().CopyTo(view);
}
```

`GetActivationFactory` is `ABI::Windows::Foundation::GetActivationFactory`: write the namespace (or `using namespace ABI::Windows::Foundation;`), otherwise the compiler only finds the 4-argument `GetActivationFactory` of WRL.

### IFrameworkView

```cpp
class App : public Microsoft::WRL::RuntimeClass<ABI::Windows::ApplicationModel::Core::IFrameworkView>
{
	InspectableClass(L"wp81LlvmDxTest.App", BaseTrust)
public:
	IFACEMETHODIMP Initialize(ABI::Windows::ApplicationModel::Core::ICoreApplicationView* applicationView) override;
	IFACEMETHODIMP SetWindow(ABI::Windows::UI::Core::ICoreWindow* window) override;
	IFACEMETHODIMP Load(HSTRING entryPoint) override;
	IFACEMETHODIMP Run() override;
	IFACEMETHODIMP Uninitialize() override;
	...
};
```

The rendering loop of `Run()` uses the dispatcher of the window received by `SetWindow`:

```cpp
ComPtr<ICoreDispatcher> dispatcher;
DX::ThrowIfFailed(m_window->get_Dispatcher(&dispatcher));
while (!m_windowClosed)
{
	if (m_windowVisible)
	{
		DX::ThrowIfFailed(dispatcher->ProcessEvents(CoreProcessEventsOption_ProcessAllIfPresent));
		m_main->Update();
		if (m_main->Render())
		{
			m_deviceResources->Present();
		}
	}
	else
	{
		DX::ThrowIfFailed(dispatcher->ProcessEvents(CoreProcessEventsOption_ProcessOneAndAllPending));
	}
}
```

### Events

`Callback<>` needs the exact type of the delegate. The simplest way to find it is to look at the declaration of the `add_<Event>` method in the ABI header, for example:

```
findstr /n "add_Closed" "C:\Program Files (x86)\Windows Phone Kits\8.1\Include\abi\windows.ui.core.h"
```

The handler receives the interfaces (`ICoreWindow*`, `ICoreWindowEventArgs*`, `IInspectable*` for `Object^`) and returns an `HRESULT`:

```cpp
EventRegistrationToken token;
DX::ThrowIfFailed(window->add_Closed(
	Callback<__FITypedEventHandler_2_Windows__CUI__CCore__CCoreWindow_Windows__CUI__CCore__CCoreWindowEventArgs>(
		this, &App::OnWindowClosed).Get(),
	&token));

HRESULT App::OnWindowClosed(ICoreWindow* sender, ICoreWindowEventArgs* args)
{
	m_windowClosed = true;
	return S_OK;
}
```

### Loading files without PPL tasks

The template loads the shaders with `ReadDataAsync` (`StorageFile` + PPL tasks). Here the file is read synchronously with `CreateFile2` / `ReadFile` in the installation folder of the package, whose path comes from `Windows.ApplicationModel.Package.Current.InstalledLocation`:

```cpp
ComPtr<IPackageStatics> statics;
ComPtr<IPackage> package;
ComPtr<IStorageFolder> folder;
ComPtr<IStorageItem> item;
HString path;
GetActivationFactory(HStringReference(RuntimeClass_Windows_ApplicationModel_Package).Get(), &statics);
statics->get_Current(&package);
package->get_InstalledLocation(&folder);
folder.As(&item);
item->get_Path(path.GetAddressOf());
// then CreateFile2(<path>\SampleVertexShader.cso, GENERIC_READ, FILE_SHARE_READ, OPEN_EXISTING, nullptr) ...
```

### Errors without C++ exceptions

Clang doesn't support C++ exceptions for 32-bit ARM Windows. The project is compiled with `/EHs-c-` and `_HAS_EXCEPTIONS=0`, and `DX::ThrowIfFailed(hr)` doesn't throw anymore: it writes the error in a log file and stops the application.

```cpp
void DX::FailFast(HRESULT hr, const char* file, int line)
{
	Log("FATAL: HRESULT 0x%08X at %s:%d", (unsigned)hr, file, line);
	ExitProcess((UINT)hr);
}
```

The log file is in the `LocalState` folder of the application (`Windows.Storage.ApplicationData.Current.LocalFolder`):  
`C:\Data\Users\DefApps\APPDATA\Local\Packages\<Identity Name>_<publisher hash>\LocalState\log.txt`  
This is very useful because there's no debugger: read this file with SSH or telnet when the application closes immediately.

`ExitProcess` is exported by `mincore.lib` but hidden by the `WINAPI_FAMILY_PHONE_APP` partition, so it's declared manually:

```cpp
extern "C" __declspec(dllimport) DECLSPEC_NORETURN void WINAPI ExitProcess(UINT exitCode);
```

## CMakePresets.json

Same preset as a console application, but with the Windows subsystem: `/SUBSYSTEM:WINDOWS` instead of `/SUBSYSTEM:CONSOLE` in `CMAKE_EXE_LINKER_FLAGS`.

## CMakeLists.txt

The differences with a console application:

```cmake
# WIN32: Windows subsystem (otherwise CMake appends /subsystem:console).
add_executable(${APP_NAME} WIN32 ${SOURCES})

target_compile_definitions(${APP_NAME} PRIVATE
    _WIN32_WINNT=0x0603
    WINAPI_FAMILY=WINAPI_FAMILY_PHONE_APP
    UNICODE
    _UNICODE
    _XM_NO_INTRINSICS_
    _HAS_EXCEPTIONS=0
)

target_compile_options(${APP_NAME} PRIVATE
    /clang:-fno-sized-deallocation
    /EHs-c-
    /Zc:threadSafeInit-
    ...
)

target_link_options(${APP_NAME} PRIVATE
    /APPCONTAINER
    /ENTRY:mainCRTStartup
    /NODEFAULTLIB:ole32.lib
)

target_link_libraries(${APP_NAME} PRIVATE
    mincore.lib
    runtimeobject.lib
    d3d11.lib
    dxgi.lib
    d2d1.lib
    dwrite.lib
    windowscodecs.lib
    uuid.lib
)
```

Why:
- `WIN32` + `/ENTRY:mainCRTStartup`: a graphical application whose entry point is `main()`, like the template.
- `/APPCONTAINER`: a Windows Phone 8.1 application runs in an AppContainer.
- `_XM_NO_INTRINSICS_`: the NEON code of DirectXMath uses types that only exist in the Microsoft compiler (`__n128`).
- `/EHs-c-` and `_HAS_EXCEPTIONS=0`: no C++ exceptions.
- `/Zc:threadSafeInit-`: the C runtime of Visual Studio 2012 (`msvcr110.dll`, the one already present on the phone) doesn't support the thread-safe initialization of static variables. Without this option the link fails with `undefined symbol: _Init_thread_header`.
- `/NODEFAULTLIB:ole32.lib`: some SDK headers ask for `ole32.lib`, which doesn't exist in the Windows Phone SDK. COM functions come from `mincore.lib`.
- Don't define `_DEBUG`: the debug version of the STL needs `msvcr110d.dll`, which isn't on the phone (`undefined symbol: _CrtDbgReportW`).

The C runtime of Visual Studio 2012 doesn't have the C99 functions either (`lround`, `round`...). Replace them, for example with `static_cast<long>(floorf(value + 0.5f))`.

The application uses `MSVCR110.dll` and `MSVCP110.dll` of the phone, so the package doesn't need the `Microsoft.VCLibs.120.00.Phone` framework of a Visual Studio build.

### Shaders

The Lumia GPUs use the feature level 9_3 of Direct3D: shaders must be compiled with the `vs_4_0_level_9_3` / `ps_4_0_level_9_3` profiles. LLVM has no HLSL compiler, and `dxc.exe` can't produce these profiles, so `fxc.exe` is used. If CMake doesn't find it, the `.cso` files of `shaders/prebuilt/` are copied instead.

```cmake
file(GLOB FXC_HINTS "C:/Program Files (x86)/Windows Kits/10/bin/*/x64" "C:/Program Files (x86)/Windows Kits/10/bin/*/x86")
find_program(FXC_EXECUTABLE fxc HINTS ${FXC_HINTS})

add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/SampleVertexShader.cso"
    COMMAND "${FXC_EXECUTABLE}" /nologo /T vs_4_0_level_9_3 /E main
            /Fo "${CMAKE_CURRENT_BINARY_DIR}/SampleVertexShader.cso" "${CMAKE_CURRENT_SOURCE_DIR}/shaders/SampleVertexShader.hlsl"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/shaders/SampleVertexShader.hlsl")
```

With these options, fxc 6.3.9600 produces `.cso` files identical, byte for byte, to the ones of the Visual Studio 2015 build.

> If `fxc.exe` is installed after the first `cmake --preset`, run `cmake --preset arm32-windows` again: CMake searches again for programs that weren't found.

### Package

The `.appx` is created at the end of the build by `tools/makeappx.py`, then validated by `tools/checkappx.py`:

```cmake
add_custom_command(OUTPUT "${APPX_FILE}"
    COMMAND "${Python3_EXECUTABLE}" -I "${CMAKE_CURRENT_SOURCE_DIR}/tools/makeappx.py"
        --manifest "${APPX_MANIFEST}" --out "${APPX_FILE}" ${APPX_PAYLOAD}
    COMMAND "${Python3_EXECUTABLE}" -I "${CMAKE_CURRENT_SOURCE_DIR}/tools/checkappx.py" "${APPX_FILE}"
    DEPENDS ${APP_NAME} ${SHADER_OUTPUTS} "${APPX_MANIFEST}" ${APPX_ASSETS}
    VERBATIM)
add_custom_target(appx ALL DEPENDS "${APPX_FILE}")
```

## The .appx package

An `.appx` is a ZIP file containing the application files plus:
- `AppxManifest.xml`: identity, executable, tiles, capabilities,
- `AppxBlockMap.xml`: the SHA-256 hash of each 64 KB block of each file,
- `[Content_Types].xml`: the MIME type of each file extension.

`makeappx.py` writes the same layout as `MakeAppx.exe` of the Windows 8.1 SDK: ZIP64 records, data descriptors, files stored without compression. The package isn't signed, like the Debug packages of Visual Studio, which is what the "deploy" of `TH.exe` expects.

`checkappx.py` opens the package with the APPX API of Windows (`IAppxFactory`, the API used by `MakeAppx.exe`) and reads every file. It fails if the ZIP layout, the hashes or the manifest are wrong:

```
Package opened: ZIP, blockmap and manifest are valid
  Assets\Logo.png                    2516 bytes, hashes OK
  ...
  wp81LlvmDxTest.exe               206336 bytes, hashes OK
OK: 9 payload files
```

### Manifest

`package/AppxManifest.xml` is the final manifest (Visual Studio generates it from `Package.appxmanifest`). Compared to the template:
- `ProcessorArchitecture="arm"` in `<Identity>`,
- `Executable="wp81LlvmDxTest.exe"` instead of `$targetnametoken$.exe`,
- `<Resource Language="en-us"/>` instead of `x-generate`,
- no `<Dependencies>` (no VCLibs framework).

```xml
<Identity Name="4eaac987-b40b-4fde-a9d3-b415812ccd74" Publisher="CN=frede" Version="1.0.0.0" ProcessorArchitecture="arm" />
<mp:PhoneIdentity PhoneProductId="4eaac987-b40b-4fde-a9d3-b415812ccd74" PhonePublisherId="00000000-0000-0000-0000-000000000000"/>
```

Use a new GUID for each application (and the same GUID for `Identity Name` and `PhoneProductId`).

### Assets

Visual Studio uses `MakePri.exe` to generate a `resources.pri` that selects `Assets\Logo.scale-240.png` when the manifest says `Assets\Logo.png`.  
Without MakePri, the images are simply renamed (`Logo.scale-240.png` → `Logo.png`): the package doesn't need a `resources.pri`, and the phone accepts it.

## Build

```
cmake --preset arm32-windows
cmake --build build
```

Or `[F7]` in VSCode. The result is `build\wp81LlvmDxTest.exe` and `build\wp81LlvmDxTest_1.0.0.0_ARM.appx`.

## Installation

Copy the package to the phone, then deploy it with `TH.exe`. The `-di` option deploys an unsigned (developer) package; if it's already deployed, TH.exe undeploys it and deploys it again.

```
scp build/wp81LlvmDxTest_1.0.0.0_ARM.appx user@<phone-ip>:C:/Data/Users/Public/wp81LlvmDxTest.appx
ssh user@<phone-ip> "TH.exe C:\Data\Users\Public\wp81LlvmDxTest.appx -di"
```

```
Device is developer unlocked.
TH: Completed Successfully.
```

The exit code of TH.exe is always 0: read its output.  
`TH.exe` without parameters lists the installed applications. A deployed application appears with `InstallType : DEBUG (Developer deploy)`:

```
 [30] AppTitle : wp81LlvmDxTest
 [30] AUMID : 4eaac987-b40b-4fde-a9d3-b415812ccd74_fkkem3zpb3x42!App
 [30] InstallType : DEBUG (Developer deploy)
 [30] InstallLocation : Internal
 [30] ProductID : {4EAAC987-B40B-4FDE-A9D3-B415812CCD74}
```

The package is unpacked in `C:\Data\SharedData\PhoneTools\<package full name>\`.  
To uninstall: `TH.exe {<ProductID>} -u`.

## Start the application

Start the application from the application list of the phone, or from an SSH/telnet session with `wp81LaunchApp.exe`, a small console application (built like the [console applications with vscode](../../ConsoleApplicationBuilding/vscode/README.md)):

```
ssh user@<phone-ip> "C:\Data\Users\Public\wp81LaunchApp.exe 4eaac987-b40b-4fde-a9d3-b415812ccd74_fkkem3zpb3x42!App"
```
```
CoCreateInstance(ApplicationActivationClient) failed: 0x80040154, trying DllGetClassObject
ActivateApplication(4eaac987-b40b-4fde-a9d3-b415812ccd74_fkkem3zpb3x42!App): hr=0x00000000 pid=1252
```

The parameter is the AUMID listed by `TH.exe`.

The tool calls `IApplicationActivationManager::ActivateApplication` of the COM class `ApplicationActivationClient` (`C:\Windows\System32\AppXExecutionClient.dll`):

```cpp
static const CLSID CLSID_ApplicationActivationClient =
    { 0x79CEEEBE, 0xBDE0, 0x4C05, { 0x9D, 0x92, 0x15, 0x5F, 0x78, 0xBB, 0x7D, 0x2B } };
static const IID IID_IApplicationActivationManager_ =
    { 0x2E941141, 0x7F97, 0x4756, { 0xBA, 0x1D, 0x9D, 0xEC, 0xDE, 0x89, 0x4A, 0x3D } };

struct IApplicationActivationManager_ : public IUnknown
{
    virtual HRESULT STDMETHODCALLTYPE ActivateApplication(LPCWSTR appUserModelId, LPCWSTR arguments, int options, DWORD* processId) = 0;
};

// CoCreateInstance(CLSID_ApplicationActivationClient, ...) fails with 0x80040154 (class not registered) in an SSH session,
// although the class is registered: create the object directly from the DLL.
HMODULE dll = LoadLibraryExW(L"C:\\Windows\\System32\\AppXExecutionClient.dll", nullptr, 0);
auto getClassObject = (HRESULT (STDAPICALLTYPE *)(REFCLSID, REFIID, void**))GetProcAddress(dll, "DllGetClassObject");
IClassFactory* factory = nullptr;
getClassObject(CLSID_ApplicationActivationClient, IID_IClassFactory, (void**)&factory);
IApplicationActivationManager_* manager = nullptr;
factory->CreateInstance(nullptr, IID_IApplicationActivationManager_, (void**)&manager);

DWORD pid = 0;
HRESULT hr = manager->ActivateApplication(aumid, L"", 0, &pid);
```

`LoadLibraryExW` is exported by `mincore.lib` but hidden by the `WINAPI_FAMILY_PHONE_APP` partition: declare it manually (`extern "C" __declspec(dllimport) HMODULE WINAPI LoadLibraryExW(LPCWSTR, HANDLE, DWORD);`).

> `TH.exe <package> -r` and `TestNavigationApi.exe LaunchModernApplication <AUMID> <TileID>` don't work from an SSH session, even with `TestNavigationApi.exe` copied to `C:\Data\Test\bin` from the test package `Microsoft.MS_TEST.MSN.Data.spkg`. TestNavigationApi first connects to the navigation server endpoint `System\NavigationServer_TestSessionManager` (created by `CoreUIComponents.dll`). This connection fails, and the tool exits without any message after `Entering testnavigationapi / LaunchModernApplication`.

Check that the application started with its log file (see [Troubleshoot](#troubleshoot)): each start adds a line `wp81LlvmDxTest started`.

## Troubleshoot

### The application closes immediately

Read the log of the application:

```
ssh user@<phone-ip> "type C:\Data\Users\DefApps\APPDATA\Local\Packages\<Identity Name>_<publisher hash>\LocalState\log.txt"
```

The publisher hash comes from the `Publisher` of the manifest (`fkkem3zpb3x42` for `CN=frede`), and it's also visible in the AUMID listed by `TH.exe`.  
If there's no log at all, the process probably failed before `main()`, for example because of a DLL that isn't on the phone. List the DLLs used by the executable:

```
"C:\Program Files\LLVM\bin\llvm-readobj.exe" --coff-imports build\wp81LlvmDxTest.exe
```

And check its header (Windows subsystem, AppContainer):

```
"C:\Program Files\LLVM\bin\llvm-readobj.exe" --file-headers build\wp81LlvmDxTest.exe
```
```
  Machine: IMAGE_FILE_MACHINE_ARMNT (0x1C4)
  Subsystem: IMAGE_SUBSYSTEM_WINDOWS_GUI (0x2)
    IMAGE_DLL_CHARACTERISTICS_APPCONTAINER (0x1000)
```

### Link errors

| Error | Solution |
|---|---|
| `undefined symbol: _Init_thread_header` | `/Zc:threadSafeInit-` |
| `undefined symbol: _CrtDbgReportW` | don't define `_DEBUG` |
| `could not open 'ole32.lib'` | `/NODEFAULTLIB:ole32.lib` |
| `use of undeclared identifier 'lround'` | C99 function missing in the VS 2012 C runtime: replace it |
| `use of undeclared identifier 'ExitProcess'` (or another Win32 function) | declare it manually if `mincore.lib` exports it, see the [console application troubleshooting](../../ConsoleApplicationBuilding/vscode/README.md#troubleshoot) |
