# Debridarr

Stream-on-demand bridge between All-Debrid (via FUSE mount), Sonarr, Radarr, and Tautulli.

## How it works

```
Startup
  └─ Scan Sonarr + Radarr
       └─ Create empty .mp4 placeholder for every monitored item
              (placed under /streams/sonarr/... and /streams/radarr/...)

User presses Play (on placeholder in Plex/Jellyfin)
  └─ Tautulli fires Playback Start webhook → POST /webhook/tautulli
       └─ Debridarr looks up the library item
            └─ Asks Sonarr/Radarr for the real file path on the FUSE mount
                 └─ Replaces placeholder with a symlink → real file
                      └─ Notifies Sonarr/Radarr to rescan
                           └─ Plex/Jellyfin re-reads and plays the real file

30 days later
  └─ Cleanup job fires (on startup + POST /api/streams/cleanup)
       └─ Symlink removed, empty placeholder restored
            └─ Sonarr/Radarr notified (rescan → file marked missing)
```

## Quick start

```bash
cp .env.example .env
# Fill in SONARR_URL, SONARR_API_KEY, RADARR_URL, RADARR_API_KEY
docker compose up -d
```

Open http://localhost:7474 to configure connections and watch streams.

## Tautulli webhook setup

1. Tautulli → Settings → Notification Agents → Add → Webhook
2. Webhook URL: `http://debridarr:7474/webhook/tautulli`
3. Trigger: **Playback Start**
4. JSON body (Data tab → JSON):

```json
{
  "event":               "{action}",
  "media_type":          "{media_type}",
  "file":                "{file}",
  "grandparent_title":   "{grandparent_title}",
  "parent_media_index":  {parent_media_index},
  "media_index":         {media_index},
  "title":               "{title}",
  "year":                {year}
}
```

> **Note:** Plex will show a brief "playback failed" or spinning state while Debridarr swaps the placeholder.
> For best results use Plex's "re-match" or have clients retry — the symlink is in place within ~1 second.

## Volumes

| Path         | Purpose |
|---|---|
| `/data`      | SQLite database |
| `/streams`   | Placeholder + symlink tree served to Plex/Jellyfin |
| `/mnt/debrid`| FUSE mount (zurg, rclone, etc.) — must propagate into container with `rshared` |

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `SONARR_URL` | — | e.g. `http://sonarr:8989` |
| `SONARR_API_KEY` | — | Sonarr API key |
| `RADARR_URL` | — | e.g. `http://radarr:7878` |
| `RADARR_API_KEY` | — | Radarr API key |
| `TAUTULLI_URL` | — | Optional |
| `SYMLINK_TTL_DAYS` | `30` | Days before symlink expires |
| `PLACEHOLDER_EXT` | `.mp4` | Extension for placeholder files |
| `STREAMS_PATH` | `/streams` | Root for placeholders/symlinks |
| `FUSE_MOUNT_PATH` | `/mnt/debrid` | FUSE mount root |
| `API_KEY` | _(empty)_ | If set, all `/api/*` endpoints require this key |

## API reference

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/library` | All library items from Sonarr/Radarr |
| `POST` | `/api/library/sync` | Re-scan Sonarr+Radarr in background |
| `POST` | `/api/library/{id}/activate` | Manually activate a stream |
| `GET` | `/api/streams` | List streams (`?status=active\|all`) |
| `POST` | `/api/streams/manual` | Manually create a stream |
| `DELETE` | `/api/streams/{id}` | Expire and revert a stream |
| `POST` | `/api/streams/cleanup` | Expire all streams past TTL |
| `POST` | `/webhook/tautulli` | Tautulli playback webhook |
| `GET` | `/api/arr/sonarr/status` | Test Sonarr connection |
| `GET` | `/api/arr/radarr/status` | Test Radarr connection |
| `GET` | `/api/fuse` | FUSE mount status |
| `GET` | `/api/settings` | Read settings |
| `PATCH` | `/api/settings` | Update settings |
