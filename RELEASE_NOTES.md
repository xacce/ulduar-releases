# Ulduar v0.1.5

The server now updates itself. `serve` and the installed Windows/Linux service
check the latest public release at every start, install a newer signed version
over their executable and restart into it before accepting connections.

## Automatic updates

- Releases carry `SHA256SUMS.sig`, an Ed25519 signature of `SHA256SUMS` made with
  the Ulduar release key. The public key is built into the server; unsigned or
  wrongly signed releases are never installed.
- Only a higher version is installed. The archive must match its SHA256, and the
  extracted executable must run and report that exact version first. Any failure
  leaves the current executable untouched.
- No network or a slow GitHub logs `update check skipped: <reason>` and starts the
  current version (5 s connect timeout). The check runs only at startup.
- Linux re-executes the new file in the same process, so systemd keeps tracking it.
  The Windows service reports a stop and the SCM restart action starts the new
  file 2 s later. `serve` on Windows runs the new version as a child bound to its
  console and job: Ctrl+C reaches it, killing the parent stops both.
- `auto_update = false` disables the check; `update_url` selects a mirror folder
  holding `SHA256SUMS`, `SHA256SUMS.sig` and the archives.

## Upgrade

v0.1.4 and earlier have no updater: stop the service, replace the executable with
v0.1.5 and start it once. Later releases arrive at the next service start.
Config, keys, certificates and stored data are unchanged. The account running the
server needs write access to the executable's folder (service defaults have it).
The Rust SDK and offline cache are unchanged: this release contains no SDK
assets, keep using `ulduar-v0.1.4-rust-sdk.zip` / `ulduar-sdk-0.1.4.crate`
from the v0.1.4 release.

## Verification

Workspace tests pass on Windows, including local signed releases with foreign keys,
missing signatures, wrong hashes, mismatched or non-runnable executables (each
leaving the executable unchanged), downgrade/same-version refusal and startup with
an unreachable release server. End to end with the real release key: a v0.1.5
`serve` installed v0.1.6 from a local mirror and served it, on Windows (child and
job kill checked) and on Linux (static musl binary, same PID after exec, SIGTERM
stop). Clippy passes with warnings denied. Actual SCM-driven restart after an
update was not exercised on a registered service.
