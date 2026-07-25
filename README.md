# rustdesk-builder-v2

Custom RustDesk client builder — based on [bryangerlach/rdgen](https://github.com/bryangerlach/rdgen) workflow logic.

## How it works

- **Daily cron** (02:00 UTC) checks `rustdesk/rustdesk` for new release tags
- **Manual trigger** builds any version on demand
- **Bridge files** generated once, shared by all platforms
- **All customizations** (server, key, appname, company, permissions) read from `configs/RustDesk.json`
- **Outputs** go to GitHub Releases with download page

## Setup

### 1. Repository secrets

| Secret | Value |
|--------|-------|
| `ANDROID_SIGNING_KEY` | Base64-encoded `.jks` keystore |
| `ANDROID_ALIAS` | Key alias (e.g. `rustdesk`) |
| `ANDROID_KEY_STORE_PASSWORD` | Keystore password |
| `ANDROID_KEY_PASSWORD` | Key password |

### 2. Enable permissions

**Settings → Actions → General → Workflow permissions:** Read and write permissions

### 3. Enable GitHub Pages

**Settings → Pages → Source:** Deploy from a branch → `main` / `docs`

### 4. Edit config

Edit `configs/RustDesk.json` with your server, key, app name, and permissions.

### 5. Build

**Actions → Build RustDesk Custom Client → Run workflow**

## Config

All customizations are in `configs/RustDesk.json`:

- `serverIP` — your RustDesk server
- `key` — your server's public key
- `apiServer` — API server URL
- `appname` — display name
- `compname` — company name
- `permanentPassword` — default password
- `delayFix` — connection delay fix
- `hidecm` — hide connection manager
- And all permission flags

## Credits

Build logic adapted from [bryangerlach/rdgen](https://github.com/bryangerlach/rdgen).
