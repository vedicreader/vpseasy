

# vpseasy

Provision VPSes, test locally with Multipass, deploy Docker Compose apps
— all from Python.

## Setup

``` sh
pip install vpseasy              # Hetzner + Multipass
pip install 'vpseasy[aws]'       # adds boto3 for Lightsail
export HCLOUD_TOKEN=your_token   # required for Hetzner
export AWS_PROFILE=your_profile  # Lightsail: any standard AWS credential source (SSO profile, env vars)
```

## Providers

All providers share `servers()`, `get(name)`, `server_ip(name)`,
`create(name, cloud_init=...)` and `delete(name)`, so
`vps_deploy(name, src, provider, **create_kw)` works with any of them.

<table>
<colgroup>
<col style="width: 23%" />
<col style="width: 16%" />
<col style="width: 30%" />
<col style="width: 30%" />
</colgroup>
<thead>
<tr>
<th>Provider</th>
<th>Class</th>
<th>Credentials</th>
<th><code>create_kw</code></th>
</tr>
</thead>
<tbody>
<tr>
<td>Hetzner Cloud</td>
<td><code>Hetzner()</code></td>
<td><code>HCLOUD_TOKEN</code></td>
<td><code>image</code>, <code>server_type</code>,
<code>location</code></td>
</tr>
<tr>
<td>AWS Lightsail</td>
<td><code>Lightsail(region='ap-southeast-2')</code></td>
<td>AWS chain (<code>AWS_PROFILE</code>)</td>
<td><code>bundle</code> (<code>small_3_2</code>),
<code>blueprint</code>, <code>zone</code>, <code>static_ip</code>,
<code>ports</code>, <code>snapshots</code></td>
</tr>
<tr>
<td>Multipass (local)</td>
<td><code>Multipass()</code></td>
<td>none</td>
<td>use <code>deploy_mp</code>, not <code>vps_deploy</code></td>
</tr>
</tbody>
</table>

## Key functions

