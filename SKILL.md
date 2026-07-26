---
name: rustdesk-builder-v2
description: >
  Full working knowledge of the penangit/rustdesk-builder-v2 project — a self-hosted
  custom RustDesk client builder (GitHub Actions) that bakes a private server + config
  into Windows / Linux / Android / macOS builds. Read this to recover context, continue
  where a chat left off, or make further modifications. Covers the build mechanism, every
  non-obvious gotcha discovered the hard way, the fixes applied, the file inventory, and
  the current state / TODO.
---

# RustDesk Custom Client Builder — Project Knowledge Base

This is the "everything we learned" doc. The public setup guide is `README.md`; this file is
the deeper technical brain-dump for recovery and further modding.

## 1. What this project is

- **Repo:** `penangit/rustdesk-builder-v2` (PUBLIC) — a stripped-down fork of
  [`bryangerlach/rdgen`](https://github.com/bryangerlach/rdgen).
- **Goal:** build custom [RustDesk](https://github.com/rustdesk/rustdesk) clients that are
  pre-pointed at a self-hosted server (`ir.remote-neo.com`) with a baked-in permanent
  password + permissions, for Windows, Linux, Android, macOS — automatically via GitHub Actions.
- **Key difference from upstream rdgen:** the config is **committed in the repo** at
  `configs/RustDesk.json` (rdgen fetches it from a server). `scripts/load-config.py` reads it and
  emits `CUSTOM_*` env vars into `$GITHUB_ENV`.
- **Download page:** `docs/index.html`, hosted on Cloudflare Pages
  (`rustdesk-builder-v2.pages.dev`), driven live by the GitHub Releases API.
- **Building:** RustDesk v1.4.9; Rust 1.75; Flutter 3.24.5. Contributors: penangit, deadboy18.

## 2. Repo structure

```
penangit/rustdesk-builder-v2/
├── .github/workflows/
│   ├── build-windows.yml    # 64-bit Windows (Flutter) → .exe + .msi
│   ├── build-linux.yml      # full 11-variant Linux set (deb/rpm/AppImage/flatpak/Arch, x86_64+arm64)
│   └── build-android.yml    # per-arch APKs (aarch64/armv7/x86_64) + universal APK
├── configs/RustDesk.json    # THE config (server, key, appname, company, password, permissions)
├── scripts/load-config.py   # reads the JSON → CUSTOM_* env vars (incl. CUSTOM_TXT + CUSTOM_B64)
├── docs/index.html          # download page (themes, OS icons, tooltips, device detection, changelog)
└── README.md                # public A-Z setup guide
```

> The delivered workflow files may be named `build-linux-full.yml` / `build-windows.yml` in the
> working folder; in the repo they live as `build-linux.yml` / `build-windows.yml` etc.

## 3. How customizations are applied (THE core mental model)

Customizations land in **two different ways**, and knowing which is which explains every bug:

**(A) Compiled into the binary** — applied by `sed`-patching the RustDesk *source* before build:
- Server address + public key → `libs/hbb_common/src/config.rs` (`rs-ny.rustdesk.com`, the default key string).
- API server → `src/common.rs` (`https://admin.rustdesk.com`).
- App name / company / URLs → various source files.
These are in the compiled `.so` / `.exe`, so they work on **every** platform and format. This is why
"server/key/company worked" even when the password didn't.

**(B) Read at runtime from a config file (`custom_.txt`)** — the permanent **password**, permission
defaults, and approval mode. On desktop RustDesk reads `custom_.txt` from the executable's own
directory at startup and calls `read_custom_client()`.

### The signature-strip patch (`allowCustom.py` / inline seds)
Stock RustDesk only accepts a *signed* custom config. `allowCustom.py` (Linux, wget'd at build) and
the equivalent inline seds (Android/Windows) remove a **9-line block** in `src/common.rs`:
the `const KEY: &str = "5Qbwsde3..."` line + `get_rs_pk(...)` + the `sign::verify(...)` block.
It also renames every `custom.txt` → `custom_.txt`.

⚠️ **It does NOT remove the `decode64(config)` line** at the very start of `read_custom_client()`
(`src/common.rs`, `decode64` defined ~line 1799 = strict `base64::decode`). **So the config the
client reads must still be base64.** ← this caused the biggest bug (see §4).

## 4. GOTCHAS / hard-won lessons (read this before touching anything)

### 4.1 ★ custom_.txt must be BASE64, not raw JSON (password bug)
`load-config.py` emits **both** `CUSTOM_TXT` (raw JSON) and `CUSTOM_B64` (base64 of that JSON).
`read_custom_client()` starts with `decode64()` (base64). If you write raw JSON to `custom_.txt`,
decode64 fails on line 1 → the function returns → **password + permissions + approve-mode are applied
on NO platform**. Server/key still work (those are category A).
**FIX: every workflow must write `CUSTOM_B64` to `custom_.txt`.** (Symptom: user had to type the
permanent password manually; same bug exists in upstream rdgen.)

### 4.2 ★ Android never file-reads custom_.txt
On Android, `custom_.txt` is a bundled Flutter **asset**, not a file at `current_exe().parent()`, so
the desktop file-read finds nothing. The config must be **handed to native code**:
- `src/flutter_ffi.rs` → `initialize(app_dir, custom_client_config)`: if the string is empty it calls
  `load_custom_client()` (file read, fails on Android); else `read_custom_client(config)`.
- Two callers passed `""`: **`flutter/lib/models/native_model.dart`** (`customClientConfig: ''`, the
  FFI/UI init) and **`flutter/android/.../MainService.kt`** (`FFI.startServer(configPath, "")`, the
  server process that authenticates incoming connections).
**FIX (build-android.yml "Embed custom config for Android" step):** sed the base64 config into both —
`FFI.startServer(configPath, "<CUSTOM_B64>")` and `customClientConfig: '<CUSTOM_B64>',`. base64 is
quote-safe. Then `read_custom_client` runs in-process and presets the password.
- Preset mechanism: `read_custom_client` puts the top-level `password` key into `HARD_SETTINGS`;
  `Config::is_using_preset_password()` (mobile) / `is_permanent_password_preset()` (desktop) read it;
  FFI `is_preset_password_mobile_only()` exists. (`hbb_common` is a git submodule — NOT in the source
  zip — so `config.rs` internals can't be inspected locally.)

### 4.3 Filename → download-page slot convention (keep it!)
The page matches assets to slots by filename regex. Tokens: 64-bit = `x86_64`; 32-bit Windows = `i686`
(substring-safe vs `x86_64`, do NOT use `x86`); arm64 = `aarch64`; arm32 = `armv7`; openSUSE rpm carries
`suse`; universal Android = `universal`. Versioned names: `RustDesk-<VERSION>-<arch>.<ext>`.
Fedora rpm regex must exclude `suse`; SUSE regex must require `suse`.

### 4.4 Linux: custom_.txt must reach the bundle, not just the .deb
`build.py` stages `custom_.txt` into the .deb dir (`tmpdeb`). deb/AppImage/flatpak extract the deb so
they get it, but **rpm(Fedora)/rpm(SUSE)/Arch package the flutter BUNDLE** → they'd ship without it.
FIX: copy `custom_.txt` into `flutter/build/linux/{x64|arm64}/release/bundle/` after build.py. (Same
gap exists in upstream rdgen.)

### 4.5 Linux arm64 needs specific runners/actions
`ubuntu-22.04-arm` runners (FREE only on PUBLIC repos), `rustdesk-org/run-on-arch-action@amd64-support`
(old-glibc ubuntu18.04 container), and `flutter-elinux` for arm64. Full Linux set = deb + rpm(Fedora) +
rpm(SUSE) + AppImage + flatpak, each ×{x86_64, aarch64}, plus Arch `pkg.tar.zst` (x86_64) = **11 artifacts**.

### 4.6 Changelog extraction (release notes AND page)
RustDesk's upstream release body starts with a **download table linking to rustdesk's OWN binaries**,
a scam-warning banner, and promo, then a `<details><summary>Changelog</summary>…</details>` block.
Both the release-notes step (Python in the workflow) and the page (JS) must extract **only** the details
block, or you'll show rustdesk's download links / scam banner. If a release still shows the upstream
table, it was generated by an OLD workflow — re-run with the current one, or edit that release's notes once.

### 4.7 Play Protect warning is inherent
Sideloaded remote-access APKs signed outside the Play Store always trip Play Protect (Google flags
remote-desktop apps as potentially-harmful). Not fixable from the build; users tap "install anyway".

### 4.8 Universal vs per-arch APK
Per-arch APKs (`--split-per-abi`) are smaller; a **universal** APK (no split, all ABIs) works on any
device with no arch guessing. Implemented as a 4th matrix leg `{ arch: universal, universal: true }` that
branches three per-target steps (native deps for all 3 ABIs, build all 3 rust libs, `flutter build apk`
with no `--split-per-abi`) → `RustDesk-<VERSION>-universal.apk`. The `ndk_*.sh` scripts each hardcode
their target (`cargo ndk --target ... build`), so running all three builds all three.

### 4.9 Android signing
Signed via `r0adkll/sign-android-release@v1` (`BUILD_TOOLS_VERSION 34.0.0`) using repo secrets
`ANDROID_SIGNING_KEY` (base64 .jks), `ANDROID_ALIAS`, `ANDROID_KEY_STORE_PASSWORD`, `ANDROID_KEY_PASSWORD`.
Gradle build uses the debug signing slot (`sed signingConfigs.release→debug`), then r0adkll re-signs with
the release keystore; the finalize step falls back to the gradle-signed apk if signing is skipped.
`androidappid` change is guarded against empty (user's is empty → keep `com.carriez.flutter_hbb` default).
Also removes the Android scam-warning prompt (`server_page.dart` `show-scam-warning` → `"N"`).

## 5. The workflows (what each does)

- **build-windows.yml** — 64-bit Windows (Flutter). Outputs `RustDesk-<VER>-x86_64.exe` (portable) +
  `.msi` (installer). `run-name` includes the version. Writes `CUSTOM_B64` to `custom_.txt`.
- **build-linux.yml** — full 11-variant set. Bundle custom_.txt fix; upstream-changelog extraction in
  release notes; `CUSTOM_B64` to `custom_.txt`. arm64 legs on `ubuntu-22.04-arm`.
- **build-android.yml** — matrix aarch64/armv7/x86_64 **+ universal** on ubuntu-24.04. Signs; embeds the
  base64 config into Kotlin + Dart (§4.2); removes scam warning; extracts changelog; versioned filenames.
- Common jobs: `detect-version` (resolve latest or manual), `generate-bridge` (flutter_rust_bridge codegen,
  uploaded as artifact), the build matrix, `create-release` (globs all `*.apk`/artifacts → release),
  `update-pages` (appends to docs/versions.txt — NOTE the page does NOT read versions.txt).

## 6. The download page (docs/index.html)

Single self-contained HTML file. Features:
- **Dark + light theme** toggle (localStorage `rd-theme`, default from `prefers-color-scheme`).
- **Custom theme-adaptive SVG OS icons** (windows/linux/android/apple/package) — hand-drawn, not exact
  brand logos.
- **Hover tooltips** on every variant explaining which to use (CSS `data-tip`).
- **Device detection** (`detectOS()` via userAgent + `userAgentData.platform`): highlights the recommended
  download with a "Recommended" badge + accent border, dims non-matching platform groups (still clickable),
  shows a device banner. Android → recommends the **Universal** APK (first item = primary). iOS → App Store note.
- **Build-in-progress banner**: polls the Actions runs API (every 2 min), shows platform(s) + RustDesk
  version (from `display_title`, hence the `run-name` in workflows) + live elapsed time (ticks every 15s).
- **Changelog per release**: lazy-loaded from rustdesk/rustdesk on expand, extracted (§4.6), minimal MD→HTML.
- Releases refresh every 5 min. Slots: Win 4 + Android 4 (incl. universal) + Linux 11 + macOS 1 = **20**.
- Config const at top of `<script>`: `REPO = 'penangit/rustdesk-builder-v2'`.
- Mind the GitHub unauthenticated API limit (~60 req/hr) with the polling.

## 7. Config reference (configs/RustDesk.json)

Get it from **<https://rdgen.crayoneater.org/>** (fill the form → download → save as `configs/RustDesk.json`).
Fields: `serverIP`, `key` (server PUBLIC key), `apiServer`, `appname` (`RustDesk` = stock / no rename),
`compname`, `permanentPassword`, `passApproveMode` (e.g. `password-click` = password AND on-screen accept),
`delayFix`, `hidecm`, `xOffline`, `removeNewVersionNotif`, the `enable*` permission flags, `androidappid`
(blank = default). `load-config.py`'s `b()` normalizes "on"→"true".

## 8. Security notes

- The **only genuinely sensitive value** in the committed config is `permanentPassword`. The server address
  and public **key are NOT secret** — RustDesk embeds the key in every distributed client by design.
- `password-click` approval means a leaked password alone can't connect (needs an on-screen accept).
- **Do NOT make the repo private** to "hide" things: it breaks the public download page (private-repo releases
  + asset URLs 404 for anonymous users) AND arm64 runners start billing minutes. If worried: drop
  `permanentPassword` from the committed config, or keep a private build repo + publish releases to a separate
  public repo / object storage.

## 9. How to make further mods (pointers)

- **Add a platform/variant:** add the build to the right workflow with a filename following §4.3, then add a
  matching slot to `GROUPS` in `docs/index.html` (with a `match` regex + tooltip). Update `TOTAL_SLOTS` is automatic.
- **Change the server/config:** edit `configs/RustDesk.json` (or regenerate from rdgen.crayoneater.org) and re-run.
- **Windows 32-bit (Sciter):** still open/deferred.
- **Anything touching the password/permissions:** remember §4.1 (base64) and §4.2 (Android embed).
- **Verifying a build:** check About (version/company = category A), then test unattended login with the baked
  password (category B — the thing that was broken). Delete old non-versioned assets from a release so the page
  picks the right file.

## 10. Current state / TODO (as of last session)

**Done:**
- Password fix applied to all 3 workflows: `custom_.txt` now uses `CUSTOM_B64`; Android additionally embeds the
  base64 config into `MainService.kt` + `native_model.dart`. → user must **redeploy all 3 workflows + re-run**
  to get the baked password actually working (affects Windows/Linux/Android).
- Full Linux 11-variant parity built successfully (with bundle custom_.txt fix).
- Windows x64 + Android per-arch built & signed successfully.
- Download page redesign: themes, OS icons, tooltips, device detection, build banner (version + elapsed),
  changelog extraction, new footer.
- **Universal Android APK** added (4th matrix leg) + Universal slot on the page (recommended for Android).
- README.md (public A-Z guide) written.

**Open / next:**
- Re-run all three with the fixed workflows and verify the permanent password now auto-applies (incl. desktop).
- Delete stale non-versioned assets from the existing v1.4.9 release (or delete the release + re-run) so the
  page maps to the right files.
- Linux filenames are NOT yet versioned (Windows + Android are) — offered as a consistency follow-up.
- Windows 32-bit (Sciter) workflow — deferred.
- Releases tab still showing rustdesk's own download links on the OLD release → re-run with current workflow or
  edit that release's notes once.

## 11. Reference facts

- rustdesk source `read_custom_client()`: `src/common.rs` ~line 2181; `decode64` ~line 1799.
- Android empty-config callers: `flutter/lib/models/native_model.dart:217`, `MainService.kt:246`
  (`ffi.kt` declares `external fun startServer(app_dir, custom_client_config)`).
- `ndk_arm64.sh` / `ndk_arm.sh` / `ndk_x64.sh` + `build_android_deps.sh` live in `flutter/`.
- Bridge codegen (flutter_rust_bridge) is a separate job whose output is restored as an artifact.
