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
http://cpa-cli-proxy:8317
```

Use the same CPA management key you configured in `config.yaml`.

## 密钥配置说明

### 1. CPA 管理密钥（CLI Proxy API）

**配置文件**：`config.yaml`

**配置字段**：`remote-management.secret-key`

```yaml
remote-management:
  allow-remote: true
  secret-key: "your-cpa-management-key"
```

**说明**：
- 此密钥用于 CPA Manager Plus 连接 CLI Proxy API 时的认证
- 首次启动后，CPA 会自动将明文密钥转换为 bcrypt 哈希存储
- 转换后无法从哈希反推出原始密钥，需记住或另存原始值

**查看当前密钥**：
```bash
grep 'secret-key' config.yaml
```

### 2. CPA Manager Plus 管理员密钥

**存储位置**：`manager-data/data.key`（自动生成，为加密密钥，非登录密钥）

**管理员密钥格式**：`cmp_admin_` 开头的字符串

#### 首次部署

首次启动时，管理员密钥会自动生成。查看方式：

```bash
docker compose logs cpa-manager | grep -i "admin"
```

> ⚠️ 如果日志中未显示密钥，需要使用 `reset-admin-key` 命令生成。

#### 生成/重置管理员密钥

```bash
# 1. 停止服务
docker compose down

# 2. 生成新的管理员密钥（会显示一次，请立即保存）
docker run --rm -v $(pwd)/manager-data:/data seakee/cpa-manager-plus:latest reset-admin-key

# 3. 重新启动服务
docker compose up -d
```

输出示例：
```
CPA Manager Plus admin key reset.
New admin key: cmp_admin_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Save this value now. It will not be shown again.
```

#### 自定义管理员密钥

```bash
# 使用自定义密钥
docker run --rm -v $(pwd)/manager-data:/data seakee/cpa-manager-plus:latest reset-admin-key --admin-key "your-custom-key"

# 或从文件读取
docker run --rm -v $(pwd)/manager-data:/data seakee/cpa-manager-plus:latest reset-admin-key --admin-key-file /path/to/key.txt
```

#### 初始化 CPA Manager Plus

访问 `http://<服务器IP>:18317/management.html`，填写：

| 字段 | 值 | 说明 |
|-----|---|------|
| 管理员密钥 | `cmp_admin_xxx...` | 上一步生成的管理员密钥 |
| CPA 连接地址 | `http://cpa-cli-proxy:8317` | Docker 内部使用服务名互通 |
| CPA 管理密钥 | `config.yaml` 中的 `secret-key` | 即 CPA 管理密钥 |

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
