# Platform migration baseline

Recorded on 9 October 2026 for roadmap task T01, before application code changes.

## Build results

Host: Windows x64, Windows version reported by `dotnet --info`: 10.0.29680.
SDK selected by the repository: .NET SDK 10.0.100 / MSBuild 18.0.2.
There is no `global.json`; the project targets .NET 8 even though the installed SDK selected for this run is newer. The release workflow selects SDK 8.0.x, so CI SDK parity remains a separate check.

Run from the repository root:

```powershell
dotnet build src/Eve-O-Preview/Eve-O-Preview.csproj --configuration Release -p:EVEOTARGET=Windows -p:AssemblyVersion=8.0.2.0 -p:FileVersion=8.0.2.0 -v:minimal
dotnet build src/Eve-O-Preview/Eve-O-Preview.csproj --configuration Release -p:EVEOTARGET=Linux -p:AssemblyVersion=8.0.2.0 -p:FileVersion=8.0.2.0 -v:minimal
dotnet build src/Eve-O-Mock/Eve-O-Mock.csproj --configuration Release -v:minimal
```

| Build | Result | Warnings |
|---|---|---|
| Windows application | Passed, 0 errors | 4 |
| Linux/Wine application, compiled on Windows | Passed, 0 errors | 1 |
| Mock application | Passed, 0 errors | 0 |

Existing warnings:

- Both application variants: CS0168, unused `ex` in `View/Implementation/MainForm.cs:84`.
- Windows variant: CS0169, unused Wine fields `_enableWineCompatabilityMode`, `_bashLocation`, and `_wmctrlLocation` in `Services/Implementation/WindowManager.cs:18`–20.

Both application variants write to `bin/net8.0-windows8.0`; building the Linux variant overwrites the Windows output. Rebuild the desired variant before launching it. The Linux build is a Windows executable intended for Wine, not a native Linux executable.

## Runtime checklist

These are expected behaviors derived from the code and documented options, not observed test passes. No application was launched during this baseline build check.

Use an isolated working directory for the preview application to keep its relative `EVE-O-Preview.json` separate from an existing installation. Launch two or more instances of `src/Eve-O-Mock/bin/Release/net8.0-windows/ExeFile.exe`. Each receives a random `Mock ...` title and color; the default executable filter includes `exefile`.

Set cycle group client names to the actual mock titles, or empty the group's order dictionary to cycle all clients. The mock currently cannot change its title interactively and does not animate its contents, so it is insufficient to verify title-change handling or live capture freshness by itself.

| Check | Procedure / expected behavior | Status |
|---|---|---|
| Single instance | Start preview twice; second process exits without another settings window | Pending: Windows desktop |
| Discovery | Start mocks before and after preview; each produces a thumbnail | Pending: Windows desktop |
| Removal | Close a mock; its thumbnail and active-client entry disappear | Pending: Windows desktop |
| Title changes | Rename a controlled test window; thumbnail label and hotkey mapping update | Pending: controllable window or EVE |
| Activation | Click each thumbnail; corresponding client becomes foreground | Pending: Windows desktop |
| Minimize / switch out | Ctrl-click minimizes; Ctrl+Shift-click returns to the tracked external application | Pending: Windows desktop |
| Per-client hotkeys | Configure actual titles in `ClientHotkey`; each shortcut activates its client | Pending: Windows desktop |
| Cycle ordering | Configure forward/backward keys and order; verify wraparound and missing clients | Pending: Windows desktop |
| Cycle exclusion | Shift-click a thumbnail; indicator updates and cycling skips it | Pending: Windows desktop |
| Visibility | Exercise active-client hiding, disabled previews, and hiding on focus loss | Pending: Windows desktop |
| Resize / zoom / move | Exercise drag, resize, hover zoom, snapping, and location locking | Pending: Windows desktop |
| Layout tracking | Enable tracking and per-client layouts; verify restoration and persistence | Pending: Windows desktop |
| Settings / tray | Change settings, close/reopen, and exercise minimize-to-tray and explicit exit | Pending: Windows desktop |
| Renderer fallback | Restart with `WineCompatibilityMode` false and true; compare DWM and static previews | Pending: Windows desktop |
| Capture freshness | Observe an animated client, including occlusion and minimization | Pending: animated window or EVE |
| Login clients | Test multiple windows titled `EVE`, login hiding, and login cycle handling | Pending: controllable windows or EVE |
| Wine integration | Verify activation, static capture, hotkeys, and tool discovery on Linux; include Flatpak if supported | Pending: Linux/Wine host |
| Real-client parity | Verify switching, overlays, capture, and layouts against actual EVE clients | Pending: EVE session |

## Completion status

T01 is in progress: compilation is verified with the local SDK; desktop behavior, CI SDK parity, and Linux/Wine runtime checks remain outstanding. Keep the roadmap task unchecked until the baseline results cover the required behavior.
