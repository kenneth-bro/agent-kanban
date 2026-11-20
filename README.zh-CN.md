# Agent Kanban（中文说明）

面向 AI 的轻量项目与任务看板：人在网页上看项目、任务、状态和进度；AI 通过 MCP 自己领任务、汇报、完成。看板本身是被动的，不启动也不控制任何 AI。

本仓库 fork 自 [hedburgaren/agent-kanban](https://github.com/hedburgaren/agent-kanban)（MIT），在其上增加了中文界面、领取超时回收和通用评审。

## 核心概念

| 概念 | 说明 |
|---|---|
| 项目 | 任务的归属，一个任务可属于一个项目 |
| 任务状态 | 待办 todo、就绪 ready、进行中 in_progress、待评审 review、已完成 done、阻塞 blocked、已取消 cancelled |
| 就绪 | 只有拖入「就绪」的任务才能被 AI 领取 |
| 令牌 | 每个数字员工一个令牌，在「管理」页生成，只显示一次 |

## 数字员工命名约定

令牌的 `agent_name` **必须使用数字员工的名称**，例如 `付泽`、`贾瓦`、`布舒`，与 QoderWake、StaffDeck 中的员工名称保持一致。

- 看板里「指定给」下拉框列出的就是已发放令牌的名称。
- 指定给某位员工的任务，只有该员工能领取。
- MCP 调用的 `agent` 参数必须与令牌名称完全一致，否则被拒绝。

## AI 接入（MCP）

端点：`http://<服务器地址>:7331/mcp`，协议 Streamable HTTP，请求头 `Authorization: Bearer <令牌>`。

```toml
[mcp_servers.kanban]
url = "http://<服务器地址>:7331/mcp"
headers = { Authorization = "Bearer <令牌>" }
```

常用工具：

| 工具 | 用途 |
|---|---|
| `get_next_task` | 取下一个可领任务 |
| `claim_task` | 原子领取，同一任务只有一个 AI 能领到 |
| `post_progress` | 汇报进度，会刷新超时计时 |
| `post_comment` / `get_comments` | 评论 |
| `request_review` | 提交评审，必须写明需要人决策的一句话 |
| `complete_task` | 完成任务 |
| `release_task` | 放回就绪池 |
| `list_projects` / `create_project` / `create_task` | 项目与任务管理 |

## 本 fork 的改动

| 改动 | 说明 |
|---|---|
| 中文界面 | 语言切换新增「中文」，中文浏览器默认使用 |
| 领取超时回收 | 进行中的任务超过 `CLAIM_TIMEOUT_MINUTES`（默认 120）分钟没有进展，自动回到就绪并留下系统记录；设为 0 关闭；检查间隔 `CLAIM_SWEEP_INTERVAL_SECONDS`（默认 60） |
| 评审改为通用 | 决策说明最少 2 个字符，记录前缀为「等待决策：」 |

## 本地运行

```bash
export SESSION_SECRET=$(python3 -c "import secrets;print(secrets.token_urlsafe(32))")
export AGENT_KANBAN_BOOTSTRAP_ADMIN_PASSWORD=<管理员密码>
docker compose up -d --build
```

访问 `http://127.0.0.1:7331`，管理员用户名 `admin`。数据在 PostgreSQL 的 Docker 卷 `ak_pgdata` 中。

## 测试

```bash
docker run -d --rm --name temp-akpg -e POSTGRES_USER=kanban -e POSTGRES_PASSWORD=kanban -p 5436:5432 postgres:16
uv run --extra dev pytest -q
```

## 部署与运维

见 [docs/deploy-48-10.md](docs/deploy-48-10.md)。
