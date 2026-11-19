# Acceptance report: virtual-display-rs PR #236

[Pull request](https://github.com/MolotovCherry/virtual-display-rs/pull/236) · [Tracking issue #237](https://github.com/MolotovCherry/virtual-display-rs/issues/237) · [master patch 1de3992](https://github.com/WilliamLambertCN/virtual-display-rs/commit/1de3992aa5e18ea8f07f34fb34c46d37787e68b2) · [v0.3.1 backport fc9d123](https://github.com/WilliamLambertCN/virtual-display-rs/commit/fc9d123c85b3a1ee2e6170d5214e0031350ca046)

**Decision: not accepted for deployment on this evidence.** The two COM ownership changes have been reviewed; there is no verified runtime result for either patched driver in this report. The application test suite is not a driver test suite.

| Check | Result | Evidence |
| --- | --- | --- |
| Device reference | Source review passed, both branches | `SetDevice` borrows `IDXGIDevice` instead of transferring the caller's reference with `into_raw()`. The Rust owner is dropped on setup failure or when processing exits. |
| Acquired surface reference | Source review passed, both branches | A non-null surface from a successful acquire is owned and released once before `FinishedProcessingFrame`. Pending/error paths do not inspect the out pointer. This matches the [Microsoft sample](https://github.com/microsoft/Windows-driver-samples/blob/9afd93066dfd9db12f66099cf9ec44b6fd734b2d/video/IndirectDisplay/IddSampleDriver/Driver.cpp#L374-L450). It is a reference per acquire, not necessarily a texture allocation per frame. |
| Patch and formatting | Passed | `git diff --check` and changed-file `rustfmt --check --edition 2021` passed on both branches using their pinned nightly toolchains. Each source diff is limited to `swap_chain_processor.rs`; there are no mode, IPC or monitor-identity changes. |
| Locked x64 Release build: master patch | Failed in the last completed attempt recorded here | With the WDK and libclang available, the generated WDF/IddCx types were missing fields, including `Size`; the build stopped in `wdf-umdf-sys` before linking a driver. |
| Locked x64 Release build: v0.3.1 backport | Failed in the last completed attempt recorded here | The generated legacy bindings lacked required WDF/IddCx types and symbols. The legacy IPC crate's test command discovered zero tests; this is not a substitute for driver validation. |
| Deployment, display regression and memory | Not run on a patched driver | No patched driver was loaded for this report. The two display smoke cycles in the [application PR](https://github.com/nicocart/snug-win/pull/1) used the **original** v0.3.1 driver. They neither reproduce nor demonstrate a post-fix memory result. |
| Contributor agreement | Separate administrative check | The posted CLA Assistant check requests the contributor's agreement; it is not a build or runtime test. |

## Recheck and release gates

From `rust/` on master or the repository root on the legacy branch, using the corresponding `rust-toolchain.toml` and the x64 MSVC/WDK environment:

```powershell
cargo build --locked -p virtual-display-driver --release --target x86_64-pc-windows-msvc
```

Before a runtime pass: verify the actual x64 DLL and the signed package containing it; record the loaded binary identity and keep a verified rollback package. Compare matched display creation, mode change, and removal cycles on the original and patched versions in the same setup. Verify removal and preservation of the primary and pre-existing paths; measure the identified driver host before the test and after comparable idle intervals. Record the observed trend without assigning all host-private memory to a single monitor or treating a short plateau as proof of a universal leak fix.
