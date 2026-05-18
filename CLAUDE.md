# solti-docker — Ansible Collection for Docker Compose Services

Refactoring of `lab-docker-stack` from manually-managed per-host compose files into an
Ansible push-configured collection. Eliminates drift across dockarr and TrueNAS.

## Core Commands

```bash
# Service lifecycle
./manage-svc.sh <service> prepare          # Create directories (one-time)
./manage-svc.sh <service> deploy           # Template + docker compose up
./manage-svc.sh <service> remove           # docker compose down + cleanup

# Target a specific host
./manage-svc.sh -h truenas <service> deploy

# Remove including data
DELETE_DATA=true ./manage-svc.sh <service> remove

# Functional verification (separate from deploy smoke test)
./svc-exec.sh <service> verify
./svc-exec.sh -h truenas <service> verify
```

## Supported Services

| Service | Notes |
|---------|-------|
| redis   | redis + redis-commander + redis-insight |

## Architecture: Three Key Patterns

### 1. Ownership Override (dockarr vs TrueNAS)

`group_vars/truenas.yml` sets:
```yaml
install_root_owner: "568"   # TrueNAS apps user
install_root_group: "568"
docker_become: true
```

`_base/tasks/prepare.yml` uses:
```yaml
become: "{{ install_root_owner != ansible_user }}"
```
This evaluates false on dockarr (same user, no escalation) and true on TrueNAS
(different user, sudo needed to chown dirs to 568).

All `docker compose` calls use `become: "{{ docker_become }}"`.

### 2. Secrets Flow

No ansible-vault, no HashiVault.

1. Export credentials in shell: `export REDIS_PASSWORD=...`
2. `group_vars/redis_svc.yml` captures them: `redis_password: "{{ lookup('env', 'REDIS_PASSWORD') }}"`
3. `_base/tasks/secrets.yml` templates `Secrets/{service}.env` at deploy time (mode 0600)
4. `compose.yaml` references it via `env_file:` with a literal path

### 3. Literal Paths in compose.yaml

Ansible resolves all paths at template time. No `${DOCKER_ROOT}` variable
interpolation inside compose files. No `sync-env.sh` needed.

```yaml
# compose.yaml.j2 — all paths are literal after templating
volumes:
  - /home/lavender/Docker/Stacks/redis:/data
env_file:
  - /home/lavender/Docker/Secrets/redis.env
```

## Install Root Directory Layout

```
{install_root}/
├── Projects/{service}/compose.yaml   # Ansible-managed, do not edit
├── Stacks/{service}/                 # Runtime data (preserved on remove)
└── Secrets/{service}.env             # Ansible-managed credentials (0600)
```

**dockarr:** `~/Docker/`  
**TrueNAS:** `/mnt/zpool/Docker/`

## State Machine

| State | Action |
|-------|--------|
| `prepare` | mkdir + chown — one-time, idempotent |
| `present` | template secrets + compose.yaml → `docker compose up` + smoke test |
| `absent` | `docker compose down` → remove compose.yaml + secrets → optionally delete Stacks dir |

`present` includes an inline smoke test (container running check).  
Full functional verification is a separate step via `svc-exec.sh verify`.

## Adding a New Service

1. `cp -r roles/redis roles/<new_service>`
2. Update `roles/<new_service>/defaults/main.yml` — set `service_properties`, images, vars
3. Update `roles/<new_service>/templates/compose.yaml.j2` — adapt compose definition
4. Update `roles/<new_service>/templates/secrets.env.j2` — any credentials
5. Update `roles/<new_service>/tasks/verify.yml` — functional health checks
6. Add host entries to `inventory/hosts.yml` under a `<new_service>_svc` capability group
7. Add `inventory/group_vars/<new_service>_svc.yml` for service defaults
8. Add service name to `SUPPORTED_SERVICES` in both `manage-svc.sh` and `svc-exec.sh`

## Two Hosts

**dockarr** (`192.168.101.7`) — Debian Linux VM  
- `ansible_user: lavender`
- `install_root: ~/Docker` (same as docker user — no become needed)
- `docker_become: false`

**truenas** (`192.168.40.6`) — TrueNAS SCALE  
- `ansible_user: lavadmin`
- `install_root: /mnt/zpool/Docker`
- `install_root_owner: "568"` (apps user)
- `docker_become: true` (sudo required for docker commands)

## Deploy Order When Starting Fresh

Traefik must deploy first — it creates the `backend_storage` and `backend_media` Docker networks
that all other services reference as `external: true`. Deploy traefik last when replacing an
existing running instance.
