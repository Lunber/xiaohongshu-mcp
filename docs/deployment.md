# 小红书 MCP 部署文档

本文档用于在新机器上部署 `xiaohongshu-mcp` 服务，覆盖 Docker 部署、MCP Inspector 调试、登录流程、升级回滚与常见问题。

## 1. 部署目标与端口

- 服务监听端口：`18060`
- 健康检查：`GET /health`
- MCP 端点：`/mcp`
- HTTP API 前缀：`/api/v1`

示例地址（本机）：
- `http://localhost:18060/health`
- `http://localhost:18060/mcp`

---

## 2. 环境准备

目标机器需要：

- Docker（建议最新稳定版）
- Docker Compose（`docker compose` 子命令可用）
- 可访问部署机 `18060` 端口（本机、局域网或公网）

自检命令：

```bash
docker --version
docker compose version
```

---

## 3. 获取代码

```bash
git clone <你的仓库地址>
cd xiaohongshu-mcp
```

---

## 4. 选择部署方案

### 方案 A：使用预构建镜像（最快）

编辑 `docker/docker-compose.yml`，使用：

```yaml
image: xpzouying/xiaohongshu-mcp
```

启动：

```bash
cd docker
docker compose up -d
```

> 国内网络较慢时，可使用 compose 里注释的阿里云镜像地址。

### 方案 B：本地构建镜像（推荐可控）

#### 4.1 ARM64（Apple Silicon）

```bash
docker build -f Dockerfile.arm64 -t xiaohongshu-mcp:local .
```

`docker/docker-compose.yml` 使用：

```yaml
image: xiaohongshu-mcp:local
environment:
  - ROD_BROWSER_BIN=/usr/bin/chromium
  - COOKIES_PATH=/app/data/cookies.json
```

然后启动：

```bash
cd docker
docker compose up -d
```

#### 4.2 AMD64（Intel/AMD）

```bash
docker build -f Dockerfile -t xiaohongshu-mcp:local .
cd docker
docker compose up -d
```

---

## 5. 部署后验证

### 5.1 容器状态

```bash
docker ps --filter name=xiaohongshu-mcp
```

### 5.2 健康检查

```bash
curl http://localhost:18060/health
```

应返回 `success: true`。

### 5.3 查看日志

```bash
docker logs -f xiaohongshu-mcp
```

看到类似以下日志说明服务启动成功：

- `Registered 13 MCP tools`
- `启动 HTTP 服务器: :18060`

---

## 6. 使用 MCP Inspector 调试

### 6.1 启动 Inspector

在任意安装 Node.js 的机器执行：

```bash
npx @modelcontextprotocol/inspector
```

### 6.2 连接参数

在 Inspector 页面：

- Transport Type: `Streamable HTTP`
- URL: `http://<部署机IP>:18060/mcp`

注意：
- 本机调试可用 `localhost`
- 跨机器调试必须替换为部署机真实 IP

---

## 7. 首次登录流程（必须）

1. 调用 `check_login_status`
2. 未登录时调用 `get_login_qrcode`
3. 用小红书 App 扫码
4. 再次调用 `check_login_status` 确认已登录

Cookies 持久化路径：
- 容器内：`/app/data/cookies.json`
- 宿主机：`docker/data/cookies.json`

只要这个文件保留，容器重启后一般无需重复扫码。

---

## 8. 运行与维护命令

### 启动

```bash
cd docker
docker compose up -d
```

### 停止

```bash
cd docker
docker compose stop
```

### 重启

```bash
cd docker
docker compose restart
```

### 查看实时日志

```bash
docker logs -f xiaohongshu-mcp
```

### 进入容器

```bash
docker exec -it xiaohongshu-mcp bash
```

---

## 9. 升级与回滚

### 9.1 预构建镜像升级

```bash
cd docker
docker compose pull
docker compose up -d
```

### 9.2 本地构建升级

```bash
cd <项目根目录>
docker build -f Dockerfile.arm64 -t xiaohongshu-mcp:local .
cd docker
docker compose up -d
```

### 9.3 快速回滚（本地镜像）

回退到旧 tag：

```bash
docker images | grep xiaohongshu-mcp
# 将 compose 里的 image 改成旧 tag
cd docker
docker compose up -d
```

---

## 10. 常见问题排查

### 10.1 `can't find a browser binary for your OS`

原因：容器内缺少可用浏览器，且自动下载失败（常见于代理网络）。

处理：
- ARM64 使用 `Dockerfile.arm64`（镜像内安装 Chromium）
- 确认 `ROD_BROWSER_BIN=/usr/bin/chromium`

### 10.2 `chromium-browser requires snap`

原因：Ubuntu 上 `chromium-browser` 是 snap 占位，Docker 内无法用。

处理：
- 使用 `Dockerfile.arm64` 的 Debian 方案
- 浏览器路径改为 `/usr/bin/chromium`

### 10.3 Inspector 连不上

检查：
- `docker ps` 是否运行
- `curl http://<IP>:18060/health` 是否通
- 防火墙/安全组是否放行 `18060`
- Inspector URL 是否写成 `http://<IP>:18060/mcp`

### 10.4 `get_login_qrcode` 超时或失败

检查：
- 小红书 App 是否已打开并准备扫码
- 网络是否可访问小红书站点
- 查看服务日志定位浏览器/网络错误

---

## 11. 生产部署建议（可选）

可以在 `docker/docker-compose.yml` 增加：

```yaml
restart: unless-stopped
init: true
tty: true
```

并建议：
- 固定镜像版本 tag，避免 `latest` 漂移
- 将 `docker/data` 做备份（包含 cookies）
- 通过反向代理统一域名和访问控制（如 Nginx/Caddy）

---

## 12. 快速检查清单

- [ ] Docker 与 Compose 已安装
- [ ] 18060 端口已放通
- [ ] 容器已启动
- [ ] `/health` 返回成功
- [ ] Inspector 可连接 `/mcp`
- [ ] `get_login_qrcode` 可返回二维码
- [ ] 扫码后登录状态为已登录
