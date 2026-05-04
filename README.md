# Hermes Agent Stack

Hermes Agent Stack 是一个本地优先的多服务编排仓库，用 Docker 把 Hermes Gateway、Hermes Dashboard、Hermes OpenAI-compatible API Server、Open WebUI、SkillServer 和 SearXNG 组合在一起。

当前默认模型链路是 GitHub Copilot / GitHub Models，Web 能力不走 Hermes 内建 `web` / `browser` toolset，而是统一改走本地 SkillServer。

## 当前架构

- `hermes` 容器运行 `gateway run`，同时暴露 Hermes API Server。
- `dashboard` 容器复用同一 Hermes 镜像，运行 Dashboard UI。
- `open-webui` 通过 `http://hermes:8642/v1` 接入 Hermes API Server。
- `skillserver` 提供本地 HTTP Web 能力，内部调用 SearXNG 和 Playwright。
- `searxng` 只在 Docker 网络内暴露，供 SkillServer 查询。
- 对宿主机开放的只有本地端口：Dashboard `9119`、Hermes API `8642`、Open WebUI `3000`，全部绑定 `127.0.0.1`。

```text
浏览器
  │
  ├── http://127.0.0.1:9119  -> Hermes Dashboard
  └── http://127.0.0.1:3000  -> Open WebUI
                                 │
                                 ▼
                       http://hermes:8642/v1
                                 │
                                 ▼
                           Hermes API Server
                                 │
                                 ▼
                            Hermes Gateway
                                 │
                ┌────────────────┴────────────────┐
                │                                 │
                ▼                                 ▼
      GitHub Copilot / GitHub Models       Local terminal tools
                                                  │
                                                  ▼
                                   http://skillserver:3000
                                                  │
                               ┌──────────────────┴──────────────────┐
                               ▼                                     ▼
                            SearXNG                           Playwright Chromium
```

## 仓库结构

```text
.
├── deploy.py
├── docker-compose.yml
├── Dockerfile.hermes
├── Dockerfile.openwebui
├── docker/
│   └── hermes-entrypoint.sh
├── hermes/
│   ├── config.yaml
│   ├── SOUL.md
│   └── skills/
│       └── web/
│           └── SKILL.md
├── searxng/
│   └── settings.yml
└── skillserver/
    ├── Dockerfile.skillserver
    ├── requirements.txt
    ├── server.py
    └── web_core.py
```

## 服务说明

| 服务 | 作用 | 宿主机端口 |
| --- | --- | --- |
| `hermes` | Hermes Gateway + API Server | `8642` |
| `dashboard` | Hermes Dashboard | `9119` |
| `open-webui` | Open WebUI 前端与聊天界面 | `3000` |
| `skillserver` | 本地 Web 能力 HTTP 服务 | 不直接暴露 |
| `searxng` | 搜索后端 | 不直接暴露 |

## 运行前提

- Docker Desktop 已启动
- Python 3.11+
- GitHub Copilot Enterprise 或 Business 账号

`deploy.py` 只依赖 Python 标准库，不要求先安装额外 Python 包。

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/JscSatoshi/deployagentstack.git
cd deployagentstack
```

### 2. 可选：创建本地虚拟环境

如果你想在宿主机上单独运行脚本或工具，可以创建本地 `.venv`：

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. 首次部署

```bash
python3 deploy.py
```

首次执行会自动完成这些步骤：

1. 检查 Docker 引擎
2. 写入 `.env` 默认值和随机密钥
3. 通过 GitHub Device Flow 获取 `GITHUB_TOKEN`
4. 构建 `hermes-agent:local` 与 `skillserver:local`
5. 启动整套容器
6. 依次检查 Dashboard、API Server、Open WebUI、SkillServer、SearXNG 连通性

启动完成后，默认访问地址：

- Hermes Dashboard: `http://127.0.0.1:9119`
- Open WebUI: `http://127.0.0.1:3000`
- Hermes API Server health: `http://127.0.0.1:8642/health`

## 常用命令

```bash
python3 deploy.py
python3 deploy.py --start
python3 deploy.py --stop
python3 deploy.py --build
python3 deploy.py --build --force
python3 deploy.py --newtoken
python3 deploy.py --check
python3 deploy.py --logs
```

命令说明：

- `python3 deploy.py`：完整部署；镜像不存在时自动构建，随后启动并检查服务
- `python3 deploy.py --start`：启动或重启现有容器，再做健康检查
- `python3 deploy.py --stop`：停止并移除容器
- `python3 deploy.py --build`：只构建镜像
- `python3 deploy.py --build --force`：无缓存重建镜像
- `python3 deploy.py --newtoken`：重新走 GitHub Device Flow，刷新 `.env` 里的 `GITHUB_TOKEN`
- `python3 deploy.py --check`：只做健康检查
- `python3 deploy.py --logs`：跟随查看容器日志

`deploy.py` 会自动优先使用 `docker compose`，如果环境里只有 `docker-compose` 也会自动回退。

## 配置文件

### `.env`

仓库提供 `.env.example`，实际运行时由 `deploy.py` 自动补默认值或生成密钥。

| 变量 | 说明 |
| --- | --- |
| `GITHUB_TOKEN` | GitHub Copilot / GitHub Models token |
| `HERMES_UID` | 容器内 Hermes 进程 UID |
| `HERMES_GID` | 容器内 Hermes 进程 GID |
| `HERMES_DASHBOARD_PORT` | Dashboard 本地端口，默认 `9119` |
| `API_SERVER_ENABLED` | Hermes API Server 开关，默认 `true` |
| `API_SERVER_PORT` | Hermes API Server 端口，默认 `8642` |
| `API_SERVER_KEY` | OpenAI-compatible API Bearer Key |
| `OPEN_WEBUI_PORT` | Open WebUI 本地端口，默认 `3000` |
| `TZ` | 时区，默认 `Asia/Shanghai` |
| `SEARXNG_SECRET` | SearXNG 密钥 |

