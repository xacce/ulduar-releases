# Ulduar v0.1.2

Ulduar stores binary assets by their BLAKE3 content hash, skips duplicate uploads,
and resumes interrupted transfers. This update fixes incomplete integrity scans
and adds a reproducible one-million-object validation test and report.

## Integrity verification fix

Previously, `verify` could ignore an error returned while enumerating a directory
and finish successfully after an incomplete scan. It now fails on that error and
reports the directory path. A regression test first checks a healthy object,
then injects an enumeration failure: the old behavior fails the test and the
corrected behavior passes. Healthy objects, hash checks, storage paths, and the
HTTP API are unchanged.

## One million stored objects

A real Windows/NTFS run created one million distinct 8-byte files in their correct
hash paths, then exercised the server and CLI:

- Startup with the populated store: 0.507 seconds.
- Presence checks for every stored object plus absent IDs: 45.16 seconds, correct.
- 1,000 verified downloads plus a new HTTP upload/download: passed.
- Sampled server peak working set: 14.12 MiB.
- Restart and retrieve existing/new objects: passed.
- Full `verify`: 1,000,001 objects checked, zero damaged, 428.81 seconds;
  sampled verifier peak working set 5.72 MiB.

These measurements use v0.1.1 production code, before the directory-error fix;
the fix has its own regression test and functional checks. The fixture was
seeded directly on disk, not uploaded a million times. Filesystem caches were
warm. This does not measure bulk upload speed, cold-cache performance, a million
simultaneous clients, or Linux runtime performance. See `SCALE.md` for the full
procedure and operational limits.

## Download and upgrade

- Windows x64: `ulduar-v0.1.2-windows-x86_64.zip`.
- Linux x64: `ulduar-v0.1.2-linux-x86_64.tar.gz` (static musl).
- Archives include the executable, README, installation guide, release notes,
  object-count report, and example config. Verify files against `SHA256SUMS`.

Stop the existing service, replace its executable, and start it again. Preserve
your config, certificates, keys, and storage. No data migration or service
reinstallation is required when those paths stay the same.

**TCP 7443 remains the default.** Set `listen` to change it and restart the
service; update firewall rules and client URLs too. The storage folder is chosen
with `root_dir` before issuing keys/installing; the legacy `data_dir` alias remains
supported. Absolute paths work, and relative paths resolve against the config
folder. Changing the root does not move files; see `INSTALL.md` for migration.
Windows SCM and Linux systemd autostart commands remain documented there.

## Validation and limits

32 functional release tests, Clippy with warnings denied, native Windows x64 and
Linux x64 musl cross-builds pass. Windows CLI/curl checks cover omitted `listen`
(default 7443), explicit 8443, a selected storage root, uploads/downloads, and
`verify`. Uploaded assets are downloaded and checked against SHA256SUMS before
publication. Binaries are unsigned.

Linux execution, installing/running SCM or systemd on a target host, and physical
ENOSPC remain unverified locally. Low-space behavior is tested with the reserve
setting. Large numbers of partial uploads or damaged objects are separate from
a healthy completed-object count; see the scale report.

The existing 10 GiB upload target of 80% of a plain disk write is not repeatably
passing. Last representative results were 1.00 GiB/s upload versus 1.40 GiB/s
plain write (72%), with 88 MiB sampled server memory. No throughput retuning or
large-transfer rebenchmark was done in this update.

Source remains private in `xacce/ulduar`; the public repository contains binaries
and documentation. GitHub Actions remain disabled.
