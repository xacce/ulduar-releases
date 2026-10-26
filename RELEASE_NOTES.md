# Ulduar v0.1.3

This release adds a binary Rust SDK with a durable offline-first asset cache.
Applications use one hash-addressed API for local and remote data while choosing
when to upload and how much bandwidth to use.

## Binary SDK and offline storage

Download `ulduar-v0.1.3-rust-sdk.zip`, extract `ulduar-sdk` into your application's
`vendor` directory and add `ulduar-sdk = { path = "vendor/ulduar-sdk" }` to Cargo.
The `.crate` archive is also provided for manual extraction. Cargo builds the
public Rust facade; the private implementation ships as prebuilt Windows x64
MSVC and Linux x64 GNU libraries. The selected library embeds into your application
and extracts into a reusable per-user binary cache. See `SDK.md` for the full guide.

- Writes commit locally before returning a BLAKE3 hash; no automatic upload.
  Inputs include owned/shared bytes, copied/moved files, readers and encoders.
- Reads resolve locally first; misses download, resume and verify remote bytes
  before publication. Offline local reads/writes remain usable.
- Worker operations support await or polling, progress and cancellation.
  Local, download and synchronization work have separate queues.
- Consumer-started synchronization recovers staging, uploads pending objects and
  cleans expired confirmed objects. Independent upload/download payload limits
  use bytes per second; zero is unlimited. Memory-only stats provide sizes,
  pending counts, active leases and synchronization state.
- Retention is 24 hours after write/hit, with a non-additive one-hour startup grace.
  Unuploaded data never expires. Leases prevent eviction while paths are in use.
- SQLite WAL metadata, transactional totals and indexed pages bound inventory work.
  Interrupted publications and moved files recover on reopening. One process owns
  each root; clone Store for concurrent threads.

## Downloads and server upgrade

The release includes Windows x64 server ZIP, static Linux x64 musl server archive,
Rust SDK ZIP and `.crate`, English guides, example configuration, `BUILD_STATUS.md`
and `SHA256SUMS`. Verify hashes before installation. Binaries are unsigned; no
signing key/certificate was used.

The server HTTP API, stored objects and access keys remain compatible. Stop the
old service, replace its executable and restart, preserving configuration,
certificates and data. No server migration or service reinstallation is needed
when paths stay unchanged. The SDK needs a dedicated client cache root and must
never use the server data root.

**TCP 7443 remains the default.** Change `listen` to choose another port, restart,
and update firewall rules and SDK URLs. Set `root_dir` to choose server storage;
the legacy `data_dir` alias remains supported. See `INSTALL.md` for Windows SCM
and Linux systemd installation/autostart and storage migration. Choose the client
root with `StoreOptions::local(path)` in your application.

## Validation and limits

44 functional release tests and Clippy with warnings denied cover the existing
server/transport and new cache/ABI behavior. Cache tests cover TTL/grace, exact
stats, deduplication, concurrent read/write/cleanup, pins, cancelled copies,
move recovery preserving a recreated source, corruption and ownership rejection.
The opt-in 100,000-real-file Windows/NTFS test passed: 49,999 evictions in 13.471 s,
50,000 pending files plus a pin preserved, 50,001-object restart in 0.014 s.
See `SCALE.md` for methodology and the earlier server million-object audit.

An independent application compiles against the packaged SDK without private
crate dependencies and runs from its executable alone. HTTP and HTTPS tests
exercise all input formats, local-first reads, persistence after process exit
without destructors, cancelled/resumed uploads/downloads, payload rate limits,
concurrent local work, refetch, unreachable-server offline use and certificate
trust/rejection. Release assets are downloaded and checked before publication.

Windows x64 server/native client builds and Linux musl server/GNU client
cross-builds pass. Linux execution, real SCM/systemd installation, physical ENOSPC
and abrupt power-loss behavior remain unverified locally. Client caches require
a local filesystem, one owning process and enough space for unexpirable pending
objects. The client GNU build targets glibc 2.17+; it is not a musl application SDK.
`read_bytes` buffers/copies the full object; use leases or exports for large assets.

The existing 10 GiB upload target of 80% of plain disk write is not repeatably
passing (last representative result: 1.00 GiB/s vs 1.40 GiB/s, 72%, server 88 MiB).
This release does not retune that throughput or repeat the large benchmark.
Source stays private in `xacce/ulduar`; public releases contain binaries,
documentation and the thin SDK binding source. GitHub Actions remain disabled.