<table>
<colgroup>
<col style="width: 31%" />
<col style="width: 28%" />
<col style="width: 40%" />
</colgroup>
<thead>
<tr>
<th>Function</th>
<th>Returns</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>load_pub_keys(paths=None)</code></td>
<td><code>list[str]</code></td>
<td>Read pub keys from <code>~/.ssh/id_*.pub</code></td>
</tr>
<tr>
<td><code>gen_key(slug, key_dir=None)</code></td>
<td><code>AttrDict(key, pub, pub_str)</code></td>
<td>Generate ed25519 key pair</td>
</tr>
<tr>
<td><code>multi_init(hostname, pub_keys=None, ...)</code></td>
<td><code>AttrDict(yaml, key)</code></td>
<td>Cloud-init for Multipass (no UFW)</td>
</tr>
<tr>
<td><code>vps_init(hostname, pub_keys=None, ...)</code></td>
<td><code>AttrDict(yaml, key)</code></td>
<td>Cloud-init for prod VPS (UFW, fail2ban, Docker)</td>
</tr>
<tr>
<td><code>Multipass().launch(name, image, ...)</code></td>
<td><code>AttrDict(name, key)</code></td>
<td>Spin up a local Ubuntu VM</td>
</tr>
<tr>
<td><code>Multipass().ip(name)</code></td>
<td><code>str</code></td>
<td>Get VM IPv4</td>
</tr>
<tr>
<td><code>Multipass().rm(name)</code></td>
<td><code>None</code></td>
<td>Delete + purge VM</td>
</tr>
<tr>
<td><code>deploy_mp(name, src, path, build)</code></td>
<td><code>None</code></td>
<td>Rsync + <code>docker compose up</code> inside Multipass VM</td>
</tr>
<tr>
<td><code>Hetzner().create(name, ...)</code></td>
<td><code>AttrDict(ip, name, key, resp)</code></td>
<td>Provision Hetzner VPS</td>
</tr>
<tr>
<td><code>Hetzner().servers()</code></td>
<td><code>list[dict]</code></td>
<td>List servers <code>[{name, ip, status}]</code></td>
</tr>
<tr>
<td><code>Hetzner().delete(name)</code></td>
<td><code>None</code></td>
<td>Delete server</td>
</tr>
<tr>
<td><code>Hetzner().key_names()</code></td>
<td><code>list[str]</code></td>
<td>SSH key names registered in Hetzner (root access only)</td>
</tr>
<tr>
<td><code>Lightsail().create(name, bundle, ...)</code></td>
<td><code>AttrDict(ip, name, key, resp)</code></td>
<td>Provision Lightsail instance (idempotent, static IP, port 22 only,
daily snapshots)</td>
</tr>
<tr>
<td><code>Lightsail().bucket(name)</code></td>
<td><code>AttrDict(bucket, region, endpoint, key_id, secret)</code></td>
<td>Bucket (created if missing) + new access key for backups; Lightsail
allows two keys per bucket, secret shown only once</td>
</tr>
<tr>
<td><code>vps_deploy(name, src, provider, ...)</code></td>
<td><code>AttrDict(ip, name, key)</code></td>
<td>Full pipeline on any provider: provision → wait → deploy
(idempotent)</td>
</tr>
<tr>
<td><code>hetzner_deploy(name, src, ...)</code></td>
<td><code>AttrDict(ip, name, key)</code></td>
<td><code>vps_deploy</code> on Hetzner</td>
</tr>
<tr>
<td><code>wait_ssh(host, u, k, tout)</code></td>
<td><code>True</code></td>
<td>Poll until SSH is up</td>
</tr>
<tr>
<td><code>wait_ready(host, u, k, tout, retries)</code></td>
<td><code>True</code></td>
<td>Poll SSH then cloud-init until done; retries on timeout</td>
</tr>
<tr>
<td><code>chk_cloud_init(host, u, k)</code></td>
<td><code>str</code></td>
<td><code>done\|running\|error\|unknown</code></td>
</tr>
<tr>
<td><code>chk_docker(host, u, k)</code></td>
<td><code>bool</code></td>
<td>Verify Docker daemon running</td>
</tr>
<tr>
<td><code>run_ssh(host, *cmds, user, key, capture)</code></td>
<td><code>str\|CompletedProcess</code></td>
<td>Run commands over SSH</td>
</tr>
<tr>
<td><code>sync(host, src, path, user, key, include, exclude)</code></td>
<td><code>None</code></td>
<td>Rsync local dir to remote</td>
</tr>
<tr>
<td><code>deploy(host, src, path, user, build, key)</code></td>
<td><code>None</code></td>
<td><code>sync</code> + <code>docker compose up -d</code></td>
</tr>
<tr>
<td><code>vols_to_binds(vols)</code></td>
<td><code>list[str]</code></td>
<td><code>["/app/data"]</code> -&gt; <code>["./data:/app/data"]</code>
for Compose bind mounts</td>
</tr>
<tr>
<td><code>caddy_stack(domain, df, vols, root, ...)</code></td>
<td><code>Compose</code></td>
<td>app + caddy + cloudflared + web network; saves files if
<code>root=</code> given</td>
</tr>
<tr>
<td><code>litestream_conf(dbs, bucket, endpoint, region, ...)</code></td>
<td><code>str</code></td>
<td>Write Litestream 0.5 <code>litestream.yml</code> (replica path
<code>&lt;prefix&gt;/&lt;db path&gt;</code>)</td>
</tr>
<tr>
<td><code>litestream_svc(vols, conf, env_file, cmd)</code></td>
<td><code>dict</code></td>
<td>Litestream service kwargs for <code>Compose.svc()</code>
(<code>cmd='replicate'</code> or a <code>restore ...</code>)</td>
</tr>
<tr>
<td><code>cloud_init_sh(yaml)</code></td>
<td><code>str</code></td>
<td>Shell-script equivalent of <code>vps_init()</code> cloud-config
(used for Lightsail user data)</td>
</tr>
<tr>
<td><code>mv_skill_md(dry_run, dir)</code></td>
<td><code>None</code></td>
<td>Install SKILL.md to <code>.agents/</code> and
<code>~/.claude/</code></td>
</tr>
</tbody>
</table>

## Common patterns

### Local test with Multipass

``` python
from vpseasy.core import *
ci = multi_init('testvm')          # AttrDict(yaml, key) — auto-generates key pair
mp = Multipass()
vm = mp.launch('testvm', cloud_init=ci)
ip = mp.ip('testvm')
deploy_mp('testvm', src='./myapp')
mp.rm('testvm')
```

### Hetzner production deploy

`hetzner_deploy` is idempotent — re-running against an existing server
skips provisioning and goes straight to cloud-init check + deploy.

``` python
from vpseasy.core import *
hz = Hetzner()                          # reads HCLOUD_TOKEN
svr = hetzner_deploy('myapp-prod', './myapp', hz=hz, location='hel1')
print(f'Deployed at {svr.ip}, key: {svr.key}')
```

