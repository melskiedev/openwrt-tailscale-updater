# openwrt-tailscale-updater

A vanilla OpenWrt Tailscale **updater** for routers with Tailscale already installed. On a router with no Tailscale yet, running the script from SSH shows a first-run menu with install instructions.

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

- Vanilla OpenWrt (any version). Vendor firmware builds are not supported (see [Known Limitations](#known-limitations))
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
| **1** | Install via OpenWrt package manager (recommended) |
| **2** | Install upstream static binaries |
| **e** | Exit |

Option 1 prints the package-manager commands to run. Option 2 is not available from this script; install with the package manager instead. On apk routers you can then use [Migration Mode](#migration-mode) to switch to upstream binaries.

**Manual package install:**

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
| `--dry-run` | Show detected values, latest version, init layout problems, the owning procd service, and duplicate or unmanaged `tailscaled` processes without making changes |
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

`--yes` skips update and delete confirmations. With no other action flags, it also skips the main menu and proceeds with a stable update. Migration is not confirmed by `--yes`; use `--yes-i-have-lan-access` for that. `--rollback` and `--delete-backup` require an SSH session (terminal) and work even when Tailscale is not currently installed, as long as backups exist.

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
- Detects split init layouts (an init script symlink, two init scripts, or a boot link that registers a different procd service name) and normalizes them to one real `/etc/init.d/tailscaled` while `tailscaled` is stopped. See [Init Script Ownership](#init-script-ownership)
- Refuses to update when more than one `tailscaled` is running, or when the running one is not owned by a procd service
- Resolves symlinks with `readlink -f` before overwriting, and removes symlinks safely before installing separate binaries
- Fetches latest stable version from `pkgs.tailscale.com/stable`
- Verifies downloaded tarball checksum (`.sha256`) before extraction
- Menu shows latest stable, release-candidate, and unstable versions (cached for 5 minutes)
- Backs up current binaries, init script, and `/etc/tailscale/` before any change
- Stops `tailscaled` through the procd service that owns it, then waits past the procd respawn timeout and aborts before installing if the daemon came back
- Installs new binaries, starts `tailscaled`, and verifies the dataplane (see [Init Script Ownership](#init-script-ownership))
- Adds key paths to `/etc/sysupgrade.conf` for persistence across sysupgrade, and drops entries for init paths that no longer exist

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
8. Installs separate upstream `tailscale` and `tailscaled` binaries
9. Normalizes the init script to one real `/etc/init.d/tailscaled` (the package script is copied, the original is backed up and removed; no symlink is created)
10. Starts `tailscaled` and verifies the dataplane
11. Updates `/etc/sysupgrade.conf`

**Node identity is preserved.** `/etc/tailscale/` is not touched. No re-auth required.

**Tailscale connectivity will drop briefly during migration.** Only run this when you have physical or LAN access to the router.

```sh
/usr/bin/openwrt-tailscale-updater --migrate-from-apk-multicall --yes-i-have-lan-access
```

After migration, use this updater for all future Tailscale updates. Do not use `apk upgrade tailscale`.

---

## Init Script Ownership

OpenWrt `rc.common` names the procd service after the target of the init script path it was called through, one link level only. If the same init script can be reached under two names, one `tailscaled` can end up registered under two procd service names. Then `/etc/init.d/tailscaled stop` acts on the wrong name, procd respawns the daemon under the other name, and a duplicate `tailscaled` can remove the running daemon's addresses from `tailscale0`.

Earlier versions of migration mode created `/etc/init.d/tailscaled` as a symlink to `/etc/init.d/tailscale`, which produces exactly this layout. The updater no longer creates init symlinks.

**Normalization.** The canonical layout is one real file at `/etc/init.d/tailscaled`. If the updater finds an init script symlink, both `/etc/init.d/tailscale` and `/etc/init.d/tailscaled`, or a boot link (`/etc/rc.d/S*`) that registers a different service name than the init script, it normalizes during the update while `tailscaled` is stopped:

1. Backs up both init paths to `/root/tailscale-updater/init-backup-<timestamp>/`
2. Removes the Tailscale boot links in `/etc/rc.d/`
3. Writes one real `/etc/init.d/tailscaled` (copied from the existing real script) and removes `/etc/init.d/tailscale`
4. Runs `/etc/init.d/tailscaled enable`

A router that already has one real `/etc/init.d/tailscaled` is not touched. A plain package install with only `/etc/init.d/tailscale` is also left as is. On vendor firmware the updater reports the layout problem and changes no init scripts or boot links. `--dry-run` reports the layout problem, the owning procd service, and any duplicate or unmanaged `tailscaled` processes, but never normalizes and never stops on those problems. If the router is already on the latest version, no update runs and nothing is normalized; use `--force` to normalize.

**Stop.** The updater finds the procd service whose instance PID is the running `tailscaled` and removes that service (`ubus call service delete`), plus any Tailscale service procd still holds in a crash loop. `killall` is only a last resort. It then waits longer than the procd respawn timeout (5 seconds by default) and aborts before installing anything if `tailscaled` came back. The old version keeps running in that case.

**Start.** The updater refuses to start `tailscaled` if any `tailscaled` process or Tailscale procd service still exists.

**Verification.** `tailscale status` alone is not enough: it can succeed while the dataplane is broken. Before stopping, the updater records the node's backend state from `tailscale status --json`. After the respawn window it always requires:

- exactly one `tailscaled` process
- owned by exactly one procd service, matching the init script
- no `safesocket.Listen`, `stale Tailscale address`, or procd `crash loop` lines in syslog since the start

If the backend state was `Running` before the update, it also requires the node's IPv4 from `tailscale ip -4` to be present on `tailscale0`. A node that was logged out, stopped, or not readable before the update passes on the process and owner checks alone, and the updater says so; check `tailscale status` yourself in that case.

If any check fails, the update stops with an error. Update, migration, and rollback all use the same stop, start, and verification steps.

---

## Known Limitations

- **Vendor firmware is not supported.** Vendor firmware builds ship and manage their own Tailscale integration. Use the vendor's update path there. If this updater is run on vendor firmware anyway, it never normalizes init scripts.
- **No coordination with external restart tools.** The updater does not take a lock that other tools (for example a cron watchdog that restarts `tailscaled` when the dataplane looks unhealthy) would respect. Such a tool can restart `tailscaled` while an update has it stopped, which can start a daemon before the new binaries are in place. Pause any such watchdog while updating.

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
ip -4 addr show dev tailscale0
ls -l /etc/init.d/tailscale* /etc/rc.d/*tailscale*
ubus call service list '{"name":"tailscaled"}'
ubus call service list '{"name":"tailscale"}'
```

Expect one PID, one real init file, and one procd service listing that PID.

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

Rollback restores the `tailscale` and `tailscaled` binaries and the init script using paths recorded in the backup. Node identity at `/etc/tailscale/` is not touched.

Backups taken on a router with the old init symlink contain that symlink. Rollback never restores an init symlink: it keeps the init script already on the router and normalizes the layout if needed. Rollback checks that a usable init script exists before it stops anything, then stops, starts, and verifies `tailscaled` the same way an update does.

Backups created by v1.4.0-rc1 and later include `.tailscale-backup-meta` inside the tarball (binary paths, init script, version, multicall flag). A readable copy is also written beside the archive as `tailscale-backup-<version>-<timestamp>.tar.gz.meta`.

Older backups without metadata still work when Tailscale binaries are currently installed: rollback falls back to the currently detected paths and prints a warning. That fallback may fail if the router layout changed since the backup was created (for example, before or after multicall migration). Backups with metadata can be restored even when binaries and the init script are missing, as long as the backup archive is intact.

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
/root/tailscale-updater/backups/tailscale-backup-<version>-YYYYMMDD-HHMMSS.tar.gz.meta
```

Includes current binaries, init script, `/etc/tailscale/`, and `.tailscale-backup-meta` inside the archive. The `.meta` sidecar is a human-readable copy of the metadata for inspection without extracting the tarball.

Tar warnings (if any) are written to `/root/tailscale-updater/backup-<timestamp>.log` instead of being discarded.

State and cache files live under `/root/tailscale-updater/` (`backups/`, `theme.conf`, `version-cache-*`).

---

## License

MIT
