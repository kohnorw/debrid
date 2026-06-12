# Debridarr

Stream-on-demand bridge between **Decypharr** (via AllDebrid), Sonarr, Radarr, Tautulli, and Plex. Placeholders appear in Plex for every monitored item. When you press Play, Debridarr finds the real file — from Decypharr's mount or Sonarr/Radarr's disk — and symlinks it instantly.

## How it works

```
Startup
  └─ Scan all Sonarr + Radarr instances
       └─ Create .mp4 placeholder for every monitored item
            └─ Stored under /streams/tv, /streams/movies
               (or /streams/<key>/tv, /streams/<key>/movies for extra instances)
               Decypharr torrents indexed at /mnt/decypharr/__all__

User presses Play in Plex
  └─ Tautulli fires Playback Start webhook → POST /webhook/tautulli
       └─ Debridarr matches title + season/episode to library
            └─ 1. Exact hash match via arr download history → Decypharr
               2. Fuzzy title+SE match against Decypharr index
               3. Trigger arr search, poll Decypharr for up to 45s
            └─ Symlink: placeholder path → real file on Decypharr mount
                 └─ Plex re-reads symlink and plays the real file
                    (+ optional Plex library refresh for instant swap)

Background
  └─ Pre-symlink pass: every library item already in Decypharr gets
     symlinked automatically so future plays are instant
  └─ Auto-match loop every 5 min catches anything newly added to AllDebrid
  └─ Decypharr index refreshed every 30s (+new torrents only)

30 days later
  └─ Symlink expires, placeholder restored, arr notified
     Pinned items: torrent re-added to AllDebrid and search re-triggered
```

## Quick start

```bash
cp .env.example .env
docker compose up -d
```

Open **http://localhost:7474** and follow the setup wizard to configure Sonarr, Radarr, Decypharr, Tautulli, and Plex.

## Prerequisites

Decypharr must be running and accessible. Debridarr talks to Decypharr over HTTP and optionally mounts its file tree for direct file access:

```yaml
volumes:
  - /docker/decypharr/mnt/decypharr/__all__:/mnt/decypharr/__all__:ro
```

If the mount path is configured (`DECYPHARR_MOUNT`), symlinks point directly to real files and Plex plays them natively. Without it, Debridarr still works but falls back to proxying.

The container needs `SYS_ADMIN` and `/dev/fuse` for legacy DFS FUSE compatibility:

```yaml
cap_add:
  - SYS_ADMIN
devices:
  - /dev/fuse:/dev/fuse:rwm
security_opt:
  - apparmor:unconfined
```

## Multiple instances

Debridarr supports any number of Sonarr and Radarr instances. Four are seeded by default: `sonarr`, `radarr`, `sonarr4k`, and `radarr4k` — the 4K pair starts blank, ready to configure.

Each instance has an immutable **key** (a slug derived from its name at creation) and a free-form **name** you can rename at any time. The default `sonarr`/`radarr` instances use the flat layout (`/streams/tv`, `/streams/movies`); any extra instance is namespaced under its key (`/streams/sonarr4k/tv`). Point a separate Plex library at each instance's folder so a 1080p and a 4K copy of the same show never collide.

## Plex library setup

| Library type | Path | Instance |
|---|---|---|
| TV Shows | `/streams/tv` | `sonarr` (default) |
| Movies | `/streams/movies` | `radarr` (default) |
| TV Shows | `/streams/sonarr4k/tv` | `sonarr4k` |
| Movies | `/streams/radarr4k/movies` | `radarr4k` |

Any custom-named instance lives at `/streams/<key>/tv` or `/streams/<key>/movies`.

Placeholders are real `.mp4` files (a short "please wait" video), so Plex always has something to show. When you press Play, Debridarr swaps the placeholder in place with a symlink to the real file.

**Instant swap (recommended):** set `PLEX_URL` and `PLEX_TOKEN` (or configure in the Connections tab). After Debridarr swaps in the real file it asks Plex to rescan, so playback starts without pressing Retry.

**NFO files:** Debridarr writes Kodi-format `.nfo` files alongside every placeholder so Plex can match metadata instantly without an online lookup. For best results set each library's Agent to **Plex NFO Series** / **Plex NFO Movie** (requires Plex Media Server ≥ 1.43.1).

## Tautulli webhook setup

1. Tautulli → Settings → Notification Agents → Add → **Webhook**
2. **Webhook URL:** `http://<debridarr-host>:7474/webhook/tautulli`
3. **Trigger:** Playback Start only
4. **JSON body** (Data tab):

```json
{"media_type":"{media_type}","title":"{title}","grandparent_title":"{grandparent_title}","year":"{year}","grandparent_year":"{grandparent_year}","parent_media_index":"{parent_media_index}","media_index":"{media_index}","rating_key":"{rating_key}","themoviedb_id":"{themoviedb_id}","thetvdb_id":"{thetvdb_id}","imdb_id":"{imdb_id}"}
```

