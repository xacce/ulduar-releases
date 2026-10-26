# Ulduar v0.1.1 build record, 2026-10-26

Source commit: `2ee11a58314ed5f2994891f1d9b75a84c564eb17`.

Passed: 31 release tests, including real-process storage root/restart coverage; Clippy with warnings denied; standalone client check; Windows x64 MSVC build; Linux x64 musl cross-build; Windows CLI/curl on default TCP 7443 with listen omitted and on explicitly configured TCP 8443.

Binaries are unsigned. SHA256SUMS records asset hashes.

Formatting: the default cargo fmt check reports existing compact-style differences throughout the source tree; no repository-wide reformat was applied.

Known performance limitation: the 10 GiB upload target (at least 80% of a plain disk write) is not repeatably passing on the development machine. See RELEASE_NOTES.md.

Not checked: running the Linux binary (WSL/Docker unavailable), systemd/SCM installation, or physical ENOSPC (reserve_bytes emulation is tested).
