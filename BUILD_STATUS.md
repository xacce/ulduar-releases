# Ulduar v2.1.2 build record, 2026-11-06

Source commit: `8274102da7315a76fc0c111e41807bdde0401031`.

Built on Windows x64 with Rust 1.96: Windows x64 MSVC server and static Linux x64 musl server (Zig 0.16 via cargo-zigbuild). Packaged, signed and published from a separate Linux host.

Built on Windows x64 too: Windows x64 MSVC and Linux x64 GNU (glibc 2.17) native client libraries for the Rust SDK.

Passed on Windows x64: workspace tests (server library, v1 API, destructive, self-update, storage v2 end to end, client cache, native ABI); workspace Clippy with warnings denied; an independent consumer built from the packaged SDK against a real server (offline, crash recovery, sync, download, shared cache, v2 calls).

SHA256SUMS records asset hashes and is signed with the Ulduar release key (SHA256SUMS.sig, Ed25519); servers from v0.1.5 verify it before updating themselves. The executables are not Authenticode-signed.

Rust SDK 2.1.2 (storage v2 only, native ABI 3): SDK zip and .crate in this release, see SDK.md.

Object-count and offline-cache scale audits: unchanged since v0.1.4, see SCALE.md.

Known performance limitation: the 10 GiB upload target (at least 80% of a plain disk write) is not repeatably passing. See RELEASE_NOTES.md.

Linux musl server smoke-run on Linux x86_64 (version, user and token, v2 project create and list). The Linux GNU client library is cross-built and inspected, not run.

Ulduar Desktop: Windows x64 MSVC and Linux x64 GNU (glibc 2.28, no X11, Wayland or GL link) executables. Passed on Windows x64: unit, end-to-end and rendered-snapshot tests; a real window drawn on the CPU through the real event loop, no GPU or graphics driver used: working set 23 MB empty, 34 MB with 100 000 rows; repaint 5 ms at 1280x800, 37 ms at 3840x2160. Paste from the system clipboard with Ctrl+V on English and Russian layouts. Installation over SSH: the real plan run through OpenSSH under dash on Ubuntu 24.04 with the real server, certificate and pinned TLS; sudo, apt-get, systemctl, useradd, runuser and ufw stubbed and the service unit not installed (scripts/provision-stub-e2e.sh).

Not checked for Ulduar Desktop: installation on a real Ubuntu 24 machine with systemd, sudo and apt; a Linux window (the binary ran only --version and the headless installer); OS drag-and-drop and file dialogs with a person at the screen on any platform. No macOS build: linking needs the Apple SDK, which this build host does not have.

Not checked: self-update through a registered SCM service or systemd unit, physical ENOSPC (reserve_bytes emulation is tested), or abrupt power-loss recovery.