### Lightsail production deploy

``` python
from vpseasy.core import *
r = vps_deploy('myapp', './myapp', Lightsail(), bundle='small_3_2')   # Sydney by default
```

### SQLite backups with Litestream

``` python
ls = Lightsail()
b = ls.bucket('myapp-backups')          # put b.key_id / b.secret in .env as LITESTREAM_ACCESS_KEY_ID / LITESTREAM_SECRET_ACCESS_KEY
caddy_stack('myapp.example.com', df, vols=['/app/data'], root=root,
            litestream=dict(dbs=['/app/data/myapp.db'], bucket=b.bucket, endpoint=b.endpoint, region=b.region))
```

For Cloudflare R2 pass
`endpoint='https://<account>.r2.cloudflarestorage.com'` and leave
`region='auto'`.

### Lower-level Hetzner flow

Use `vps_init` + `Hetzner.create` + `wait_ready` + `deploy` when you
need more control:

``` python
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

Any dockeasy Dockerfile (fasthtml_app, go_app, rust_app, node_app,
python_app, …) can be deployed behind Cloudflare Tunnel with one call.
`caddy_stack` writes `Dockerfile`, `docker-compose.yml`, and `Caddyfile`
when `root=` is given; pass `**kw` through to `caddy_svc` for dns,
email, crowdsec, etc.

``` python
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

``` python
mv_skill_md(dry_run=False)   # writes to .agents/skills/vpseasy/ and ~/.claude/skills/vpseasy/
```

## SSH key resolution

`_resolve_key` resolves the SSH key in this order:

1.  Explicit `key=` path argument
2.  `name=` slug → `~/.ssh/<name>` (raises `FileNotFoundError` if
    missing)
3.  `SSH_KEY_PATH` environment variable
4.  `SSH_PRIVATE_KEY` environment variable (PEM string, written to
    `/tmp/.vpseasy_<uid>.pem`)
5.  `None` (falls back to SSH agent)

## Env vars

<table>
<colgroup>
<col style="width: 23%" />
<col style="width: 42%" />
<col style="width: 33%" />
</colgroup>
<thead>
<tr>
<th>Var</th>
<th>Used by</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>HCLOUD_TOKEN</code></td>
<td><code>Hetzner()</code></td>
<td>Required for Hetzner</td>
</tr>
<tr>
<td><code>SSH_KEY_PATH</code></td>
<td><code>_resolve_key()</code></td>
<td>Optional explicit key path</td>
</tr>
<tr>
<td><code>SSH_PRIVATE_KEY</code></td>
<td><code>_resolve_key()</code></td>
<td>PEM string, written to
<code>/tmp/.vpseasy_&lt;uid&gt;.pem</code></td>
</tr>
</tbody>
</table>

## Gotchas

- `ssh_keys` on `Hetzner.create()` injects into the **root** user only —
  not the `deploy` user created by cloud-init. Passing both `ssh_keys`
  and `cloud_init` raises `ValueError`.
- `cloud-init status` returns exit code 2 (“done with warnings”) on
  Ubuntu 24.04 — `chk_cloud_init` uses `check=False` to handle this
  correctly.
- SSH `StrictHostKeyChecking=accept-new` silently accepts new hosts but
  rejects changed host keys. `vps_deploy` (and so `hetzner_deploy`) runs
  `ssh-keygen -R <ip>` after provisioning to clear stale `known_hosts`
  entries when an IP is reused.
- Lightsail runs user data as a shell script, so `Lightsail.create()`
  converts `vps_init()` cloud-config with `cloud_init_sh()` (supports
  hostname, users, write_files, apt conf, packages, runcmd; other keys
  raise). Progress: `/var/log/vpseasy-init.log`, done marker
  `/var/lib/vpseasy/ready`.
- Lightsail bundle ids change over time (`small_3_0` is gone; current
  ones end in `_3_2`). List them with `Lightsail()._c.get_bundles()`.
- Litestream db paths must be absolute container paths inside one of the
  app’s `vols`; `litestream_conf` raises `ValueError` otherwise. A
  one-shot `litestream-restore` service must complete before `app`
  starts (`condition: service_completed_successfully`); the `litestream`
  replicate service starts after `app`. Verified end to end with
  `litestream/litestream:0.5` (v0.5.17): empty bucket → restore exits 0;
  wiped server → db restored before `app`.
- Always use the private key path (no `.pub`) with `-i`. The public key
  is for `authorized_keys`, not for SSH client auth.
