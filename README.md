# Debridarr

Stream-on-demand bridge between AllDebrid (via DFS FUSE), Sonarr, Radarr, and Tautulli. Placeholders appear in Plex for every monitored item. When you press Play, Debridarr finds the real file — from AllDebrid's FUSE mount or Sonarr/Radarr's disk — and symlinks it instantly.

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

Add two libraries in Plex pointing at the streams folder:

| Library type | Path |
|---|---|
| TV Shows | `/streams/sonarr` |
| Movies | `/streams/radarr` |

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

Leave these blank to configure via the UI. Only set them if you want env vars to permanently override the database.

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
| `GET` | `/api/tautulli/status` | Test Tautulli connection |
| `GET` | `/api/fuse` | DFS FUSE mount status + registered torrents |
| `POST` | `/api/fuse/reload` | Re-fetch AllDebrid torrents without restart |
| `GET` | `/api/alldebrid/status` | Test AllDebrid connection |
| `GET` | `/api/debug/placeholders` | Diagnose missing/broken placeholders |
| `POST` | `/api/debug/placeholders/repair` | Re-write all broken placeholders |
| `POST` | `/api/webhook/test` | Fire a synthetic play event for testing |
