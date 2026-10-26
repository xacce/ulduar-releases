# Ulduar v0.1.2 build record, 2026-10-26

Source commit: `743dd5af90d13270e1f4221264bacb6cc4a40976`.

Passed: 32 release tests, including directory-enumeration failure regression and real-process storage root/restart coverage; Clippy with warnings denied; standalone client check; Windows x64 MSVC build; Linux x64 musl cross-build; Windows CLI/curl on default TCP 7443 with listen omitted and on explicitly configured TCP 8443.

Binaries are unsigned. SHA256SUMS records asset hashes.

Object-count audit: one million real 8-byte objects on Windows/NTFS passed on v0.1.1 production code before the error-path correction. Startup 0.507 s; presence checks 45.16 s; server sampled peak 14.12 MiB; full verify of 1,000,001 objects 428.81 s, zero damaged, sampled peak 5.72 MiB. See SCALE.md for procedure and limits.

Formatting: the default cargo fmt check reports existing compact-style differences throughout the source tree; no repository-wide reformat was applied.

Known performance limitation: the 10 GiB upload target (at least 80% of a plain disk write) is not repeatably passing on the development machine. See RELEASE_NOTES.md.

Not checked: running the Linux binary (WSL/Docker unavailable), systemd/SCM installation, or physical ENOSPC (reserve_bytes emulation is tested).
