# Self-hosted VPS deployment baseline

Research for [#7](https://github.com/potluck-labs/kitty/issues/7), part of map [#2](https://github.com/potluck-labs/kitty/issues/2).
Researched on 2026-10-28. All prices are monthly, in EUR, excluding VAT unless noted, as shown on the provider page on that date. Prices change. Re-check them before ordering.

This file lists options and trade-offs. It does not pick a winner. A later ticket decides.

## Question

What is a sound, learnable baseline for deploying two Dockerized services (the Telegram bot and the backend) and Postgres on our own VPS? The ticket asks about:

1. EU VPS providers and rough costs.
2. Reverse proxy and TLS.
3. Secrets handling.
4. Postgres backups and restore testing.
5. Basic monitoring and log collection.
6. Zero or low-downtime deploys.
7. GitHub Actions deploy patterns.

## Context that shapes the answer

- Telegram offers two mutually exclusive ways to receive updates: long polling with `getUpdates`, or a webhook. The docs call them "mutually exclusive ways of receiving updates." ([Telegram Bot API](https://core.telegram.org/bots/api#getting-updates))
- A webhook needs a public HTTPS URL on port 443, 80, 88, or 8443. Telegram supports self-signed certificates if you upload the public key, and an optional `secret_token` that it sends in the `X-Telegram-Bot-Api-Secret-Token` header on every request. ([Telegram Bot API, setWebhook](https://core.telegram.org/bots/api#setwebhook))
- With long polling, the bot opens outbound connections only. Telegram keeps pending updates, but they "will not be kept longer than 24 hours." ([Telegram Bot API](https://core.telegram.org/bots/api#getting-updates))
- For a failed webhook delivery (any non-2XX status), Telegram will "repeat the request and give up after a reasonable amount of attempts." The docs give no exact count. ([Telegram Bot API](https://core.telegram.org/bots/api#getting-updates))

Consequence for this ticket: if the bot uses long polling and talks to the backend over the private Docker network, nothing needs a public HTTPS endpoint at all. A reverse proxy with TLS is needed only if the bot uses a webhook, or if the backend exposes a public API. That choice belongs to the architecture tickets, but it changes how much of section 2 applies.

## 1. EU VPS providers and costs

Fetched 2026-10-28. "From" prices may assume a commitment, as noted.

| Provider | Plan | vCPU | RAM | Disk | Traffic | Price | Terms and notes |
|---|---|---|---|---|---|---|---|
| Hetzner Cloud (DE, FI) | CX23 (shared, x86) | 2 | 4 GB | 40 GB NVMe | 20 TB | 5.99, IPv4 included | Hourly billing. Page showed "currently unavailable" for all Cost-Optimized plans. |
| Hetzner Cloud (DE, FI) | CAX11 (shared, Arm) | 2 | 4 GB | 40 GB | 20 TB | 6.49, IPv4 included | Same availability note. Arm needs arm64 images. |
| Hetzner Cloud (DE, FI) | CX33 | 4 | 8 GB | 80 GB | 20 TB | 8.99, IPv4 included | Same availability note. |
| Hetzner Cloud (DE, FI) | CPX22 (shared, AMD) | 2 | 4 GB | 80 GB | 20 TB | 19.99, IPv4 included | Page also showed "currently unavailable" on Create. |
| OVHcloud | VPS-1 | 2 | 4 GB | 40 GB NVMe | Unlimited, 500 Mbps | from 3.81 | Configurator links carry `pricing=upfront12`, so the price likely assumes 12 months paid upfront. Daily backup included. |
| OVHcloud | VPS-2 | 4 | 8 GB | 75 GB NVMe | Unlimited, 1 Gbps | from 7.21 | Same as above. |
| netcup | VPS 500 G12.5 | 2 | 4 GB | 64 GB | Not stated | 8.26 (9.50 with 19% VAT) | Price for a 12-month term. 1 and 24 month terms exist. Locations Vienna, Nuremberg, Amsterdam. |
| netcup | VPS 1000 G12.5 | 4 | 8 GB | 128 GB | Not stated | 14.50 (16.68 with VAT) | Same terms. |
| Scaleway | | | | | | Not verified | scaleway.com returned HTTP 403 to the fetch tool. Check by hand. |

Sources: [Hetzner Cost-Optimized](https://www.hetzner.com/cloud/cost-optimized/), [Hetzner Regular Performance](https://www.hetzner.com/cloud/regular-performance/), [OVHcloud VPS (en-ie)](https://www.ovhcloud.com/en-ie/vps/), [netcup VPS](https://www.netcup.com/en/server/vps).

Hetzner price and stock changes in 2026:

- Hetzner raised cloud prices for new orders and rescales from 15 June 2026. Servers already rented keep their old price. Examples, excluding IPv4: CX23 went from 3.99 to 5.49, CAX11 from 4.49 to 5.99, CPX22 from 7.99 to 19.49. ([Hetzner price adjustment](https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/), [press release](https://www.hetzner.com/pressroom/standardization-and-price-adjustment-of-our-server-products/))
- The live product pages list prices "incl. IPv4", which is about 0.50 more than the price-adjustment table. That gap matches a primary IPv4 charge. This is my inference from the two numbers.
- Both Hetzner plan pages marked plans "currently unavailable" on 2026-10-28. Third-party reports say a Hetzner status notice about limited cloud availability has been open since 26 June 2026. I did not confirm this on Hetzner's status page. Check stock before planning around Hetzner.

Backup add-ons and off-site storage:

- Hetzner Cloud backups cost from 1.198 per month, priced relative to the server. ([Hetzner Cost-Optimized](https://www.hetzner.com/cloud/cost-optimized/))
- OVHcloud VPS include a daily backup on a rolling 7-day schedule, replicated three times in the same datacentre. A premium backup with off-site storage is an add-on. ([OVHcloud VPS](https://www.ovhcloud.com/en-ie/vps/))
- Hetzner Storage Box BX11 has 1 TB for 3.20, in Germany or Finland. It supports SFTP, rsync over SSH, BorgBackup, restic, and rclone. ([Hetzner Storage Box](https://www.hetzner.com/storage/storage-box/))

Sizing: a bot, a backend, Postgres, and a reverse proxy at MVP load are small. A 2 vCPU, 4 GB plan is a reasonable starting point. A self-hosted Prometheus, Grafana, and Loki stack (section 5) adds memory, which points to 8 GB. This is my estimate, not a measured number.

Trade-offs between providers:

- Hetzner has hourly billing, a mature API and Terraform provider, and cheap Storage Boxes. Prices rose sharply in June 2026 and stock of the cheapest plans was unavailable when checked.
- OVHcloud has the lowest listed price and backups included, but the listed price likely needs a 12-month upfront payment.
- netcup gives the most disk per euro here, but the price needs a 12-month term.

## 2. Reverse proxy and TLS

All three options below get free certificates from Let's Encrypt over ACME. Renewal must be automatic. Let's Encrypt is shortening certificate lifetimes: the default profile moves to 64 days on 10 February 2027 and to 45 days on 16 February 2028. They advise clients to use ACME Renewal Information (ARI), to avoid manual renewal, and to alert on failed renewals. ([Let's Encrypt, "From 90 to 45"](https://letsencrypt.org/2025/12/02/from-90-to-45/))

### Caddy

- Automatic HTTPS is on by default. Requirements: DNS A/AAAA records point to the server, ports 80 and 443 are open, and the data directory is writable and persistent. ([Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https))
- It redirects HTTP to HTTPS on its own. It tries Let's Encrypt, then falls back to ZeroSSL if issuance fails, and uses the Let's Encrypt staging endpoint while retrying to avoid rate limits. ([Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https))
- In Docker, the certificate storage must be on a named volume. Losing it forces re-issuance, and hitting production rate limits "can block HTTPS access for up to a week." ([Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https))
- `reverse_proxy` can hold requests during a restart with `lb_try_duration` (off by default, docs suggest `5s` as a start) and supports active health checks with `health_uri`. ([Caddy reverse_proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy))
- Configuration is one small Caddyfile, edited and reloaded by hand or by the deploy script.

### Traefik (v3)

- Discovers services from Docker container labels, for example `traefik.http.routers.<name>.rule=Host(...)`. Routing config lives next to each service in `compose.yaml`. ([Traefik Docker provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/))
- Needs the Docker API, usually by mounting `/var/run/docker.sock`. The docs warn: "Accessing the Docker API without any restriction is a security concern," since a compromised Traefik could reach the host. They suggest a filtering socket proxy or TLS client certificates. ([Traefik Docker provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/))
- `exposedByDefault` defaults to `true`, so every container is exposed unless you set it to `false` and opt in with `traefik.enable=true`. ([Traefik Docker provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/))
- Supports HTTP-01 (port 80), TLS-ALPN-01 (port 443), and DNS-01 (needed for wildcards). Stores certificates in `acme.json`. Renews 30 days before expiry for 90-day certificates. ([Traefik ACME resolver](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/))
- More concepts to learn (entrypoints, routers, services, middlewares, resolvers) than Caddy for the same result on one host.

### nginx with certbot

- certbot has three relevant authenticators: `--nginx` (obtains and edits nginx config), `--webroot` (writes a file under `.well-known/acme-challenge` that nginx serves), and `--standalone` (runs its own server on port 80). ([certbot user guide](https://eff-certbot.readthedocs.io/en/stable/using.html))
- Renewal runs from a cron job or systemd timer calling `certbot renew`. Since certbot 4.0.0 it renews when less than a third of the lifetime remains. A `--deploy-hook` or a script in `/etc/letsencrypt/renewal-hooks/deploy` runs only after a successful renewal, which is where nginx gets reloaded. ([certbot user guide](https://eff-certbot.readthedocs.io/en/stable/using.html))
- The certbot docs have almost no Docker guidance. Running certbot and nginx in containers means wiring a shared webroot volume, a shared certificate volume, a renewal schedule, and a reload signal yourself.
- Most moving parts of the three. Also the most transferable knowledge, since nginx config appears in many jobs.

### Summary

| | Caddy | Traefik | nginx and certbot |
|---|---|---|---|
| TLS setup | Automatic, on by default | Resolver config plus labels | Separate tool, cron or timer, reload hook |
| Where routing lives | Caddyfile | Labels in `compose.yaml` | nginx config files |
| Docker socket needed | No | Yes, with a documented risk | No |
| Hold requests during a restart | `lb_try_duration` | Health checks and retries | `proxy_next_upstream` (not researched here) |
| Parts to learn | Fewest | Medium | Most |

## 3. Secrets handling

Secrets for Kitty: the Telegram bot token, the webhook `secret_token` if used, the Postgres password, any backend-to-bot shared secret, the registry pull token, and the backup encryption password.

Options, from least to most setup:

1. A `.env` file on the server, outside Git, readable only by the deploy user. Compose reads it for variable substitution. Simple. The secret ends up in container environment variables.
2. Compose secrets. Define a top-level `secrets` entry from a file, grant it per service, and the container sees it at `/run/secrets/<name>`. Docker warns that environment variables can lead to "unintentional information exposure," since they are visible to all processes and can end up in logs. The official `postgres` image reads `POSTGRES_PASSWORD_FILE`, which points at the secret file. Our own Python and TypeScript services would need to read the file themselves. ([Docker Compose secrets](https://docs.docker.com/compose/how-tos/use-secrets/))
3. GitHub Actions secrets as the source of truth. The deploy job writes them to the server at deploy time, into a `.env` or secret files. Environment secrets are only given to jobs that use that environment, after its protection rules pass. Secrets are not passed to workflows triggered from forks. Avoid passing secrets on the command line. ([GitHub Actions secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets), [GitHub environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments))
4. SOPS with age. Commit encrypted secret files (YAML, JSON, ENV, INI) to the repo and decrypt on the server or in CI with an age key. Secrets get version history and review. The age private key becomes the one secret to protect. ([SOPS](https://github.com/getsops/sops))

Notes:

- `potluck-labs/kitty` is public. GitHub Free allows environments on public repositories. Private repositories need Team or Pro. ([GitHub environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments))
- Option 2 combines with 1, 3, or 4. Options 3 and 4 are about where secrets come from. Option 2 is about how they reach the container.

## 4. Postgres backups and restore testing

PostgreSQL 18 is current (18.6 docs on 2026-10-28). The docs list three methods: SQL dump, file system level backup, and continuous archiving. ([PostgreSQL backup](https://www.postgresql.org/docs/current/backup.html))

### Logical dumps with pg_dump

- A dump "represents a snapshot of the database at the time pg_dump began running" and does not block normal reads and writes. ([pg_dump chapter](https://www.postgresql.org/docs/current/backup-dump.html))
- Custom format (`pg_dump -Fc`) is compressed and restores with `pg_restore`, including single tables. ([pg_dump chapter](https://www.postgresql.org/docs/current/backup-dump.html))
- pg_dump skips roles and tablespaces. Run `pg_dumpall --globals-only` too, or recreate roles from code. ([pg_dump chapter](https://www.postgresql.org/docs/current/backup-dump.html))
- Dumps "can generally be re-loaded into newer versions of PostgreSQL," unlike file-level and WAL backups, which are "extremely server-version-specific." ([pg_dump chapter](https://www.postgresql.org/docs/current/backup-dump.html))
- Recovery point equals the time of the last dump. A nightly dump can lose up to 24 hours of Transactions.

### Continuous archiving and point-in-time recovery (PITR)

- Needs a base backup (`pg_basebackup`) plus an unbroken archive of WAL files via `archive_command` or `archive_library`. Allows restore to any moment, not only the last dump. ([continuous archiving](https://www.postgresql.org/docs/current/continuous-archiving.html))
- Restores the whole cluster only, needs "a lot of archival storage," and does not cover `postgresql.conf` or `pg_hba.conf`. If the WAL disk fills because archiving fails, "PostgreSQL will do a PANIC shutdown." ([continuous archiving](https://www.postgresql.org/docs/current/continuous-archiving.html))
- Tools such as pgBackRest and WAL-G wrap this. I did not read their docs for this ticket.
- More to learn and more ways to break. Much smaller data loss window.

### Getting backups off the server

- A backup on the same VPS is lost with the VPS. Copy dumps off the host.
- restic encrypts repositories with a password and supports SFTP, S3-compatible storage, Backblaze B2, a REST server, and rclone targets. ([restic repositories](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html))
- A Hetzner Storage Box (1 TB for 3.20) speaks SFTP, restic, and BorgBackup. Any S3-compatible bucket also works. ([Hetzner Storage Box](https://www.hetzner.com/storage/storage-box/))
- Provider VPS backups (section 1) snapshot the disk. They help with a broken server. They are not a substitute for database-level dumps you can restore elsewhere. This is my judgement, not a provider claim.

### Restore testing

The Postgres docs say to test the archiving procedure before the first base backup, to "inspect the contents of the database to ensure you have recovered to the desired state," and to keep several backup sets. ([continuous archiving](https://www.postgresql.org/docs/current/continuous-archiving.html))

A testable pattern built from the documented commands:

1. On a schedule (weekly, say), start a throwaway Postgres container of the same major version.
2. Fetch the latest dump from off-site storage.
3. Restore with `pg_restore`, or for plain SQL with `psql -X --set ON_ERROR_STOP=on --single-transaction`, so any error fails the run. ([pg_dump chapter](https://www.postgresql.org/docs/current/backup-dump.html))
4. Run a few checks, such as row counts and the newest Transaction date.
5. Report success to a dead man's switch (section 5), so a silent failure raises an alert.

This can run as a cron job on the VPS or as a scheduled GitHub Actions workflow. In CI, the runner needs read access to the backup store.

## 5. Monitoring and log collection

Two facts apply to every option:

- Docker's default `json-file` log driver does not rotate: "By default, no log-rotation is performed." Logs can fill the disk. Docker recommends the `local` driver, which rotates by default, or `max-size` and `max-file` options in `/etc/docker/daemon.json`. The change applies only to containers created afterwards. ([Docker logging](https://docs.docker.com/engine/logging/configure/))
- A monitor running on the same VPS cannot alert when that VPS is down. Something outside the box should probe it. This is my reasoning.

Options:

1. Minimal. Rotated Docker logs read with `docker compose logs`, an external uptime check, and a dead man's switch for cron jobs.
   - Uptime Kuma checks HTTP, TCP, ping, Docker containers, and push heartbeats, and sends alerts to Telegram among 90+ services. It runs as one container (`louislam/uptime-kuma:2`) and does not support NFS for its data. Run it somewhere other than the Kitty VPS, or use a hosted checker. ([Uptime Kuma](https://github.com/louislam/uptime-kuma))
   - Healthchecks.io gives each job a ping URL. If no ping arrives within the period plus a grace time, it alerts. The free Hobbyist plan covers 20 jobs. It can be self-hosted. ([Healthchecks docs](https://healthchecks.io/docs/), [pricing](https://healthchecks.io/pricing/))
2. Self-hosted metrics and logs. Prometheus with node_exporter (host metrics on port 9100), Grafana for dashboards, and Loki for logs, collected by Grafana Alloy. Teaches the most. Uses memory and disk on the same small VPS, and goes down with it. ([node_exporter guide](https://prometheus.io/docs/guides/node-exporter/), [Grafana Alloy](https://grafana.com/docs/alloy/latest/))
3. Hosted metrics and logs. Run only the Alloy agent on the VPS and ship to Grafana Cloud. The free tier allows 10k active metric series, 50 GB of logs per month, 14 days retention, and 3 users. I did not confirm an EU region on the pricing page. ([Grafana pricing](https://grafana.com/pricing/))

Error tracking and alert rules are part of the separate "Observability" open item on map #2.

## 6. Zero or low-downtime deploys

- `docker compose up` "picks up the changes by stopping and recreating the containers." The old container stops before the new one is ready, so each deploy has a short gap. `--wait` blocks until services are running or healthy, which lets a deploy script fail when a new container never becomes healthy. ([docker compose up](https://docs.docker.com/reference/cli/docker/compose/up/))
- How much a gap matters depends on how the bot receives updates (see "Context that shapes the answer"). With long polling, the bot is simply offline for a few seconds and Telegram keeps updates for up to 24 hours. With a webhook, Telegram retries failed deliveries a limited number of times. ([Telegram Bot API](https://core.telegram.org/bots/api#getting-updates))

Options:

1. Accept a few seconds of downtime. Plain `docker compose pull` then `docker compose up -d --wait`. Add a Compose healthcheck per service. Simplest. Easy to reason about.
2. Same as 1, but have the proxy hold requests. Caddy's `lb_try_duration` keeps requests waiting "while rebooting or redeploying an upstream." Clients see a slow response instead of an error, for HTTP traffic only. ([Caddy reverse_proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy))
3. Blue-green by hand. Start the new version next to the old one under a different name, wait for its health check, switch the proxy upstream, then stop the old one. Real zero downtime for HTTP. Needs a deploy script and care with the bot: two long-polling bot instances cannot both call `getUpdates` cleanly, so the bot would still need a stop-then-start. Database migrations must work with both versions during the overlap.
4. Kamal 2. A deploy tool that builds, pushes, and runs containers over SSH. Its kamal-proxy routes traffic to a new container only after it answers `200 OK` on `/up`, then stops the old one, which the docs call "gapless deployments." `ssl: true` gets Let's Encrypt certificates on a single server. Accessories such as Postgres "do not have zero-downtime deployments" and "are not updated when you deploy." Kamal installs as a Ruby gem or runs in a container, and wants root SSH by default. ([Kamal installation](https://kamal-deploy.org/docs/installation/), [Kamal proxy](https://kamal-deploy.org/docs/configuration/proxy/), [Kamal accessories](https://kamal-deploy.org/docs/configuration/accessories/))

Trade-off with learning goals: options 1 to 3 show every step in our own scripts. Kamal hides many steps behind one tool. Map #2 lists "building our own deploy tool or platform" as out of scope, which argues against growing option 3 into a large custom system.

## 7. GitHub Actions deploy patterns

Registry facts:

- GHCR is free for public packages, including storage and bandwidth. Private packages on GitHub Free get 500 MB storage and 1 GB transfer per month. ([GitHub Packages billing](https://docs.github.com/en/billing/concepts/product-billing/github-packages))
- Workflows push with `GITHUB_TOKEN`. New packages start private. A server pulling a private image logs in with a classic personal access token with `read:packages`. Public images pull anonymously. ([GHCR docs](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry))

Patterns:

1. Build in CI, push to GHCR, SSH to deploy. The runner builds and pushes images tagged with the commit SHA, then SSHes to the VPS and runs `docker compose pull` and `docker compose up -d --wait`. The server never builds. Rolling back means deploying an older tag. Needs an SSH private key and the server's host key stored as secrets.
2. Build in CI, deploy through a Docker context over SSH. Create a context with `--docker host=ssh://user@host`, then run `docker compose` on the runner against the remote daemon. The remote user "must have permissions to access the docker socket." ([Docker protect access](https://docs.docker.com/engine/security/protect-access/)) No deploy script on the server, but the Compose file and env files must be available to the runner.
3. Build on the server. SSH in, `git pull`, `docker compose build`, `docker compose up -d`. No registry. Builds use VPS CPU and memory, and the running image has no tag tied to a commit unless you add one.
4. Pull-based. The server polls the registry and updates itself, so CI needs no SSH access to the server. Tools such as Watchtower do this. I did not research them for this ticket. Harder to see which version is live and when it changed.
5. Kamal from CI. CI runs `kamal deploy`, which builds, pushes, and deploys over SSH. See section 6.

Security notes that apply to patterns 1, 2, 3, and 5:

- Membership of the `docker` group, or access to the Docker socket, is root on the host. Docker says to guard such access "as you would a root password." ([Docker protect access](https://docs.docker.com/engine/security/protect-access/)) A deploy SSH key with Docker access is a root key.
- Use a GitHub environment (for example `production`) to scope deploy secrets, limit deploys to `main`, and optionally require one of the two of us to approve. Available on the free plan because the repo is public. ([GitHub environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments))
- Docker-published ports bypass ufw: "Docker and ufw use firewall rules in ways that make them incompatible with each other." Bind internal services such as Postgres to the Docker network only, or to `127.0.0.1`, rather than relying on ufw. ([Docker packet filtering](https://docs.docker.com/engine/network/packet-filtering-firewalls/))

## Example baselines

These are coherent combinations, not recommendations.

| | A. Smallest | B. Proxy-centric | C. Tool-driven |
|---|---|---|---|
| Bot updates | Long polling, no public port | Webhook | Webhook |
| Proxy and TLS | None at first | Caddy or nginx and certbot | kamal-proxy |
| Secrets | `.env` or Compose secrets from GitHub secrets | Compose secrets, SOPS or GitHub secrets | Kamal secrets (not researched) |
| Backups | Nightly `pg_dump -Fc` with restic to off-site, weekly restore test | Same, or PITR | Same as A or B |
| Monitoring | Log rotation, external uptime check, Healthchecks.io | Plus Grafana Cloud or self-hosted stack | Same as B |
| Deploys | Pattern 1, option 1 | Pattern 1 or 2, option 2 or 3 | Pattern 5, option 4 |
| What we learn | Fewest parts, every step visible | Proxy, TLS, health-checked switch | A real deploy tool, less of the inside |

## Open questions for the decision ticket

1. Long polling or webhook for the bot? This decides whether we need public HTTPS at launch.
2. Does the backend need a public endpoint in the MVP?
3. How much Transaction data can we lose: up to 24 hours (nightly dump) or minutes (PITR)?
4. Do we accept a few seconds of downtime per deploy?
5. Which provider has stock on the day we order, and do we accept a 12-month term?
6. Should images be public on GHCR (free, anonymous pull) or private (needs a pull token on the server)?

## Sources

- Telegram Bot API: https://core.telegram.org/bots/api
- Hetzner Cloud plans: https://www.hetzner.com/cloud/cost-optimized/ and https://www.hetzner.com/cloud/regular-performance/
- Hetzner price adjustment, 15 June 2026: https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/
- Hetzner Storage Box: https://www.hetzner.com/storage/storage-box/
- OVHcloud VPS: https://www.ovhcloud.com/en-ie/vps/
- netcup VPS: https://www.netcup.com/en/server/vps
- Let's Encrypt lifetime change: https://letsencrypt.org/2025/12/02/from-90-to-45/
- Caddy automatic HTTPS: https://caddyserver.com/docs/automatic-https
- Caddy reverse_proxy: https://caddyserver.com/docs/caddyfile/directives/reverse_proxy
- Traefik Docker provider: https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- Traefik ACME: https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/
- certbot user guide: https://eff-certbot.readthedocs.io/en/stable/using.html
- Docker Compose secrets: https://docs.docker.com/compose/how-tos/use-secrets/
- Docker logging: https://docs.docker.com/engine/logging/configure/
- docker compose up: https://docs.docker.com/reference/cli/docker/compose/up/
- Docker daemon access over SSH: https://docs.docker.com/engine/security/protect-access/
- Docker and firewalls: https://docs.docker.com/engine/network/packet-filtering-firewalls/
- PostgreSQL backup chapter: https://www.postgresql.org/docs/current/backup.html
- pg_dump: https://www.postgresql.org/docs/current/backup-dump.html
- Continuous archiving and PITR: https://www.postgresql.org/docs/current/continuous-archiving.html
- restic repositories: https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html
- SOPS: https://github.com/getsops/sops
- GitHub Actions secrets: https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets
- GitHub environments: https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments
- GHCR: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
- GitHub Packages billing: https://docs.github.com/en/billing/concepts/product-billing/github-packages
- Kamal: https://kamal-deploy.org/docs/installation/, https://kamal-deploy.org/docs/configuration/proxy/, https://kamal-deploy.org/docs/configuration/accessories/
- Uptime Kuma: https://github.com/louislam/uptime-kuma
- Healthchecks.io: https://healthchecks.io/docs/ and https://healthchecks.io/pricing/
- Prometheus node_exporter: https://prometheus.io/docs/guides/node-exporter/
- Grafana Alloy: https://grafana.com/docs/alloy/latest/
- Grafana Cloud pricing: https://grafana.com/pricing/
