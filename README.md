# factorioctl v1.14.1

v1.14.1 fixes startup on Factorio 2.1: `--rcon-bind` already specifies the
listening port and cannot be combined with `--rcon-port`.

## Server console (v1.14)

RCON is enabled automatically at the next server start, bound **only to 127.0.0.1**.
Each instance uses a distinct TCP port (server-1: 27015, server-2: 27016).
A random password is created automatically in `server-N/.rcon-password`, with
owner-only permissions. Do not share it. The RCON password is passed as a Factorio
command-line argument, so other trusted local users may be able to see it in
process listings; keep shell access to the host restricted.

After replacing `factorioctl`, restart the instance **once** to activate RCON:

```bash
./factorioctl restart 1
./factorioctl console 1
# At the factorio[1]> prompt:
/promote Blargh2015
exit
```

Or run a single console command directly:

```bash
./factorioctl console 1 /promote Blargh2015
```

The console works on the Factorio host; it does not expose RCON publicly.

Single-file Factorio headless server manager for Python 3.9+ (standard library only). The three systemd unit templates are embedded in `factorioctl`; on normal management commands it offers to install/update the user units when needed.

## Changes since v1.11

- `bootstrap N` now enables **both** `factorio@N.service` (start on boot) and `factorio-restart@N.timer` (daily 02:00 local time). It does not start the game or create a save.
- `status N` shows the systemd service and timer enabled/active/next-run information.
- `doctor N` warns if startup updates are enabled but the daily restart timer isn't active.
- `schedule N daily HH:MM` can change the time; `schedule N disable` opts out.
- Retains the cross-filesystem mod-download fix introduced in v1.11.

## Upgrade existing installation

As the `factorio` user, extract this ZIP directly into `/home/factorio` (no subfolder) and make `factorioctl` executable. The script will prompt before changing systemd unit definitions.

```bash
chmod +x /home/factorio/factorioctl
./factorioctl status 1
./factorioctl schedule 1 daily 02:00  # if upgrading an existing instance with a disabled timer
```

## Fresh server-1

```bash
./factorioctl stop 1
# Save anything you want from server-1, then remove the old instance directory manually.
rm -rf /home/factorio/server-1
./factorioctl bootstrap 1
```

Copy your new save ZIP into `/home/factorio/server-1/saves/`, your `mod-list.json` and `mod-settings.dat` into `/home/factorio/server-1/mods/`, and any local/unpublished mod ZIPs into the same mods directory. Then:

```bash
./factorioctl mods update 1
./factorioctl doctor 1
./factorioctl start 1
./factorioctl status 1
```

Automatic updates are enabled by default on game startup (editable constants near the top of the script). The timer only restarts an already-running server; it won't launch a server that's been deliberately stopped. Missing mandatory third-party mod ZIPs fail validation rather than silently allowing an unmodded start.

**Note:** This version was syntax-checked and CLI-tested, but systemd behavior and downloads have not been exercised on your host.

## Enable and disable mods (v1.13)

Stop the instance first; both commands refuse to edit a running server.

```bash
factorioctl stop 1
factorioctl enmod 1 UPSFriendlyNixieTubeDisplay
factorioctl enmod 1 AutoDeconstruct DiscoScience
factorioctl dismod 1 AutoDeconstruct DiscoScience
factorioctl start 1
```

`enmod` adds missing mod names to `mod-list.json`, enables requested entries,
then uses the mod updater to download the requested mods and their required
dependencies. It also updates other enabled mods as part of this operation.
A timestamped copy of the old mod list is retained. If downloading fails,
the mod list is restored; any verified downloaded ZIPs remain available for retry.
If a requested mod is already enabled, `enmod` still checks/install its ZIP.

`dismod` changes existing entries to `enabled: false`, without removing ZIPs.
It refuses unknown mod names. Dependency-related errors remain possible when
other enabled mods depend on a disabled mod; Factorio checks these at startup.
Both commands accept multiple **exact, case-sensitive** portal mod names.
Mod changes take effect when the game next starts.
