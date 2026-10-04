# Gitea + Woodpecker CI — self-hosted CI/CD (homelab)

[Version française](README.fr.md)

Self-hosted CI/CD stack on a homelab (Raspberry Pi 5), with no public exposure at all. Access is only through a mesh VPN (Tailscale or equivalent) plus local DNS resolution on the client. Secrets are managed in [Infisical](https://github.com/Alithiel31/infisical-homelab) and injected at deploy time.

## Architecture

```
Browser (client on the mesh VPN)
        │  resolves gitea.homelab.internal / woodpecker.homelab.internal
        │  through a local hosts entry → homelab VPN IP
        ▼
   Traefik (reverse proxy already in place, entrypoint "web", host port 8000)
        │  routing by Host() header, traefik-net network
        ▼
   ┌─────────────┐         ┌────────────────────┐
   │    Gitea    │◄───────►│  Woodpecker Server  │
   │ (git forge) │  OAuth2 │  (CI orchestrator)  │
   └──────┬──────┘         └──────────┬──────────┘
          │                           │ gRPC (port 9000)
          │ native Postgres           ▼
          │ (172.16.0.1:5432)  ┌──────────────────┐
          └───────────────────►│ Woodpecker Agent  │
                                │ (runs the Docker  │
                                │  pipelines)       │
                                └──────────────────┘
```

- **Gitea**: self-hosted Git forge, replaces GitHub so the whole chain (webhooks included) stays strictly internal.
- **Woodpecker Server**: receives Gitea webhooks, orchestrates pipelines, serves the web UI.
- **Woodpecker Agent**: actually runs the pipelines in Docker containers (through the host's Docker socket).
- **Traefik** ([traefik-homelab](https://github.com/Alithiel31/traefik-homelab)): reverse proxy already running on the homelab, routing by domain (Docker labels, `exposedbydefault=false`).
- **Postgres**: shared native instance of the homelab (no dedicated container), one database per service (`gitea`, `woodpecker`).
- **Infisical** ([infisical-homelab](https://github.com/Alithiel31/infisical-homelab)): provides the secrets (DB passwords, gRPC secret, OAuth2 credentials) injected by `deploy.sh`; no secret lives in `.env` anymore.

Dedicated Docker network `ci-net` (`172.16.0.0/24` by default, configurable via `.env`), separate from `traefik-net`.

## Prerequisites

Before starting this stack, the homelab must already have:

- **Docker + Docker Compose** installed.
- **Traefik** deployed and working, with:
  - an external Docker network named `traefik-net` (`docker network create traefik-net` if needed);
  - the Docker provider enabled with `exposedbydefault=false` (label-only routing);
  - an entrypoint `web` listening on host port `8000` (or adapt the URLs in this README/the compose to your actual port).
- **Infisical** deployed and reachable, with a "Shared Keys" project (`prod` environment) and the [Infisical CLI](https://infisical.com/docs/cli/overview) installed on the host.
- **Postgres** (native on the host, not a container) reachable from Docker containers, with `listen_addresses` including the gateway IP of the future `ci-net` network.
- **ufw** (or equivalent) enabled with default-deny inbound — the exact allow rules are given below.
- A **mesh VPN** (Tailscale or equivalent) giving client machines access to the homelab.
- **Gitea and Woodpecker don't need to be installed beforehand** — this stack deploys both.

## Installation from scratch

### 1. Choose the dedicated Docker subnet

Check that no existing Docker network on the host conflicts with the future `ci-net`:
```bash
docker network ls -q | xargs -I{} docker network inspect {} --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```
Adjust `CI_NET_SUBNET`/`CI_NET_GATEWAY` in `.env` if `172.16.0.0/24` is already taken.

### 2. Create the Postgres databases and users

On the host, as the Postgres admin user:
```sql
CREATE USER gitea_app WITH PASSWORD 'a-strong-password';
CREATE DATABASE gitea OWNER gitea_app;

CREATE USER woodpecker_app WITH PASSWORD 'another-strong-password';
CREATE DATABASE woodpecker OWNER woodpecker_app;
```
⚠️ For `WOODPECKER_DB_PASS`, generate the password with `openssl rand -hex 24` (hexadecimal only) rather than `base64`: Woodpecker uses it in a DSN URL (`postgres://user:pass@host/db`) and a special character such as `/` breaks the parsing.

Add the access rule to `pg_hba.conf` (adjust the path to your Postgres version):
```
host    gitea        gitea_app        <CI_NET_SUBNET>    scram-sha-256
host    woodpecker   woodpecker_app   <CI_NET_SUBNET>    scram-sha-256
```
Then reload Postgres (`sudo systemctl reload postgresql` or equivalent).

### 3. Open the required firewall ports

The `ci-net` network must be able to reach Postgres (5432) and Traefik (8000) on the host:
```bash
sudo ufw allow from <CI_NET_SUBNET> to any port 5432 proto tcp comment "gitea+woodpecker -> shared postgres"
sudo ufw allow from <CI_NET_SUBNET> to any port 8000 proto tcp comment "gitea+woodpecker -> traefik"
```
Without these rules: silent timeout (no explicit rejection) at startup — see the Troubleshooting section below.

### 4. Configure internal DNS on the client side

Add to the hosts file of every client machine that needs access (`C:\Windows\System32\drivers\etc\hosts` on Windows, `/etc/hosts` on Linux/macOS):
```
<HOMELAB_VPN_IP>  gitea.homelab.internal
<HOMELAB_VPN_IP>  woodpecker.homelab.internal
```
(replace with the domains actually chosen in `.env` if different)

### 5. Prepare configuration and secrets

**Non-secret settings** (`.env`):
```bash
cp .env.example .env
```
Fill in the values (see [Environment variables](#environment-variables)).

**Secrets** (Infisical, "Shared Keys" project, `prod` environment): create these entries. For `WOODPECKER_GITEA_CLIENT`/`WOODPECKER_GITEA_SECRET`, the Gitea OAuth2 app doesn't exist yet: they will be added in step 7.
- `GITEA_DB_PASS`
- `WOODPECKER_DB_PASS`
- `WOODPECKER_AGENT_SECRET` (e.g. `openssl rand -hex 32`)

**`deploy.sh` access to Infisical**: create a Machine Identity `woodpecker-ci-deploy` (Universal Auth, read access to the project), then:
```bash
cp .infisical-identity.env.example .infisical-identity.env
```
and fill in the Client ID, Client Secret, project ID (`INFISICAL_PROJECT_ID`) and `INFISICAL_API_URL`. This file is never committed.

### 6. Start Gitea alone, then create the admin account

`deploy.sh` forwards its arguments to `docker compose up -d`. To start Gitea only:
```bash
./deploy.sh gitea
```
Since `GITEA__security__INSTALL_LOCK=true` is already set, the web installer is bypassed. Create the admin account directly from the CLI:
```bash
docker exec -u git gitea gitea admin user create --username <user> --password "<pass>" --email <email> --admin
```
(`-u git` is mandatory — the Gitea process runs as `git`, not `root`.)

Then log in at `http://<GITEA_DOMAIN>:8000/` with this account.

### 7. Create the OAuth2 application in Gitea for Woodpecker

In Gitea: **Site Administration → Applications → Manage OAuth2 Applications → Create OAuth2 Application**.
- Name: `Woodpecker CI` (or anything)
- Redirect URI: `http://<WOODPECKER_DOMAIN>:8000/authorize`

Copy the generated **Client ID** and **Client Secret** into Infisical ("Shared Keys" project, `prod` environment) as `WOODPECKER_GITEA_CLIENT` / `WOODPECKER_GITEA_SECRET`.

### 8. Start the rest of the stack

```bash
./deploy.sh
```
The script authenticates to Infisical with the Machine Identity, then runs `docker compose up -d` with the secrets injected. Check the logs:
```bash
docker logs woodpecker-server --tail 30
docker logs woodpecker-agent --tail 30
```
The server must **not** print `WOODPECKER_GRPC_SECRET is not set` (otherwise `WOODPECKER_AGENT_SECRET` wasn't picked up correctly). The agent must print `polling new workflow` with no `fatal` error.

### 9. Check that the agent is connected

Log in at `http://<WOODPECKER_DOMAIN>:8000/` with "Login with Gitea", then go to **Admin → Agents** (gear icon, only visible if your Gitea account is listed in `WOODPECKER_ADMIN_USER`). An agent with a recent "last contact" confirms everything works.

## Access

| Service | URL | Notes |
|---|---|---|
| Gitea | `http://gitea.homelab.internal:8000/` | admin: `<your-username>` |
| Woodpecker | `http://woodpecker.homelab.internal:8000/` | login through "Login with Gitea" (OAuth2) |

## Files

- `docker-compose.yml` — definition of the 3 services (`gitea`, `woodpecker-server`, `woodpecker-agent`)
- `deploy.sh` — Infisical authentication + `docker compose up -d [args]` with secrets injected (e.g. `./deploy.sh gitea`)
- `.env` — non-secret settings (never committed, see `.env.example`)
- `.infisical-identity.env` — Machine Identity Client ID/Secret (never committed, see `.infisical-identity.env.example`)

## Environment variables

### In `.env` (non-secret, see `.env.example`)

| Variable | Usage |
|---|---|
| `CI_NET_SUBNET` / `CI_NET_GATEWAY` | Docker subnet dedicated to Gitea/Woodpecker and its gateway |
| `GITEA_DOMAIN` / `WOODPECKER_DOMAIN` | internal domains used by Traefik and resolved on the client side |
| `WOODPECKER_ADMIN_USER` | Gitea username that must have admin rights on Woodpecker |

### In Infisical ("Shared Keys" project, `prod` environment)

| Variable | Usage |
|---|---|
| `GITEA_DB_PASS` | Postgres password of the `gitea_app` user |
| `WOODPECKER_DB_PASS` | Postgres password of the `woodpecker_app` user (generated as hex, never base64 — a `/` breaks the `postgres://user:pass@host/db` DSN parsing) |
| `WOODPECKER_AGENT_SECRET` | shared gRPC secret server↔agent. **Careful**: mapped to `WOODPECKER_GRPC_SECRET` on the server and `WOODPECKER_AGENT_SECRET` on the agent in the compose — two different names for the same value (v3.18.0 pitfall, see below) |
| `WOODPECKER_GITEA_CLIENT` / `WOODPECKER_GITEA_SECRET` | credentials of the "Woodpecker CI" OAuth2 app created in Gitea (see installation step 7) |

## Security

- `WOODPECKER_OPEN=true`: any Gitea account can log in to Woodpecker. Acceptable on a private network; revisit if other users get a Gitea account.
- The agent mounts the host's Docker socket (`/var/run/docker.sock`, read-write): a pipeline can control Docker on the host. Only run trusted repositories.
- Secrets live only in Infisical; never commit `.env` or `.infisical-identity.env`.

## Troubleshooting / pitfalls met

1. **ufw silently blocks new Docker subnets.** Any outgoing connection from a `ci-net` container to a host port (Postgres 5432, Traefik 8000) needs an explicit rule (see installation step 3). Without it: silent timeout, no explicit rejection — diagnose with `docker exec <container> wget -T 3 -O- http://<host>:<port>` (timeout exactly at `-T` = network blocked; near-instant error = port reachable). Note: the Woodpecker images are "distroless", with no `wget`/`sh` — this test only works on containers that have a shell (e.g. Gitea).

2. **`extra_hosts: host.docker.internal:host-gateway` doesn't work on a custom network.** The magic `host-gateway` mapping always resolves to the default Docker bridge gateway (`172.17.0.1`), never to a custom network's. The real gateway IP must be hardcoded (`CI_NET_GATEWAY`).

3. **`WOODPECKER_GITEA_URL` is used by both the server AND the client browser.** Never put an internal Docker service name in it (`http://gitea:3000`): it breaks the "Login with Gitea" OAuth redirect on the browser side, which can't resolve that name. Use the internal domain resolved on the client side, and add an `extra_hosts` on `woodpecker-server` so it resolves it too.

4. **The shared gRPC secret has a different variable name depending on the service** (Woodpecker v3.18.0):
   - Server: `WOODPECKER_GRPC_SECRET`
   - Agent: `WOODPECKER_AGENT_SECRET`
   Same value, two different names. A mismatch gives either `signature is invalid` (wrong value) or `please provide a token` (variable not recognized by the binary). When in doubt about the variables really accepted, rely on `docker exec <container> <binary> --help` rather than the online docs (which may describe a different version).

5. **Two distinct agent mechanisms in the Woodpecker UI**, not to be confused:
   - *System agent* (shared secret `WOODPECKER_GRPC_SECRET`/`WOODPECKER_AGENT_SECRET`): self-registers on first contact, only visible in **Admin → Agents**.
   - *Agent Token* ("Add agent" button in the user account settings): a single token generated by hand, visible in **Account settings → Agents**. Different mechanism, not used in this deployment.

6. **Being a Gitea admin doesn't automatically make you a Woodpecker admin.** `WOODPECKER_ADMIN_USER` must be set correctly on the server, then restart the container and log out/in (admin status is checked at login, not in real time) for the Admin menu to appear.

## Status

✅ Full deployment working: Gitea and Woodpecker operational, agents connected, OAuth login validated.

⬜ Still to validate: a real first pipeline run (create a test repository on Gitea, activate it on Woodpecker, push a `.woodpecker.yml`).

## Useful commands

```bash
# Redeploy after changing the compose or .env (secrets injected from Infisical)
./deploy.sh

# Force the recreation of a specific service
./deploy.sh --force-recreate <service>

# Logs
docker logs gitea --tail 50
docker logs woodpecker-server --tail 50
docker logs woodpecker-agent --tail 50

# Check that an environment variable is correctly passed to a container
docker exec <container> env | grep <VAR>
```

## Related projects

| Repository | Role |
| --- | --- |
| [traefik-homelab](https://github.com/Alithiel31/traefik-homelab) | Reverse proxy |
| [infisical-homelab](https://github.com/Alithiel31/infisical-homelab) | Secrets manager |
| [plantuml-traefik](https://github.com/Alithiel31/plantuml-traefik) | PlantUML server |

## License

MIT — see [LICENSE](./LICENSE).
