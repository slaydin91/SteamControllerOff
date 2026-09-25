# Steam Controller Off

Turn off a Steam Controller (2026) from a script. The result is the same as when you hold the Steam button for 5 seconds.

The scripts use PowerShell only. You do not need Python, Steam Big Picture, or admin rights.

## Files

| File | Purpose |
|---|---|
| `sc2_off.ps1` | Standalone script. Run it directly. |
| `SteamControllerOff.ps1` | Reusable function `Stop-SteamController`. Dot-source it into your own scripts. |

## Usage

### Standalone

```powershell
# Show the matching devices only. Sends nothing.
powershell -NoProfile -ExecutionPolicy Bypass -File .\sc2_off.ps1 -List

# Turn off the controller
powershell -NoProfile -ExecutionPolicy Bypass -File .\sc2_off.ps1
```

### In a PowerShell script

```powershell
. "$PSScriptRoot\SteamControllerOff.ps1"

if (Stop-SteamController) { Write-Host 'Controller off.' }
else                      { Write-Host 'No controller found.' }
```

Add `-Verbose` to see the result for each slot.

### In a CMD or batch script

```bat
powershell -NoProfile -ExecutionPolicy Bypass -Command ". '%~dp0SteamControllerOff.ps1'; if (Stop-SteamController) { exit 0 } else { exit 1 }"
```

`%ERRORLEVEL%` is `0` when the controller turned off, and `1` when no controller was found.

### Double-click shortcut

Make a file named `sc2_off.cmd` in the same folder, with this line:

```bat
@powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0sc2_off.ps1"
```

## How it works

1. **Load the Windows HID functions.** PowerShell cannot talk to HID devices directly. The script uses `Add-Type` to compile a small C# class that calls `kernel32.dll` (`CreateFile`) and `hid.dll` (`HidD_GetPreparsedData`, `HidP_GetCaps`, `HidD_SetFeature`). It compiles the class only once per session.

2. **Find the device.** The script uses `Get-PnpDevice` to find HID devices with Valve's vendor ID `28DE`. It keeps only these devices:
   - the controller itself: `PID_1302` or `PID_1303`
   - the controller slots on the Puck: `PID_1304` or `PID_1305`, interfaces `MI_02` to `MI_05`

3. **Build the device path.** The script changes the instance ID into the path format that `CreateFile` needs. For example, `HID\VID_28DE&...` becomes `\\?\HID#VID_28DE&...#{4d1e55b2-f16f-11cf-88cb-001111000030}`.

4. **Open each interface.** The script opens each interface with read and write access, in shared mode, so it does not interfere with Steam.

5. **Check the collection.** The script reads the HID capabilities of the interface. It continues only if the usage page is `FF00` (Valve's vendor-defined collection) and the feature report has space for the command. It skips all other collections, such as the mouse and keyboard ones.

6. **Send the command.** The script sends a 64-byte feature report with `HidD_SetFeature`. The Puck passes the report to the controller.

   | Byte | Value | Meaning |
   |---|---|---|
   | 0 | `01` | Report ID |
   | 1 | `9F` | Valve message ID `ID_TURN_OFF_CONTROLLER` |
   | 2 | `04` | Payload length |
   | 3–6 | `6F 66 66 21` | ASCII `off!`, the confirmation value |
   | 7–63 | `00` | Padding |

7. **Report the result.** The slot your controller is on accepts the report. Empty slots reject it with Win32 error 31. This is expected, and the script ignores these errors. The script closes every handle it opens.

The script changes no controller settings. It writes nothing to disk. It sends only one command, and only to Valve devices.

## Example output

```
> .\sc2_off.ps1 -List
HID\VID_28DE&PID_1304&MI_05&COL03\...  UsagePage FF00  FeatureLen 64
HID\VID_28DE&PID_1304&MI_04&COL03\...  UsagePage FF00  FeatureLen 64
HID\VID_28DE&PID_1304&MI_02&COL03\...  UsagePage FF00  FeatureLen 64
HID\VID_28DE&PID_1304&MI_03&COL03\...  UsagePage FF00  FeatureLen 64

> .\sc2_off.ps1
SetFeature failed (Win32 31): HID\VID_28DE&PID_1304&MI_05&COL03\...
SetFeature failed (Win32 31): HID\VID_28DE&PID_1304&MI_04&COL03\...
Sent: HID\VID_28DE&PID_1304&MI_02&COL03\...
SetFeature failed (Win32 31): HID\VID_28DE&PID_1304&MI_03&COL03\...
```

## Troubleshooting

| Error | Cause | Action |
|---|---|---|
| `No Steam Controller / Puck HID interface found.` | The Puck or controller is not connected. | Connect the Puck. Turn on the controller. |
| `Win32 31` on some slots | Those Puck slots have no controller. | None. This is normal. |
| `Win32 31` on all slots | The controller is not connected to the Puck. | Turn on the controller, then run again. |
| `Win32 5` or `Win32 32` | Another program has the interface open. | Exit Steam, then run again. |
| `Win32 87` | The device rejected the report format. | Open an issue and include the `-List` output. |
| `Cannot convert value "-1073741824"...` | An old copy of the script. Windows PowerShell 5.1 reads `0xC0000000` as a negative Int32. | Use the current version. It uses `[uint32]3221225472`. |

## Requirements

- Windows 10 or 11
- Windows PowerShell 5.1 or PowerShell 7
- Steam Controller (2026), connected with the Steam Controller Puck, USB, or Bluetooth

## Compatibility

Tested on Windows with a Steam Controller (2026) connected through the Puck.

Message ID `0x9F` comes from Valve's shared controller constants in SDL. The Valve 2015 Steam Controller also uses this ID. USB and Bluetooth connections use the same frame format, but they are not tested yet.

## References

- [SDL `controller_constants.h`](https://github.com/libsdl-org/SDL/blob/main/src/joystick/hidapi/steam/controller_constants.h): Valve feature report message IDs
- [SDL `SDL_hidapi_steam_triton.c`](https://github.com/libsdl-org/SDL/blob/main/src/joystick/hidapi/SDL_hidapi_steam_triton.c): Steam Controller (2026) HID driver and feature report frame
- [sc2-research](https://github.com/CouchTurtle/sc2-research): Steam Controller (2026) and Puck USB IDs and protocol notes

## Disclaimer

This project is not affiliated with Valve. Steam and Steam Controller are trademarks of Valve Corporation. Use at your own risk.

## License

MIT
