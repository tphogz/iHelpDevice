# iHelpDevice

A Windows desktop tool for iPhone owners and repair technicians. It detects what state an iPhone is in, shows its identity information, controls power and Recovery mode, guides you into DFU mode, and installs firmware from signed IPSW files.

The interface is inspired by qFlipper: orange on dark, grid background, rounded frame, an animated log panel and a status bar.

> Status: experimental. Back up your device before updating firmware. See [Limitations](#limitations).

## Features

**Device detection**
- Detects Normal, Recovery and DFU mode automatically when a device is plugged in or removed.
- Shows an iPhone picture that matches the current mode.

**Device information** (click any row to copy its value)
- Device name and model identifier (for example `iPhone13,4`)
- iOS version
- Serial number
- ECID (hex)
- IMEI 1 and IMEI 2
- UDID
- Chip name and iBoot version when available (Recovery and DFU)

**Controls**
- Shutdown and Reboot (Normal mode), Reboot (Recovery mode)
- Enter Recovery mode (from Normal) and Exit Recovery mode (from Recovery)
- Refresh device information
- DFU Helper: step-by-step instructions with countdown timers for each iPhone generation, chosen automatically from the model identifier:
  - iPhone 8 and later (including SE 2nd and 3rd generation)
  - iPhone 7 and 7 Plus
  - iPhone 6s and older (physical Home button)

  The helper tells you when the device has actually entered DFU mode.
- Find IPSW: search firmware for a device identifier on [ipsw.me](https://ipsw.me), see which versions Apple still signs, download with a progress bar, or open an IPSW you already have.
- Update Firmware with two modes:
  - Retain user data (update)
  - Erase all (full restore, asks for confirmation)

**Log panel**
- Animated, collapsible log with timestamps removed in qFlipper style (`[APP] ...`), and a CLEAR button.

## Requirements

- Windows 10 or 11, x64
- .NET 8 SDK to build, .NET 8 Desktop Runtime to run
- A good USB cable, connected directly to the PC (avoid hubs)
- For Normal mode: the Apple USB driver and Apple Mobile Device service, which come with iTunes or the Apple Devices app
- For DFU mode: a WinUSB-compatible driver bound to the device in DFU. Without it, opening the device through libirecovery fails (error code -3) and DFU actions cannot talk to the device.

## Build

```
git clone <your repository url>
cd iHelpDevice
dotnet build -c Release
```

The output is `iHelpDevice.exe` in `bin/Release/net8.0-windows`. You can also open the project in Visual Studio 2022 and press F5.

NuGet package used: `iMobileDevice-net` 1.3.17. The build copies the native libraries from `runtimes/win-x64/native` next to the executable.

### Optional files

- `iHelpDevice.ico`: put it next to `iHelpDevice.csproj` and the build uses it as the application icon.
- `libirecovery.dll`: if you have a newer build of libirecovery, put it next to the `.csproj` and it is copied to the output folder. The app looks for `libirecovery-1.0.dll`, `libirecovery.dll` and `irecovery.dll`, in that order, next to the exe and in `runtimes/win-x64/native`.

## Usage

1. Start iHelpDevice and plug in the iPhone. Unlock it and tap Trust if it is in Normal mode.
2. The left panel shows device information and the right panel shows the device and its buttons.
3. To install firmware:
   1. Press FIND IPSW, check the identifier, press SEARCH, pick a version marked SIGNED and press DOWNLOAD. Or choose an IPSW file you already have.
   2. Press UPDATE FIRMWARE, choose RETAIN USER DATA or ERASE ALL and press START UPDATE.
   3. Keep the device connected and the PC awake until the update finishes. The device restarts several times.
4. If the device will not turn on or is stuck, use DFU HELPER and follow the steps.

Button availability:

| Button | Normal | Recovery | DFU |
|---|---|---|---|
| Shutdown | yes | no | no |
| Reboot | yes | yes | no |
| Refresh | yes | yes | yes |
| Enter / Exit Recovery | Enter | Exit | no |
| DFU Helper, Find IPSW | yes | yes | yes |
| Update Firmware | yes | yes | yes |

## How it works

| Task | Method |
|---|---|
| Detect devices | usbmuxd for Normal mode, libirecovery and Win32 SetupAPI USB scanning for Recovery and DFU |
| Read device info (Normal) | libimobiledevice lockdown through iMobileDevice-net |
| Shutdown and Reboot (Normal) | libimobiledevice diagnostics relay service |
| Enter Recovery | lockdown `enter_recovery` |
| Exit Recovery and Reboot (Recovery) | libirecovery: set `auto-boot` to `true`, save environment, reboot |
| Firmware list and download | `https://api.ipsw.me/v4/device/<identifier>` |
| Firmware install | `idevicerestore.exe -y [-e] -i <ECID> <ipsw>` (or `-u <UDID>` in Normal mode) |

Recovery and DFU devices run no iOS, so model, iOS version and IMEI cannot be read there. They are taken from a small cache written the last time the same device (matched by ECID) was connected in Normal mode:

```
%LOCALAPPDATA%\iHelpDevice\device_cache.json
```

If a device was never connected in Normal mode, those fields show `N/A`.

## Project layout

```
iHelpDevice.csproj
Program.cs
NativeCore/
  UsbmuxdWatcher.cs      device detection, info reading, iPhone model table
  IRecoveryCommands.cs   Recovery reboot and exit through libirecovery
  NativeBridge.cs
  UsbDkInstaller.cs
Services/
  DeviceControlService.cs  shutdown, reboot, exit Recovery
  RecoveryService.cs       enter Recovery
  IpswService.cs           ipsw.me search and download
  RestorePipeline.cs       runs idevicerestore
  DfuGuide.cs              DFU steps per iPhone generation
UI/
  Theme.cs                 colors, fonts, grid painting
  Controls/                themed controls (buttons, info panel, device canvas, log dock)
  Views/                   main window and dialogs (DFU helper, IPSW, update)
```

The internal C# namespace is still `CyberRestore` (the project's former name). The assembly and window title are `iHelpDevice`.

## Limitations

- DFU mode: only Refresh, DFU Helper, Find IPSW and Update Firmware are available. There is no reboot or exit action from DFU. To leave DFU, force restart the device with its hardware buttons.
- The bundled `idevicerestore.exe` is an older build. It may fail on very recent iOS versions or newest devices. In that case it reports an error and the device stays in Recovery. Restore it with Apple Devices or iTunes, or replace the binary with a newer build.
- The bundled `libirecovery` is also older and may not recognise newer chips by name. Information then falls back to Win32 USB data.
- Only firmware versions still signed by Apple can be installed. Older unsigned versions are rejected.
- `idevicerestore` output arrives in bursts, not in real time, because it is buffered when piped.
- Erase all deletes everything on the device. If Find My is on, you need the Apple ID password to activate it afterwards. This tool does not bypass Activation Lock.
- The project has not been tested across a wide range of devices and iOS versions.

## Disclaimer

iHelpDevice is an independent project. It is not affiliated with, endorsed by or sponsored by Apple Inc. iPhone, iPad, iOS and iTunes are trademarks of Apple Inc. Use it at your own risk. The authors are not responsible for data loss or damaged devices.

## Credits

- [libimobiledevice](https://libimobiledevice.org/) and [idevicerestore](https://github.com/libimobiledevice/idevicerestore)
- [libirecovery](https://github.com/libimobiledevice/libirecovery)
- [iMobileDevice-net](https://github.com/libimobiledevice-win32/imobiledevice-net)
- [ipsw.me](https://ipsw.me) API for firmware data
- [qFlipper](https://github.com/flipperdevices/qFlipper) for the visual inspiration

These components have their own licenses. Check each project before you redistribute binaries.

## License

Add your license here (for example MIT) and include a `LICENSE` file in the repository.
