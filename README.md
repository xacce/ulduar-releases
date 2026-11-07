# Ulduar

A content-addressed asset server and an offline-first binary Rust SDK. Objects are immutable
bytes identified by their 64-character, lowercase BLAKE3 hash. The server checks
every completed upload before publishing it with an atomic rename. Interrupted
uploads and downloads can be resumed.

Use Ulduar to share build assets, VFX caches, or other large binary files between
machines. Since 2.0.0 a second storage, v2, adds users with permanent tokens,
projects with read/write/delete rights, chunk-level deduplication across projects,
asset types and names; see [Storage v2 API](#storage-v2-api). Identical content has one ID, so clients can skip files already stored
and resume interrupted transfers. It stores the bytes; your pipeline keeps the
filenames and the mapping from assets to their IDs.

Download Windows x64 and Linux x64 binaries from
[Ulduar releases](https://github.com/xacce/ulduar-releases/releases).
See [INSTALL.md](INSTALL.md) for setup, certificates, firewall access, and
installation from source. Source code is private; releases contain binaries and
documentation. No GitHub Actions are used.

For people rather than pipelines, [Ulduar Desktop](desktop/README.md) (`DESKTOP.md` in a
release) is a portable client: accounts, projects, drag-and-drop upload, download, search,
and one-button server installation on Ubuntu 24 over SSH.

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

## Choose the storage root

Set `root_dir` in `ulduar.toml` **before issuing keys or running `install`**.
This is where Ulduar puts objects, partial uploads, access-key records, quarantine,
and logs. For example, on Windows:

```toml
root_dir = 'D:\UlduarData'
```

On Linux:

```toml
root_dir = "/srv/ulduar"
```

The example's `root_dir = "data"` uses a folder beside the configuration file.
Relative paths are resolved against that file's directory, including when the
service starts at boot. The executable and config can live elsewhere. `install`
prints the resolved storage root. The service account needs write access to it.
An explicit `log_dir` overrides the default `<root_dir>/logs` location.

Existing `data_dir` settings remain supported with the same meaning; use either
`root_dir` or `data_dir`, never both. Changing the path does not move existing
files. See [INSTALL.md](INSTALL.md#change-the-storage-root-of-an-existing-server)
for the stop, copy, and restart procedure.

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
| `key add/list/rm` | Issue, inspect, and revoke v1 access keys |
| `user`, `token`, `project`, `grant`, `revoke`, `grants`, `migrate` | Manage storage v2; see [Console](#console) |
| `verify` | Optional integrity check: re-hash objects and move damaged files to `root_dir/quarantine` |

The OS runs the server process after `install`; you do not also launch `serve`
on that port. Linux's generated unit calls `serve` internally; Windows uses an
internal SCM entry point. `verify` is a maintenance command, not a startup step.

## Automatic updates

From v0.1.5, `serve` and the installed service check the latest public release
each time they start. A newer version is installed over the running executable,
then the server restarts into it before accepting connections. Nothing else changes:
config, keys, certificates and stored data stay where they are.

An update is installed only when all of these hold:

- `SHA256SUMS.sig` is a valid Ed25519 signature of the release's `SHA256SUMS`
  by the Ulduar release key built into the executable;
- the release version is higher than the running one (no downgrades);
- the platform archive matches its SHA256 in `SHA256SUMS`;
- the extracted executable runs and reports exactly that version.

Otherwise, or when GitHub cannot be reached within a few seconds, the server
logs `update check skipped: <reason>` and starts the current version. The check
runs only at startup; restart the service to pick up a new release. The account
running the server needs write access to the executable's folder (the default
LocalSystem/root service accounts have it). On Windows the replaced file stays as
`ulduar.exe.<pid>.old` until the next start removes it.

The check runs inside the server process, under the service's own user and sandbox, not
under whoever ran the restart. A non-root `User=` without write access to the folder, or
`ProtectSystem=` / `ReadWritePaths=` excluding it, makes it log `update check skipped` on
every start; update by hand as described in INSTALL.md.

```toml
auto_update = false   # disable the check
# update_url = "https://github.com/xacce/ulduar-releases/releases/latest/download"
```

`update_url` points to any folder serving `SHA256SUMS`, `SHA256SUMS.sig` and the
archives of a release signed with the same key, for example an internal mirror.
v0.1.4 and earlier have no updater: install v0.1.5 manually once.

## HTTP API (v1)

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

## Storage v2 API

Ulduar 2.0.0 adds a second, independent storage under `root_dir/v2/` with its own
API under `/v2`. The v1 API above, its keys and its data are unchanged; v2 does
not see v1 objects until they are copied with `migrate`. The Rust SDK 2 uses only v2.

- **Users and tokens.** Every v2 request needs `Authorization: Bearer uld2_...`,
  a permanent token of a user. A user may hold many tokens; the server stores only
  their hashes. v1 keys are not accepted on `/v2`.
- **Projects and rights.** Objects belong to projects, addressed by name (1-64 of
  `A-Z a-z 0-9 - _ .`). A user has read, write and delete rights per project; delete
  is never given by default. A user with the create-project permission may create
  projects through the API and receives read and write in them.
- **Chunks and dedup.** Clients cut content into content-defined chunks (FastCDC,
  64 KiB min, 256 KiB average, 1 MiB max) and upload only those the server lacks.
  Chunks and objects are shared by all projects: content known anywhere on the
  server is linked into another project without being sent again. Anyone who can
  write a project and knows an object's hash can link that object into it.
- **Types.** An entry is (project, type, hash). The type is a user-defined u32
  (FourCC by convention, `MESH` = little-endian bytes), not part of the hash; 0 is
  untyped. Publishing typed bytes removes their untyped entry.
- **Names.** Optional, one current name per (project, object), 1-4096 bytes, kept
  in a separate `v2/names.db`. An upload with a name replaces it, an upload without
  one keeps it. No history and no name deletion; deleting an object drops its name.

| Request | Right | Result |
| --- | --- | --- |
| `GET /v2/me` | any token | `{"user","token","create_project"}` |
| `GET /v2/projects` | any token | `[{"name","read","write","delete"}]` with any right |
| `POST /v2/projects` `{"name"}` | create-project | `201`; `409` if the name is taken |
| `POST /v2/projects/<p>/chunks/missing` | write | JSON array of the supplied chunk hashes the server lacks |
| `PUT /v2/projects/<p>/chunks/<h>` | write | Chunk bytes, at most 4 MiB; `201` new, `200` known, `422` hash mismatch |
| `GET /v2/projects/<p>/chunks/<h>` | read | Chunk bytes |
| `PUT /v2/projects/<p>/objects/<h>` | write | `{"type","name"?,"chunks"?}`: link or commit, see below |
| `GET`/`HEAD /v2/projects/<p>/objects/<h>` | read | Object bytes with single-range support, as v1 |
| `GET /v2/projects/<p>/objects/<h>/manifest` | read | The object's ordered `[chunk hash, length]` list |
| `DELETE /v2/projects/<p>/objects/<h>` | delete | `204`; removes every type of the object and its name from the project |
| `POST /v2/projects/<p>/objects/missing` | read or write | Supplied object hashes this project lacks |
| `GET /v2/projects/<p>/objects?type=&after=&limit=` | read | `{"objects":[...],"next"}`; type must be nonzero |
| `GET /v2/projects/<p>/names?name=&after=&limit=` | read | Page of objects carrying the name |
| `GET /v2/projects/<p>/names/<h>` | read | `{"name"}`; `404` without one |
| `GET /v2/projects/<p>/files?q=&after=&limit=` | read | `{"files":[{"hash","name","size","types","created"}],"next"}`: named objects by (name, hash), then unnamed by hash; see below |
| `GET /v2/projects/<p>/stats` | read | `{"objects","bytes"}`: distinct objects of the project and their total size |

`PUT .../objects/<h>` without `chunks` links content the server already has:
`201` if the project gains the entry, `200` if it already had it, `409
{"unknown":true}` if the server does not know the object. With `"chunks":
[[hash, length], ...]` it commits the object after re-hashing the chunks into
`<h>`: `409 {"missing":[...]}` lists chunks to send first, `422` means the chunks
do not make up content with this hash. Pages are ordered by hash; `limit`
defaults to 1000, at most 10,000; pass `next` as `after` until it is absent.
Batches of hashes hold at most 10,000 entries in a body of at most 1 MiB.
A project the token has no right in answers `404 {"error":"no such project"}`, a
missing right `403`.

`GET .../files` is the catalogue. `q` is whitespace-separated terms, each a
case-insensitive Unicode substring of the name, all of which must match; a `q` of 4
to 64 hex characters also matches hashes with that prefix, which is the only way to
find unnamed objects by search. `limit` defaults to 1000 and is clamped to 1-5000;
`name` is `null` for unnamed objects and `types` lists the nonzero types. A search
scans for at most about 2 seconds per request, so a page may be short or even empty
while `next` is set: keep passing `next` as `after` until it is absent. Cursors are
opaque. Names whose object is gone are skipped. Renaming is `PUT .../objects/<h>`
with the object's existing type and the new `name`.

Chunks no object uses are deleted after `upload_sweep_days`, checked every ten
minutes, so a slow upload never loses its chunks. Data layout:

```text
v2/chunks/<2 hex>/<2 hex>/<chunk hash>
v2/meta.db, v2/names.db   (SQLite WAL: users, token hashes, projects, rights, objects, entries, names)
v2/tmp/, v2/trash/, v2/quarantine/
```

### Console

Run these with the same `--config` as the server; they work while it runs.

| Command | Effect |
| --- | --- |
| `user add <name> [--create-projects]` | Add a user |
| `user list`, `user rm <name>` | List users; remove one with tokens and rights (uploads stay) |
| `user create-projects <name> <true\|false>` | Allow or forbid creating projects through the API |
| `token add <user> [--label text]` | Issue a permanent token; printed once, stored as a hash |
| `token list [user]`, `token rm <id>` | List tokens (never secrets); revoke one |
| `project add <name>`, `project list` | Create and list projects |
| `grant <user> <project> [--read] [--write] [--delete]` | Set the user's rights exactly; rights not given are removed |
| `revoke <user> <project>`, `grants [user]` | Remove all rights; list rights |
| `migrate <project> [--type MESH] [ids...]` | Copy v1 objects (all or the given ids) into a v2 project; v1 stays |
| `verify` | Also re-hashes every v2 chunk and reports chunks in use but missing |

## Rust SDK

Download `ulduar-v2.0.0-rust-sdk.zip` from the public release, extract its
`ulduar-sdk/` folder into your application's `vendor/`, and add:

```toml
[dependencies]
ulduar-sdk = { path = "vendor/ulduar-sdk" }
```

```rust,no_run
use ulduar_sdk::{AssetType, ImportMode, Meta, Remote, Store, StoreOptions, SyncOptions};

async fn assets() -> Result<(), ulduar_sdk::Error> {
    let remote = Remote::new("https://assets.example.com:7443", "uld2_your_token");
    let store = Store::open(StoreOptions::remote("application-cache/assets", "game", remote)).await?;
    let meta = Meta::typed(AssetType::fourcc(b"MESH")).named("props/crate");
    let written = store.import_file("asset.bin", ImportMode::Copy, meta).await?;
    // Locally committed. No upload until the application asks for synchronization.
    let lease = store.get(written.hash).await?;
    println!("Read this path while holding the lease: {}", lease.path().display());
    lease.release().await?;
    store.synchronize(SyncOptions { upload_bps: 2 * 1024 * 1024, download_bps: 0 }).await?;
    store.close().await?;
    Ok(())
}
```

The SDK contains prebuilt Windows x64 MSVC and Linux x64 GNU client libraries
and Rust bindings. Cargo builds the small bindings and their public dependencies;
it does not need access to the private implementation. The matching native
library is embedded into the application and extracted into a per-user cache on
first use, so deployment does not need a separate DLL beside the executable.

SDK 2 works with storage v2 only: a store names its project, each write carries
an optional type and name, and `list`, `named`, `name`, `delete`, `projects`,
`create_project` and `me` query the server. SDK 0.1.x remains usable with v1.

Writes accept owned/shared bytes, copied/moved files, readers and encoder callbacks.
Reads by hash use local data first and download verified missing objects. Work runs
on SDK workers; await operations or poll `try_complete()` in the application's main
loop. Only the application starts uploads/cleanup with `synchronize`, with separate
upload/download byte-per-second limits. `stats()` and operation progress are
memory-only snapshots for UI size and status displays.

The cache keeps objects for 24 hours after a write or hit, grants a one-hour grace
on startup, protects active leases across processes, and never expires unuploaded
writes. Multiple applications and different EXEs can share one cache directory.
Only one cleaner runs at a time; others skip cleanup and keep working. See
[SDK.md](https://github.com/xacce/ulduar-releases/blob/main/SDK.md) for the full API,
retention, recovery, TLS, deployment and integration instructions.

The private workspace's blocking `ulduar-client` transport remains available to
source builds. The public SDK uses the local store facade described above.

## Storage and operations

v1 paths are relative to the configured `root_dir` (v2 is under `v2/`, above):

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
It exits unsuccessfully if corruption is found or a directory cannot be fully
enumerated. There is no garbage collector for
completed objects; plan capacity and backups separately.

## Large object counts

Objects are partitioned by the first four hash characters: 256 top-level buckets
and up to 65,536 leaf directories. At one million distinct objects this averages
about 15 files per leaf. Requests resolve a known hash directly; they do not scan
the entire store or load an object index into memory. Startup checks `tmp/`, not
every completed object.

`POST /missing` accepts at most 10,000 IDs per request. The Rust client's `missing`
method splits larger slices automatically. Call it with batches if your pipeline
must also bound its own input/output list memory. `verify` reads every object's
contents, so its running time depends on object count, total bytes, and storage.

See [SCALE.md](https://github.com/xacce/ulduar-releases/blob/main/SCALE.md) for the
reproducible million-object check, measured results, and the distinction between
stored object count and request concurrency. `SHA256SUMS` applies to files attached
to the corresponding release; repository documentation may be newer.

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

The separate object-count test creates one million real 8-byte objects and
checks startup, batched presence queries, sample downloads, a new HTTP upload,
restart, and full integrity verification. It seeds the fixture directly on disk;
it does not measure a million uploads. Leave at least 8 GiB free for filesystem
metadata and run it without other disk-heavy work:

```sh
cargo test --release --locked --test many -- --ignored --nocapture
```

`ULDUAR_MANY_OBJECTS` overrides the fixture count for diagnostics. The default
million-object run requires successful memory samples below 256 MiB for the
server and verifier. The temporary fixture is removed at the end.
