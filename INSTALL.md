# Install Ulduar

Ulduar stores large binary assets by their BLAKE3 content hash. Use it to share
assets between machines, skip duplicate uploads, and resume interrupted transfers.
Download the Windows x64 zip or Linux x64 musl tar.gz from
[the public releases](https://github.com/xacce/ulduar-releases/releases).
Each archive contains the executable, README, this guide, and an example config.
Compare its SHA256 with the release's `SHA256SUMS` before extracting it.

## Port, configuration, and certificates

**The default is TCP port 7443 on all IPv4 interfaces.** Configuration lives in
`ulduar.toml` beside the executable, or at the path supplied with `--config`.
Copy `ulduar.example.toml` to that location before starting or installing.

```toml
listen = "0.0.0.0:7443"
data_dir = "data"
tls_cert = "server.crt"
tls_key = "server.key"
```

Omitting `listen` also binds `0.0.0.0:7443`. To change the port, edit the setting:

```toml
listen = "0.0.0.0:8443"
```

Restart the service afterward (commands below), allow the new TCP port in the
host/cloud firewall, and update client URLs, for example
`https://assets.example.com:8443`. Connect to a hostname or server IP;
`0.0.0.0` means the local bind address. No separate administration port is used.

Supply a PEM certificate chain and PEM private key matching your server hostname.
All relative paths are resolved against the configuration file's directory.
For a local test, OpenSSL can generate a self-signed certificate:

```sh
openssl req -x509 -newkey rsa:3072 -sha256 -nodes -days 365 -keyout server.key -out server.crt -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

Place those files beside `ulduar.toml`; the test certificate covers localhost,
not a remote production hostname. Protect the private key and the data directory.

For a TLS-terminating proxy or a local HTTP test, remove `tls_cert` and `tls_key`
and set `plain_http = true`. This serves HTTP on the same configured port. Bearer
keys travel unencrypted on that connection, so restrict access to it.

The default free-space reserve is 2 GiB. Keep `data_dir/tmp` and
`data_dir/objects` on one volume and leave room for partial uploads. Unknown
configuration fields are rejected.

## Windows: install once for automatic startup

Extract the zip into a permanent directory, such as `C:\Ulduar`. In PowerShell:

```powershell
Get-FileHash .\ulduar-v0.1.0-windows-x86_64.zip -Algorithm SHA256
Expand-Archive .\ulduar-v0.1.0-windows-x86_64.zip C:\Ulduar
Copy-Item C:\Ulduar\ulduar.example.toml C:\Ulduar\ulduar.toml
# Edit ulduar.toml and place server.crt and server.key in C:\Ulduar.
C:\Ulduar\ulduar.exe --config C:\Ulduar\ulduar.toml key add workstation --read --write
```

Save the printed key securely; the secret is shown only once. Then, from an
**elevated PowerShell**, install and start the service:

```powershell
C:\Ulduar\ulduar.exe --config C:\Ulduar\ulduar.toml install
Get-Service ulduar
```

This creates the `ulduar` Windows SCM service with automatic startup, starts it
immediately, and configures restart after failures. Windows starts it after a
reboot without a console window or user login. It runs as LocalSystem, which must
be able to read the config/certificate/key and write the data directory.

Allow TCP 7443 in the server's Windows and network/cloud firewall. For example,
in elevated PowerShell:

```powershell
New-NetFirewallRule -DisplayName 'Ulduar TCP 7443' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 7443
```

Use your configured port if you changed it. Restart after a configuration edit:

```powershell
Restart-Service ulduar
```

Stop and remove the service registration, while keeping stored data:

```powershell
C:\Ulduar\ulduar.exe uninstall
```

## Linux: install once for automatic startup

On an x86_64 Linux host with systemd:

```sh
sha256sum -c SHA256SUMS --ignore-missing
tar -xzf ulduar-v0.1.0-linux-x86_64.tar.gz
sudo install -m 0755 ulduar /usr/local/bin/ulduar
sudo mkdir -p /etc/ulduar /var/lib/ulduar
sudo cp ulduar.example.toml /etc/ulduar/ulduar.toml
```

Edit `/etc/ulduar/ulduar.toml`: use `data_dir = "/var/lib/ulduar"` and your chosen
`listen` value (default `0.0.0.0:7443`). Place `server.crt` and `server.key` in
`/etc/ulduar`, or use absolute certificate paths in the config.

```sh
sudo chmod 600 /etc/ulduar/server.key
sudo /usr/local/bin/ulduar --config /etc/ulduar/ulduar.toml key add workstation --read --write
sudo /usr/local/bin/ulduar --config /etc/ulduar/ulduar.toml install
systemctl status ulduar
```

Save the printed key securely. `install` writes
`/etc/systemd/system/ulduar.service`, enables boot startup, and starts the service
immediately. systemd restarts it if it exits. The generated unit runs as root;
to use a dedicated account, grant it access to the config, certificates, and data,
then set `User=` and `Group=` with `sudo systemctl edit ulduar` and restart.

Allow TCP 7443 (or your configured port) in the host and network/cloud firewall.
After editing the configuration:

```sh
sudo systemctl restart ulduar
systemctl status ulduar
journalctl -u ulduar
```

To stop and remove the service registration while keeping data:

```sh
sudo /usr/local/bin/ulduar uninstall
```

## Why install, uninstall, and serve are separate

- `install` registers the OS service, enables boot startup, and starts it now.
  Run it once after putting the files in their permanent locations.
- `uninstall` stops and removes that registration; it retains the data files.
- `serve` runs the server in your current console until Ctrl+C. Use it for manual
  operation, troubleshooting, or your own process supervisor. It does not register
  an OS service. Linux's generated systemd unit invokes it internally.

After `install`, the OS runs the process for you. Do not start a second `serve`
process on the same address and port. You do not need cron, the Windows Startup
folder, or a scheduled task for automatic startup.

## Check that the server is reachable

Use the key issued above and a hostname matching the certificate. PowerShell:

```powershell
$key = 'uld_replace_with_your_key'
curl.exe --fail-with-body --cacert C:\Ulduar\server.crt -H "Authorization: Bearer $key" -H 'Content-Type: application/json' --data-binary '[]' https://localhost:7443/missing
```

Bash on Linux:

```sh
read -r -s -p 'Ulduar key: ' ULDUAR_KEY; echo
curl --fail-with-body --cacert /etc/ulduar/server.crt -H "Authorization: Bearer $ULDUAR_KEY" -H 'Content-Type: application/json' --data-binary '[]' https://localhost:7443/missing
```

The expected response is `[]`. From another machine, replace `localhost` with the
server's hostname/IP. If using a publicly trusted certificate, omit `--cacert`.
For plain HTTP, use an `http://` URL and omit `--cacert`.

For a complete transfer check with the separate `b3sum` utility installed:

```sh
ID=$(b3sum asset.bin | cut -d ' ' -f 1)
curl --fail-with-body --cacert /etc/ulduar/server.crt -H "Authorization: Bearer $ULDUAR_KEY" -T asset.bin "https://localhost:7443/objects/$ID"
curl --fail-with-body --cacert /etc/ulduar/server.crt -H "Authorization: Bearer $ULDUAR_KEY" -o downloaded.bin "https://localhost:7443/objects/$ID"
cmp asset.bin downloaded.bin
```

## What verify does

`verify` is an optional integrity check, not an installation or startup step. It
reads every stored object, computes its BLAKE3 hash, and compares it with its ID.
Damaged files move to `data_dir/quarantine` and stop being served; the command
returns a nonzero exit code if corruption is found. Healthy files stay in place.

```sh
ulduar --config /absolute/path/ulduar.toml verify
ulduar --config /absolute/path/ulduar.toml key list
ulduar --config /absolute/path/ulduar.toml key rm workstation
```

Revoked keys stop authorizing new requests within about a second. Logs rotate
daily under `data_dir/logs` unless `log_dir` is set.

## Build on the server from private source

You need access to `xacce/ulduar`, Rust/Cargo, and a native C/C++ build toolchain.
On Linux:

```sh
git clone git@github.com:xacce/ulduar.git
cd ulduar
cargo build --release --locked -p ulduar
sudo install -m 0755 target/release/ulduar /usr/local/bin/ulduar
```

Continue with the configuration, certificate, key, and `install` steps above.
On Windows, the same Cargo command produces `target\release\ulduar.exe`; copy it
to `C:\Ulduar` before installing. For an upgrade, stop the existing service,
replace the executable, and start the service again. Keep your config, keys,
certificates, and stored data; do not rerun first-install config-copy steps.

## Validation limits

Windows API/CLI tests and Linux cross-compilation are checked locally. Executing
the Linux binary, systemd/SCM installation on a target host, and physical ENOSPC
have not been checked locally. Low-space behavior is tested using `reserve_bytes`.
The 10 GiB upload target of 80% of disk-write throughput is not consistently met
on the development machine. See the release notes for the full validation record.
