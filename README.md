# Ulduar

A content-addressed asset store and a blocking Rust client. Objects are immutable
bytes identified by their 64-character, lowercase BLAKE3 hash. The server checks
every completed upload before publishing it with an atomic rename. Interrupted
uploads and downloads can be resumed.

Use Ulduar to share build assets, VFX caches, or other large binary files between
machines. Identical content has one ID, so clients can skip files already stored
and resume interrupted transfers. It stores the bytes; your pipeline keeps the
filenames and the mapping from assets to their IDs.

Download Windows x64 and Linux x64 binaries from
[Ulduar releases](https://github.com/xacce/ulduar-releases/releases).
See [INSTALL.md](INSTALL.md) for setup, certificates, firewall access, and
installation from source. Source code is private; releases contain binaries and
documentation. No GitHub Actions are used.

## Port and configuration

**The default port is TCP 7443.** The server reads `ulduar.toml` beside its
executable, or the file supplied with `--config`. The example uses HTTPS and
listens on every IPv4 interface:

```toml
listen = "0.0.0.0:7443"
```

Omitting `listen` also uses `0.0.0.0:7443`. To use another port, edit the same
setting, for example `listen = "0.0.0.0:8443"`, then restart the service and update
your firewall rule and client URLs. Connect using the server's hostname or IP,
such as `https://assets.example.com:7443`; `0.0.0.0` is the bind address.
`plain_http = true` changes the protocol to HTTP on the same configured port.

## Automatic startup

After placing the executable, configuration, certificate, and key in their final
locations, run `install` once with administrator/root privileges:

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

`install` registers and immediately starts the service. Windows SCM or Linux
systemd starts it again after a reboot and restarts it after failures. You do not
need a separate Startup-folder entry, scheduled task, or cron job.

| Command | Purpose |
| --- | --- |
| `install` | Register the OS service, enable boot startup, and start it now |
| `uninstall` | Stop and remove the service registration; keep stored data |
| `serve` | Run in the current console for manual operation or troubleshooting; stop with Ctrl+C |
| `key add/list/rm` | Issue, inspect, and revoke access keys |
| `verify` | Optional integrity check: re-hash objects and move damaged files to `data_dir/quarantine` |

The OS runs the server process after `install`; you do not also launch `serve`
on that port. Linux's generated unit calls `serve` internally; Windows uses an
internal SCM entry point. `verify` is a maintenance command, not a startup step.

## HTTP API

Every endpoint requires `Authorization: Bearer <key>`. Keys have independent read
and write permissions. There is no endpoint to list, search, or delete objects.
API errors normally have a JSON body such as `{"error":"no such object"}`.

| Request | Permission | Result |
| --- | --- | --- |
| `GET /objects/<id>` | read | `200`, object bytes; `404` if absent |
| `HEAD /objects/<id>` | read | The GET headers without the body |
| `PUT /objects/<id>` | write | Upload the whole object; `201` when stored, `200` if already present |
| `POST /missing` | read or write | JSON array of supplied IDs that are absent |
| `GET /uploads/<id>` | write | `{"offset":123}` for this key's partial upload, or offset zero |
| `PATCH /uploads/<id>` | write | Append at `Upload-Offset`; `204` with the new `Upload-Offset` until completed |

Object responses include `Content-Length`, `ETag: "<id>"`, `Accept-Ranges: bytes`,
and `Cache-Control: private, max-age=31536000, immutable`. GET and HEAD support a
single byte range: `bytes=100-199`, `bytes=100-`, or `bytes=-100`. Successful ranges
return `206` and `Content-Range`; unsatisfiable ranges return `416`. Multiple ranges
and unsupported range syntax are ignored and serve the full object.

`POST /missing` accepts a JSON array of at most 10,000 IDs and a body of at most
1 MiB. It only reports on the IDs supplied by the caller.

For resumable uploads, get the current offset and send a PATCH with:

```http
Upload-Offset: 123
Upload-Length: 1000
Content-Length: 877
Upload-Complete: 1
```

`Upload-Offset` is required. `Upload-Length`, when supplied, is the whole object's
size and enables an early free-space check. Set `Upload-Complete: 1` on the final
PATCH to verify and publish the object; without it, bytes remain a partial upload.
A wrong offset returns `409` with the server's `Upload-Offset`. Partial uploads
are isolated by key. A completed PUT interrupted in transit can also be resumed
through PATCH. Both PUT and PATCH accept streaming bodies.

| Status | Meaning |
| --- | --- |
| `400` | Invalid ID, headers, JSON, or interrupted upload |
| `401` | Missing, invalid, or revoked key |
| `403` | Key lacks permission |
| `404` | Missing object or endpoint |
| `408` | Upload body idle timeout; the partial can be resumed |
| `409` | Wrong upload offset or another upload of the same object by the same key is active |
| `413` | `/missing` body exceeds its byte limit |
| `416` | Byte range outside the object |
| `422` | Uploaded bytes do not hash to the supplied ID; the partial is removed |
| `507` | Insufficient free space; reads remain available |

An upload of an existing object is idempotent. The server can answer before
consuming a rejected or redundant large body, so clients must tolerate a closed
connection and query the object or upload state before retrying.

## Rust client

Add `ulduar-client` as a path dependency from this workspace:

```toml
[dependencies]
ulduar-client = { path = "path/to/ulduar/client" }
```

```rust,no_run
use std::path::Path;
use ulduar_client::Client;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::new("https://assets.example.com:7443", "uld_your_key")?;
    let id = client.put_file(Path::new("asset.bin"))?;
    client.get_to_file(&id, Path::new("download.bin"))?;
    Ok(())
}
```

`with_root_certificate` adds a PEM CA or self-signed server certificate to the
trusted roots. `put_bytes`, `put_from`, `missing`, `has`, `upload_offset`,
`get_bytes`, and `get_to_writer` are also available. `put_from` takes a known hash,
length, and a reader factory that can reopen at a requested offset. The client
skips stored objects, retries transient failures (eight attempts by default), and
verifies downloads. File downloads use a `.ulduar-part` file and rename it after
verification. `get_to_writer` streams without retrying and may leave partial
output on failure. This is a blocking API; call it outside async runtime tasks.

## Storage and operations

Paths are relative to `data_dir`:

```text
objects/<first 2 hex>/<next 2 hex>/<id>
tmp/<id>.<key id>.part
quarantine/<id>
keys.json
logs/
```

Keep `tmp` and `objects` on the same volume. Stale partial uploads are swept
hourly after the configured age. Keys are stored as hashes; secrets are printed
only when issued. Revocations affect new requests within about a second; a
transfer already authorized may finish. TLS certificates are checked for changes
every ten seconds; an invalid replacement leaves the previous certificate active.
Logs rotate daily and retain fourteen files. Set `ULDUAR_LOG` to change the log
filter (default `info`).

`ulduar verify` re-hashes stored objects and moves damaged ones to quarantine.
It exits unsuccessfully if corruption is found. There is no garbage collector for
completed objects; plan capacity and backups separately.

## Build and checks

```sh
cargo build --release --locked
cargo test --release --locked
cargo clippy --all-targets --locked -- -D warnings
```

The large transfer test is opt-in, needs at least 11 GiB free, and compares a
10 GiB upload with a synced plain write on the same disk while sampling server
memory. Its gates are at least 80% of disk-write throughput and under 256 MiB of
server working set. Run it without other disk-heavy jobs:

```sh
cargo test --release --locked --test big -- --ignored --nocapture
```

For a smaller diagnostic run, set `ULDUAR_BIG_GIB=2` (`$env:ULDUAR_BIG_GIB = '2'`
in PowerShell). This does not replace the 10 GiB check.
