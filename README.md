# Stremio @ 42

*This project has been created as part of the 42 curriculum by mide-fre.*

---

## ⚠️ Disclaimer — Academic Use Only

**This project is NOT intended for recreational use.**

It exists **purely for academic and educational purposes**: to study how to compile a real-world Qt/MPV application from source, resolve shared-library dependencies by hand, and deploy software inside a restricted, root-less environment such as the 42 cluster.

It is **not** meant for streaming, media consumption, or any leisure activity on school machines. By using these scripts you acknowledge that you are doing so to learn about build systems, dynamic linking, and dependency management — and that you remain solely responsible for complying with 42's internal rules and with all applicable laws and content-licensing terms. The authors take no responsibility for any misuse.

---

## Table of Contents

1. [Description](#description)
2. [The Problem: Compiling Without Root](#the-problem-compiling-without-root)
3. [Key Comparisons](#key-comparisons)
4. [Design Choices](#design-choices)
5. [Quick Start](#quick-start)
   - [Quick one-liner — Install](#quick-one-liner--install)
   - [Quick one-liner — Remove](#quick-one-liner--remove)
   - [Manual usage](#manual-usage)
6. [How It Works](#how-it-works)
   - [Local Dependencies](#1-local-dependencies)
   - [Qt and WebEngine](#2-qt-and-webengine)
   - [Compiling the Shell](#3-compiling-the-shell)
   - [The Streaming Server](#4-the-streaming-server)
   - [The Launcher](#5-the-launcher)
7. [Installed Layout](#installed-layout)
8. [Troubleshooting Log](#troubleshooting-log)
9. [Command Glossary](#command-glossary)
10. [Requirements](#requirements)
11. [Resources](#resources)

---

## Description

This project builds the [Stremio](https://www.stremio.com/) desktop shell from source on a 42 school workstation, **without root access** and **without modifying the system**. Every dependency is downloaded as a `.deb`, extracted locally, and linked against from a personal directory.

42 workstations impose two hard constraints that this project is designed to study and work around:

- **No root / no `apt install`** — system packages cannot be installed, so all shared libraries (MPV, FFmpeg, codecs) must be fetched and extracted into a user-writable directory.
- **A tight home-directory quota** — anything large must live in `~/sgoinfre` instead of `~`, with only a lightweight symlink left in the home directory.

The deliverable is two scripts — `stremio-install` and `stremio-remove` — that automate the full build/deploy and a complete clean-up.

---

## The Problem: Compiling Without Root

A normal Stremio install assumes you can `apt install` system libraries and drop binaries into `/usr`. Neither is possible here. The build therefore has to:

1. Download every required library as a `.deb` from the school's package proxy.
2. Extract those `.deb` archives into `~/sgoinfre/libs_locais` with `dpkg -x` (no installation, no root).
3. Recreate the unversioned `.so` development symlinks that the linker needs.
4. Compile the Qt shell against those local libraries with explicit `-L`, `-rpath`, and `INCLUDEPATH` flags.
5. Inject `LD_LIBRARY_PATH` and `FFMPEG_PATH` at runtime so both the GUI and the streaming server find their libraries.

Each of these steps replaces something that root would normally do for you.

---

## Key Comparisons

### goinfre vs sgoinfre

Both are large, quota-friendly scratch directories on 42 machines, used here instead of the home directory.

| | goinfre | sgoinfre |
|---|---|---|
| **Scope** | Local to the machine | Shared across the cluster |
| **Speed** | Fast (local disk) | Slower (network-backed) |
| **Persistence** | Wiped frequently / on logout | More persistent |
| **Quota** | Generous | Generous |
| **Best for** | Temporary heavy builds | Installs you want to keep |

This project installs into **sgoinfre** so the build survives between sessions. The scripts still clean any stale `goinfre` install, since earlier versions targeted it.

### dpkg -x vs apt install

| | `dpkg -x` | `apt install` |
|---|---|---|
| **Root needed** | No | Yes |
| **Touches the system** | No — extracts to any folder | Yes — writes to `/usr`, `/etc` |
| **Dependency resolution** | Manual | Automatic |
| **Use here** | Extract libs into sgoinfre | Forbidden (no root) |

Because `apt install` is impossible, the installer resolves dependencies manually with `apt-cache depends --recurse` and extracts each `.deb` with `dpkg -x`.

### System Node vs nvm Node 18

| | System `/usr/bin/node` | nvm Node 18 |
|---|---|---|
| **Version on cluster** | v22+ | v18 LTS |
| **Compatibility with `server.js`** | Spawns but mishandles args | Stable |
| **Control** | None (fixed by admins) | Full (user-installed) |

The Stremio streaming server (`server.js`) is built for an older Node. A pinned Node 18, installed via `nvm`, avoids the argument-handling differences seen on the cluster's newer system Node.

---

## Design Choices

### Why install into sgoinfre?

The home directory has a strict quota that the Qt framework (~1.5 GB) and the extracted libraries would blow through instantly, producing `no space left on device` mid-build. `~/sgoinfre` has the space and persists between sessions, so the whole tree lives there and only a small symlink (`~/stremio`) is placed in the home directory.

### Why extract libraries locally instead of using system ones?

The cluster's system libraries are incomplete for a from-source Stremio build (no `libmpv-dev`, no matching codec set) and cannot be added without root. Downloading and extracting the exact versions into `libs_locais` gives a self-contained, reproducible dependency set that the build links against via `rpath`.

### Why disable the WebEngine sandbox?

QtWebEngine's Chromium sandbox relies on kernel features the cluster blocks, which crashes the renderer with `code 159`. Disabling the sandbox (`QTWEBENGINE_DISABLE_SANDBOX=1` plus `--no-sandbox`) is required for the UI to render in this environment.

### Why a `node` wrapper next to the binary?

`main.qml` launches the streaming server as `applicationDirPath + "/node" server.js` — it expects a `node` executable **next to the `stremio` binary**, not on `$PATH`. The installer places a wrapper there that exports `LD_LIBRARY_PATH` and `FFMPEG_PATH` and then delegates to the real Node 18.

---

## Quick Start

### Quick one-liner — Install

Paste this single line into your terminal. It clones the repo, runs the installer, and deletes the cloned folder afterwards:

```zsh
clear && git clone https://github.com/mdfpva/stremio_42.git && chmod +x stremio_42/stremio-install && ./stremio_42/stremio-install; rm -rf stremio_42
```

Then launch with:

```zsh
~/stremio
```

### Quick one-liner — Remove

Paste this single line to completely uninstall everything the installer created:

```zsh
clear && git clone https://github.com/mdfpva/stremio_42.git && chmod +x stremio_42/stremio-remove && ./stremio_42/stremio-remove; rm -rf stremio_42
```

This removes the build, the local libraries, the Qt install, the launcher, the streaming-server cache, the Node 18 version installed via `nvm`, and the `aqtinstall` tool — everything the installer created.

### Manual usage

If you cloned the repo and want to keep it around:

```zsh
git clone https://github.com/mdfpva/stremio_42.git
cd stremio_42
chmod +x stremio-install stremio-remove
./stremio-install     # build & deploy
./stremio-remove      # full uninstall
```

---

## How It Works

### 1. Local Dependencies

The installer downloads MPV, FFmpeg, and the full codec chain as `.deb` files into `~/sgoinfre/libs_locais`, resolving recursive dependencies with `apt-cache depends --recurse`. Each archive is unpacked with `dpkg -x`, then unversioned `.so` symlinks (e.g. `libmpv.so` -> `libmpv.so.1`) are recreated so the linker can find them.

### 2. Qt and WebEngine

Qt 5.15.2 with the QtWebEngine module is fetched into `~/sgoinfre/qt` using `aqtinstall` — no root, no system Qt required.

### 3. Compiling the Shell

The Stremio shell source is cloned and built with `qmake` + `make`, pointed at the local libraries through `INCLUDEPATH`, `-L`, `-lmpv -lluajit-5.1 -lmujs`, and an `-rpath` so the resulting binary knows where to find its `.so` files at runtime.

### 4. The Streaming Server

`server.js` is downloaded from the URL baked into `server-url.txt` by the build. A `node` wrapper placed next to the `stremio` binary exports `LD_LIBRARY_PATH`, `FFMPEG_PATH`, and `FFPROBE_PATH` before invoking Node 18, so the server can spawn FFmpeg for transcoding.

### 5. The Launcher

`~/sgoinfre/stremio` sets `LD_LIBRARY_PATH`, disables the WebEngine sandbox, and execs the shell with `--no-sandbox`. A symlink at `~/stremio` lets you start it from anywhere while keeping the home directory empty.

---

## Installed Layout

| Path | Contents |
|---|---|
| `~/sgoinfre/libs_locais` | Locally-extracted `.deb` libraries (MPV, FFmpeg, codecs) |
| `~/sgoinfre/qt` | Qt 5.15.2 + QtWebEngine (via `aqtinstall`) |
| `~/sgoinfre/stremio-shell` | Cloned source, compiled `stremio` binary, `server.js`, `node` wrapper |
| `~/sgoinfre/stremio` | Launcher script (sets env vars, then runs the shell) |
| `~/stremio` | Symlink to the launcher |
| `~/.stremio-server` | Streaming-server settings & cache (created at runtime) |
| `~/.nvm` -> Node 18 | Node runtime used by the streaming server |

---

## Troubleshooting Log

Problems encountered during the build, kept as documentation of *why* each workaround exists.

| Symptom | Cause | Fix |
|---|---|---|
| `cannot open shared object file: libXXX.so` | Library not extracted | Add the package to the download list / resolve recursively |
| `undefined reference to 'mpv_*'` | Linker can't find `libmpv.so` (only `.so.1` exists) | Recreate unversioned `.so` dev symlinks |
| `render process terminated with code 159` | WebEngine Chromium sandbox blocked by kernel | `QTWEBENGINE_DISABLE_SANDBOX=1` + `--no-sandbox` |
| `no space left on device: ~/stremio` | Home-directory quota exceeded | Put everything in sgoinfre, symlink in home |
| `server-crash 0 null` | Shell launches `applicationDirPath/node`, not `$PATH` | Drop a `node` wrapper next to the `stremio` binary |
| `ffmpeg: null` in server log | `find` matched a lintian override, not the binary | Search only under `*/bin/ffmpeg` |
| `find: 'dirname' terminated by signal 13` | SIGPIPE from `-exec dirname` per file | Use `find -printf '%h\n'` instead |

---

## Command Glossary

### Package Handling (root-less)

| Command | Explanation |
|---|---|
| `apt-get download <pkg>` | Downloads a `.deb` into the current directory **without installing** it. |
| `apt-cache depends --recurse <pkg>` | Lists a package's dependencies recursively, used to gather the full library set. |
| `dpkg -x file.deb .` | Extracts the contents of a `.deb` into a directory without installing (no root). |
| `ln -sf target link` | Creates/refreshes a symlink — used to make unversioned `.so` dev links. |

### Build Toolchain

| Command | Explanation |
|---|---|
| `qmake INCLUDEPATH+=... LIBS+=...` | Generates a Makefile, here pointed at local headers and libraries. |
| `make -j$(nproc)` | Compiles using one job per CPU core. |
| `-L<dir>` | Tells the linker an extra directory to search for libraries. |
| `-l<name>` | Links against `lib<name>.so` (e.g. `-lmpv` -> `libmpv.so`). |
| `-Wl,-rpath,<dir>` | Embeds a runtime library search path into the binary. |
| `-fuse-ld=gold` | Uses the gold linker, which handles this dependency set more reliably. |

### Runtime Environment

| Variable | Explanation |
|---|---|
| `LD_LIBRARY_PATH` | Extra directories the dynamic loader searches for `.so` files at runtime. |
| `FFMPEG_PATH` / `FFPROBE_PATH` | Absolute paths the streaming server uses to locate FFmpeg binaries. |
| `QTWEBENGINE_DISABLE_SANDBOX=1` | Disables the Chromium sandbox so the renderer can start on the cluster. |
| `QTWEBENGINE_CHROMIUM_FLAGS` | Extra Chromium flags (`--no-sandbox`, `--disable-gpu-sandbox`, ...). |

### Node / nvm

| Command | Explanation |
|---|---|
| `nvm install 18` | Installs Node 18 LTS into `~/.nvm` (user-level, no root). |
| `nvm which 18` | Prints the absolute path to the Node 18 binary. |
| `nvm uninstall 18` | Removes the Node 18 version (used by the uninstaller). |

### Diagnostics

| Command | Explanation |
|---|---|
| `ldd binary \| grep "not found"` | Lists shared libraries a binary needs but can't resolve. |
| `find <dir> -printf '%h\n'` | Prints each match's directory without spawning a process per file (avoids SIGPIPE). |
| `pkill -f "pattern"` | Terminates processes whose command line matches the pattern. |

---

## Requirements

- A 42-style Ubuntu workstation (tested on the 42 Porto cluster)
- `zsh`, `git`, `python3`, `pip`, `curl`, and a working `qmake`/`make` toolchain
- Network access to the school's package proxy and to GitHub
- Sufficient free space in `~/sgoinfre`

---

## Resources

- [Stremio Shell (GitHub)](https://github.com/Stremio/stremio-shell)
- [MPV / libmpv Documentation](https://mpv.io/manual/stable/)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [Qt 5.15 Documentation](https://doc.qt.io/qt-5.15/)
- [aqtinstall Documentation](https://aqtinstall.readthedocs.io/)
- [nvm (Node Version Manager)](https://github.com/nvm-sh/nvm)
- [dpkg Manual](https://manpages.debian.org/bookworm/dpkg/dpkg.1.en.html)
- [QtWebEngine Sandbox Notes](https://doc.qt.io/qt-5/qtwebengine-platform-notes.html)

### AI Usage

An AI assistant was used throughout this project for the following purposes:
- Diagnosing the cascade of missing shared-library errors and identifying the packages to download.
- Explaining the root cause of each failure (linker symlinks, WebEngine sandbox, home quota, the `applicationDirPath/node` launch path).
- Writing and iteratively debugging the `stremio-install` and `stremio-remove` scripts.
- Drafting this README, including the comparison tables, troubleshooting log, and command glossary.

All build decisions were verified and understood before being applied.

---

*For academic and educational purposes only. Not for recreational use.*
