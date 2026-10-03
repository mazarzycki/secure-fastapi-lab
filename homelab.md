# Secure FastAPI Homelab

**Target:** v1.0 complete by **30 November 2026**

## Goal

Build one coherent homelab project around a real FastAPI backend, then use Docker and security practices to harden, test, deliberately break, detect, and fix it.

The project should demonstrate:

- backend engineering
- Docker and Docker Compose
- PostgreSQL
- authentication and authorization
- automated testing
- CI/CD
- container and application security
- vulnerability analysis
- secure design thinking

The application is the center of the lab. Security tools support the application; they are not the project by themselves.

---

## v1.0 Scope

### Core application

- FastAPI
- PostgreSQL
- SQLAlchemy
- Alembic
- JWT authentication
- at least one authenticated endpoint
- pytest
- structured configuration/settings
- Docker
- Docker Compose
- Nginx
- health checks
- persistent database volume
- separated Docker networks where useful

### Secrets

Use file-based secrets with the `*_FILE` convention.

Example:

```text
POSTGRES_PASSWORD_FILE=/run/secrets/postgres_password
JWT_SECRET_FILE=/run/secrets/jwt_secret
```

Compose secrets are mounted as files under:

```text
/run/secrets/<secret_name>
```

The FastAPI settings layer should support both:

```text
SETTING=value
```

and:

```text
SETTING_FILE=/run/secrets/setting
```

Do not commit real secrets.

### Ignore secrets from the first commit

Gitleaks only arrives in Week 2, so nothing scans the Evening 1 commits. The `.gitignore` must exist before the first commit:

```gitignore
secrets/*
!secrets/.gitkeep
.env
```

### Secret file permissions

Docker Compose implements file-backed secrets as bind mounts, so the host file's ownership and permissions matter. For secrets sourced from host files, Compose does not remap `uid`, `gid`, or `mode`.

The official PostgreSQL image reads `POSTGRES_PASSWORD_FILE` through its entrypoint before dropping privileges to the `postgres` user. Our FastAPI container, however, will run directly as a non-root user and therefore must be able to read its mounted secret.

Do not assume that one host permission setting is correct for every container. When the FastAPI image is implemented, inspect its runtime UID/GID and choose the narrowest host-file permissions that allow the intended container process to read the secret.

Record the resulting choice in the threat model as a deliberate trade-off, not an oversight.

---

## CI/CD

The main branch should eventually run:

- pytest
- Hadolint
- Gitleaks
- Semgrep
- Trivy

The pipeline should block merges/builds on meaningful findings once the initial baseline is clean.

Do not spend time adding overlapping scanners unless a concrete learning need appears.

---

## Vulnerability Experiments

Intentional vulnerabilities live in branches matching:

```text
vuln/*
```

These branches use a separate **report-only** security workflow.

Candidate scenarios:

- raw SQL injection
- JWT signature verification disabled or misconfigured
- IDOR / BOLA
- mass assignment
- insecure secret handling
- excessive container permissions
- unsafe Docker configuration

For every scenario, document:

1. What was changed?
2. How can it be exploited?
3. Which automated tools detected it?
4. Which tools missed it?
5. Why?
6. How was it fixed?
7. Which regression test prevents it from returning?

A key project outcome is showing that automated scanners do **not** detect every design-level vulnerability.

---

## Networking Rules for v1.0

Vulnerable targets must never be unintentionally exposed to the LAN or Internet.

Prefer:

```yaml
ports:
  - "127.0.0.1:8000:8000"
```

or Docker networks configured as:

```yaml
internal: true
```

Access private services through localhost, SSH tunnelling, or another deliberate mechanism.

Firewall-specific Docker behaviour is a later topic.

### Later note

With Docker's iptables backend, custom filtering is typically applied through `DOCKER-USER`.

The nftables backend behaves differently and should be studied separately.

This is **not part of v1.0**.

---

# Initial Architecture

```text
                    Developer
                        |
                        v
                  Git / GitHub
                        |
                     CI/CD
                        |
        +---------------+----------------+
        |               |                |
      pytest         Semgrep          Trivy
                    Gitleaks         Hadolint

                        |
                        v

                    Nginx
                        |
                        v
                    FastAPI
                        |
                        v
                  PostgreSQL
```

Later phases may add monitoring and runtime detection, but the application stack remains the center of the project.

---

# Repository Scaffold