### `hermes/config.yaml`

当前 Hermes 配置重点：

- 模型提供方：`copilot`
- 默认模型：`gpt-5.4`
- 终端后端：`local`
- 工作目录：`/workspace`
- 外部 skills 目录：`/workspace/hermes/skills`
- 显式禁用 Hermes 原生 `web` 和 `browser` toolset

这意味着仓库里的 Web 访问统一通过 `hermes/skills/web/SKILL.md` 指向本地 SkillServer。

### `hermes/SOUL.md`

Hermes 人设文件定义了默认交互风格：中文优先、直接、偏工程化、先做事再解释。

## Bootstrap 与持久化

`docker/hermes-entrypoint.sh` 在容器启动时会做这些事情：

1. 按 `.env` 的 `HERMES_UID` / `HERMES_GID` 调整容器内用户
2. 把 `hermes/config.yaml` 复制到持久化目录中的 `config.yaml`
3. 把 `hermes/SOUL.md` 复制到持久化目录中的 `SOUL.md`
4. 初始化 Hermes home 下的 `sessions`、`logs`、`memories`、`skills` 等目录

当前 Docker 卷：

- `hermes-home`：Hermes 持久化数据
- `open-webui`：Open WebUI 数据
- `screenshot-media`：SkillServer 生成的截图文件

## SkillServer 能力

SkillServer 是一个 FastAPI 服务，统一封装了搜索、网页抓取、结构化提取和截图功能。

当前接口：

| 接口 | 说明 |
| --- | --- |
| `/health` | 健康检查 |
| `/search` | 通过 SearXNG 做快速搜索 |
| `/deep_search` | 搜索后并发抓取页面正文 |
| `/navigate` | 抓取渲染后的页面内容，支持 `text` / `html` |
| `/extract_text` | 从 CSS selector 提取文本 |
| `/extract_links` | 提取页面链接 |
| `/headlines` | 提取页面标题层级 |
| `/screenshot` | 截图并返回 `MEDIA:` 路径 |

实现细节：

- `skillserver/server.py` 提供 FastAPI HTTP 接口
- `skillserver/web_core.py` 负责 SearXNG 查询、Playwright 渲染、URL 校验和并发控制
- 默认会阻止 `localhost`、回环地址和私有网络地址，避免代理访问宿主机内网资源

## Hermes Web Skill

`hermes/skills/web/SKILL.md` 明确要求 Hermes 不使用原生 web/browser toolset，而是通过终端执行：

```bash
curl -s --max-time 15 "http://skillserver:3000/search?q=YOUR+QUERY"
```

因此，这个仓库里的网页能力链路是：

1. Hermes 触发本地 `web` skill
2. skill 在终端里调用 `http://skillserver:3000/...`
3. SkillServer 再访问 SearXNG 或 Playwright
4. Hermes 整理结果后返回给用户

## Open WebUI 集成

Open WebUI 通过环境变量接入 Hermes：

- `OPENAI_API_BASE_URL=http://hermes:8642/v1`
- `OPENAI_API_KEY=${API_SERVER_KEY}`

注意点：

- Open WebUI 首次初始化时会读取这些环境变量
- 如果后续改了 API 地址或密钥，需要去 Open WebUI 后台设置里更新，或者清掉其数据卷后重新初始化

## 健康检查

`python3 deploy.py --check` 当前会依次验证：

1. Hermes Dashboard HTTP 可访问
2. Hermes API Server `/health` 可访问
3. Open WebUI HTTP 可访问
4. SkillServer `/health` 正常
5. SkillServer 容器内可访问 `http://searxng:8080/`
6. `hermes`、`hermes-dashboard`、`open-webui`、`skillserver`、`searxng` 容器都存在

## 排障建议

| 问题 | 处理方式 |
| --- | --- |
| `deploy.py` 提示 Docker 未运行 | 先启动 Docker Desktop，再重试 |
| Dashboard 打不开 | 运行 `python3 deploy.py --check`，确认 `9119` 未被占用 |
| Open WebUI 打不开 | 确认 `3000` 端口未冲突，并查看 `python3 deploy.py --logs` |
| Open WebUI 模型列表为空 | 确认 `http://127.0.0.1:8642/health` 正常，且连接地址带 `/v1` |
| GitHub 模型鉴权失败 | 运行 `python3 deploy.py --newtoken` 刷新 `GITHUB_TOKEN` |
| Hermes 不会搜索网页 | 检查 `hermes/config.yaml` 是否禁用了原生 web/browser，同时 `hermes/skills/web/SKILL.md` 是否存在 |
| SkillServer 抓网页失败 | 查看 SkillServer 日志，并确认目标 URL 不在私有网络或本地地址范围内 |
| 改了 `hermes/config.yaml` 或 `hermes/SOUL.md` 没生效 | 重新执行 `python3 deploy.py --start`，让 bootstrap 文件重新复制进 Hermes home |

## 开发备注

- `Dockerfile.hermes` 会克隆指定版本的 `NousResearch/hermes-agent`，在镜像内创建独立虚拟环境，并编译 Dashboard 前端
- `Dockerfile.openwebui` 本地构建 `open-webui:local`
- `skillserver/Dockerfile.skillserver` 在镜像内安装 Playwright Chromium 依赖
- 根目录 `.venv` 只是宿主机本地开发环境，不参与容器运行
