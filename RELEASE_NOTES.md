# Ulduar v0.1.1

Ulduar stores binary assets by their BLAKE3 content hash, skips duplicate uploads,
and resumes interrupted transfers. This update makes the storage location an
explicit installation setting.

## Choose the storage root

Set `root_dir` in `ulduar.toml` before issuing access keys or running `install`.
Windows example:

```toml
root_dir = 'D:\UlduarData'
```

Linux example:

```toml
root_dir = "/srv/ulduar"
```

Objects, partial uploads, `keys.json`, quarantine, and default logs are stored
under that folder. The executable and config can live elsewhere. Relative roots,
including the example's `root_dir = "data"`, resolve against the config directory,
independently of the shell or service working directory. `install` now prints
the resolved storage root. An explicit `log_dir` keeps its existing meaning.

Existing `data_dir` configs work unchanged. You can rename that setting to
`root_dir` without changing the path; do not specify both. No data-format or HTTP
API changes are required. Changing the configured path does not move files:
stop the service, copy the entire old storage folder including `keys.json`, edit
the config, grant write access, and restart. See `INSTALL.md` for migration and
rollback steps. Keep the old copy until the new location is verified.

## Downloads and setup

- Windows x64: `ulduar-v0.1.1-windows-x86_64.zip`.
- Linux x64: `ulduar-v0.1.1-linux-x86_64.tar.gz` (statically linked musl).
- Check the archive against `SHA256SUMS`. Archives include the executable,
  `README.md`, `INSTALL.md`, release notes, and `ulduar.example.toml`.

**The default port remains TCP 7443.** Omitting `listen` binds `0.0.0.0:7443`.
To change it, set e.g. `listen = "0.0.0.0:8443"`, restart the service, and update
the firewall and client URLs. The example uses HTTPS; supply your own PEM
certificate and private key.

Place the files in permanent locations, select `root_dir`, and issue a key with
`key add workstation --read --write`. Then enable automatic startup:

```powershell
# Windows: elevated PowerShell
C:\Ulduar\ulduar.exe --config C:\Ulduar\ulduar.toml install
```

```sh
# Linux: systemd host
sudo /usr/local/bin/ulduar --config /etc/ulduar/ulduar.toml install
```

`install` registers and starts the service and enables startup after reboot.
For an existing installation, stop the service, replace the executable, and start
it again. Keep your existing config, certificates, keys, and stored files.

## Validation and known limits

All 31 functional release tests and Clippy pass on Windows, alongside native
Windows x64 and Linux x64 musl cross-builds. New coverage checks absolute roots
outside the config directory, relative roots, spaces in paths, legacy `data_dir`,
conflicting settings, and explicit log paths. A real server/CLI test confirms
selected locations for keys, uploaded objects, temporary storage, logs, and
quarantine, and verifies access after a process restart from a different working
directory. Release checks also cover ports 7443 and 8443, archive contents, and
checksums. Binaries are unsigned; SHA256 checksums are provided.

The following have **not** been verified locally:

- Running the Linux binary on Linux; WSL and Docker are unavailable here.
- Installing/running the systemd or Windows SCM service on a target host.
- A physically full disk; low-space behavior is tested through `reserve_bytes`.

The existing performance target of uploading 10 GiB at 80% of a plain disk write
is **not consistently met** on the development machine. The last representative
measurements were about 1.00 GiB/s upload versus 1.40 GiB/s plain write (72%),
0.86 GiB/s verified download, and 88 MiB sampled server memory. Performance was
not retuned or rebenchmarked for this configuration update.

Source code remains in private `xacce/ulduar`. The public release repository
contains binaries and documentation. No GitHub Actions are used.
