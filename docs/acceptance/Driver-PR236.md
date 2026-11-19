# Acceptance report: virtual-display-rs PR #236

[Pull request](https://github.com/MolotovCherry/virtual-display-rs/pull/236) · [Tracking issue #237](https://github.com/MolotovCherry/virtual-display-rs/issues/237) · [master patch 1de3992](https://github.com/WilliamLambertCN/virtual-display-rs/commit/1de3992aa5e18ea8f07f34fb34c46d37787e68b2) · [v0.3.1 backport fc9d123](https://github.com/WilliamLambertCN/virtual-display-rs/commit/fc9d123c85b3a1ee2e6170d5214e0031350ca046)

**Decision: source review, both locked x64 Release builds and ABI checks passed.** Neither patched driver has been accepted for deployment on these checks alone. The application test suite is not a driver test suite.

| Check | Result | Evidence |
| --- | --- | --- |
| Device reference | Source review passed, both branches | `SetDevice` borrows `IDXGIDevice` instead of transferring the caller's reference with `into_raw()`. The Rust owner is dropped on setup failure or when processing exits. |
| Acquired surface reference | Source review passed, both branches | A non-null surface from a successful acquire is owned and released once before `FinishedProcessingFrame`. Pending/error paths do not inspect the out pointer. This matches the [Microsoft sample](https://github.com/microsoft/Windows-driver-samples/blob/9afd93066dfd9db12f66099cf9ec44b6fd734b2d/video/IndirectDisplay/IddSampleDriver/Driver.cpp#L374-L450). It is a reference per acquire, not necessarily a texture allocation per frame. |
| Patch and formatting | Passed | `git diff --check` and changed-file `rustfmt --check --edition 2021` passed on both branches using their pinned nightly toolchains. Each source diff is limited to `swap_chain_processor.rs`; there are no mode, IPC or monitor-identity changes. |
| Locked x64 Release build: master patch | Passed | Pinned Rust nightly-2024-07-26, libclang 17.0.6, x64 MSVC and WDK. Raw unsigned DLL: 724,992 bytes, SHA-256 `92547e2d57bdba76773c985a80529aaaded4575c16d2b6a591aaeeb76371096f`. |
| Locked x64 Release build: v0.3.1 backport | Passed | Pinned Rust nightly-2023-09-19, same libclang and WDK. Raw unsigned DLL: 394,240 bytes, SHA-256 `f66487139e9a01adaadd4504206a454336420d67903b7a2214c4354379f16ee7`. The legacy IPC crate discovered zero tests; this is not a driver runtime result. |
| WDF and IddCx ABI and exports | Passed on both builds | On each branch 33/33 size, alignment and field-offset comparisons against the native WDK headers matched (`WDF_OBJECT_ATTRIBUTES`: 56 bytes; `IDD_CX_CLIENT_CONFIG`: 168 bytes). Both binaries are PE32+ AMD64 and export `DriverEntry`, `FxDriverEntryUm`, `IddMinimumVersionRequired`. |
| Legacy package integrity | Passed offline, not a release signature | A local-only package was prepared for the backport. The catalog covers only the INF and DLL; both members matched, with the signed image's only differences from the raw build attributable to signing. No kernel binary was added. |
| Windows deployment preflight | Blocked before staging | `SetupVerifyInfFileW` returned `CERT_E_UNTRUSTEDROOT` for the local signer. No trust anchor was added, no security policy was relaxed, and the original driver remained bound. Offline signature integrity does not mean Windows trusts or loaded the package. |
| Display regression and memory | Not run on a patched driver | A physical-primary output interruption was reported with the **original** driver installed. Display testing was stopped. See the [application PR](https://github.com/nicocart/snug-win/pull/1). There is no matched before/after series, no attribution to this COM defect, and no runtime result for the patched driver. |
| Contributor agreement | Separate administrative check | The posted CLA Assistant check requests the contributor's agreement; it is not a build or runtime test. |

## Recheck and release gates

Run from the **driver crate directory** (`rust/virtual-display-driver` on master, `virtual-display-driver` on the legacy branch). Use the pinned Rust toolchain, x64 MSVC, WDK and libclang 17 with its matching resource headers. Keep the repository's own Cargo configuration and linker flags. The earlier failures were reproduced with libclang 23: legacy bindgen omitted WDF aliases and the master bindings omitted struct fields. Switching to libclang 17 fixed both builds without changing the lockfiles or WDK headers.

```powershell
cargo build --release --locked --target x86_64-pc-windows-msvc
```

Before a runtime pass: verify the actual x64 DLL and the signed package containing it; record the loaded binary identity and keep a verified rollback package. Compare matched display creation, mode change, and removal cycles on the original and patched versions in the same setup. Verify removal and preservation of the primary and pre-existing paths; measure the identified driver host before the test and after comparable idle intervals. Record the observed trend without assigning all host-private memory to a single monitor or treating a short plateau as proof of a universal leak fix.
