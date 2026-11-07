# Ulduar binary Rust SDK 2.1.2

Ulduar is an immutable asset store addressed by the 64-character lowercase BLAKE3
hash of each object's bytes. This SDK adds a durable local store to the remote
server: writes commit locally, reads resolve locally first, and your application
chooses when to upload and clean the cache. Identical content shares one object.

SDK 2 talks only to the server's storage v2 (Ulduar 2.0.0 or newer): objects live
in **projects**, every write may carry a user-defined **asset type** and a
**name**, and uploads send only the 64 KiB-1 MiB chunks the server does not
already have from any project. See "Projects, types and names" below. SDK 0.1.x
keeps working against the unchanged v1 API; the two use different cache
directories and different credentials (v1 keys, v2 user tokens).

## Install and deploy

Download `ulduar-v2.1.2-rust-sdk.zip` from the
[public release](https://github.com/xacce/ulduar-releases/releases/tag/v2.1.2),
verify its SHA256 against the release's `SHA256SUMS`, and extract `ulduar-sdk`
into your application's `vendor` directory:

```toml
[dependencies]
ulduar-sdk = { path = "vendor/ulduar-sdk" }
```

`ulduar-sdk-2.1.2.crate` contains the same bindings and native libraries in Cargo's
archive format; unpack it and use a path dependency. It is not published on
crates.io. Do not point Cargo at the private implementation repository.

The package contains a prebuilt Windows x64 MSVC DLL and Linux x64 GNU shared
library, plus a small Rust facade. Cargo compiles the facade and its public
support dependencies; it does not compile the private transport, SQLite or store
implementation. Supported application targets:

| Target | Native requirements |
| --- | --- |
| `x86_64-pc-windows-msvc` | Windows x64; standard MSVC runtime support |
| `x86_64-unknown-linux-gnu` | Linux x64 with glibc 2.17 or newer; loader must allow shared library execution |

Other targets, including Linux musl applications, ARM, macOS and 32-bit Windows,
need a matching native build and are rejected at compile time. The Linux server
executable is separately built with musl; that does not make the client SDK musl.
The GNU library is cross-built and inspected; Linux execution has not been tested
on this development machine.

The selected library is embedded into your application executable. On the first
`Store::open`, an SDK worker extracts it atomically into
`%LOCALAPPDATA%/ulduar-sdk/<binary-sha256>/` on Windows, or
`${XDG_CACHE_HOME:-$HOME/.cache}/ulduar-sdk/<binary-sha256>/` on Linux. Existing
bytes are checked before reuse. This directory must be writable and allow native
library loading; Linux `noexec` mounts cannot host it. You can deploy your
application executable without placing a DLL beside it. The loaded library stays
loaded until process exit, and the cache is reused across processes. Remove old
binary cache versions while applications using them are stopped. This binary cache
is separate from the asset cache selected below. Libraries are unsigned; release
hashes verify downloaded bytes, not a code-signing certificate.

## Open a store

```rust,no_run
use ulduar_sdk::{AssetType, Meta, Remote, Store, StoreOptions, SyncOptions};

const TEXTURE: AssetType = AssetType::fourcc(b"TEXR");

async fn example() -> Result<(), ulduar_sdk::Error> {
    let remote = Remote::new(
        "https://assets.example.com:7443",
        std::env::var("ULDUAR_TOKEN").expect("set ULDUAR_TOKEN"),
    );
    let options = StoreOptions::remote("D:/MyApplication/assets", "game", remote);
    let store = Store::open(options).await?;
    let written = store.write_bytes(b"asset bytes".to_vec(), Meta::typed(TEXTURE).named("hero/diffuse")).await?;
    // No upload has been attempted. Persist written.hash in your application.
    let bytes = store.read_bytes(written.hash.clone()).await?;
    assert_eq!(bytes, b"asset bytes");
    let stats = store.synchronize(SyncOptions {
        upload_bps: 2 * 1024 * 1024,
        download_bps: 0,
    }).await?;
    println!("{} bytes still pending", stats.pending_bytes);
    store.close().await?;
    Ok(())
}
```

Use an absolute root for a predictable installation location. Relative roots
resolve against the application's current working directory when opened. This
root contains `client.sqlite3`, its WAL files, small maintenance/session lock files,
sharded `objects` and staging `tmp`. It must be a dedicated writable local filesystem directory;
never reuse the server's `root_dir`. Network filesystems with unreliable locking,
SQLite WAL semantics or rename durability are unsupported.

Multiple processes, different EXE names, different application binary hashes and
application versions can open the same cache root simultaneously. For example,
SFX Engine and four background agents can all use one cache. Application identity
does not choose ownership. Each process opens its own Store; clones are useful
for threads inside that process. Repeated opens in the same process also work.

Only one process at a time obtains the nonblocking cleanup lock. Another process
skips cleanup immediately and continues its work. A live AssetLease holds a shared
lock on its actual immutable file, so a cleaner in any process skips that file.
Lease locks and maintenance ownership are released if their process exits or dies.
Recovery distinguishes another process's live imports from abandoned work using
locked session files, without PID or EXE-name assumptions. Never manually delete
lock/session files while a cache is open.

All applications sharing a root must use a compatible SDK cache protocol and the
same remote URL. App versions need not match. SDK 2 uses cache schema 3 and
refuses caches made by SDK 0.1.x (schema 1-2) without touching them, with
`InvalidArgument` "this cache was made by SDK 0.1 for storage v1". Give SDK 2 a
new, empty cache directory; SDK 0.1.x keeps using the old one against v1. Old
SDKs likewise reject a schema-3 cache. Stores with different projects may share
one root: their objects and leases are shared, each synchronizes only its own
project's queued writes.

For fully offline use, leave `options.remote = None` but still set
`options.project`: writes are queued for that project. Later close and reopen the
same root with a remote to upload the pending objects. Reopening with changed
credentials for the same URL is supported. A cache containing confirmed remote
objects is bound to that URL: use a new root to change servers. Changing a URL
on a pending-only cache requires closing all its users first. Changing the
contents of a server at the same URL cannot be detected by the client; preserve
server data and use a new root after replacing that server with an empty store.

The server's default is **TCP 7443**. `listen = "0.0.0.0:8443"` in `ulduar.toml`
changes it; restart the server, adjust firewall rules and use `:8443` in the SDK
URL. See [INSTALL.md](https://github.com/xacce/ulduar-releases/blob/main/INSTALL.md)
for server setup, `root_dir`, users, tokens, projects and Windows SCM/Linux systemd autostart.
The SDK itself is a library inside your process, with no client service to install.

## Threading, awaiting and polling

All disk access, hashing, SQLite writes, network requests, initialization and
cleanup run on SDK workers. Local operations have four workers, fetches have two,
and synchronization has a separate worker. A slow download does not consume the
local write workers. Fetch misses are serialized within a store so duplicate
requests share the first completed cached object across processes. Long local operations can still
queue other local operations. Filesystem/system calls and your encoder/reader
callbacks cannot be forcibly interrupted.

Every operation starts when submitted, and implements `Future`. Await it with
your normal executor; no Tokio runtime is required in the application. For a game
loop or GUI, keep `Operation<T>` and call `try_complete()` each frame:

```rust,no_run
# use ulduar_sdk::{Store, SyncOptions};
# fn frame_example(store: &Store) {
let mut operation = store.synchronize(SyncOptions::default());
// Keep operation in application state; on subsequent frames:
if let Some(progress) = operation.progress() {
    println!("{}: {}/{} objects", progress.phase,
             progress.objects_done, progress.objects_total);
}
if let Some(result) = operation.try_complete() {
    match result {
        Ok(stats) => println!("{} pending", stats.pending_objects),
        Err(error) => eprintln!("{error}"),
    }
}
# }
```

Call `try_complete` until it returns `Some`; after completion it returns `None`.
`progress()` and `stats()` copy memory-only snapshots, without directory walks,
SQLite queries or network traffic. Call `refresh_stats().await` or poll its Operation
to refresh shared totals changed by other processes, then use the updated memory
snapshot. This indexed query runs on a worker; no directory scan or payload read.
A progress snapshot may be temporarily `None`
if the worker is updating it. They are suitable for UI polling, but are not hard
real-time APIs: ordinary allocation, short registry locks and OS scheduling remain.
Submitting an operation allocates a small task and can initialize worker threads
on first use. Each work queue holds at most 256 waiting tasks; overload completes
with `Busy` instead of blocking the submitting thread. Bound your application's
number of in-flight operations and bytes too; a count limit is not a byte budget.

`Operation::wait()` explicitly blocks its caller and is intended for worker code,
CLI programs and tests. Do not call it on a UI/render thread. `cancel()` requests
cooperative cancellation; dropping an unfinished operation requests it too. Wait
for cancellation completion before assuming its worker has stopped. Network waits
check cancellation about every 25 ms, file loops between bounded chunks. A user
callback must return or write again to observe cancellation. Cancellation racing
successful publication may leave a complete object committed; partial objects are
never returned as successful writes/downloads.

## Write APIs

| Method | Input and ownership |
| --- | --- |
| `write_bytes(Vec<u8>, meta)` | Moves an owned buffer into a worker; no main-thread payload copy |
| `write_shared(Arc<[u8]>, meta)` | Shares immutable bytes without copying them on submission |
| `import_file(path, ImportMode::Copy, meta)` | Copies/hash-checks a file on a worker; leaves source |
| `import_file(path, ImportMode::Move, meta)` | Renames into staging on the same volume, then hashes; copies across volumes |
| `write_from(reader, meta)` | Takes a `Read + Send + 'static` reader and streams it on a worker |
| `write_with(meta, closure)` | Runs an encoder/serializer against a buffered `Write` sink on a worker |

`meta` is a `Meta { asset_type, name }`; `Meta::default()` is untyped and unnamed.
Writes need `StoreOptions::project`, otherwise they fail with `InvalidArgument`.
Writes return `Written { hash, size, source_removed }` after complete bytes and
metadata have been committed. Persist the returned hash, then read through SDK
APIs. An existing object deduplicates locally; writing it refreshes its TTL and
still queues one publication with this write's type and name. A failed
encoder or cancelled incomplete copy is not published. Files are immutable once
published. File paths passed through the ABI must be valid UTF-8.

```rust,no_run
# use ulduar_sdk::{Store, ImportMode, Meta};
# async fn inputs(store: &Store) -> Result<(), ulduar_sdk::Error> {
let copied = store.import_file("input.exr", ImportMode::Copy, Meta::default()).await?;
let moved = store.import_file("temporary.exr", ImportMode::Move, Meta::default()).await?;
println!("{}; source removed: {}", moved.hash, moved.source_removed);
let streamed = store.write_from(std::io::Cursor::new(vec![1, 2, 3]), Meta::default()).await?;
let encoded = store.write_with(Meta::default(), |output| {
    output.write_all(b"header")?;
    output.write_all(&[1, 2, 3])
}).await?;
# Ok(()) }
```

The caller grants exclusive access to an imported source until completion: do
not modify, replace or read it concurrently with Move. Same-volume Move avoids
a payload copy but still reads all bytes to compute BLAKE3. Across volumes, the
source is removed only after publication; `source_removed == false` means cleanup
of the original failed and the caller still owns it. On interrupted same-volume
Move, the SDK restores the source without overwriting a newly created file, or
retains the staged data for recovery. A crash may leave a recovered object whose
hash the caller did not receive; use Copy if the original path must survive all
failures. The cache is not a filename database.

## Projects, types and names

A server project is a named set of objects with per-user rights: read, write and
delete. A Store reads from and publishes to `StoreOptions::project`. Bytes are
deduplicated globally on the server, so an upload whose chunks already exist in
any project sends almost nothing; each project still sees only what was
published into it.

- **Type**: `AssetType(u32)`, user-defined, `AssetType::fourcc(b"MESH")` for
  four-character codes. `AssetType::NONE` (0) is untyped. The type is not part of
  the hash: the same bytes may be published under several types. Publishing
  typed bytes removes their untyped entry.
- **Name**: an optional UTF-8 string of 1-4096 bytes, one current name per object
  and project. A write with a name replaces the object's name on the next
  synchronization; a write without one keeps the name it has. Names have no
  history and no delete call; deleting the object drops its name.
- **Publication**: every write queues one publication (project, type, name).
  `synchronize` sends them in write order, so the last write of the same bytes
  decides the name.

Online calls run on the fetch workers, need a remote (`Offline` without one), and
see only what has been synchronized:

| Method | Result |
| --- | --- |
| `me()` | `Me { user, token, create_project }` for the token |
| `projects()` | `Vec<Project { name, read, write, delete }>` the token has any right in |
| `create_project(name)` | Needs the user's create-project permission; the creator gets read and write, never delete. `Conflict` if taken |
| `list(asset_type, after, limit)` | `Page { objects, next }` of hashes with this nonzero type, ascending; `limit` 1-10000 |
| `named(name, after, limit)` | `Page` of hashes currently carrying this name |
| `name(hash)` | `Option<String>`: the object's current name, `None` if it has none or the project lacks it |
| `delete(hash)` | Needs delete. Removes the object (all its types) from the project and drops its queued publications there; `false` if the project did not hold it. Local bytes stay until evicted |

Paginate by passing the previous page's `next` as `after` until it is `None`.
There is no listing of untyped objects and no listing of everything; keep your
own index of untyped hashes.

```rust,no_run
# use ulduar_sdk::{AssetType, Store};
# async fn meshes(store: &Store) -> Result<(), ulduar_sdk::Error> {
let mut after = None;
loop {
    let page = store.list(AssetType::fourcc(b"MESH"), after, 1000).await?;
    for hash in &page.objects {
        println!("{hash}: {:?}", store.name(hash.clone()).await?);
    }
    match page.next {
        Some(next) => after = Some(next),
        None => break,
    }
}
# Ok(()) }
```

## Read APIs and leases

`get(hash)` first looks in the local index. If found, it returns an `AssetLease`.
If absent, it downloads from the store's project, resumes a partial transfer if possible,
checks BLAKE3, commits it into the cache, then returns a lease. A missing remote
object, or a project the token cannot see, returns `NotFound`. Local hits do not
check projects: a root shared by stores of several projects shares its bytes. Without a remote, a local miss returns `Offline`;
with an unreachable remote it returns `Network`. Neither prevents other local
reads and writes.

`AssetLease::path()`, `hash()` and `size()` are memory-only. Keep the lease alive
for as long as any code uses its path. Cloning it shares the same pin. Cleanup
cannot evict pinned objects. Treat the path as read-only: never edit, rename or
delete files inside the cache. Use your own I/O worker if opening the path directly
would block your main thread. `get` checks size without rehashing the complete
local file on every hit; `read_bytes` and `export_file` additionally verify BLAKE3.

`read_bytes(hash)` returns the full bytes, so its memory use is proportional to
the asset size and crosses the native boundary with a copy. Prefer `get` and a
leased path or `export_file` for very large assets. `export_file(hash, destination)`
streams through a temporary file in the destination directory, verifies it,
flushes it and atomically replaces the destination. Failure/cancellation before
publication preserves the previous destination. The parent directory must exist;
export destinations inside the client cache are rejected.

Dropping a lease schedules worker-side release; it never writes SQLite on the
calling thread. For deterministic shutdown, await `lease.release()` after dropping
all its clones. Drop schedules `Store` cleanup as well; `store.close().await`
requires every Store clone, lease and operation to have been released/dropped.
Otherwise it returns `Busy`. An operation retained after `try_complete` still owns
its ticket: drop it before closing. Do not unload a host plugin or native library
while SDK operations/callbacks are active.

## Synchronization, cleanup and bandwidth

No upload or timer starts automatically. Call `synchronize(SyncOptions)` when your
application allows background work. It recovers interrupted staging, cleans expired
confirmed objects, uploads the FIFO pending set captured at the start of the
upload phase for this store's project, then cleans again. New objects after that cutoff wait for the next pass. Synchronization
is single-file and bounded in metadata pages, not a whole-store in-memory queue.
Across processes, cleanup uses a separate try-lock: an unsuccessful contender
skips cleanup and continues. Uploads use their own try-lock to avoid sending the
same shared pending queue concurrently; a contender returns success with phase
`upload_busy` and current pending totals. It can retry later. This does not block
local reads/writes. Downloads of cache misses are serialized across the shared
root to protect resumable staging; local hits continue on the local workers.
Each publication first asks the server to link known content (no payload); otherwise
the file is cut into content-defined chunks, the server reports which it lacks, and
only those are sent before the manifest commit. Failed/cancelled uploads remain
pending; already sent chunks are not sent again. One failing object stops that pass;
inspect the error and call again later. Successful completion with `pending_objects
== 0` means all currently recorded local writes are confirmed at the configured
server. `pending_objects` counts objects of the whole root that still have a queued
publication in any project; a store of one project cannot clear another's. Writes committed concurrently may produce a nonzero pending count.

`upload_bps` and `download_bps` are independent shared payload limits, in **bytes
per second**, per Store. Zero is unlimited. The selected values persist for that
open Store until the next synchronize call, including downloads triggered by later
reads. On opening a store both are unlimited. To limit downloads before any reads,
call synchronize first; an empty cache does not upload anything. Existing downloads
observe changed limits. Pacing uses bounded chunks with up to 64 KiB accumulated
burst; it does not account for HTTP/TLS overhead, TCP buffering or bandwidth used
by other processes. Retries can transfer bytes again. Limits reduce network load;
worker execution keeps work off your main thread, but it is not a CPU percentage
or frame-time scheduler. Reader/encoder callbacks should also avoid heavy shared
locks with the application thread.

With no remote configured, synchronization still recovers and cleans, then returns
success with phase `offline` and unchanged pending data. With a configured but
unreachable server, the pass returns a network error. Local data remains usable.
No directory watcher or background filesystem scan runs.

## Retention rules

- A completed write/download lives for 24 hours. Every successful user hit extends
  expiry to at least `now + 24h`, including release after a long-held user lease.
- Opening the Store gives all existing objects a global one-hour grace. The
  effective expiry is `max(object_expiry, startup_time + 1h)`, never an additive
  extension. It works after a week offline without updating every object row.
- An object may be evicted only after server confirmation, after its expiry and
  startup grace have elapsed, and with no live lease.
- Pending writes are never evicted by TTL. A fully local store with no server can
  therefore grow indefinitely; monitor pending size and available disk space.
- Cleanup runs only when synchronization is called and only while the application
  runs. Expiry does not imply a wall-clock deletion timer. Hot objects stay; cold
  confirmed objects disappear on a later maintenance pass.

## UI status and size

`Store::stats()` returns:

| Field | Meaning |
| --- | --- |
| `objects`, `local_bytes` | Committed local object count and logical payload bytes |
| `pending_objects`, `pending_bytes` | Subset awaiting remote confirmation |
| `active_leases` | Pins held by this Store; cloned public leases share one pin |
| `synchronizing` | Whether this Store currently has a maintenance pass running |

These totals exclude SQLite, WAL, directory allocation, staging files, partial
network transfers, source files and exported copies. Use filesystem free-space
metrics for disk capacity. Aggregates are maintained transactionally in the index
and published as snapshots; fields sampled during concurrent work are observational.
Object/byte totals describe the shared root as of this Store's last refresh.
Use `refresh_stats()` for a worker-side refresh after changes by another process,
for example periodically while displaying a UI panel. `active_leases` and
`synchronizing` describe this Store, not a count of every other process's handles.
The cleaner always checks OS locks from all processes regardless of those UI fields.

`Operation::progress()` reports phase (such as `recovering`, `cleaning`, `offline`,
`uploading`, `downloading`, `reading`, `writing`, `exporting`, `complete`, `failed`
or `cancelled`; initially empty), object totals/completed,
bytes uploaded/downloaded/processed, bytes total, and evicted objects/bytes.
Operation completion/error is authoritative: a last phase string alone does not
mean success. Upload/download byte counters count this call's payload transfer,
including retransmissions, not all historical bytes. Resume or an object already
on the server may complete with few or zero transferred bytes. `bytes_total` is
logical work, so do not assume `transferred / total` is a perfectly monotonic
percentage. For sync, use objects_done/objects_total plus pending bytes; show an
indeterminate phase while recovering/cleaning or while the total is unknown.

## Durability and failure handling

The private implementation uses a SQLite WAL index with FULL synchronization,
transactional aggregate counters, indexed dirty/expiry queries and a durable
staging journal. Files are flushed before successful publication. Restart
recovers completed publications and interrupted moved files, retains resumable
downloads, and discards incomplete uncommitted copies. Confirmed files missing
externally are removed from the index and can be refetched; a missing pending
file is reported as corruption and its dirty record is retained. Never repair a
cache by deleting its database: that loses which bytes have not reached the server.
Disk/controller guarantees still apply; abrupt power loss on the actual target
storage and physical disk-full behavior have not been validated locally.

Handle `Error { kind, message, http_status }`. Useful kinds are `Cancelled`,
`Offline`, `Busy`, `Conflict`, `NotFound`, `Unauthorized`, `Forbidden`, `HashMismatch`,
`ServerFull`, `Io`, `Network`, `InvalidArgument`, `Http`, `Native` and `Load`.
`Forbidden` means the token lacks the right in this project; `Conflict` is a taken
project name or an upload the server kept refusing.
`HashMismatch` also covers detected local corruption. Errors preserve underlying
TLS causes. Configure `Remote::attempts` above zero (default 3); retries apply to
transient network/server failures, not authentication errors or absent objects.
Correct the cause before retrying. Cancellation is cooperative and cannot roll
back an upload already accepted by the server.

For a private CA or self-signed **server** certificate, set
`remote.root_certificate = Some(pem_bytes)` before opening. Normal certificate
chain/hostname/expiry checks remain enabled; the URL hostname must match its SAN.
Keep tokens in your application's secret configuration, never in source or
logs. A bidirectional cache needs read and write in its project. There is no
certificate-validation bypass and no client signing key generated by the SDK.

## Scale and verification

Metadata operations use SQLite indexes, keyset pages of 128 entries and maintained
aggregates. Startup does not scan all object files; recovery work is proportional
to interrupted staging. Active leases and operations use memory proportional to
their count, not the full object inventory. The functional tests cover TTL/grace,
deduplication, concurrent reads/writes/cleanup, pins, move recovery, corruption,
cancellation, publication queues per project and refusal of SDK 0.1 caches. A real subprocess test runs four distinct EXEs
against one root, overlapping writes and duplicate publication, protecting foreign
leases, skipping a busy cleaner, surviving owner termination, recovering moved
files from a killed owner and preserving another process's active import.
An opt-in Windows/NTFS test seeds 100,000 real 8-byte
files, checks exact totals, evicts 49,999 confirmed expired files while preserving
50,000 pending files and a pin, then reopens the index. See
[SCALE.md](https://github.com/xacce/ulduar-releases/blob/main/SCALE.md) for measured
results and the separate server million-object test.

An independent consumer builds against the packaged public SDK, runs with only
its application executable, exercises offline input formats and restart after
process exit without destructors, projects, rights, types, names, listing and deletion, sync cancellation/resume, independent upload
and download pacing, refetch, offline operation with an unreachable server and
TLS trust/rejection. It also runs a host and four agent executables with different
names and SHA256 hashes against the same root and remote. Linux native binaries
are cross-built, not runtime-validated.

For source maintainers: build the workspace native libraries for the two targets
on the build machine, then run `python3 scripts/package-sdk.py` and
`python3 scripts/test-sdk.py` on the host that drives it through `nx`.
Native ABI 3 uses C-compatible buffers, opaque handles and matching allocation/free
functions; consumers should use the safe Rust facade rather than calling symbols
without the lifetime/threading contracts. A future incompatible ABI requires a new
binding/library pair. Server v1 data is not visible to SDK 2; copy it into a
project with the server's `migrate` command.
