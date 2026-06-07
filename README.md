# all-dev-n8n

Secure, AI-native workflow automation — [n8n](https://n8n.io), packaged as a
strictly-confined, self-contained snap.

## Overview

n8n is a fair-code workflow automation platform: 400+ integrations, a visual
editor, native AI nodes, and a Code node for JavaScript/Python when you need it.

This snap is fully self-contained:
- **Bundles Node.js 22** (n8n requires Node ≥22.22; core24 apt only has 18).
- Installs the **official n8n npm release** (v2.23.4).
- Keeps **all state** — the SQLite database, the credential-encryption key,
  binary data and logs — under `$SNAP_COMMON/.n8n`. Nothing is written outside
  the snap.

## Quick Start

```bash
sudo snap install all-dev-n8n
```

Open `http://<host>:5678` and create the owner account on first run.

> n8n listens on `0.0.0.0:5678`. The auth cookie is served over plain HTTP so
> login works on a LAN. For remote/internet exposure, put a TLS-terminating
> reverse proxy in front.

## Configuration

The only operator-facing setting is the listen port (default `5678`):

```bash
sudo snap set all-dev-n8n port=5679   # restarts n8n automatically
```

Advanced settings: n8n is configured entirely by environment variables. The
launcher (`snap/local/n8n-run`) sets sensible defaults
(`N8N_USER_FOLDER`, `N8N_SECURE_COOKIE=false`, `N8N_RUNNERS_ENABLED=true`,
`DB_TYPE=sqlite`, diagnostics off). The persistent encryption key lives in
`/var/snap/all-dev-n8n/common/.n8n/config` — **back this up**; without it,
saved credentials cannot be decrypted.

## Services

| Service | Role |
|---|---|
| `all-dev-n8n.n8n` | The n8n editor/API/webhook server (port 5678) |
| `all-dev-n8n.ct-engine` | Control Tower reporting sidecar |

```bash
snap services all-dev-n8n
sudo snap logs -f all-dev-n8n.n8n
```

## Interfaces

Auto-connected via Control Tower (`reference/POST.json`):
`network`, `network-bind`, `home`, `removable-media`.

`network` lets workflow nodes make outbound calls; `home`/`removable-media`
let the local-filesystem nodes read/write files outside the snap. Note that
under strict confinement the **Execute Command** node can only run binaries
bundled in the snap.

## Documentation

- **[Deployment Guide](docs/deployment-guide.md)** — build, install, configure, back up, troubleshoot.

## License

n8n is distributed under the [Sustainable Use License](https://github.com/n8n-io/n8n/blob/master/LICENSE.md)
(fair-code). See upstream for details.
