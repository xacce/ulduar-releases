# Ulduar v2.0.0

Storage v2: users, projects, rights, chunk-level deduplication across projects,
asset types and names, and Rust SDK 2 built on them. The v1 storage, its API,
keys and SDK 0.1.x keep working unchanged next to it.

## Server: storage v2

- New storage under `root_dir/v2/` with its own API under `/v2`; see README
  "Storage v2 API". v1 objects stay where they are and are not visible in v2
  until copied with `ulduar migrate <project> [--type MESH] [ids...]`.
- Users with permanent tokens (`uld2_...`, many per user, stored as hashes).
- Projects with per-user read, write and delete rights. Delete is never granted by
  default. Users with the create-project permission can create projects through
  the API and get read and write in them.
- Content-defined chunks (FastCDC, 64 KiB / 256 KiB / 1 MiB) deduplicated across
  all projects, for storage and for transfer: clients send only chunks the server
  lacks, and known objects are linked without any payload.
- A user-defined u32 asset type per entry, not part of the hash. Listing is by
  nonzero type only, in pages ordered by hash.
- Optional object names in a separate database: an upload with a name replaces it,
  without one keeps it; lookup by name and by object. No history, no name deletion.
- `DELETE` of an object from a project (delete right). Chunks no object uses are
  removed after `upload_sweep_days`.
- Console: `user`, `token`, `project`, `grant`, `revoke`, `grants`, `migrate`;
  `verify` also re-hashes v2 chunks.

## Rust SDK 2.0.0

- Works with storage v2 only. `StoreOptions::project` selects the project;
  `Remote::new(url, token)` takes a v2 user token.
- Every write takes a `Meta` (asset type, optional name) and queues one
  publication; `synchronize` sends them in write order with chunked dedup uploads.
- New online calls: `me`, `projects`, `create_project`, `list`, `named`, `name`,
  `delete`; new `ErrorKind::Conflict`. `AssetType::fourcc(b"MESH")` helper.
- Cache schema 3. Caches made by SDK 0.1.x are refused untouched: give SDK 2 a new
  cache directory. Native ABI 3.

## Upgrade

Servers v0.1.5 update themselves at the next start; older ones need the new
executable installed once. Config, v1 keys and v1 data are kept; `v2/` is created
empty. Then create users, projects, grants and tokens (INSTALL.md "Storage v2").
Applications move to SDK 2 by adding a project, `Meta` arguments and a token, and
by pointing the store at a new cache directory.

## Verification

On Windows: server tests (library, v1 API, destructive, update, and v2 end to end:
console-managed users and tokens, rights including default-off delete, chunked
uploads deduplicated across projects, types, names, listing, deletion, chunk
collection and v1 migration), client cache tests (publication queues, refusal of
SDK 0.1 caches, recovery) and native ABI tests pass; Clippy passes with warnings
denied. An independent consumer built from the packaged SDK passed offline, crash
recovery, rate-limited and cancelled sync, download, four processes sharing one
cache, the v2 calls (projects, Conflict, listing, names, deletion, project
isolation) and offline use against a stopped server. The static Linux musl server
was smoke-run on Linux (users, tokens, v2 projects); the Linux GNU client library
(glibc 2.17) is cross-built and inspected, not run.
