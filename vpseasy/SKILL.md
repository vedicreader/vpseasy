---
name: vpseasy
description: >
  VPS lifecycle in Python: cloud-init generation, Hetzner and AWS Lightsail provisioning,
  Multipass local testing, SSH helpers, Docker Compose deployment, Litestream SQLite backups.
---

# vpseasy

Provision VPSes, test locally with Multipass, deploy Docker Compose apps — all from Python.

## Setup

```sh
pip install vpseasy              # Hetzner + Multipass
pip install 'vpseasy[aws]'       # adds boto3 for Lightsail
export HCLOUD_TOKEN=your_token   # required for Hetzner
export AWS_PROFILE=your_profile  # Lightsail: any standard AWS credential source (SSO profile, env vars)
```

## Providers

All providers share `servers()`, `get(name)`, `server_ip(name)`, `create(name, cloud_init=...)` and `delete(name)`, so `vps_deploy(name, src, provider, **create_kw)` works with any of them.

| Provider | Class | Credentials | `create_kw` |
|----------|-------|-------------|-------------|
| Hetzner Cloud | `Hetzner()` | `HCLOUD_TOKEN` | `image`, `server_type`, `location` |
| AWS Lightsail | `Lightsail(region='ap-southeast-2')` | AWS chain (`AWS_PROFILE`) | `bundle` (`small_3_2`), `blueprint`, `zone`, `static_ip`, `ports`, `snapshots` |
| Multipass (local) | `Multipass()` | none | use `deploy_mp`, not `vps_deploy` |

## Key functions

| Function | Returns | Description |
|----------|---------|-------------|
| `load_pub_keys(paths=None)` | `list[str]` | Read pub keys from `~/.ssh/id_*.pub` |
| `gen_key(slug, key_dir=None)` | `AttrDict(key, pub, pub_str)` | Generate ed25519 key pair |
| `multi_init(hostname, pub_keys=None, ...)` | `AttrDict(yaml, key)` | Cloud-init for Multipass (no UFW) |
| `vps_init(hostname, pub_keys=None, swap='4G', swappiness=10, ...)` | `AttrDict(yaml, key)` | Cloud-init for prod VPS (UFW, fail2ban, Docker, swap file; `swap=None` skips swap) |
| `Multipass().launch(name, image, ...)` | `AttrDict(name, key)` | Spin up a local Ubuntu VM |
| `Multipass().ip(name)` | `str` | Get VM IPv4 |
| `Multipass().rm(name)` | `None` | Delete + purge VM |
| `deploy_mp(name, src, path, build)` | `None` | Rsync + `docker compose up` inside Multipass VM |
| `Hetzner().create(name, ...)` | `AttrDict(ip, name, key, resp)` | Provision Hetzner VPS |
| `Hetzner().servers()` | `list[dict]` | List servers `[{name, ip, status}]` |
| `Hetzner().delete(name)` | `None` | Delete server |
| `Hetzner().key_names()` | `list[str]` | SSH key names registered in Hetzner (root access only) |
| `Lightsail().create(name, bundle, ...)` | `AttrDict(ip, name, key, resp)` | Provision Lightsail instance (idempotent, static IP, port 22 only, daily snapshots) |
| `Lightsail().bucket(name)` | `AttrDict(bucket, region, endpoint, key_id, secret)` | Bucket (created if missing) + new access key for backups; Lightsail allows two keys per bucket, secret shown only once |
| `vps_deploy(name, src, provider, ...)` | `AttrDict(ip, name, key)` | Full pipeline on any provider: provision → wait → deploy (idempotent) |
| `hetzner_deploy(name, src, ...)` | `AttrDict(ip, name, key)` | `vps_deploy` on Hetzner |
| `wait_ssh(host, u, k, tout)` | `True` | Poll until SSH is up |
| `wait_ready(host, u, k, tout, retries)` | `True` | Poll SSH then cloud-init until done; retries on timeout |
| `chk_cloud_init(host, u, k)` | `str` | `done\|running\|error\|unknown` |
| `chk_docker(host, u, k)` | `bool` | Verify Docker daemon running |
| `run_ssh(host, *cmds, user, key, capture)` | `str\|CompletedProcess` | Run commands over SSH |
| `sync(host, src, path, user, key, include, exclude, port, check=True)` | `int` | Rsync local dir to remote; raises on failure unless `check=False`, then returns the exit code |
| `sync_paths(host, paths, src, path, ..., last=None, dry_run=False)` | `list` | One rsync per path, `--delete` scoped to it; `last` pattern sent in a second pass; refuses a path naming the root (`.`, `''`, `/`); `host=None` copies to a local folder |
| `rsync_cmds(paths, src, dest, ssh_e, ...)` | `list` | The commands `sync_paths` runs |
| `ssh_cmd(host, user, key, port)`, `res_key(key, name)`, `askpass_env(pw)` | | The `ssh` command line, key lookup, and password-prompt context manager, for your own ssh or rsync |
| `deploy(host, src, path, user, build, key, port, prune=True, keep=3)` | `None` | `sync` + `docker compose up -d`; keeps the new and the replaced image for `rollback`, prunes dangling images after |
| `keep_image(host, path, svc, img, keep, prev=None)` | `str\|None` | Tag the running image and `prev` with their build times, tag `prev`, keep the newest `keep` timestamps plus those two |
| `kept_images(host, path, svc, img)` | `list[(tag, id)]` | Kept timestamped images, newest first |
| `rollback(host, to=None, path, svc, img)` | `str` | Retag a kept image as latest (newest older than the running one by default), tag the replaced one `prev`, restart the service; `to='prev'` undoes |
| `wait_http(url, ua, tries, poll)` | `bool` | Poll until the URL answers 200 |
| `server_health(host, ...)` | `AttrDict` | RAM, swap, pct, load, disk, containers |
| `vols_to_binds(vols)` | `list[str]` | `["/app/data"]` -> `["./data:/app/data"]` for Compose bind mounts |
| `caddy_stack(domain, df, vols, root, app_kw, caddy_kw, cloudflared_kw, caddyfile, write_caddyfile, ...)` | `Compose` | app + caddy + cloudflared + web network; `*_kw` add service settings (their `volumes` are appended); saves files if `root=` given |
| `litestream_conf(dbs, bucket, endpoint, region, ...)` | `str` | Write Litestream 0.5 `litestream.yml` (replica path `<prefix>/<db path>`) |
| `litestream_svc(vols, conf, env_file, cmd)` | `dict` | Litestream service kwargs for `Compose.svc()` (`cmd='replicate'` or a `restore ...`) |
| `cloud_init_sh(yaml)` | `str` | Shell-script equivalent of `vps_init()` cloud-config (used for Lightsail user data) |
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

