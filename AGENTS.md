# Locus Agent Development Guide

This guide applies to the entire repository. **Locus** uses idevice FFI and the device developer tunnel to call Apple's location simulation service. Its interface is built with SwiftUI, and its maps and routes use MapKit.

## Getting Started

- Read [README.md](README.md) to understand the features and build process. When changing pairing, first-run setup, or installation flows, also read [SETUP.md](SETUP.md).
- Check `git status --short` first and preserve existing changes so each task stays within its requested scope.
- Treat `project.yml` as the source of truth for platforms, deployment versions, linker settings, and signing configuration. The project currently has an app target only and no automated test target.

## Code Map

- `Locus/App/LocusApp.swift`: app entry point, shared state injection, first-run routing, and external file handling.
- `Locus/Features/`: map, joystick, routes, settings, and first-run setup screens. Views call business logic through shared state.
- `Locus/Engine/SpoofSession.swift`: location session state, movement controls, route execution, keep-alive behavior, favorites, and history.
- `Locus/Engine/LocationEngine.swift`: the Swift/C FFI boundary for location simulation; manages the tunnel and native handles.
- `Locus/Engine/PairingStore.swift`: pairing file import and local storage. `PairOnDeviceService.swift` and `PairableHostAdvertiser.swift` implement on-device pairing and service advertising.
- `Locus/Engine/RouteBuilder.swift`: route sampling and GPX encoding/decoding. `Locus/Support/` contains the theme, file picker, and other shared support code.
- `Locus/Resources/`: permission declarations, entitlements, and resources. `Vendor/idevice/` contains the related headers, module map, and prebuilt static libraries.

## Change Conventions

- Follow the existing Swift style: four-space indentation, `UpperCamelCase` types, and `lowerCamelCase` members. The interface supports English and Simplified Chinese. Add user-facing text with `String(localized:)` or SwiftUI localization keys, and update both `Locus/Resources/en.lproj` and `zh-Hans.lproj`.
- Keep observable state such as `SpoofSession` and `PairingStore` isolated to `@MainActor`. Run blocking pairing work in the background and return UI state updates to the main actor.
- Keep native operations serialized when changing `LocationEngine`. Check ownership, release, and failure paths for handles and error objects, especially the transfer of remote server ownership after creating a simulation service.
- When changing sessions, routes, or keep-alive behavior, check that timers, tasks, and background activities are cleaned up on stop, cancellation, failure, and reconnect paths.
- Pairing files contain sensitive data. Keep them in local storage with restricted file permissions, and never include real pairing data in logs or the repository. Preserve paired acquisition and release of security-scoped resources when importing external files.
- Preserve the file picker, share-sheet opening, and clipboard entry points when changing pairing import. See `SETUP.md` for the LiveContainer-compatible file picker path.
- Follow the existing `locus.*` convention for persistence keys. Handle compatibility with saved data when changing existing keys or data formats.
- Prefer existing Apple frameworks for dependencies. Before changing the idevice interface, check `Vendor/idevice/idevice.h`; keep the headers, module map, and static libraries compatible.

## Project Configuration and Builds

Make project configuration changes in `project.yml`, then run `xcodegen generate` from the repository root and review the resulting `Locus.xcodeproj` changes. Changes made only to the generated project will be lost the next time it is generated. Treat `AppIcon.icon` as a complete asset package.

For a routine unsigned iOS device build, use:

```sh
xcodebuild -project Locus.xcodeproj -scheme Locus -configuration Debug \
  -destination 'generic/platform=iOS' -derivedDataPath .DerivedData \
  CODE_SIGNING_ALLOWED=NO build
```

See `README.md` for signed builds to install on a device. Pass the developer Team ID through local settings or the command line, and keep team configuration usable by other developers. Builds require a complete Xcode installation and an available iOS SDK; record specific errors if the toolchain or asset format is incompatible.

## Verification and Delivery

- Run relevant build checks after Swift or project configuration changes. For documentation-only changes, check the content, paths, and diff.
- There is currently no directly runnable automated test suite. Do not describe a successful build as a successful functional test.
- Tunnel, pairing, and background behavior require on-device verification using `SETUP.md`. A simulator or unsigned build cannot verify these behaviors.
- Choose manual checks based on the changes: first-run setup and pairing import, Teleport and stop, joystick, routes and GPX import/export, background and resume, and tunnel disconnection and reconnection.
- Before delivery, run `git diff --check`, review the scope, and report the changes, checks actually performed, and any device behavior not yet verified.
