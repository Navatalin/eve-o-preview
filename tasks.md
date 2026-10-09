# Native platform abstraction roadmap

## Objective

Separate shared application behavior from operating system integration so that Windows, the existing Wine implementation, and future native Linux implementations can use the same client cycling, layout, and configuration logic.

Start by preserving existing behavior. Native Linux support requires both a native backend and a Linux-capable frontend; extracting interfaces alone does not make the current WinForms/WPF application portable.

## Current baseline

- `src/Eve-O-Preview` targets `net8.0-windows8.0` and primarily uses WinForms, with WPF dispatcher infrastructure.
- `Program.cs` wires services through LightInject and messages through MediatR.
- `Services/Implementation/ThumbnailManager.cs` coordinates discovery, switching, previews, hotkeys, visibility, and layouts.
- `IWindowManager` exposes native handles, DWM types, image types, and conditional platform signatures.
- `WindowManager` combines Win32 integration with Wine activation through `bash` and `wmctrl`.
- `ProcessMonitor` discovers windows using process main-window handles.
- Preview rendering uses DWM or captured images selected by the Wine compatibility setting.
- `Eve-O-Mock` provides windows for manual testing; there is no automated test project.

## Phase 1 — Establish behavior and platform requirements

- [ ] **T01: Verify the existing build and record baseline behavior.**
  - Progress: Windows, Linux/Wine, and mock builds passed locally on 9 October 2026. See [baseline results and runtime checklist](docs/platform-baseline.md); desktop and Linux runtime checks remain pending.
  - Build the Windows and Linux/Wine configurations using the existing release properties.
  - Use multiple mock windows to check discovery, title changes, closing, activation, per-client hotkeys, cycle groups, preview visibility, resizing, zoom, and configuration persistence.
  - Record which checks require real EVE clients or a Linux host and complete those before claiming parity.
  - Acceptance: reproducible build commands and a behavior checklist with results and unresolved failures.

- [ ] **T02: Define the platform support and capability matrix.**
  - Distinguish Windows, Wine compatibility, native X11, and native Wayland.
  - List required and optional capabilities: discovery, focus tracking, activation, minimization, positioning, capture, global shortcuts, always-on-top previews, and client decoration changes.
  - Investigate the target Linux desktops, XWayland/Wine window visibility, portal availability, capture consent, minimized-window capture, and compositor-specific control APIs.
  - Decide the minimum Windows version and initial Linux support scope from prototype evidence.
  - Acceptance: documented support targets and explicit degraded behavior for unsupported capabilities.

## Phase 2 — Introduce stable contracts without changing behavior

- [ ] **T03: Define platform-neutral contracts and data types.** Depends on T02.
  - Introduce `IWindowDiscovery`, `IWindowControl`, `IPreviewProvider`, and `IGlobalShortcutService`.
  - Use an opaque `WindowId`, window metadata, portable geometry, and shortcut definitions rather than public `IntPtr`, WinForms `Keys`, DWM types, or `System.Drawing.Image`.
  - Define capability reporting and operation outcomes, including unsupported, denied, cancelled, unavailable, and closed-window cases.
  - Define async operations where permissions or external services require them, plus session disposal and event delivery/threading rules.
  - Design preview sessions to permit compositor thumbnails or frame streams; avoid requiring CPU bitmap copies for every backend.
  - Acceptance: contracts compile in a plain `net8.0` project without Windows desktop references or conditional platform signatures.

- [ ] **T04: Wrap existing Windows integration behind the contracts.** Depends on T03.
  - Adapt process discovery, Win32 window operations, DWM previews, screenshot fallback, and hotkey registration.
  - Keep native handles and resources inside the Windows implementation.
  - Preserve animation and client decoration options as optional backend features.
  - Remove direct native API access from shared callers, including caption changes currently performed by `ThumbnailManager`.
  - Acceptance: Windows behavior matches T01 and resources are released when windows or preview sessions close.

- [ ] **T05: Isolate the existing Wine compatibility implementation.** Depends on T03.
  - Move Wine-specific activation and configuration into a dedicated adapter.
  - Preserve the existing launch path while reporting missing tools and activation failures explicitly.
  - Avoid shell string construction from window titles; use structured arguments where possible and bound external-process waits.
  - Acceptance: the Wine path remains usable and its implementation no longer changes shared interface signatures.

- [ ] **T06: Select and register backends at application startup.** Depends on T04 and T05.
  - Move backend selection into the composition root.
  - Treat a Wine-hosted Windows application separately from a native Linux process; OS detection alone is insufficient.
  - Preserve existing configuration compatibility and document selection overrides.
  - Acceptance: shared services use the same contracts for both existing backends, with no platform conditionals in their callers.

## Phase 3 — Extract shared application behavior

- [ ] **T07: Create `EveOPreview.Core` and platform projects.** Depends on T06.
  - Move contracts and portable models into the core; isolate Windows and Wine adapters in appropriate projects.
  - Extract portable configuration and provide migration for UI-specific values such as fonts, colors, and shortcut representations.
  - Separate config file location/storage policy from application settings; retain compatibility with existing JSON files.
  - Acceptance: core builds without WinForms, WPF, User32, GDI, or DWM dependencies; existing configurations retain their values.

