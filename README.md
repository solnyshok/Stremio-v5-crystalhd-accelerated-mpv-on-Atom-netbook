# Custom Stremio v5 with external crystalhd/xv mpv on an Atom N455

## The problem

Stremio's desktop client renders video through an **embedded** libmpv instance. Embedded
libmpv means `vo=libmpv` — a render-to-texture path that only speaks OpenGL to the host's
surface. On a GMA3150 with mesa-amber that means **software GL (llvmpipe)**, which an Atom
N455 cannot sustain for video.

The only architecture that reaches the hardware's real capability (CrystalHD decode →
`vo=xv` X11 overlay) is a **standalone mpv process**. Stremio on Linux has no
"always use external player" option — that's what this project added.

Notably, this limitation is not specific to v5's Qt shell: the newer official
`stremio-linux-shell` (v6, gtk4/WebKitGTK) hardcodes `vo=libmpv` into a `gtk::GLArea`
and explicitly rejects attempts to set `vo`. Every embedded shell hits the same wall.

---

## Machine roles

| Machine | Role |
|---|---|
| ThinkPad L490 (i5-8250u, Ubuntu) | Build box — Docker container targeting Debian trixie |
| Mint box "server" (Celeron 1037U) | Hosts patched web frontend + Stremio streaming server |
| Netbook "atom" (Atom N455, 2GB, Debian 13 trixie, 1024x600) | Runs the shell + standalone mpv |

Offloading the frontend and streaming server to the Mint box keeps the netbook doing
almost nothing but running the thin Qt shell and mpv.

---

## Components and versions

- **stremio-web** `v5.0.0-beta.39`
- **stremio-core / stremio-core-web** `0.59.0`
- **stremio-shell** from `master` (needed for the `--webui-url` flag, absent from the stock .deb)
- **mpv 0.36** + **ffmpeg 4.4.4**, hand-built with `-Dxv=enabled` and crystalhd decoders
- **mesa-amber 21.x** via a `run-amber` wrapper (`/opt/mesa22`)

### Why the dependency pinning was necessary

stremio-shell embeds **QtWebEngine 5.15 = Chromium 87 (2020)**, which predates the WASM
reference-types proposal. Modern `wasm-bindgen` (0.2.100+) makes `externref` a *mandatory*
part of its JS↔Wasm glue, so current builds produce WASM that engine cannot instantiate
(`invalid value type 'externref'`).

Pinned in `stremio-core-web/Cargo.toml`:

```toml
wasm-bindgen = { version = "=0.2.92", optional = true }
wasm-bindgen-futures = { version = "=0.4.42", optional = true }
gloo-utils = { version = "0.2", features = ["serde"], optional = true }
```

`gloo-utils` had to come down too — its 0.3 line requires `js-sys ≥0.3.91`, which pins
`wasm-bindgen` to 0.2.114, still above the mandatory-externref cutoff.

---

## Building against Debian trixie

The build happens in a **`debian:trixie` Docker container** on the ThinkPad, not on the
host. Two independent reasons:

**glibc symbol versioning is forward-only.** A binary built against Ubuntu 24.04's glibc
will not start on trixie (`version 'GLIBC_2.XX' not found`). Building inside trixie
guarantees the shell binary links against exactly the glibc, Qt5, and libmpv the netbook has.

**CPU baseline.** The N455 is Bonnell — no SSE4, no AVX. The build host has both. Never
pass `-march=native`, `-mtune=native`, or `RUSTFLAGS="-C target-cpu=native"`; the default
generic x86_64 (SSE2) baseline is what's needed. The stremio repos set no `target-cpu` of
their own, so plain `qmake && make` and an unmodified cargo build are safe — the danger is
only in adding flags.

WASM output is architecture-independent, so `stremio-core-web` has no CPU concern at all.

### Trixie package gotchas

Upstream's `DEBIAN.md` list is **incomplete** — it names the QML *runtime* modules but omits
the qmake *development* packages, and each missing one only surfaces after the previous is
fixed:

