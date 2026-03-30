---
title: "VN Presence"
description: "Discord Rich Presence daemon for Visual Novels on Linux — detects games launched via Lutris or Steam, looks them up on VNDB, and displays title, cover art, and playtime in your Discord status."
date: 2026-03-30T01:27:09-07:00
image: 
categories:
    - linux
    - Coding
tags:
    - tools
    - c++
comments: true
draft: true
build:
    list: always    # Change to "never" to hide the page from the list
---

[![Github Repo](https://img.shields.io/badge/github-repo-blue?logo=github)](https://github.com/jayng9663/VN-Presence)
[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![C++23](https://img.shields.io/badge/C%2B%2B-23-00599C?logo=cplusplus)](https://en.cppreference.com/w/cpp/23)

Discord Rich Presence daemon for Visual Novels on Linux. Detects games launched via **Lutris** or **Steam**, looks them up on [VNDB](https://vndb.org), and shows the title, cover art, and total playtime in your Discord status.

### Voice channel

![Voice channel presence with cover](voice_with_cover.png) ![Voice channel presence without cover](voice_no_cover.png)

### User activity

![User activity with cover](activity_with_cover.png) ![User activity without cover](activity_no_cover.png)

---

## Features

- Detects games from **Lutris** (exact name from wrapper) and **Steam** (AppID → ACF → store name)
- Fuzzy title matching against VNDB using trigram similarity + full-width character normalisation
- Fallback: searches the **release** endpoint and follows the relation back to the parent VN
- Suppresses explicit cover images (sexual ≥ 1.80 or violence ≥ 1.80) — title still shows
- Playtime from **Lutris DB** (`pga.db`) or **Steam VDF** (`localconfig.vdf`) — no API key needed
- Discord elapsed timer reflects **total hours played**, not just this session
- Editable `cache.csv` for aliases, hard-links, and SKIP entries — reloaded live while running
- `ignore.txt` with exact-match for short entries (< 4 chars) to prevent false positives like `sh`

---

## How it works

### Main loop

Every 5 seconds the daemon runs detection, debounces the result, checks the cache, queries VNDB if needed, and updates Discord.

```mermaid
flowchart LR
    A[Detection\nLutris or Steam] --> B[Debounce\nstable 2 polls]
    B --> C{Cache\nlookup}
    C -- hit + fresh --> E[Discord RPC]
    C -- hit + expired --> D[VNDB search]
    C -- miss --> D
    D --> E
```

---

### Detection

```mermaid
flowchart TD
    P[Scan /proc every 5s]
    P --> L{lutris-wrapper\nin cmdline?}
    L -- yes --> LN[Game name from\nwrapper args]
    L -- no --> S{SteamAppId\nin environ?}
    S -- yes --> SA[Read appmanifest_ID.acf\nfrom all library paths]
    SA --> SN[Steam store name]
    S -- no --> NONE[Nothing detected]
    LN --> ST[Read starttime\nfrom /proc/pid/stat field 22]
    SN --> ST
    ST --> SORT[Sort candidates\nby starttime ascending]
```

Lutris passes the game name explicitly as command-line arguments after `lutris-wrapper`, so the name is always exact. Steam injects `SteamAppId` as an environment variable into every game process — the daemon reads it from `/proc/<pid>/environ`, then finds the matching `.acf` file across all Steam library paths (including custom drives read from `libraryfolders.vdf`).

After all candidates are collected, the daemon reads **field 22** (`starttime`) from each process's `/proc/<pid>/stat`. Candidates are then **sorted ascending by starttime**, so the process that launched first is always tried first during VNDB resolution. This makes multi-candidate priority deterministic and reproducible across polls.

> [!NOTE]
> See the [proc/[PID]/stat field reference](/blog/p/proc/pid/stat-field-reference/) for a full breakdown of all stat fields.

> [!TIP]
> Run with `--verbose` to see candidates list in the debug output:
> ```
> [DEBUG] src/main.cpp:73 4 game candidate(s) found:
>   [DEBUG] src/main.cpp:75   [lutris] pid=2037324  name="終ノ空 remake"  starttime(clock ticks)=4906595
>   [DEBUG] src/main.cpp:75   [lutris] pid=2086738  name="X-Plane 12"  starttime(clock ticks)=4906701
>   [DEBUG] src/main.cpp:75   [steam-appid] pid=2037822  name="心象天儀本線 ~Per aspera ad astra~ Demo"  starttime(clock ticks)=4906857
>   [DEBUG] src/main.cpp:75   [lutris] pid=2038526  name="サクラノ詩－櫻の森の上を舞う－"  starttime(clock ticks)=4907260
> ```

---

### VNDB resolution

```mermaid
flowchart TD
    Q[Search query] --> VN[POST /kana/vn]
    VN --> SIM{Trigram similarity\n≥ 0.35 OR substring match?}
    SIM -- yes --> MATCH[VnInfo returned]
    SIM -- no --> REL[POST /kana/release]
    REL --> FOUND{Release found?}
    FOUND -- yes --> ID[Fetch parent VN\nby exact ID]
    ID --> MATCH
    FOUND -- no --> NOMATCH[No match\nclear presence]
```

Before the similarity check, full-width ASCII (`！２→!2`, `（→(`) is normalised to half-width so Japanese game names from Lutris match VNDB's stored titles. There is also a substring boost — if the query appears inside the returned title, the score is raised to at least 0.6, handling cases where the detected name is a short prefix of a long VNDB title.

---

### Cache TTL

VNDB results are stored persistently in `cache.csv`. On each cache hit the daemon compares the entry's `cached_at` timestamp against `VNDB_CACHE_TTL`. If the entry is older than the TTL, it is treated as expired and a fresh VNDB query is issued, updating the stored data and timestamp.

```mermaid
flowchart TD
    HIT[Cache hit] --> AGE{age > VNDB_CACHE_TTL?}
    AGE -- no --> USE[Use cached VnInfo]
    AGE -- yes --> REQUERY[Re-query VNDB\nupdate cache.csv]
    REQUERY --> USE
```

This means ratings, cover images, and release dates in the cache are automatically refreshed over time without any manual intervention. The log line when a re-query fires looks like:

```
[INFO] Cache expired for "..."  age=1442min/1440min — re-querying VNDB
```

---

### Playtime → Discord elapsed timer

```mermaid
flowchart TD
    SRC{Source?}
    SRC -- lutris --> LDB[Lutris DB\npga.db · hours]
    SRC -- steam-appid --> VDF[Steam VDF\nlocalconfig.vdf · minutes]
    LDB --> PT[playtime in seconds]
    VDF --> PT
    PT --> TS[start_ts = now − playtime]
    TS --> DISC[Discord: elapsed timer\ncounts up from start_ts]
```

---

### Cache file (`cache.csv` / `cache.db`)

Located at `~/.config/vn-discord-rpc/cache.csv` (default) or `cache.db` when `CACHE_USE_DB = true`. Reloaded automatically whenever the file changes on disk.

| Column | Purpose |
|---|---|
| `key` | Detected Visual Novels name (the search term) |
| `alias` | Redirect this key to a different search term |
| `vndb_id` | `v562`, `v67`, `SKIP`, or empty |
| `title` | Romanised VNDB title (auto-filled) |
| `alt_title` | Original script title, e.g. Japanese (auto-filled) |
| `image_url` | Cover image URL (auto-filled, blank if explicit(sexual or violence)) |
| `image_sexual` | VNDB sexual rating 0–2 (auto-filled) |
| `image_violence` | VNDB violence rating 0–2 (auto-filled) |
| `rating` | VNDB community rating 0–100 (auto-filled) |
| `released` | Release date (auto-filled) |
| `cached_at` | Unix timestamp of last write (auto-filled) |

**Examples:**

```csv
# Alias — fix wrong or garbled detection
Nice boat!,School Days,,,,,,,,

# Skip — suppress presence for this title
妹ぱらだいす！２,,SKIP,,,,,,,

# Hard-link — bypass VNDB query, point directly to an entry
My VN Title,,v67,,,,,,,
```

---

### Ignore list (`ignore.txt`)

Located at `~/.config/vn-discord-rpc/ignore.txt`. One entry per line, `#` for comments. Reloaded automatically while running.

Matching rules:
- Entry **< 4 characters**: exact match (case-insensitive) — (ie. prevents `sh` matching `Higurashi`)
- Entry **≥ 4 characters**: case-insensitive substring match

The default file includes common false-positives: Steam runtimes, Proton, Wine helpers, and launchers.

> [!WARNING]
> If a title has no VNDB match and is not in the ignore list, the daemon will not add it
> to ignore automatically. It will retry the VNDB query on every detection. Add a `SKIP`
> entry in `cache.csv` or an entry in `ignore.txt` to suppress it permanently.

---

## Configuration (`src/config.hpp`)

> [!CAUTION]
> Be caution on changing `IMAGE_SEXUAL` and `IMAGE_VIOLENCE` thresholds in `src/config.hpp`.
> Those thresholds are set to `1.80` (slightly below VNDB's **Explicit** level of `2.00`)
> because VNDB image ratings are user-reported and may be underrated by a small margin —
> the 0.20 buffer ensures borderline explicit covers are still suppressed.
> Raising the threshold above `2.00` would cause explicit (pornographic) cover art to appear
> in your Discord status, visible to everyone on your friends list. Displaying explicit
> content in Discord Rich Presence may violates [Discord's Terms of Service](https://discord.com/terms)
> and **may result in a permanent account ban**.

| Constant | Default | Description |
|---|---|---|
| `DISCORD_APP_ID` | `1482345564698841189` | Discord application ID |
| `DISCORD_ACTIVITY_TYPE` | `0` | Activity type: 0=Game, 1=Streaming, 2=Listening, 3=Watching |
| `IMAGE_SEXUAL` | `1.80` | Maximum sexual rating before cover is suppressed |
| `IMAGE_VIOLENCE` | `1.80` | Maximum violence rating before cover is suppressed |
| `IMAGE_VOTECOUNT` | `5` | Minimum vote count before ratings are trusted |
| `VNDB_MIN_SIMILARITY` | `0.35` | Minimum trigram score to accept a VNDB match |
| `POLL_INTERVAL` | `5s` | How often to scan `/proc` for running processes |
| `VNDB_CACHE_TTL` | `24h` | How long a cache entry is valid before re-querying VNDB |
| `STABLE_TITLE_POLLS` | `2` | Polls a candidate must be stable before acting |
| `VNDB_MAX_RESULTS` | `1` | Maximum results per VNDB query (index 0 is used) |
| `CACHE_USE_DB` | `false` | Use SQLite (`cache.db`) instead of CSV *(experimental)* |
| `RPC_STATE_READING` | `"Reading"` | Discord state string while a VN is active |
| `RPC_STATE_IDLE` | `"Idle"` | Discord state string while idle |
| `RPC_DEFAULT_DETAILS` | `"Playing a Visual Novel"` | Details line when no VNDB match is found |
| `RPC_SMALL_IMG_KEY` | `"vndb_logo"` | Discord asset key for the small VNDB logo image |
| `RPC_SMALL_IMG_TEXT` | `"VNDB"` | Tooltip text for the small image |

---
 
## Architecture
 
The project is composed of nine focused, single-responsibility modules:
 
| Module | Source Files | Responsibility |
|---|---|---|
| **Entry point** | `main.cpp` | Main loop, debounce state machine, candidate resolution pipeline |
| **Process scanner** | `process_scanner.cpp/.hpp` | Iterates `/proc`, detects Lutris and Steam game processes, reads `starttime` |
| **Steam detector** | `steam_detector.cpp/.hpp` | Reads `SteamAppId` env vars, parses ACF manifests and VDF playtime files |
| **VNDB client** | `vndb_client.cpp/.hpp` | HTTP POST to VNDB Kana API, trigram matching, release endpoint fallback |
| **VN cache** | `vn_cache.cpp/.hpp` | Persistent CSV/SQLite cache with alias, SKIP, TTL, and live-reload logic |
| **Ignore list** | `ignore_list.cpp/.hpp` | Live-reloadable process-name suppression list with exact/substring rules |
| **RPC manager** | `rpc_manager.cpp/.hpp` | Discord IPC wrapper with rate limiting, deferred flush, and change detection |
| **Config** | `config.hpp` | All compile-time constants in one place |
| **Logger** | `logger.hpp` | Thread-safe ANSI-coloured logger singleton with `LOG_DEBUG/INFO/WARN/ERR` macros |
| **Lutris DB** | `lutris_db.cpp/.hpp` | Reads and formats playtime from Lutris's SQLite `pga.db` |
 
### Key Design Decisions
 
**Deterministic multi-candidate priority** — when multiple games are running, candidates are sorted by `/proc/<pid>/stat` field 22 (`starttime`, clock ticks since boot). The process that launched earliest is always tried first, making priority stable and reproducible across polls without any user configuration.
 
**Discord rate-limit compliance** — Discord silently drops `SET_ACTIVITY` calls faster than ~15 seconds apart. `RpcManager` tracks the wall-clock time of the last successful push and defers any call that falls within a 16-second window. Deferred updates are flushed on the next `runCallbacks()` tick, ensuring no update is ever lost.
 
**No presence flicker on stable sessions** — the early-out in the main loop checks whether the currently displayed VN is still in the candidate set before doing any cache or VNDB work. If it is still running, the loop sleeps immediately, so Discord is never unnecessarily cleared and re-set.
 
**Cache-first, ignore-list-second** — VNDB HTTP queries are only issued on a true cache miss or TTL expiry. Candidates that fail VNDB resolution are added to the ignore list so subsequent polls skip them instantly, letting the next candidate be tried without any network round trip.
 
**Live file reloading without restart** — both `cache.csv` and `ignore.txt` are checked for `mtime` changes on every poll using `std::filesystem::last_write_time`. Edits made while the daemon is running take effect within one poll cycle with no signals or process restart needed.
 
**Explicit content safety** — image ratings are only trusted when backed by at least `IMAGE_VOTECOUNT` votes. Cover URLs are stripped from cache entries for explicit results so they can never accidentally appear after a cache reload even if thresholds are later changed.
 
---

## Building

> [!IMPORTANT]
> Initialise submodules before building — the two header-only dependencies are not
> downloaded automatically.
> ```bash
> git submodule update --init --recursive
> ```

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

## Dependencies

| Library | Purpose |
|---|---|
| [discord-presence](https://github.com/EclipseMenu/discord-presence) | Discord Rich Presence (modern C++ rewrite) |
| [nlohmann/json](https://github.com/nlohmann/json) | JSON parsing |
| [libcurl](https://curl.se/libcurl/) | HTTP requests to VNDB API |
| [libsqlite3](https://www.sqlite.org/) | Read Lutris playtime database |

---

## Usage

> [!TIP]
> Run with `--verbose` if a title is not being detected or matched — the debug output shows
> exactly which process was found, what VNDB returned, and the similarity score.

```bash
./vn-discord-rpc            # normal
./vn-discord-rpc --verbose  # debug logging
./vn-discord-rpc --help
```

Launch a game through **Lutris** or **Steam**, and the daemon will detect it automatically. Press `Ctrl+C` to quit cleanly.

---

## File locations

| File | Path |
|---|---|
| Cache (CSV) | `~/.config/vn-discord-rpc/cache.csv` |
| Cache (SQLite, experimental) | `~/.config/vn-discord-rpc/cache.db` |
| Ignore list | `~/.config/vn-discord-rpc/ignore.txt` |
| Lutris DB | `~/.local/share/lutris/pga.db` |
| Steam VDF | `~/.local/share/Steam/userdata/<id>/config/localconfig.vdf` |

