# TraefikV3 — Simple guide (Easy to Read)

This folder contains a ready‑to‑use Traefik v3 setup with CrowdSec.
It helps you put a secure reverse proxy in front of your apps.

Note: The `dynamic_host` folder is work in progress. Ignore it for now.

---

## What is inside

- Traefik (reverse proxy) with HTTPS via Let’s Encrypt.
- CrowdSec (blocks bad IPs; reads Traefik access logs).
- A small test app (`whoami`).
- Log rotation for Traefik logs.

Main files:
- `docker-compose.yml` — starts Traefik, CrowdSec, and logrotate.
- `traefik.yml` — Traefik static config (entry points, ACME, plugin).
- `traefik_dynamic.yml` — Traefik dynamic config (CrowdSec middleware).
- `crowdsec/config/acquis.yaml` — tells CrowdSec where to read logs.
- `test_conteneur/docker-compose.yml` — demo service with Traefik labels.

---

## Quick start

1) Create the Docker network (Traefik uses it):
```bash
docker network create frontend
```

2) Prepare Let’s Encrypt storage file and copy dynamic conf :
```bash
mkdir -p letsencrypt
cp traefik_dynamic_exemple.yml traefik_dynamic.yml
```

3) Start Traefik + CrowdSec:
```bash
docker compose up -d
```

4) Create a CrowdSec bouncer key (copy the printed key):
```bash
docker exec -t crowdsec cscli bouncers add traefik-bouncer
```

5) Put the key in `traefik_dynamic.yml`:
- Edit `crowdsecLapiKey: "YOUR_GENERATED_BOUNCER_API_KEY_FROM_CROWDSEC"`.
- Save the file.
- Traefik watches the file and will reload it.

6) (Optional) Install the Traefik collection in CrowdSec:
```bash
docker exec -t crowdsec cscli collections install crowdsecurity/traefik
```

7) Verify CrowdSec reads the logs:
```bash
docker exec -t crowdsec cscli metrics
```

---

## How to add a test service (whoami)

1) Set your domain shell variable (replace with your real domain):
```bash
export DOMAIN=example.com
```

2) Start the test service:
```bash
cd test_conteneur
DOMAIN=$DOMAIN docker compose up -d
```

3) Check it in a browser:
- Open: `https://example.com` (replace with your domain)
- The certificate comes from Let’s Encrypt.

4) See CrowdSec status:
```bash
docker exec -t crowdsec cscli bouncer list
```

---

## How this works (short)

- Traefik listens on ports 80 (HTTP) and 443 (HTTPS).
- All HTTP (80) is redirected to HTTPS (443).
- Let’s Encrypt issues certificates using TLS‑ALPN challenge.
- Traefik writes access logs in `./traefik_logs`.
- CrowdSec reads these logs and decides which IPs to block.
- The Traefik plugin `crowdsec-bouncer-traefik-plugin` applies the block.
- The middleware is attached globally to `websecure` and can also be added per‑router.

---

## Commands you may need

- Check Metrics and file acquisition:
```bash
docker exec -t crowdsec cscli metrics
```

- List CrowdSec collections:
```bash
docker exec -t crowdsec cscli collections list
```

- List decisions (bans):
```bash
docker exec -t crowdsec cscli decisions list
```

- Manually ban/unban an IP:
```bash
docker exec -t crowdsec cscli decisions add --ip 1.2.3.4
docker exec -t crowdsec cscli decisions delete -i 1.2.3.4
```

- Check Traefik logs:
```bash
tail -f traefik_logs/access.log
```

- List and inspect alert :
```bash
docker exec -t crowdsec cscli alert list
docker exec -t crowdsec cscli alerts inspect -d 56
```

---

## Configuration notes

- `traefik.yml`
  - Docker provider enabled. Only containers with `traefik.enable=true` are exposed.
  - File provider watches `traefik_dynamic.yml`.
  - Global middleware on `websecure` adds CrowdSec protection by default.
  - Let’s Encrypt: storage in `./letsencrypt/acme.json`, TLS challenge enabled.
  - Access logs in JSON, both success and error ranges recorded.
  - CrowdSec plugin declared in `experimental.plugins`.

- `traefik_dynamic.yml`
  - Defines `middlewares.crowdsec` using the Traefik CrowdSec bouncer plugin.
  - Replace the API key with the one you generated.
  - AppSec is disabled (you can enable it if you run CrowdSec AppSec).

- `docker-compose.yml`
  - Mounts Docker socket read‑only (good practice).
  - Persists Let’s Encrypt data and logs to host.
  - CrowdSec reads Traefik logs and uses the same `frontend` network.
  - `logrotate` rotates Traefik logs daily and keeps 7 copies.

- `crowdsec/config/acquis.yaml`
  - Tells CrowdSec to read `/var/log/traefik/access.log` with label `type: traefik`.

- `test_conteneur/docker-compose.yml`
  - Example labels for router, TLS resolver, and middleware.
  - Uses `$DOMAIN` for the host rule.

---

## TODO : Security and configuration check (weak points + tips)

- Traefik image tag `traefik:chabichou`:
  - This is the latest as I write these lines. Check back regularly.

- Public email in config:
  - `traefik.yml` contains an email for Let’s Encrypt. If this repo is public, consider moving it to an env var.

- Secrets in plain text:
  - `crowdsecLapiKey` is stored in a file. Consider using an env var or a mounted secret (`${CROWDSEC_LAPI_KEY}`) and reference it in the dynamic file.

- Resource limits and security options:
  - No CPU/RAM limits are set. You can add `deploy.resources.limits` (Swarm) or `--cpus/--memory` (Compose v2) where needed.
  - Consider adding `read_only: true` and dropping capabilities where possible, plus healthchecks.

- Network note:
  - The `frontend` network is marked `external: true`. You must create it before starting services.

- Logs and rotation:
  - Rotation is present. Make sure `traefik_logs` has enough disk space and correct permissions.

- Plugin pinning:
  - The CrowdSec plugin is pinned to `v1.4.5`. Keep it updated to the latest stable when you can.

- CrowdSec AppSec:
  - AppSec is disabled (`crowdsecAppsecEnabled: false`). Enable it only if you deploy the AppSec component and configure it properly.

If all the above points are fine for your context, the setup is OK.

---

## Troubleshooting

- Certificates are not created:
  - Check DNS for your domain points to this server.
  - Ensure port 443 is open and reachable from the Internet.
  - Read Traefik logs in `traefik_logs/access.log` and Traefik container logs.

- CrowdSec shows no metrics:
  - Ensure Traefik writes logs to `/var/log/traefik` in the container.
  - Check that `crowdsec/config/acquis.yaml` points to the correct path.

- 404 or wrong router:
  - Check your labels (spelling matters).
  - Confirm the container is on the `frontend` network.

---

## Source

- https://blog.levassb.ovh/post/crowdsec/
- https://blog.lrvt.de/configuring-crowdsec-with-traefik/

---


## Clean up
```bash
docker compose down
cd test_conteneur && docker compose down
```

---

## License MIT

See `LICENSE`.
