# Debridarr

Stream-on-demand bridge between AllDebrid (via DFS FUSE), Sonarr, Radarr, and Tautulli. Placeholders appear in Plex for every monitored item. When you press Play, Debridarr finds the real file — from AllDebrid's FUSE mount or Sonarr/Radarr's disk — and symlinks it instantly.

## Multiple instances

Debridarr supports any number of Sonarr and Radarr instances, each with a name you choose (for example a 4K Sonarr/Radarr pair alongside your 1080p ones). Out of the box four instances are seeded: `sonarr`, `radarr`, `sonarr4k`, and `radarr4k` — the 4K pair starts blank, ready to configure in the setup wizard or the Connections tab.

Each instance has an immutable **key** (a slug derived from its name on creation) and a free-form **name** you can rename at any time. The key anchors two things: the library rows for that instance and its on-disk placeholder tree under `/streams/<key>/...`. Point a separate Plex library at each instance's subfolder so a 1080p and a 4K copy of the same show never collide.

## How it works

```
Startup
  └─ Scan Sonarr + Radarr
       └─ Create placeholder .mp4 for every monitored item
            └─ Stored under /streams/sonarr/... and /streams/radarr/...
               AllDebrid torrents registered in the DFS FUSE filesystem
               at /docker/debridarr/mnt-alldebrid/<torrent>/<file>

User presses Play in Plex
  └─ Tautulli fires Playback Start webhook → POST /webhook/tautulli
       └─ Debridarr matches title + season/episode to library
            └─ 1. Check FUSE registry (instant if already in AllDebrid)
               2. Check Sonarr/Radarr file on disk
               3. Trigger fresh arr search in background (always)
            └─ Symlink: placeholder path → real file
                 └─ Plex re-reads symlink and plays the real file

30 days later
  └─ Symlink expires, placeholder restored, arr notified
```

## Quick start

```bash
cp .env.example .env
docker compose up -d
```

Open **http://localhost:7474** and follow the setup wizard to configure AllDebrid, Sonarr, Radarr, and Tautulli.

## Prerequisites

The DFS FUSE mount needs these on the **host**:

```bash
apt install fuse
echo "user_allow_other" >> /etc/fuse.conf
```

And in `docker-compose.yml` (already included):

```yaml
devices:
  - /dev/fuse:/dev/fuse:rwm
cap_add:
  - SYS_ADMIN
security_opt:
  - apparmor:unconfined
volumes:
  - /docker/debridarr:/docker/debridarr:rshared
```

The `rshared` volume mount on the full `/docker/debridarr` path is what makes the FUSE mount visible on the host at `/docker/debridarr/mnt-alldebrid`.

## Plex library setup

Add one Plex library per instance, each pointing at that instance's subfolder under the streams root:

| Library type | Path | Instance |
|---|---|---|
| TV Shows | `/streams/sonarr` | `sonarr` (1080p) |
| Movies | `/streams/radarr` | `radarr` (1080p) |
| TV Shows | `/streams/sonarr4k` | `sonarr4k` |
| Movies | `/streams/radarr4k` | `radarr4k` |

Any custom-named instance lives at `/streams/<its-key>`. The key is shown next to each instance in the Connections tab.

For faster metadata scanning, set each library's Agent to **Plex NFO Series** / **Plex NFO Movie** (requires Plex Media Server ≥ 1.43.1).

## Tautulli webhook setup

1. Tautulli → Settings → Notification Agents → Add → **Webhook**
2. **Webhook URL:** `http://<debridarr-host>:7474/webhook/tautulli`
3. **Trigger:** Playback Start only
4. **JSON body** (Data tab):

```json
{"media_type":"{media_type}","title":"{title}","grandparent_title":"{grandparent_title}","year":"{year}","grandparent_year":"{grandparent_year}","parent_media_index":"{parent_media_index}","media_index":"{media_index}","rating_key":"{rating_key}","themoviedb_id":"{themoviedb_id}","thetvdb_id":"{thetvdb_id}","imdb_id":"{imdb_id}"}
```

> All settings and credentials can be set in the web UI — leave env vars blank and configure via the wizard instead.

## Volumes

| Path | Purpose |
|---|---|
| `./data` | SQLite database (settings, library, streams) |
| `./streams` | Placeholder + symlink tree served to Plex |
| `./cache` | DFS chunk cache for AllDebrid streaming |
| `/docker/debridarr` | Host path shared with container (`rshared`) — FUSE mount appears here |

