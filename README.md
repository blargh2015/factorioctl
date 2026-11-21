# factorioctl

`factorioctl` is a single-file Python command-line manager for **Factorio headless dedicated servers on Linux**. It bootstraps independent server instances, controls them with user-level systemd, installs game and mod updates, schedules daily restarts, and provides an interactive console over local RCON.

It uses only Python's standard library. Systemd unit templates are embedded in the script, so there are no separate unit files to distribute.

## Features

- Numbered, independent servers (`server-1`, `server-2`, etc.).
- Game and mod updates automatically on startup (configurable).
- Fast mod-portal master-index lookup with a 60-minute cache.
- Bulk enable/disable of named mods, including downloads and mandatory dependencies.
- Daily scheduled restarts that never start an intentionally stopped game.
- Interactive or one-shot administration through localhost-only RCON.
- Checks for missing saves, config files, mod ZIPs, and disabled timers.

## Requirements and assumptions

- Linux x86-64, **systemd user services**, and **Python 3.9+**.
- Network access to Factorio downloads and the mod portal if you want automatic updates.
- Mod-portal download credentials for third-party portal mods.
- An existing game save ZIP. **Bootstrap does not generate a save.**
- The supported layout is a dedicated Linux user named `factorio`, with the script at `/home/factorio/factorioctl` and instances under `/home/factorio/server-N`.

The `FACTORIO_HOME` environment variable can change the instance root for direct CLI use, but the embedded systemd unit currently has `/home/factorio` paths hardcoded. If using another account or directory, modify the embedded unit template before installing it.

For background user services that survive logout, an administrator can enable lingering:

```bash
sudo loginctl enable-linger factorio
```

All routine `factorioctl` operations run as the unprivileged `factorio` user, not root.

## Install

Extract the release ZIP **directly into `/home/factorio`**. It contains `factorioctl`, `README.md`, and `HISTORY.md` with no enclosing directory. As the `factorio` user:

```bash
cd /home/factorio
chmod +x ./factorioctl
./factorioctl --help
./factorioctl units sync
```

`units sync` offers to install or update embedded systemd templates, shows changes, and backs up existing files before replacing them. Other normal management commands also check the units. For scripted unit installation, place the approval flag **before** the command: `./factorioctl --yes units sync`.

## Set up your first server

```bash
./factorioctl bootstrap 1
```

Bootstrap downloads the headless game, creates the instance configuration and directories, and enables `factorio@1.service` and `factorio-restart@1.timer`. It **does not start the server**, does not create a save, and refuses to overwrite an existing `server-1` directory.

Copy your own game files into the new installation:

| What | Destination |
| --- | --- |
| Save (`*.zip`) | `/home/factorio/server-1/saves/` |
| `mod-list.json` | `/home/factorio/server-1/mods/mod-list.json` |
| `mod-settings.dat` (optional) | `/home/factorio/server-1/mods/mod-settings.dat` |
| Local or unpublished mod ZIPs | `/home/factorio/server-1/mods/` |
| Server configuration | `/home/factorio/server-1/data/server-settings.json` |

Edit the server settings for your game name, visibility, and authentication. To download portal mods, supply the `service-username` and `service-token` fields in `server-settings.json` (or supported credentials in `player-data.json`). Keep credentials private.

Then:

```bash
./factorioctl mods update 1
./factorioctl doctor 1
./factorioctl start 1
./factorioctl status 1
./factorioctl logs 1
```

To add another server, run `./factorioctl bootstrap 2`, then provide its own save and mod list under `server-2`.

## Commands

| Command | What it does |
| --- | --- |
| `bootstrap N [--branch stable\|experimental]` | Install a fresh instance; enable systemd service and daily timer. |
| `start N` | Start the instance, running configured startup maintenance. |
| `stop N` | Stop the instance gracefully. |
| `restart N` | Restart, checking for game and mod updates again. |
| `status N` | Show service and restart-timer status. |
| `logs N` | Follow the server journal. |
| `doctor N` | Validate required installation, save, mod, and timer details. |
| `update N` | Update game and enabled mods while stopped. |
| `update N --binary-only` | Update only the Factorio binary/data. |
| `update N --mods-only` | Update only enabled mods. |
| `mods update N` | Update installed enabled mods and dependencies while stopped. |
| `enmod N ModA ModB` | Enable/add named mods and download needed files; stop first. |
| `dismod N ModA ModB` | Disable named mods, keeping ZIPs; stop first. |
| `console N` | Open interactive game console via RCON. |
| `console N /command args` | Send a single game-console command. |
| `schedule N daily HH:MM` | Enable daily restart at server-local time. |
| `schedule N disable` | Turn off daily restarts. |
| `schedule N status` | Inspect the restart timer. |
| `units sync` | Review/install embedded systemd units. |

