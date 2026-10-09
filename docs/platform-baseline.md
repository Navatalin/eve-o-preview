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

The initial build check did not launch the application. The Windows runtime results below were subsequently recorded on 9 October 2026; untested behaviors remain explicitly pending.

Use an isolated working directory for the preview application to keep its relative `EVE-O-Preview.json` separate from an existing installation. Launch two or more instances of `src/Eve-O-Mock/bin/Release/net8.0-windows/ExeFile.exe`. Each receives a random `Mock ...` title and color; the default executable filter includes `exefile`.

Set cycle group client names to the actual mock titles, or empty the group's order dictionary to cycle all clients. The mock currently cannot change its title interactively and does not animate its contents, so it is insufficient to verify title-change handling or live capture freshness by itself.

| Check | Procedure / expected behavior | Status |
|---|---|---|
| Single instance | Start preview twice; second process exits without another settings window | Passed: second process exited with code 0 |
| Discovery | Start mocks before and after preview; each produces a thumbnail | Passed: both mock titles appeared in the Active Clients UI; preview windows were observed |
| Removal | Close a mock; its thumbnail and active-client entry disappear | Passed: list changed from two entries to one, then zero; final capture had no thumbnail windows |
| Title changes | Rename a controlled test window; thumbnail label and hotkey mapping update | Pending: controllable window or EVE |
| Activation | Click each thumbnail; corresponding client becomes foreground | Partial: a static thumbnail click reported focus on its mock client; repeat for both clients and DWM |
| Minimize / switch out | Ctrl-click minimizes; Ctrl+Shift-click returns to the tracked external application | Pending: Windows desktop |
| Per-client hotkeys | Configure actual titles in `ClientHotkey`; each shortcut activates its client | Passed: Ctrl+F11 and Ctrl+F12 reported focus on the expected mock windows |
| Cycle ordering | Configure forward/backward keys and order; verify wraparound and missing clients | Partial: all-client forward/reverse cycling with two mocks passed; no-client cycling remained responsive; explicit order/missing entries pending |
| Cycle exclusion | Shift-click a thumbnail; indicator updates and cycling skips it | Pending: Windows desktop |
| Visibility | Exercise active-client hiding, disabled previews, and hiding on focus loss | Pending: Windows desktop |
| Resize / zoom / move | Exercise drag, resize, hover zoom, snapping, and location locking | Pending: Windows desktop |
| Layout tracking | Enable tracking and per-client layouts; verify restoration and persistence | Pending: Windows desktop |
| Settings / tray | Change settings, close/reopen, and exercise minimize-to-tray and explicit exit | Partial: width changed from 384 to 400, saved as `400, 216`, and restored in the UI after restart; normal exit passed; tray pending |
| Renderer fallback | Restart with `WineCompatibilityMode` false and true; compare DWM and static previews | Partial: both configurations launched; static capture matched mock color and label; DWM content verification pending |
| Capture freshness | Observe an animated client, including occlusion and minimization | Pending: animated window or EVE |
| Login clients | Test multiple windows titled `EVE`, login hiding, and login cycle handling | Pending: controllable windows or EVE |
| Wine integration | Verify activation, static capture, hotkeys, and tool discovery on Linux; include Flatpak if supported | Deferred by user: Linux/Wine host |
| Real-client parity | Verify switching, overlays, capture, and layouts against actual EVE clients | Pending: EVE session |

## Windows runtime session

Rebuilt the Windows variant before testing. The About page identified the running binary as `8.0.2.0 Windows`.

The application and its configuration were copied into an isolated temporary directory:

```text
C:\Users\lumbe\AppData\Local\Temp\eve-o-runtime-c79db0d4daff4b9ebcef162febc00e5f
```

Two mock windows were used: `Mock 4A8343A9` and `Mock 68CC4030`. One existed before preview startup and the second was launched afterwards. The test configuration used `OriginalAnimation`, an empty group-one order dictionary, and per-client Ctrl+F11/Ctrl+F12 shortcuts. Saved thumbnail positions were separated to avoid the initial default overlap with the settings window.

The primary run used `WineCompatibilityMode: false`. A second renderer run used `WineCompatibilityMode: true`, `ShowThumbnailFrames: true`, and full opacity, still on Windows. Static preview pixels matched the first mock's blue background and showed its correct overlay label. This verifies the Windows screenshot fallback, not Linux/Wine execution.

Runtime evidence was collected with the Computer Use skill: window inventories, accessibility trees, reported focused elements, UI screenshots, process exit status, and inspection of the isolated JSON file. Keyboard navigation and setting the numeric width field worked. Some mouse clicks did not produce the intended control change, and a thumbnail drag did not produce a confirmed location change. Screenshot ordering changed between captures, and input/capture geometry was inconsistent. These observations do not establish an application defect; mouse gestures and related settings require a repeatable follow-up test.

DWM thumbnail windows and labels were observed, but the automation captures did not conclusively show the mock content inside them. Do not count this as either a DWM render pass or failure without direct desktop verification.

All test preview and mock processes were closed normally at the end. No crash log was generated in the isolated directory. No application source changes were made, and the user's other applications were left running.

## Completion status

T01 is in progress: local builds and the basic Windows runtime checks above are verified. Mouse gestures, visibility options, tray behavior, layout restoration, title changes, live capture freshness, and real EVE behavior remain outstanding. CI SDK parity also remains unchecked. Linux/Wine runtime testing is deferred at the user's request. Keep the roadmap task unchecked until the required baseline coverage is complete.
