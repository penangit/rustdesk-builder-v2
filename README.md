# RustDesk Custom Client Builder (Neo⁺ / Infinite Remote)

Build **your own pre-configured [RustDesk](https://rustdesk.com/) clients** — pointed at *your*
self-hosted server, with your server address, key, app name, company, permanent password, and
permissions baked in — automatically, for **Windows, Linux, Android, and macOS**, using GitHub
Actions. Every build is published to GitHub Releases and listed on an auto-updating download page.

This project stands on:
- **[bryangerlach/rdgen](https://github.com/bryangerlach/rdgen)** — the build/workflow logic this is adapted from.
- **[rdgen.crayoneater.org](https://rdgen.crayoneater.org/)** — the hosted config generator used to create `configs/RustDesk.json`.
- **[rustdesk/rustdesk](https://github.com/rustdesk/rustdesk)** — the upstream open-source app that actually gets built.
- **Infinite Remote (`ir.remote-neo.com`)** — the self-hosted deployment this repo is configured for.

> Deep technical notes (the build mechanism, every gotcha, current state) live in **`SKILL.md`**. This
> README is the step-by-step setup guide.

---

## Table of contents
1. [What it produces](#1-what-it-produces)
2. [How it works (high level)](#2-how-it-works-high-level)
3. [Prerequisites](#3-prerequisites)
4. [Repository layout](#4-repository-layout)
5. [Step-by-step setup (A–Z)](#5-step-by-step-setup-az)
6. [Config field reference](#6-config-field-reference)
7. [The workflows explained](#7-the-workflows-explained)
8. [The download page](#8-the-download-page)
9. [Security & repo visibility](#9-security--repo-visibility)
10. [Troubleshooting](#10-troubleshooting)
11. [Credits](#11-credits)

---

## 1. What it produces

Per RustDesk version, one GitHub Release containing:

| Platform | Variants |
|---|---|
| **Windows** | 64-bit `.exe` (portable) + `.msi` (installer) |
| **Android** | signed `.apk` for **arm64, armv7, x86_64**, plus a **universal** APK (all ABIs in one) |
| **Linux** | `.deb`, `.rpm` (Fedora), `.rpm` (openSUSE), `.AppImage`, `.flatpak` — each for **x86_64 + arm64** — plus Arch `.pkg.tar.zst` (x86_64) |
| **macOS** | `.dmg` *(optional)* |

Filenames follow `RustDesk-<version>-<arch>.<ext>` so the download page can sort them automatically.

## 2. How it works (high level)

Each workflow checks out the **official RustDesk source** at a chosen version tag, **patches your
settings in**, builds, and **uploads to a GitHub Release**. Your settings land in two ways:

- **Compiled into the binary** (via `sed` on the Rust source): server address, public key, API server,
  app name, company. These work on every platform/format.
- **Read at runtime from a bundled `custom_.txt`**: the permanent password, permission defaults, and
  approval mode.

The **download page** (`docs/index.html`) reads your releases live from the GitHub API and shows every
variant, the upstream changelog, a live "build in progress" indicator, and device-aware download
recommendations.

## 3. Prerequisites

1. **Your own RustDesk server** — self-hosted `hbbs`/`hbbr` (see
   [rustdesk-server](https://github.com/rustdesk/rustdesk-server)) or RustDesk Server Pro. You need its
   **address** and **public key**.
2. A **GitHub account** and a repository (public recommended — see §9).
3. **Java's `keytool`** (bundled with any JDK) for the Android signing key.

## 4. Repository layout

Create a repo with exactly these paths:

```
your-repo/
├── .github/
│   └── workflows/
│       ├── build-windows.yml     # 64-bit Windows → .exe + .msi
│       ├── build-linux.yml       # full Linux set (deb/rpm/AppImage/flatpak/Arch, x86_64 + arm64)
│       └── build-android.yml     # per-arch APKs + universal APK
├── configs/
│   └── RustDesk.json             # YOUR config (created in Step 2)
├── scripts/
│   └── load-config.py            # reads the JSON → CUSTOM_* env vars
├── docs/
│   └── index.html                # the download page
├── README.md
└── SKILL.md                      # deep technical notes (optional but recommended)
```

## 5. Step-by-step setup (A–Z)

### Step 1 — Create the repository
Create a new GitHub repo and add the files from §4. Keep it **public** (see §9 for why).

### Step 2 — Generate your config
1. Open **<https://rdgen.crayoneater.org/>**.
2. Fill in:
   - **Server IP / address** — e.g. `ir.remote-neo.com`
   - **Key** — your server's **public key**
   - **API server** — usually `https://<your-server>/`
   - **App name** (keep `RustDesk` to skip renaming), **Company name**
   - **Permanent password** and **approval mode** (e.g. *password + click*)
   - **Permissions** (keyboard, clipboard, file transfer, audio, …)
3. Download the generated `.json`.
4. Save it in your repo as **`configs/RustDesk.json`** (commit it).

> Keeping `appname = RustDesk` avoids invalid package names; set a custom name only for full white-labelling.

### Step 3 — Create the Android signing key
Android APKs must be signed. Generate a keystore **once**:

```bash
keytool -genkey -v -keystore rustdesk-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias rustdesk
```

It prompts for a **keystore password**, a **key password**, and identity fields. Note the passwords and
the alias (`rustdesk`).

Base64-encode the keystore so it can be stored as a secret:

```bash
# Linux
base64 -w0 rustdesk-release.jks > keystore_b64.txt
# macOS
base64 -i rustdesk-release.jks -o keystore_b64.txt
# Windows (PowerShell)
[Convert]::ToBase64String([IO.File]::ReadAllBytes("rustdesk-release.jks")) > keystore_b64.txt
```

### Step 4 — Add repository secrets
Repo → **Settings → Secrets and variables → Actions → New repository secret**. Add all four:

| Secret name | Value |
|---|---|
| `ANDROID_SIGNING_KEY` | the whole base64 string from `keystore_b64.txt` |
| `ANDROID_ALIAS` | the key alias (e.g. `rustdesk`) |
| `ANDROID_KEY_STORE_PASSWORD` | the keystore password |
| `ANDROID_KEY_PASSWORD` | the key password |

> ⚠️ **Never commit** `rustdesk-release.jks` or `keystore_b64.txt`. They belong only in Secrets.
>
> These four are for Android. Optional code-signing secrets for **macOS** and **Windows**
> (`MACOS_P12_BASE64`, `MACOS_P12_PASSWORD`, `WINDOWS_PFX_BASE64`, `WINDOWS_PFX_PASSWORD`) are covered in
> [Step 8](#step-8--optional-code-signing) — skip them if you don't have certificates.

### Step 5 — Enable workflow write permissions
Repo → **Settings → Actions → General → Workflow permissions** → select **Read and write permissions**
(so the workflows can publish releases and update the download page). Save.

### Step 6 — Host the download page
Either:
- **GitHub Pages:** Settings → Pages → *Deploy from a branch* → `main` / `docs`, **or**
- **Cloudflare Pages:** connect the repo, set the output directory to `docs/`.

Then edit `docs/index.html` and set the repo constant near the top of `<script>`:
```js
const REPO = 'your-username/your-repo';
```

### Step 7 — Run a build
Repo → **Actions** → pick **🪟 Build Windows / 🐧 Build Linux / 🤖 Build Android** → **Run workflow**:
- **Version** — leave blank for the latest RustDesk release, or type one (e.g. `1.4.9`).
- **force_build** — keep `true` to add a platform's builds to an existing release.

Run them one at a time the first time. When a run finishes, its files appear in the Release and on the
download page automatically.

Once you're happy, use **🚀 Build All** to build every platform in one run — and it runs on a daily
schedule that only builds when a *new* stable RustDesk version appears (nothing new = it skips fast).
(`build-all.yml` calls the four workflows as reusable workflows, so their filenames in
`.github/workflows/` must be exactly `build-windows.yml`, `build-linux.yml`, `build-android.yml`,
`build-macos.yml`.)

### Step 8 — Optional: code signing

By default the builds are **unsigned** and work fine — users just get a one-time OS warning. If you have
signing certificates, add the matching secrets and the workflows sign automatically.

**macOS** (Apple Developer ID cert, from the Apple Developer Program — ~$99/yr):
1. Export your *Developer ID Application* certificate as a `.p12`.
2. Base64 it: `base64 -i cert.p12 -o cert_b64.txt` (macOS) or `base64 -w0 cert.p12 > cert_b64.txt` (Linux).
3. Add secrets `MACOS_P12_BASE64` (the base64 string) and `MACOS_P12_PASSWORD` (the `.p12` password).

Without these, the DMG is unsigned → first launch shows *"unidentified developer / damaged"*; users
right-click the app → **Open**, or run `xattr -cr /Applications/RustDesk.app` once. (Full notarization,
which removes the warning entirely, is a further step we can add if you go that route.)

**Windows** (Authenticode code-signing cert, e.g. from a CA like DigiCert/Sectigo):
1. Export the cert as a `.pfx`.
2. Base64 it (PowerShell): `[Convert]::ToBase64String([IO.File]::ReadAllBytes("cert.pfx")) > pfx_b64.txt`.
3. Add secrets `WINDOWS_PFX_BASE64` and `WINDOWS_PFX_PASSWORD`.

Without these, the `.exe`/`.msi` are unsigned → SmartScreen shows *"unknown publisher"*; users click
**More info → Run anyway**. (Linux and Android don't use OS publisher certs the same way — Android is
already signed with your keystore from Step 3.)

## 6. Config field reference

Fields in `configs/RustDesk.json`:

- `serverIP` — your RustDesk server address
- `key` — your server's **public** key
- `apiServer` — API server URL
- `appname` — display name (`RustDesk` = stock / no rename)
- `compname` — company name shown in About
- `permanentPassword` — baked-in password for unattended access
- `passApproveMode` — e.g. `password-click` (needs password **and** an on-screen accept)
- `delayFix` — `on` reduces connection delay
- `hidecm` / `xOffline` / `removeNewVersionNotif` — behaviour toggles
- `enableKeyboard`, `enableClipboard`, `enableFileTransfer`, `enableAudio`, … — permission defaults
- `androidappid` — leave blank to keep the default app id (`com.carriez.flutter_hbb`)

## 7. The workflows explained

All three: resolve the version, generate the flutter↔rust bridge, check out RustDesk, apply the
customizations (see below), build, and publish to a Release. `run-name` includes the version so the
download page can show it.

**Customization steps (what patches what):**
- **Server / key / API** → `libs/hbb_common/src/config.rs`, `src/common.rs` (compiled in).
- **App name / company / URLs** → various Rust + Flutter files (app name skipped when `RustDesk`).
- **Strip signature check** → removes RustDesk's custom-client signature block in `src/common.rs` so your
  config is accepted, and renames `custom.txt` → `custom_.txt`.
- **`custom_.txt`** → written as **base64** (`CUSTOM_B64`) — the client base64-decodes it, so raw JSON
  will silently fail (this is the #1 gotcha; see `SKILL.md`).
- **Android only** → the base64 config is also embedded into `MainService.kt` and `native_model.dart`,
  because Android can't file-read `custom_.txt`; the scam-warning prompt is removed.

**Per workflow:**
- **build-windows.yml** — 64-bit Windows (Flutter) → `.exe` + `.msi`.
- **build-linux.yml** — the full 11-variant set; arm64 legs run on `ubuntu-22.04-arm`.
- **build-android.yml** — per-arch APKs (matrix) **+** a separate universal job that reuses the per-arch
  native libs and runs `flutter build apk` with no split → one APK for any device. If the universal job
  fails, the per-arch APKs still publish.

## 8. The download page

`docs/index.html` is a single self-contained file. Features: dark/light theme toggle, OS icons, hover
tooltips explaining which build to pick, **device detection** (highlights the recommended download for
the visitor's OS — Android is pointed at the Universal APK), a live build-in-progress banner (platform +
version + elapsed time), and the upstream changelog per release. It's driven entirely by the GitHub
Releases API — no server needed.

## 9. Security & repo visibility

The **only genuinely sensitive value** in the committed config is `permanentPassword`:
- Your server address and **public key are not secrets** — RustDesk embeds the key in every client you
  distribute, so it's public by design.
- With `password-click` approval, a leaked password alone can't connect (it needs an on-screen accept).

**Don't make the repo private to "hide" things** — it breaks the public download page (private-repo
releases + asset URLs 404 for anonymous visitors) and makes arm64 runners bill minutes. If the baked
password worries you: remove `permanentPassword` from the config (rely on click-approval / per-device
passwords), or keep a private build repo that publishes releases to a separate public repo / object store.

## 10. Troubleshooting

- **Baked password isn't applied / you have to type it** → `custom_.txt` must be **base64**
  (`CUSTOM_B64`), not raw JSON. On Android the config must also be embedded in `MainService.kt` +
  `native_model.dart`. The current workflows do both — redeploy them and re-run.
- **Play Protect warns on install (Android)** → inherent to any remote-access APK sideloaded outside the
  Play Store; tap "install anyway". Not fixable from the build.
- **Universal APK build failed but per-arch succeeded** → the universal job reuses the per-arch native
  libs; if it fails, per-arch APKs still publish. Re-run just that job, or download the per-arch APK.
- **Release notes show rustdesk's own download links** → that release was made by an older workflow;
  re-run with the current one (it extracts only the changelog) or edit the notes once.
- **arm64 Linux legs fail with billing/runner errors** → `ubuntu-22.04-arm` is free only on **public**
  repos.

## 11. Credits

- Build logic adapted from **[bryangerlach/rdgen](https://github.com/bryangerlach/rdgen)**
- Config generator: **[rdgen.crayoneater.org](https://rdgen.crayoneater.org/)**
- Upstream app: **[rustdesk/rustdesk](https://github.com/rustdesk/rustdesk)**
- Deployment: **Infinite Remote — `ir.remote-neo.com`**
