# openwrt-tailscale-updater

A vanilla OpenWrt Tailscale **updater** for routers with Tailscale already installed. On a router with no Tailscale yet, running the script from SSH shows a first-run install menu (package install and upstream binary install are rolling out in v1.4.0).

No vendor-specific logic. No UPX. No tiny binaries. It detects `apk` or `opkg`, resolves the active Tailscale binary and init paths, protects multicall installs, and can migrate apk-managed multicall layouts to standalone upstream binaries when explicitly requested.

---

## Disclaimer

This script is provided for personal use only. Use at your own risk.

The author is not responsible for any damage, data loss, bricked devices, broken networks, or any other issues that may result from using this script.

Always perform a full sysupgrade backup before running this script. Ensure you have physical or LAN access to your router before proceeding with any update or migration.

By using this script, you acknowledge that you understand the risks and accept full responsibility for the outcome.

---

## Quick Start

**Using wget** (most OpenWrt routers):
```sh
wget -q https://raw.githubusercontent.com/melskiedev/openwrt-tailscale-updater/main/openwrt-tailscale-updater -O /usr/bin/openwrt-tailscale-updater && chmod +x /usr/bin/openwrt-tailscale-updater && /usr/bin/openwrt-tailscale-updater
```

**Using curl** (if wget lacks HTTPS support):
```sh
curl -sL https://raw.githubusercontent.com/melskiedev/openwrt-tailscale-updater/main/openwrt-tailscale-updater -o /usr/bin/openwrt-tailscale-updater && chmod +x /usr/bin/openwrt-tailscale-updater && /usr/bin/openwrt-tailscale-updater
```

> Run manually. Do not add to cron.

---

## Requirements

