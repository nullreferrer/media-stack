# media-stack

Self-hosted home media server. Requests content through Jellyseerr, downloads via qBittorrent over ExpressVPN, and serves it through Jellyfin to any device on your network.

```
Jellyseerr (request)
    ↓
Radarr / Sonarr (automation)
    ↓
Prowlarr (indexer — IPTorrents)
    ↓
gluetun → qBittorrent (download over VPN)
    ↓
Jellyfin (stream)
```

Files are hardlinked on import — no duplication, qBittorrent keeps seeding.

---

## Services

| Service | Port | Purpose |
|---|---|---|
| Jellyseerr | 5055 | Request interface |
| Jellyfin | 8096 | Media server |
| Radarr | 7878 | Movie automation |
| Sonarr | 8989 | TV automation |
| Prowlarr | 9696 | Indexer manager |
| qBittorrent | 8080 | Download client (via gluetun) |

---

## Prerequisites

- Docker and Docker Compose
- An [ExpressVPN](https://www.expressvpn.com) subscription (credentials from the dashboard, not your login)
- An [IPTorrents](https://iptorrents.com) account

---

## Setup

### 1. Clone and configure environment

```bash
git clone <repo-url> media-stack
cd media-stack
cp .env.example .env
```

Edit `.env`:

```env
PUID=1000        # output of: id -u
PGID=1000        # output of: id -g
TZ=Europe/London
DATA_PATH=./data # use an absolute path on Linux, e.g. /mnt/data

EXPRESSVPN_USERNAME=  # from expressvpn.com dashboard
EXPRESSVPN_PASSWORD=
EXPRESSVPN_SERVER=UK  # country name or code
```

### 2. Create data directories

```bash
mkdir -p data/{torrents/{movies,tv},media/{movies,tv},config/{gluetun,qbittorrent,jellyfin,jellyseerr,radarr,sonarr,prowlarr}}
```

### 3. Start the stack

```bash
docker compose up -d
```

### 4. Verify VPN

```bash
docker logs gluetun               # should show "Tunnel is up"
docker exec gluetun curl -s ifconfig.io  # should return a VPN IP, not yours
```

---

## Service Configuration

Work through these in order. Each service has a web UI at `http://localhost:<port>`.

### qBittorrent — `localhost:8080`

Default username is `admin`. The password is randomly generated on first run — retrieve it with:

```bash
docker logs qbittorrent 2>&1 | grep "temporary password"
```

Change it immediately after logging in.

1. **Settings → Downloads** — set Default Save Path to `/data/torrents`
2. **Settings → BitTorrent** — add two categories:
   - `movies` → Save path `/data/torrents/movies`
   - `tv` → Save path `/data/torrents/tv`

### Prowlarr — `localhost:9696`

1. **Indexers → Add Indexer** — search for `IPTorrents`, select the Torznab entry
2. IPTorrents requires a session cookie rather than an API key. To get it:
   - Log into IPTorrents in your browser
   - Open DevTools → Application → Cookies → `https://iptorrents.com`
   - Copy the value of the `iptorrents` cookie
   - Paste it into the **Cookie** field in the Prowlarr indexer config
   - Still in DevTools, open the **Network** tab, reload the page, click any request to `iptorrents.com`, and copy the `User-Agent` value from the request headers
   - Paste it into the **User Agent** field in the same Prowlarr indexer config
3. Test and save
4. **Settings → Apps → Add Application** — add Radarr:
   - Prowlarr server: `http://prowlarr:9696`
   - Radarr server: `http://radarr:7878`
   - API key: copy from Radarr → Settings → General
5. Repeat for Sonarr (`http://sonarr:8989`)

Prowlarr will sync the indexer to both apps automatically.

### Radarr — `localhost:7878`

1. **Settings → Media Management**
   - Enable "Use Hardlinks instead of Copy"
   - Movie folder format: `{Movie Title} ({Release Year})`
   - Movie file format: `{Movie Title} ({Release Year}) {Quality Full}`
2. **Settings → Download Clients → Add** — qBittorrent:
   - Host: `gluetun` (not `qbittorrent` — it shares gluetun's network)
   - Port: `8080`
   - Category: `movies`
3. **Settings → Root Folders → Add** — `/data/media/movies`

### Sonarr — `localhost:8989`

1. **Settings → Media Management**
   - Enable "Use Hardlinks instead of Copy"
   - Series folder format: `{Series Title}`
   - Season folder format: `Season {season:00}`
   - Episode file format: `{Series Title} - S{season:00}E{episode:00} - {Episode Title} {Quality Full}`
2. **Settings → Download Clients → Add** — qBittorrent:
   - Host: `gluetun`, Port: `8080`
   - Category: `tv`
3. **Settings → Root Folders → Add** — `/data/media/tv`

### Jellyfin — `localhost:8096`

Complete the setup wizard on first launch.

1. **Dashboard → Libraries → Add Media Library**
   - Add **Movies** → folder `/data/media/movies`
   - Add **TV Shows** → folder `/data/media/tv`
2. Note your Jellyfin URL and create an API key under **Dashboard → API Keys** — needed for Jellyseerr.

### Jellyseerr — `localhost:5055`

1. Sign in with your Jellyfin account (URL: `http://jellyfin:8096`)
2. Follow the setup wizard to connect Radarr and Sonarr:
   - Use the internal hostnames (`http://radarr:7878`, `http://sonarr:8989`)
   - Paste the API keys from each service's Settings → General page

---

## Deploying to a New Machine

1. Check out the repo
2. Copy `.env.example` to `.env` and fill in values — update `DATA_PATH` to the correct mount point
3. Run `mkdir -p ...` (step 2 above) to create the data directories
4. `docker compose up -d`

Service configuration (Prowlarr, Radarr, Sonarr, etc.) is persisted in `DATA_PATH/config` — if you're migrating an existing install, copy that directory across and skip the configuration steps above.

---

## Notes

- **ExpressVPN does not support port forwarding.** Seeding works via outbound connections but ratio efficiency is lower. Switching to Mullvad or AirVPN is a single env var change in `.env` if this becomes a problem.
- All Servarr services communicate over Docker's internal network using container names as hostnames. Never use `localhost` when configuring inter-service connections.
