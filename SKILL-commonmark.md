

# vpseasy

Provision VPSes, test locally with Multipass, deploy Docker Compose apps
— all from Python.

## Setup

``` sh
pip install vpseasy
export HCLOUD_TOKEN=your_token   # required for Hetzner
```

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
<td><code>hetzner_deploy(name, src, ...)</code></td>
<td><code>AttrDict(ip, name, key)</code></td>
<td>Full pipeline: provision → wait → deploy (idempotent)</td>
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
  rejects changed host keys. `hetzner_deploy` runs `ssh-keygen -R <ip>`
  after provisioning to clear stale `known_hosts` entries when an IP is
  reused.
- Always use the private key path (no `.pub`) with `-i`. The public key
  is for `authorized_keys`, not for SSH client auth.
