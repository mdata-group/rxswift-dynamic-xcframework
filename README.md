# RxSwift Dynamic XCFrameworks

Prebuilt **dynamic** `.xcframework` distribution of the [ReactiveX/RxSwift](https://github.com/ReactiveX/RxSwift) family
(`RxSwift`, `RxRelay`, `RxCocoa`, `RxTest`, `RxBlocking`) for consumption via Swift Package Manager as **binary targets**.

> This repo does **not** modify RxSwift's source. It repackages the official, unmodified RxSwift at a pinned tag into
> dynamic frameworks. RxSwift is MIT-licensed (see `LICENSE`).

---

## Why this exists (the problem)

On **Xcode 26 / 27**, Apple **removed the x86_64 / Rosetta iOS Simulator**. On Apple Silicon the app must therefore
run on the **arm64 simulator**. That surfaced a long-standing SwiftPM limitation that the old x86_64 simulator had been
masking:

- SwiftPM builds RxSwift as a **static** library.
- `RxTest` / `RxBlocking` statically depend on the `RxSwift` **target**, so **any** binary that links them embeds its
  own full copy of RxSwift.
- A typical app links `RxSwift`/`RxCocoa` into the **app**, and the unit-test bundle links `RxTest`/`RxBlocking` (which
  drag in RxSwift) into the **test bundle**. Result: **two copies of RxSwift in one process.**
- Two copies means two sets of RxSwift's Swift **generic metadata**. At runtime the metadata for a generic `Sink`
  subclass (e.g. `AnonymousObservableSink`, `FlatMapSink`, `DoSink`) can't resolve its superclass across the duplicate,
  and the process aborts:

```
libswiftCore.dylib  swift::fatalError(...)
libswiftCore.dylib  getSuperclassMetadata + 576
<app>               type metadata completion function for AnonymousObservableSink
...
Thread 0 crashed:   EXC_CRASH (SIGABRT)
objc[]: Class _TtC7RxSwift...DisposeBag is implemented in both <App> and <TestBundle>.
```

This is Swift bug **[SR-12303](https://bugs.swift.org/browse/SR-12303)** and is reported across
RxSwift issues [#2057](https://github.com/ReactiveX/RxSwift/issues/2057),
[#2107](https://github.com/ReactiveX/RxSwift/issues/2107),
[#2127](https://github.com/ReactiveX/RxSwift/issues/2127),
[#2296](https://github.com/ReactiveX/RxSwift/issues/2296).

### Why it "used to work" on Xcode ≤ 25 / x86_64

The x86_64 simulator's linker/back-deploy behaviour tolerated the duplication; the arm64 simulator's does not. It's the
**simulator architecture** that changed, not RxSwift. (You can reproduce the same crash on Xcode 26 by forcing an arm64
simulator build.) Workarounds like an `-all_load` umbrella framework, `.dynamic` SwiftPM products, mergeable libraries,
`-undefined dynamic_lookup`, or `DEAD_CODE_STRIPPING=NO` do **not** fix it — Apple DTS confirms the linker
[cannot de-duplicate code symbols across a dynamic boundary](https://developer.apple.com/forums/thread/741545), and each
`.dynamic` SwiftPM product still statically embeds its own RxSwift copy.

## The solution

Ship each RxSwift module as its **own dynamic framework**, the way Carthage always did:

- `RxSwift.framework` is the single home of RxSwift's code.
- `RxCocoa.framework` / `RxRelay.framework` / `RxTest.framework` / `RxBlocking.framework` **dynamically link**
  `RxSwift.framework` via `@rpath` — they do **not** embed their own copy.

So the whole process (app host **and** its xctest bundle) shares **exactly one** `RxSwift.framework` — one metadata
identity — and the `getSuperclassMetadata` crash disappears.

`RxCocoa`'s internal `RxCocoaRuntime` module is compiled into `RxCocoa.framework`, so `import RxCocoa` resolves without
the "missing required module 'RxCocoaRuntime'" SwiftPM error.

## Usage

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/mdata-group/rxswift-dynamic-xcframework", exact: "6.8.0-xcode26.2"),
],
targets: [
    .target(name: "App", dependencies: [
        .product(name: "RxSwift", package: "rxswift-dynamic-xcframework"),
        .product(name: "RxCocoa", package: "rxswift-dynamic-xcframework"),
        .product(name: "RxRelay", package: "rxswift-dynamic-xcframework"),
    ]),
    .testTarget(name: "AppTests", dependencies: [
        // Test target adds RxTest/RxBlocking. Because they dynamically link the
        // SAME RxSwift.framework the host already embeds, there is only one copy.
        .product(name: "RxTest", package: "rxswift-dynamic-xcframework"),
        .product(name: "RxBlocking", package: "rxswift-dynamic-xcframework"),
    ]),
]
```

In an Xcode project: add the package, link the products you need, and let Xcode **Embed & Sign** the frameworks on the
**app** target. On the **unit-test** target link the products **Do Not Embed** (the host app already provides them).

The frameworks are **binary-compatible with the identical module names** (`RxSwift`, `RxCocoa`, …), so existing
`import RxSwift` / `import RxCocoa` source needs **no changes**.

### Slices

Each xcframework contains:

| slice | archs |
|-------|-------|
| `ios-arm64` | device |
| `ios-arm64_x86_64-simulator` | Apple-silicon **and** Intel simulator |

Built with `BUILD_LIBRARY_FOR_DISTRIBUTION=YES` (stable `.swiftinterface`).

### Privacy manifests

`RxSwift` and `RxCocoa` are on Apple's list of commonly used third-party SDKs, so the consuming app has to ship their
privacy manifests. Upstream (RxSwift 6.8.0+) declares them via SwiftPM `.copy("PrivacyInfo.xcprivacy")` under
`Sources/<module>/`, which `Rx.xcodeproj`'s framework targets never reference — an `xcodebuild archive` would drop them.
`build.sh` therefore copies each upstream manifest to its **framework bundle root**
(`RxSwift.framework/PrivacyInfo.xcprivacy`), where Apple's privacy report reads it from for a framework.

`RxSwift`, `RxRelay` and `RxCocoa` carry a manifest; `RxTest` / `RxBlocking` have none upstream (they never ship in an
App Store build). RxSwift **6.7.0 and earlier have no manifests at all** — use 6.8.0 or newer.

## How it's built / how to update to a new RxSwift version

Everything is produced by [`build.sh`](build.sh) — no manual steps, fully reproducible:

```bash
./build.sh <rxswift-version>
# e.g.
./build.sh 6.8.0
```

`build.sh`:
1. Clones the **official** `ReactiveX/RxSwift` at the exact tag (`--branch <version>`).
2. Archives each module (`RxSwift`, `RxRelay`, `RxCocoa`, `RxTest`, `RxBlocking`) from RxSwift's own `Rx.xcodeproj`
   framework schemes, for **device** and **simulator**, with library evolution.
3. Injects the upstream `PrivacyInfo.xcprivacy` of each module into its framework bundle root.
4. Assembles the five `.xcframework`s (device + arm64/x86_64 simulator).
5. Zips each, computes the SwiftPM checksum.
6. Regenerates `Package.swift` with the release URL + checksums for that tag.
7. Verifies each xcframework has both slices, that `RxCocoa` `@rpath`-links `RxSwift`, and that the manifests are in place.

To publish a new version:

```bash
# Pick the Xcode that will produce the .swiftinterface, and tag accordingly.
DEVELOPER_DIR=/Applications/Xcode-26.2.0.app/Contents/Developer \
RELEASE_TAG=6.8.0-xcode26.2 ./build.sh 6.8.0
git add build.sh Package.swift README.md
git commit -m "RxSwift 6.8.0 dynamic xcframeworks (6.8.0-xcode26.2)"
git tag 6.8.0-xcode26.2 && git push --tags
gh release create 6.8.0-xcode26.2 release/*.xcframework.zip \
  --title "RxSwift 6.8.0 (dynamic xcframeworks, Xcode 26.2)"
```

The prebuilt `.swiftinterface` is tied to the Swift toolchain that produced it, so the release tag is
`<rxswift-version>-xcode<xcode-version>` (`RELEASE_TAG`, defaults to the bare version). Consumers pin the exact pair they
want via `exact: "<rxswift-version>-xcode<xcode-version>"`.

## Requirements

- iOS 15.0+
- Xcode 16+ (built and verified on Xcode 27)

## License

RxSwift is © the RxSwift authors, MIT License (bundled `LICENSE`). This repository only repackages the unmodified
upstream binaries; all RxSwift copyright and license terms apply.
