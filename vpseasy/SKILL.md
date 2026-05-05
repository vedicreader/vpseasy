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
| `deploy_mp(name, src, path, build)` | `str` | Rsync + `docker compose up` inside Multipass VM |
| `Hetzner().create(name, ...)` | `AttrDict(ip, name, key, resp)` | Provision Hetzner VPS |
| `Hetzner().servers()` | `list[dict]` | List servers `[{name, ip, status}]` |
| `Hetzner().delete(name)` | `None` | Delete server |
| `Hetzner().key_names()` | `list[str]` | SSH key names registered in Hetzner |
| `wait_ssh(host, u, k, tout)` | `True` | Poll until SSH is up |
| `chk_cloud_init(host, u, k)` | `str` | `done\|running\|error\|unknown` |
| `chk_docker(host, u, k)` | `bool` | Verify Docker daemon running |
| `run_ssh(host, *cmds, user, key, capture)` | `str\|CompletedProcess` | Run commands over SSH |
| `sync(host, src, path, user, key, include, exclude)` | `None` | Rsync local dir to remote |
| `deploy(host, src, path, user, build, key)` | `None` | `sync` + `docker compose up -d` |
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

```python
from vpseasy.core import *
pub_keys = load_pub_keys()
ci = vps_init('myapp-prod', pub_keys)   # AttrDict(yaml, key=None when pub_keys given)
hz = Hetzner()                          # reads HCLOUD_TOKEN
svr = hz.create('myapp-prod', cloud_init=ci, ssh_keys=hz.key_names(), location='hel1')
wait_ssh(svr.ip, tout=300)
assert chk_cloud_init(svr.ip) == 'done'
deploy('./myapp', svr.ip)
```

### Install this skill

```python
mv_skill_md(dry_run=False)   # writes to .agents/skills/vpseasy/ and ~/.claude/skills/vpseasy/
```

## Env vars

| Var | Used by | Notes |
|-----|---------|-------|
| `HCLOUD_TOKEN` | `Hetzner()` | Required |
| `SSH_KEY_PATH` | `_resolve_key()` | Optional path override |
| `SSH_PRIVATE_KEY` | `_resolve_key()` | PEM string, written to `/tmp/.vpseasy_<uid>.pem` |
