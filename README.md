# CPA Docker

Docker Compose setup for running CLI Proxy API with CPA Manager Plus.

This repository is intended to keep only reusable deployment files. Local
configuration, OAuth credentials, logs, and Manager Plus data are ignored by
Git and must stay private.

## Included Files

- `docker-compose.yml`: starts CLI Proxy API and CPA Manager Plus.
- `config.example.yaml`: sanitized CPA configuration template.
- `DEPLOYMENT_NOTES.md`: detailed deployment notes and troubleshooting commands.
- `.gitignore`: excludes local credentials, logs, databases, and backup archives.

## Private Local Files

Do not commit these paths:

- `config.yaml`
- `auths/`
- `logs/`
- `manager-data/`

`config.yaml` may contain management keys and upstream API keys. `auths/` stores
OAuth credential files. `logs/` may contain request or error details.
`manager-data/` contains Manager Plus database and key material.

## Quick Start

Create local runtime files:

```bash
cp config.example.yaml config.yaml
mkdir -p auths logs manager-data
```

Edit `config.yaml` and set local values:

- `remote-management.secret-key`: set a new CPA management key.
- `api-keys`: set client API keys if you want CPA to require them.
- `proxy-url`: keep `http://host.docker.internal:7890` when the proxy runs on
  the Docker host, or change it for your environment.

If you already have a local CLI Proxy API auth directory, copy it into this
project before starting Docker:

```bash
rsync -a ~/.cli-proxy-api/ ./auths/
```

Start services:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

## Service URLs

- CLI Proxy API: `http://127.0.0.1:8317`
- CPA Manager Plus: `http://127.0.0.1:18317/management.html`

Inside Docker, Manager Plus should connect to CPA through:

```text
http://cli-proxy-api:8317
```

Use the same CPA management key you configured in `config.yaml`.

## Manager Plus Admin Key

Set or reset the Manager Plus admin key with:

```bash
docker compose stop cpa-manager-plus
docker compose run --rm cpa-manager-plus reset-admin-key --admin-key "<MANAGER_PLUS_ADMIN_KEY>"
docker compose up -d cpa-manager-plus
```

Keep this key private.

## Verifying CPA

```bash
curl http://127.0.0.1:8317/
```

Management API requests require the CPA management key:

```bash
curl -H 'Authorization: Bearer <CPA_MANAGEMENT_KEY>' \
  http://127.0.0.1:8317/v0/management/config
```

## Secret Handling

If `remote-management.secret-key` has already been converted to a bcrypt hash by
CPA, the plaintext key cannot be recovered from that hash. Set a new plaintext
key in local `config.yaml`, restart CPA, and let CPA write the new hash.

Before publishing changes, scan tracked files:

```bash
git ls-files
rg -n -i "(sk-|bearer|token|secret|password|api[_-]?key|账号|密码|密钥)" .
```

Review any matches and keep real secrets only in ignored local files.
