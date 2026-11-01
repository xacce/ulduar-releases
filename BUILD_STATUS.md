# Ulduar v2.0.0 build record, 2026-11-01

Source commit: `f3bd6885954448bdf1883d55a261316e4b4423c2`.

Built on Windows x64 with Rust 1.96: Windows x64 MSVC server and static Linux x64 musl server (Zig 0.16 via cargo-zigbuild). Packaged, signed and published from a separate Linux host.

Built on Windows x64 too: Windows x64 MSVC and Linux x64 GNU (glibc 2.17) native client libraries for the Rust SDK.

Passed on Windows x64: workspace tests (server library, v1 API, destructive, self-update, storage v2 end to end, client cache, native ABI); workspace Clippy with warnings denied; an independent consumer built from the packaged SDK against a real server (offline, crash recovery, sync, download, shared cache, v2 calls).

SHA256SUMS records asset hashes and is signed with the Ulduar release key (SHA256SUMS.sig, Ed25519); servers from v0.1.5 verify it before updating themselves. The executables are not Authenticode-signed.

Rust SDK 2.0.0 (storage v2 only, native ABI 3): SDK zip and .crate in this release, see SDK.md.

Object-count and offline-cache scale audits: unchanged since v0.1.4, see SCALE.md.

Known performance limitation: the 10 GiB upload target (at least 80% of a plain disk write) is not repeatably passing. See RELEASE_NOTES.md.

Linux musl server smoke-run on Linux x86_64 (version, user and token, v2 project create and list). The Linux GNU client library is cross-built and inspected, not run.

Not checked: self-update through a registered SCM service or systemd unit, physical ENOSPC (reserve_bytes emulation is tested), or abrupt power-loss recovery.
