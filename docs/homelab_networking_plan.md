# Secure FastAPI Homelab — Backend Networking Edition

**Revision:** 3.2 — 10 October 2026
**v1.0 target:** tagged by **30 December 2026**
**Primary goal:** learn the networking a strong backend/DevOps engineer should be able to reason about and troubleshoot, using a real FastAPI stack as the workload.
**v1.1:** a separate, deeper networking track that starts only after v1.0 ships.

How to read this plan:

- Every command has an expected outcome. Where the outcome depends on the host (Docker version, firewall backend, userland proxy), the plan says **predict and record** instead of promising a result.
- Milestones are done in the order given in section 23. Blocks have no dates; only the version target does.
- Every block has a **core** and a **stretch** part. Stretch is the first thing cut when time runs short.
- The revision notes at the end list what changed since the previous version and why.

---



# 1. Project Philosophy

The project is deliberately neither a pure FastAPI project nor a miniature network-engineering curriculum.

The center is a realistic backend stack:

```text
client -> Nginx -> FastAPI -> PostgreSQL
                      |
                      +-----> Redis
```

Networking is learned by asking what actually happens when these components communicate.

The core habit:

> **Predict -> test -> capture -> explain.**

Before running a request, predict what should happen. Then run it. If reality differs from the prediction, capture the traffic and explain the difference.

For every important connection, answer these ten questions:

1. Who initiates the connection?
2. What source IP and source port are used?
3. What destination IP and destination port are used?
4. How was the destination name resolved?
5. Which route is selected?
6. Is the destination in the same Layer-2 domain or behind a router?
7. Is NAT involved?
8. Which filtering/segmentation rule allows or blocks the flow?
9. Where is TLS terminated, if anywhere?
10. How can the answer be proven with commands or packet capture?

A corollary that runs through the whole plan:

> **A failed test is only evidence once you know which layer it failed at.**
> "Can't connect" because a name didn't resolve proves something different from "can't connect" because a packet was dropped.

---



# 2. v1.0 Scope Boundary

v1.0 is constrained to **backend-relevant networking**.

## In v1.0

- FastAPI, PostgreSQL, Redis, Nginx, Docker Compose
- Docker user-defined bridge networks with explicit bridge names
- Docker internal networks and what they actually change
- Docker embedded DNS
- container network namespaces, veth pairs, Linux bridges
- Linux network namespaces built by hand: one veth pair, one bridge, one router
- ARP, ICMP, DNS, TCP, HTTP, TLS and Redis RESP captures
- listening sockets vs established connections
- Docker port publishing, and NAT only as deep as Docker's own behavior
- Nginx reverse proxying and timeouts
- trusted forwarding headers and client identity
- client-IP rate limiting in Redis
- TLS termination with a local CA
- telling apart: DNS failure, refused, timeout, no route, proxy timeout, TLS failure
- eight controlled failure drills
- CI security tooling
- diagrams and written explanations



## Explicitly not in v1.0

- multi-router labs, dynamic routing, policy routing
- deliberate asymmetric routing
- manual SNAT/DNAT labs
- stateful firewall design beyond reading Docker's rules
- VLANs, managed switches, inter-VLAN routing
- OPNsense, Suricata
- Kubernetes, overlay networks, macvlan/ipvlan
- Azure implementation work

These belong to **v1.1** (section 27).

The rule:

> If a topic makes you better at answering "why can this backend service not communicate correctly?", it probably belongs in v1.0. If it turns the project into network-engineer training, it moves to v1.1.

---



# 3. Safety Rules

The lab must not accidentally expose deliberately weak services.

During v1.0:

- publish Nginx only on host loopback (`127.0.0.1`)
- reach the stack from another machine only through an SSH tunnel to the host's loopback, never by publishing on a LAN address
- never publish FastAPI, PostgreSQL or Redis
- never expose `vuln/*` branches to the LAN or Internet
- use only dummy data in anything you capture
- never put real credentials into Redis or into requests you capture
- never commit private keys, secrets or raw `.pcap` files (section 22)
- run destructive network experiments only inside namespaces you created, never on the host's real interfaces
- never flush or edit Docker's generated firewall rules

Before deliberately exposing anything to the LAN later, document:

```text
service
listener address and port
published host address and port
expected source network
authentication requirement
TLS state
```

---



# 4. Prerequisites and Version Pins

Several expected results in this plan depend on versions. Record the actual versions in `docs/host-network-baseline.md`.


| Component         | Requirement                                                | Why it matters                                                                                                                                                                                                                                                                  |
| ----------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Docker Engine     | **26.0 or later**                                          | Older engines forward DNS queries out of `internal` networks (CVE-2024-29018), which changes the expected matrix results in section 15.5                                                                                                                                        |
| Docker Compose    | v2 (`docker compose`, not `docker-compose`)                | `depends_on: condition: service_healthy` and top-level `name:`                                                                                                                                                                                                                  |
| Uvicorn           | **0.31.0 or later**, exact version pinned in the lock file | CIDR ranges in `FORWARDED_ALLOW_IPS` were added in 0.31.0. On older versions the subnet string never matches and Experiment A silently behaves like Experiment B. Proxy-header handling has also changed in later releases, so record the exact version in `proxy-client-ip.md` |
| PostgreSQL image  | `postgres:17`                                              | PostgreSQL 18 images changed the data-directory mount path. Do not switch major versions mid-project                                                                                                                                                                            |
| Redis image       | `redis:7-alpine`                                           |                                                                                                                                                                                                                                                                                 |
| Nginx image       | `nginx:stable-alpine`, then pin the exact tag you tested   |                                                                                                                                                                                                                                                                                 |
| Debug image       | `nicolaka/netshoot:v0.16`                                  | All in-container network tools (section 11). Pull it once in advance; change the tag deliberately, not by pulling `latest`                                                                                                                                                      |
| Test client image | `curlimages/curl:8.22.0`                                   | Second client on `edge_net` and the bonus experiment                                                                                                                                                                                                                            |
| Host tools        | section 12                                                 |                                                                                                                                                                                                                                                                                 |




## Lab host decision (setup block)

Decide where the evidence is captured:

- **Option 1:** the t740 is the lab host from the setup block onward. All captures and host-specific findings come from it.
- **Option 2:** the laptop is the lab host. Then run a clean-clone smoke test on the t740 in Block 2, and repeat the host-specific checks of section 15.6 there before writing the final docs.

**Decision (10 October 2026): Option 1.** The host baseline (Milestone 0) is taken on the t740.

Reason: source addresses, firewall backend and Docker version can differ between hosts. A document written on one host and verified on another can contradict itself.

The t740 is headless and managed over SSH and Tailscale. Capture there with `tcpdump -w`, copy the file to the laptop, and open it in Wireshark.

How the t740 was built (Ubuntu Server 26.04 LTS, SSH keys, ufw, Tailscale, Docker Engine): [docs/lab-host-setup.md](../docs/lab-host-setup.md).

---



# 5. Addressing

The lab uses **RFC 1918 private address space**:

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

These are not documentation ranges. The IPv4 documentation ranges (RFC 5737) are used only for fake values such as forged headers:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```



## Docker lab networks

```text
edge_net    172.30.10.0/24
app_net     172.30.20.0/24
db_net      172.30.30.0/24
cache_net   172.30.40.0/24
```

**Known conflict to check:** Docker's default address pools hand out `172.17.0.0/16` through `172.31.0.0/16`. Another Compose project can be given `172.30.0.0/16` automatically. If that happens first, `docker compose up` fails with "Pool overlaps with other one on this address space".

Check before the first `up`:

```bash
docker network inspect $(docker network ls -q) \
  --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}} {{end}}'
ip route
```

If something already uses `172.30.0.0/16`, remove that network or choose another unused `/16` for the whole lab and update every address in this plan consistently.

## Namespace labs

These live in their own namespaces and cannot clash with the host:

```text
veth lab     10.1.1.0/24     ns-left 10.1.1.1, ns-right 10.1.1.2
bridge lab   10.1.2.0/24     ns-h1 10.1.2.11, ns-h2 10.1.2.12
router lab   10.10.10.0/24   client 10.10.10.2, router 10.10.10.1
             10.20.20.0/24   server 10.20.20.2, router 10.20.20.1
```



## Lab hostnames

Use the reserved `.test` top-level domain: `api.lab.test`.

Do not use `.local`. It is reserved for multicast DNS and can cause slow or surprising lookups.

No `/etc/hosts` edits are needed; `curl --resolve` maps the name for each request (section 17.4).

---



# 6. Final v1.0 Topology

```text
                         Host client
                 127.0.0.1:8080 / 127.0.0.1:8443
                              |
                    Docker port publishing
                              |
                          edge_net
                              |
                      +-------+-------+
                      |     Nginx     |
                      | reverse proxy |
                      | TLS boundary  |
                      +-------+-------+
                              |
                          app_net
                         (internal)
                              |
                      +-------+-------+
                      |    FastAPI    |
                      +---+-------+---+
                          |       |
                   db_net |       | cache_net
                (internal)|       | (internal)
                          |       |
                  +-------+--+  +-+---------+
                  |PostgreSQL|  |   Redis   |
                  +----------+  +-----------+
```

Network membership:

```text
nginx  -> edge_net + app_net
api    -> app_net + db_net + cache_net
db     -> db_net only
redis  -> cache_net only
```

Only Nginx has published host ports.

Why `app_net`, `db_net` and `cache_net` are internal:

```text
Nginx may reach outward through edge_net.
FastAPI, PostgreSQL and Redis have no route to the Internet.
FastAPI reaches only its two dependencies.
```

This is a claim to prove, not to assume. Section 15.5 tests every flow **by name and by IP** and records the layer where each blocked flow stops.

**Trust boundary that segmentation does not cover:** `internal: true` does not isolate containers from the Docker host. The Docker host, and anyone with root or Docker access on it, is part of the trusted computing base. The matrix's host -> container rows measure what that means on your host.

---



# 7. Repository Layout

```text
secure-fastapi-lab/
|-- app/
|   |-- main.py
|   |-- config.py            # settings loader with *_FILE support
|   |-- db.py
|   |-- cache.py
|   |-- auth.py
|   `-- routes/
|-- tests/
|-- migrations/              # Alembic
|-- nginx/
|   |-- default.conf
|   `-- certs/.gitkeep       # server.crt + server.key only; generated, never committed
|-- pki/.gitkeep             # lab CA cert + CA private key; never mounted, never committed
|-- scripts/
|   |-- make-secrets.sh
|   |-- make-lab-certs.sh
|   |-- matrix.sh
|   |-- netns-veth.sh
|   |-- netns-bridge.sh
|   |-- netns-router.sh
|   `-- netns-cleanup.sh
|-- secrets/.gitkeep         # generated, never committed
|-- captures/.gitkeep        # raw pcaps never committed
|-- docs/
|-- .github/workflows/
|   |-- ci.yml
|   `-- vuln-report.yml
|-- .gitignore
|-- .dockerignore
|-- Dockerfile
|-- compose.yml
|-- pyproject.toml           # plus a lock file with exact pins
|-- alembic.ini
`-- README.md
```

