# Object-count validation

This check targets a store containing one million distinct completed objects.
It does not certify a million simultaneous requests, nor does it measure the
time needed to upload a million assets through the API.

## Storage and request behavior

- Object paths are `objects/<aa>/<bb>/<64-character BLAKE3 hash>`. There are up to
  65,536 leaf buckets, averaging about 15 files per bucket at one million objects.
- GET, HEAD, and upload existence checks address a known hash directly. No full
  catalog is loaded or scanned for those requests.
- Startup prepares directories and scans partial uploads in `tmp/`. Its startup
  work does not traverse completed objects.
- `/missing` accepts up to 10,000 IDs and 1 MiB per request. The Rust client
  automatically chunks larger slices. Caller-owned input/result lists still use
  memory proportional to their length; callers can submit batches themselves.
- `verify` traverses directories incrementally and hashes one file at a time.
  The report retains damaged IDs, so memory can grow with the number of damaged
  objects. Healthy objects do not accumulate in an in-memory catalog.

## Measured result (2026-10-26)

Windows x64, NTFS, release build of v0.1.1 production code. The dataset contained
1,000,000 distinct real files of 8 bytes each, spread over all 65,536 leaf buckets.
The fixture was seeded directly on disk, so its creation time is not HTTP upload
throughput. No other builds or disk-heavy test jobs ran during the measured phases.

| Check | Result |
| --- | --- |
| Direct fixture creation, eight workers | 144.98 s |
| Server startup / TCP readiness | 0.507 s |
| Presence queries for all 1,000,000 IDs plus absent IDs | 45.16 s, correct |
| 1,000 verified HTTP downloads plus one new HTTP upload/download | 0.51 s, correct |
| Sampled server peak working set | 14.12 MiB |
| Restart and retrieve old/new objects | 0.513 s, correct |
| Full CLI integrity check, including the newly uploaded object | 1,000,001 checked, 0 damaged; 428.81 s |
| Sampled verifier peak working set | 5.72 MiB |

The complete object-count test passed. Memory was sampled periodically, not
instrumented for every allocation. The following directory-enumeration error
fix changes the error path only; the million-object measurements above were
taken before that fix. Its regression test and the normal functional suite are
checked separately.

## Integrity-check correction

The audit found that `verify` discarded errors yielded while advancing a
directory iterator. A scan could therefore finish successfully after skipping
an entry that the filesystem could not enumerate. This is distinct from a file
whose bytes fail their hash check.

The correction makes enumeration errors fail verification and includes the
directory path in the error. The regression fixture checks a healthy object,
then injects an enumeration error and requires failure instead of success.
Healthy-object storage, HTTP APIs, paths, and data formats remain unchanged.

## Reproduce

Run the release-mode integration test alone, with at least 8 GiB free for files
and filesystem metadata:

```sh
cargo test --release --locked --test many -- --ignored --nocapture
```

The test writes one million distinct 8-byte payloads into their correct hash
paths using eight fixture workers. It then starts a real server process, queries
every existing ID plus known absent IDs in batches crossing the client's
10,000-ID boundary, downloads 1,000 samples with client-side hash verification,
uploads and downloads a new object through HTTP, restarts the server, and runs
the actual `verify` CLI against the entire store. Server and verifier memory
must both be sampled successfully and remain below 256 MiB. Data is deleted at
the end. `ULDUAR_MANY_OBJECTS` can reduce the fixture size for diagnostics; a
smaller run does not replace the million-object check. Very short diagnostic
verifier processes may exit before a memory sample is available.

## Operational limits

An object-count check with tiny payloads primarily exercises filesystem metadata
and catalog handling. It does not replace testing the intended asset sizes,
network, TLS, concurrent users, and storage hardware. Results after fixture
creation use a warm filesystem cache. Process restarts do not simulate a power
loss or cold operating-system cache.

The current local environment is Windows with NTFS. Linux execution and native
service installation still require checking on a target host. The existing
10 GiB upload throughput gate is not repeatably passing; see the release notes.

Free space must cover payloads, filesystem metadata, partial uploads, logs, and
backups. Linux deployments must also have sufficient free inodes. There is no
automatic garbage collection for completed objects. Request concurrency consumes
memory and handles independently of the number of stored objects.

`tmp/` and `quarantine/` use flat directories. A million interrupted uploads or
damaged files is a separate workload from a million healthy completed objects;
startup/sweeps scan partial uploads and large damage reports retain their IDs.

## Offline client cache: 100,000 real files (v0.1.3)

The client cache has its own SQLite index and retention rules. An opt-in
release-mode test on Windows/NTFS seeded 100,000 distinct 8-byte files directly in
hash paths and inserted metadata in one transaction. Half were confirmed uploaded
and half pending. Setup took 64.650 seconds; this is not normal import throughput.

With the fake clock advanced eight days and one uploaded object pinned:

- 100,000 memory-only stats snapshots took 0.001 seconds (rounded; no timing gate).
- Indexed cleanup removed 49,999 expired confirmed objects in 13.471 seconds.
- All 50,000 pending objects and the pin survived, with exact totals.
- Reopening the remaining 50,001-object store took 0.014 seconds and recovered
  exact pending bytes (400,000). Startup uses a global one-hour grace rather than
  updating every object row.
- The complete test, including fixture removal, passed in 99.81 seconds.

Reproduce from the private source workspace:

```powershell
cargo test --release -p ulduar-client-store hundred_thousand -- --ignored --nocapture
```

This checks indexed retention/aggregates with real files, not 100,000 simultaneous
leases, writes, processes or interrupted staging records. Metadata pages select
128 rows; active jobs/leases consume additional memory. Stats do not query SQLite
or walk directories. Consumer tests separately exercise normal writes, process-exit
recovery, offline reads/writes, limited/cancelled resumable transfers and TLS.
The server million-object results above remain the earlier v0.1.1 baseline;
that full fixture was not repeated for this client feature.

## Shared client cache update (v0.1.4)

The same 100,000-file test was repeated with cross-process file leases and a
single-cleaner try-lock. All retention/count assertions passed: 50,000 pending
objects and one leased confirmed object survived 49,999 expired evictions.
Direct fixture setup took 69.957 s, 100,000 memory stats snapshots 0.001 s rounded,
cleanup 22.860 s, and reopening the remaining 50,001 objects 0.021 s. Total with
fixture removal was 114.88 s. These measurements replace the v0.1.3 cache timings
for current code; the earlier numbers above remain labeled as a baseline.

A separate real-process test opens one cache from a host and four differently
named executables with different binary contents. It verifies concurrent distinct
writes and shared-content deduplication, a busy cleaner being skipped, imports
remaining intact while another owner writes them, foreign leases preventing
eviction, cleanup ownership release after termination, and recovery of a moved
file from a killed owner. The packaged Rust SDK consumer additionally exercises
a host and four agent EXEs with distinct SHA256 hashes against a real server.
No EXE identity or PID determines access to the cache.
