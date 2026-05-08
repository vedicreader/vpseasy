---
name: vpseasy
description: >
  VPS lifecycle in Python: cloud-init generation, Hetzner provisioning,
  Multipass local testing, SSH helpers, Docker Compose deployment.
---

# vpseasy

Provision VPSes, test locally with Multipass, deploy Docker Compose apps — all from Python.

## Setup

```sh
pip install vpseasy
export HCLOUD_TOKEN=your_token   # required for Hetzner
```

## Key functions

| Function | Returns | Description |
|----------|---------|-------------|
| `load_pub_keys(paths=None)` | `list[str]` | Read pub keys from `~/.ssh/id_*.pub` |
| `gen_key(slug, key_dir=None)` | `AttrDict(key, pub, pub_str)` | Generate ed25519 key pair |
| `multi_init(hostname, pub_keys=None, ...)` | `AttrDict(yaml, key)` | Cloud-init for Multipass (no UFW) |
| `vps_init(hostname, pub_keys=None, ...)` | `AttrDict(yaml, key)` | Cloud-init for prod VPS (UFW, fail2ban, Docker) |
| `Multipass().launch(name, image, ...)` | `AttrDict(name, key)` | Spin up a local Ubuntu VM |
| `Multipass().ip(name)` | `str` | Get VM IPv4 |
| `Multipass().rm(name)` | `None` | Delete + purge VM |
| `deploy_mp(name, src, path, build)` | `None` | Rsync + `docker compose up` inside Multipass VM |
| `Hetzner().create(name, ...)` | `AttrDict(ip, name, key, resp)` | Provision Hetzner VPS |
| `Hetzner().servers()` | `list[dict]` | List servers `[{name, ip, status}]` |
| `Hetzner().delete(name)` | `None` | Delete server |
| `Hetzner().key_names()` | `list[str]` | SSH key names registered in Hetzner (root access only) |
| `hetzner_deploy(name, src, ...)` | `AttrDict(ip, name, key)` | Full pipeline: provision → wait → deploy (idempotent) |
| `wait_ssh(host, u, k, tout)` | `True` | Poll until SSH is up |
| `wait_ready(host, u, k, tout, retries)` | `True` | Poll SSH then cloud-init until done; retries on timeout |
| `chk_cloud_init(host, u, k)` | `str` | `done\|running\|error\|unknown` |
| `chk_docker(host, u, k)` | `bool` | Verify Docker daemon running |
| `run_ssh(host, *cmds, user, key, capture)` | `str\|CompletedProcess` | Run commands over SSH |
| `sync(host, src, path, user, key, include, exclude)` | `None` | Rsync local dir to remote |
| `deploy(host, src, path, user, build, key)` | `None` | `sync` + `docker compose up -d` |
| `vols_to_binds(vols)` | `list[str]` | `["/app/data"]` -> `["./data:/app/data"]` for Compose bind mounts |
| `caddy_stack(domain, df, vols, root, ...)` | `Compose` | app + caddy + cloudflared + web network; saves files if `root=` given |
| `mv_skill_md(dry_run, dir)` | `None` | Install SKILL.md to `.agents/` and `~/.claude/` |

## Common patterns

### Local test with Multipass

```python
from vpseasy.core import *
ci = multi_init('testvm')          # AttrDict(yaml, key) — auto-generates key pair
mp = Multipass()
vm = mp.launch('testvm', cloud_init=ci)
ip = mp.ip('testvm')
deploy_mp('testvm', src='./myapp')
mp.rm('testvm')
```

### Hetzner production deploy

`hetzner_deploy` is idempotent — re-running against an existing server skips provisioning and goes straight to cloud-init check + deploy.

```python
from vpseasy.core import *
hz = Hetzner()                          # reads HCLOUD_TOKEN
svr = hetzner_deploy('myapp-prod', './myapp', hz=hz, location='hel1')
print(f'Deployed at {svr.ip}, key: {svr.key}')
```

### Lower-level Hetzner flow

Use `vps_init` + `Hetzner.create` + `wait_ready` + `deploy` when you need more control:

```python
from vpseasy.core import *
hz = Hetzner()
ci = vps_init('myapp-prod')             # auto-generates ~/.ssh/myapp-prod key pair
svr = hz.create('myapp-prod', cloud_init=ci, location='hel1')
# NOTE: do not pass ssh_keys= together with cloud_init= — they are conflicting strategies.
# ssh_keys injects into root only; cloud_init creates a deploy user with its own key.
wait_ready(svr.ip, k=svr.key, tout=600)
deploy(svr.ip, './myapp', key=svr.key)
```

### App with Caddy and Cloudflare Tunnel

Any dockeasy Dockerfile (fasthtml_app, go_app, rust_app, node_app, python_app, ...) can be deployed behind Cloudflare Tunnel with one call. `caddy_stack` writes `Dockerfile`, `docker-compose.yml`, and `Caddyfile` when `root=` is given; pass `**kw` through to `caddy_svc` for dns, email, crowdsec, etc.

```python
from vpseasy.core import *
from dockeasy import fasthtml_app  # or go_app, rust_app, etc.

root = Path(__file__).resolve().parent

def mk_compose():
    df = fasthtml_app(pkgs=['sqlite3'], vols=['/app/data'], healthcheck='/health')
    return caddy_stack('myapp.example.com', df, vols=['/app/data'], root=root)

def deploy2prod():
    mk_compose()
    r = hetzner_deploy('myapp', root, include=['app/', 'pyproject.toml',
                        'docker-compose.yml', 'Dockerfile', 'Caddyfile', '.env'])
    print(f'deployed: {r.ip}')
```

### Install this skill

```python
mv_skill_md(dry_run=False)   # writes to .agents/skills/vpseasy/ and ~/.claude/skills/vpseasy/
```

## SSH key resolution

`_resolve_key` resolves the SSH key in this order:

1. Explicit `key=` path argument
2. `name=` slug → `~/.ssh/<name>` (raises `FileNotFoundError` if missing)
3. `SSH_KEY_PATH` environment variable
4. `SSH_PRIVATE_KEY` environment variable (PEM string, written to `/tmp/.vpseasy_<uid>.pem`)
5. `None` (falls back to SSH agent)

## Env vars

| Var | Used by | Notes |
|-----|---------|-------|
| `HCLOUD_TOKEN` | `Hetzner()` | Required for Hetzner |
| `SSH_KEY_PATH` | `_resolve_key()` | Optional explicit key path |
| `SSH_PRIVATE_KEY` | `_resolve_key()` | PEM string, written to `/tmp/.vpseasy_<uid>.pem` |

## Gotchas

- `ssh_keys` on `Hetzner.create()` injects into the **root** user only — not the `deploy` user created by cloud-init. Passing both `ssh_keys` and `cloud_init` raises `ValueError`.
- `cloud-init status` returns exit code 2 ("done with warnings") on Ubuntu 24.04 — `chk_cloud_init` uses `check=False` to handle this correctly.
- SSH `StrictHostKeyChecking=accept-new` silently accepts new hosts but rejects changed host keys. `hetzner_deploy` runs `ssh-keygen -R <ip>` after provisioning to clear stale `known_hosts` entries when an IP is reused.
- Always use the private key path (no `.pub`) with `-i`. The public key is for `authorized_keys`, not for SSH client auth.