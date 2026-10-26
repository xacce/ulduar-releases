# Ulduar v0.1.4

Multiple processes can now use one offline client cache. SFX Engine and its
background agents, separate EXEs and different application binary hashes can
open the same root and read/write concurrently. v0.1.3's exclusive cache owner
restriction has been removed.

## Shared cache behavior

- One process at a time acquires the cleanup try-lock. Other processes skip
  cleanup immediately and keep reading/writing. No owner service is required.
- Leases protect the actual immutable file across processes. The cleaner skips
  every file currently leased by any participating application. Process exit
  releases its locks automatically.
- SQLite transactions protect lookup/publication/TTL against deletion. Concurrent
  publication of identical bytes deduplicates without replacing leased objects.
- Recovery skips another process's live import. A crashed owner's staged move or
  completed publication can be recovered without relying on PID or EXE names.
- Upload and resumable-download ownership are separate from cleanup and local I/O.
  An upload contender returns current totals with phase upload_busy and can retry.
- New `refresh_stats()` queries shared indexed totals on a worker. `stats()` stays
  a memory-only snapshot; active_leases/synchronizing describe that Store instance.

## Upgrade

Download `ulduar-v0.1.4-rust-sdk.zip` or `ulduar-sdk-0.1.4.crate`. Use the public
Rust facade as a path dependency; the private implementation remains prebuilt
Windows x64 MSVC and Linux x64 GNU/glibc2.17+ libraries. Full English instructions
and the host/agent usage model are in `SDK.md`.

Stop processes using SDK v0.1.3 before the first open with v0.1.4. Cache schema 1
upgrades transactionally to schema 2, retaining objects, dirty state and recovery
journals. Update consumers to the new SDK. Old SDKs cannot join the shared cache:
a live old owner is rejected before migration, and the old SDK rejects schema 2.
Different application versions can share a root when their SDK cache protocol is
compatible; their executable name/hash does not matter. Do not downgrade by
editing the schema version. See the guide for stopped-cache backup/rollback.

TTL remains 24 hours after write/hit with one-hour startup grace. Unuploaded
objects never expire. Writes remain local first, misses fetch remote bytes, all
heavy work stays on workers, and the application starts synchronization. Bandwidth
limits are per Store, not a global cap across applications. Use the same remote
URL and a dedicated local filesystem root for every participant.

Server API, storage and configuration are unchanged. The server's default remains
TCP 7443; edit `listen` to change it and `root_dir` to choose its storage folder.
See `INSTALL.md` for Windows SCM/Linux systemd installation and autostart. Replacing
the server executable needs a service stop/start with existing config/data kept.

## Verification

47 functional release tests pass, including real subprocesses with four distinct
EXEs, cross-process lease protection, cleaner contention and process termination,
active import preservation, schema-1 pending/journal recovery and deduplication.
Workspace and packaged SDK Clippy pass with warnings denied. The independent
packaged SDK consumer passes over HTTP and HTTPS, including a host and four agent
EXEs with different SHA256 hashes, concurrent shared writes/reads/synchronization,
plus previous offline, resume, cancellation, rate-limit and TLS checks.

The 100,000-real-file Windows/NTFS cache test passed again: 49,999 evictions in
22.860 s, all 50,000 pending objects plus a pin retained, 50,001-object reopen in
0.021 s. See `SCALE.md` for exact scope and the earlier server million-object audit.
Windows server/native builds and Linux musl server/GNU native cross-builds pass.
All 12 release assets are downloaded and checked against SHA256 before publication.
Binaries are unsigned.

Linux execution, actual SCM/systemd installation, physical ENOSPC and abrupt
power loss remain unverified locally. Network filesystems are unsupported. Pending
objects require disk capacity indefinitely when no server can confirm them.
The earlier 10 GiB upload target of 80% of plain write is still not repeatably
passing (representative 1.00 vs 1.40 GiB/s, 72%, server 88 MiB); it was not retuned here.
Source remains private, thin SDK bindings are public, GitHub Actions stay disabled.