A clean clone must work with three commands:

```bash
./scripts/make-secrets.sh
./scripts/make-lab-certs.sh      # from Block 4 on
docker compose up -d --build
```

---



# 8. Compose File

This is the complete v1.0 skeleton. Create it in the setup block with all four networks defined from the start, so no later block has to migrate the network layout.

```yaml
name: sfl

services:
  nginx:
    image: nginx:stable-alpine
    container_name: nginx
    ports:
      - "127.0.0.1:8080:80"
      # - "127.0.0.1:8443:443"   # enable in Block 4 together with the TLS server block
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    networks: [edge_net, app_net]
    depends_on:
      api:
        condition: service_healthy
    restart: unless-stopped

  api:
    build: .
    container_name: api
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
    environment:
      DATABASE_HOST: db
      DATABASE_NAME: app
      DATABASE_USER: app
      DATABASE_PASSWORD_FILE: /run/secrets/postgres_password
      REDIS_HOST: redis
      JWT_SECRET_FILE: /run/secrets/jwt_secret
      FORWARDED_ALLOW_IPS: "172.30.20.0/24"
      RATE_LIMIT: "5"
      RATE_WINDOW_SECONDS: "60"
      SLOW_MAX_SECONDS: "30"
      ENABLE_DEBUG_ROUTES: "true"
    secrets: [postgres_password, jwt_secret]
    networks: [app_net, db_net, cache_net]
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]
      interval: 5s
      timeout: 3s
      retries: 12
      start_period: 10s
    restart: unless-stopped

  db:
    image: postgres:17
    container_name: db
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
    secrets: [postgres_password]
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks: [db_net]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 12
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: redis
    networks: [cache_net]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 12
    restart: unless-stopped

volumes:
  pgdata: {}

secrets:
  postgres_password:
    file: ./secrets/postgres_password
  jwt_secret:
    file: ./secrets/jwt_secret

networks:
  edge_net:
    name: sfl_edge
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-edge
    ipam:
      config:
        - subnet: 172.30.10.0/24

  app_net:
    name: sfl_app
    driver: bridge
    internal: true
    driver_opts:
      com.docker.network.bridge.name: br-app
    ipam:
      config:
        - subnet: 172.30.20.0/24

  db_net:
    name: sfl_db
    driver: bridge
    internal: true
    driver_opts:
      com.docker.network.bridge.name: br-db
    ipam:
      config:
        - subnet: 172.30.30.0/24

  cache_net:
    name: sfl_cache
    driver: bridge
    internal: true
    driver_opts:
      com.docker.network.bridge.name: br-cache
    ipam:
      config:
        - subnet: 172.30.40.0/24
```

Before the Redis service exists (setup block), leave out `redis`, its `depends_on` entry and `cache_net` membership, or add Redis on day one. Either works; don't leave a `depends_on` pointing at a missing service.

## Why each non-obvious line is there

- `container_name`**.** Compose would otherwise name containers `sfl-api-1` and so on, and every command in this plan that says `api`, `nginx`, `db` or `redis` would fail.
- **Healthchecks plus** `depends_on: service_healthy`**.** Nginx must not start before FastAPI answers, and FastAPI must not start before PostgreSQL accepts connections. Startup order is then deterministic. This is **startup orchestration only**, not runtime dependency management: if PostgreSQL dies ten minutes later, Compose does not stop or restart FastAPI because of this relationship. Handling that at runtime is the application's job, which is what Drills 3–5 test.
- **The API healthcheck uses Python, not curl.** Slim Python images have no curl. It checks `127.0.0.1` inside the container, which matters in Drill 6.
- `FORWARDED_ALLOW_IPS` **as an environment variable.** Uvicorn reads it directly, so the trusted network is visible in `compose.yml` and changeable per experiment without editing the command.
- **No API port mapping, by design.** FastAPI is an internal service, and all application ingress must pass through Nginx. The architecture does not rely on how Docker handles published ports on internal-only networks.
- **Port 443 commented out until Block 4.** Nginx refuses to start if a `listen 443 ssl` block points at certificate files that don't exist yet.
- `postgres:17` **with** `/var/lib/postgresql/data`**.** That is the correct mount path for this major version.



## Persistence check

`/db-check` runs `SELECT 1`. It proves PostgreSQL is back, not that data survived. Write a marker row and verify that exact row:

```bash
docker compose exec db psql -U app -d app -c \
  "CREATE TABLE IF NOT EXISTS persistence_check (marker text PRIMARY KEY, created_at timestamptz DEFAULT now());
   INSERT INTO persistence_check (marker) VALUES ('before-down-$(date +%s)') RETURNING marker;"

docker compose down        # keeps the pgdata volume
docker compose up -d

docker compose exec db psql -U app -d app -c \
  "SELECT marker, created_at FROM persistence_check ORDER BY created_at DESC LIMIT 1;"
# expect: the same marker the INSERT returned

docker compose down -v     # deletes the volume (only when you mean it)
```

---



# 9. Nginx Configuration

`nginx/default.conf` (mounted as `/etc/nginx/conf.d/default.conf`):

```nginx
# Re-resolve service names through Docker's embedded DNS.
# Without this, Nginx resolves "api" once at startup and keeps
# sending traffic to the old IP after the API container is recreated.
resolver 127.0.0.11 valid=10s ipv6=off;

server {
    listen 80;
    server_name _;

    location / {
        set $api_upstream http://api:8000;
        proxy_pass $api_upstream;

        # Single trusted proxy: overwrite, never append, client-supplied values.
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP         $remote_addr;

        proxy_connect_timeout 3s;
        proxy_read_timeout    10s;
    }
}
```

Port 80 stays in place after TLS is added. It is used for plaintext captures, and there is deliberately no redirect to HTTPS in v1.0.

The TLS server block is added in Block 4 (section 17.4).

Reload after editing:

```bash
docker compose exec nginx nginx -t && docker compose exec nginx nginx -s reload
```

---



# 10. Application Design

Build the smallest useful backend. Every endpoint exists to make one network behavior observable.


| Endpoint                        | Purpose                                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------------- |
| `GET /health`                   | Liveness only. Never touches PostgreSQL or Redis                                            |
| `POST /login`, `GET /protected` | JWT issue and check; at least one authorization test                                        |
| `GET /db-check`                 | Trivial `SELECT 1`; reports success or the error class, never credentials                   |
| `GET /cache-check`              | Redis `PING`                                                                                |
| `GET /limited`                  | Rate-limited by `request.client.host`                                                       |
| `GET /debug/request`            | Returns only `client_host`, `scheme` and `host`; lab-only, enabled by `ENABLE_DEBUG_ROUTES` |
| `GET /slow?seconds=N`           | Waits N seconds, capped at `SLOW_MAX_SECONDS`                                               |




## Behaviors that the experiments depend on

- **Lazy dependency connections.** The app must start even when PostgreSQL or Redis is unreachable, so `/health` keeps working during Drills 1, 4 and 5. Connect on first use, not at import time.
- **Explicit client timeouts.** PostgreSQL connect timeout of 2 seconds; Redis `socket_connect_timeout` and `socket_timeout` of 1 second. Dependency failures must be fast and visible, not hangs.
- **Defined failure behavior for the limiter.** If Redis errors or times out, `/limited` returns **503** (fail-closed). Document the trade-off: many production systems fail open to protect availability.
- `/slow` **uses** `await asyncio.sleep()`**.** `time.sleep()` would block the event loop and stall every other request, which would contaminate the timeout experiments.
- **The settings loader strips trailing newlines** from `*_FILE` contents.
- **The application never reads** `X-Forwarded-For` **itself.** It uses `request.client.host` and lets Uvicorn apply the trust rules.



## Rate limiter

```text
client = request.client.host
window = floor(now / RATE_WINDOW_SECONDS)
key    = "rl:{client}:{window}"

one pipeline, MULTI/EXEC:
    INCR   key
    EXPIRE key RATE_WINDOW_SECONDS
-> count

count > RATE_LIMIT  -> 429
Redis error/timeout -> 503
```

Why the pipeline: `INCR` followed by a separate `EXPIRE` is two round trips; a crash in between leaves a key without expiry. The window index in the key keeps this harmless here, but MULTI/EXEC makes it atomic. This is also a concrete answer to "how do you handle race conditions?", and the transaction shows up nicely in the RESP capture.

It is a learning implementation, not a production limiter.

## Migrations

Run Alembic as a one-off, not at app startup:

```bash
docker compose run --rm api alembic upgrade head
```

---



# 11. Debug Tooling Without Bloating Images

Minimal images (`python:*-slim`, `postgres`, `nginx:alpine`, `redis:alpine`) lack `ip`, `ss`, `tcpdump` and `dig`. Do not install them into the application image.

## Method A — netshoot joined to the target's network namespace

```bash
docker run --rm -it --network container:api nicolaka/netshoot:v0.16
```

Inside:

```bash
ip -br addr
ip route
ip neigh
ss -ntp
dig db
dig redis
curl -v http://api:8000/health
tcpdump -ni any
```

netshoot shares the target's **network namespace**: interfaces, addresses, routes and sockets. `/etc/resolv.conf` is something else. It is a file in the container's filesystem (mount namespace), and sharing a network namespace does not by itself share files.

Docker does, in practice, hand a container started with `--network container:<target>` the same `hosts` and `resolv.conf` files as the target. That is a Docker convenience, not a property of network namespaces. Method B shows the difference: `nsenter -n` joins only the network namespace and keeps the host's `resolv.conf`.

So don't assume, verify:

```bash
docker exec api cat /etc/resolv.conf 2>/dev/null || echo "(no cat in the image)"
docker run --rm --network container:api nicolaka/netshoot:v0.16 cat /etc/resolv.conf
```

For Docker DNS experiments, query `@127.0.0.11` explicitly whenever it matters which resolver answers.

For long sessions or captures with a second terminal, run it detached:

```bash
docker run -d --name dbg-api --network container:api nicolaka/netshoot:v0.16 sleep infinity
docker exec -it dbg-api bash
docker rm -f dbg-api        # when done
```

Capabilities: `tcpdump` works with Docker's default capabilities. Reading firewall rules (`iptables -S`) inside netshoot additionally needs `--cap-add NET_ADMIN`. Method B avoids that.

If the target container is recreated, a netshoot container joined to it loses its namespace. Start a new one.

## Method B — host tools via `nsenter`

