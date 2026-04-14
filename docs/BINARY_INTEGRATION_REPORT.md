# Binary Integration Report

## Purpose

This repository distributes a clean iOS LiteRT runtime package that can be
embedded into an iOS application without carrying application-specific source
artifacts.

## Delivered Artifacts

### 1. Runtime xcframework

- `LiteRtLmRuntime.xcframework`
- Device slice: `ios-arm64/liblitertlm_runtime.dylib`
- Simulator slice: `ios-arm64-simulator/liblitertlm_runtime.dylib`

### 2. Constraint provider xcframework

- `GemmaModelConstraintProvider.xcframework`
- Device slice: `ios-arm64/libGemmaModelConstraintProvider.dylib`
- Simulator slice: `ios-arm64-simulator/libGemmaModelConstraintProvider.dylib`

## Why Two xcframeworks Are Required

The runtime binary is not self-contained. Its install-name and dependency
layout require a companion constrained-decoding provider dylib.

Observed runtime linkage for the device runtime binary:

```text
@rpath/liblitertlm_runtime.dylib
@loader_path/libGemmaModelConstraintProvider.dylib
/System/Library/Frameworks/AudioToolbox.framework/AudioToolbox
/System/Library/Frameworks/AVFoundation.framework/AVFoundation
/System/Library/Frameworks/Foundation.framework/Foundation
/System/Library/Frameworks/Metal.framework/Metal
/System/Library/Frameworks/Security.framework/Security
/System/Library/Frameworks/UIKit.framework/UIKit
/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
/usr/lib/libc++.1.dylib
/usr/lib/libobjc.A.dylib
/usr/lib/libSystem.B.dylib
/usr/lib/libresolv.9.dylib
```

The critical point is:

```text
@loader_path/libGemmaModelConstraintProvider.dylib
```

That means the runtime expects the provider dylib to be co-located in the same
embedded framework directory at runtime.

## Custom Additions Required To Make The Binary Work

The packaged binary relies on several non-default packaging steps.

### 1. Dynamic library install-name normalization

The runtime dylib must have its id rewritten to:

```text
@rpath/liblitertlm_runtime.dylib
```

The provider dylib must have its id rewritten to:

```text
@rpath/libGemmaModelConstraintProvider.dylib
```

This makes both libraries embeddable in a normal iOS app framework search path.

### 2. Runtime dependency rewrite

The runtime dylib dependency on the provider must be rewritten from an external
or source-build path to:

```text
@loader_path/libGemmaModelConstraintProvider.dylib
```

Without this change, the runtime package is not relocation-safe and will fail
when embedded outside the original build environment.

### 3. Companion provider packaging

The constrained-decoding provider must be shipped as a second xcframework.
The runtime package is not complete without it.

### 4. Dual-slice xcframework packaging

Both delivered packages include:

- device `arm64`
- simulator `arm64`

This allows the same package repository to support both on-device integration
and simulator validation.

### 5. Header bundling

The xcframeworks are packaged with public headers so consumers can integrate the
runtime through a stable binary package boundary rather than reconstructing
header layout manually.

### 6. Clean binary-only export boundary

This repository intentionally exports only:

- packaged xcframeworks
- integration documentation

It does not include:

- application project files
- app resources
- app-specific configuration
- model files
- packaging scripts from the source app repo

## Consumer Integration Steps

### Xcode integration

1. Add `LiteRtLmRuntime.xcframework` to the app target.
2. Add `GemmaModelConstraintProvider.xcframework` to the app target.
3. Set both to `Embed & Sign`.
4. Ensure both end up in the app's embedded frameworks output.

### Runtime expectations

The package only provides the runtime libraries. The consuming app remains
responsible for:

- shipping a compatible LiteRT model file
- choosing GPU or CPU backend policy
- selecting cache directory policy
- handling any higher-level app feature flags

## Audit For Project-Specific Artifact Leakage

An explicit audit was performed against the exported repository contents.

### Files checked

- `README.md`
- `LiteRtLmRuntime.xcframework/Info.plist`
- `GemmaModelConstraintProvider.xcframework/Info.plist`
- all packaged dylibs via `strings`

### Identifiers checked

- source application name
- source repository name
- source packaging-directory names

### Results

- No matching strings were found inside the packaged dylibs.
- No matching strings were found in the xcframework `Info.plist` files.
- Repository-level source-project references were removed from the exported
  documentation.

## Current Scope Limits

This repository is a binary distribution package, not a full SDK.

It does not provide:

- model download tooling
- a sample application
- Swift package manifest support
- release automation scripts

## Verification

To verify the package locally:

1. Inspect linkage:

```bash
otool -L LiteRtLmRuntime.xcframework/ios-arm64/liblitertlm_runtime.dylib
otool -L GemmaModelConstraintProvider.xcframework/ios-arm64/libGemmaModelConstraintProvider.dylib
```

2. Audit for leaked project strings:

```bash
strings LiteRtLmRuntime.xcframework/ios-arm64/liblitertlm_runtime.dylib | rg -i '<legacy-project-identifiers>'
strings GemmaModelConstraintProvider.xcframework/ios-arm64/libGemmaModelConstraintProvider.dylib | rg -i '<legacy-project-identifiers>'
rg -n -i '<legacy-project-identifiers>' .
```

3. Embed both xcframeworks into a test app and confirm launch resolves:

- `liblitertlm_runtime.dylib`
- `libGemmaModelConstraintProvider.dylib`

## Tradeoffs

- Delivering two xcframeworks is slightly heavier than a single binary, but it
  matches the actual runtime dependency graph.
- Shipping xcframeworks instead of loose dylibs simplifies Xcode embedding and
  reduces integration ambiguity.
- This repository stays binary-only, which keeps it clean, but consumers must
  still own model packaging and runtime policy.
