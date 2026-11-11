# mediaserver-autooff

Automatically powers off a home media server (built for a Proxmox host) when it's idle.

Every 15 minutes (configurable) it checks:

1. **Is anyone watching Plex or Jellyfin?** → stay on (paused streams count too, configurable).
2. **Is qBittorrent actively downloading?** (state `downloading`/`forcedDL`; stalled, seeding, paused and stopped torrents don't count) → stay on.
3. **Is the network busy?** Average download/upload over the whole interval vs. thresholds → stay on.
4. **Can't reach Plex/Jellyfin/qBittorrent?** → stay on (fail-safe).
5. **Idle for N checks in a row?** → shut down, unless a guard applies:
   boot grace period, pause file, outside the shutdown time window, or a user logged in.

Every check is written as one timestamped line to `/var/log/netwatch-shutdown.log` (lines older than 24 h are pruned automatically) and to the journal. Each `KEEP` line names **every** reason the node stayed on.

Pure bash + systemd timer. Needs `curl` and `python3` (for parsing Plex XML / Jellyfin + qBittorrent JSON). All decisions are logged to the journal.

## Install

```bash
install -m 755 netwatch-shutdown /usr/local/sbin/netwatch-shutdown
install -m 600 netwatch-shutdown.conf.example /etc/netwatch-shutdown.conf
nano /etc/netwatch-shutdown.conf        # set interfaces, thresholds, Plex token, Jellyfin API key, qBittorrent login
netwatch-shutdown --install             # creates + enables the systemd timer
```

It starts in **dry-run mode** (`DRY_RUN=1`): it only logs what it would do. Watch the log for a few hours, tune the thresholds, then set `DRY_RUN=0`.

## Usage

```bash
netwatch-shutdown --status              # what's keeping it awake right now + last 5 checks
netwatch-shutdown --log                 # every check from the last 24 h
journalctl -t netwatch-shutdown -f      # follow decisions
touch /etc/netwatch-shutdown.pause      # pause auto-shutdown (rm to resume)
netwatch-shutdown --install             # re-run after changing CHECK_INTERVAL_MIN
netwatch-shutdown --uninstall           # remove timer/service
```

Example log:

```
KEEP: media active | Plex: 1 (alice: Some Show - Episode [playing]); Jellyfin: 0; | avg over 900s on [nic0]: ...
IDLE 1/2 | avg over 900s on [nic0]: down 3.8 KB/s, up 5.5 KB/s, total 9.3 KB/s | ...
WOULD SHUT DOWN (DRY_RUN=1): idle 2/2 | ...
```

## Configuration

See [`netwatch-shutdown.conf.example`](netwatch-shutdown.conf.example). Every option is commented. Highlights:

| Option | Default | Meaning |
|---|---|---|
| `INTERFACES` | `nic0` | Interfaces to measure (the physical NIC covers host + all guests) |
| `CHECK_INTERVAL_MIN` | `15` | Check interval |
| `RX/TX/TOTAL_THRESHOLD_KBPS` | `0/0/500` | Avg KB/s that counts as "busy"; `0` disables |
| `CONSECUTIVE_LOW_CHECKS` | `2` | Idle intervals in a row before shutdown |
| `PLEX_*`, `JELLYFIN_*` | | Media server URL + token/API key |
| `QBIT_*` | | qBittorrent WebUI URL, username, password, which states count as downloading |
| `BOOT_GRACE_MIN` | `30` | Never shut down soon after boot |
| `SHUTDOWN_WINDOW` | empty | e.g. `23:00-07:00`; empty = any time |
| `SKIP_IF_USERS_LOGGED_IN` | `1` | Don't shut down while someone is logged in |
| `DRY_RUN` | `1` | Log only; set `0` to arm |
| `LOG_FILE` | `/var/log/netwatch-shutdown.log` | Per-check log; empty = journal only |
| `LOG_RETENTION_HOURS` | `24` | Prune older log lines; `0` = keep all |

The real config contains secrets: keep it `chmod 600` and never commit it (`.gitignore` excludes `*.conf`).

Powering the machine back on (Wake-on-LAN, BIOS RTC alarm, smart plug) is out of scope.