```bash
PID=$(docker inspect -f '{{.State.Pid}}' api)

sudo nsenter -t "$PID" -n ip -br addr
sudo nsenter -t "$PID" -n ip route
sudo nsenter -t "$PID" -n ss -ntp
sudo nsenter -t "$PID" -n tcpdump -ni any
sudo nsenter -t "$PID" -n iptables -t nat -S
```

**DNS caveat:** `nsenter -n` enters only the network namespace. Your commands still read the **host's** `/etc/resolv.conf`, so `getent hosts db` or plain `dig db` gives wrong answers. For DNS under nsenter, query Docker's resolver explicitly:

```bash
sudo nsenter -t "$PID" -n dig @127.0.0.11 db
```

Use Method A for anything involving name resolution.

---



# 12. Host Toolbelt

Install on the lab host (Debian/Ubuntu package names):

```bash
sudo apt install \
  iproute2 iputils-ping dnsutils netcat-openbsd tcpdump traceroute \
  nftables conntrack ethtool curl openssl util-linux
```

`netcat-openbsd` matters: `scripts/matrix.sh` parses its messages.

Group commands by the question they answer:

```text
Where am I?                      ip -br addr; ip -br link
Where will this destination go?  ip route; ip route get <ip>
Who is on my Layer-2 segment?    ip neigh; bridge fdb show br <bridge>
Is something listening?          sudo ss -lntp; sudo ss -lnup
Did DNS resolve?                 dig <name>; getent hosts <name>
Can TCP connect?                 nc -vz -w 3 <host> <port>; curl -v <url>
What crossed the interface?      sudo tcpdump -ni <interface> <filter>
```

---



# 13. Milestone 0 — Host Baseline (setup block)

Record the host **before this project's stack exists**. If Docker is already installed, its current state (`docker0`, existing networks, existing rules) is part of the baseline.

```bash
hostnamectl
uname -r
ip -br link
ip -br addr
ip route
ip rule
resolvectl status 2>/dev/null || cat /etc/resolv.conf
sudo ss -lntup
sysctl net.ipv4.ip_forward
iptables --version
sudo iptables -S 2>/dev/null | head -n 50
sudo nft list ruleset 2>/dev/null | head -n 50
lsmod | grep br_netfilter || echo "br_netfilter not loaded"
docker version
cat /etc/docker/daemon.json 2>/dev/null || echo "no daemon.json"
sudo ufw status verbose
tailscale status
docker network inspect $(docker network ls -q) \
  --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}} {{end}}'
```

Two different "backends" are easy to confuse:

- `iptables --version` shows whether the `iptables` command writes to nf_tables or to legacy iptables.
- **Docker's firewall backend** is a separate setting (`"firewall-backend"` in `daemon.json`; absent means Docker's default, iptables).

Save as `docs/host-network-baseline.md`.

On the t740, ufw and Tailscale were set up before this baseline (see `docs/lab-host-setup.md`). Expect a `tailscale0` interface and ufw/Tailscale chains in the firewall output, and record them as pre-existing. Take the baseline after removing any test Compose projects (`docker compose down -v`), so their networks don't show up in the subnet check.

Questions to answer:

- Which interface owns the default route, and what is the gateway?
- Which resolver is configured?
- Is IPv4 forwarding already enabled?
- Which services listen on all interfaces, and which only on loopback?
- Which subnets do existing Docker networks use? Any overlap with `172.30.0.0/16`?
- Is the Docker Engine version 26.0 or later?
- Is `br_netfilter` loaded?

Change nothing yet.

---



# 14. Milestone 1 — Workload (setup block + Block 1)



## Setup block

- repository, `.gitignore` (section 22) before the first commit
- `scripts/make-secrets.sh`
- FastAPI with `/health` and `/db-check`
- settings loader with `*_FILE` support
- `compose.yml` with Nginx, API and PostgreSQL and **all four networks** already defined
- `nginx/default.conf` (port 80 only)
- one DB-free pytest test
- CI: pytest and Gitleaks

Exit check from a fresh `docker compose up -d --build`:

```bash
curl -s http://127.0.0.1:8080/health      # 200
curl -s http://127.0.0.1:8080/db-check    # 200
```



## Block 1

- Redis service, `/cache-check`
- `/limited` with the pipeline limiter and fail-closed 503
- `/debug/request`
- `/slow` with async sleep and the cap
- `POST /login`, `/protected`, one authorization test
- dependency timeouts
- persistence check (section 8)
- Alembic baseline migration
- stretch: Hadolint, Semgrep and Trivy in CI, **report-only**

Exit check:

```bash
for p in health db-check cache-check limited debug/request; do
  printf '%-15s ' "$p"; curl -s -o /dev/null -w '%{http_code}\n' "http://127.0.0.1:8080/$p"
done
curl -s -o /dev/null -w '%{http_code}\n' --max-time 5 "http://127.0.0.1:8080/slow?seconds=2"
```

All return 200.

---



# 15. Milestone 2 — Docker Networking (Block 2)



## 15.1 Watch Docker build the networks

```bash
docker compose down                 # keeps the volume
ip -br link > /tmp/links-before.txt
docker compose up -d
ip -br link > /tmp/links-after.txt
diff /tmp/links-before.txt /tmp/links-after.txt
```

Then:

```bash
ip -br addr show br-edge
ip -br addr show br-app
ip -br addr show br-db
ip -br addr show br-cache
bridge link
```

Questions:

- Which new interfaces appeared, and which bridge is each `veth` attached to?
- What address does Docker give each bridge? Do the internal networks' bridges get one?
- Which containers have more than one interface, and why?



## 15.2 Inspect container namespaces

```bash
for c in nginx api db redis; do
  PID=$(docker inspect -f '{{.State.Pid}}' "$c")
  echo "== $c"; sudo nsenter -t "$PID" -n ip -br addr; sudo nsenter -t "$PID" -n ip route
done
```

Predict before running:

- Nginx has a default route via the `edge_net` gateway.
- API, PostgreSQL and Redis have **no default route**, only connected routes, because all their networks are internal.

Draw `container eth0 -> veth -> br-xxx -> host` for one container. Do not move on until you can explain what the veth pair does.

## 15.3 Docker DNS

From the right namespaces (Method A, because DNS is involved):

```bash
docker run --rm --network container:nginx nicolaka/netshoot:v0.16 dig +short api
docker run --rm --network container:api   nicolaka/netshoot:v0.16 dig +short db
docker run --rm --network container:api   nicolaka/netshoot:v0.16 dig +short redis
docker run --rm --network container:nginx nicolaka/netshoot:v0.16 dig +short db       # predict: empty
docker run --rm --network container:api   nicolaka/netshoot:v0.16 cat /etc/resolv.conf
```

Questions:

- Which resolver address is in the container's `resolv.conf`?
- Why does `db` resolve from the API but not from Nginx?
- Why is "the name resolves" a different claim from "the service is reachable"?



## 15.4 Capture Docker DNS

Docker's embedded resolver (`127.0.0.11`) lives **inside each container's own namespace**. DNS queries never cross `br-app` or any other bridge, so a capture on a bridge shows nothing.

Capture on the container's loopback instead:

```bash
docker run -d --name dbg-api --network container:api nicolaka/netshoot:v0.16 sleep infinity

# terminal 1
docker exec -it dbg-api tcpdump -ni lo -c 20 udp

# terminal 2
docker exec dbg-api dig db
docker exec dbg-api dig example.com      # predict: no answer (internal-only container)

docker rm -f dbg-api
```

Filter on `udp`, not `port 53`. With Docker's iptables backend, queries to `127.0.0.11:53` are NATed to a random high port where dockerd listens. A `port 53` filter can therefore show answers without the matching questions.

See the NAT rules that do it:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' api)
sudo nsenter -t "$PID" -n iptables -t nat -S
# if nothing appears:
sudo nsenter -t "$PID" -n nft list ruleset
```

This is a small, complete NAT example inside one container, and a good warm-up for section 15.6.

Questions:

- Which address and port did the query go to, and where did the answer come from?
- UDP or TCP?
- Why did `example.com` fail from the API? (Docker 26+ stops forwarding queries from internal-only containers to upstream resolvers. Before that fix, DNS was a way out of an "internal" network.)



## 15.5 Communication matrix — by name, by IP, and by layer

Testing blocked flows only by name proves DNS scoping, not segmentation. Nginx can't resolve `db`, so "Nginx -> db" fails before a packet exists. Every blocked flow is therefore tested twice, and the result records **where** it stopped:


| Result     | Meaning                                                        |
| ---------- | -------------------------------------------------------------- |
| `OPEN`     | TCP handshake completed                                        |
| `DNS`      | name did not resolve, no packet sent                           |
| `NO ROUTE` | `Network is unreachable` or `No route to host`                 |
| `REFUSED`  | RST came back: reachable, nothing listening or explicit reject |
| `TIMEOUT`  | no answer: dropped/filtered somewhere                          |


Expected results:


| Flow                            | Expected                   | Why                                                                                                                           |
| ------------------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| host -> 127.0.0.1:8080          | OPEN                       | published port                                                                                                                |
| nginx -> api:8000               | OPEN                       | shared `app_net`                                                                                                              |
| api -> db:5432                  | OPEN                       | shared `db_net`                                                                                                               |
| api -> redis:6379               | OPEN                       | shared `cache_net`                                                                                                            |
| nginx -> db (by name)           | DNS                        | `db` is not on any of Nginx's networks                                                                                        |
| nginx -> db (by IP)             | not OPEN; record the layer | Nginx has a default route via the host; this test exercises Docker's isolation rules, which is the real segmentation evidence |
| nginx -> redis (by name)        | DNS                        | as above                                                                                                                      |
| nginx -> redis (by IP)          | not OPEN; record the layer | as above                                                                                                                      |
| api -> example.com:443 (name)   | DNS                        | internal-only containers get no upstream DNS                                                                                  |
| api -> 1.1.1.1:443 (IP)         | NO ROUTE                   | no default route                                                                                                              |
| db -> 1.1.1.1:443               | NO ROUTE                   | no default route                                                                                                              |
| redis -> 1.1.1.1:443            | NO ROUTE                   | no default route                                                                                                              |
| nginx -> 1.1.1.1:443            | OPEN (needs Internet)      | `edge_net` is not internal; accepted and documented                                                                           |
| host -> db container IP:5432    | predict and record         | "not published" is not the same as "unreachable from the host"                                                                |
| host -> redis container IP:6379 | predict and record         | same                                                                                                                          |
| host -> api container IP:8000   | predict and record         | same                                                                                                                          |


Run it with `scripts/matrix.sh`:

```bash
#!/usr/bin/env bash
# scripts/matrix.sh — run the v1.0 communication matrix.
# Blocked flows are tested by name AND by IP; the result records the failing layer.
# Needs: running stack, nicolaka/netshoot:v0.16 pulled, netcat-openbsd on the host.
set -uo pipefail