### Lightsail production deploy

```python
from vpseasy.core import *
r = vps_deploy('myapp', './myapp', Lightsail(), bundle='small_3_2')   # Sydney by default
```

### SQLite backups with Litestream

```python
ls = Lightsail()
b = ls.bucket('myapp-backups')          # put b.key_id / b.secret in .env as LITESTREAM_ACCESS_KEY_ID / LITESTREAM_SECRET_ACCESS_KEY
caddy_stack('myapp.example.com', df, vols=['/app/data'], root=root,
            litestream=dict(dbs=['/app/data/myapp.db'], bucket=b.bucket, endpoint=b.endpoint, region=b.region))
```

For Cloudflare R2 pass `endpoint='https://<account>.r2.cloudflarestorage.com'` and leave `region='auto'`.

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
- SSH `StrictHostKeyChecking=accept-new` silently accepts new hosts but rejects changed host keys. `vps_deploy` (and so `hetzner_deploy`) runs `ssh-keygen -R <ip>` after provisioning to clear stale `known_hosts` entries when an IP is reused.
- Lightsail runs user data as a shell script, so `Lightsail.create()` converts `vps_init()` cloud-config with `cloud_init_sh()` (supports hostname, users, write_files, apt conf, packages, runcmd; other keys raise). Progress: `/var/log/vpseasy-init.log`, done marker `/var/lib/vpseasy/ready`.
- Lightsail bundle ids change over time (`small_3_0` is gone; current ones end in `_3_2`). List them with `Lightsail()._c.get_bundles()`.
- Litestream db paths must be absolute container paths inside one of the app's `vols`; `litestream_conf` raises `ValueError` otherwise. A one-shot `litestream-restore` service must complete before `app` starts (`condition: service_completed_successfully`); the `litestream` replicate service starts after `app`. Verified end to end with `litestream/litestream:0.5` (v0.5.17): empty bucket → restore exits 0; wiped server → db restored before `app`.
- Always use the private key path (no `.pub`) with `-i`. The public key is for `authorized_keys`, not for SSH client auth.
- `rollback` restores the app image only. Files `sync` copied (Compose file, Caddyfile, bind mounts) and volumes stay as the last deploy left them.
