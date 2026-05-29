# Debridarr

On-the-fly AllDebrid streaming for Plex. When you play something in Plex, Debridarr searches for a cached torrent on AllDebrid, mounts it via FUSE, and symlinks it into your Plex library — no downloading, no storage required.

---

## How It Works

```
Plex play → Tautulli webhook → Debridarr
  → Search TPB for cached torrent on AllDebrid
  → Add magnet to AllDebrid
  → Mount file in FUSE at /docker/debridarr/mnt-alldebrid/<torrent>/<file.mkv>
  → Symlink plex-strm/movies|tv/... → FUSE file
  → Plex streams bytes via HTTP Range from AllDebrid CDN
```

Placeholder `.mp4` files (140 bytes) are created in your Plex library so Plex can scan and index content before it's played. On first play, the placeholder is replaced by a symlink to the real stream.

---

## Requirements

- Docker + Docker Compose
- AllDebrid account with API key
- Overseerr (for library browsing)
- Tautulli (to detect when something is played in Plex)
- Plex Media Server

---

## First-Time Setup

```
mkdir -p /tmp/dd && \
tar -xzf ~/debridarr_docker.tar.gz -C /tmp/dd && \
mkdir -p /docker/debridarr && \
cp -r /tmp/dd/docker/debridarr/. /docker/debridarr/ && \
rm -rf /tmp/dd

cd /docker/debridarr && \
mkdir -p data plex-strm mnt-alldebrid config/tautulli && \
docker-compose build --no-cache && \
docker-compose up -d && \
docker logs debridarr -f
```

Open `http://<server-ip>:7474` and complete the setup wizard.

---

## Update / Rebuild

```bash
cd /docker/debridarr
docker-compose down
docker rmi debridarr:latest --force 2>/dev/null || true

mkdir -p /tmp/dd && tar -xzf ~/debridarr_docker.tar.gz -C /tmp/dd
cp -r /tmp/dd/docker/debridarr/. /docker/debridarr/
rm -rf /tmp/dd

docker-compose build --no-cache
docker-compose up -d
docker logs debridarr -f
```

---

## Tautulli Setup

1. Settings → Notification Agents → Add → **Webhook**
2. **Webhook URL:** `http://debridarr:7474/webhook/tautulli`
3. **Method:** POST
4. **Triggers:** Playback Start only
5. **Data tab → Playback Start**, paste:

```json
{
  "media_type": "{media_type}",
  "title": "{title}",
  "grandparent_title": "{grandparent_title}",
  "year": "{year}",
  "grandparent_year": "{grandparent_year}",
  "parent_media_index": "{parent_media_index}",
  "media_index": "{media_index}",
  "rating_key": "{rating_key}",
  "imdb_id": "{imdb_id}",
  "themoviedb_id": "{themoviedb_id}",
  "thetvdb_id": "{thetvdb_id}"
}
```

---

## Plex Setup

Add two libraries pointing to the `plex-strm` folders:

| Library | Path |
|---------|------|
| Movies  | `/docker/debridarr/plex-strm/movies` |
| TV Shows | `/docker/debridarr/plex-strm/tv` |

Also add `/docker/debridarr/mnt-alldebrid` as a network path Plex can access (it must be able to follow symlinks into this folder).

---

## Directory Structure

```
/docker/debridarr/
├── app/                  # Application code
├── data/                 # Database + config (.env, .setup_complete)
├── plex-strm/
│   ├── movies/           # Movie placeholder + symlink files
│   └── tv/               # TV placeholder + symlink files
├── mnt-alldebrid/        # FUSE mount — AllDebrid torrents appear here
│   └── <torrent-name>/
│       └── <file.mkv>    # Virtual file, streamed from AllDebrid CDN
├── config/tautulli/      # Tautulli config
├── docker-compose.yml
├── Dockerfile
└── entrypoint.sh
```

---

## Web UI Pages

| Page | Description |
|------|-------------|
| `/library` | Browse Overseerr requests, add to Plex library |
| `/my-library` | View and manage added items |
| `/streams` | Active resolved streams |
| `/settings` | Configure AllDebrid, Overseerr, DFS cache |

---

## DFS Cache (Streaming Performance)

Debridarr pre-fetches chunks from AllDebrid CDN into a local disk cache so Plex never stalls waiting for network. Configurable in **Settings → DFS Cache**:

| Setting | Default | Description |
|---------|---------|-------------|
| Cache Directory | `/tmp/cache` | Where chunks are stored |
| Disk Cache Size | 30 GB | Maximum cache size |
| Cache Expiry | 24h | How long to keep chunks |
| Chunk Size | 8 MB | Fetch unit from CDN |
| Read Ahead | 500 MB | Pre-fetch buffer |
| Cleanup Interval | 20 min | How often to enforce limits |

---

## How Content Gets Into Plex

1. Go to `/library` in the Debridarr UI
2. Browse your Overseerr requests
3. Click **Add to Library** on a movie or show
4. Debridarr creates placeholder `.mp4` files — Plex scans and indexes them
5. Press play in Plex → Tautulli fires → Debridarr resolves the stream
6. The placeholder is replaced by a symlink → streaming starts

---

## Cache Expiry

Resolved streams are cached for 30 days (configurable via `CACHE_TTL_DAYS`). When a cache entry expires:
- The symlink is removed
- The placeholder `.mp4` is restored
- Next play will re-resolve the stream from AllDebrid

---

## Environment Variables

Set in `data/.env` via the Settings UI or manually:

| Variable | Description |
|----------|-------------|
| `ALLDEBRID_API_KEY` | Your AllDebrid API key |
| `OVERSEERR_URL` | Overseerr URL e.g. `http://192.168.1.x:5055` |
| `OVERSEERR_API_KEY` | Overseerr API key |
| `PLEX_ROOT` | Path to plex-strm inside container (default `/plex-strm`) |
| `FUSE_MOUNT` | FUSE mount path (default `/docker/debridarr/mnt-alldebrid`) |
| `FUSE_SYMLINK_ROOT` | Host path for symlink targets (default `/docker/debridarr/mnt-alldebrid`) |
| `CACHE_TTL_DAYS` | Days before stream cache expires (default `30`) |
| `LOG_LEVEL` | `INFO` or `DEBUG` |

---

## Ports

| Port | Service |
|------|---------|
| 7474 | Debridarr UI |
| 8181 | Tautulli |
