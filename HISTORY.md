# Release history

This document tracks notable changes. See [README.md](README.md) for current usage. Version notes deliberately omit user-specific player names, hosts, addresses, and credentials.

## 1.14.1

- Fixed RCON startup: `--rcon-bind` already specifies the port and cannot be combined with `--rcon-port`.

## 1.14

- Added interactive `console N` and one-shot `console N /command`.
- Added per-instance generated RCON credentials with a localhost-only listener.

## 1.13

- Added bulk `enmod` and `dismod` commands for enabled mods.
- Added automatic downloads/dependency checks when enabling mods; preserved old ZIPs and backed up mod lists.

## 1.12

- Bootstrap enables both the main service and the daily 02:00 restart timer.
- Added timer state to `status` and `doctor`, plus the `schedule` command.

## 1.11

- Fixed cross-device ZIP move errors by staging downloads on the destination filesystem.

## 1.1–1.10 (earlier development)

- Consolidated game updates, mod installation, and server lifecycle commands into one script.
- Embedded systemd templates, with consent prompts and backups when updating unit files.
- Added clean bootstrap without save creation, instance checks, and clearer output.
- Corrected config-path handling and older Python tar-extraction compatibility.
- Excluded bundled Factorio mods from portal downloads.
- Replaced slow per-mod lookups with a cached mod-portal master index.
- Improved dependency parsing, including version constraints such as `base >= 2.1`.
- Added configurable automatic game and mod updates at startup.
- Added guardrails for accidentally empty third-party mod lists and graceful startup-maintenance interruption.
