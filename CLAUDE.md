# solti-docker — Ansible Collection for Docker Compose Services

Refactoring of `lab-docker-stack` from manually-managed per-host compose files into an
Ansible push-configured collection. Eliminates drift across dockarr and TrueNAS.

## Core Commands

> **Surfacing to unified docs:** Update `solti-docs.yml` at the collection root
> to declare which files and CLAUDE.md sections should appear on solti.jackaltx.com.
> Local `docs/` detail stays local — only declare what matters to the suite-wide audience.
> See [solti-docs/HARVEST.md](https://github.com/jackaltx/solti-docs/blob/main/HARVEST.md).


```bash
# Service lifecycle (svc-manage.sh replaces manage-svc.sh + svc-exec.sh)
./svc-manage.sh <service> prepare          # Create directories (one-time)
./svc-manage.sh <service> deploy           # Template + docker compose up
./svc-manage.sh <service> remove           # docker compose down + cleanup

# Target a specific host
./svc-manage.sh -h truenas <service> deploy

# Remove including data
DELETE_DATA=true ./svc-manage.sh <service> remove

# Task commands (verify, configure, any tasks_from file)
./svc-manage.sh <service> verify
./svc-manage.sh -h truenas <service> verify

# Pass extra vars
./svc-manage.sh <service> deploy -e some_var=value

# Skip confirmation prompt
./svc-manage.sh -y <service> deploy

# Old scripts (manage-svc.sh, svc-exec.sh) preserved but superseded
```

## Supported Services

| Service | Notes |
|---------|-------|
| dockhand | Docker Compose stack management UI (LinuxServer.io) |
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

## Planned Feature: Image Version Pinning

Currently all roles default to `latest`. The intended pattern when version pinning
is implemented is to split the image tag into its own variable in `defaults/main.yml`:

```yaml
# roles/redis/defaults/main.yml
redis_image_tag: "latest"
redis_image: "redis:{{ redis_image_tag }}"
```

Override per service in `group_vars/<service>_svc.yml` to pin globally:

```yaml
redis_image_tag: "7.2-alpine"
```

Or per host inline in `inventory/hosts.yml` to run different versions:

```yaml
redis_svc:
  hosts:
    dockarr:
      redis_image_tag: "7.2-alpine"
    truenas:
      redis_image_tag: "latest"
```

Services with multiple images (redis, traefik) get one tag var per image:
`redis_image_tag`, `redis_commander_image_tag`, `redis_insight_image_tag`.

Implementation is a mechanical split of every `image: "name:tag"` in every
`defaults/main.yml` — no logic changes required.

## Deploy Order When Starting Fresh

Traefik must deploy first — it creates the `backend_storage` and `backend_media` Docker networks
that all other services reference as `external: true`. Deploy traefik last when replacing an
existing running instance.

## Known Gotchas

**rustfs uid 10001**: Runs as non-root uid 10001. `prerequisites.yml` must
`become: true` to chown `Stacks/rustfs` before starting. Pattern to reuse
for any image that runs as a non-root uid that doesn't match `ansible_user`.
Also: rustfs requires auth on ALL endpoints (including health checks) — 403
is the healthy response. Verify accepts `http_code in [200, 403]`.

**rustfs log dir**: `RUSTFS_OBS_LOG_DIRECTORY` defaults to `/logs` (inside
container root, read-only for uid 10001). Redirected to `/data/logs` in
compose template to keep everything in the writable `/data` mount.

**jellyfin disk guard**: Jellyfin v10.10+ refuses to start if less than 2 GiB
free on the config path. Symptom: perpetual restarting. Fix: `docker image
prune -a` to reclaim unused image layers, then `docker restart jellyfin`.

**minio health check**: Port 9000 is not exposed to the host — health check
must use `docker exec minio curl` not the `uri` module from the controller.

**group_vars vs role defaults**: `service_puid`, `service_pgid`, `service_tz`
must live in `group_vars/all.yml`, not `roles/_base/defaults/main.yml`. Role
defaults are only loaded when that role runs — they are not shared across roles.

**svc-exec.sh include_role**: Must use `include_role: tasks_from: verify` (not
`include_tasks: verify.yml`) so role defaults load and `service_properties` is
available inside verify tasks.

**hyphenated service names**: `it-tools` style names require underscore
conversion in two places in `manage-svc.sh` and `svc-exec.sh`: the `_svc`
inventory group name and the `<service>_state` variable name.

**docker socket + user directive**: When a role uses `user: "{{ service_puid }}:{{ service_pgid }}"`,
also add `group_add: ["{{ docker_gid }}"]` so the process can read `/var/run/docker.sock`.
`docker_gid` lives in each host's inventory vars (dockarr: `"994"`). Check with
`getent group docker` on the host if unknown.

**gluetun stale servers.json**: The gluetun Docker image bundles a pre-baked `servers.json`
that can be months old. If the volume is seeded from the image layer rather than a live
download, `UPDATER_PERIOD=4h` won't help — the updater runs after VPN connects, but VPN
can't connect because the server list is stale (chicken-and-egg). Symptom: `EHOSTUNREACH`
on startup, servers.json shows 500+ days old in logs. Fix: `rm Stacks/arr-stack/gluetun/servers.json`
and restart — gluetun will fetch a fresh list. TODO: seed servers.json at deploy time via the role.

**stale root-owned data**: If a container was ever started without `user:` and created files
in its bind-mount path, those files will be owned by root. Switching to a non-root `user:`
will cause `SQLITE_READONLY` or similar write failures. Fix: `ansible <host> -m command -b
-a "chown -R <uid>:<gid> <Stacks/service/>"` before redeploying.

## Open Items for Review

**TrueNAS**: `arr-stack` and `jellyfin` still on manual lab-docker-stack
compose. Before migrating: decide on 568 ownership for media dirs and whether
arr-stack VPN config (Gluetun + PIA) needs a dedicated role or stays manual.

**jellyfin role**: Complete — but not tested on TrueNAS. `jellyfin_media_root`
must be set to `/mnt/zpool/Media` in the `truenas` inventory entry.

**openclaw**: Not added — user confirmed not needed at this time.

**filebrowser**: Skipped — user looking for a replacement.

**image version pinning**: Planned feature documented above. All roles
currently use `latest`. Implement before any production TrueNAS migration.

**minio vs rustfs**: Both roles exist. On TrueNAS decide which to use as the
primary S3 store. They share the same port layout (9000/9001) so only one
can run on a given host without Traefik hostname differentiation.

**mongodb `mongodb_host_port`**: Defaults to `""` (no host exposure). Set to
`"{{ traefik_bind_ip }}:27017"` in inventory if services outside the Docker
network need direct connections.

**traefik dashboard auth hash**: Stored as raw bcrypt (single `$`) in env var
`TRAEFIK_DASHBOARD_AUTH`. Template uses `replace('$', '$$')` for compose
label escaping. Regenerate with:
`echo $(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g` then remove
the extra `$` before exporting.
