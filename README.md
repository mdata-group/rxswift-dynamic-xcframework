# RxSwift Dynamic XCFrameworks

Prebuilt **dynamic** `.xcframework` distribution of the [ReactiveX/RxSwift](https://github.com/ReactiveX/RxSwift) family
(`RxSwift`, `RxRelay`, `RxCocoa`, `RxTest`, `RxBlocking`) for consumption via Swift Package Manager as **binary targets**.

> This repo does **not** modify or rebuild RxSwift. Since `6.9.1` it **splits the official
> `RxSwift.xcframework.zip` release asset** into one zip per module, leaving each xcframework — and its
> `Apple Distribution: Shai Mishali (272EB7D3H3)` signature — byte-identical. RxSwift is MIT-licensed (see `LICENSE`).
>
> Why splitting is needed at all: SwiftPM does **not** match a binary artifact to its target by name. Point five
> `.binaryTarget`s at the one official zip and it picks an arbitrary xcframework for each —
> `multiple potential binary artifacts found: ... using the one in .../RxRelay.xcframework` — so a target named
> `RxSwift` silently gets `RxRelay`.

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
simulator build.) Workarounds like an `-all_load` umbrella framework, mergeable libraries, `-undefined dynamic_lookup`, or
`DEAD_CODE_STRIPPING=NO` do **not** fix it — Apple DTS confirms the linker
[cannot de-duplicate code symbols across a dynamic boundary](https://developer.apple.com/forums/thread/741545).

### Why not just link upstream's own `-Dynamic` products?

RxSwift's `Package.swift` has declared `RxSwift-Dynamic`, `RxRelay-Dynamic`, `RxCocoa-Dynamic`, `RxTest-Dynamic` and
`RxCocoa-Dynamic` (`type: .dynamic`) since **6.8.0** — that is upstream's own answer to this bug, and in a *pure*
SwiftPM package it works. Through **Xcode project** SPM integration it does not. Measured on Xcode 27.0 with RxSwift
6.10.2:

| linked products | build | unit tests | `simctl install` + `launch` |
|---|---|---|---|
| all five `-Dynamic` | ❌ `clang: error: no such file or directory: PackageFrameworks/RxSwift-Dynamic.framework/RxSwift-Dynamic` | — | — |
| `RxSwift`/`RxRelay` plain + rest `-Dynamic` | ✅ | ✅ 566 passed | ❌ `OS_REASON_DYLD \| Library not loaded: @rpath/RxCocoa-Dynamic.framework/RxCocoa-Dynamic`, died in 392 ms |
| `-Dynamic` on the test target only | ✅ | ✅ 566 passed | ✅ launches, but `RxRelay` ends up duplicated **in the shipping app**: `Class _TtC7RxRelay..._BundleFinder is implemented in both RxCocoa.framework/RxCocoa and <app>.debug.dylib` |

Two Xcode defects: (1) when a target has both a plain and a `-Dynamic` product **and** another dynamic product depends
on it, the `-Dynamic` framework is emitted as an empty shell (`Info.plist` only); (2) `-Dynamic` products are linked via
`@rpath` but never auto-embedded into the bundle.

Note the second row: the app **cannot launch** yet the whole unit-test suite passes, because the test harness adds the
build-products directory to the framework search path. A CI job that only runs tests will not catch it.

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
    .package(url: "https://github.com/mdata-group/rxswift-dynamic-xcframework", exact: "6.9.1"),
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

In an Xcode project: add the package and link the products you need — embedding is automatic for SPM binary framework
products. The test bundle gets its own copy of the frameworks on disk and there is no build phase to turn that off, but
it is harmless: dyld resolves by `@rpath` install name, so exactly one image is loaded per process (verify with
`Class ... is implemented in both` being absent from the test log).

The frameworks are **binary-compatible with the identical module names** (`RxSwift`, `RxCocoa`, …), so existing
`import RxSwift` / `import RxCocoa` source needs **no changes**.

### Slices

Since `6.9.1` the slices are whatever upstream ships, which is everything:
`ios-arm64`, `ios-arm64_x86_64-simulator`, `ios-arm64_x86_64-maccatalyst`, `macos-arm64_x86_64`, `tvos-arm64`,
`tvos-arm64_x86_64-simulator`, `watchos-arm64_arm64_32_armv7k`, `watchos-arm64_x86_64-simulator`, `xros-arm64`,
`xros-arm64_x86_64-simulator` (`RxTest` has no watchOS slice).

That is ~116 MB of zips for an iOS-only consumer. Trimming the unused slices would invalidate upstream's code
signature, so they are kept intact — SwiftPM caches the artifacts per checksum, so the cost is one download per version
bump. Earlier, self-built tags (`≤ 6.8.0-xcode26.2`) carry only the two iOS slices.

All slices are built by upstream with `BUILD_LIBRARY_FOR_DISTRIBUTION=YES`. RxSwift `6.9.1` was produced with
**Apple Swift 6.2** (Xcode 26.0); consuming that `.swiftinterface` with a *newer* compiler is the supported direction.

### Privacy manifests

`RxSwift` and `RxCocoa` are on Apple's list of commonly used third-party SDKs, so the consuming app has to ship their
privacy manifests. Upstream's release xcframeworks carry `PrivacyInfo.xcprivacy` at the framework bundle root for
`RxSwift`, `RxRelay` and `RxCocoa`; `RxTest` / `RxBlocking` have none (they never ship in an App Store build).

**Use 6.9.1 or newer.** Upstream only fixed the missing manifest in the Carthage/xcframework artifacts in
[6.9.0](https://github.com/ReactiveX/RxSwift/releases/tag/6.9.0) — the `6.8.0` release xcframework has none, and
RxSwift 6.7.0 and earlier have no manifests anywhere. (`build-from-source.sh` injects them manually, because
`Rx.xcodeproj`'s framework targets never reference the files that upstream's `Package.swift` declares via
`.copy("PrivacyInfo.xcprivacy")`.)

## How it's built / how to update to a new RxSwift version

Everything is produced by [`build.sh`](build.sh) — no manual steps, fully reproducible:

### Primary path — repackage the official release ([`build.sh`](build.sh))

Nothing is compiled, so the output does not depend on the local Xcode.

```bash
./build.sh 6.9.1
git add build.sh build-from-source.sh Package.swift README.md
git commit -m "RxSwift 6.9.1 dynamic xcframeworks (repackaged from official release)"
git tag 6.9.1 && git push origin main --tags
gh release create 6.9.1 release/*.xcframework.zip \
  --title "RxSwift 6.9.1 (official dynamic xcframeworks, repackaged)"
```

`build.sh`:
1. Downloads the official `RxSwift.xcframework.zip` asset from `ReactiveX/RxSwift` at that tag.
2. Unpacks and checks all five xcframeworks are present.
3. **Verifies the upstream signature** — refuses to republish unless every xcframework is signed by team
   `272EB7D3H3` (`EXPECT_TEAM` to override).
4. Zips each xcframework separately and computes its SwiftPM checksum.
5. Regenerates `Package.swift` with the release URLs + checksums.

Then verifies every slice is `MH_DYLIB`, that `RxRelay`/`RxCocoa`/`RxTest`/`RxBlocking` `@rpath`-link
`RxSwift.framework` rather than embedding it, and that the privacy manifests are present.

Release tags match the upstream version exactly (`6.9.1`), because the binaries **are** the upstream binaries.

### Fallback — build from source ([`build-from-source.sh`](build-from-source.sh))

Upstream does not attach an xcframework to every release (**6.10.2 has none**). When that happens, or when a slice
combination upstream doesn't ship is needed, build from source instead:

```bash
DEVELOPER_DIR=/Applications/Xcode-26.2.0.app/Contents/Developer \
RELEASE_TAG=6.10.2-xcode26.2 ./build-from-source.sh 6.10.2
```

Its output *is* tied to the toolchain that produced the `.swiftinterface`, so those tags carry a
`-xcode<version>` suffix and a new Xcode major version means a rebuild and a new tag. It also has to inject the
privacy manifests by hand. Prefer `build.sh` whenever the asset exists.

## Requirements

- iOS 15.0+
- Xcode 16+ (verified on Xcode 27)

## License

RxSwift is © the RxSwift authors, MIT License (bundled `LICENSE`). This repository only repackages the unmodified
upstream binaries; all RxSwift copyright and license terms apply.
