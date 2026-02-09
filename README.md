**EDR-Injector-Driver-POC**

![Windows](https://img.shields.io/badge/Windows-10%2F11-blue)
![Architecture](https://img.shields.io/badge/Architecture-x64%20%7C%20WOW64-green)
![License](https://img.shields.io/badge/License-Educational-orange)
![Driver](https://img.shields.io/badge/Type-Kernel%20Driver-red)


**A Windows kernel-mode driver** that automatically injects a DLL into newly created processes using **APC (Asynchronous Procedure Call)** 
injection. 
This driver **intelligently waits** for essential system DLLs to load before injecting, 
ensuring a stable injection environment.

This **proof-of-concept** demonstrates techniques commonly used by 
**EDR (Endpoint Detection and Response)** solutions for process instrumentation and monitoring.

## Overview

This driver implements DLL injection by monitoring **both process creation and image (DLL) loading events**. 
Rather than injecting immediately at process creation, the driver **intelligently waits** for critical system DLLs to be loaded:

**ntdll.dll** (native) - Required to obtain the `LdrLoadDll` function address  
**kernel32.dll** - Required because most injected DLLs depend on kernel32 APIs  
**WOW64 DLLs** (for 32-bit processes on 64-bit Windows) - wow64cpu.dll and the WOW64 ntdll.dll

Once these essential DLLs are loaded, the driver performs **"thunkless" APC injection** by directly calling 
`LdrLoadDll` from ntdll.


## Key Features
**Smart Injection Timing** | Waits for essential system DLLs before injecting (ntdll, kernel32, WOW64 DLLs)
**Image Load Monitoring** | Uses `PsSetLoadImageNotifyRoutine` to track DLL loading per process |
**Thunkless APC Injection** | Directly invokes `LdrLoadDll` from ntdll without requiring shellcode |
**WOW64 Support** | Handles both native x64 and WOW64 (32-bit on 64-bit) processes |
**Process State Tracking** | Maintains a linked list of processes and their injection state |
**Protected Process Detection** | Automatically skips injection for protected processes (PPL) |
**Export Table Walking** | Resolves `LdrLoadDll` address by parsing ntdll's export directory |

## How It Works
The driver uses **two kernel callbacks** to implement its injection logic:
### 1. Process Creation Callback (`PsSetCreateProcessNotifyRoutine`)
- Tracks new process creation and termination
- Maintains a linked list of `PROCESS_INFO` structures for each process
### 2. Image Load Callback (`PsSetLoadImageNotifyRoutine`)
- Monitors **every DLL** that loads in each process
- Tracks specific system DLLs using **bitflags**:
 - `SYSTEM32_NTDLL_LOADED` (0x0001)
 - `SYSTEM32_KERNEL32_LOADED` (0x0002)
 - `SYSWOW64_NTDLL_LOADED` (0x0004)
 - `SYSTEM32_WOW64CPU_LOADED` (0x0020)
- When ntdll.dll loads, **extracts** the `LdrLoadDll` function address from its export table
- Once all required DLLs are loaded, **queues an APC** for injection
### 3. APC Injection Process
When all essential DLLs are loaded:
1. **Create shared memory** - Uses `ZwCreateSection` and `ZwMapViewOfSection`
2. **Copy DLL path** - Writes the target DLL path into the shared section
3. **Queue APC** - Sets `LdrLoadDll` as the APC routine
4. **Execute** - The APC runs in the target process context, calling `LdrLoadDll` to load your DLL

Process Created → Track in List → Monitor DLL Loads → Wait for ntdll.dll + kernel32.dll
 ↓
 DLL Loaded ← Execute APC ← Queue APC ← All DLLs Ready!

## Requirements
### Development Environment
- **OS**: Windows 10/11 (x64)
- **IDE**: Visual Studio 2019 or later
- **WDK**: Windows Driver Kit (WDK) 10
- **SDK**: Windows SDK
### Runtime Environment
-  **OS**: Windows 10/11 (x64)
- **Privileges**: Administrator/System privileges
- **Signing**: Test signing enabled OR proper driver signing certificate

## Building
**Step 1:** Clone the repository
```bash
git clone https://github.com/yardenadi24/EDR-Injector-Driver-POC.git
cd EDR-Injector-Driver-POC
```
**Step 2:** Open in Visual Studio
- Open `InjectionDriver.sln` in Visual Studio
**Step 3:** Select Configuration
- Choose **Debug** or **Release**
- Select **x64** platform
**Step 4:** Build
- Press `F7` or go to **Build → Build Solution**
- The compiled driver (`.sys` file) will be in the output directory

## Installation & Usage
### Prerequisites
#### Option 1: Enable Test Signing (Recommended for Development)
```cmd
bcdedit /set testsigning on
```
Then **reboot** your system.

#### Option 2: Disable Driver Signature Enforcement (Temporary)
- Restart Windows
- Press `F8` during boot
- Select **"Disable Driver Signature Enforcement"**

### Loading the Driver
#### Method 1: Using `sc` Command
```cmd
# Create the service
sc create InjectionDriver type= kernel binPath= C:\path\to\InjectionDriver.sys
# Start the driver
sc start InjectionDriver
```

#### Method 2: Using OSR Driver Loader (Recommended for Testing)
1. Download [OSR Driver Loader](https://www.osronline.com/article.cfm%5Earticle=157.htm)
2. Select the driver file
3. Click **"Register Service"**
4. Click **"Start Service"**

### Unloading the Driver
```cmd
# Stop the driver
sc stop InjectionDriver
# Delete the service
sc delete InjectionDriver
```
## Configuration
### DLL Paths
The DLL paths are defined in `InjDrv.cpp` and `InjDrv.h`:
```cpp
#define DLL_PATH64 L"C:\\Windows\\System32\\HookDllx64.dll"
#define DLL_PATH86 L"C:\\Windows\\System32\\HookDllx86.dll"
```

### Setup Instructions
1. **Place your 64-bit DLL** at:
 ```
 C:\Windows\System32\HookDllx64.dll
 ```
2. **Place your 32-bit DLL** at (for WOW64 processes):
 ```
 C:\Windows\System32\HookDllx86.dll
 ```
3. The driver **automatically selects** the appropriate DLL based on target process architecture
4. To use **different paths**, modify the `#define` statements before compilation


### WARNING: Educational Proof-of-Concept
This is a **proof-of-concept** for **educational purposes only**.