- Vanilla OpenWrt (any version)
- Tailscale installed for update mode (see [First-Time Install](#first-time-install) if nothing is installed yet)
- At least 60 MB free on `/tmp`
- `wget` or `curl` for downloads (the script installs `wget` if missing, then tries `curl`)
- `sha256sum` for checksum verification (`coreutils-sha256sum` on OpenWrt if not present)
- Supported CPU: `arm64`, `arm`, `amd64`, `386`, `mips`, `mipsle`

The script must use Unix line endings (LF). If you copy it from Windows and `sh -n` fails, run `tr -d '\r'` on the router before use.

---

## First-Time Install

If Tailscale is not installed, run the script from SSH with no flags:

```sh
/usr/bin/openwrt-tailscale-updater
```

You will see a first-run menu:

| Option | Description |
|---|---|
| **1** | Install via OpenWrt package manager (recommended; full automation in v1.4.0-rc1) |
| **2** | Install upstream static binaries (v1.4.0-rc2) |
| **e** | Exit |

Until v1.4.0-rc1/rc2 ship, option 1 prints the manual package-manager commands. Option 2 is not available yet.

**Manual package install** (works today):

```sh
# OpenWrt 25.12+
apk update && apk add tailscale

# OpenWrt 24.10 and older
opkg update && opkg install tailscale
```

Then run `tailscale up`, and verify the updater sees your install:

```sh
/usr/bin/openwrt-tailscale-updater --dry-run
```

If the package install uses a multicall layout (both binaries resolve to one file), the updater detects it and offers migration to standalone upstream binaries. See [Migration Mode](#migration-mode).

**Partial or broken installs** (some pieces missing) are refused. The script prints what is missing and suggests `apk fix tailscale` or `opkg install tailscale`. It does not auto-repair.

---

## Menu

Run with no arguments from SSH:

```sh
/usr/bin/openwrt-tailscale-updater
```

**If Tailscale is installed**, you get the updater menu. The dashboard shows router info, installed version, latest stable / release-candidate / unstable versions (fetched from `pkgs.tailscale.com`, cached for 5 minutes under `/root/tailscale-updater/version-cache-*`), and the active theme.

| Section | Options |
|---|---|
| Update | stable, unstable, release candidate, or pick a specific stable version |
| Recovery / Maintenance | rollback, delete old backups |
| Options | theme picker (`t`), reset defaults (`r`), about (`a`), exit (`e`) |

**If Tailscale is not installed**, you get the first-run install menu instead. See [First-Time Install](#first-time-install).

`--theme NAME` and `--reset-theme` set the theme and exit. They do not run an update. Use menu option `t` to change theme from the updater menu.

---

## Supported OpenWrt Versions

| OpenWrt Version | Package Manager | Status |
|---|---|---|
| 25.12+ | `apk` | Supported |
| 24.10 and older | `opkg` | Supported |

---

## Arguments

| Argument | Description |
|---|---|
| *(none)* | Show menu (updater menu if installed, first-run install menu if not) |
| `--force` | Reinstall even if already on latest version |
| `--dry-run` | Show detected values and latest version without making changes |
| `--keep-tmp` | Keep downloaded files in `/tmp/tailscale-update` after install |
| `--yes` / `-y` | Skip confirmation prompts (use with `--force` to skip the menu and update) |
| `--verbose` | Show detailed detection and operation logs |
| `--theme NAME` | Set menu theme: `classic`, `green`, `amber`, `ocean`, or `mono` |
| `--reset-theme` | Reset saved menu theme to `classic` |
| `--unstable` | Use the unstable track (odd minor versions, e.g. 1.97.x). Not for production. |
| `--release-candidate` | Use the release-candidate track. Use only for testing. |
| `--select-release` | List available releases from the active track and choose a specific version |
| `--rollback` | Restore `tailscale` and `tailscaled` binaries from a backup |
| `--delete-backup` | List available backups and delete selected entries |
| `--migrate-from-apk-multicall` | Convert apk-managed multicall install to standalone upstream binaries |
| `--yes-i-have-lan-access` | Required confirmation flag for migration mode |
| `--help` | Show help |

`--select-release` lists versions from the currently selected track. Combine with `--unstable` or `--release-candidate` to select from those tracks:

```sh
/usr/bin/openwrt-tailscale-updater --select-release
/usr/bin/openwrt-tailscale-updater --unstable --select-release
/usr/bin/openwrt-tailscale-updater --release-candidate --select-release
```

> Stable is always the recommended track for routers and exit nodes.

`--yes` skips update and delete confirmations. It does not skip the menu by itself. Migration is not confirmed by `--yes`; use `--yes-i-have-lan-access` for that. `--rollback` and `--delete-backup` require an SSH session (terminal).

---

## Usage Examples

**Standard update:**
```sh
/usr/bin/openwrt-tailscale-updater
```

**Dry run (no changes):**
```sh
/usr/bin/openwrt-tailscale-updater --dry-run
```

**Force reinstall:**
```sh
/usr/bin/openwrt-tailscale-updater --force
```

**Update without prompts:**
```sh
/usr/bin/openwrt-tailscale-updater --yes --force
```

**Rollback to a previous version:**
```sh
/usr/bin/openwrt-tailscale-updater --rollback
```

**Delete old backups:**
```sh
/usr/bin/openwrt-tailscale-updater --delete-backup
```

**Set theme from CLI:**
```sh
/usr/bin/openwrt-tailscale-updater --theme ocean
/usr/bin/openwrt-tailscale-updater --reset-theme
```

**Migrate from apk multicall layout** (see Migration Mode below):
```sh
/usr/bin/openwrt-tailscale-updater --migrate-from-apk-multicall --yes-i-have-lan-access
```

---

## What It Does

- Detects install state: complete, empty, multicall, or partial (refuses partial installs)
- Detects CPU architecture and maps to the correct Tailscale tarball
- Detects package manager (`apk` or `opkg`) by available command
- Detects Tailscale binary paths dynamically, supports `/usr/sbin` and `/usr/bin` layouts
- Detects init script name (`/etc/init.d/tailscaled` or `/etc/init.d/tailscale`)
- Resolves symlinks with `readlink -f` before overwriting, and removes symlinks safely before installing separate binaries
- Fetches latest stable version from `pkgs.tailscale.com/stable`
- Verifies downloaded tarball checksum (`.sha256`) before extraction
- Menu shows latest stable, release-candidate, and unstable versions (cached for 5 minutes)
- Backs up current binaries, init script, and `/etc/tailscale/` before any change
- Stops `tailscaled`, installs new binaries, restarts and verifies
- Adds key paths to `/etc/sysupgrade.conf` for persistence across sysupgrade

## What It Does NOT Do

- Does not run `tailscale up` automatically
- Does not change ACLs, auth keys, exit node approval, or DNS settings
- Does not touch any router firmware wrappers or vendor-specific scripts

---

## Multicall Layout Protection

Some OpenWrt builds install Tailscale as a single shared binary where `/usr/sbin/tailscale` is a symlink to `/usr/sbin/tailscaled`. Overwriting both paths with separate binaries in this layout would corrupt the install.

This updater detects the multicall layout and refuses to proceed in normal mode:

```
[tailscale-updater] Detected multicall Tailscale layout:
[tailscale-updater]   /usr/sbin/tailscale -> /usr/sbin/tailscaled
[tailscale-updater]   /usr/sbin/tailscaled -> /usr/sbin/tailscaled
[tailscale-updater] This updater does not overwrite shared multicall binaries.
[tailscale-updater]   apk update && apk upgrade tailscale
[tailscale-updater] To convert this router to standalone upstream binaries, run:
[tailscale-updater]   openwrt-tailscale-updater --migrate-from-apk-multicall --yes-i-have-lan-access
[tailscale-updater] Refusing to overwrite multicall layout.
```

Use `apk upgrade tailscale` to stay package-managed, or use migration mode to convert to standalone binaries. Migration mode is **apk only**. On opkg routers with a multicall layout, the menu offers stay package-managed or cancel only.

---

## Migration Mode

Migration mode converts an apk-managed multicall Tailscale install to standalone upstream binaries tracked by this updater instead of `apk`.

**When to use it:**
- Your router uses OpenWrt 25.12+ with `apk`
- Tailscale is installed as an apk package
- The apk repo version is behind upstream and no upgrade is available via `apk upgrade tailscale`
- You want to track upstream Tailscale releases directly

**What it does:**
1. Verifies preconditions (apk-managed, node state present, init script present, LAN confirmed)
2. Backs up current binaries, init script, and `/etc/tailscale/`
3. Downloads the latest upstream tarball
4. Stops `tailscaled`
5. Removes the apk package entry with `apk del tailscale`
6. Restores the init script if apk removed it
7. Reinstalls `kmod-tun` if apk removed it as a dependency
8. Creates `/etc/init.d/tailscaled` compatibility symlink if needed
9. Installs separate upstream `tailscale` and `tailscaled` binaries
10. Starts `tailscaled` and verifies
11. Updates `/etc/sysupgrade.conf`

**Node identity is preserved.** `/etc/tailscale/` is not touched. No re-auth required.

**Tailscale connectivity will drop briefly during migration.** Only run this when you have physical or LAN access to the router.

```sh
/usr/bin/openwrt-tailscale-updater --migrate-from-apk-multicall --yes-i-have-lan-access
```

After migration, use this updater for all future Tailscale updates. Do not use `apk upgrade tailscale`.

---

## GitHub Latest vs Static Tarball

This updater checks GitHub releases for visibility only.

Installation always uses the latest verified static tarball available from `pkgs.tailscale.com/<track>/` for the detected CPU architecture.

If GitHub shows a newer version but the static tarball is not yet available, the updater warns and installs only the latest available static tarball.

Downloaded tarballs are verified against the matching `.sha256` checksum from the Tailscale package server before extraction.

---

## Verify After Install

```sh
tailscale version
tailscale status
pidof tailscaled
ip link show tailscale0
```

Reapply your preferred router state if needed:

```sh
tailscale up --advertise-exit-node --accept-routes --accept-dns=false
```

---

## Rollback

Each update creates a timestamped backup before making any changes. If an update causes issues, you can restore the previous binaries:

```sh
/usr/bin/openwrt-tailscale-updater --rollback
```

The rollback menu lists available backups with version and timestamp:

```
  [1] version 1.98.3  (20260522-202937)  tailscale-backup-1.98.3-20260522-202937.tar.gz
  [2] version 1.98.2  (20260521-110000)  tailscale-backup-1.98.2-20260521-110000.tar.gz
Select backup to restore [1-2]:
```

Rollback restores only the `tailscale` and `tailscaled` binaries. Node identity at `/etc/tailscale/` is not touched.

---

## Backup Management

To free up space, delete old backups:

```sh
/usr/bin/openwrt-tailscale-updater --delete-backup
```

You can delete a single backup by number or all backups at once. Both options require confirmation.

---

## Backups

Each run creates a timestamped backup before making any changes:

```
/root/tailscale-updater/backups/tailscale-backup-<version>-YYYYMMDD-HHMMSS.tar.gz
```

Includes current binaries, init script, and `/etc/tailscale/`.

State and cache files live under `/root/tailscale-updater/` (`backups/`, `theme.conf`, `version-cache-*`).

---

## License

MIT
