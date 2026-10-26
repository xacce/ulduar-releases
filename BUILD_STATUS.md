# Ulduar v0.1.3 build record, 2026-10-27

Source commit: `c32dc646e09a13f6e8985eb780652d4f620b9c63`.

Passed: 44 functional release tests, including offline cache/ABI, directory-enumeration failure regression and real-process storage root/restart coverage; workspace and packaged SDK Clippy with warnings denied; independent packaged Rust consumer over HTTP and HTTPS; all offline input forms, process exit without destructors/reopen, cancellation/resume, separate payload rate limits, remote refetch, offline with unreachable server, TLS trust/rejection; Windows x64 MSVC server/native client builds; Linux x64 musl server and glibc 2.17 GNU native client cross-builds; Windows CLI/curl on default TCP 7443 with listen omitted and on explicitly configured TCP 8443.

Binaries are unsigned. SHA256SUMS records asset hashes.

Object-count audit: one million real 8-byte objects on Windows/NTFS passed on v0.1.1 production code before the error-path correction. Startup 0.507 s; presence checks 45.16 s; server sampled peak 14.12 MiB; full verify of 1,000,001 objects 428.81 s, zero damaged, sampled peak 5.72 MiB. See SCALE.md for procedure and limits.

Offline client scale test: 100,000 real 8-byte files directly seeded on Windows/NTFS; 49,999 evictions 13.471 s, 50,000 pending files plus a pin preserved; 50,001-object reopen 0.014 s. This is separate from normal import or network throughput.

Formatting: the default cargo fmt check reports existing compact-style differences throughout the source tree; no repository-wide reformat was applied.

Known performance limitation: the 10 GiB upload target (at least 80% of a plain disk write) is not repeatably passing on the development machine. See RELEASE_NOTES.md.

Not checked: running Linux binaries (WSL/Docker unavailable), systemd/SCM installation, physical ENOSPC (reserve_bytes emulation is tested), or abrupt power-loss recovery. One process owns each client cache directory; pending data never expires.
