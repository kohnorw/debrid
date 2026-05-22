# AllDebrid Indexer

A self-hosted torrent search web UI + Torznab indexer backed by The Pirate Bay, with AllDebrid cache checking. Add it to Prowlarr and it syncs automatically to Sonarr and Radarr.

---

## What it does

- Searches The Pirate Bay for any movie or TV show
- Checks AllDebrid to see which torrents are instantly available (cached)
- Exposes a Torznab API endpoint that Prowlarr, Sonarr, and Radarr can talk to
- Web UI for manual searching and adding torrents directly to AllDebrid

---

## Requirements

- Docker
- An AllDebrid account and API key → https://alldebrid.com/apikeys

---

## Setup

### 1. Extract and configure

```bash
tar -xzf alldebrid-web.tar.gz
cd alldebrid-web
```

Edit `docker-compose.yml` and set your keys:

```yaml
environment:
  - ALLDEBRID_API_KEY=your_alldebrid_api_key_here
  - TORZNAB_APIKEY=pick_any_secret_string
```

`TORZNAB_APIKEY` is a password you make up — Prowlarr will use it to authenticate.

### 2. Build and run

```bash
# Modern Docker
docker compose up --build -d

# Older Docker
docker-compose up --build -d

# Without Compose
docker build -t alldebrid-web .
docker run -d -p 5000:5000 \
  -e ALLDEBRID_API_KEY=your_key \
  -e TORZNAB_APIKEY=changeme \
  --restart unless-stopped \
  alldebrid-web
```

### 3. Open the web UI

```
http://localhost:5000
```

---

## Adding to Prowlarr

1. Go to **Prowlarr → Indexers → Add Indexer**
2. Search for **Torznab** and select **Generic Torznab**
3. Fill in:
   - **Name**: AllDebrid Indexer (or anything you like)
   - **URL**: `http://<your-docker-host-ip>:5000/torznab`
   - **API Key**: whatever you set as `TORZNAB_APIKEY`
4. Click **Test** — should return green
5. Click **Save**
6. Prowlarr will sync it to Sonarr and Radarr automatically via your app profiles

> If your Prowlarr runs in Docker too, use the host's LAN IP instead of `localhost`.

---

## Adding directly to Sonarr / Radarr (without Prowlarr)

**Settings → Indexers → Add → Torznab**

| Field   | Value                                      |
|---------|--------------------------------------------|
| Name    | AllDebrid Indexer                          |
| URL     | `http://<host>:5000/torznab`               |
| API Key | your `TORZNAB_APIKEY`                      |
| Categories | 5000 (TV) for Sonarr, 2000 (Movies) for Radarr |

---

## Torznab API endpoints

| URL | Description |
|-----|-------------|
| `/torznab?t=caps&apikey=...` | Capabilities — Prowlarr calls this on Test |
| `/torznab?t=search&q=...&apikey=...` | General search |
| `/torznab?t=tvsearch&q=...&season=1&ep=5&apikey=...` | TV search with season/episode |
| `/torznab?t=movie&q=...&apikey=...` | Movie search |

---

## Web UI search examples

| Query | What it does |
|-------|-------------|
| `Breaking Bad` | All results for Breaking Bad |
| `Breaking Bad S03` | Season 3 only |
| `Breaking Bad S03E05` | Specific episode |
| `The Bear Season 2` | Converts to S02 automatically |
| `Oppenheimer 2023` | Search by title and year |
| `Vanderpump Villa S01 1080p` | Season 1, filtered to 1080p |
| `It's Always Sunny in Philadelphia S10` | Apostrophes handled automatically |

---

## Sort options (web UI)

After searching, use the pills to sort results:

- **✅ Cached first** — AllDebrid instant downloads at the top (default)
- **❌ Not cached first** — torrents not yet on AllDebrid
- **🌱 Most seeded** — best availability regardless of cache
- **📺 Episode (high→low)** — sorted by season and episode number

---

## Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ALLDEBRID_API_KEY` | Yes | Your AllDebrid API key |
| `TORZNAB_APIKEY` | Yes | Password for the Torznab endpoint |
| `PORT` | No | Port to run on (default: `5000`) |

---

## How cache checking works

1. Torrents are fetched from The Pirate Bay
2. All hashes are uploaded to AllDebrid to check if they're instantly cached
3. AllDebrid IDs are immediately deleted — nothing is added to your account
4. Results marked ✅ are available instantly; ❌ means AllDebrid would need to download it
5. When you click **Add** (web UI) or Sonarr/Radarr grabs a release, it's then added for real

---

## Troubleshooting

**No results in Prowlarr test**
Prowlarr's Test button only checks capabilities (`t=caps`). Use the Search button on a specific show in Sonarr/Radarr for a real test.

**"File is not a valid torrent" error**
This was a known bug with how magnets were encoded. Make sure you're on the latest build.

**Apostrophes in show names (e.g. It's Always Sunny)**
Handled automatically — special characters are stripped before querying.

**Container can't reach AllDebrid or apibay**
Check your Docker network settings. The container needs outbound internet access on port 443.

**Prowlarr is in Docker too**
Use your host machine's LAN IP (e.g. `192.168.1.x`) instead of `localhost` in the indexer URL.
