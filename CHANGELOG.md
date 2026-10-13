# Release notes

<!-- do not remove -->

## 0.0.19
- `Lightsail` provider (AWS, boto3 via `pip install 'vpseasy[aws]'`): idempotent `create` with static IP, port 22 only and daily snapshots; `bucket()` returns a bucket and access key for backups
- `vps_deploy(name, src, provider)`: provider-neutral provision → cloud-init → deploy; `hetzner_deploy` now wraps it, and `Hetzner.get(name)` was added
- Breaking (positional callers only): `hetzner_deploy`'s 4th positional parameter is now `image` (was `user`); `user`, `key`, `path`, etc. are keyword-only
- Litestream: `litestream_conf`, `litestream_svc`, and `caddy_stack(..., litestream=dict(...))` stream SQLite dbs to S3/Lightsail/R2; a one-shot `litestream-restore` service restores missing dbs before `app` starts
- Lightsail user data: `cloud_init_sh()` turns `vps_init()` cloud-config into a shell launch script; `chk_cloud_init` honours its `/var/lib/vpseasy` markers

## 0.0.18
rsync progress bar



## 0.0.17
less pwd prompts



## 0.0.16




## 0.0.15
password bug fix



## 0.0.14
refactor



## 0.0.13
bug fix on deploy



## 0.0.12
use subprocess run everywhere



## 0.0.11
cloud-init canonical



## 0.0.10
pass user



## 0.0.9
pass key to hetzner_deploy



## 0.0.8
servers with password



## 0.0.7
sync takes extraextra



## 0.0.6
caddy stack and ssh updates



## 0.0.5
run_ssh uses subprocess now



## 0.0.4
caddy stack



## 0.0.3
vpseasy fixes



## 0.0.2
rsync, skill and smaller package



## 0.0.1
vps provisioning with multipass and hetzner