`run-server N` is an **internal systemd entry point**, not intended for interactive use. Use `./factorioctl COMMAND --help` for detailed options.

### Mod management

Names must match the mod's exact, case-sensitive Factorio name. The server must be stopped before modifying its installed mods:

```bash
./factorioctl stop 1
./factorioctl enmod 1 AutoDeconstruct DiscoScience
./factorioctl dismod 1 DiscoScience
./factorioctl start 1
```

`enmod` adds/enables the requested names in `mod-list.json`, updates other enabled mods, and fetches required dependencies. It backs up the mod list and restores it if the update fails; already-verified downloaded ZIPs are retained. `dismod` disables entries but retains the ZIPs. Factorio still validates actual mod compatibility at game startup.

**Safety guard:** A default installation refuses to start with **zero enabled third-party mods** to avoid accidentally launching a previously modded save as vanilla. If you deliberately want a vanilla or expansion-only game, set `ALLOW_VANILLA_MOD_LIST = True` near the top of the script.

### Game console and administrators

At startup, a random RCON password is created in `server-N/.rcon-password` with owner-only permissions. RCON is bound to **127.0.0.1**, not exposed publicly. If you are upgrading from a version without RCON support, restart the game once to activate it.

```bash
./factorioctl console 1
```

Then issue game commands at the prompt:

```text
factorio[1]> /promote ExamplePlayer
factorio[1]> /players
factorio[1]> exit
```

For a one-shot command:

```bash
./factorioctl console 1 /promote ExamplePlayer
```

**Security note:** Factorio takes the RCON password as a process command-line argument. Other local users with permission to inspect processes might see it. Restrict shell access, and never expose the RCON port on your firewall.

### Scheduled restarts and automatic updates

Bootstrap enables a **02:00 daily restart**, using the server's local timezone. Customize or disable it:

```bash
./factorioctl schedule 1 daily 03:30
./factorioctl schedule 1 status
./factorioctl schedule 1 disable
```

The timer only **restarts an already-running instance**. It will not launch a server you manually stopped. The main systemd service has its own enablement for startup. Check timer activation directly with `systemctl --user list-timers --all`.

Configure startup maintenance using constants near the beginning of `factorioctl`:

```python
AUTO_UPDATE_ON_START = True
AUTO_UPDATE_BRANCH = 'stable'
AUTO_UPDATE_GAME = True
AUTO_UPDATE_MODS = True
START_IF_UPDATE_FAILS = False
ALLOW_VANILLA_MOD_LIST = False
```

Updates run **on startup or scheduled restart**, not continuously while the game is running. By default, network or mod-update failures prevent game startup rather than risking inconsistent saves. `START_IF_UPDATE_FAILS = True` favors availability over that safety guard.

## Ports and files

| Instance | Game (UDP) | Console (TCP, localhost only) | systemd service |
| --- | ---: | ---: | --- |
| 1 | 34197 | 27015 | `factorio@1.service` |
| 2 | 34198 | 27016 | `factorio@2.service` |
| N | 34196 + N | 27014 + N | `factorio@N.service` |

Forward only the **UDP game port** required for external players. Each instance has its own `bin/`, `data/`, `config/`, `mods/`, and `saves/` directories. The daily restart timer is `factorio-restart@N.timer`, which triggers a separate `factorio-restart@N.service` oneshot helper.

## Troubleshooting

- **Server not found:** Bootstrap that instance first (`factorioctl bootstrap N`).
- **No save or failed doctor:** Check `server-N/saves/` and validate the copied save ZIP.
- **Missing mod/download failure:** Check enabled mod names, Factorio-version compatibility, portal credentials, and network access. Local unpublished mods can be copied as ZIPs.
- **Game refuses startup after an update error:** Review `factorioctl logs N`; this is intentional by default to protect saves from mismatched mod sets.
- **Console connection refused:** Ensure the instance is running and was restarted after adding RCON support.
- **No daily restart:** Inspect `factorioctl schedule N status`, then enable it with `factorioctl schedule N daily 02:00` if appropriate.
- **Unexpected systemd-unit prompt:** Review the proposed embedded-template changes before accepting.

## Security and backups

Use a dedicated unprivileged user. Keep `server-settings.json`, `player-data.json`, and `.rcon-password` private. Back up saves and mod configurations before major upgrades; retaining older mod ZIPs does **not** back up game saves. Do not publish save files, logs, or authentication tokens without review.

The CLI has been syntax-tested and the RCON client tested against a mock server; it has not been comprehensively tested with every Factorio/Linux combination.

See [HISTORY.md](HISTORY.md) for version notes and major fixes.
