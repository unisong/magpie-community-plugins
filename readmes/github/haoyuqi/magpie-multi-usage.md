# magpie-multi-usage

**English** | [简体中文](README.zh-CN.md)

An open-source, zero-dependency multi-device token usage dashboard plugin for [Magpie](https://github.com/yetone/magpie), using a dedicated private GitHub repository as a sanitized stats-only relay — with zero Magpie backup decryption keys, zero writes to `~/.config/magpie`, and zero external cron jobs.

## Highlights

- **Works immediately out of the box**: Displays your local machine's usage (both gateway requests and Magpie-discovered native agent sessions such as Claude Code, OpenCode, Codex, and Antigravity) even before configuring a GitHub relay.
- **Magpie-native UI**: Matches Magpie's built-in Usage page layout, light/dark themes, and `--magpie-*` host tokens, with English and Chinese (`auto` / `en` / `zh`) language selection in Settings.
- **Independent multi-device aggregation**: Combines workstations, laptops, and headless `magpie serve` nodes without merging configs, providers, or sync targets.
- **Zero decryption passphrase**: Never touches Magpie backups, credentials, or sync keys; relays only sanitized aggregate counters via conflict-free per-device JSON files.
- **Pure ESM, zero runtime dependencies**: Runs cleanly on Magpie's bundled Bun host and Node.js 22+.

## Why this plugin exists

Magpie's built-in sync supports sharing usage across machines, but ties every machine to a single sync target and shares one encryption passphrase between full configuration backups (`.magpie-backup`) and usage files (`.magpie-usage`).

When you run Magpie across multiple machines with independent configurations — such as a primary workstation, headless servers, and intermittently online laptops — this plugin gives you a unified token usage dashboard without:

- Merging configs, providers, or sync targets across machines;
- Exposing Magpie gateway ports or requiring Tailscale, public IPs, or SSH access;
- Providing Magpie's backup decryption passphrase to a third-party plugin.

## How it works

1. **Reads local usage from Magpie**: Incrementally pulls this machine's own usage feed from Magpie's loopback gateway API (`GET /v1/magpie/usage/requests?computer=this&cursor=...`), including both gateway traffic and native agent sessions discovered by Magpie.
2. **Syncs headless servers automatically**: Uses Magpie's read-only async `magpie.hooks` (`onResponse`, debounced) so headless `magpie serve` nodes publish updates automatically without external cron jobs.
3. **Publishes atomically to a dedicated private GitHub repo**: Writes sanitized monthly snapshots and a manifest into conflict-free per-device folders using the Git Data API (single atomic commit with compare-and-swap ref updates), strictly refusing public repositories before any read or write.
4. **Aggregates on read inside Magpie**: Renders combined and per-device totals, two-tone daily usage bars, clickable device filter rows, and tabbed **Agents / Models / Providers** breakdowns. When a remote device fails or is offline, its last hash-verified cached snapshot is preserved and flagged as cached.

## Installation

Install directly from GitHub via Magpie's CLI:

```bash
magpie plugin add github:haoyuqi/magpie-multi-usage
```

Or install from a local clone:

```bash
git clone https://github.com/haoyuqi/magpie-multi-usage.git
magpie plugin add /path/to/magpie-multi-usage
```

Open **Multi-Device Usage** (**多端用量**) in the Magpie sidebar — local usage loads automatically with zero setup.

## Multi-device relay setup

### 1. Create a dedicated private GitHub repository

Create a new **Private** GitHub repository dedicated solely to sanitized usage stats (for example, `<you>/magpie-usage-stats`) and initialize its default branch (such as `main` with a README).

> [!IMPORTANT]
> Do **not** reuse your Magpie configuration sync repository, and do **not** use a public repository. This plugin verifies `private === true` before every read and ref update, and refuses public repositories (`PUBLIC_REPO_REFUSED`).

### 2. Create a Fine-grained GitHub PAT

Create a GitHub **Fine-grained personal access token** scoped strictly to **Only select repositories → your dedicated stats repo**:

- **Publishing machine** (workstation or headless server): `Contents: Read and write` (GitHub automatically includes `Metadata: Read-only`).
- **View-only machine** (optional read-only dashboard): `Contents: Read-only` (uncheck **Publish this machine's usage** in Settings).

### 3. Configure each machine

- **Magpie Desktop / `magpie web`**:
  1. Open **Multi-Device Usage** in the Magpie sidebar and click the **Settings** gear icon.
  2. Set **Device display name** (optional), **Private GitHub repository** (`owner/repo`), and **GitHub Fine-grained PAT**.
  3. Click **Test connection**, then **Save and sync**.
- **Headless `magpie serve` node**:
  1. Set non-sensitive options (`repo`, `deviceName`, `pushEnabled`, `pushIntervalSec`) via `magpie plugin options`.
  2. Supply the GitHub PAT via the environment variable `MAGPIE_MULTI_USAGE_GITHUB_TOKEN` (or store `{"githubToken":"..."}` in `<dataDir>/credentials.json` with strict mode `0600`). Never place a PAT in plaintext `plugins.json`.

> [!NOTE]
> Automatic background publishing on headless servers is triggered after completed gateway requests. If a headless machine routes zero traffic through Magpie's gateway (100% direct native agent sessions only), open the plugin page or trigger a refresh to publish those sessions.

## Configuration reference

### Settings & plugin options

| Field | Default | Description |
| --- | --- | --- |
| `lang` | `"auto"` | UI language (`auto`, `en`, or `zh`), persisted in `<dataDir>/config.json`. |
| `deviceName` | `""` | Custom label for this machine. Falls back to Magpie's `usage-computer.json` name, then OS hostname, then device ID. |
| `repo` | `""` | Dedicated private GitHub repository in `owner/repo` format. |
| `branch` | `""` | Optional target branch (defaults to the repository's default branch). |
| `prefix` | `"devices"` | Folder prefix inside the repository (`<prefix>/<device-id>/...`). |
| `pushEnabled` | `true` | Publish this machine's sanitized snapshots to the relay repo. Disable for read-only viewer mode. |
| `pushIntervalSec` | `300` | Minimum debounce interval (seconds, `30`–`86400`) between automatic hook-triggered pushes. |
| `filterForwarded` | `true` | Exclude cross-Magpie forwarded calls (`via !== ""`) to prevent double-counting when both nodes report. |
| `port` | `18792` | Loopback port for the local page service (`127.0.0.1`). |
| `dataDir` | `""` | Optional override for the plugin's isolated data directory. |

### Environment variables

| Variable | Description |
| --- | --- |
| `MAGPIE_MULTI_USAGE_GITHUB_TOKEN` | Fine-grained GitHub PAT (takes precedence over `<dataDir>/credentials.json`). |
| `MAGPIE_MULTI_USAGE_DATA_DIR` | Override plugin data directory (default: `$XDG_DATA_HOME/magpie-multi-usage` or `~/.local/share/magpie-multi-usage` on POSIX; `%LOCALAPPDATA%\magpie-multi-usage` on Windows). |
| `MAGPIE_ADDR` | Optional loopback Magpie gateway address (falls back to read-only `<magpieConfigDir>/settings.json`, then `http://127.0.0.1:3425`). |
| `MAGPIE_CONFIG_DIR` | Optional read-only Magpie configuration directory path. |

## Relay repository layout

Each device writes exclusively under its own `<prefix>/<device-id>/` folder in a single atomic commit, eliminating cross-machine merge conflicts:

```text
devices/
└── <device-id>/
    ├── manifest.json   # device metadata, coverage window, and SHA-256 hash per month
    ├── 2026-10.json    # sanitized daily & model/agent/provider aggregate counters
    └── 2026-11.json
```

Only aggregate counters (`calls`, `errors`, `excludedForwarded`, `input`, `output`, `cacheRead`, `cacheWrite`, `cost`) and dimension labels (`model`, `agent`, `provider`) are stored — never prompts, responses, headers, keys, or IPs.

## Token accounting

- **Total tokens**: `input + output + cacheRead + cacheWrite` (cache tokens are never double-counted inside `input`).
- **Cache hit rate**: `cacheRead / (input + cacheRead + cacheWrite)`, matching Magpie's `Totals.HitRate()`.
- **Forwarding deduplication**: Calls with a non-empty `via` field are excluded by default (`filterForwarded: true`) and reported in the `forwarded excluded` counter.

## Privacy and security guarantees

- Never asks for or reads Magpie's backup decryption passphrase, `.magpie-backup`, `.magpie-usage`, `providers.json`, `logins.json`, `sync.json`, or `plugin-auth.json`.
- Never writes to `~/.config/magpie/*`; all plugin state, caches, and credentials live in an isolated data directory outside Magpie's configuration folder.
- Never uploads prompts, model responses, provider keys, account emails, or client IPs.
- Re-verifies `private === true` and immutable `repoId` before reads and ref updates, refusing public repositories (`PUBLIC_REPO_REFUSED`).
- Stores GitHub PATs only in `0600` `<dataDir>/credentials.json` (refusing group/world-readable files on POSIX) or `MAGPIE_MULTI_USAGE_GITHUB_TOKEN`, never in `plugins.json`, logs, or API responses.
- Binds the local page service strictly to `127.0.0.1` with loopback `Host`/`Origin`/navigation checks, scoped `Origin: null` sandbox CORS, and a random per-process `X-Multi-Usage-Token` header.

## Troubleshooting

- **`PUBLIC_REPO_REFUSED`**: The configured repository is public. Switch to a dedicated private repository.
- **`BRANCH_NOT_FOUND` on Test connection**: Initialize the target branch on GitHub first (for example, add an initial README when creating the repo).
- **Device name shows hostname (e.g. `dev`)**: Set a friendly name in **Settings → Device display name** and click **Save and sync**.
- **Reset local plugin state**: Remove `<dataDir>` (`~/.local/share/magpie-multi-usage` by default); your Magpie history in `~/.config/magpie` is untouched and will re-ingest cleanly on next open.

## Documentation & development

- Architecture and file index: [docs/code-map.md](docs/code-map.md)
- Engineering and security standards: [docs/code-standards.md](docs/code-standards.md)

All tests run completely offline against isolated temporary directories and local HTTP fixtures:

```bash
HOME=$(mktemp -d) node --test test/*.test.mjs
HOME=$(mktemp -d) bun test test/
```

## License

MIT License — see [LICENSE](LICENSE).