NETSHOOT="nicolaka/netshoot:v0.16"

ip_on() {   # ip_on <container> <docker-network-name>
  docker inspect -f "{{with index .NetworkSettings.Networks \"$2\"}}{{.IPAddress}}{{end}}" "$1"
}

classify() {   # classify "<probe output>"
  case "$1" in
    *DNS-FAIL*)                                       echo "DNS" ;;
    *succeeded*)                                      echo "OPEN" ;;
    *refused*)                                        echo "REFUSED" ;;
    *"Network is unreachable"*|*"No route to host"*)  echo "NO ROUTE" ;;
    *"timed out"*)                                    echo "TIMEOUT" ;;
    *)                                                echo "OTHER: ${1//$'\n'/ }" ;;
  esac
}

from_container() {   # from_container <source-container> <name-or-ip> <port>
  docker run --rm --network "container:$1" "$NETSHOOT" sh -c '
    target="$1"; port="$2"
    case "$target" in
      *[!0-9.]*)
        ip=$(dig +short +time=2 +tries=1 @127.0.0.11 "$target" | grep -E "^[0-9.]+$" | head -n1)
        [ -n "$ip" ] || { echo DNS-FAIL; exit 0; } ;;
      *) ip="$target" ;;
    esac
    nc -vz -w 3 "$ip" "$port" 2>&1' _ "$2" "$3"
}

from_host() {   # from_host <ip> <port>
  nc -vz -w 3 "$1" "$2" 2>&1
}

check() {   # check <label> <expected> <actual>
  local mark
  case "$2" in
    PREDICT)  mark="record" ;;
    NOT-OPEN) if [[ $3 != OPEN ]]; then mark="ok"; else mark="UNEXPECTED"; fi ;;
    *)        if [[ $3 == "$2" ]]; then mark="ok"; else mark="UNEXPECTED"; fi ;;
  esac
  printf '%-34s %-10s %-24s %s\n' "$1" "$2" "$3" "$mark"
}

DB_IP=$(ip_on db sfl_db)
REDIS_IP=$(ip_on redis sfl_cache)
API_IP=$(ip_on api sfl_app)

printf '%-34s %-10s %-24s %s\n' "FLOW" "EXPECTED" "ACTUAL" "RESULT"

check "host -> 127.0.0.1:8080"        OPEN       "$(classify "$(from_host 127.0.0.1 8080)")"
check "nginx -> api:8000"             OPEN       "$(classify "$(from_container nginx api 8000)")"
check "api -> db:5432"                OPEN       "$(classify "$(from_container api db 5432)")"
check "api -> redis:6379"             OPEN       "$(classify "$(from_container api redis 6379)")"

check "nginx -> db (name)"            DNS        "$(classify "$(from_container nginx db 5432)")"
check "nginx -> db (IP $DB_IP)"       NOT-OPEN   "$(classify "$(from_container nginx "$DB_IP" 5432)")"
check "nginx -> redis (name)"         DNS        "$(classify "$(from_container nginx redis 6379)")"
check "nginx -> redis (IP $REDIS_IP)" NOT-OPEN   "$(classify "$(from_container nginx "$REDIS_IP" 6379)")"

check "api -> example.com:443"        DNS        "$(classify "$(from_container api example.com 443)")"
check "api -> 1.1.1.1:443"            "NO ROUTE" "$(classify "$(from_container api 1.1.1.1 443)")"
check "db -> 1.1.1.1:443"             "NO ROUTE" "$(classify "$(from_container db 1.1.1.1 443)")"
check "redis -> 1.1.1.1:443"          "NO ROUTE" "$(classify "$(from_container redis 1.1.1.1 443)")"
check "nginx -> 1.1.1.1:443"          OPEN       "$(classify "$(from_container nginx 1.1.1.1 443)")"

check "host -> db IP:5432"            PREDICT    "$(classify "$(from_host "$DB_IP" 5432)")"
check "host -> redis IP:6379"         PREDICT    "$(classify "$(from_host "$REDIS_IP" 6379)")"
check "host -> api IP:8000"           PREDICT    "$(classify "$(from_host "$API_IP" 8000)")"
```

Rules for the results:

- Write your prediction for every `PREDICT` and `NOT-OPEN` row **before** running the script.
- Any `UNEXPECTED` row is a finding, not a script bug. Explain it in `docker-networking.md` before moving on.
- If `api -> example.com` shows anything other than `DNS`, check the Docker Engine version first (section 4).
- Keep the script output in the doc. It is the core evidence for the segmentation claims.



## 15.6 Port publishing and NAT

v1.0 needs enough NAT to explain Docker's behavior, not a NAT course.

Predict before running `curl http://127.0.0.1:8080/debug/request`:

```text
Which address does the host connect to?
Which process accepts that connection on the host?
Does Docker translate the destination? Where?
Which source address will Nginx see?
Which bridge carries the container-side traffic?
```

Then verify:

```bash
sudo ss -lntp | grep 8080
sudo tcpdump -ni lo -c 10 'tcp port 8080'       # terminal 1
sudo tcpdump -ni br-edge -c 10 'tcp port 80'    # terminal 2
curl -s http://127.0.0.1:8080/debug/request     # terminal 3
docker logs --tail 5 nginx                      # $remote_addr as Nginx saw it
```

Read Docker's rules without changing them:

```bash
sudo iptables -t nat -S
sudo iptables -S
# if no DOCKER chains appear, the rules may be in the other iptables variant or in nftables:
sudo iptables-legacy -t nat -S 2>/dev/null
sudo iptables-nft -t nat -S 2>/dev/null
sudo nft list ruleset
```

Do not encode one source-IP result as universally expected. It depends on the Docker Engine version, the firewall backend, whether the userland proxy is enabled, and where the client is (loopback, another container, another machine). The objective is to explain **your host's actual path**, captured on the host named in section 4.

Concepts you must be able to explain afterwards:

```text
DNAT        changes destination address/port
SNAT        changes source address
masquerade  SNAT using the outgoing interface's address
conntrack   remembers connections so replies are translated back
```

Comparison (core): a second client on `edge_net`:

```bash
docker run --rm --network sfl_edge curlimages/curl:8.22.0 -s http://nginx/debug/request
```

Predict and record the source address Nginx sees for both clients. Section 17.2 reuses both clients.

Stretch: a LAN machine against a temporarily `0.0.0.0`-published port. Revert immediately afterwards.

## Deliverable

`docs/docker-networking.md`:

- bridge list and network membership table
- route tables for all four containers
- Docker DNS explanation and the loopback DNS capture
- matrix script output, with every `PREDICT`/`UNEXPECTED` row explained
- port-publishing path as observed, with captures
- one packet-flow diagram: host -> Nginx -> API

---



# 16. Milestone 3 — Linux Primitives and Plaintext Captures (Block 3)

Build each lab **by hand first**. Once it works, save the commands as the matching script in `scripts/`. All three labs live in namespaces you create, so Docker's host rules never touch them.

## 16.1 Two namespaces and a veth pair

```bash
sudo ip netns add ns-left
sudo ip netns add ns-right
sudo ip link add veth-l type veth peer name veth-r
sudo ip link set veth-l netns ns-left
sudo ip link set veth-r netns ns-right
sudo ip -n ns-left  addr add 10.1.1.1/24 dev veth-l
sudo ip -n ns-right addr add 10.1.1.2/24 dev veth-r
sudo ip -n ns-left  link set lo up
sudo ip -n ns-left  link set veth-l up
sudo ip -n ns-right link set lo up
sudo ip -n ns-right link set veth-r up

sudo ip -n ns-left route                       # predict: one connected route
sudo ip netns exec ns-left ping -c 3 10.1.1.2  # predict before running
```

Learn: namespace creation, moving interfaces, link state, connected routes.

## 16.2 A Linux bridge inside its own switch namespace

Do **not** create the teaching bridge in the root namespace. Docker usually sets the host's `FORWARD` policy to `DROP` and loads `br_netfilter`, so traffic across a root-namespace bridge can be silently dropped. A bridge inside `ns-sw` has an empty, permissive ruleset of its own.

```bash
for ns in ns-sw ns-h1 ns-h2; do sudo ip netns add "$ns"; done

sudo ip -n ns-sw link add br0 type bridge

sudo ip link add h1-eth0 type veth peer name sw-h1
sudo ip link add h2-eth0 type veth peer name sw-h2
sudo ip link set h1-eth0 netns ns-h1
sudo ip link set h2-eth0 netns ns-h2
sudo ip link set sw-h1 netns ns-sw
sudo ip link set sw-h2 netns ns-sw

sudo ip -n ns-sw link set sw-h1 master br0
sudo ip -n ns-sw link set sw-h2 master br0
sudo ip -n ns-sw link set br0 up
sudo ip -n ns-sw link set sw-h1 up
sudo ip -n ns-sw link set sw-h2 up

sudo ip -n ns-h1 addr add 10.1.2.11/24 dev h1-eth0
sudo ip -n ns-h2 addr add 10.1.2.12/24 dev h2-eth0
for ns in ns-h1 ns-h2; do sudo ip -n "$ns" link set lo up; done
sudo ip -n ns-h1 link set h1-eth0 up
sudo ip -n ns-h2 link set h2-eth0 up

sudo ip netns exec ns-h1 ping -c 3 10.1.2.12
```

Inspect:

```bash
sudo ip -n ns-sw link show master br0
sudo bridge -n ns-sw fdb show br br0
sudo ip -n ns-h1 neigh
sudo ip -n ns-h2 neigh
```

ARP capture:

```bash
sudo ip -n ns-h1 neigh flush all
sudo ip netns exec ns-h1 tcpdump -eni h1-eth0 -c 4 arp      # terminal 1
sudo ip netns exec ns-h1 ping -c 1 10.1.2.12                # terminal 2
```

Questions:

- What does the ARP table hold, and what does the bridge FDB hold?
- Why are they separate tables in separate places?
- Why is no router needed here?



## 16.3 One router namespace

```bash
for ns in ns-client ns-router ns-server; do sudo ip netns add "$ns"; done

sudo ip link add c-eth0 type veth peer name r-client
sudo ip link add s-eth0 type veth peer name r-server
sudo ip link set c-eth0   netns ns-client
sudo ip link set r-client netns ns-router
sudo ip link set r-server netns ns-router
sudo ip link set s-eth0   netns ns-server

sudo ip -n ns-client addr add 10.10.10.2/24 dev c-eth0
sudo ip -n ns-router addr add 10.10.10.1/24 dev r-client
sudo ip -n ns-router addr add 10.20.20.1/24 dev r-server
sudo ip -n ns-server addr add 10.20.20.2/24 dev s-eth0

for ns in ns-client ns-router ns-server; do sudo ip -n "$ns" link set lo up; done
sudo ip -n ns-client link set c-eth0 up
sudo ip -n ns-router link set r-client up
sudo ip -n ns-router link set r-server up
sudo ip -n ns-server link set s-eth0 up

sudo ip -n ns-client route add default via 10.10.10.1
sudo ip -n ns-server route add default via 10.20.20.1
```

