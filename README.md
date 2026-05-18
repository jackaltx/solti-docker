# solti-docker

Ansible collection for managing Docker Compose services using the SOLTI state-machine pattern.

Refactored from [lab-docker-stack](https://github.com/jackaltx/true-lab-docker-stack) to eliminate
configuration drift across multiple hosts via Ansible push-configuration.

## Quick Start

```bash
# Install dependencies
ansible-galaxy collection install -r requirements.yml

# Deploy redis to dockarr
./manage-svc.sh redis prepare
./manage-svc.sh redis deploy
./svc-exec.sh redis verify
```

## Supported Services

- `redis` — Redis cache + redis-commander + redis-insight

## Hosts

- **dockarr** — Debian Linux VM (`~/Docker/`)
- **truenas** — TrueNAS SCALE (`/mnt/zpool/Docker/`, apps user 568)

See [CLAUDE.md](CLAUDE.md) for architecture details.
