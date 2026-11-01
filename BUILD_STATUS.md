# Ulduar v0.1.5 build record, 2026-11-01

Source commit: `3b09e36d494e050d0d340001558c8afa86d31e6b`.

Built on Windows x64 with Rust 1.96: Windows x64 MSVC server and static Linux x64 musl server (Zig 0.16 via cargo-zigbuild). Packaged, signed and published from a separate Linux host.

Passed: workspace tests natively on Linux x86_64 (server API, destructive, self-update, cache, ABI) and on Windows x64; workspace Clippy with warnings denied; self-update end to end with the real release key on Windows (released Windows build updating from a local signed mirror, restart, job kill) and Linux (released musl build, exec in place, SIGTERM stop).

SHA256SUMS records asset hashes and is signed with the Ulduar release key (SHA256SUMS.sig, Ed25519); servers from v0.1.5 verify it before updating themselves. The executables are not Authenticode-signed.

The Rust SDK is unchanged and not part of this release: use the v0.1.4 SDK assets (see SDK.md).

Object-count and offline-cache scale audits: unchanged since v0.1.4, see SCALE.md.

Known performance limitation: the 10 GiB upload target (at least 80% of a plain disk write) is not repeatably passing. See RELEASE_NOTES.md.

Not checked: self-update through a registered SCM service or systemd unit, physical ENOSPC (reserve_bytes emulation is tested), or abrupt power-loss recovery.