The "before" test has to start from a known state. A new namespace may inherit forwarding settings from the host, depending on the kernel, so read the value and then set it explicitly:

```bash
sudo ip netns exec ns-router sysctl net.ipv4.ip_forward
sudo ip netns exec ns-router sysctl -w net.ipv4.ip_forward=0
sudo ip netns exec ns-client ping -c 3 -W 1 10.20.20.2      # predict: fails

sudo ip netns exec ns-router sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec ns-client ping -c 3 10.20.20.2           # predict: works
```

ICMP capture on both router interfaces:

```bash
sudo ip netns exec ns-router tcpdump -envi r-client -c 4 icmp   # terminal 1
sudo ip netns exec ns-router tcpdump -envi r-server -c 4 icmp   # terminal 2
sudo ip netns exec ns-client ping -c 2 10.20.20.2               # terminal 3
```

Observe and explain:

- Ethernet source/destination MACs change at the routed hop.
- IP source/destination stay end to end (no NAT here).
- TTL is one lower on `r-server` than on `r-client` (Linux sends with TTL 64).

This is the full routing depth for v1.0. Multi-router labs are v1.1.

## 16.4 Cleanup

```bash
#!/usr/bin/env bash
# scripts/netns-cleanup.sh — deleting a namespace also deletes the veths inside it
for ns in ns-left ns-right ns-sw ns-h1 ns-h2 ns-client ns-router ns-server; do
  sudo ip netns del "$ns" 2>/dev/null || true
done
ip netns list
```



## 16.5 Plaintext captures on the Docker stack

TCP handshake to Nginx:

```bash
sudo tcpdump -ni br-edge -c 10 'tcp port 80'          # terminal 1
curl -s http://127.0.0.1:8080/health                  # terminal 2
```

Identify SYN, SYN-ACK, ACK, the request, and FIN or RST at close.

HTTP between Nginx and the API:

```bash
sudo tcpdump -ni br-app -A -c 20 'tcp port 8000'      # terminal 1
curl -s http://127.0.0.1:8080/debug/request           # terminal 2
```

The HTTP request is readable, including the `X-Forwarded-For` header Nginx wrote. Note that this is a **new** TCP connection, not the client's connection continued.

Redis RESP:

```bash
sudo tcpdump -ni br-cache -A -c 20 'tcp port 6379'    # terminal 1
curl -s http://127.0.0.1:8080/limited                 # terminal 2
```

Find `MULTI`, `INCR`, `EXPIRE` and `EXEC`, and the replies.

Questions:

- Which side opens TCP/6379, and with which source port?
- Redis has no password in this lab. What controls who can reach it, and who can reach it regardless of segmentation? (The Docker host is part of the trusted computing base, section 6.)
- Why does segmentation still matter when the protocol is plaintext?

Redis TLS is a v1.1 exercise. Plaintext is intentional here.

## Deliverables

- `docs/linux-network-primitives.md`: three diagrams, commands, ARP/FDB comparison, the routed ping explained hop by hop
- `docs/packet-captures.md` part 1: ARP, ICMP, TCP handshake, HTTP, RESP, plus the DNS capture from Block 2

For each capture write: prediction, interface chosen and why, what was observed, explanation.

---



# 17. Milestone 4 — Proxy Identity, TLS and Timeouts (Block 4)



## 17.1 TCP peer vs logical client

```text
client --TCP 1--> Nginx --TCP 2--> FastAPI
```

FastAPI's TCP peer is always Nginx. The real client is known only through a header that Nginx writes and that Uvicorn decides whether to trust:

- Nginx **overwrites** `X-Forwarded-For` with `$remote_addr` (section 9), so a client-supplied value never passes through.
- Uvicorn trusts forwarding headers only from `FORWARDED_ALLOW_IPS=172.30.20.0/24`, the network that only Nginx and FastAPI share.

Check the version requirement (section 4) and the membership assumption:

```bash
docker compose exec api python -c "import uvicorn; print(uvicorn.__version__)"
docker network inspect sfl_app --format '{{range .Containers}}{{.Name}} {{end}}'   # expect: nginx api
```



## 17.2 Two-client rate-limit test

A single client cannot show the difference between correct and broken trust, because with one client both look the same. Use the two clients from section 15.6.

```bash
docker exec redis redis-cli FLUSHDB      # lab-only reset; cache_net holds only limiter keys

# client 1: host via the published port
for i in 1 2 3 4 5 6; do
  curl -s -o /dev/null -w '%{http_code} ' http://127.0.0.1:8080/limited
done; echo

# client 2: a container on edge_net
docker run --rm --network sfl_edge curlimages/curl:8.22.0 \
  -s -o /dev/null -w '%{http_code}\n' http://nginx/limited

docker exec redis redis-cli --scan --pattern 'rl:*'
```

Run the three commands back to back. The fixed window is 60 seconds. If a result looks wrong, check whether the window rolled over between commands (`TTL` on the key) and repeat.

## 17.3 Experiments



### Experiment A — correct trust (main branch)

Expected: client 1 gets `200 200 200 200 200 429`; client 2 gets `200`; Redis has two keys with two different client addresses.

### Experiment B — Uvicorn does not trust Nginx

