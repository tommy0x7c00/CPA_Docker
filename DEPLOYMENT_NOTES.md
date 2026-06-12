# CPA Docker Parallel Deployment Notes

记录日期：2026-06-10

## 部署目标

在不影响本机现有 CPA 的前提下，并行启动一套 Docker 版 CPA 和 CPA Manager Plus。

当前端口规划：

- 本机原 CPA：`http://localhost:8317`
- Docker 版 CPA：`http://localhost:8318`
- CPA Manager Plus：`http://localhost:18317/management.html`

## 工作目录

```bash
~/cpa-docker
```

目录内容：

```text
~/cpa-docker/config.yaml        # Docker 版 CPA 本地配置，包含密钥，勿提交
~/cpa-docker/config.example.yaml # 可提交的脱敏示例配置
~/cpa-docker/auths/             # Docker 版 CPA 认证文件副本，勿提交
~/cpa-docker/logs/              # Docker 版 CPA 日志目录，可能包含请求信息，勿提交
~/cpa-docker/manager-data/      # CPA Manager Plus 数据目录，勿提交
~/cpa-docker/docker-compose.yml # Docker Compose 配置
```

原始配置备份：

```text
~/cpa-backup-20260610-145138
```

## 数据准备

从本机 CPA 配置目录复制数据到 Docker 测试目录：

```bash
mkdir -p ~/cpa-docker/auths ~/cpa-docker/logs ~/cpa-docker/manager-data
rsync -a --delete ~/.cli-proxy-api/ ~/cpa-docker/auths/
cp ~/.cli-proxy-api/config.yaml ~/cpa-docker/config.yaml
```

## CPA 配置修改

修改文件：

```text
~/cpa-docker/config.yaml
```

成功使用的关键配置：

```yaml
host: ""
port: 8317

remote-management:
  allow-remote: true
  secret-key: "<CPA Management Key 的 bcrypt 哈希>"

auth-dir: "/root/.cli-proxy-api"
logging-to-file: true
proxy-url: "http://host.docker.internal:7890"
```

配置说明：

- `auth-dir` 改为容器内路径 `/root/.cli-proxy-api`
- `logging-to-file` 改为 `true`
- `remote-management.allow-remote` 改为 `true`，否则 Manager Plus 容器访问 CPA Management API 会被 CPA 返回 `403`
- `remote-management.secret-key` 先设置为明文，CPA 启动后自动转换为 bcrypt 哈希
- `proxy-url` 改为 `http://host.docker.internal:7890`，让 Docker CPA 使用 Mac 宿主机上的本地代理；容器内不能使用 `http://127.0.0.1:7890` 访问宿主机代理

本次 Docker 测试版 CPA Management Key 明文：

```text
<CPA_MANAGEMENT_KEY>
```

如需查看或重置本地实际密钥，请检查本机私有配置 `~/cpa-docker/config.yaml` 的
`remote-management.secret-key`。如果该字段已经被 CPA 转换为 bcrypt 哈希，无法从哈希反查明文，
应设置一个新的明文管理密钥后重启 CPA，让程序重新写入哈希。

## Docker Compose 配置

文件：

```text
~/cpa-docker/docker-compose.yml
```

成功配置：

```yaml
services:
  cli-proxy-api:
    image: eceasy/cli-proxy-api:latest
    container_name: cli-proxy-api-docker-test
    restart: unless-stopped
    ports:
      - "8318:8317"
      - "8085:8085"
      - "1455:1455"
      - "54545:54545"
      - "51121:51121"
      - "11451:11451"
    volumes:
      - ./config.yaml:/CLIProxyAPI/config.yaml
      - ./auths:/root/.cli-proxy-api
      - ./logs:/CLIProxyAPI/logs

  cpa-manager-plus:
    image: seakee/cpa-manager-plus:latest
    container_name: cpa-manager-plus
    restart: unless-stopped
    ports:
      - "18317:18317"
    volumes:
      - ./manager-data:/data
    depends_on:
      - cli-proxy-api
```

## 启动命令

```bash
cd ~/cpa-docker
docker compose up -d
```

查看状态：

```bash
docker compose ps
```

查看日志：

```bash
docker logs cli-proxy-api-docker-test
docker logs cpa-manager-plus
```

## Manager Plus 管理员密钥

Manager Plus 管理员密钥用于登录 Manager Plus 前端。

本次设置为：

```text
<MANAGER_PLUS_ADMIN_KEY>
```

重置命令：

```bash
cd ~/cpa-docker
docker compose stop cpa-manager-plus
docker compose run --rm cpa-manager-plus reset-admin-key --admin-key "<MANAGER_PLUS_ADMIN_KEY>"
docker compose up -d cpa-manager-plus
```

## Manager Plus 初始化

Manager Plus 前端地址：

```text
http://localhost:18317/management.html
```

Manager Plus 绑定 Docker CPA 时使用：

```text
CPA URL: http://cli-proxy-api:8317
CPA Management Key: <CPA_MANAGEMENT_KEY>
```

说明：

- `http://cli-proxy-api:8317` 是 Manager Plus 容器访问 CPA 容器的地址
- `http://localhost:8318` 是 Mac 宿主机访问 Docker CPA 的地址，不用于 Manager Plus 容器内部配置

本次成功初始化使用的接口请求：

```bash
curl -X POST http://localhost:18317/setup \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <MANAGER_PLUS_ADMIN_KEY>' \
  -d '{
    "cpaBaseUrl": "http://cli-proxy-api:8317",
    "managementKey": "<CPA_MANAGEMENT_KEY>",
    "ensureUsageStatisticsEnabled": true
  }'
```

成功返回：

```json
{"ok":true,"upstream":"http://cli-proxy-api:8317"}
```

## 验证命令

验证 Docker CPA API：

```bash
curl http://localhost:8318/
```

成功返回包含：

```json
{"message":"CLI Proxy API Server"}
```

验证 CPA Management API：

```bash
curl -H 'Authorization: Bearer <CPA_MANAGEMENT_KEY>' \
  http://localhost:8318/v0/management/config
```

成功返回 CPA 配置 JSON。

验证 Manager Plus 状态：

```bash
curl http://localhost:18317/usage-service/info
```

成功状态：

```json
{
  "configured": true,
  "setupRequired": false,
  "projectInitialized": true
}
```

## 当前成功状态

- 原本机 Homebrew CPA 已卸载
- 原本机 CPA 旧配置目录已从 `~/.cli-proxy-api` 移走并归档到 `~/cpa-old-homebrew-cli-proxy-api-20260610-172111`
- Docker 版 CPA 成功运行，端口 `8317`
- CPA Manager Plus 成功运行，端口 `18317`
- Manager Plus 已绑定 Docker CPA
- Docker CPA 已加载原有认证数据副本
- Manager Plus 数据持久化在 `~/cpa-docker/manager-data`

## 正式 Docker 端口

正式部署使用：

```text
http://127.0.0.1:8317
```

Docker Compose 端口映射：

```yaml
- "8317:8317"
```
