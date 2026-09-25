# The Home Lab & Dev Journey — Mohamed Zaeem Motala

I'm Mohamed Zaeem Motala, currently at Emeris Sandton finishing my Higher Certificate in Mobile Application and Web Development — distinctions across the board so far, working ahead of the class rather than behind it.

The lab started for a simple reason: I wanted to stop paying for subscriptions. Once I started digging into what was actually possible to self-host, it turned into something bigger than that — a way to find out how much of my digital life I could run myself instead of renting it.

It started on a Core 2 Duo with 8GB RAM. It died within months. I let it sit for a while, then rebuilt on a Ryzen 5500 with 16GB RAM and 4TB storage — better hardware, but early on I lost the Nextcloud encryption key and everything with it. No backups yet at that point. It sat dead for another stretch after that.

The current build is Rockstor (openSUSE) on that same Ryzen box, running around 20 Docker containers: Immich, Jellyfin, Frigate (NVR), Nextcloud, the *arr stack (Sonarr, Radarr, Prowlarr), qBittorrent, Pi-hole, Ollama, Kavita, Bookshelf, Glances, Homepage, Dozzle, ntfy, Gotify, FlareSolverr, ClamAV, and Cloudflared. Everything is Tailscale-only — no ports forwarded on the router.

I had zero experience going in. I used AI throughout, and I'm not going to pretend otherwise — but it wasn't copy-paste. When a command didn't make sense I asked why before running it. When something broke I made my own attempt first, and only asked for guidance after actually trying. AI explaining something clearly isn't cheating at learning it — it's how I learned it. Hours went into this, not minutes.

## Outages and infrastructure failures

A power outage once took the whole stack down through two compounding failures at once: the TP-Link Deco mesh's bridging between wired and WiFi clients corrupted, which meant the NAS could reach the router and the internet but no WiFi device could see it — fixed with a full cold power-cycle of every mesh node, starting with the main unit. Separately, Rockstor's immutable flag tripped on `/mnt2/home` and blocked Docker from starting at all, a repeat of a prior failure. I cleared it with `chattr -i`, then fixed the root cause permanently by adding a systemd drop-in that auto-clears the flag before Docker starts on every future boot, so the same outage can't take the system down the same way twice. On top of that, Tailscale had silently overwritten DNS resolution via MagicDNS, breaking name lookups until I diagnosed and disabled it.

## RAM mystery

The server was showing only 7.6GB of RAM instead of the expected 16GB. I diagnosed it methodically — `free -h`, `/proc/meminfo`, load tests — before physically opening the case and finding a loose RAM stick, probably from earlier hardware work. Reseated it, and containers that had been RAM-starved (qBittorrent alone dropped from 1.35GB to ~99MB usage) suddenly had headroom again.

## The *arr stack and Bookshelf integration

Adding book automation (Bookshelf + Kavita) on top of the existing Sonarr/Radarr/Prowlarr/qBittorrent stack surfaced a run of real bugs: a Docker image tag that didn't actually exist, a port collision between two services defaulting to the same port, file permission errors blocking writes to the app's own data folder, and — hardest to pin down — a qBittorrent authentication failure that turned out to be a confirmed compatibility bug in a specific qBittorrent version, fixed by pinning to an older release. I also caught a missing volume mount that left one app blind to files another app had already downloaded.

## Jellyfin

Multiple separate issues over time: images silently failing to load in the web UI, transcoding and hardware acceleration settings to work through, and a persistent annoyance where already-watched shows kept reappearing in "Recently Added" — traced to Jellyfin sorting by file date instead of scan date, fixed by changing the library's date-added behavior in settings.

## CSS (separate project, same habits)

Separate from the server, building a portfolio page for university: a div stretching full width from default block behavior, boxes overlapping despite `gap` being set correctly because of a stray leftover `margin-top: -300px` from earlier experimentation nobody had cleaned up, and a two-column layout with stacked boxes in the right column that only worked once I nested flexbox two layers deep — an outer horizontal flex container with a vertical one inside it. Small thing conceptually. Took real time to actually see why it wasn't working.

## Where it stands

Now working through OverTheWire Bandit, with TryHackMe next. I'm not lazy, and I'm not done.

All told — counting the dead builds, the time it sat untouched, and the rebuild to where it is now — this project has taken three years from start to today. It's not a weekend project or a tutorial I followed once. It's proof of the struggle and the exhilaration both — the dead hardware, the lost data, the outages, the hours of not understanding and then understanding. That's the actual story behind the containers running today.