## Environment variables

Leave these blank to configure via the UI. The `SONARR_*`/`RADARR_*` vars only seed the **default** `sonarr` and `radarr` instances on first run; all other instances (including the 4K pair) live in the database and are managed from the wizard or Connections tab. Set an env var only if you want it to permanently override that instance's database value.

When a Sonarr/Radarr instance imports a file, point its webhook (Settings → Connect → Webhook, triggers: On Import, On Upgrade, On File Delete) at the matching per-instance URL — e.g. `http://<debridarr-host>:7474/webhook/sonarr/sonarr4k` — so imports map to the right library tree. The bare `/webhook/sonarr` and `/webhook/radarr` URLs still work and route to the default instances.

| Variable | Default | Description |
|---|---|---|
| `SONARR_URL` | — | e.g. `http://192.168.1.x:8989` |
| `SONARR_API_KEY` | — | Sonarr → Settings → General → API Key |
| `RADARR_URL` | — | e.g. `http://192.168.1.x:7878` |
| `RADARR_API_KEY` | — | Radarr → Settings → General → API Key |
| `TAUTULLI_URL` | — | e.g. `http://192.168.1.x:8181` |
| `TAUTULLI_API_KEY` | — | Tautulli → Settings → Web Interface → API key |
| `ALLDEBRID_API_KEY` | — | [alldebrid.com/apikeys](https://alldebrid.com/apikeys) |
| `DFS_FUSE_MOUNT` | `/docker/debridarr/mnt-alldebrid` | AllDebrid FUSE mount point |
| `DFS_CACHE_DIR` | `/docker/debridarr/cache/dfs` | Chunk cache directory |
| `DFS_CACHE_SIZE_GB` | `50` | Max chunk cache size in GB |
| `DFS_CACHE_TTL_H` | `12` | Hours before cached chunks expire |
| `DFS_CHUNK_MB` | `32` | Download chunk size in MB |
| `DFS_READ_AHEAD_MB` | `128` | Read-ahead prefetch in MB |
| `SYMLINK_TTL_DAYS` | `30` | Days before symlink expires and placeholder is restored |
| `STREAMS_PATH` | `/streams` | Root path for placeholder/symlink tree |
| `API_KEY` | _(empty)_ | If set, all `/api/*` endpoints require this key |
| `PORT` | `7474` | HTTP port |

## API reference

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `GET` | `/api/settings` | Read all settings |
| `PATCH` | `/api/settings` | Update settings |
| `GET` | `/api/library` | All library items |
| `POST` | `/api/library/sync` | Re-scan Sonarr + Radarr |
| `POST` | `/api/library/{id}/activate` | Manually activate a stream |
| `GET` | `/api/streams` | List streams (`?status=active\|all`) |
| `DELETE` | `/api/streams/{id}` | Expire a stream and restore placeholder |
| `POST` | `/api/streams/cleanup` | Expire all streams past TTL |
| `POST` | `/webhook/tautulli` | Tautulli playback webhook |
| `GET` | `/api/arr/sonarr/status` | Test Sonarr connection |
| `GET` | `/api/arr/radarr/status` | Test Radarr connection |
| `GET` | `/api/instances` | List all Sonarr/Radarr instances (api keys masked) |
| `POST` | `/api/instances` | Add an instance (`name`, `type`, `url`, `api_key`); key is derived from the name |
| `PATCH` | `/api/instances/{key}` | Update an instance (rename, change URL/key, enable/disable) |
| `DELETE` | `/api/instances/{key}?purge=true` | Remove an instance (and optionally its library rows + `/streams/{key}` tree) |
| `GET` | `/api/instances/{key}/status` | Test a specific instance's connection |
| `POST` | `/webhook/sonarr/{key}` | Per-instance Sonarr import webhook |
| `POST` | `/webhook/radarr/{key}` | Per-instance Radarr import webhook |
| `GET` | `/api/tautulli/status` | Test Tautulli connection |
| `GET` | `/api/fuse` | DFS FUSE mount status + registered torrents |
| `POST` | `/api/fuse/reload` | Re-fetch AllDebrid torrents without restart |
| `GET` | `/api/alldebrid/status` | Test AllDebrid connection |
| `GET` | `/api/debug/placeholders` | Diagnose missing/broken placeholders |
| `POST` | `/api/debug/placeholders/repair` | Re-write all broken placeholders |
| `POST` | `/api/webhook/test` | Fire a synthetic play event for testing |
