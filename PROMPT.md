# One-Shot Prompt

> Feed this prompt to Claude Code (or any comparable coding agent) to regenerate the entire app from scratch.

Please implement a macOS menu-bar mindful-breathing companion app called **VibeCodingBreath** (Swift 5.10, macOS 14+, Apple Silicon + Intel universal) with the following requirements:

1. The app is a near-invisible status-bar utility built around one idea: 'AI is thinking, you are breathing.' While the user waits for an AI agent (Cursor / Copilot / Claude Code, etc.), if the keyboard and mouse stay idle for 5 seconds the app fades a soft breathing halo into the center of the main screen to guide a micro mindfulness session; the moment the user touches the mouse or keyboard the halo fades out within 200 ms. Zero notifications, zero onboarding, zero configuration UI in MVP.

2. Run as a pure menu-bar agent: `LSUIElement = true`, no Dock icon, no Cmd+Tab presence, no default window. Bundle ID `pro.nanzhi.VibeCodingBreath`, `LSMinimumSystemVersion = 14.0`. Enforce single instance by checking `NSWorkspace.runningApplications` for the same bundle id at launch and terminating duplicates. Register login-item auto-start via `SMAppService.mainApp.register()` on first launch; failures are logged silently and retried next launch — never show UI.

3. Status bar: a template `NSStatusItem` (auto adapts to light/dark) whose menu contains exactly two items, fully localized:
   - 'Open VibeCodingBreath' (`status.menu.open`) — manually force-show the halo, ignoring idle judgment, until the next user input.
   - 'Quit' (`status.menu.quit`) — `NSApp.terminate(_:)`, leaves no residual window or status item.

