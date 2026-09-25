# Services

All containers run on this host and are only reachable over Tailscale — nothing is exposed to the public internet.

## Managed via docker-compose

**immich** — photo backup and management
- immich_server, immich_machine_learning, immich_redis, immich_postgres

**nas-dashboard**
- homepage — dashboard
- dozzle — live container log viewer

**books-stack**
- bookshelf — book library
- kavita — comic/ebook reader

## Standalone (docker run)

| Service | Image | Purpose |
|---|---|---|
| jellyfin | jellyfin/jellyfin:10.11.8 | Media server |
| radarr | linuxserver/radarr | Movie management |
| sonarr | linuxserver/sonarr | TV management |
| prowlarr | linuxserver/prowlarr | Indexer management |
| qbittorrent | linuxserver/qbittorrent:4.6.4 | Torrent client |
| flaresolverr | flaresolverr/flaresolverr | Cloudflare bypass for indexers |
| frigate | blakeblackshear/frigate | NVR / CCTV with object detection |
| ollama | ollama/ollama | Local LLM runner |
| nextcloud | nextcloud | File sync/storage |
| clamav | clamav/clamav | Antivirus scanning |
| pihole | pihole/pihole | Network-wide DNS/ad-blocking |
| glances | nicolargo/glances | System resource monitor |
| ntfy | binwiederhier/ntfy | Push notifications |
| gotify | gotify/server | Push notifications (secondary) |
| cloudflared | cloudflare/cloudflared | Cloudflare tunnel client |

*Planned: migrate the standalone containers to compose files for reproducibility.*

*Note: `gotify` and `cloudflared` weren't visible in a live `docker ps` snapshot taken during this documentation session — may not have been running at that moment. Confirm before relying on this list, and remove either here if they've genuinely been retired.*
