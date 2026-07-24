# pi-leo-sbx

A custom sandbox kit (`kind: sandbox`) that bundles:

- **Pi** — the [`@earendil-works/pi-coding-agent`](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) CLI
- **OmniRoute** extension — pre-installed and configured to connect to `http://host.docker.internal:20128`
- **Git SSH signing** — commits and tags signed automatically with your host's SSH key
- **GitHub SSH** — host keys pre-populated so `git push`/`pull`/`clone` work without prompts

## Usage

```console
$ sbx run --kit "git+https://github.com/Leoo0x1/pi-leo-sbx.git" pi-leo-sbx
```

The first launch installs everything. Subsequent launches reuse the sandbox.

## Setting up your OmniRoute API key (secure)

The API key is handled through the sandbox proxy's **sentinel-swap** mechanism:
the container stores the placeholder `proxy-managed`, and the proxy replaces it
with your real key at request time. Your key **never enters the container**.

### Option A: via `sbx secret set` (easiest)

On your host, run:

```console
$ sbx secret set <sandbox-name> omniroute
```
When prompted input <your-omni-api-key>

### Option B: via credentials file

Add to `~/.config/sbx/credentials.yaml` on your host:

```yaml
bindings:
  omniroute:
    discovery:
      - env: [OMNIROUTE_API_KEY]
    allowedDomains:
      - host.docker.internal:20128
```

And set the env var:

```console
$ export OMNIROUTE_API_KEY=<your-key>
```

Either way, the proxy intercepts requests from pi to OmniRoute, swaps
`proxy-managed` for your real key, and forwards the request. The container
never sees the real value.

## Prerequisites for signing and push

On your host, load your SSH key and register it with GitHub:

```console
$ ssh-add ~/.ssh/id_ed25519
```

Inside the sandbox, verify SSH agent forwarding:

```console
$ ssh-add -L
ssh-ed25519 AAAA... you@example.com
```

## Verifying

```console
$ git log --show-signature -1
commit abc1234...
Good "git" signature for you@example.com with ED25519 key SHA256:...

$ git push origin main
Everything up-to-date
```

## What you get

| Capability | How |
|---|---|
| **Pi agent** | Installed via `npm install -g` at creation |
| **OmniRoute** | Extension pre-installed, configured for `host.docker.internal:20128`, API key via proxy sentinel-swap |
| **Signed commits** | Git configured to sign with the forwarded SSH key automatically |
| **SSH push/pull** | GitHub host keys in `known_hosts` — no interactive prompts |

## Network

| Domain | Purpose |
|---|---|
| `host.docker.internal:20128` | OmniRoute server on the host (via proxy) |
| `registry.npmjs.org` | npm install for pi |
| `github.com:22`, `github.com:443` | SSH push/pull + HTTPS clone |
| `api.github.com:443` | Fetch GitHub host keys |

## Install order (runs once at creation)

1. **npm** — install pi globally (agent user, with retries)
2. **OmniRoute** — write config (`apiKey: proxy-managed`), then `pi install git:github.com/Leoo0x1/omniroute-pi-ext`
3. **Git config** — enable SSH commit signing system-wide (root)
4. **known_hosts** — fetch GitHub host keys from `api.github.com/meta` (root)
