# Technical Verification Report: instrustar SDK (Linux + Raspberry Pi automation)

Repository analyzed: `https://github.com/instrustar-dev/SDK` (local checkout)

## 1) Supported devices (explicitly stated)

The repository README explicitly states:

> "The open source support ISDS205 ISDS210 ISDS220 and ISDS206 oscilloscopes."

Source: `README.md` line 2.

## 2) Linux support (official instructions, deps, commands, sudo/libusb)

The README has a dedicated Linux section with explicit instructions:

- Install libusb source package (`libusb-1.0.24.tar.bz2`), configure, and install.
- Copy Linux `.so` file(s) into system library paths like `/lib` or `/usr/lib`.
- Run the CLI demo with sudo.

Exact commands shown in README:

```text
tar xvjf libusb-1.0.24.tar.bz2
./configure --build=x86_64-linux --disable-udev
make install / sudo make install
sudo ./DllTest
```

README also says for Qt demo:

> "Note:To start Qt, please use sudo. This ensures that libusb can correctly detect the device"

In build config, `DllTest/CMakeLists.txt` links Linux builds against `usb-1.0`:

```cmake
target_link_libraries(${PROJECT_NAME} vdso usb-1.0)
```

This is direct evidence that Linux use depends on libusb (`usb-1.0`) and that sudo is documented by the project for demo execution/device detection.

## 3) CPU architecture support from provided binaries

### Binary files present under `SharedLibrary`

Linux:
- `SharedLibrary/Linux/X64/Debug/libvdso.so.1.0`
- `SharedLibrary/Linux/X64/Release/libvdso.so.1.0`

Windows:
- `SharedLibrary/Windows/X86/Debug/vdso.dll`
- `SharedLibrary/Windows/X86/Release/vdso.dll`
- `SharedLibrary/Windows/X64/Debug/vdso.dll`
- `SharedLibrary/Windows/X64/Release/vdso.dll`
- `SharedLibrary/Windows/AMR64/Debug/vdso.dll`
- `SharedLibrary/Windows/AMR64/Release/vdso.dll`

No Linux ARM (`arm`, `armhf`, `aarch64`) `.so` files are present in the repository tree.

### ELF architecture check on provided Linux `.so`

Commands run:

```bash
readelf -h SharedLibrary/Linux/X64/Release/libvdso.so.1.0
readelf -h SharedLibrary/Linux/X64/Debug/libvdso.so.1.0
```

Observed ELF header fields for both shared libraries:
- `Class: ELF64`
- `Machine: Advanced Micro Devices X86-64`

So the shipped Linux shared libraries are x86_64 ELF binaries.

## 4) Python automation capability

Python demo exists at `dome-Python/test.py` and uses `ctypes` with `WinDLL`:

```python
mdll = ctypes.WinDLL("VDSO.dll")
```

The script binds SDK functions including:
- initialization/device status (`InitDll`, `IsDevAvailable`, callbacks)
- acquisition setup (`SetOscChannelRange`, `GetOscSupportSamples`, `SetOscSample`)
- capture flow (`Capture`, `SetDataReadyCallBack`)
- data read (`ReadVoltageDatas`)

In its callback, it reads voltage samples, computes min/max, and starts next capture again.

`dome-Python/how to run.txt` explicitly discusses Windows DLL path examples and VC2019 runtime requirement.

Evidence indicates Python automation is provided via `ctypes` on Windows DLL (`WinDLL`) in this repo.

## 5) Data acquisition capability in SDK API

`SharedLibrary/VdsoLib.h` exports acquisition and waveform-read APIs, including:

- `GetMemoryLength()`
- `Capture(int length, char force_length)`
- `SetDataReadyCallBack(...)`
- `IsDataReady()`
- `ReadVoltageDatas(char channel, double* buffer, unsigned int length)`
- `IsVoltageDatasOutRange(char channel)`
- `GetVoltageResolution(char channel)`

It also exports trigger/configuration APIs (examples):
- `SetTriggerMode`, `SetTriggerStyle`, `SetTriggerSource`, `SetTriggerLevel`, `SetTriggerSenseDiv`, `SetPreTriggerPercent`, `TriggerForce`.

This is direct programmatic acquisition support (capture + readiness + buffer read of voltage data).

## 6) Linux examples in repository

README labels Linux-tested demos:

- `DllTest`: "a command line Demo written with c++, test in windows and ubuntu linux"
- `DllTestQt`: "a Demo written with Qt, test in windows and ubuntu linux"

`DllTest/DllTest.cpp` demonstrates:
- init DLL and register callbacks
- acquisition loop using `Capture` / `IsDataReady`
- waveform voltage retrieval via `ReadVoltageDatas`
- basic software analysis (min/max computation)
- re-triggering capture continuously

`DllTestQt/vmusbwave.cpp` demonstrates (Qt side):
- device callback wiring and capture start
- trigger configuration methods (mode/style/source/level)
- channel range/coupling controls
- repeated capture (`nextCapture`) and UI update signal

## 7) Suitability for automation (strict evidence-based)

Based on repository contents:

- Device connect/detect APIs exist (`IsDevAvailable`, callbacks).
- Programmatic capture and data retrieval APIs exist (`Capture`, `IsDataReady`, `ReadVoltageDatas`).
- Example code demonstrates continuous acquisition and software-side processing (min/max).

Therefore the SDK contents are sufficient to build an automated system that connects, acquires waveform voltage data, and analyzes in software.

## 8) Raspberry Pi compatibility (from provided binaries/code only)

- Provided Linux shared libraries are only in `SharedLibrary/Linux/X64/...` and identified as x86_64 ELF.
- No Linux ARM/aarch64 `.so` binaries are shipped.
- Build scripts contain platform-selection logic mentioning ARM/aarch64 compiler names, but that is build configuration logic, not shipped ARM binaries.

Therefore, from shipped artifacts alone, Raspberry Pi (ARM) direct run is not supported by included Linux `.so`; ARM-compatible binaries would need to be built/provided.

## 9) Final verdict

**Verdict: Suitable with modifications.**

Reason (strictly from repo evidence):
- SDK/API and demos clearly support automated acquisition workflows.
- Linux support is documented and examples are marked Ubuntu-tested.
- However, bundled Linux binaries are x86_64 only, with no provided ARM/aarch64 Linux `.so` for Raspberry Pi.

Main limitations/risks present in repository:
- Linux docs hard-code x86_64 libusb configure example.
- Python demo is Windows-oriented (`ctypes.WinDLL`) and marked Windows-tested in README.
- No packaged Raspberry Pi ARM Linux shared library included.