- [ ] **T08: Extract cycling, visibility, and layout policies from `ThumbnailManager`.** Depends on T07.
  - Separate client tracking, cycle ordering/exclusion, active-client transitions, preview visibility, and layout decisions from UI manipulation.
  - Keep UI dispatch and preview-window creation in the frontend.
  - Replace direct `DispatcherTimer` ownership in shared logic with an injected scheduler or explicit update entry point.
  - Preserve login-screen clients, duplicate titles, per-client overrides, priority clients, and delayed layout saves.
  - Acceptance: shared policies operate with fake platform services and the Windows frontend still passes the baseline checklist.

- [ ] **T09: Add focused automated tests for shared policies and adapter contracts.** Depends on T08.
  - Cover cycle wraparound, missing clients, exclusions, multiple login windows, focus transitions, visibility delays, and layout persistence.
  - Cover closed-window races, unsupported operations, cancellation, and session cleanup through fakes.
  - Acceptance: deterministic tests exercise meaningful behavior without requiring EVE or a desktop session.

## Phase 4 — Prove native Linux integration

- [ ] **T10: Prototype a native X11 backend.** Depends on T03 and T02; integrate after T07.
  - Use X11/EWMH for discovery, focus events, activation requests, and supported window state changes.
  - Use stable native window identifiers instead of title matching.
  - Evaluate XComposite/capture options against GPU-rendered Wine clients, occlusion, minimization, and multi-client performance.
  - Evaluate native global shortcut registration and conflicts.
  - Acceptance: a native Linux harness discovers actual Wine/EVE windows, displays previews, and switches clients without launching `bash` or `wmctrl`.

- [ ] **T11: Prototype a Wayland backend and document its limits.** Depends on T03 and T02; integrate after T07.
  - Evaluate ScreenCast portal/PipeWire capture and Global Shortcuts portal support on the chosen desktops.
  - Handle consent, session restoration where supported, cancellation, revocation, and stream termination.
  - Investigate discovery and activation separately; capture portals do not supply general window management.
  - Determine whether supported compositor integrations are necessary and whether their identifiers can be correlated with captured windows.
  - Do not assume XWayland provides access to all native Wayland windows or identical control behavior.
  - Acceptance: a documented, tested capability matrix and a working prototype for each claimed desktop; unsupported features are reported clearly.

- [ ] **T12: Decide the Linux implementation scope from prototype results.** Depends on T10 and T11.
  - Choose initial X11, Wayland, or mixed support and identify any desktop-specific adapters.
  - Decide whether native interop stays in C# or uses a small native helper based on actual capture and packaging needs.
  - Acceptance: an implementation decision records supported behavior, required dependencies, and remaining gaps.

## Phase 5 — Provide a native Linux frontend

- [ ] **T13: Choose and prototype the frontend approach.** Depends on T07 and T12.
  - Compare a shared cross-platform frontend, such as Avalonia, with retaining WinForms and adding a Linux frontend.
  - Prototype multiple borderless preview windows, frame rendering, overlays, hover zoom, dragging, scaling, and tray behavior.
  - Validate always-on-top and placement behavior on supported desktops, plus efficient rendering of backend preview sessions.
  - Acceptance: the chosen approach can deliver the required preview interactions on Windows and supported Linux targets.

- [ ] **T14: Implement the chosen frontend and integrate Linux backends.** Depends on T13 and T08.
  - Connect settings, previews, shortcuts, and layout actions to the shared core.
  - Show capability-dependent settings and actionable permission/session failures.
  - Preserve existing configuration and establish platform-appropriate defaults and storage locations.
  - Acceptance: native Linux execution works without hosting the preview application in Wine, and Windows parity is verified.

## Phase 6 — Validate, package, and document

- [ ] **T15: Run the cross-platform integration and performance matrix.** Depends on T14 and T09.
  - Test real clients, title changes, reconnects, rapid switching, closed windows, minimized clients, mixed DPI, multiple monitors, and suspend/resume.
  - Test portal refusal/revocation, missing dependencies, shortcut conflicts, and compositor differences.
  - Measure CPU/GPU use, memory, preview latency, and resource growth across repeated session creation and disposal.
  - Acceptance: results meet explicitly recorded budgets and unsupported behavior matches the capability matrix.

- [ ] **T16: Update CI and release packaging.** Depends on T15.
  - Build and test the core and each supported platform/frontend target.
  - Package native Linux dependencies and any helpers; keep Wine compatibility distribution if still supported.
  - Preserve version stamping and verify clean-machine installation/startup for each package.
  - Acceptance: reproducible Windows and native Linux artifacts with documented runtime requirements.

- [ ] **T17: Update user and contributor documentation.** Depends on T16.
  - Correct the runtime requirements and explain native Linux versus Wine compatibility support.
  - Document backend selection, desktop limitations, portal permission flows, configuration migration, build commands, and test procedures.
  - Acceptance: documentation matches the shipped platforms and observed behavior.

## Delivery checkpoints

1. **Existing behavior behind stable interfaces:** T01–T06.
2. **Portable, tested core with the existing frontend:** T07–T09.
3. **Evidence-backed native Linux support scope:** T10–T12.
4. **Native frontend and integrated backends:** T13–T14.
5. **Validated releases and documentation:** T15–T17.

Windows Graphics Capture is an optional follow-up if the selected frontend needs frame-based rendering. Keep DWM initially; replace or supplement it only after measuring compatibility, resource use, and rendering requirements.