4. Idle / context detection (no Accessibility, no Screen Recording, no Notifications permission required):
   - Poll every 500 ms with `CGEventSource.secondsSinceLastEventType(.combinedSessionState, eventType:)` over the union of: `.mouseMoved, .leftMouseDown, .rightMouseDown, .otherMouseDown, .leftMouseDragged, .rightMouseDragged, .scrollWheel, .keyDown, .flagsChanged`. Take the minimum 'seconds since' across all types; if it exceeds the 5 s threshold, emit `idle = true`, otherwise `false`. De-duplicate so the change callback only fires on real edges.
   - A `ForegroundContextMonitor` observes `NSWorkspace.activeSpaceDidChangeNotification` and `NSWorkspace.didActivateApplicationNotification`, and uses a heuristic (menu-bar-hidden frame check + frontmost app's `activationPolicy == .regular`) to expose `isFullscreenContext`. While true, the halo must NEVER appear, even if the user manually picked 'Open VibeCodingBreath'.

5. Breathing halo overlay window — a transparent click-through `NSPanel` subclass:
   - `styleMask = [.borderless, .nonactivatingPanel]`, `isOpaque = false`, `backgroundColor = .clear`, `hasShadow = false`, `level = .statusBar`, `ignoresMouseEvents = true`, `animationBehavior = .none`, `isReleasedWhenClosed = false`, `canBecomeKey = false`, `canBecomeMain = false`.
   - `collectionBehavior = [.canJoinAllSpaces, .stationary, .ignoresCycle, .fullScreenAuxiliary]`.
   - Centered on `NSScreen.main.frame`; window size = `maxDiameter (320) + padding (64)`. Re-center on `NSApplication.didChangeScreenParametersNotification`. If the main screen disappears, hide.
   - Show: `alphaValue = 0` → `orderFrontRegardless()` → start engine → `NSAnimationContext` fade in over 0.4 s to 1.0.
   - Hide: `NSAnimationContext` fade out over 0.2 s to 0 → stop engine → `orderOut`. Halo is always click-through (verify by clicking through onto Finder icons underneath).

6. Breathing rhythm engine — `@MainActor @Observable final class BreathingEngine` driving a strict 4-2-6-2 cycle (inhale 4 s → hold 2 s → exhale 6 s → hold 2 s, total 14 s) via a single `Task { while !Task.isCancelled { for phase in BreathPhase.allCases { self.phase = phase; try? await Task.sleep(for: .seconds(phase.duration)) } } }`. `start()` / `stop()` are idempotent. Define `enum BreathPhase: Int, CaseIterable, Sendable { case inhale, holdAfterInhale, exhale, holdAfterExhale }` with `duration` and `targetScale` (1.0 at exhale/rest, 2.0 at inhale/peak).

7. Halo visual (SwiftUI): a single `Circle()` filled with a `RadialGradient` (default palette `#7FB3D5`, opacity stops adapted per `ColorScheme` — light: 0.65 → 0.35 → 0; dark: 0.55 → 0.30 → 0), `.blur(radius: 24)`, base size 120 pt, scaling between 1.0× and 2.0× so visual diameter spans 120 → 240 pt and never exceeds 25% of the screen short edge. Use `withAnimation(.easeInOut(duration: phase.duration)) { scale = phase.targetScale }` driven by `onChange(of: engine.phase)`. `accessibilityHidden(true)`. NO bars, NO waveforms, NO text labels in MVP — just the soft breathing halo. The view holds no business state; it observes the engine.

8. Architecture — single executable target, all UI types `@MainActor`, no `DispatchQueue.main.async` (use `Task { @MainActor in ... }`):
   - `App/VibeCodingBreathApp.swift`: `@main struct` with only `Settings { EmptyView() }` plus `@NSApplicationDelegateAdaptor(AppDelegate.self)`.
   - `App/AppDelegate.swift`: builds `AppCoordinator` in `applicationDidFinishLaunching`, returns `false` from `applicationShouldTerminateAfterLastWindowClosed`, runs the single-instance check, calls `coordinator.stop()` in `applicationWillTerminate`, hides on `NSWorkspace.willSleepNotification` and re-evaluates on `didWakeNotification`.
   - `App/AppCoordinator.swift`: owns `StatusItemController`, `IdleMonitor`, `ForegroundContextMonitor`, `OverlayWindowController`, `LoginItemManager`, `BreathingEngine`. Exposes `start() / stop() / openManually()`. State machine `.hidden → .showing → .dismissing → .hidden`; `evaluate()` reconciles `(isIdle, isFullscreenContext, manualOverride)` into `show()` or `hide()`; `manualOverride` is set true by 'Open VibeCodingBreath' and cleared on the next active edge.
   - `Status/StatusItemController.swift`, `Idle/{IdleMonitor,ForegroundContextMonitor}.swift`, `Overlay/{OverlayPanel,OverlayWindowController,BreathingOverlayView}.swift`, `Breathing/{BreathPhase,BreathingEngine}.swift`, `System/LoginItemManager.swift`, `Support/{Constants,Palette,Logger+App,Protocols}.swift`.
   - `Support/Constants`: `idleThreshold = 5.0`, `pollInterval = 0.5`, `minDiameter = 120`, `maxDiameter = 320`, `overlayPadding = 64`, `fadeIn = 0.4`, `fadeOut = 0.2`, `primaryHex = "#7FB3D5"`, `bundleIdentifier = "pro.nanzhi.VibeCodingBreath"`.
   - `Support/Logger+App`: `os.Logger` with subsystem = bundle id and categories `app / idle / overlay / status / login`.
   - `Support/Protocols.swift`: `@MainActor` protocols (e.g. `ForegroundContextProviding`) used to inject fakes in tests.

9. Swift Package Manager only — no `.xcodeproj`. Use `// swift-tools-version: 5.10`, `defaultLocalization: "en"`, `platforms: [.macOS(.v14)]`, single `executableTarget` with `path: "Sources/VibeCodingBreath"`, `resources: [.process("Resources")]`, and `swiftSettings`: `.enableUpcomingFeature("BareSlashRegexLiterals")`, `.enableUpcomingFeature("ConciseMagicFile")`, `.enableExperimentalFeature("StrictConcurrency=complete")`. Test target `VibeCodingBreathTests` (XCTest, since Swift 5.10 has no Swift Testing). Zero third-party dependencies. Two SwiftPM 5.10 quirks to plan around: (a) `Info.plist` is forbidden as a top-level SwiftPM resource and MUST live outside `Sources/` (put it in `Bundle/Info.plist`, assembled by the build script), and (b) `.xcstrings` is not compiled by SwiftPM 5.10 — use legacy `Resources/<lang>.lproj/Localizable.strings` files instead. Ship at least `en` and `zh-Hans`.

10. Avoid a known Swift 5.10 IRGen crash ('SmallVector unable to grow'): a `@MainActor final class`'s designated init that takes a parameter whose default value is an `@escaping` closure will crash the compiler. Work around it by using a stored Bool flag and constructing the closure inside `start()` instead of as a default parameter.

11. Bundle assembly + signing — three scripts under `Scripts/`:
    - `Scripts/build-app.sh [debug|release]`: runs `swift build` (release uses `--arch arm64 --arch x86_64` for a universal binary), then assembles `.build/<config>/VibeCodingBreath.app` with `Contents/MacOS/VibeCodingBreath`, `Contents/Info.plist` copied from `Bundle/Info.plist`, `Contents/Resources/AppIcon.icns` produced from `Bundle/Resources/AppIcon.iconset` via `iconutil`, and the SwiftPM-emitted `*_VibeCodingBreath.bundle` placed under `Contents/Resources/`. Each embedded `.bundle` must be `codesign`'d separately BEFORE the outer `.app` is signed.
    - `Scripts/codesign.sh`: `codesign --deep --options runtime --timestamp --entitlements Bundle/VibeCodingBreath.entitlements --sign "Developer ID Application: ..."` then `xcrun notarytool submit --wait --keychain-profile <profile>` then `xcrun stapler staple`. Entitlements request no network, no files, no camera/mic — hardened runtime only.
    - `Scripts/release.sh --version <x.y.z> --keychain-profile <profile>`: full pipeline of `swift test` → `build-app.sh release` → `codesign.sh` → `create-dmg` outputting `dist/VibeCodingBreath-<version>-<arch>.dmg`.

12. Tests under `Tests/VibeCodingBreathTests/` (XCTest, all green via `swift test`):
    - `BreathingEngineTests`: phase order is `inhale → holdAfterInhale → exhale → holdAfterExhale`; `stop()` halts further phase changes; repeated `start()` calls do not spawn multiple Tasks.
    - `BreathPhaseTests`: durations and `targetScale` are exactly 4/2/6/2 and 2.0/2.0/1.0/1.0.
    - `IdleMonitorTests`: with an injected fake 'seconds-since' provider, crossing the 5 s threshold fires the callback exactly once per edge; same-state ticks do not.
    - `AppCoordinatorTests`: `(idle, fullscreen, manualOverride)` truth-table — only `(idle && !fullscreen) || (manualOverride && !fullscreen)` results in show; fullscreen always wins.
    - `ConstantsTests`: guard the default values so they are not silently changed.
    - `LocalizationTests`: every menu / phase localization key resolves in both `en` and `zh-Hans`.
    - Provide `Tests/VibeCodingBreathTests/Fakes/Fakes.swift` implementing the `@MainActor` protocols from `Support/Protocols.swift` for deterministic testing.

13. Acceptance criteria (all observably true on a clean macOS 14 install):
    - No Dock icon; status-bar template icon appears.
    - Status menu contains exactly two items, both localized in zh-Hans / en per system language.
    - 5 s of no input on the desktop → halo fades in at the center of the main screen, scaling on the 4-2-6-2 rhythm.
    - Any mouse move or key press makes the halo fade out within 200 ms.
    - The halo NEVER intercepts clicks (clicks reach Finder / underlying apps).
    - Frontmost app entering true fullscreen (Keynote / video / etc.) suppresses the halo entirely.
    - After macOS reboot the app auto-launches into the menu bar.
    - 'Quit' terminates the process and leaves nothing behind.
    - Idle CPU < 1%, resident memory < 80 MB.
    - No system permission prompts ever appear.

14. Provide a concise top-level `README.md` linking out to `docs/PRD.md`, `docs/TECH_PLAN.md`, `docs/TEST_PLAN.md` (do NOT inline the spec — keep README a high-level index per project convention), and listing the three core commands: `swift test`, `./Scripts/build-app.sh debug`, `./Scripts/release.sh --version <x.y.z> --keychain-profile <profile>`.
