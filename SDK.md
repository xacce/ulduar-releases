# Ulduar binary Rust SDK 0.1.3

Ulduar is an immutable asset store addressed by the 64-character lowercase BLAKE3
hash of each object's bytes. This SDK adds a durable local store to the remote
server: writes commit locally, reads resolve locally first, and your application
chooses when to upload and clean the cache. Keep your own filenames and asset-to-hash
mapping. Identical content shares one object.

## Install and deploy

Download `ulduar-v0.1.3-rust-sdk.zip` from the
[public release](https://github.com/xacce/ulduar-releases/releases/tag/v0.1.3),
verify its SHA256 against the release's `SHA256SUMS`, and extract `ulduar-sdk`
into your application's `vendor` directory:

```toml
[dependencies]
ulduar-sdk = { path = "vendor/ulduar-sdk" }
```

`ulduar-sdk-0.1.3.crate` contains the same bindings and native libraries in Cargo's
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
use ulduar_sdk::{Remote, Store, StoreOptions, SyncOptions};

async fn example() -> Result<(), ulduar_sdk::Error> {
    let mut options = StoreOptions::local("D:/MyApplication/assets");
    options.remote = Some(Remote::new(
        "https://assets.example.com:7443",
        std::env::var("ULDUAR_KEY").expect("set ULDUAR_KEY"),
    ));
    let store = Store::open(options).await?;
    let written = store.write_bytes(b"asset bytes".to_vec()).await?;
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
root contains `client.sqlite3`, its WAL files, `client.lock`, sharded `objects`
and staging `tmp`. It must be a dedicated writable local filesystem directory;
never reuse the server's `root_dir`. Network filesystems with unreliable locking,
SQLite WAL semantics or rename durability are unsupported.

One process owns each root. Opening it again, even from that same process,
returns `ErrorKind::Busy`. Clone the existing `Store` to share it between threads.
Separate applications/processes should have separate cache roots. The OS releases
the ownership lock after a crash. Back up a stopped store as a whole, including
metadata and pending objects. Do not copy only the SQLite main file while open.

For fully offline use, leave `options.remote = None`. Later close and reopen the
same root with a remote to upload the pending objects. Reopening with changed
credentials for the same URL is supported. A cache containing confirmed remote
objects is bound to that URL: use a new root to change servers. Changing the
contents of a server at the same URL cannot be detected by the client; preserve
server data and use a new root after replacing that server with an empty store.

The server's default is **TCP 7443**. `listen = "0.0.0.0:8443"` in `ulduar.toml`
changes it; restart the server, adjust firewall rules and use `:8443` in the SDK
URL. See [INSTALL.md](https://github.com/xacce/ulduar-releases/blob/main/INSTALL.md)
for server setup, `root_dir`, access keys and Windows SCM/Linux systemd autostart.
The SDK itself is a library inside your process, with no client service to install.

## Threading, awaiting and polling

All disk access, hashing, SQLite writes, network requests, initialization and
cleanup run on SDK workers. Local operations have four workers, fetches have two,
and synchronization has a separate worker. A slow download does not consume the
local write workers. Fetch misses are serialized within a store so duplicate
requests share the first completed cached object. Long local operations can still
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
SQLite queries or network traffic. A progress snapshot may be temporarily `None`
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
| `write_bytes(Vec<u8>)` | Moves an owned buffer into a worker; no main-thread payload copy |
| `write_shared(Arc<[u8]>)` | Shares immutable bytes without copying them on submission |
| `import_file(path, ImportMode::Copy)` | Copies/hash-checks a file on a worker; leaves source |
| `import_file(path, ImportMode::Move)` | Renames into staging on the same volume, then hashes; copies across volumes |
| `write_from(reader)` | Takes a `Read + Send + 'static` reader and streams it on a worker |
| `write_with(closure)` | Runs an encoder/serializer against a buffered `Write` sink on a worker |

Writes return `Written { hash, size, source_removed }` after complete bytes and
metadata have been committed. Persist the returned hash, then read through SDK
APIs. An existing object deduplicates; writing it refreshes its TTL. A failed
encoder or cancelled incomplete copy is not published. Files are immutable once
published. File paths passed through the ABI must be valid UTF-8.

```rust,no_run
# use ulduar_sdk::{Store, ImportMode};
# async fn inputs(store: &Store) -> Result<(), ulduar_sdk::Error> {
let copied = store.import_file("input.exr", ImportMode::Copy).await?;
let moved = store.import_file("temporary.exr", ImportMode::Move).await?;
println!("{}; source removed: {}", moved.hash, moved.source_removed);
let streamed = store.write_from(std::io::Cursor::new(vec![1, 2, 3])).await?;
let encoded = store.write_with(|output| {
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

## Read APIs and leases

`get(hash)` first looks in the local index. If found, it returns an `AssetLease`.
If absent, it downloads from the remote, resumes a partial transfer if possible,
checks BLAKE3, commits it into the cache, then returns a lease. A missing remote
object returns `NotFound`. Without a remote, a local miss returns `Offline`;
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
upload phase, then cleans again. New objects after that cutoff wait for the next pass. Synchronization
is single-file and bounded in metadata pages, not a whole-store in-memory queue.
Failed/cancelled uploads remain pending until a later call confirms server presence
or successfully completes a resumable upload. One failing object stops that pass;
inspect the error and call again later. Successful completion with `pending_objects
== 0` means all currently recorded local writes are confirmed at the configured
server. Writes committed concurrently may produce a nonzero pending count.

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
| `active_leases` | Native asset pins; cloned public leases share one pin |
| `synchronizing` | Whether this Store currently has a maintenance pass running |

These totals exclude SQLite, WAL, directory allocation, staging files, partial
network transfers, source files and exported copies. Use filesystem free-space
metrics for disk capacity. Aggregates are maintained transactionally in the index
and published as snapshots; fields sampled during concurrent work are observational.

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
`Offline`, `Busy`, `NotFound`, `Unauthorized`, `Forbidden`, `HashMismatch`,
`ServerFull`, `Io`, `Network`, `InvalidArgument`, `Http`, `Native` and `Load`.
`HashMismatch` also covers detected local corruption. Errors preserve underlying
TLS causes. Configure `Remote::attempts` above zero (default 3); retries apply to
transient network/server failures, not authentication errors or absent objects.
Correct the cause before retrying. Cancellation is cooperative and cannot roll
back an upload already accepted by the server.

For a private CA or self-signed **server** certificate, set
`remote.root_certificate = Some(pem_bytes)` before opening. Normal certificate
chain/hostname/expiry checks remain enabled; the URL hostname must match its SAN.
Keep access keys in your application's secret configuration, never in source or
logs. Keys need read and write access for a bidirectional cache. There is no
certificate-validation bypass and no client signing key generated by the SDK.

## Scale and verification

Metadata operations use SQLite indexes, keyset pages of 128 entries and maintained
aggregates. Startup does not scan all object files; recovery work is proportional
to interrupted staging. Active leases and operations use memory proportional to
their count, not the full object inventory. The functional tests cover TTL/grace,
deduplication, concurrent reads/writes/cleanup, pins, move recovery, corruption,
cancellation and ownership. An opt-in Windows/NTFS test seeds 100,000 real 8-byte
files, checks exact totals, evicts 49,999 confirmed expired files while preserving
50,000 pending files and a pin, then reopens the index. See
[SCALE.md](https://github.com/xacce/ulduar-releases/blob/main/SCALE.md) for measured
results and the separate server million-object test.

An independent consumer builds against the packaged public SDK, runs with only
its application executable, exercises offline input formats and restart after
process exit without destructors, sync cancellation/resume, independent upload
and download pacing, refetch, offline operation with an unreachable server and
TLS trust/rejection. Linux native binaries are cross-built, not runtime-validated.

For source maintainers: build the workspace native libraries for the two targets,
run `python scripts/package-sdk.py`, then `python scripts/test-sdk.py --sdk
 dist/ulduar-v0.1.3-rust-sdk.zip --server
 target/x86_64-pc-windows-msvc/release/ulduar.exe` (arguments on one command line).
Add `--cert target/sdk-test.crt --key target/sdk-test.key` for the local TLS case.
Native ABI 2 uses C-compatible buffers, opaque handles and matching allocation/free
functions; consumers should use the safe Rust facade rather than calling symbols
without the lifetime/threading contracts. A future incompatible ABI requires a new
binding/library pair. Existing server data/config need no migration for this SDK.