| Missing package | Symptom |
|---|---|
| `qtwebengine5-dev` | `Project ERROR: Unknown module(s) in QT: webengine webchannel` |
| `libqt5webchannel5-dev` | same (note the `libqt5…` prefix — there is no `qtwebchannel5-dev` in Debian) |
| `libqt5opengl5-dev` | `CMake Error … Could not find … "Qt5OpenGL"` |
| `cmake` | `/bin/sh: 1: cmake: not found` — `release.makefile` shells out to cmake |
| `wget` | `release.makefile` uses it to fetch `server.js` |

Version floors that bite:

- **Node 22+ required.** stremio-web declares `engines: { node: ">=22", pnpm: ">=11" }`.
  Node 20 fails with `ERR_UNKNOWN_BUILTIN_MODULE: node:sqlite` — pnpm 11 uses a module that
  only exists in Node 22+.
- **Rust via rustup**, not trixie's system `rustc`, so the toolchain can be matched to the
  checked-out tag's `rust-toolchain.toml`.

### Version pairing

`stremio-web` and `stremio-core-web` are released independently and must be matched. Check
before building:

```bash
cd stremio-web && git tag --sort=-creatordate | head -1   # e.g. v5.0.0-beta.39
git checkout v5.0.0-beta.39
grep '"@stremio/stremio-core-web"' package.json           # -> 0.59.0
```

