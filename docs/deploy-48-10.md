# 48.10 部署与更新流程

本文档记录 Agent Kanban 在开发机 48.10（Mac Studio，用户 `jrtz-aigc`）上的部署方式和更新步骤。

## 部署信息

| 项 | 值 |
|---|---|
| 运行目录 | `~/Developer/agent-kanban/src` |
| 运行方式 | Docker Compose，两个容器：`src-app-1`（应用）、`src-postgres-1`（PostgreSQL 16） |
| 应用镜像 | `agent-kanban-app:local`（本机构建后传入，不在 48.10 构建） |
| 内网地址（AI 接入用） | `http://192.168.48.10:7331`，MCP 为 `/mcp` |
| 外网地址（给用户看） | `http://14.103.230.165:19006` |
| 外网隧道 | LaunchAgent `com.investoday.agent-kanban-tunnel`，SSH 反向隧道 `19006 -> 127.0.0.1:7331` |
| 配置 | `~/Developer/agent-kanban/src/.env`（权限 600，含 SESSION_SECRET 和管理员密码） |
| 数据 | Docker 命名卷 `src_ak_pgdata` |
| 备份 | 不做定时备份；需要时手动 `pg_dump` |

## 为什么不在 48.10 构建镜像

48.10 访问 Docker Hub 和 GitHub 会超时（连接被重置）。因此镜像在本机构建，用 `docker save` 传到 48.10，源码用 `git archive` 打包传输。两台机器都是 arm64。

## 更新步骤

1. 在本机拉取并合并最新代码，运行测试。
2. 构建并传输镜像：

```bash
cd <本机仓库目录>
docker build -t agent-kanban-app:local .
docker save agent-kanban-app:local | gzip | ssh dev-48-10 'gunzip | docker load'
```

3. 在 48.10 重启应用容器，数据卷保留：

```bash
ssh dev-48-10 'cd ~/Developer/agent-kanban/src && docker compose up -d'
```

4. 验证：外网地址返回 200，登录后看板正常。

如果 `docker-compose.yml` 有改动，需要同步到 48.10；注意 48.10 上的该文件把 `build: .` 改成了 `image: agent-kanban-app:local`。

## 常用命令

```bash
ssh dev-48-10 'docker ps --format "{{.Names}} {{.Status}}" | grep src-'
ssh dev-48-10 'cd ~/Developer/agent-kanban/src && docker compose logs app --tail 50'
ssh dev-48-10 'launchctl list | grep agent-kanban'
ssh dev-48-10 'docker exec src-postgres-1 pg_dump -U kanban kanban' > kanban-备份.sql
```

`docker` 已加入 48.10 的 PATH（写在 `~/.zshenv`），非交互 SSH 可直接使用。

## 端口占用

48.10 上已用的外网端口：19000（SSH）、19001（VNC 和开发服务）、19004、19005（Paperclip）、19006（本服务）、19820（QoderWake）。新增服务请避开。

## 接入 QoderWake

MCP 配置按数字员工逐个添加，且提案必须由人在 QoderWake 的同源 Web Console 中确认，CLI 无法代为确认。确认窗口只有 90 秒。

参数：名称 `kanban`，传输 `http`，地址 `http://192.168.48.10:7331/mcp`，请求头 `Authorization: Bearer <该员工的令牌>`。令牌名称必须与员工名称一致。

## MCP 主机白名单（必配）

MCP 端点带 DNS 重绑定保护，默认只放行 localhost。用内网 IP 或外网域名接入时，请求会被拒绝（HTTP 421 Invalid Host header），QoderWake 表现为「工具发现失败」。

在 `.env` 中增加以下变量（JSON 数组），并在 `docker-compose.yml` 的 `app.environment` 中传入 `MCP_ALLOWED_HOSTS` 和 `MCP_ALLOWED_ORIGINS`：

```
MCP_ALLOWED_HOSTS=["127.0.0.1:*","localhost:*","192.168.48.10:*","14.103.230.165:*"]
MCP_ALLOWED_ORIGINS=["http://127.0.0.1:*","http://localhost:*","http://192.168.48.10:*","http://14.103.230.165:*"]
```

新增访问地址时同步追加。MCP 地址请带结尾斜杠：`/mcp/`。

## 首次闭环验证记录（2026-11-20）

数字员工「付泽」通过 QoderWake 接入，领取任务、汇报进度、完成，网页可见全过程，发现 20 个工具。
