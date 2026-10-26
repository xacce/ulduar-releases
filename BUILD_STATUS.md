# Ulduar v0.1.0 build record, 2026-10-26

Source commit: `ebe77d87f3c769cd209d8eb9c0db026f3f939477`.

Passed: 25 release tests; Clippy with warnings denied; standalone client check; Windows x64 MSVC build; Linux x64 musl cross-build; Windows CLI/curl on default TCP 7443 with listen omitted and on explicitly configured TCP 8443.

Known performance limitation: the 10 GiB upload target (at least 80% of a plain disk write) is not repeatably passing on the development machine. See RELEASE_NOTES.md.

Not checked: running the Linux binary (WSL/Docker unavailable), systemd/SCM installation, or physical ENOSPC (reserve_bytes emulation is tested).