then check out `stremio-core-web-v0.59.0` in the `stremio-core` repo. (Guessing wrong here
means building a core whose WASM bindings don't match the frontend.)

Point the frontend at the local core build:

```json
"@stremio/stremio-core-web": "file:../stremio-core/stremio-core-web"
```

### Build order

```bash
# 1. core -> WASM   (after applying the deep_links patch and the Cargo.toml pins)
cd /work/stremio-core && rm -f Cargo.lock && cargo generate-lockfile
cd stremio-core-web && npm run build

# 2. frontend       (after applying the CONSTANTS.js and App.js patches)
cd /work/stremio-web
rm -rf node_modules/.pnpm/@stremio+stremio-core-web* node_modules/@stremio/stremio-core-web
pnpm install
md5sum node_modules/@stremio/stremio-core-web/stremio_core_web_bg.wasm \
       /work/stremio-core/stremio-core-web/stremio_core_web_bg.wasm   # MUST match
rm -rf node_modules/.cache && pnpm run build

# 3. shell          (after applying the main.qml patches)
cd /work/stremio-shell
touch main.qml qml.qrc          # QML is a Qt resource; without this the binary keeps old QML
make -j -C build
```

### Why v6 was rejected

The official `stremio-linux-shell` (gtk4/libadwaita/WebKitGTK, Rust) requires gtk **4.22**,
libadwaita **1.9**, and WebKitGTK **2.52**. Trixie ships **4.18 / 1.6 / 2.50**. Those aren't
build-script floors that can be pinned down — the `Cargo.toml` requests `features = ["v4_22"]`,
`["v1_9"]`, `["v2_52"]`, meaning the source calls APIs that don't exist in trixie's versions.
Downgrading would mean porting the app backward across four library generations.

The only trixie-era lineage (tags ≤ `v1.0.0-beta.13`) is a completely different pre-rewrite
codebase built on **CEF** — full Chromium, the heaviest option, and abandoned since the June
2026 rewrite. And v6 hardcodes `vo=libmpv` anyway, so it could never do `xv`.

---

## The four patches

### 1. `stremio-core` — generate a Linux mpv deeplink

`src/deep_links/mod.rs`, the `"mpv"` match arm:

```rust
"mpv" => Some(OpenPlayerLink {
    macos: Some(format!("mpv://{url}")),
    linux: Some(format!("mpv:{}", utf8_percent_encode(url.as_str(), URI_COMPONENT_ENCODE_SET))),
    ..Default::default()
}),
```

The URI **shape** matters and took three attempts:

| Form | Result |
|---|---|
| `mpv://https://host/...` | GIO parses `https//host` as a hostname → illegal chars → malformed, handler never invoked |
| `mpv:///percent-encoded` | Parsed as a mountable location → GVfs has no `mpv` backend → "Operation not supported" |
| `mpv:percent-encoded` | **Works** — opaque URI (like `mailto:`/`magnet:`), dispatched via scheme handler |

### 2. `stremio-web` — expose MPV in Settings on Linux

`src/common/CONSTANTS.js`:

```js
{ label: 'MPV', value: 'mpv', platforms: ['macos', 'linux'] },
```

### 3. `stremio-web` — stop the Chromecast crash

`src/App/App.js` line ~90 reads `chrome.cast.AutoJoinPolicy.PAGE_SCOPED`. QtWebEngine
isn't real Chrome, so Google's Cast SDK never loads and that throws an **uncaught**
TypeError during app mount. Wrapped `onChromecastStateChange`'s body in try/catch.

### 4. `stremio-shell` — actually launch the handler

Two changes in `main.qml`:

```qml
Process { id: externalPlayerProcess }
```

and inside `onNavigationRequested`, before the host-allowlist check:

```qml
var reqUrl = req.url.toString()
if (reqUrl.indexOf("mpv:") === 0 || reqUrl.indexOf("vlc:") === 0) {
    console.log("onNavigationRequested: external player URL " + reqUrl);
    externalPlayerProcess.start("/home/atom/mpv-url-handler.sh", [reqUrl], "");
    req.action = WebEngineView.IgnoreRequest;
    return;
}
```

**`Qt.openUrlExternally()` does not work here** even though `gio open` and `xdg-open`
handle the identical URL correctly — Qt's own dispatch refuses the custom scheme. The
shell's registered `Process` type (a `QProcess` subclass) bypasses it entirely. Its
QML-invokable `start()` takes **three** arguments — `(program, arguments, mPattern)` —
pass `""` for the third.

### 5. `stremio-shell` — suppress the harmless startup dialog

v5 dropped Qt webChannel support entirely (WebView2 only), so `initShellComm` never
exists and the shell's injected `try { initShellComm() }` always throws — popping a
"User Interface could not be loaded" dialog on every launch. In `main.qml`'s `injectJS()`
error branch, suppress *only* the `initShellComm` case (still show the dialog for genuine
load failures):

```qml
} else if (String(err).indexOf("initShellComm") !== -1) {
    console.log("injectJS: " + err)   // harmless; v5 has no initShellComm
} else {
    // ... original errorDialog code ...
}
```

### 6. Disable analytics / telemetry (two independent pipelines)

The **frontend** pipeline (`stremio-web/src/core/createTransport.ts`) — make `analytics()`
a no-op, which neutralizes all call sites (App.js, SharePrompt, StreamsList) at once:

```js
const analytics = (_event: object): Promise<void> => {
    return Promise.resolve();   // analytics disabled
};
```

The **Rust core** pipeline (`stremio-core-web/src/env.rs`) has its own, separate from the
frontend's. Its `analytics` feature can't simply be dropped — `env.rs` references the
`Analytics` type unconditionally, so removing the feature breaks the build
(`unresolved import stremio_core::analytics`). Instead, remove the single line that queues
events; `send_next_batch()`/`flush()` then have nothing to send:

```rust
// was: ANALYTICS.emit(name, data, &model.ctx, &model.streaming_server, path);
let _ = (name, data, model, path);   // analytics disabled
```

WATCH OUT: don't let a `sed` toggling the `analytics` feature land on the wrong line. The
feature belongs only on the `stremio-core` dependency in `stremio-core-web/Cargo.toml`
(`features = ["derive", "analytics"]`) — never on `serde`, which has no such feature and
fails with `serde does not have that feature`.

Both `api.strem.io` calls for genuine account functions (login, library sync, addon
collection) remain — those are required for the app to work at all.

---

## Runtime wiring

**Launcher** (`~/stremio5` on the netbook):

```bash
env DISPLAY=:0 \
  LD_LIBRARY_PATH=/usr/local/lib/x86_64-linux-gnu \
  QTWEBENGINE_DISABLE_GPU=1 \
  MESA_GL_VERSION_OVERRIDE=2.1 \
  XDG_DATA_HOME="$HOME/.stremio-custom-data/share" \
  XDG_CACHE_HOME="$HOME/.stremio-custom-data/cache" \
  XDG_CONFIG_HOME="$HOME/.stremio-custom-data/config" \
  run-amber ~/stremio-custom --development --webui-url=https://<HOST>:8174/#
```

The `XDG_*` overrides isolate the Chromium profile from the stock install (both hardcode
the same Qt org/app name, and concurrent access corrupts the shared LevelDB store).

**URI handler** (`~/mpv-url-handler.sh`), registered for `x-scheme-handler/mpv`:

```bash
#!/bin/bash
raw="$1"; raw="${raw#mpv:///}"; raw="${raw#mpv://}"; raw="${raw#mpv:}"
url=$(printf '%b' "${raw//%/\\x}")
pkill -x mpv 2>/dev/null && sleep 1          # CrystalHD is single-instance hardware
export XDG_CONFIG_HOME="/home/atom/.config"  # undo launcher's isolation so mpv.conf is found
setsid /home/atom/.local/bin/mpv-amber --config-dir=/home/atom/.config/mpv "$url" \
    >> /tmp/mpv-output.log 2>&1 &
exit 0                                        # background, don't exec — frees the QProcess slot
```

Three non-obvious requirements, each learned the hard way:

- **`setsid ... &` + `exit 0`, not `exec`** — a `QProcess` manages one child at a time; if
  the handler stays alive as that child, the second Play fails with
  `QProcess::start: Process is already running`.
- **`--config-dir` and resetting `XDG_CONFIG_HOME`** — the launcher's profile isolation is
  inherited by the child, hiding `~/.config/mpv/mpv.conf` and silently losing `vo=xv` and
  the crystalhd settings.
- **`pkill -x mpv`** — CrystalHD is single-instance; a lingering mpv silently blocks the
  next one. (`pkill -f mpv-amber` does *not* match: the wrapper `exec`s through
  `run-amber` into `mpv`, replacing the process image each time.)

**Services on the Mint box:**

| Port | Service | Exposed as | Via |
|---|---|---|---|
| 8173 | static frontend (`python3 -m http.server`) | 8174 | stunnel |
| 11470 | Stremio streaming server (`node server.js`) | 11471 | Caddy |

Both run as systemd units. The split matters: the frontend is static and same-origin, so
plain TLS forwarding suffices — but the streaming server needs **CORS headers injected**,
and stunnel is pure TCP/TLS with no notion of HTTP. Hence Caddy for that port only
(see attached `Caddyfile`).

Two constraints force this shape:

- **CORS** — the official client loads its frontend from `app.strem.io`, an origin the
  streaming server accommodates. A self-hosted frontend is a different origin, so every
  request needs explicit `Access-Control-Allow-Origin`. The preflight also needs
  `Access-Control-Allow-Headers: Content-Type` or playback fails on the `stats.json` call.
- **Mixed content** — the frontend is served over HTTPS, and Chromium auto-upgrades then
  blocks insecure subresources. An `http://` streaming-server URL is rejected regardless
  of CORS, which is why the server needs its own TLS endpoint rather than being reached
  directly on `:11470`.

**The duplicate-header trap (cost two broken attempts).** `server.js` sets its *own*
`Access-Control-Allow-Origin`. Caddy's `header_down Access-Control-Allow-Origin` **adds**
a second one, and a response carrying two ACAO headers is rejected by every browser. The
fix is Caddy's replace syntax — a leading `>` deletes any existing header before setting:

```
reverse_proxy 127.0.0.1:11470 {
    header_down >Access-Control-Allow-Origin "*"
    header_down >Access-Control-Allow-Headers "Content-Type, Range, Authorization"
}
```

With the duplicate eliminated, `"*"` works for **all** clients simultaneously (the netbook,
plus stock desktop/web/Android pointed at the same streaming-server URL). To find the real
Origin a client sends rather than guessing, add a temporary Caddy access `log` and read it —
the other desktop client here turned out to send `http://127.0.0.1:11470` (its own local
server's UI), not `app.strem.io`.

Caveat with multi-client use: all clients share one `server.js`, which carries the
single-stream patch below — so a second device starting a stream destroys the first's
engine. For genuinely simultaneous playback, change that patch to keep N engines.

---

## Server-side single-stream patch

Stremio's `server.js` never frees torrent engines. Its own idle handling only *pauses*
swarms, and the pause doesn't stick — observed **eight simultaneous engines**, four pulling
~15 MB/s aggregate, with two "finished" torrents at 20,000+ connection attempts each. On a
1037U that starves whatever you're actually watching (burst-then-stall playback).

The `/INFOHASH/destroy` endpoint doesn't work. But `server.js`, while a proprietary
prebuilt bundle with no public source, ships **unminified** (~102k readable lines), so it's
patchable. In `createEngine()` (~line 18114), before `EngineFS.beforeCreateEngine`:

```js
try {
    var _keep = String(infoHash).toLowerCase();
    Object.keys(engines).forEach(function(_h) {
        if (_h.toLowerCase() !== _keep) {
            console.log("single-stream patch: destroying engine " + _h);
            removeEngine(_h, function() {});
        }
    });
} catch (e) { console.log("single-stream patch error: " + e); }
```

It reuses their own `removeEngine()` (which destroys, emits, and deletes the key) and is
try/catch-wrapped so a failure can't break stream creation. Result: exactly one engine
alive at any time. **Reapply after any server.js update; keep `server.js.orig`.**

---

## Playback ceiling

`hwdec=crystalhd-copy` is the working mode — plain `hwdec=crystalhd` is rejected by this
build (`Unsupported hwdec: crystalhd`) and *silently falls back to software*, which is easy
to misread as success.

CrystalHD outputs **yuyv422**; the xv VO wants **uyvy422** (or yuv420p), so every frame goes
through a software colorspace conversion (`swscaler ... using C`). That conversion, not the
decode, is the binding constraint.

| Content | Pixels/sec | Result |
|---|---|---|
| 1920x800 @ 24 (letterboxed film) | 36.9M | smooth |
| 1280x720 @ 60 | 55.3M | smooth |
| 1920x1080 @ 24 | 49.8M | marginal — A-V drift, drops |
| 1920x1080 @ 60 | 124.4M | fails hard |

**The real limit is ~50M pixels/sec, not a resolution label.** That happens to coincide with
H.264 Level 4.1 / Blu-ray (1080p24), which is what CrystalHD was specified against — no
published pixels/sec figure exists to confirm whether the chip or the CPU conversion binds
first. Practically: letterboxed 24fps 1080p movies play fine; 60fps content doesn't.

A second, independent failure mode exists on some MP4s: pathological interleaving causing
thousands of demuxer seeks per second. Raising `demuxer-max-bytes` to 300M didn't help.

Since the panel is 1024x600, 1080p is downscaled below 720p's pixel count before display —
so there's no visual benefit to recover from pushing past this ceiling.

---

## Build gotchas worth remembering

- **pnpm caches `file:` dependencies.** After rebuilding the wasm you must
  `rm -rf node_modules/.pnpm/@stremio+stremio-core-web* node_modules/@stremio/stremio-core-web && pnpm install`,
  then verify with `md5sum` on both copies — otherwise you ship a stale binary. (Cost us
  three false "the fix didn't work" cycles.)
- **QML is embedded as a Qt resource.** After editing `main.qml`: `touch main.qml qml.qrc`
  before `make`, or the binary keeps the old QML.
- **The service worker precaches ~18MB.** After deploying a new web build, unregister it in
  DevTools or the netbook keeps serving the old bundle.
- **Remote debugging binds to localhost by default.** Use
  `QTWEBENGINE_REMOTE_DEBUGGING=0.0.0.0:9222` to inspect from another machine — essential,
  since DevTools on a 1024x600 screen is unusable. Turn it off afterward.
- **`pseudo-gui` mode sets `terminal=no`**, suppressing all mpv diagnostics. Debug with
  `--player-operation-mode=cplayer --terminal=yes -v`.
- **Deploying over a running binary fails** (ETXTBSY). `scp` to `.new` and `mv` into place.
- The whole build environment is snapshotted as the Docker image `stremio-v5-build:working`.
- **Clearing the service worker without DevTools:** it precaches ~18MB and will serve a
  stale bundle after deploy. On a headless/small screen, instead of DevTools →
  Unregister, just delete the isolated profile's cache from disk
  (`rm -rf ~/.stremio-custom-data/cache/*`), or wipe the whole profile
  (`rm -rf ~/.stremio-custom-data`, costs a re-login and re-selecting MPV).

---

## Menu entry and lingering gotchas

A `~/.local/share/applications/stremio5.desktop` launches it from the XFCE menu (its
`Icon=smartcode-stremio` came from the now-removed stock package — replace if it vanishes).

- **`playerType` syncs per Stremio account, not per device.** Logging into Stremio on any
  other machine can reset the netbook's External Player to disabled, after which Play uses
  the internal player. If mpv stops launching, check Settings → Player *first*. A separate
  account for the netbook fixes it permanently, at the cost of shared library/history.
- **`--single-process` is ignored by QtWebEngine** (unsupported); it silently keeps the
  separate renderer process. `--renderer-process-limit=1` is the honored equivalent. But
  with ~1GB of 2GB free during use, memory isn't the constraint here — the playback
  ceiling is — so this tuning changes little.
- **Deskflow** (keyboard/mouse sharing) running on the netbook hijacks mpv's input; quit it
  for full-screen playback control.

---

## Applying the patches

The six changes are provided as `git format-patch` files under `patches/`, grouped by
repo. They apply against the exact upstream tags used here:

| Repo | Tag / ref | Patches |
|---|---|---|
| stremio-core | `stremio-core-web-v0.59.0` | `0001`, `0002` |
| stremio-web | `v5.0.0-beta.39` | `0003`, `0004`, `0005` |
| stremio-shell | `master` (Qt5 shell) | `0006` |

For each repo, clone it, check out the matching ref, and apply its patches with
`git am` (which preserves the commit messages) or `git apply` (which just changes files):

```bash
# stremio-core
git clone https://github.com/Stremio/stremio-core.git
cd stremio-core && git checkout stremio-core-web-v0.59.0
git am /path/to/patches/stremio-core/*.patch
cd ..

# stremio-web
git clone https://github.com/Stremio/stremio-web.git
cd stremio-web && git checkout v5.0.0-beta.39
git am /path/to/patches/stremio-web/*.patch
cd ..

# stremio-shell
git clone --recurse-submodules https://github.com/Stremio/stremio-shell.git
cd stremio-shell        # master; the Qt5 shell
git am /path/to/patches/stremio-shell/*.patch
cd ..
```

If `git am` stops on a conflict (upstream moved since these tags), fall back to a
tolerant apply and resolve by hand:

```bash
git apply --3way --reject /path/to/patches/<repo>/*.patch
# then inspect any *.rej files
```

**Edit before building:** patch `0006` hardcodes the handler path
`/home/atom/mpv-url-handler.sh`. Change it to your own path in `main.qml` before compiling
the shell. Then build in the order and with the pinning described above.

### What is NOT a patch here

The `server.js` single-stream change is **not** included as a file — `server.js` is
Stremio's proprietary prebuilt streaming server, not open source, so a modified copy can't
be redistributed. The change is described in the "Server-side single-stream patch" section
above; apply it by hand to your own downloaded `server.js`.

---

## Publishing this to GitHub

This umbrella repo holds the writeup, the Docker/Caddy configs, and the patch set — it does
**not** vendor upstream source. Recommended layout:

1. **Create the umbrella repo** (this directory):

   ```bash
   cd stremio-v5-atom-mpv
   git init
   git add .
   git commit -m "Stremio v5 external crystalhd/xv mpv on Atom N455: writeup + patches"
   git branch -M main
   git remote add origin https://github.com/<you>/stremio-v5-atom-mpv.git
   git push -u origin main
   ```

2. **(Optional) fork the three upstream repos** on GitHub and push the patched branches to
   them, so people can build from a ready tree instead of applying patches:

   ```bash
   # after `git am` above, in each repo:
   git remote add fork https://github.com/<you>/stremio-core.git
   git push fork HEAD:atom-mpv         # a branch named atom-mpv
   ```

   Then link those branches from this README. Keep the upstream `LICENSE` files intact — all
   three repos are open source (GPL/MPL) and the patches inherit their license.

3. **Do not commit** your own `<HOST>`, LAN IPs, `stunnel.pem`, or account details. The
   configs here use `<HOST>` placeholders; keep them that way in the public repo and put
   real values only in a local, untracked copy.

### Suggested repo description / topics

> Make Stremio (v5) always hand playback to a standalone hardware-accelerated mpv
> (Broadcom CrystalHD + Xv) on a 2GB Atom N455 netbook running Debian trixie.

Topics: `stremio`, `mpv`, `crystalhd`, `atom-n455`, `debian`, `xv`, `low-end-hardware`,
`qtwebengine`, `external-player`.
