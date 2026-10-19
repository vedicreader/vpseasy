# Release notes

<!-- do not remove -->

## 0.0.23
- `deploy` keeps the image it replaces, tagged with the UTC time and `prev`, keeping the newest `keep=3`; `rollback(host, to=None)` points the service at a kept image and restarts it, tagging the replaced one `undo`; `keep_image` and `kept_images` are the pieces
- `deploy` prunes dangling images and the build cache after the build unless `prune=False`
- `deploy`, `chk_docker`, `chk_cloud_init`, `wait_ready` and `vps_deploy` take `port`, like `sync`
- `sync` returns rsync's exit code and takes `check=False` to return it instead of raising
- `sync_paths` copies chosen paths with one rsync each, so `--delete` stays inside each path, with an optional second pass for a `last` pattern, `dry_run`, and a local folder when `host` is None; `rsync_cmds` returns its commands
- `ssh_cmd`, `res_key` and `askpass_env` are public; `_ssh`, `_res_key` and `_askpass_env` remain as aliases
- `caddy_stack` takes `app_kw`, `caddy_kw` and `cloudflared_kw` for per-service settings, whose `volumes` are appended, plus `caddyfile` and `write_caddyfile`; without `conf` it mounts `./Caddyfile` wherever `root` is
- `vps_init` adds a 4 GB swap file and sets swappiness to 10 (`swap=None` skips it); `swap_cmds` returns those commands
- `wait_http` polls a URL until it answers 200; `server_health` reports memory, swap, load, disk and containers
- Needs dockeasy 0.0.9

## 0.0.22
`sync` raises `CalledProcessError` when rsync fails, so `deploy` stops before `docker compose up`

## 0.0.21
`Multipass.rm` passes no empty argument to `multipass delete`; `sync` takes `port` for its SSH and rsync connections

## 0.0.19
lightsail provider, vps_deploy, litestream backups (`hetzner_deploy` args after `src` are keyword-only)



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


