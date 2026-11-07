# Ulduar Desktop

A portable desktop client for Ulduar storage v2. One executable, no installer: download
it, run it.

- `ulduar-desktop-v<version>-windows-x86_64.exe`: Windows 10 or newer, x64.
- `ulduar-desktop-v<version>-linux-x86_64.tar.gz`: x64 with glibc 2.28 or newer, plus X11
  or Wayland.

The interface is drawn on the CPU: no video driver, GPU or OpenGL is needed.

The executables are not Authenticode-signed, so Windows SmartScreen may warn on first
start. Compare the SHA256 with the release's `SHA256SUMS`.

## What it does

- Several accounts, on one or more servers. Add one with the server address and a v2
  token (`uld2_...`). A self-signed certificate is shown by its SHA-256 fingerprint and
  pinned after you confirm it. Plain `http://` works, with a warning.
- Create projects when your user may; the rights you hold show next to each project.
- Upload by dropping files or folders on the window, or with the Upload button. A folder
  keeps its relative paths in the names. An object has one name: uploading content already
  stored under another name keeps that name and says so.
- Download a selection into a folder of your choice. Downloads are verified by hash and
  resume after interruption. Server names are sanitized and cannot leave that folder. An
  existing file is never overwritten: the new one gets a ` (2)` suffix.
- Details of a file: full BLAKE3 hash, size, chunk count, types, date. Rename and delete
  appear only with the matching right.
- Search by parts of a name, or by four or more leading hex digits of a hash. Lists are
  loaded in pages as you scroll; 100 000 files stay smooth.
- Dark themes: Graphite, Midnight, Ember. English and Russian.

Keys: `Ctrl+F` search, `Ctrl+A` select all, `Del` delete, `F2` rename, `Ctrl+L` lock.

Requires a 2.1 or newer server: older ones have no file listing. A 2.0 server updates
itself when its service restarts.

## Password and PIN

Both are optional and set in Settings.

- **Password** encrypts the stored accounts and tokens (Argon2id, XChaCha20-Poly1305) and
  is asked at start. It cannot be recovered: losing it means adding the accounts again.
  Files on servers are not affected.
- **PIN** (4 to 12 digits) locks the running window, by hand or after idle minutes. It is
  not encryption: without a password the accounts file stays readable to anyone who can
  read your user's files. Wrong attempts add a growing delay. With a password set, five
  wrong PINs leave only the password. Transfers continue under the lock.

Without a password, tokens are stored in plain text, in a file only your user may read
(on Windows, this relies on your profile's permissions).

Data lives in `%APPDATA%\ulduar-desktop` on Windows and `~/.config/ulduar-desktop` on
Linux. Set `ULDUAR_DESKTOP_DIR` to keep it elsewhere, for example next to the executable
on a USB drive.

## Install a server

**Install server** sets up Ulduar on a fresh **Ubuntu 24** x86_64 machine over SSH. Other
systems are refused before anything changes.

Sign in with a login and password, a key file or pasted key with an optional passphrase,
or the login alone to use your SSH agent and `~/.ssh/id_ed25519`, `id_ecdsa`, `id_rsa`.
A login other than root needs sudo rights.

You confirm the host key fingerprint, then the installer:

1. Checks the system and installs `curl`, `ca-certificates`, `openssl` and `tar` if missing.
2. Creates the system user `ulduar`.
3. Downloads the server from the latest release. The signature of `SHA256SUMS` is checked
   on your computer and the archive hash on the server.
4. Writes `/etc/ulduar/ulduar.toml` and a self-signed certificate, and starts a systemd
   service that updates itself from signed releases.
5. Creates the Ulduar user and project, with read, write and delete rights, and a token.
6. Opens the port in `ufw` if it is active, checks the service, and adds the account here.

**SSH logins, passwords and keys are used once and never stored.** The app keeps only an
ordinary account: the server's address, the Ulduar token and the pinned certificate. A
cloud provider's own firewall is not touched: open the TCP port (7443 by default) there if
the app cannot connect at the end.

Running it again on the same machine keeps the configuration, certificate and data, and
adds a new token.
