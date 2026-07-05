# Watchtower — Automatic Container Image Updates

[Watchtower](https://containrrr.dev/watchtower/) watches the running containers,
checks their image registries on a schedule, and pulls + recreates containers when a
newer image is published for the tag they track.

## Update Policy

**Auto-update everything, except two deliberate exceptions.**

| Service | Behaviour | Why |
|---------|-----------|-----|
| All *arr apps, Jellyfin, qBittorrent, AdGuard, Homarr, Dashdot, gluetun, cloudflared, restic, Watchtower itself | **Auto-update** daily | Low-risk, easily rolled back; benefit most from staying current |
| **Immich** (`immich_server`, `immich_machine_learning`) | **Monitor-only** (notify, manual update) | Immich ships breaking DB migrations between releases — read the release notes first |
| **Traefik** | **Monitor-only** (notify, manual update) | Pinned to `v3.6`; the ingress must not be auto-recreated on a patch re-tag |
| Immich `immich_postgres`, `immich_redis` | Never touched | Images are **digest-pinned** (`@sha256:…`), so Watchtower ignores them regardless |

Runs **daily at 06:00** (`WATCHTOWER_SCHEDULE=0 0 6 * * *`), prunes old images
(`WATCHTOWER_CLEANUP`), and sends **one consolidated ntfy report per run**.

> **No "wait until stable" feature.** Watchtower has no soak/delay — it updates the
> instant a tracked tag moves. The two riskiest services (Immich, Traefik) are exactly
> the monitor-only exceptions, so their updates always stay in your hands. If you ever
> want a hard stop on unattended updates across the board, set
> `WATCHTOWER_MONITOR_ONLY=true` in `watchtower/compose.yaml` — then *nothing*
> auto-updates and every service is notify-only.

## How a service opts out of auto-update

Add this label to the service's `labels:` block:

```yaml
      - "com.centurylinklabs.watchtower.monitor-only=true"
```

Currently applied to `traefik` (`traefik/compose.yaml`), `immich-server` and
`immich-machine-learning` (`immich/compose.yaml`). Add it to any future service you
want to update by hand.

## Notifications (ntfy)

1. Pick a **long, random topic name** (public `ntfy.sh` topics are unauthenticated —
   anyone who guesses the topic can read your notifications).
2. Set it in `.env`:
   ```bash
   WATCHTOWER_NOTIFICATION_URL=ntfy://ntfy.sh/<your-secret-topic>
   ```
   Self-hosted ntfy with auth: `ntfy://<user>:<pass>@<your-ntfy-host>/<topic>`.
   Format reference: <https://containrrr.dev/shoutrrr/services/ntfy/>
3. Subscribe to the topic in the [ntfy app](https://ntfy.sh/app) or via
   `curl -s ntfy.sh/<your-secret-topic>/json`.

## Manual update workflow (for the monitor-only services)

Run from the repo root. When a notification says an update is available:

**Immich** — read the [release notes](https://github.com/immich-app/immich/releases)
and any migration warnings first, then bump the version and restart:
```bash
# Edit IMMICH_VERSION in immich/.env to the new release tag
docker compose pull immich-server immich-machine-learning
docker compose up -d immich-server immich-machine-learning
```

**Traefik** — only when intentionally moving off `v3.6`:
```bash
# Edit the image tag in traefik/compose.yaml (e.g. traefik:v3.7)
docker compose up -d traefik
```

## Caveat: gluetun + qBittorrent

`qbittorrent` runs with `network_mode: "service:gluetun"` — it shares gluetun's network
namespace. When Watchtower auto-updates **gluetun**, it recreates that container and
qBittorrent can lose its network until it is also recreated. If qBittorrent shows no
connectivity after a gluetun update, fix it with:

```bash
docker compose up -d --force-recreate qbittorrent
```

If this becomes a recurring annoyance, exclude the pair from auto-update by adding
`- "com.centurylinklabs.watchtower.monitor-only=true"` to **gluetun** in
`arr/compose.yaml` and updating both manually.

## Verify it works

```bash
# 1. Validate the merged compose config (set WATCHTOWER_NOTIFICATION_URL in .env first)
docker compose config

# 2. Start Watchtower and watch it schedule the daily run
docker compose up -d watchtower
docker logs -f watchtower

# 3. Test notifications end-to-end without waiting for 06:00 (one-off run)
docker run --rm \
  -e WATCHTOWER_NOTIFICATION_URL="$WATCHTOWER_NOTIFICATION_URL" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower --run-once
```

The one-off run reports which containers are update-eligible and which are monitor-only
(`traefik`, `immich_server`, `immich_machine_learning`), and delivers a summary to your
ntfy topic.

> Watchtower flag/label names above are the stable, long-standing ones. For the latest
> options, see the [official docs](https://containrrr.dev/watchtower/) (Context7:
> `/containrrr/watchtower`).
