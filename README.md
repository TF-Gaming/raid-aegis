# raid-aegis-daemon

Go daemon that calls the Rust `raid_collector` library via cgo, polls game memory on a
configurable interval, detects data changes via SHA-256, shows a live status window, and
POSTs snapshots to a remote server when the data changes.

## Prerequisites

- Windows only (cgo wiring is behind `//go:build windows`).
- Raid: Shadow Legends must be running (transient errors are tolerated and retried each tick).
- `raid_collector.dll` must be present beside the built binary.
- `native/libraid_collector.dll.a` must be present for linking (built from `raid-collector-rs`).
- `native/raid_collector.h` is bundled in this repo.
- `layouts/raid_v2026_07.json` must exist (bundled, or override via config).

## Build and run (Windows, Git Bash)

Build the Rust DLL from the [`raid-collector-rs`](https://github.com/your-org/raid-collector-rs) repo first:

```bash
cd raid-collector-rs
cargo build --release
cp target/release/raid_collector.dll      ../raid-aegis-daemon/native/
cp target/release/raid_collector.dll      ../raid-aegis-daemon/
cp target/release/libraid_collector.dll.a ../raid-aegis-daemon/native/libraid_collector.dll.a
```

Compile the exe icon resource (`windres`, bundled with the MinGW-w64 toolchain already
required for cgo) — without this, `go build` still succeeds but the exe has no icon, so
its Start Menu/Desktop shortcuts and taskbar entry show Windows' generic default:

```bash
windres -i resource_windows.rc -O coff -o rsrc_windows_amd64.syso
```

Then build and run the daemon (`go build` auto-links the `.syso` above — no special flag):

```bash
cd raid-aegis-daemon
go build -ldflags="-H windowsgui -X main.version=$(git describe --tags --dirty --always)" -o raid_daemon.exe .
./raid_daemon.exe
```

### Administrator rights (self-elevation)

Reading `Raid.exe` memory requires High integrity, because PlariumPlay launches the
game elevated. The daemon **elevates itself on startup**: when run without admin it
relaunches via a UAC prompt (`ShellExecute("runas")`) and the ordinary instance exits.
So you can start it from an ordinary terminal (or `make run`) — no elevated shell
needed; just accept the UAC prompt.

Set `AEGIS_NO_ELEVATE=1` to skip elevation (debugging without admin, or when the
daemon is already launched from an elevated shell).

## GUI

A Fyne v2 window titled **Raid Aegis** opens on startup:

```
┌─ Raid Aegis ──────────────────────────────┐
│ Account ID     121645029                   │
│ Status         Running                     │
│ Server         http://localhost:8080 ✓     │
│ Raid process   Last update 18/06/2026 …   │
│ Hero count     1242 champions              │
│ Artifact count 318 artifacts               │
│ Relic count    52 relics                   │
│ Last push      200 OK  07:33:00            │
│ [Live Updates ☑]           [Push Now]      │
├────────────────────────────────────────────┤
│ 07:33:00  CLS | TF  heroes=1242  changed   │
│ 07:33:10  no change                        │
└────────────────────────────────────────────┘
```

- **Live Updates** checkbox starts/stops the polling daemon.
- **Push Now** triggers an immediate push of the last captured snapshot.
- Closing the window minimises to the **system tray** — right-click to restore, toggle Live Updates, enable/disable auto-start, or quit.
- **Auto-start**: tray menu item writes/removes `HKCU\...\Run\RaidAegis` so the daemon launches automatically on Windows login (no elevation needed).

## Installer & auto-update

`installer/raid_aegis.iss` (Inno Setup 6) packages `raid_daemon.exe`, `raid_collector.dll`,
`layouts/*.json`, and `daemon.example.toml` into a single per-user installer targeting
`%LocalAppData%\RaidAegis` — no admin rights needed for the install step itself (the app
still self-elevates at runtime as above). Build it with:

```bash
cd raid-aegis-metadata
make installer   # requires Inno Setup 6; override ISCC= if not at the default path
```

Output: `raid-aegis-daemon/installer/output/RaidAegisSetup-<version>.exe`.

### Release automation — Conventional Commits → semantic-release → goreleaser

**Releases happen automatically on every push to `main` — there's no manual tagging
step.** `.github/workflows/release.yml`:

1. Runs [semantic-release](https://semantic-release.gitbook.io/) (`.releaserc.json`)
   against this repo's commit history since the last release. It only proceeds if
   there's at least one releasable [Conventional Commit](https://www.conventionalcommits.org/)
   (`feat:` → minor, `fix:` → patch, `BREAKING CHANGE:`/`!` → major; `chore:`/`docs:`/etc.
   never trigger a release). If so, it computes the next version, regenerates
   `CHANGELOG.md`, and commits + tags (`vX.Y.Z`) that back to `main` — **no GitHub
   Release object is created on this repo**, by design; it stays a plain private git
   history with no user-facing surface.
2. Only if a release happened: builds the Rust DLL (from the **pinned**
   `raid-collector-rs` commit — see below), the Go exe, and the Inno Setup
   installer, then publishes the installer + `checksums.txt` via
   [goreleaser](https://goreleaser.com/) (`.goreleaser.yaml`, publish-only — it
   doesn't build anything) to the separate, public, **source-free** repo
   [`TF-Gaming/raid-aegis`](https://github.com/TF-Gaming/raid-aegis), and copies
   this repo's `README.md`/`CHANGELOG.md` there too so public users have docs
   and release history without ever seeing this repo's source.

To build a manual local installer for testing without going through CI, `make installer`
(above) still works — it just uses whatever `main.version` your local `git describe`
resolves to, unrelated to the automated semantic-release versioning.

#### Pinning raid-collector-rs

CI checks out `raid-collector-rs` at the commit SHA recorded in
[`RAID_COLLECTOR_RS_REF`](./RAID_COLLECTOR_RS_REF) — **not** at whatever `main`
happens to be. This matters for two reasons:

- **Reproducibility.** Without a pin, a release build could silently pick up an
  in-progress/unrelated `raid-collector-rs` change that happened to land on its
  `main` between one run and the next.
- **It's the only thing that makes a collector-only change ship at all.**
  semantic-release only reads *this* repo's commit history — a `raid-collector-rs`
  push, by itself, never triggers anything here and is invisible to the version
  bump decision. Bumping the pin turns "a collector change happened" into a real,
  visible commit in `raid-aegis-daemon`, which is what both triggers a release
  *and* lets you describe what actually changed.

To pick up a `raid-collector-rs` change:

```bash
cd raid-aegis-metadata
make bump-collector-ref   # rewrites raid-aegis-daemon/RAID_COLLECTOR_RS_REF to collector-rs's current HEAD
cd ../raid-aegis-daemon
git add RAID_COLLECTOR_RS_REF
git commit -m "fix: bump raid-collector-rs for <describe the actual change>"
git push
```

Use `fix:`/`feat:` (whichever matches the underlying change) so semantic-release
picks it up — the commit *type* should describe what changed in `raid-collector-rs`,
not the mechanical fact that it's "just" a one-line ref bump. A `chore:`-typed bump
will not trigger a release.

Requires three repo secrets in `raid-aegis-daemon` (the default `GITHUB_TOKEN` handles
semantic-release's own commit-and-tag push back to this repo, given the workflow's
`permissions: contents: write`):

| Secret | Purpose |
|---|---|
| `RAID_COLLECTOR_RS_PAT` | Read access to `raid-collector-rs` for the cross-repo checkout |
| `RAID_AEGIS_RELEASES_PAT` | Write access to `TF-Gaming/raid-aegis` for goreleaser + the README/CHANGELOG sync |

The running app polls `GET /repos/TF-Gaming/raid-aegis/releases/latest` (`updater_windows.go`)
every 6 hours (plus once ~30s after startup), and via the tray's "Check for Updates" item.
On finding a newer clean-semver release it shows an "Update Now / Later" dialog; confirming
downloads the installer to `%TEMP%`, verifies it against the release's `checksums.txt`, then
runs it `/VERYSILENT` and quits — the installer's own `[Run]` entry relaunches the app.
Builds without a clean `vX.Y.Z` version (i.e. `dev` or a dirty `git describe`) never check —
see `parseSemver` — so local/dev builds never see update prompts. Override the poll target
with `AEGIS_UPDATE_CHECK_URL` for testing against a prerelease.

## Configuration

Copy `daemon.example.toml` to `daemon.toml` in the same directory as the binary, then edit:

```toml
poll_interval_secs = 10
layout_path        = "layouts/raid_v2026_07.json"
endpoint           = "http://localhost:8080/api/v1/snapshot"
api_key            = "your-secret"
```

Environment variables overlay the TOML file and take highest priority:

| Env var | TOML key | Default |
|---|---|---|
| `AEGIS_POLL_INTERVAL_SECS` | `poll_interval_secs` | `10` |
| `AEGIS_LAYOUT_PATH` | `layout_path` | `layouts/raid_v2026_07.json` |
| `AEGIS_ENDPOINT` | `endpoint` | `""` (push disabled) |
| `AEGIS_API_KEY` | `api_key` | `""` |
| `AEGIS_NO_ELEVATE` | _(none)_ | unset — set to `1` to skip UAC self-elevation |

When `endpoint` is empty, no HTTP push is attempted.

## HTTP push

On every tick where the SHA-256 of the snapshot changes, the daemon POSTs a `PushPayload`
to `endpoint`:

```json
{
  "schema_version": 1,
  "captured_at":    "2026-06-18T07:33:00Z",
  "account":        { "level": 96, "account_id": 121645029, "name": "CLS | TF", ... },
  "heroes":         [ { "id": 1, "name": "Galek", ... }, ... ],
  "artifacts":      [ { "id": 4321, "kind_id": 5, "rank": 6, "rarity_id": 5, ... }, ... ],
  "relics":         [ { "id": 1, "type_id": 3, "rank": 4, "level": 12, "hero_id": 42 }, ... ]
}
```

Retry policy: up to 3 attempts, backoff 1 s / 2 s / 4 s. No retry on 4xx. Retries on 5xx
or network error.

The `Authorization: Bearer <api_key>` header is sent when `api_key` is non-empty.

### Wiring the receiver server

Start the bundled server from the [`raid-aegis-server`](https://github.com/your-org/raid-aegis-server) repo:

```bash
cd raid-aegis-server
go build -o aegis_server.exe .
AEGIS_SERVER_API_KEY=your-secret ./aegis_server.exe
# Listening on :8080
```

Then set `endpoint` and `api_key` in `daemon.toml` to match.

## Running tests

```bash
go test -timeout 30s ./...
```

Tests use `mockCapturer` / `varyingCapturer` — no DLL or live Raid required.

| File | What it tests |
|---|---|
| `daemon_test.go` | Polling loop, change detection, JSON helpers |
| `push_test.go` | HTTP push retry logic via `httptest.NewServer` |
| `updater_test.go` | Semver parsing/comparison, release/asset selection, download+checksum verification via `httptest.NewServer` |

## Architecture

```
main()  (ffi_windows.go)
  LoadConfig()               config.go  — daemon.toml + AEGIS_* env vars
  runGUI(cfg)                gui_windows.go
    ├─ Fyne window (binding.String fields, Live Updates checkbox, Push Now button)
    ├─ startDaemon()
    │    └─ go RunDaemon(ctx, cfg, ffiCapturer{}, httpPusher{}, events)
    │              daemon.go
    │                doTick()
    │                  cap.CaptureAccount()    → ffiCapturer → rc_capture_account_json (Rust)
    │                  cap.CaptureHeroes()     → ffiCapturer → rc_capture_heroes_json (Rust)
    │                  cap.CaptureArtifacts()  → ffiCapturer → rc_capture_artifacts_json (Rust)
    │                  cap.CaptureRelics()       → ffiCapturer → rc_capture_relics_json (Rust)
    │                  cap.CaptureGreatHall()   → ffiCapturer → rc_capture_great_hall_json (Rust, best-effort)
    │                  cap.CaptureObservatory() → ffiCapturer → rc_capture_observatory_json (Rust, best-effort)
    │                  cap.CaptureInbox()       → ffiCapturer → rc_capture_inbox_json (Rust, best-effort)
    │                  snapshotHash()          SHA-256 change detection
    │                  pusher.Push(cfg, ev)    push.go (only when Changed & endpoint set)
    │                  DaemonEvent{} → events  non-blocking channel send
    └─ event consumer goroutine
         guiState.applyEvent(ev)   updates binding.String fields (goroutine-safe)
```

### Key interfaces

**`Capturer`** (daemon.go) — abstracts the Rust FFI for testability:

```go
type Capturer interface {
    CaptureAccount() (string, error)
    CaptureHeroes() (string, error)
    CaptureArtifacts() (string, error)
    CaptureRelics() (string, error)
    CaptureGreatHall() (string, error)   // best-effort; failure keeps previous value
    CaptureObservatory() (string, error) // best-effort; failure keeps previous value
    CaptureInbox() (string, error)       // best-effort; failure keeps previous value
}
```

**`Pusher`** (push.go) — abstracts HTTP for testability:

```go
type Pusher interface {
    Push(cfg DaemonConfig, ev DaemonEvent) PushResult
}
```

## Directory structure

```
raid-aegis-daemon/
├── daemon.go               Capturer/Pusher interfaces, DaemonConfig, DaemonEvent, RunDaemon()
├── daemon_test.go          Unit tests (mockCapturer — no DLL / Raid required)
├── config.go               LoadConfig() / LoadConfigFrom() — TOML + env var overlay
├── daemon.example.toml     Documented config template
├── ffi_windows.go          Windows cgo wiring + ffiCapturer + main()
├── gui_windows.go          Fyne v2 status window (Windows build tag)
├── autostart_windows.go    Registry auto-start helpers (Windows build tag)
├── autostart_stub.go       No-op stubs for non-Windows
├── elevate_windows.go      ensureElevated() — self-relaunch via UAC (Windows build tag)
├── push.go                 httpPusher — retry/backoff; Pusher interface
├── push_test.go            Push tests via httptest.NewServer
├── updater_windows.go      checkForUpdate/downloadInstaller/launchInstallerAndExit (Windows build tag)
├── updater_test.go         Updater tests via httptest.NewServer
├── version.go              var version — set via -ldflags at build time
├── assets.go               //go:embed assets/tf-gaming-logo.png
├── assets/
│   ├── tf-gaming-logo.png  App + systray icon (set at Fyne-runtime only)
│   └── raid_aegis.ico      Multi-res icon (16-256px) — compiled into the exe itself
├── resource_windows.rc     windres source — ICON resource pointing at assets/raid_aegis.ico
├── rsrc_windows_amd64.syso Compiled from the above (git-ignored) — go build auto-links it
├── layouts/
│   ├── raid_v2026_05.json  IL2CPP offset layout (older game version, bundled)
│   └── raid_v2026_07.json  IL2CPP offset layout (current default, bundled)
├── installer/
│   ├── raid_aegis.iss      Inno Setup script — see "Installer & auto-update" above
│   └── output/             make installer output (git-ignored)
├── .releaserc.json         semantic-release config — Conventional Commits → version + CHANGELOG.md
├── CHANGELOG.md            Generated by semantic-release (git-committed on each release)
├── RAID_COLLECTOR_RS_REF   Pinned raid-collector-rs commit SHA — see "Pinning raid-collector-rs" above
├── .goreleaser.yaml        Publish-only config — pushes releases to TF-Gaming/raid-aegis
├── .github/workflows/
│   └── release.yml         semantic-release + build + goreleaser publish, on every push to main
├── main.go                 Non-Windows stub
├── raid_daemon.exe         Built binary (git-ignored)
├── raid_collector.dll      Runtime DLL copy (git-ignored)
└── native/
    ├── raid_collector.h          C API header (bundled)
    ├── libraid_collector.dll.a   Import library (git-ignored — build from raid-collector-rs)
    ├── raid_collector.dll        DLL copy (git-ignored)
    └── README.md
```

## cgo wiring notes

- Header: `#cgo CFLAGS: -I${SRCDIR}/native` — `raid_collector.h` is bundled in `native/`
- Library: `#cgo LDFLAGS: -L${SRCDIR}/native -lraid_collector`
- Links against `libraid_collector.dll.a` (DLL import lib), **not** the static lib.
- Build requires MinGW-w64 GCC in PATH (use Git Bash with `~/.bashrc` setting MinGW path).
- See [`raid-collector-rs/docs/ffi_integration.md`](https://github.com/your-org/raid-collector-rs) for the full C API reference.
