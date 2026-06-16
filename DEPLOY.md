# 部署指南

## 部署方案总览

| 方案 | 适用场景 | 端口 | 防火墙 |
|------|----------|------|--------|
| **Docker Compose**（推荐） | 服务器长期运行 | 8501 | Docker 自动放行 |
| pip install + streamlit | 本地开发调试 | 8501 | 需手动开放 |
| CLI 直接运行 | 一次性分析 | - | - |

---

## 方案一：Docker Compose（推荐）

### 前提

- 已安装 Docker + Docker Compose v2
- 项目根目录有 `.env` 文件（含 LLM API Key）

### 启动

```bash
# Web UI（默认服务）
docker compose up -d web

# 访问 http://<服务器IP>:8501
```

### 停止

```bash
docker compose down
```

### 查看日志

```bash
docker compose logs -f web
```

### 更新

```bash
git pull
docker compose down
docker compose up -d --build web
```

### 容器说明

```
tradingagents-astock-web-1  →  Streamlit Web UI（端口 8501）
```

`docker-compose.yml` 中定义了以下服务，但 **默认 `up` 只启动 `web`**：

| 服务 | 状态 | 说明 |
|------|------|------|
| `web` | ✅ 默认启动 | Streamlit 界面，端口 8501 |
| `tradingagents` | ⏸️ 注释状态 | CLI 交互式终端，需要时用 `docker compose run --rm tradingagents` 临时起 |
| `proxy` + `mysql` | ⏸️ profile | 第三方中转站，`docker compose --profile proxy up -d` |
| `tradingagents-proxy` | ⏸️ profile | 连接中转站的 Web 实例（8502 端口） |
| `ollama` | ⏸️ profile | 本地 Ollama，`docker compose --profile ollama up -d` |
| `tradingagents-ollama` | ⏸️ profile | 连接 Ollama 的 Web 实例（8503 端口） |

### 为什么不常驻 CLI 容器？

原 `docker-compose.yml` 中 `tradingagents` 服务配置了 `tty: true` + `stdin_open: true`，这是为交互式终端设计的。但在 `docker compose up -d` 后台模式下，CLI 容器没有人交互，变成空跑进程浪费资源。因此已改为注释状态，需要时用：

```bash
docker compose run --rm tradingagents
```

临时启动一个交互式 CLI 实例，退出后自动清理。

### Profile 模式

```bash
# 启动 One-API 中转站 + 对应的 Web 实例
docker compose --profile proxy up -d

# 启动 Ollama + 对应的 Web 实例
docker compose --profile ollama up -d

# 同时启动多个 profile
docker compose --profile proxy --profile ollama up -d
```

### 第三方中转站配置

已有 One-API / New-API 等中转站运行在宿主机 `localhost:3000`：

```bash
# .env 中配置
LLM_PROVIDER=proxy
PROXY_API_KEY=sk-your-proxy-key
BACKEND_URL=http://host.docker.internal:3000/v1

docker compose up -d web
```

> **注意**：macOS/Windows Docker 用 `host.docker.internal` 访问宿主机；Linux 改为 `172.17.0.1`。

### 数据持久化

Docker volume `tradingagents_data` 持久化以下数据：
- `~/.tradingagents/` 目录下的 checkpoint（SQLite 断点续跑）
- 分析历史缓存

```bash
# 查看 volume
docker volume ls | grep tradingagents

# 备份
docker run --rm -v tradingagents_data:/data -v $(pwd):/backup alpine tar czf /backup/ta-data.tar.gz -C /data .
```

---

## 方案二：pip install（本地开发）

### 安装

```bash
git clone https://github.com/simonlin1212/tradingagents-astock.git
cd tradingagents-astock
pip install -e .
```

### 启动 Web UI

```bash
streamlit run web/launch.py --server.port 8501 --server.address 0.0.0.0
```

### 启动 CLI

```bash
tradingagents
```

### 防火墙注意

pip 方式直接绑定端口，**不受 Docker iptables 规则保护**。如果服务器启用了 ufw，需要手动放行：

```bash
sudo ufw allow 8501/tcp
```

---

## 方案三：CLI 直接运行

```bash
cd tradingagents-astock
pip install -e .
tradingagents
```

按提示输入股票代码、日期、分析偏好等参数即可开始分析。

---

## 常见问题

### 端口 8501 无法访问

1. **Docker 部署**：Docker 自动放行映射端口，如果仍不通，检查 `docker ps` 确认容器 Running
2. **pip 部署**：检查 `ufw status`，确保 `8501/tcp` 已 allow
3. 检查服务器安全组（云主机）是否放行 8501 端口

### OOM Killed

容器退出码 **137** 表示被系统内存不足强杀。多 Agent 并发调用 LLM 时内存占用较高，建议：
- 服务器内存 ≥ 4GB
- 或限制并发轮数（`max_debate_rounds`, `max_risk_discuss_rounds` 设为 1）

### 更新依赖

```bash
# Docker
docker compose down
docker compose build --no-cache
docker compose up -d

# pip
pip install -e . --upgrade
```
