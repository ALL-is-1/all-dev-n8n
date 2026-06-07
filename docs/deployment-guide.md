# n8n Snap — Deployment Guide

This snap packages **n8n v2.23.4** (official npm release) with a bundled
**Node.js 22** runtime, strictly confined, for `core24` / amd64.

---

## 1. Build

Snapcraft needs Linux + LXD (or multipass) — it **cannot** build on macOS.

```bash
cd to_snap/all-dev-n8n
snapcraft            # or: snapcraft --use-lxd
```

Build-time downloads (network is allowed in snapcraft build steps):

| Part | Source | Notes |
|---|---|---|
| `node` | nodejs.org Node **v22.22.3** linux-x64 tarball | n8n needs ≥22.22; core24 apt has 18 |
| `n8n`  | `npm install -g n8n@2.23.4` | pulls n8n + runtime deps (large) |

> The `n8n` npm install is heavy (hundreds of MB of `node_modules`) and may
> compile a native dep or two (`better-sqlite3`) — `build-essential` + `python3`
> are provided for that. `compression: xz` keeps the final download reasonable.

### Pinning / upgrading

- n8n version: bump `version:` — the `n8n` part installs `n8n@${CRAFT_PROJECT_VERSION}`,
  so the snap version and the installed n8n version stay in lockstep.
- Node: bump `NODE_VERSION` in the `node` part (keep ≥22.22).

---

## 2. Install

```bash
sudo snap install all-dev-n8n_2.23.4_amd64.snap --dangerous
```

Control Tower makes the interface connections from `reference/POST.json`.
Manually, only `removable-media` doesn't auto-connect:

```bash
sudo snap connect all-dev-n8n:removable-media
# network, network-bind, home auto-connect
```

Open `http://<host>:5678` and create the owner account.

---

## 3. Configure

| What | How |
|---|---|
| Port | `sudo snap set all-dev-n8n port=5679` (n8n auto-restarts) |
| Other settings | edit `snap/local/n8n-run` env (rebuild) or layer a drop-in proxy |

State tree (all under the snap common dir):

```
/var/snap/all-dev-n8n/common/.n8n/
├── config                # contains N8N_ENCRYPTION_KEY  ← BACK THIS UP
├── database.sqlite       # workflows, credentials (encrypted), executions
├── binaryData/           # binary data from executions
└── n8nEventLog*.log
```

### ⚠️ Back up the encryption key

`/var/snap/all-dev-n8n/common/.n8n/config` holds the credential-encryption key.
If you lose it (e.g. reinstall without preserving `$SNAP_COMMON`), every saved
credential becomes undecryptable. Back up `config` (and `database.sqlite`).

---

## 4. Verify

```bash
snap services all-dev-n8n
sudo snap logs -n=120 all-dev-n8n.n8n

# bundled runtime resolves under confinement:
sudo snap run --shell all-dev-n8n.n8n -c '
  $SNAP/usr/lib/node/bin/node --version;
  $SNAP/usr/lib/node/bin/node $SNAP/opt/n8n/lib/node_modules/n8n/bin/n8n --version'

# health endpoint:
curl -fsS http://localhost:5678/healthz && echo OK
```

Then load the editor, create the owner account, build a trivial workflow
(Manual Trigger → Set → Execute) and run it.

---

## 5. Troubleshooting

**Login page reloads / "cookie" errors over HTTP**
- Expected fix already applied: the launcher sets `N8N_SECURE_COOKIE=false`.
  If you front n8n with HTTPS, you can drop that for stricter cookies.

**Editor loads but webhooks return the wrong URL**
- Behind a reverse proxy, set `WEBHOOK_URL=https://your-host/` (add to the
  launcher env and rebuild). n8n derives webhook URLs from it.

**Code node fails / "task runner" warnings**
- Task runners are enabled (`N8N_RUNNERS_ENABLED=true`, internal mode — spawns a
  child `node`). If the child can't start under confinement, check
  `sudo snap logs all-dev-n8n.n8n` for the runner subprocess error.

**A node that shells out (Execute Command) fails**
- Strict confinement limits it to binaries inside the snap. Bundle the needed
  tool (add a `stage-packages` entry) or use a dedicated node instead.

**Native module / GLIBC errors at startup**
- The snap bundles its own Node; ensure the build host produced linux-x64
  prebuilds for `better-sqlite3`. Rebuild on a clean core24 LXD if in doubt.

**Port already in use** — `sudo snap set all-dev-n8n port=5679`.