```text
secure-fastapi-lab/
|
|-- app/
|   |-- __init__.py
|   |-- main.py
|   |-- config.py
|   |-- database.py
|   |-- models/
|   |-- schemas/
|   |-- api/
|   |-- services/
|   `-- security/
|
|-- tests/
|   |-- __init__.py
|   `-- test_auth.py
|
|-- migrations/
|
|-- nginx/
|   `-- nginx.conf
|
|-- secrets/
|   `-- .gitkeep
|
|-- .github/
|   `-- workflows/
|       |-- ci.yml
|       `-- vuln-report.yml
|
|-- .gitignore
|-- .dockerignore
|-- Dockerfile
|-- compose.yml
|-- pyproject.toml
|-- alembic.ini
|-- README.md
`-- HOMELAB.md
```

The exact structure may evolve while building. Avoid reorganizing files merely for aesthetics.

---

# Evening 1

## Objective

Finish the evening with a working application that proves the basic development loop.

### Checklist

- [ ] Create repository
- [ ] Run `git init`
- [ ] Create Python project
- [ ] Add FastAPI
- [ ] Add `/health` endpoint
- [ ] Add one JWT-protected endpoint
- [ ] Add settings loader with `*_FILE` support
- [ ] Run PostgreSQL through Docker Compose
- [ ] Connect FastAPI to PostgreSQL
- [ ] Add one pytest test
- [ ] Create GitHub repository
- [ ] Add GitHub Actions workflow
- [ ] CI runs pytest successfully
- [ ] `.gitignore` excludes `secrets/*` and `.env`
- [ ] Make first meaningful commit

### Initial commands

```bash
mkdir secure-fastapi-lab
cd secure-fastapi-lab

git init

git branch -M main

mkdir -p app tests .github/workflows secrets

touch app/__init__.py
touch tests/__init__.py
touch secrets/.gitkeep

touch README.md
touch HOMELAB.md
touch .gitignore
touch .dockerignore
touch compose.yml
touch Dockerfile
touch pyproject.toml
```

### Where to build

Nothing in v1.0 depends on the t740. Start on the laptop now. When the t740 is ready, deploying from a clean clone onto it is the test for the "repository can be built from a clean clone" item in the Definition of Done.

### PostgreSQL in CI

The GitHub Actions runner has no PostgreSQL. If a test touches the database, CI fails.

- **Evening 1:** keep the first tests DB-free (`/health` and the JWT-protected endpoint, with the database dependency overridden).
- **When the first DB-backed test lands:** add a `services: postgres` block with a health check to `ci.yml`.

### First milestone

The first evening is successful when this works:

```text
git push
   |
   v
GitHub Actions
   |
   v
pytest
   |
   v
PASS
```

Do not add Trivy, Semgrep, Gitleaks, Hadolint, Grafana, Falco, Juice Shop, Kubernetes, or OPNsense before this basic loop works.

---

# Four-Week v1.0 Plan

## Week 1 — Application Foundation

Focus:

- FastAPI structure
- PostgreSQL
- SQLAlchemy
- Alembic
- authentication
- authorization
- pytest
- Docker Compose

Expected result:

A working backend that can be started consistently from the repository.

---

## Week 2 — Container Hardening and CI

Focus:

- multi-stage Dockerfile
- non-root container
- health checks
- secrets using `*_FILE`
- Docker networking
- Nginx
- Hadolint
- Gitleaks
- Semgrep
- Trivy

Expected result:

The main branch has a meaningful security pipeline.

---

## Week 3 — Vulnerability Scenarios

Create selected `vuln/*` branches.

Suggested first scenarios:

```text
vuln/sql-injection
vuln/jwt-validation
vuln/idor
vuln/mass-assignment
```

For each vulnerability:

- reproduce
- document
- run security tools
- compare detection results
- fix
- add regression test

Expected result:

The repository demonstrates the difference between tool-detectable vulnerabilities and design flaws.

---

## Week 4 — Polish and Publish

Focus:

- clean README
- architecture diagram
- threat model
- vulnerability table
- screenshots / CI evidence where useful
- remove dead code
- verify clean setup from scratch
- tag v1.0

Expected result:

A repository that another engineer can clone, understand, run, and review.

---

# v1.0 Definition of Done

v1.0 is complete when:

- [ ] application starts via Docker Compose
- [ ] PostgreSQL data persists
- [ ] authentication works
- [ ] authorization is tested
- [ ] configuration supports `*_FILE`
- [ ] secrets are not stored in Git
- [ ] tests run automatically
- [ ] main CI runs pytest
- [ ] Trivy runs in CI
- [ ] Semgrep runs in CI
- [ ] Gitleaks runs in CI
- [ ] Hadolint runs in CI
- [ ] important findings can fail the main pipeline
- [ ] vulnerable branches use report-only security checks
- [ ] at least two vulnerability scenarios are documented
- [ ] at least one scanner-missed design vulnerability is demonstrated
- [ ] fixes have regression tests
- [ ] README explains architecture and security decisions
- [ ] repository can be built from a clean clone (verified on the t740)
- [ ] v1.0 is tagged by 31 October 2026

If the deadline is at risk, **cut scope rather than move the date**.

---

# Explicitly Out of Scope for v1.0

Do not add these unless required to solve a real problem:

- Kubernetes
- OPNsense
- Suricata
- Wazuh
- Windows Active Directory
- Falco
- Loki
- Juice Shop as a permanent service
- WebGoat as a permanent service
- DVWA as a permanent service
- complex VLAN design
- multi-node infrastructure
- elaborate dashboards

These are later projects or later versions.

---

# Possible v1.1+

After v1.0 is published:

```text
v1.1  Runtime detection
      -> Falco

v1.2  Observability
      -> Prometheus + Grafana
      -> Loki if useful

v1.3  Additional attack targets
      -> Juice Shop / WebGoat

v2.0  Network-security lab
      -> hypervisor or separate hardware
      -> second NIC
      -> OPNsense
      -> VLANs
      -> Suricata
      -> isolated attacker/victim networks
```

---

# Guiding Rule

**Build first. Add tools only when they solve a problem or teach something specific.**
