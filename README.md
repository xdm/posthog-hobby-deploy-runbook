# PostHog Hobby (Self-Hosted) — Deployment Runbook

Battle-tested notes for deploying the **PostHog hobby** stack (the free, single-server, Docker-Compose self-hosted build) on a fresh VPS — and, more importantly, for *keeping it alive*.

Most public guides stop at "run the one-liner." This one is about everything that goes wrong *after* that: silent container deaths, out-of-memory thrashing, and a `latest`-image bug that quietly kills half the ingestion pipeline. The order of sections is the order you should work in.

> Scope: the **hobby** deployment (`bin/deploy-hobby`, Docker Compose on one host). Not the Kubernetes/enterprise install, and not PostHog Cloud.

---

## TL;DR — the three things that cost the most time

1. **Right-size the server up front.** 4 GB RAM does **not** run this stack reliably: it swaps, Redis times out, and the Node services silently die. Provision **16 GB RAM / 4+ vCPU** for production load (~500k events/day). 8 GB is test-only and tight.
2. **PostHog's Node services exit with code 0 on failure** (they self-terminate gracefully after hitting a connection-error limit). Docker's default `on-failure` policy will **not** restart a container that exited 0 — so the service stays "silently dead" while the setup UI shows a red *Plugin server · Node*. Fix with `restart: always`.
3. **The `latest` image has a bug:** the logs/traces ingestion consumers look for Redis on `127.0.0.1` because the hobby compose file doesn't set their host env vars. You must add them (see §3).

---

## 1. Before you install

- **OS:** Ubuntu 24.04 LTS is the reference (all PostHog docs and scripts target it). 26.04 works too, but you'll be first to hit any doc drift. **x86_64 only, not ARM** — ClickHouse on ARM is historically problematic in this stack.
- **Resources:** minimum **16 GB RAM / 4 vCPU / 100+ GB SSD** for production. Give the disk plenty of headroom — ClickHouse grows continuously with event volume.
- **DNS first:** the domain's A record must resolve to the server IP **before** you install. The bundled Caddy proxy provisions TLS via Let's Encrypt for that domain on startup; no record → no certificate.
- **Add swap** regardless of RAM size — cheap insurance against the OOM killer:
  ```bash
  fallocate -l 8G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
  echo '/swapfile none swap sw 0 0' >> /etc/fstab
  ```

## 2. Install

Official hobby installer (it will prompt for your domain and an email for Let's Encrypt):
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/PostHog/posthog/HEAD/bin/deploy-hobby)"
```

What to expect:
- **Migrations are slow on weak CPUs** — up to an hour on 2 vCPU. It's not hung, just wait. Much faster on 4+ vCPU.
- **Ubuntu 26.04 note:** make sure Docker's apt repo actually serves packages for your release. Image pulls can be slow or flaky (Docker Hub); if a pull fails, loop it with retries until every image is present before continuing.

## 3. Mandatory post-install fixes (without these it dies silently)

Put **all** customisation in a separate **`docker-compose.override.yml`**, and do **not** edit the main `docker-compose.yml`. The `upgrade-hobby` script rewrites the main compose file on upgrade and your edits are lost; it never touches the override.

Create `docker-compose.override.yml` next to the main compose file:

```yaml
services:
  # --- fix for the latest-image bug: point logs/traces consumers at Redis ---
  plugins:
    restart: always
    environment:
      LOGS_REDIS_HOST: redis7
      LOGS_REDIS_PORT: 6379
      TRACES_REDIS_HOST: redis7
      TRACES_REDIS_PORT: 6379
  ingestion-logs:
    restart: always
    environment:
      LOGS_REDIS_HOST: redis7
      LOGS_REDIS_PORT: 6379
      TRACES_REDIS_HOST: redis7
      TRACES_REDIS_PORT: 6379
  ingestion-traces:
    restart: always
    environment:
      LOGS_REDIS_HOST: redis7
      LOGS_REDIS_PORT: 6379
      TRACES_REDIS_HOST: redis7
      TRACES_REDIS_PORT: 6379

  # --- restart: always for every other long-lived service ---
  # web, worker, capture, recordings, proxy, clickhouse, kafka, zookeeper,
  # db, redis7, objectstorage, seaweedfs, temporal, elasticsearch, cymbal, ...
  # (skip the one-shot jobs: kafka-init, asyncmigrationscheck, temporal-admin-tools)
  web: { restart: always }
  worker: { restart: always }
  # ...and so on for each service from `docker compose config --services`
```

Quick way to generate the `restart: always` block for every service at once:
```bash
docker compose config --services | grep -vE 'kafka-init|asyncmigration|temporal-admin' \
  | awk 'BEGIN{print "services:"} {print "  "$1":\n    restart: always"}' \
  > docker-compose.override.yml
# then hand-add the `environment:` blocks for plugins/ingestion-logs/ingestion-traces (above)
docker compose config --quiet && docker compose up -d
```

**Why `restart: always` and not `on-failure`:** when a Node service (plugins, ingestion-*) exhausts its Redis connection-error budget, it performs a graceful shutdown with **exit code 0**. To Docker that's a *successful* exit, so `on-failure` leaves it down. `always` brings it back every time, including after a reboot.

## 4. Diagnosing "service exited 0" (the signature failure of this stack)

If the setup UI shows a red *Plugin server · Node*, or a container is stuck in `Exited (0)`:

```bash
docker ps -a | grep -E 'plugins|ingestion'      # look for Exited (0)
docker logs root-plugins-1 --tail 60            # what it logged before dying
dmesg -T | grep -i 'oom\|killed process'        # did the OOM killer strike?
free -h                                          # is swap maxed out?
```

Reading a Node service's death log:
- `ECONNREFUSED` to `127.0.0.1:6379` → `LOGS_REDIS_HOST`/`TRACES_REDIS_HOST` not set → apply §3.
- `ETIMEDOUT` to Redis + `"Enough of this, I quit!"` + `Shutting nodejs services down gracefully with SIGTERM` → Redis isn't responding in time. This is almost always **RAM pressure**: the stack is swapping and Redis stalls. Check `dmesg` for OOM and `free -h` — if swap is full and `available` is near zero, **add RAM**. There is no config workaround.

Gotcha: container memory in `docker stats` can look modest precisely because most of it has been swapped to disk. Don't trust it — trust `free -h` and `dmesg`.

## 5. Post-deploy verification

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://DOMAIN/_health   # expect 200
curl -s -o /dev/null -w '%{http_code}\n' https://DOMAIN/login     # 200
```
- Right after the `web` container is recreated you may see transient **502s** — Caddy cached the old container's IP. It clears itself within a minute or two; nothing to fix.
- In the UI preflight, *Plugin server · Node* only turns green when `plugins` is alive and emitting a heartbeat — if it's crash-looping (§4), this stays red.

## 6. Operating it

- **Disk:** watch `df -h /`. ClickHouse grows with event volume; plan for expansion or set a TTL on old data.
- **Upgrades:** `upgrade-hobby` rewrites the main `docker-compose.yml`. Anything in `docker-compose.override.yml` survives — so keep every customisation there.
- **After any reboot,** check `docker ps -a` for `Exited (0)`; with `restart: always` in place there should be none.

---

## Notes

- **Multiple projects:** the *New project* button is disabled on the hobby build — multiple projects is a licensed feature, and self-hosted licenses are no longer sold (billing is cloud-only). The free, supported path is to create a separate **organization** per project (one project each).
- PostHog explicitly does not provide support guarantees for self-hosted hobby instances. Treat this runbook as community knowledge, not official documentation.

---

*Contributions welcome — if you hit a failure mode not covered here, open a PR.*
