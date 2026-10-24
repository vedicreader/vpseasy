# Release notes

<!-- do not remove -->

## 0.0.27

- `sync` and `sync_paths` protect Compose bind sources inside the deploy root from deletion. Local files in those bind sources still transfer. A bind source that names the whole deploy root raises `ValueError`. Add other paths with `exclude`. The local Compose file supplies the bind sources. Variable interpolation and override files are not resolved.
- `deploy` prunes build cache older than 168 hours. Set `prune=False` to skip pruning.
- Python 3.10 and 3.11 remain supported by the package metadata. SSH command f-strings use compatible quote delimiters.
- Existing rollback and image retention fixes preserve build tags and the previous running image. Selective sync rejects paths that reach or leave the deploy root.

## 0.0.26
- A rollback no longer orphans a build. `deploy` tags each new build with its build time after the build. `keep_image` deletes timestamped tags only past the newest `keep` by build time. It never deletes the tags of the running image and of `prev`. Before this change, two rollbacks in a row left the newest build untagged and out of reach of `undo`, and the next prune deleted it
- `prev` always names the image that ran before the current one. `deploy` sets it after a build that changed the image. `rollback` sets it to the image it replaces. `rollback(host, 'prev')` undoes either. `rollback` no longer writes the `undo` tag
- `rollback` without `to` picks the newest kept image that is older than the running one. Repeated calls keep going back instead of alternating
- `keep_image` takes `prev`, the image ID that ran before. It returns the tag of the running image even when that tag already existed. An image with two timestamps counts once toward `keep`. A build that shares the second of another build takes the next free second
- `deploy` tags the new build and sets `prev` even when the restart after the build fails
- `sync_paths` and `rsync_cmds` raise `ValueError` for a path that names the deploy root, such as `.`, `''` or `/`. They also raise it for a path that resolves to the deploy root or leaves it through `..` or a symlink. The `--delete` of such a path would reach the whole folder

## 0.0.25
- `wait_http` verifies HTTPS against the system trust store through `truststore`. A root added there, such as the root of a TLS-inspecting proxy, no longer makes every request fail. `wait_http` prints an error when it differs from the previous one instead of waiting in silence
- `server_health` leaves the stderr of `ssh`, such as a first-connection warning, out of the disk and container lines
- Needs truststore

## 0.0.24
- `keep_image` and `rollback` read image IDs without the stderr of `ssh`. A first-connection warning no longer breaks the match or the `docker tag` command
- A failed `docker rmi`, for a tag that a stopped container still uses, no longer stops the deploy before `docker compose up`
- The image name drops a leading `-` or `_` from the project name, as Compose does, and resolves `path='.'`. `vps_deploy` takes `img`
- `wait_http` keeps polling on a truncated or malformed HTTP response
- `rsync_cmds` adds the trailing slash to a folder named without one. A second run no longer nests the folder inside its copy

## 0.0.23
- `deploy` keeps the image it replaces, tagged with the UTC time and `prev`, and keeps the newest `keep=3`. `rollback(host, to=None)` points the service at a kept image and restarts it. It tags the replaced image `undo`. `keep_image` and `kept_images` are the building blocks
- `deploy` prunes dangling images and the build cache after the build unless `prune=False`
- `deploy`, `chk_docker`, `chk_cloud_init`, `wait_ready` and `vps_deploy` take `port`, like `sync`
- `sync` returns the rsync exit code. `check=False` makes it return a failing code instead of raising
- `sync_paths` copies chosen paths with one rsync for each path. `--delete` stays inside each path. It has an optional second pass for a `last` pattern, a `dry_run` option, and a local folder when `host` is None. `rsync_cmds` returns its commands
- `ssh_cmd`, `res_key` and `askpass_env` are public. `_ssh`, `_res_key` and `_askpass_env` remain as aliases
- `caddy_stack` takes `app_kw`, `caddy_kw` and `cloudflared_kw` for the settings of one service. Their `volumes` are appended to the volumes of the service. It also takes `caddyfile` and `write_caddyfile`. Without `conf`, it mounts `./Caddyfile` wherever `root` is
- `vps_init` adds a 4 GB swap file and sets swappiness to 10. `swap=None` skips it. `swap_cmds` returns those commands
- `wait_http` polls a URL until it answers 200. `server_health` reports memory, swap, load, disk and containers
- Needs dockeasy 0.0.9

## 0.0.22
`sync` raises `CalledProcessError` when rsync fails. `deploy` stops before `docker compose up`

## 0.0.21
`Multipass.rm` passes no empty argument to `multipass delete`. `sync` takes `port` for its SSH and rsync connections

## 0.0.19
`Lightsail` provider, `vps_deploy`, Litestream backups. `hetzner_deploy` arguments after `src` are keyword-only



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


