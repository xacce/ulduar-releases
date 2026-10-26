# Ulduar v0.1.0

Ulduar is a content-addressed asset store for sharing large binary assets between
machines. Files are identified by BLAKE3 hashes, duplicate uploads can be skipped,
and interrupted transfers can be resumed. Downloads are verified by the Rust
client. Use it for build assets, VFX caches, and other pipeline data.

## Downloads and setup

- Windows x64: `ulduar-v0.1.0-windows-x86_64.zip`.
- Linux x64: `ulduar-v0.1.0-linux-x86_64.tar.gz` (statically linked musl).
- Check the archive against `SHA256SUMS`. Each archive includes `README.md`,
  `INSTALL.md`, and `ulduar.example.toml`.

**The default port is TCP 7443**, with HTTPS in the example configuration. The
server binds `0.0.0.0:7443` even when `listen` is omitted. To change it, set
`listen = "0.0.0.0:8443"` in `ulduar.toml`, restart the service, and update the
firewall and client URLs.

Place the executable, config, PEM certificate, and private key in their permanent
locations. Issue an access key with `key add workstation --read --write`, then
enable automatic startup:

```powershell
# Windows: elevated PowerShell
C:\Ulduar\ulduar.exe --config C:\Ulduar\ulduar.toml install
Get-Service ulduar
```

```sh
# Linux: systemd host
sudo /usr/local/bin/ulduar --config /etc/ulduar/ulduar.toml install
systemctl status ulduar
```

`install` registers and starts the service immediately and enables startup after
reboot. `uninstall` stops/removes the service registration and retains data.
`serve` runs in the current console for manual operation; it is not an additional
step after `install`. `verify` optionally checks stored hashes and quarantines
damaged files. See `INSTALL.md` for certificates, firewall access, changing the
port, source builds, and curl checks.

## Included

- Authenticated HTTP/HTTPS API with separate read/write key permissions.
- Immutable objects, byte-range downloads, missing-object queries, resumable
  uploads, and atomic publication after hash verification.
- Windows SCM and Linux systemd service installers with automatic restart.
- Blocking Rust client in the private source repository. Transfers are also
  accessible directly through the HTTP API; a transfer-specific CLI is not included.
- Streaming improvements, bounded queues, and parallel server-side BLAKE3 hashing.

## Validation and known limits

All 25 functional release tests and Clippy pass on Windows, alongside
native Windows x64 and Linux x64 musl cross-builds. Release verification includes
the default port, explicit port overrides, CLI/curl requests, archive contents,
and checksums.

The following have **not** been verified locally:

- Running the Linux binary on Linux; WSL and Docker are unavailable here.
- Installing/running the systemd or Windows SCM service on a target host.
- A physically full disk; low-space behavior is tested through `reserve_bytes`.

The performance target of uploading 10 GiB at 80% of a plain disk write is **not
consistently met** on the development machine. Representative measurements after
the streaming changes were about 1.00 GiB/s upload versus 1.40 GiB/s plain write
(72%), 0.86 GiB/s verified download, and 88 MiB sampled server memory. These are
local loopback measurements, not a production throughput guarantee.

Source code remains in the private `xacce/ulduar` repository. The public release
repository contains binaries and documentation. No GitHub Actions are used.
