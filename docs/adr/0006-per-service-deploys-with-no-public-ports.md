# Each service deploys its own Compose stack to one VPS with no public ports

The bot uses long polling, and the backend is reached only by the bot over a private Docker network, so nothing in the MVP needs to be public. The whole system runs on one EU VPS. Only port 22 is open, for SSH with keys only, and no Compose file publishes a port, because Docker-published ports bypass `ufw`. There is no domain, TLS, or reverse proxy until a webhook or a public API needs one.

Each service repo owns `deploy/compose.yaml` for its own containers: the backend with Postgres, and the bot with Redis. The two stacks meet on an external Docker network named `kitty`, and the bot reaches the backend at the hostname `backend`. Server setup that belongs to neither service lives in the public `kitty-ops` repo. Only `main` deploys, and there is no staging. A merge to `main` builds the image in CI, pushes it to GHCR as a public package tagged with the version and the commit, and deploys over SSH with `docker compose pull` and `docker compose up -d --wait`. A backend deploy takes a `pg_dump` and runs migrations in a one-off container before the new backend starts. Rollback redeploys an older tag and never rolls back a migration.

This costs a few seconds of downtime per deploy, two compose files that share the `kitty` and `backend` names only by agreement, and a CI deploy user whose Docker access equals root on the server. In return, each service releases alone, a bot release never restarts Postgres, production runs the image CI tested, and there is no proxy or certificate to run.

## Considered Options

- A webhook for the bot. It needs a domain, public HTTPS, and a proxy, and buys nothing at 20 messages a minute per group.
- One Compose file for the whole server in a separate repo. It gives one view of the server, but each release would need a cross-repo trigger and a second PR. We can move to it later.
- A staging stack deployed from `develop`. It would share the one VPS with production. We test with CI and a shared test bot instead.
- Building images on the server. Builds would compete with live services for 4 GB of memory, and the running image would differ from the one CI tested.
- Kamal or blue-green deploys. They remove the downtime but hide steps or add a proxy and switch scripts.
- A tunnel such as Tailscale or Cloudflare Tunnel to close port 22. It adds a dependency that can lock us out. Tailscale is the first step if we want zero open ports later.