> All settings and credentials can be configured through the web UI — leave env vars blank and use the wizard instead.

## Arr webhook setup

Point each Sonarr/Radarr instance's webhook at Debridarr so imports are symlinked immediately without waiting for a play event.

In Sonarr/Radarr: **Settings → Connect → Webhook**

| Trigger | Sonarr URL | Radarr URL |
|---|---|---|
| On Import, On Upgrade, On File Delete | `http://<host>:7474/webhook/sonarr` | `http://<host>:7474/webhook/radarr` |

For non-default instances use the per-instance URLs (shown in the Connections tab):

```
http://<host>:7474/webhook/sonarr/sonarr4k
http://<host>:7474/webhook/radarr/radarr4k
```

## Volumes

| Path | Purpose |
|---|---|
| `./data` | SQLite database (settings, library, streams) |
| `./streams` | Placeholder + symlink tree served to Plex |
| `/mnt/decypharr/__all__` | Decypharr mount (read-only; enables direct file symlinks) |

## Environment variables

Leave these blank to configure via the UI. The `SONARR_*`/`RADARR_*` vars only seed the **default** `sonarr` and `radarr` instances on first run; all other instances (including the 4K pair) live in the database and are managed from the wizard or Connections tab.

| Variable | Default | Description |
|---|---|---|
| `SONARR_URL` | — | e.g. `http://192.168.1.x:8989` |
| `SONARR_API_KEY` | — | Sonarr → Settings → General → API Key |
| `RADARR_URL` | — | e.g. `http://192.168.1.x:7878` |
| `RADARR_API_KEY` | — | Radarr → Settings → General → API Key |
| `TAUTULLI_URL` | — | e.g. `http://192.168.1.x:8181` |
| `TAUTULLI_API_KEY` | — | Tautulli → Settings → Web Interface → API key |
| `PLEX_URL` | — | e.g. `http://192.168.1.x:32400` |
| `PLEX_TOKEN` | — | X-Plex-Token for library refresh after swap |
| `DECYPHARR_URL` | `http://decypharr:8282` | Decypharr base URL |
| `DECYPHARR_API_TOKEN` | — | Decypharr API token (leave blank if auth disabled) |
| `DECYPHARR_MOUNT` | `/mnt/decypharr/__all__` | Path where Decypharr's `__all__` folder is mounted |
| `ALLDEBRID_API_KEY` | — | [alldebrid.com/apikeys](https://alldebrid.com/apikeys) — used by the built-in Torznab indexer |
| `SYMLINK_TTL_DAYS` | `30` | Days before a symlink expires and placeholder is restored |
| `STREAMS_PATH` | `/streams` | Root path for placeholder/symlink tree |
| `STRM_ROOT` | `/streams` | Root path for NFO files and placeholder tree (usually same as `STREAMS_PATH`) |
| `DEBRIDARR_BASE_URL` | `http://debridarr:7474` | Public base URL of this container (used in proxy URLs) |
| `PLACEHOLDER_EXT` | `.mp4` | Extension for placeholder files (`.mp4` recommended) |
| `API_KEY` | _(empty)_ | If set, all `/api/*` endpoints require this key |
| `PORT` | `7474` | HTTP port |

## Features

### Pre-symlink
After every library sync, Debridarr scans all library items against the Decypharr index and symlinks anything that's already in AllDebrid. This makes future plays instant even without a Tautulli event.

### Pinned items
Pin a library item to keep it permanently available. Pinned items are re-added to AllDebrid and their search is re-triggered automatically when their symlink expires, instead of reverting to a placeholder.

### Spot check
The **Spot Check** button (Library tab) fires a search for every un-linked library item, waits 30 seconds for imports, then symlinks anything that resolved. Items with no release found after search are added to a **Not Found** list and skipped on future syncs (you can clear them individually or all at once).

### Not Found list
Items confirmed to have no available release are tracked in the Not Found list. They are skipped during library syncs to avoid repeated searches. Clear individual items or the whole list from the Library tab.

### Episode pre-caching
After a play event, Debridarr pre-activates the next 3 episodes of the same series so binge-watching is seamless.

### NFO metadata
Debridarr writes Kodi-format NFO files (`tvshow.nfo`, `movie.nfo`, episode NFOs) alongside every placeholder using data from Sonarr/Radarr. This lets Plex match metadata instantly via the NFO Agent without hitting external APIs.

## Built-in Torznab indexer (AllDebrid cached)

Debridarr exposes a Torznab indexer that searches The Pirate Bay and returns **only releases AllDebrid already has cached**, so Sonarr/Radarr only ever grab instantly-available torrents.

Configure it in **Connections → Indexer (AllDebrid cached)**:
- **AllDebrid API Key** — from https://alldebrid.com/apikeys
- **Torznab API Key** — any secret string; Prowlarr uses it to authenticate
- **Only return cached releases** — on by default

Then add it to **Prowlarr → Indexers → Add → Generic Torznab**:
- URL: `http://<debridarr-host>:7474/torznab`
- API Key: your Torznab API Key

Prowlarr syncs it to Sonarr/Radarr automatically. You can also add it directly in Sonarr/Radarr (Settings → Indexers → Torznab).

Endpoints: `t=caps`, `t=search`, `t=tvsearch` (`season`, `ep`), `t=movie`.

## API reference

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `GET` | `/api/settings` | Read all settings |
| `PATCH` | `/api/settings` | Update settings |
| `GET` | `/api/setup/status` | Wizard step completion status |
| `GET` | `/api/library` | All library items |
| `POST` | `/api/library/sync` | Re-scan all configured Sonarr + Radarr instances |
| `POST` | `/api/library/add` | Add a single series or movie to the library |
| `POST` | `/api/library/{id}/activate` | Manually activate a stream |
| `POST` | `/api/library/{arr_id}/reset-placeholders` | Restore `.mp4` placeholders for a series (optional `?season=N`) |
| `POST` | `/api/library/restore-placeholders` | Delete all symlinks for a section and restore placeholders (`?media_type=movie\|tv\|all`) |
| `POST` | `/api/library/rebuild-placeholders` | Re-write all placeholder files and fix stale paths (`?arr=all`) |
| `GET` | `/api/streams` | List streams (`?status=active\|all`) |
| `POST` | `/api/streams/manual` | Manually create a stream from a file path |
| `POST` | `/api/streams/cleanup` | Expire all streams past TTL |
| `DELETE` | `/api/streams/{id}` | Expire a stream and restore placeholder |
| `GET` | `/api/pinned` | List pinned items |
| `POST` | `/api/pinned/{library_id}` | Pin a library item |
| `DELETE` | `/api/pinned/{library_id}` | Unpin a library item |
| `DELETE` | `/api/pinned` | Clear all pins |
| `POST` | `/api/spot-check` | Run spot check in background (`?limit=N`) |
| `GET` | `/api/spot-check/status` | Check if spot check is running |
| `GET` | `/api/not-found` | List not-found items |
| `DELETE` | `/api/not-found/{id}` | Remove an item from the not-found list |
| `DELETE` | `/api/not-found` | Clear entire not-found list |
| `POST` | `/webhook/tautulli` | Tautulli playback webhook |
| `POST` | `/webhook/arr` | Generic arr import webhook (auto-routes to Sonarr or Radarr) |
| `POST` | `/webhook/sonarr` | Sonarr import webhook (default instance) |
| `POST` | `/webhook/radarr` | Radarr import webhook (default instance) |
| `POST` | `/webhook/sonarr/{key}` | Per-instance Sonarr import webhook |
| `POST` | `/webhook/radarr/{key}` | Per-instance Radarr import webhook |
| `GET` | `/api/arr/sonarr/status` | Test default Sonarr connection |
| `GET` | `/api/arr/radarr/status` | Test default Radarr connection |
| `GET` | `/api/arr/sonarr/series` | List all series from Sonarr with episode counts |
| `GET` | `/api/arr/radarr/movies` | List all movies from Radarr |
| `GET` | `/api/tautulli/status` | Test Tautulli connection |
| `GET` | `/api/plex/test` | Test Plex connection |
| `GET` | `/api/instances` | List all Sonarr/Radarr instances (API keys masked) |
| `POST` | `/api/instances` | Add an instance (`name`, `type`, `url`, `api_key`) |
| `PATCH` | `/api/instances/{key}` | Update an instance (rename, change URL/credentials, enable/disable) |
| `DELETE` | `/api/instances/{key}` | Remove an instance (`?purge=true` also deletes library rows and `/streams/<key>` tree) |
| `GET` | `/api/instances/{key}/status` | Test a specific instance's connection |
| `GET` | `/api/decypharr/status` | Decypharr connection status + torrent count |
| `POST` | `/api/decypharr/reload` | Reload full Decypharr torrent index |
| `POST` | `/api/decypharr/reload/new` | Incremental reload (new torrents only) |
| `GET` | `/api/decypharr/test` | Test Decypharr connection |
| `GET` | `/api/decypharr/debug` | Raw Decypharr API responses for debugging |
| `GET` | `/api/fuse` | DFS FUSE mount status + registered torrents |
| `GET` | `/api/debug/placeholders` | Diagnose missing/broken placeholders |
| `POST` | `/api/debug/placeholders/repair` | Re-write all broken placeholders |
| `POST` | `/api/webhook/test` | Fire a synthetic play event for testing |
| `GET` | `/torznab` | Torznab indexer endpoint |
| `GET` | `/torznab/api` | Torznab indexer endpoint (alternate path) |
| `GET` | `/api/indexer/search` | JSON search via Torznab indexer (for UI test button) |