Set `FORWARDED_ALLOW_IPS: "127.0.0.1"` (loopback only, which is effectively Uvicorn's default) and recreate the API:

```bash
docker compose up -d api
```

Expected: client 2 now gets `429`. Both clients share one key, keyed on Nginx's `app_net` address. Everyone behind the proxy is one user.

Restore `172.30.20.0/24` afterwards.

### Experiment C — spoofable trust (branch `vuln/forwarded-header-trust`)

The spoof only works when a forged value **survives Nginx** and **Uvicorn is told to believe it**. Both changes are needed on the branch:

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;   # append instead of overwrite
```

```yaml
FORWARDED_ALLOW_IPS: "*"
```

Attack:

```bash
docker exec redis redis-cli FLUSHDB
for i in $(seq 1 10); do
  curl -s -o /dev/null -w '%{http_code} ' \
    -H "X-Forwarded-For: 198.51.100.$i" http://127.0.0.1:8080/limited
done; echo
docker exec redis redis-cli --scan --pattern 'rl:*'
```

Expected: ten `200`s and ten keys with forged addresses. The limiter is bypassed.

Then show that **either fix alone closes it** in this topology, and explain why main keeps both:

- **Fix 1 only, Nginx overwrites:** the forged entry never reaches Uvicorn.
- **Fix 2 only, Uvicorn trusts the subnet:** Uvicorn walks the list from the right and takes the first untrusted entry, which is the address Nginx appended, not the forged one. This works **only because** the address Nginx appends (the client on the edge side) is outside `172.30.20.0/24`. A client inside the trusted range could still spoof. That is what the bonus experiment below shows, and why main keeps both fixes.



### Optional bonus — subnet trust is membership trust

Trusting `172.30.20.0/24` means trusting everything that can join `sfl_app`:

```bash
docker run --rm --network sfl_app curlimages/curl:8.22.0 -s \
  -H 'X-Forwarded-For: 198.51.100.99' http://api:8000/debug/request
```

Expected on the main config: `client_host` is `198.51.100.99`. That is why the membership check in 17.1 belongs in the proof.

The forwarded-header vulnerability write-up (section 21) **is** Experiment C. Write it once in `docs/vulnerabilities/forwarded-header-trust.md` and link to it from `proxy-client-ip.md`.

## 17.4 TLS termination

`scripts/make-lab-certs.sh` creates a local CA and a server certificate for `api.lab.test`.

The CA and the server certificate live in different places on purpose:

- `pki/` holds the CA certificate and the **CA private key**. It is never mounted into any container.
- `nginx/certs/` holds only `server.crt` and `server.key`, which is all Nginx needs. It is the only directory Nginx mounts.

If the CA key sat in the mounted directory, anyone who compromised Nginx could issue certificates the lab trusts. Give each service only the secrets it needs.

```bash
#!/usr/bin/env bash
# scripts/make-lab-certs.sh — lab CA in pki/ (never mounted), server cert in nginx/certs/
set -euo pipefail
cd "$(dirname "$0")/.."

PKI=pki
CERTS=nginx/certs
mkdir -p "$PKI" "$CERTS"
chmod 700 "$PKI"

if [ -s "$CERTS/server.crt" ] && [ -s "$CERTS/server.key" ]; then
  echo "server cert already exists in $CERTS"; exit 0
fi

if [ ! -s "$PKI/lab-ca.key" ]; then
  openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
    -keyout "$PKI/lab-ca.key" -out "$PKI/lab-ca.crt" -subj "/CN=secure-fastapi-lab CA"
  chmod 600 "$PKI/lab-ca.key"
fi

openssl req -newkey rsa:2048 -nodes \
  -keyout "$CERTS/server.key" -out "$PKI/server.csr" -subj "/CN=api.lab.test"

printf '%s\n' \
  "subjectAltName=DNS:api.lab.test" \
  "basicConstraints=CA:FALSE" \
  "keyUsage=digitalSignature,keyEncipherment" \
  "extendedKeyUsage=serverAuth" > "$PKI/server.ext"

openssl x509 -req -in "$PKI/server.csr" -CA "$PKI/lab-ca.crt" -CAkey "$PKI/lab-ca.key" \
  -CAcreateserial -out "$CERTS/server.crt" -days 365 -extfile "$PKI/server.ext"

rm -f "$PKI/server.csr" "$PKI/server.ext"
echo "CA in $PKI/ (not mounted); server.crt + server.key in $CERTS/"
```

Clients verify against `pki/lab-ca.crt`.

Add the TLS server block to `nginx/default.conf`:

```nginx
server {
    listen 443 ssl;
    server_name api.lab.test;

    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    location / {
        set $api_upstream http://api:8000;
        proxy_pass $api_upstream;

        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP         $remote_addr;

        proxy_connect_timeout 3s;
        proxy_read_timeout    10s;
    }
}
```

Uncomment `127.0.0.1:8443:443` in `compose.yml`, then:

```bash
./scripts/make-lab-certs.sh
docker compose up -d nginx

curl --cacert pki/lab-ca.crt \
  --resolve api.lab.test:8443:127.0.0.1 \
  https://api.lab.test:8443/debug/request          # predict: scheme "https"

openssl s_client -connect 127.0.0.1:8443 -servername api.lab.test \
  -CAfile pki/lab-ca.crt -verify_hostname api.lab.test </dev/null
```

Learn: subject/SAN, issuer, chain, trust store, hostname validation, SNI, and the TLS handshake as a separate step from the HTTP request.

Note: `openssl s_client` without `-verify_hostname` completes the handshake even when the name doesn't match. It checks the chain, not the hostname. Drill 8 uses this.

## 17.5 TLS capture

```bash
sudo tcpdump -ni br-edge -w captures/tls13.pcap 'tcp port 443'     # terminal 1
curl -s --cacert pki/lab-ca.crt --resolve api.lab.test:8443:127.0.0.1 \
  https://api.lab.test:8443/health                                 # terminal 2
```

With TLS 1.3, likely the default, you will see the ClientHello (including SNI) and ServerHello in clear. **The certificate is encrypted** in TLS 1.3.

To see the certificate on the wire, repeat with TLS 1.2:

```bash
sudo tcpdump -ni br-edge -w captures/tls12.pcap 'tcp port 443'
curl -s --tls-max 1.2 --cacert pki/lab-ca.crt \
  --resolve api.lab.test:8443:127.0.0.1 https://api.lab.test:8443/health
```

Open both files in Wireshark and compare what each version hides. Compare both with the plaintext HTTP and RESP captures from Block 3.

## 17.6 Timeouts

Current settings: Nginx connect timeout 3 s, read timeout 10 s; `/slow` capped at 30 s.

```bash
# successful TCP + slow application, inside the proxy budget
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' "http://127.0.0.1:8080/slow?seconds=3"

# proxy timeout: predict 504 after ~10 s
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' "http://127.0.0.1:8080/slow?seconds=15"

# client timeout first: predict curl exit 28; Nginx logs 499 (client closed request)
curl -s -o /dev/null --max-time 5 "http://127.0.0.1:8080/slow?seconds=8"; echo "curl exit $?"
docker logs --tail 5 nginx
```

Build this table for `docs/proxy-client-ip.md` or `troubleshooting.md`:

```text
symptom              who gave up   status/error   where it shows up
connection refused   ...
connect timeout      ...
slow app, in budget  ...
proxy read timeout   ...
client timeout       ...
DNS failure          ...
TLS failure          ...
```



## Deliverables

- `docs/proxy-client-ip.md`: TCP peer vs logical client, Nginx policy, Uvicorn version and trust configuration, Experiments A–C, rate-limit consequences, the timeout table
- `docs/packet-captures.md` part 2: TLS 1.3 vs 1.2

---



# 18. Milestone 5 — The Eight Failure Drills (Block 5)

Eight distinct drills, no duplicates of Block 4's experiments.

For every drill, record:

```text
symptom
prediction
first hypothesis
commands used (section 19 order)
capture or log evidence
root cause (which layer)
fix
how to recognize it faster next time
```

Restore the stack after each drill:

```bash
git checkout -- compose.yml nginx/default.conf
docker compose up -d --force-recreate
```



### Drill 1 — wrong service name for PostgreSQL

Set `DATABASE_HOST: db-wrong`, then `docker compose up -d api`.

Expect: `/health` 200 (lazy connect); `/db-check` fails quickly; the app log shows a name-resolution error. Layer: DNS.

### Drill 2 — Nginx points at the wrong API port

Change the upstream to `http://api:8001` and reload Nginx.

Expect: 502; the Nginx error log shows `connect() failed (111: Connection refused)`. Name and route are fine; nothing listens on 8001, so an RST comes back. Layer: transport.

### Drill 3 — API container stopped

```bash
docker compose stop api
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/health
docker logs --tail 5 nginx
docker compose start api
```

Expect: 502. Once the resolver cache expires (10 s), the Nginx log shows the name could not be resolved, because a stopped container leaves Docker DNS. Inside the cache window you may instead see a connect error to the old IP. Record which one you got. Compare with Drill 2: same status code, different layer.

### Drill 4 — API disconnected from `db_net`

```bash
docker network disconnect sfl_db api
curl -s http://127.0.0.1:8080/db-check
docker network connect sfl_db api
```

Expect: `/db-check` fails; `/health` and `/limited` still work. Inspect the API's routes and DNS while disconnected. Learn: network membership defines both DNS scope and reachability.

### Drill 5 — API disconnected from `cache_net`

Same pattern with `sfl_cache`.

Expect: `/limited` returns 503 within about 1 s (fail-closed with timeouts); `/health` and `/db-check` still work. Learn: partial dependency failure and why client timeouts matter.

### Drill 6 — API listening on the wrong address

Change the command to `--host 127.0.0.1`, then `docker compose up -d api`.

Expect:

- `docker ps` shows the API as **healthy**, because the healthcheck connects from inside the container.
- Nginx returns 502 with `111: Connection refused`.
- `ss -lntp` in the API's namespace shows `127.0.0.1:8000`, not `0.0.0.0:8000`.

Same symptom as Drill 2, different root cause. Lesson: a healthcheck proves only what it tests.

### Drill 7 — proxy timeout

Covered in section 17.6, now as a full drill: capture on `br-app` and correlate with the Nginx error log (`upstream timed out (110: ...) while reading response header`).

Then fix it properly: decide whether the endpoint should be faster or the timeout longer, and justify it.

### Drill 8 — TLS hostname mismatch

```bash
curl --cacert pki/lab-ca.crt \
  --resolve wrong.lab.test:8443:127.0.0.1 https://wrong.lab.test:8443/health
openssl s_client -connect 127.0.0.1:8443 -servername wrong.lab.test \
  -CAfile pki/lab-ca.crt </dev/null
```

Expect: curl fails with a certificate name mismatch. `s_client` completes the handshake and reports the chain as OK, because it doesn't check the hostname without `-verify_hostname`. TCP works, TLS negotiates, validation fails; the application is never involved.

### Optional replacement drill

Delete the default route in `ns-client` (router lab) and diagnose from the client side only.

---



# 19. Standard Troubleshooting Workflow

Use the same order every time. For anything inside a container, use netshoot (Method A), or `nsenter` with `dig @127.0.0.11` for DNS.

1. **State the intended flow:** source container, destination name, destination port, expected network. Example: `api -> redis:6379 over cache_net`.
2. **Resolve the name:** `dig redis` (netshoot) or `nsenter ... dig @127.0.0.11 redis`. If it fails, stop: the problem is DNS or network membership.
3. **Check local addresses:** `ip -br addr`.
4. **Check route selection:** `ip route get <destination-ip>`.
5. **Check neighbors** on the same L2 segment: `ip neigh`.
6. **Check the listener** on the destination: `ss -lntp`. Note the address as well as the port.
7. **Test the transport:** `nc -vz -w 3 <ip> <port>`, and read the exact error.
8. **Read the filtering** (read only): `docker network inspect`, `iptables -S`, `nft list ruleset`.
9. **Capture:**
  ```text
   no SYN leaves         -> local: resolution, route, or local filter
   SYN leaves, no reply  -> path or filter drop, or host down
   RST returns           -> reachable; nothing listening or explicit reject
   handshake completes   -> move up to TLS and the application
  ```
10. **TLS and protocol:** `openssl s_client`, `curl -v`.
11. **Application logs**, only after the layers below are proven.

---



# 20. Known Gotchas (consolidated)


| Gotcha                                                               | Where it bites                                         | Prevention                                                                      |
| -------------------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------- |
| Compose container names like `sfl-api-1`                             | every `docker inspect api`                             | `container_name` (section 8)                                                    |
| Slim images have no network tools                                    | every in-container command                             | netshoot or `nsenter` (section 11)                                              |
| `nsenter -n` uses the host's `resolv.conf`                           | DNS checks                                             | netshoot, or `dig @127.0.0.11`                                                  |
| netshoot needs `NET_ADMIN` to read iptables                          | firewall reads                                         | `nsenter` from the host instead                                                 |
| Docker DNS never crosses a bridge                                    | DNS capture                                            | capture `lo` in the container, filter `udp` (15.4)                              |
| Testing blocked flows only by name                                   | segmentation proof                                     | by name **and** IP, with layer (15.5)                                           |
| Older Docker forwards DNS from internal networks                     | matrix results                                         | Docker 26.0+ (section 4)                                                        |
| `172.30.0.0/16` is in Docker's default pool                          | first `compose up`                                     | check existing subnets (section 5)                                              |
| Docker's FORWARD DROP and `br_netfilter`                             | bridge lab in the root namespace                       | bridge inside `ns-sw` (16.2)                                                    |
| New namespaces may inherit `ip_forward`                              | router "before" test                                   | set it to 0 explicitly (16.3)                                                   |
| Nginx caches upstream IPs from startup                               | 502s after the API is recreated                        | `resolver 127.0.0.11` + variable upstream (section 9)                           |
| `listen 443 ssl` with missing certs                                  | Nginx won't start                                      | TLS block and port added in Block 4                                              |
| TLS 1.3 encrypts the certificate                                     | TLS capture                                            | compare with `--tls-max 1.2` (17.5)                                             |
| `.local` is mDNS                                                     | lab hostname                                           | `api.lab.test` with `--resolve`                                                 |
| Uvicorn below 0.31 ignores CIDR trust                                | Experiment A                                           | pin and verify the version (17.1)                                               |
| Single-client rate-limit test                                        | Experiment B                                           | two clients (17.2)                                                              |
| Fixed-window boundary during a test                                  | flaky 429s                                             | run back to back; check `TTL`                                                   |
| `time.sleep` in async endpoints                                      | timeout experiments                                    | `asyncio.sleep`                                                                 |
| Blocking app startup on dependencies                                 | Drills 1, 4, 5                                         | lazy connections                                                                |
| Healthcheck tests loopback only                                      | false "healthy"                                        | understood in Drill 6                                                           |
| Secret files unreadable by container users                           | PostgreSQL init fails                                  | dir 700, files 644 (section 22)                                                 |
| `POSTGRES_PASSWORD_FILE` only applies at first init                  | auth fails after regenerating secrets                  | never regenerate an existing secret; `make-secrets.sh` skips existing files     |
| Trailing newline in secret files                                     | auth or JWT mismatch                                   | `printf '%s'` when writing; strip when reading                                  |
| Committed private keys                                               | Gitleaks failures, real leak                           | `nginx/certs/*` and `pki/*` ignored; generated by script                        |
| CA private key inside a mounted directory                            | a compromised Nginx could issue trusted certs          | CA lives in `pki/`, which is never mounted (17.4)                               |
| `/db-check` used as a persistence test                               | false confidence: `SELECT 1` proves nothing about data | marker row before and after (section 8)                                         |
| `depends_on` read as runtime supervision                             | expecting Compose to react when PostgreSQL dies later  | it's startup ordering only; the app handles runtime failure (section 8)         |
| Assuming netshoot "shares `resolv.conf`" because it shares the netns | wrong mental model of namespaces                       | verify with `cat /etc/resolv.conf`; query `@127.0.0.11` explicitly (section 11) |
| Unpinned debug images                                                | results drift between runs                             | `netshoot:v0.16`, `curl:8.22.0` (section 4)                                     |
| PostgreSQL 18 mount path change                                      | data "disappears"                                      | stay on `postgres:17`                                                           |
| Docker rules in the other iptables variant                           | empty `iptables -S`                                    | check `iptables-legacy`, `iptables-nft`, `nft`                                  |


Never "fix" anything by flushing Docker's rules.

---



# 21. Security and CI

CI is a supporting track, but it starts early so it never surprises the deadline.


| When                                   | What                                                                         |
| -------------------------------------- | ---------------------------------------------------------------------------- |
| Setup block                            | pytest and **Gitleaks** (from the first commit)                              |
| Block 1 (stretch; at the latest Block 2) | **Hadolint**, **Semgrep**, **Trivy**, all **report-only**                    |
| Block 6                                 | triage findings, fix or document accepted risks, switch main to **blocking** |


Practical notes:

- Pin third-party GitHub Actions to a full commit SHA, not a tag.
- The Gitleaks GitHub Action requires a license key for repositories owned by an organization. A personal repository doesn't need one. Alternatively, run the Gitleaks CLI directly in a job.
- Give Semgrep an explicit ruleset (for example `p/python`) so CI doesn't depend on an account login.
- Trivy scans the image CI builds from the `Dockerfile`.
- Tests are DB-free and set settings via environment variables, so CI needs no secrets files.
- Branches matching `vuln/*` run a separate report-only workflow (`vuln-report.yml`) and never deploy anywhere.



## Vulnerability experiments for v1.0

Exactly two:

1. `vuln/forwarded-header-trust`: this is Experiment C (17.3). Document it once.
2. `vuln/sql-injection`: one deliberately raw-SQL search endpoint on the branch only.

For each, document:

1. change
2. reproduction
3. observable symptom
4. scanner result
5. why the scanners did or did not catch it
6. fix
7. regression test on main

Other vulnerability ideas (JWT validation, IDOR) are v1.1.

---



# 22. Secrets and Repository Hygiene

`scripts/make-secrets.sh`:

```bash
#!/usr/bin/env bash
# scripts/make-secrets.sh — generate lab secrets once; never overwrite existing ones
set -euo pipefail
cd "$(dirname "$0")/.."

mkdir -p secrets
chmod 700 secrets

for name in postgres_password jwt_secret; do
  if [ ! -s "secrets/$name" ]; then
    printf '%s' "$(openssl rand -hex 32)" > "secrets/$name"
  fi
  chmod 644 "secrets/$name"
done

echo "secrets ready: directory 700, files 644"
```

Why these permissions:

- Compose file secrets are bind mounts that keep the host file's ownership and mode.
- PostgreSQL reads its password file as the `postgres` user, and the API runs as a non-root user. Both need read access, so the files are 644.
- The **directory** is 700, so other host users cannot reach the files. Inside the container the host directory's mode doesn't apply.

Why the script never overwrites: `POSTGRES_PASSWORD_FILE` is only used when the data volume is first initialized. A regenerated password would no longer match the database.

`.gitignore` from the first commit:

```gitignore
.env
secrets/*
!secrets/.gitkeep
nginx/certs/*
!nginx/certs/.gitkeep
pki/*
!pki/.gitkeep
captures/*.pcap
captures/*.pcapng
```

Captures may contain request data. Commit sanitized excerpts or screenshots, never raw files.

---



# 23. Order of Work

v1.0 has one date: **30 December 2026**. Blocks are done in order and have no dates of their own: Setup, Blocks 1–6, then a protected buffer before the tag.


| Block | Milestone | Core | Stretch (cut first) | Exit check |
| --- | --- | --- | --- | --- |
| Setup | M0 + M1 part 1 | lab-host decision; host baseline **before** the first `compose up`; repo and `.gitignore`; secrets script; `/health`, `/db-check`; settings loader; Compose with Nginx, API, PostgreSQL and all four networks; one pytest; CI with pytest and Gitleaks | Alembic baseline migration | `/health` and `/db-check` return 200 through `127.0.0.1:8080` from a fresh `up` |
| Block 1 | M1 part 2 | Redis; `/cache-check`; `/limited` (pipeline, 503 fail-closed); `/debug/request`; `/slow`; JWT login, `/protected` and an authorization test; dependency timeouts; healthchecks; persistence check; Alembic if not done | Hadolint, Semgrep, Trivy report-only | Block 1 exit commands (section 14) all 200 |
| Block 2 | M2 | sections 15.1–15.6; `matrix.sh` with every row explained; DNS capture; port publishing; clean-clone smoke test on the final host (if different from the lab host); report-only scanners if not done | LAN-client comparison (15.6) | `docker-networking.md` with matrix output committed |
| Block 3 | M3 | veth, bridge-in-`ns-sw` and router labs saved as scripts; ARP, ICMP, TCP, HTTP and RESP captures | Wireshark screenshots (text excerpts are enough) | `linux-network-primitives.md` and `packet-captures.md` part 1 committed |
| Block 4 | M4 | 17.1–17.3 (A, B, C); certs script; TLS block; TLS 1.3 capture; timeout table | subnet-membership bonus; TLS 1.2 comparison capture | `proxy-client-ip.md` and the Experiment C write-up committed |
| Block 5 | M5 | eight drills | optional router drill | `troubleshooting.md` with eight entries committed |
| Block 6 | CI + docs | scanner triage, main switches to blocking; `vuln/sql-injection` write-up; one-page threat model; four diagrams; README | diagram polish | CI green and blocking on main; all DoD docs exist |
| Buffer | ship | final clean clone on the final host; DoD review; fix broken instructions; remove dead experiments; tag | — | `v1.0` tag pushed |




## Gates

- Don't start a block until the previous block's exit check passes, or its open items have been cut.
- Every Sunday: commit everything, make sure the current block's deliverable doc exists (even if rough), and tick the finished DoD boxes.
- Halfway to the tag date (mid-November), Block 3 should be done. If it isn't, apply the cut order immediately rather than borrowing from the buffer.

## Cut order

Cut from the top until back on track:

1. the current block's stretch items
2. the subnet-membership bonus experiment
3. the TLS 1.2 comparison capture (keep TLS 1.3)
4. the LAN-client comparison
5. Wireshark screenshots (replace with `tcpdump` text excerpts)

**Never cut:** the communication matrix, Experiments A–C, the eight drills, the clean clone, or the tag date.

If a DoD item is still open three days before the tag date, move it to a "v1.0.1" list in the README and tag v1.0 on time anyway.

No VLANs, OPNsense or other v1.1 topics in the buffer.

---



# 24. v1.0 Definition of Done



## Workload

- [ ] FastAPI, PostgreSQL, Redis and Nginx run in Docker with fixed `container_name`s
- [ ] only Nginx is published, and only on `127.0.0.1`
- [ ] healthchecks gate startup order
- [ ] JWT login and `/protected` work; one authorization test passes
- [ ] PostgreSQL persistence proven with a marker row written before `docker compose down` and read back after `up`
- [ ] Redis-backed limiter uses one MULTI/EXEC pipeline and fails closed with 503
- [ ] PostgreSQL and Redis client timeouts are set
- [ ] `*_FILE` secrets work from a clean clone via `make-secrets.sh`
- [ ] pytest runs in CI



## Docker networking

- [ ] four networks with explicit bridge names; three internal
- [ ] container namespaces inspected with netshoot and `nsenter`
- [ ] Docker DNS explained, including why `nsenter -n` needs `dig @127.0.0.11`
- [ ] DNS capture taken on the container's loopback, with the NAT rule behind it explained
- [ ] `matrix.sh` run; blocked flows tested by name **and** IP; every row's layer explained
- [ ] absence of Internet routes for API, PostgreSQL and Redis proven (`NO ROUTE`)
- [ ] port-publishing path explained from captures and rules on the final host



## Linux networking

- [ ] veth pair lab
- [ ] bridge built inside `ns-sw`; ARP table vs bridge FDB explained
- [ ] router namespace with forwarding set explicitly to 0, then 1
- [ ] one routed packet explained hop by hop (MAC change, IP unchanged, TTL −1)
- [ ] lab scripts and cleanup script committed



## Captures

- [ ] ARP
- [ ] ICMP across the router
- [ ] Docker DNS (container loopback)
- [ ] TCP handshake
- [ ] plaintext HTTP between Nginx and API
- [ ] TLS 1.3 handshake (certificate encrypted noted)
- [ ] Redis RESP with MULTI/EXEC, dummy data only



## Proxy and TLS

- [ ] Nginx overwrites `X-Forwarded-For`; upstream re-resolved through `127.0.0.11`
- [ ] Uvicorn version recorded (0.31.0 or later); trusts only `172.30.20.0/24`
- [ ] `sfl_app` membership verified as Nginx + API only
- [ ] Experiments A, B and C done with two clients
- [ ] TLS terminates at Nginx with a local CA; certificates generated by script
- [ ] Nginx mounts only `server.crt` and `server.key`; the CA key in `pki/` is never mounted
- [ ] timeout table completed (refused, connect timeout, slow-in-budget, 504, 499, DNS, TLS)



## Troubleshooting

- [ ] eight distinct drills documented in the section 18 format
- [ ] each identifies the failing layer
- [ ] DNS vs TCP, refused vs timeout, proxy timeout vs network timeout, TLS vs HTTP failures each distinguished at least once



## Security and CI

- [ ] Gitleaks, Hadolint, Semgrep and Trivy run; main is blocking
- [ ] third-party Actions pinned to SHAs
- [ ] two vulnerability experiments documented (forwarded-header trust, SQL injection)
- [ ] no vulnerable branch exposed beyond loopback
- [ ] no secrets, keys or raw pcaps in the repository history



## Portfolio

- [ ] diagrams: architecture, Docker network membership, one detailed packet flow, trust boundaries
- [ ] `host-network-baseline.md`, `docker-networking.md`, `linux-network-primitives.md`, `packet-captures.md`, `proxy-client-ip.md`, `troubleshooting.md`, `threat-model.md`
- [ ] README explains what was learned, not only which tools were used
- [ ] clean clone works on the final host with the three commands in section 7
- [ ] `v1.0` tagged by 30 December 2026

---



# 25. Lab Journal Template

```markdown
# Lab: <name>

## Goal
## Topology
## Addressing
## Prediction
## Commands used
## What happened
## Packet evidence
## Why it happened
## Failure introduced (if any)
## Troubleshooting steps
## Fix
## Questions still open
```

---



# 26. Portfolio and README

Recommended documents:

```text
docs/
├── architecture.md
├── host-network-baseline.md
├── docker-networking.md
├── linux-network-primitives.md
├── packet-captures.md
├── proxy-client-ip.md
├── troubleshooting.md
├── threat-model.md
└── vulnerabilities/
    ├── forwarded-header-trust.md
    └── sql-injection.md
```

A packet-flow diagram should annotate: source IP and port, destination IP and port, the DNS step, any NAT step, TLS state, and the new TCP connection at the proxy.

README opening:

```text
# secure-fastapi-lab

A FastAPI + PostgreSQL + Redis backend used to study the networking and
security behavior of a containerized service in depth: Docker networking,
Linux namespaces and bridges, DNS, TCP, reverse proxies, TLS, client-IP
trust, segmentation, packet capture, failure injection and systematic
troubleshooting.

Method: predict -> test -> capture -> explain.
```

Show the topology before the technology list. Then make concrete, proven statements:

```text
- FastAPI, PostgreSQL and Redis have no route to the Internet (tested by IP, not just by name).
- FastAPI reaches exactly two dependencies; Nginx is its only ingress.
- No database, cache or API port is published to the host.
- Client identity is accepted only from the proxy network, and the trust
  boundary is tested with forged headers from two clients.
- Packet captures document ARP, Docker DNS, TCP, HTTP, TLS 1.3 and Redis RESP.
- Eight deliberately broken scenarios are diagnosed layer by layer.
```

---



# 27. v1.1 — Network Track (after v1.0 ships)

**Target date:** set when v1.0 is tagged.

Order follows dependency, not novelty.

1. **Routing in depth.** Static routes, longest-prefix match, metrics, multiple routers, return paths, asymmetric routing. When building asymmetric paths, check `rp_filter` before declaring the topology broken (0 = off, 1 = strict, 2 = loose). Predict, capture, and change it only as a documented experiment.
2. **Manual NAT.** SNAT, masquerade, DNAT, port forwarding and conntrack on your own namespace router, then compared with Docker's rules.
3. **Stateful firewalling with nftables.** In namespaces: input/forward/output, default deny, established/related, logging drops, rule order.
4. **Docker firewall internals.** iptables vs nftables backend, `DOCKER-USER` (iptables backend only), forwarding policy, direct routing, gateway modes. Use the t740 or a disposable VM.
5. **VLANs.** 802.1Q, access and trunk ports, native VLAN, inter-VLAN routing; virtual first, then a managed switch.
6. **OPNsense.** Only after 1–3, so the GUI maps onto concepts you already understand.
7. **macvlan and ipvlan.** Why macvlan gives containers their own MACs; why host-to-macvlan traffic surprises people; why ipvlan reduces MAC proliferation.
8. **Redis TLS.** Compare with the plaintext RESP capture.
9. **More vulnerability experiments.** JWT validation, IDOR.
10. **IPv6 dual-stack rebuild** of one namespace lab.

---



# 28. Read Later, Not Now


| Topic                 | Read enough to understand                                           | Becomes useful when                                             |
| --------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------- |
| IPv6                  | /64, link-local, SLAAC, Neighbor Discovery, ICMPv6, no ARP          | dual-stack clouds, "IPv4 works, IPv6 doesn't"                   |
| MTU / PMTUD           | MTU, DF bit, Packet Too Big, MSS clamping                           | VPNs or overlays where small requests work and large ones stall |
| DHCP and relay        | DORA, leases, options 3 and 6, relay                                | VLANs, OPNsense                                                 |
| VPNs                  | route- vs policy-based, site-to-site, split tunnel; WireGuard first | remote admin, hybrid cloud                                      |
| STP/RSTP              | loops, broadcast storms, root bridge                                | multiple physical switches                                      |
| LACP                  | aggregation, hashing, redundancy                                    | multi-NIC servers                                               |
| OSPF                  | link-state, adjacency, areas, cost                                  | many routed networks                                            |
| BGP                   | AS, advertisement, path attributes, policy                          | Internet routing, ExpressRoute, Calico/Cilium                   |
| DNSSEC                | integrity and authenticity, not confidentiality                     | public DNS operation                                            |
| eBPF                  | kernel hooks for tracing, networking, policy                        | Cilium, modern observability                                    |
| Kubernetes networking | pod IPs, CNI, Services, NetworkPolicy, ingress                      | after the Docker mental model is solid                          |
| Advanced PKI          | intermediates, revocation, OCSP, automation                         | mTLS, service meshes                                            |


---



# 29. Azure Mapping — Reference Only

No Azure resources before v1.0 ships. Note analogies as anchors for later study; they are never exact.


| Local concept                 | Azure topic to study later        |
| ----------------------------- | --------------------------------- |
| subnet / interface            | VNet, subnet, NIC                 |
| route table                   | system routes, UDRs               |
| host or container filtering   | NSG                               |
| routed firewall               | Azure Firewall                    |
| Nginx reverse proxy           | Application Gateway / Front Door  |
| Docker DNS                    | Azure DNS, Private DNS zones      |
| internal-only service network | Private Endpoint / Private Link   |
| port publishing / DNAT        | Load Balancer rules, NAT concepts |
| packet capture                | Network Watcher, flow logs        |
| TLS termination               | Application Gateway / Front Door  |


---



# 30. Final Guiding Rule

The project succeeds when you can look at a failing request and reason downward through the stack:

```text
application
HTTP / RESP / PostgreSQL protocol
TLS
TCP
routing
IP addressing
ARP / Layer 2
interface / namespace / bridge
```

and then **prove** where the failure occurs.

The objective is not to become a network engineer by v1.0. It is to become a backend/DevOps engineer who is unusually comfortable with networks, with a clean v1.1 path if that interest keeps pulling.

---



# Appendix — Revision Notes (rev 2 -> rev 3)

What changed, and the failure each change prevents:

1. **Matrix tested by name and by IP, with a failure-layer column** (15.5). Blocked flows previously failed at DNS and proved nothing about segmentation. Host-to-container-IP rows were added because "not published" isn't "unreachable".
2. `matrix.sh` **added.** It turns the segmentation claim into repeatable evidence.
3. **Docker DNS capture moved to the container's loopback** (15.4). The resolver lives inside each container's namespace, so bridge captures were empty; the filter is `udp` because of Docker's DNS NAT.
4. **Docker 26.0+ required** (section 4). Older engines leak DNS out of internal networks (CVE-2024-29018), which changes the matrix results.
5. `nsenter` **DNS caveat** (section 11). `nsenter -n` reads the host's `resolv.conf`.
6. `container_name` **set for every service** (section 8). Otherwise every `api`/`nginx`/`db`/`redis` command fails.
7. **Uvicorn 0.31.0+ pinned and verified** (sections 4, 17.1). Older versions silently ignore CIDR trust.
8. **Experiment C defined precisely** (17.3). The previous Nginx config overwrote the forged header, so the spoof could never succeed.
9. **Experiment B uses two clients** (17.2). One client can't show a shared bucket.
10. **Nginx re-resolves upstreams** (section 9). Without it, a recreated API container causes persistent 502s.
11. **TLS work in one block** (Block 4). The TLS capture was scheduled a week before TLS existed; port 443 is added only with its certificates.
12. **TLS 1.3 vs 1.2 capture** (17.5). TLS 1.3 encrypts the certificate, so the old capture goal was impossible.
13. **Port publishing (old section 15) assigned to Block 2.** It previously had no schedule slot.
14. **JWT listed once** (Block 1). It was in two blocks before.
15. **Nginx moved into the setup block.** The API sits only on internal networks, so it could not be reached from the host without it.
16. **Drill 6 replaced** with "API listening on 127.0.0.1". The old Drill 6 duplicated Experiment B. The forwarded-header vulnerability write-up is Experiment C, written once.
17. `172.30.0.0/16` **default-pool conflict check** (section 5).
18. **Router lab sets** `ip_forward=0` **explicitly** before the "fails" test (16.3).
19. **Bridge lab inside** `ns-sw` with full commands (16.2), plus scripts and cleanup for all namespace labs.
20. **App behaviors the drills rely on are now required:** lazy connections, client timeouts, fail-closed limiter, async `/slow`, newline-safe secret loading (section 10).
21. **Rate limiter made atomic** with MULTI/EXEC (section 10).
22. **Secrets permissions fixed** (dir 700, files 644), and the script never overwrites existing secrets (section 22).
23. **Certificates generated by script and git-ignored.** This prevents Gitleaks failures and makes a clean clone work.
24. **CI scanners start report-only in Block 1–2** and become blocking in Block 6, instead of all arriving in the last build week.
25. `.local` **replaced by** `api.lab.test` with `curl --resolve`.
26. **PostgreSQL pinned to 17** to avoid the PostgreSQL 18 data-path change.
27. **Baseline taken before the first** `compose up`, including Docker's existing state if Docker is already installed.
28. **Lab-host decision and early clean-clone smoke test** (sections 4, 23), so evidence and the final host can't disagree.
29. **Weekly gates, core/stretch split, and an explicit cut order**, so the 30 November tag doesn't depend on everything going right.



## Revision 3.1

1. **Persistence test uses a marker row** (section 8). `/db-check` is `SELECT 1`, which only proves PostgreSQL came back.
2. **netshoot and** `resolv.conf` **explained precisely** (section 11). It shares the network namespace; Docker additionally hands it the target's `resolv.conf` file. The plan now says to verify rather than assume, which teaches network vs mount namespace.
3. **CA private key moved to** `pki/`, never mounted (17.4). Nginx mounts only its server certificate and key.
4. **Docker host named as part of the trusted computing base** (section 6). `internal: true` does not isolate containers from the host.
5. **netshoot and curl images pinned** (`v0.16`, `8.22.0`).
6. `depends_on` **described as startup orchestration only** (section 8), not runtime dependency management.
7. **"No API port mapping" stated as a design decision**, not as a consequence of Docker's handling of internal networks.


## Revision 3.2

1. **Dated schedule removed.** Each version has one target date: v1.0 is 30 December 2026 (was 30 November); v1.1 gets its date when v1.0 is tagged.
2. **"Week N" renamed "Block N"** throughout. Blocks are an order of work, not calendar weeks.
3. **Weekly Tuesday rule replaced** by block gates and one halfway check (section 23).
4. **DoD cut-off is relative:** three days before the tag date.
5. **Lab host provisioned** and documented in `docs/lab-host-setup.md`; baseline now records ufw and Tailscale (sections 4, 13).
6. **SSH tunnel rule added** for reaching the loopback-only stack from another machine (section 3).
