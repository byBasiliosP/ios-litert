# ios-litert

Private distribution repository for a clean iOS LiteRT runtime package.

## Package Contents

- `LiteRtLmRuntime.xcframework`
- `GemmaModelConstraintProvider.xcframework`

## Required Integration Contract

`LiteRtLmRuntime.xcframework` is not standalone. The runtime binary is linked to
load `libGemmaModelConstraintProvider.dylib` from the same embedded framework
location, so both xcframeworks must be embedded together.

## Documentation

- [Binary Integration Report](docs/BINARY_INTEGRATION_REPORT.md)

## Artifact Scope

This repository is intentionally limited to the packaged runtime deliverables and
their integration documentation. Application-specific project files are excluded.
