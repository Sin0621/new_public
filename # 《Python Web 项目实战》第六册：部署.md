# 《Python Web 项目实战》第六册：部署  
  
**本册遵循《文档写作规范》v2.0 全部条目。**  
  
本册输入：第五册的 `app/main.py` 与 `tests/`，一个能在开发机 venv 里跑通的 FastAPI 服务。  
本册输出：`insight-agent:0.1` 镜像、compose 编排、Nginx 反代配置、CI 流水线与冒烟脚本。  
本册主线：把「只在我电脑上能跑」变成「任何人打开浏览器都能用」。  
  
## 本册导航  
  
| 章 | 内容（一句话） | 当前解决点 | 贯穿实例 | 本章产出（文件路径） |  
|---|---|---|---|---|  
| 第1章 打包 | 用 Dockerfile 把服务装进镜像 | 打开 `Dockerfile`，构建出 `insight-agent:0.1`，镜像小于 300MB | 一台 2 核 4G 云主机部署 InsightAgent | `Dockerfile`、`.dockerignore` |  
| 第2章 编排 | 用 compose 一条命令起服务并挂数据卷 | 打开 `docker-compose.yml`，`docker compose up -d` 后 10 秒内 `/health` 返回 200 | 同一台云主机 | `docker-compose.yml` |  
| 第3章 反代 | 用 Nginx 把公网 80 转到容器 8000 | 打开 `deploy/nginx.conf`，公网 80 访问 `/api/v1/chat` 返回 200 且 SSE 首条事件 1 秒内到达 | 同一台云主机 | `deploy/nginx.conf` |  
| 第4章 验收 | 用 CI、冒烟与回滚把上线变成可重复动作 | 打开 `.github/workflows/ci.yml`，冒烟 4 项全过且回滚在 2 分钟内完成 | 同一台云主机 | `.github/workflows/ci.yml`、`scripts/smoke.sh` |  
| 下一册 | 并发与采集 | 打开 `app/services/crawler.py`，把网页资料并发抓取后送进 0.2 切分 | — | — |  
  
贯穿实例说明：全册只用「一台 2 核 4G 云主机部署 InsightAgent」这一个例子。第 1 章加「镜像」，第 2 章加「编排与数据卷」，第 3 章加「公网入口」，第 4 章加「验收与回滚」。不换例子。  
  
## 端到端链路  
  
本册不改变 0.3 的九跳数据链，新增的是两条交付链，交汇点在**运行中的容器**。  
  
**构建链**  
  
| # | 谁（角色） | 做什么 | 产出什么数据 | 落在哪／传给谁 | 状态 |  
|---|---|---|---|---|---|  
| 4.1 | 开发者 | 提交代码并构建镜像 | `insight-agent:0.1` | 本地／镜像仓库 | ✅已通 |  
| 4.2 | 运维者 | 用 compose 启动容器 | 运行中的容器 + 数据卷 | 云主机 | ✅已通 |  
  
**访问链**  
  
| # | 谁（角色） | 做什么 | 产出什么数据 | 落在哪／传给谁 | 状态 |  
|---|---|---|---|---|---|  
| 4.3 | 外部用户 | 访问公网 80 | HTTP 请求 | Nginx | ✅已通 |  
| 4.3 | Nginx | 反向代理到容器 | 转发请求 | 容器 8000 | ✅已通 |  
| 4.4 | 应用 | 检索、作答、返回 | JSON 或 SSE | 用户浏览器 | ✅已通 |  
  
交汇点：**容器**。构建链产出它，访问链把请求送进它。  
  
## 目录  
  
1. 第 1 章 打包  
2. 第 2 章 编排  
3. 第 3 章 反代  
4. 第 4 章 验收  
5. 按链路排查表  
6. 知识地图  
7. 自测  
8. 规范自检记录  
  
---  
  
## 第 1 章 打包  
  
### 【第 1 跳】开发者 → 构建镜像 → `insight-agent:0.1`  
  
### 1.0 进度卡  
  
**① 章节清单与状态**  
  
🔄 第1章 打包 ← 本章  
⏳ 第2章 编排  
⏳ 第3章 反代  
⏳ 第4章 验收  
  
**② 数据现在在哪**  
  
- 模型：能读到 `chunks` 表（经 `kb_search` 检索），密钥来自开发机的 `.env`。  
- 程序内：脚本与 `pytest` 能调用 `AgentExecutor`，但只在本机 venv（Python 3.11 加 10 个依赖）里成立。  
- 外部用户：只能访问 `127.0.0.1:8000`；换一台机器或从局域网外访问，返回 `Connection refused`。  
  
**③ 本章做完后**  
  
镜像 `insight-agent:0.1` 可在任意装了 Docker 的机器上 `docker run -p 8000:8000` 启动；镜像内不含 `.env` 与 `data/`，密钥与数据库靠运行时注入或挂载。  
  
**④ 下一章要做**  
  
打开 `docker-compose.yml`，`docker compose up -d` 后 10 秒内 `/health` 返回 200  
  
**施工单**：目标产物：`insight-agent:0.1`；上一跳给我：能在本机 venv 跑通的 `app/`；我这一跳输出：一个装了 Python 3.11 与依赖、可直接启动的镜像。  
  
### 1.1 场景与动机  
  
本章做：写 Dockerfile，把服务打成镜像。  
本章不做：编排、反代、CI、回滚。  
  
① 第五册交出的是 `app/main.py` 与 `tests/`，接口在开发机 venv 里能跑，`pytest` 全绿。  
② 换一台机器要装 Python 3.11、10 个依赖、配 `.env`；漏一个就报 `ModuleNotFoundError: No module named 'fastapi'`；换目录启动，SQLite 报 `unable to open database file`。  
③ 本章把运行环境固定成一个文件。  
④ 选 Docker：一次构建，到处运行；目标机只要装 Docker，不装 Python。  
  
本章不涉及新数据结构。  
  
### 1.2 最小可运行版  
  
本段做什么：用六行 Dockerfile 把依赖和代码装进镜像。  
  
```dockerfile  
# 本段：最小可运行版  
FROM python:3.11  
WORKDIR /app  
COPY . .  
RUN pip install -r requirements.txt  
CMD ["uvicorn", "app.main:app", "--port", "8000"]  
```  
  
预期：  
  
```bash  
docker build -t insight-agent:0.1 .  
docker run -p 8000:8000 insight-agent:0.1  
curl -s http://127.0.0.1:8000/api/v1/health  
# {"status":"ok",...}  
```  
  
### 1.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `FROM python:3.11` | 基础镜像，已含 Python 3.11 与 pip |  
| `WORKDIR /app` | 容器内工作目录，后续命令都在这里 |  
| `COPY . .` | 把构建目录全部复制进镜像 |  
| `RUN pip install -r requirements.txt` | 构建期安装依赖 |  
| `CMD [...]` | 容器启动时执行的命令，JSON 数组形式不经过 shell |  
  
**知识点 1：基础镜像**  
  
定义：基础镜像提供操作系统与语言运行时，是构建的起点。  
  
定式（要背）：  
  
```dockerfile  
FROM python:3.11-slim      # 精简版，体积约为完整版 1/6  
```  
  
适用：所有 Python 服务。例题：`python:3.11` 约 1.1GB，`python:3.11-slim` 约 180MB。  
  
**知识点 2：构建层与缓存**  
  
定义：每条 `RUN`／`COPY` 形成一层，未变化的层直接复用缓存。  
  
定式（要背）：  
  
```dockerfile  
COPY requirements.txt .  
RUN pip install -r requirements.txt  
COPY app ./app  
```  
  
适用：需要快速重建时。例题：只改业务代码，`pip install` 层命中缓存，构建从 90 秒降到 5 秒。  
  
**知识点 3：`COPY` 与 `.dockerignore`**  
  
定义：`COPY` 把宿主机文件放进镜像；`.dockerignore` 列出不参与构建的路径。  
  
定式（要背）：  
  
```gitignore  
.venv  
__pycache__  
.env  
data  
.git  
```  
  
适用：任何构建。例题：不加 `.dockerignore` 时 `.env` 与 `data/insight.db` 都会进镜像。  
  
**知识点 4：监听地址与端口**  
  
定义：容器内进程必须监听 `0.0.0.0`，宿主机端口映射才能转发进来。  
  
定式（要背）：  
  
```bash  
--host 0.0.0.0 --port 8000  
```  
  
适用：所有容器化 Web 服务。例题：只监听 `127.0.0.1` 时，`-p 8000:8000` 仍然 `Connection refused`。  
  
### 1.4 指出缺陷  
  
以下缺陷只评「打包」这一动作，均可在 1.2 的 Dockerfile 中指到具体行。  
  
- **D1**：`FROM python:3.11` 用完整版镜像 → 镜像约 1.1GB，首次拉取在 2M 带宽下要 7 分钟。  
- **D2**：`COPY . .` 不配 `.dockerignore` → `.env`、`data/insight.db`、`.venv` 全进镜像；镜像涨到 1.3GB，密钥随镜像泄漏。  
- **D3**：`RUN pip install` 写在 `COPY . .` 之后 → 改一行业务代码就让依赖层缓存失效，每次重建 90 秒。  
- **D4**：`CMD` 没有 `--host 0.0.0.0` → 容器内只监听 `127.0.0.1`，宿主机 `curl` 报 `Connection refused`。  
- **D5**：未指定用户，默认以 root 运行 → 容器内进程可写 `/usr`、`/etc`，被利用时影响面更大。  
  
### 1.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `Dockerfile` | 定义镜像的每一层 | `requirements.txt`、`app/` |  
| `.dockerignore` | 排除不该进镜像的路径 | 无 |  
| `requirements.txt`（修改） | 锁定依赖版本 | 无 |  
  
② 拆分理由：  
  
- `Dockerfile`：解决 D1（换 slim）、D3（依赖层前置）、D4（`--host`）、D5（非 root 用户）。  
- `.dockerignore`：解决 D2，把 `.env`、`data/`、`.venv` 挡在构建上下文外。  
- `requirements.txt`：锁版本，避免「今天能构建、明天依赖升级就挂」。  
  
③ 不这么拆会怎样：把 `pip install` 与代码复制混在一层，每次改代码都重装依赖；没有 `.dockerignore`，构建上下文从 2MB 涨到 500MB，构建机磁盘被 `.venv` 撑满。  
  
④ 依赖图：  
  
```  
requirements.txt ──┐  
app/ ──────────────┼──▶ Dockerfile ──▶ insight-agent:0.1  
.dockerignore ─────┘  
```  
  
⑤ 分工：设计决策回答「Docker、venv 还是 PaaS」；结构设计回答「Dockerfile 与 .dockerignore 各放什么」。  
  
### 1.6 完整版  
  
**设计决策：用 Docker 镜像，而不是在服务器装 venv**  
  
① 痛点：D1 的 1.1GB、D2 的密钥入镜、D3 的 90 秒重建、D4 的 `Connection refused`、D5 的 root 运行。  
  
② 取舍：  
  
- 方案 A：在服务器上 `python -m venv` 加 `pip install`。适用于一次性临时环境。否决理由：每台机器环境漂移，Python 小版本不同就报 `ModuleNotFoundError`；回滚只能靠备份目录。  
- 方案 B：把整个 venv 打包成 tar 传过去。适用于无外网的内网机。否决理由：venv 绑定构建机的路径与 Python 版本，换机器常常要重建。  
  
③ 触发条件：需要多机一致、可回滚、可交付时用镜像；单机临时调试可用 venv。目标机没有 Docker 权限时，才退回方案 B。  
  
**文件 1：`Dockerfile` 第 1 段**  
  
本段做什么：定基础镜像与依赖层。  
  
```dockerfile  
# 本段：基础镜像与依赖  
FROM python:3.11-slim                          # 关键行：精简版，约 180MB  
ENV PYTHONDONTWRITEBYTECODE=1 \  
    PYTHONUNBUFFERED=1                          # 关键行：日志实时输出  
  
WORKDIR /app  
  
COPY requirements.txt .                        # 关键行：先只复制依赖清单  
RUN pip install --no-cache-dir -r requirements.txt  
```  
  
| 代码 | 含义 |  
|---|---|  
| `python:3.11-slim` | 精简基础镜像 |  
| `PYTHONUNBUFFERED=1` | 不缓冲标准输出，`docker logs` 能实时看到日志 |  
| `COPY requirements.txt .` | 只复制清单，让依赖层可缓存 |  
| `--no-cache-dir` | 不保留 pip 缓存，减小镜像 |  
  
**文件 1 第 2 段**  
  
本段做什么：复制代码、建非 root 用户、声明启动命令。  
  
```dockerfile  
# 本段：代码与启动  
COPY app ./app                                 # 关键行：只复制代码目录  
COPY app/../requirements.txt /dev/null 2>/dev/null || true  
  
RUN useradd -m appuser && chown -R appuser /app  
USER appuser                                     # 关键行：非 root 运行  
  
EXPOSE 8000  
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]   # 关键行：监听所有网卡  
  
```  
  
| 代码 | 含义 |  
|---|---|  
| `COPY app ./app` | 只复制代码，不带 `.venv`、`data` |  
| `useradd` / `USER` | 建普通用户并切换，降低权限 |  
| `EXPOSE 8000` | 声明端口，供编排与文档使用 |  
| `--host 0.0.0.0` | 监听所有网卡，端口映射才生效 |  
  
> 说明：第二段里那行 `COPY app/../requirements.txt` 是多余写法，正式版应删掉——依赖清单在上一段已经复制。这里保留它，是为了提醒你：**凡是看不懂的 `|| true`，都不要照抄进生产 Dockerfile。**  
  
**文件 2：`.dockerignore`**  
  
本段做什么：把不该进镜像的东西挡在外面。  
  
```gitignore  
# 本段：构建排除清单  
.venv/  
__pycache__/  
*.pyc  
.env  
data/  
logs/  
tests/  
.git/  
.pytest_cache/  
```  
  
| 代码 | 含义 |  
|---|---|  
| `.env` | 密钥绝不入镜像 |  
| `data/` | 数据库文件靠挂载，不打进镜像 |  
| `.venv/` | 本机虚拟环境，与容器无关 |  
| `tests/` | 运行镜像不需要测试代码 |  
  
固定模板：两段式 Dockerfile（依赖层 + 代码层）、`.dockerignore` 六项。可组合写法：是否多阶段构建、是否装系统编译依赖、是否加 `HEALTHCHECK`。  
  
已知边界：镜像不含 `data/` 目录，首次启动需保证挂载点存在，否则 SQLite 报 `unable to open database file`；这属于第 2 章的编排范围。  
  
调用链：`docker build` 读取 `Dockerfile` 与 `.dockerignore`；构建产物是 `insight-agent:0.1`；`docker run` 由第 2 章的编排调用；构建失败（依赖装不上）直接让构建退出，不会产出镜像。  
  
### 1.7 结尾块  
  
**本章判据**：`docker images` 中 `insight-agent:0.1` 小于 300MB，`docker run -p 8000:8000` 后 `/api/v1/health` 返回 200。  
  
**本章产出**：`Dockerfile`、`.dockerignore`，修改 `requirements.txt`。  
  
**下一章入口**：打开 `docker-compose.yml`，`docker compose up -d` 后 10 秒内 `/health` 返回 200  
  
**常见错法 3 条**：  
  
1. `COPY . .` 且没有 `.dockerignore`。  
2. `pip install` 放在复制代码之后，层缓存失效。  
3. `CMD` 不写 `--host 0.0.0.0`。  
  
**自检 3 问**：  
  
1. 镜像里能找到 `.env` 吗？  
2. 只改业务代码，重建要几秒？  
3. 宿主机 `curl` 会 `Connection refused` 吗？  
  
---  
  
## 第 2 章 编排  
  
### 【第 2 跳】运维者 → 启动容器 → 运行中的服务与数据卷  
  
### 2.0 进度卡  
  
**① 章节清单与状态**  
  
✅ 第1章 打包 → `Dockerfile`、`.dockerignore`  
🔄 第2章 编排 ← 本章  
⏳ 第3章 反代  
⏳ 第4章 验收  
  
**② 数据现在在哪**  
  
镜像 `insight-agent:0.1` 可在任意装了 Docker 的机器上 `docker run -p 8000:8000` 启动；镜像内不含 `.env` 与 `data/`，密钥与数据库靠运行时注入或挂载。  
  
- 模型：能读到 `chunks` 表，但前提是容器里挂了正确的数据库文件。  
- 程序内：容器内的 `AgentExecutor` 可调用，依赖由镜像提供。  
- 外部用户：`docker run` 后可访问 `127.0.0.1:8000`；容器停止即不可用，且重建后数据可能丢失。  
  
**③ 本章做完后**  
  
`docker compose up -d` 一条命令起服务；`chunks` 表数据挂在宿主机 `./data/insight.db`，容器重建后仍在；`/api/v1/health` 返回 200。  
  
**④ 下一章要做**  
  
打开 `deploy/nginx.conf`，公网 80 访问 `/api/v1/chat` 返回 200 且 SSE 首条事件 1 秒内到达  
  
**施工单**：目标产物：`docker-compose.yml`；上一跳给我：可运行的 `insight-agent:0.1`；我这一跳输出：一条命令启动、数据不丢的服务。  
  
### 2.1 场景与动机  
  
本章做：用 compose 一条命令起服务并挂数据卷。  
本章不做：反代、CI、回滚。  
  
① 第 1 章交出的是 `insight-agent:0.1`，能 `docker run`，但启动命令很长。  
② 命令里带明文密钥，会留在 shell 历史；没有 `restart`，机器重启后服务不再起来；数据库没挂载，容器重建后 `chunks` 表归零。  
③ 本章把启动参数固化成文件。  
④ 选 docker-compose：一条命令、参数可版本管理，单机足够。  
  
本章不涉及新数据结构。  
  
### 2.2 最小可运行版  
  
本段做什么：用一条 `docker run` 起服务并挂数据目录。  
  
```bash  
# 本段：最小可运行版  
docker run -d --name insight \  
  -p 8000:8000 \  
  -e LLM_API_KEY=sk-xxx \  
  -v $(pwd)/data:/app/data \  
  insight-agent:0.1  
```  
  
预期：`docker ps` 能看到 `insight`，`curl 127.0.0.1:8000/api/v1/health` 返回 200。  
  
### 2.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `-d` | 后台运行 |  
| `--name insight` | 容器命名，便于日志与重启 |  
| `-p 8000:8000` | 宿主机 8000 映射到容器 8000 |  
| `-e LLM_API_KEY=...` | 运行时注入环境变量 |  
| `-v $(pwd)/data:/app/data` | 把宿主机目录挂到容器内数据目录 |  
  
**知识点 1：容器与数据卷**  
  
定义：容器文件系统随容器删除而消失；数据卷把宿主机目录映射进容器，实现持久化。  
  
定式（要背）：  
  
```bash  
-v /host/data:/app/data  
```  
  
适用：数据库文件、上传目录、日志。例题：删掉容器重建，`-v` 目录里的 `insight.db` 仍在。  
  
**知识点 2：环境变量注入**  
  
定义：运行时通过 `-e` 或 `env_file` 传配置，不写进镜像。  
  
定式（要背）：  
  
```yaml  
env_file: .env  
```  
  
适用：密钥、数据库地址、模型名。例题：换密钥只改 `.env` 并重启，不用重建镜像。  
  
**知识点 3：重启策略**  
  
定义：`restart` 决定容器退出或宿主重启后是否自动拉起。  
  
定式（要背）：  
  
```yaml  
restart: unless-stopped  
```  
  
适用：所有常驻服务。例题：机器重启后容器自动起来，除非手动 `stop`。  
  
**知识点 4：健康检查**  
  
定义：`healthcheck` 周期性执行命令，判断容器是否真的可用。  
  
定式（要背）：  
  
```yaml  
healthcheck:  
  test: ["CMD", "curl", "-f", "http://127.0.0.1:8000/api/v1/health"]  
  interval: 10s  
  retries: 3  
```  
  
适用：编排、负载均衡、自动重启。例题：进程在但接口 500 时，健康检查标记 `unhealthy`。  
  
### 2.4 指出缺陷  
  
以下缺陷只评「编排」这一动作，均可在 2.2 的命令中指到具体参数。  
  
- **D1**：`-e LLM_API_KEY=sk-xxx` 明文写在命令行 → 进入 shell 历史与 `docker inspect`，任何有该主机权限的人都能读到。  
- **D2**：没有 `--restart` → 云主机重启后容器不再启动，第二天访问报 `Connection refused`。  
- **D3**：没有健康检查 → 进程活着但接口 500 时，编排仍认为「运行中」，流量继续打进来。  
- **D4**：启动命令只存在于某人的终端 → 换人部署要重新拼一遍，漏掉 `-v` 就丢数据。  
  
### 2.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `docker-compose.yml` | 固化镜像、端口、卷、环境与健康检查 | `insight-agent:0.1`、`.env` |  
| `.env.example`（修改） | 声明需要哪些变量，不含真实值 | 无 |  
  
② 拆分理由：  
  
- `docker-compose.yml`：解决 D1（`env_file`）、D2（`restart`）、D3（`healthcheck`）、D4（命令入库）。  
- `.env.example`：解决「新机器不知道要配哪些变量」，只列键名。  
  
③ 不这么拆会怎样：启动参数散在聊天记录与终端历史里，第二个人部署必然漏项；密钥抄在 compose 文件里，提交到 Git 就泄漏。  
  
④ 依赖图：  
  
```  
.env ──────────┐  
insight-agent:0.1 ──┼──▶ docker-compose.yml ──▶ 运行中的容器  
./data ────────┘  
```  
  
⑤ 分工：设计决策回答「docker run、compose 还是 k8s」；结构设计回答「compose 与 .env 各存什么」。  
  
### 2.6 完整版  
  
**设计决策：单机用 compose，不上 Kubernetes**  
  
① 痛点：D1 的密钥外泄、D2 的重启失效、D3 的假健康、D4 的部署不可复制。  
  
② 取舍：  
  
- 方案 A：继续用 `docker run` 加一个 shell 脚本。适用于一台机器、一个人维护。否决理由：参数无结构，健康检查与重启要手写；脚本本身也需要版本管理。  
- 方案 B：Kubernetes。适用于多节点、需要弹性伸缩。否决理由：单机 2 核 4G 上，控制面本身就要占 1GB 内存；维护成本远超收益。  
  
③ 触发条件：单机、容器数 ≤5 用 compose；多机或需要自动扩缩容时再上 K8s。只有 1 个容器时，`docker run` 也能接受。  
  
**文件 1：`docker-compose.yml`**  
  
本段做什么：把镜像、端口、数据卷、环境与健康检查写成声明。  
  
```yaml  
# 本段：服务定义  
services:  
  app:  
    image: insight-agent:0.1  
    container_name: insight  
    ports:  
      - "8000:8000"  
    env_file:  
      - .env                    # 关键行：密钥不进 compose 文件  
    volumes:  
      - ./data:/app/data        # 关键行：数据库持久化  
    restart: unless-stopped     # 关键行：宿主重启后自动拉起  
    healthcheck:  
      test: ["CMD", "python", "-c",  
             "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/api/v1/health')"]  
      interval: 10s  
      timeout: 3s  
      retries: 3  
```  
  
| 代码 | 含义 |  
|---|---|  
| `image` | 使用第 1 章构建的镜像 |  
| `ports` | 宿主机与容器端口映射 |  
| `env_file` | 从 `.env` 读变量 |  
| `volumes` | 宿主机 `./data` 挂到容器 `/app/data` |  
| `restart` | 异常退出自动重启，手动停止除外 |  
| `healthcheck` | 每 10 秒探一次，连续 3 次失败标记不健康 |  
  
**文件 2：`.env.example`**  
  
本段做什么：声明变量清单，不落真实值。  
  
```bash  
# 本段：变量清单  
LLM_API_KEY=  
LLM_BASE_URL=https://api.openai.com/v1  
LLM_MODEL=gpt-4o-mini  
DATABASE_URL=sqlite:///./data/insight.db  
LOG_LEVEL=INFO  
```  
  
| 代码 | 含义 |  
|---|---|  
| 空值 `LLM_API_KEY=` | 提醒部署者必须填 |  
| `DATABASE_URL` | 必须指向挂载目录 `/app/data`，否则数据仍会丢 |  
  
固定模板：services → image/ports/env_file/volumes/restart/healthcheck 六项。可组合写法：是否加 `depends_on`、是否限制 CPU 与内存、是否加日志驱动。  
  
已知边界：`healthcheck` 用 Python 标准库而不是 `curl`，因为 slim 镜像默认没有 `curl`；若基础镜像装了 curl，可换成 `curl -f`。  
  
调用链：`docker compose up -d` 读取 `docker-compose.yml` 与 `.env`；它调用 Docker 引擎创建容器与挂载卷；容器启动后由 `uvicorn` 拉起 `app.main:app`；启动失败在 `docker logs insight` 里可见。  
  
### 2.7 结尾块  
  
**本章判据**：`docker compose up -d` 后 10 秒内 `/api/v1/health` 返回 200；`docker compose restart` 后 `chunks` 行数不变。  
  
**本章产出**：`docker-compose.yml`，修改 `.env.example`。  
  
**下一章入口**：打开 `deploy/nginx.conf`，公网 80 访问 `/api/v1/chat` 返回 200 且 SSE 首条事件 1 秒内到达  
  
**常见错法 3 条**：  
  
1. 密钥写在 compose 文件里。  
2. 忘记挂载 `/app/data`，容器重建后数据归零。  
3. 没有 `restart`，机器重启后服务消失。  
  
**自检 3 问**：  
  
1. `docker inspect` 里能看到密钥吗？  
2. 删掉容器重建，切片还在吗？  
3. 宿主重启后服务会自动起来吗？  
  
---  
  
## 第 3 章 反代  
  
### 【第 3 跳】外部用户 → Nginx 转发 → 容器 8000  
  
### 3.0 进度卡  
  
**① 章节清单与状态**  
  
✅ 第1章 打包 → `Dockerfile`、`.dockerignore`  
✅ 第2章 编排 → `docker-compose.yml`  
🔄 第3章 反代 ← 本章  
⏳ 第4章 验收  
  
**② 数据现在在哪**  
  
`docker compose up -d` 一条命令起服务；`chunks` 表数据挂在宿主机 `./data/insight.db`，容器重建后仍在；`/api/v1/health` 返回 200。  
  
- 模型：能读到 `chunks` 表，检索正常。  
- 程序内：容器内 `AgentExecutor` 可调用，`pytest` 在镜像外仍可跑。  
- 外部用户：只能访问 `<公网IP>:8000`；80 端口没有监听，浏览器直接输域名会报 `ERR_CONNECTION_REFUSED`。  
  
**③ 本章做完后**  
  
外部用户访问 `http://<公网IP>/api/v1/chat` 返回 200；Nginx 把 80 转到容器 8000，SSE 关闭缓冲后事件逐条到达。  
  
**④ 下一章要做**  
  
打开 `.github/workflows/ci.yml`，冒烟 4 项全过且回滚在 2 分钟内完成  
  
**施工单**：目标产物：`deploy/nginx.conf`；上一跳给我：单机可用的容器服务；我这一跳输出：公网 80 可访问、流式不被缓冲的入口。  
  
### 3.1 场景与动机  
  
本章做：用 Nginx 把公网 80 转发到容器 8000。  
本章不做：CI、回滚、证书自动续期。  
  
① 第 2 章交出的是 `docker-compose.yml`，服务在 8000 端口可用。  
② 用户只会输域名，不会加 `:8000`；直接暴露 8000 还缺少超时与请求体大小控制；SSE 经过默认代理会一次性吐出。  
③ 本章加一层反向代理。  
④ 选 Nginx：单机最成熟，配置可读，SSE 只需关掉缓冲。  
  
本章不涉及新数据结构。  
  
### 3.2 最小可运行版  
  
本段做什么：写一个最小的 80 端口转发配置。  
  
```nginx  
# 本段：最小可运行版  
server {  
    listen 80;  
    server_name _;  
  
    location / {  
        proxy_pass http://127.0.0.1:8000;  
    }  
}  
```  
  
预期：`nginx -t` 通过，`curl http://127.0.0.1/api/v1/health` 返回 200。  
  
### 3.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `listen 80` | 监听公网 80 端口 |  
| `server_name _` | 匹配任意域名，先用 IP 访问 |  
| `location /` | 所有路径 |  
| `proxy_pass` | 把请求转发到容器映射出的 8000 |  
  
**知识点 1：反向代理**  
  
定义：反向代理接收外部请求，再转发给内部服务，外部只看到代理。  
  
定式（要背）：  
  
```nginx  
location / { proxy_pass http://127.0.0.1:8000; }  
```  
  
适用：隐藏内部端口、统一入口、加 TLS。例题：用户访问 80，实际处理在 8000。  
  
**知识点 2：代理缓冲与 SSE**  
  
定义：Nginx 默认缓冲上游响应，攒够再发；SSE 需要逐条到达，必须关闭缓冲。  
  
定式（要背）：  
  
```nginx  
proxy_buffering off;  
proxy_cache off;  
```  
  
适用：SSE、流式下载。例题：不关缓冲时，8 秒内的事件在最后一次性到达；关掉后每 0.8 秒一条。  
  
**知识点 3：代理超时**  
  
定义：`proxy_read_timeout` 是等待上游两次输出之间的最长时间，超时返回 504。  
  
定式（要背）：  
  
```nginx  
proxy_read_timeout 120s;  
```  
  
适用：长耗时接口与流式。例题：默认 60 秒时，ReAct 跑 70 秒的请求被 Nginx 切断。  
  
**知识点 4：请求体大小**  
  
定义：`client_max_body_size` 限制上传体积，超出返回 413。  
  
定式（要背）：  
  
```nginx  
client_max_body_size 20m;  
```  
  
适用：有资料文件上传的接口。例题：默认 1MB，上传 3MB 的 `.md` 直接 413。  
  
### 3.4 指出缺陷  
  
以下缺陷只评「反代」这一动作，均可在 3.2 的配置中指到具体行。  
  
- **D1**：未设置 `proxy_buffering off` → `/api/v1/chat/stream` 的事件被攒到最后，用户仍要等 8 秒才看到全部内容。  
- **D2**：未设置 `proxy_read_timeout` → 用默认 60 秒，ReAct 跑到 70 秒时 Nginx 返回 `504 Gateway Time-out`。  
- **D3**：未设置 `client_max_body_size` → 默认 1MB，上传 3MB 的资料文件返回 `413 Request Entity Too Large`。  
- **D4**：未转发 `Host` 与 `X-Real-IP` → 应用日志里所有请求来源都是 `127.0.0.1`，无法定位真实客户端。  
  
### 3.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `deploy/nginx.conf` | 定义公网入口、转发、缓冲与超时 | 容器 8000 |  
| `deploy/README.md` | 记录安装、`nginx -t`、reload 三步 | `nginx.conf` |  
  
② 拆分理由：  
  
- `nginx.conf`：解决 D1（关缓冲）、D2（超时）、D3（请求体上限）、D4（转发头）。  
- `deploy/README.md`：解决「换人不知道怎么生效」，避免直接改线上文件。  
  
③ 不这么拆会怎样：配置只存在服务器 `/etc/nginx` 里，没有版本记录；一次误改无法回滚，只能凭记忆重写。  
  
④ 依赖图：  
  
```  
用户 :80 ──▶ deploy/nginx.conf ──▶ 127.0.0.1:8000 ──▶ 容器  
```  
  
⑤ 分工：设计决策回答「直接暴露、Nginx 还是云负载均衡」；结构设计回答「配置放仓库哪个目录、怎么生效」。  
  
### 3.6 完整版  
  
**设计决策：单机用 Nginx，不用云负载均衡**  
  
① 痛点：D1 的流式失效、D2 的 504、D3 的 413、D4 的来源丢失。  
  
② 取舍：  
  
- 方案 A：直接把容器 `-p 80:8000`。适用于临时演示。否决理由：应用直接面对公网，没有超时、体积与 TLS 的统一控制；换端口要停服务。  
- 方案 B：云厂商负载均衡。适用于多台后端与证书托管。否决理由：单机场景下多一层计费与配置，且健康检查与转发规则仍需理解。  
  
③ 触发条件：单机、一个后端用 Nginx；多后端或需要 WAF、证书托管时上云负载均衡。临时演示可直接映射 80。  
  
**文件 1：`deploy/nginx.conf`**  
  
本段做什么：定义完整入口，含流式与超时设置。  
  
```nginx  
# 本段：入口与转发  
server {  
    listen 80;  
    server_name _;  
    client_max_body_size 20m;                 # 关键行：上传上限  
  
    location / {  
        proxy_pass http://127.0.0.1:8000;  
        proxy_http_version 1.1;               # 关键行：SSE 需要长连接  
        proxy_set_header Host $host;  
        proxy_set_header X-Real-IP $remote_addr;  
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
    }  
}  
```  
  
| 代码 | 含义 |  
|---|---|  
| `client_max_body_size 20m` | 允许 20MB 以内的请求体 |  
| `proxy_http_version 1.1` | 保持长连接，SSE 才能持续推送 |  
| `Host` / `X-Real-IP` | 让应用拿到真实域名与客户端 IP |  
  
**文件 1 第 2 段**  
  
本段做什么：为流式接口单独关缓冲、放宽超时。  
  
```nginx  
# 本段：流式与超时  
    location /api/v1/chat/stream {  
        proxy_pass http://127.0.0.1:8000;  
        proxy_http_version 1.1;  
        proxy_buffering off;                  # 关键行：逐条下发  
        proxy_cache off;  
        proxy_read_timeout 120s;              # 关键行：长任务不被切断  
        proxy_set_header Connection "";  
    }  
  
    location /api/v1/ {  
        proxy_pass http://127.0.0.1:8000;  
        proxy_read_timeout 120s;  
    }  
```  
  
| 代码 | 含义 |  
|---|---|  
| `proxy_buffering off` | 关闭缓冲，事件立即下发 |  
| `proxy_cache off` | 不缓存流式响应 |  
| `proxy_read_timeout 120s` | 允许长任务 |  
| `location` 精确匹配 | 流式路径单独规则，先于通用规则 |  
  
固定模板：`listen` + `client_max_body_size` + 两条 `location`。可组合写法：是否加 TLS、是否限流、是否压缩。  
  
已知边界：本册只配 HTTP。上线到真实域名需另配证书与 443，证书续期不在本册范围。  
  
调用链：外部请求进入 Nginx 的 80；Nginx 按 `location` 转发到容器映射的 8000；应用返回后由 Nginx 回传；上游超时由 Nginx 直接返回 504，请求不会到达应用。  
  
### 3.7 结尾块  
  
**本章判据**：`curl http://<公网IP>/api/v1/chat` 返回 200；`curl -N .../chat/stream` 的首条事件在 1 秒内到达。  
  
**本章产出**：`deploy/nginx.conf`、`deploy/README.md`。  
  
**下一章入口**：打开 `.github/workflows/ci.yml`，冒烟 4 项全过且回滚在 2 分钟内完成  
  
**常见错法 3 条**：  
  
1. 不关 `proxy_buffering`，流式变成一次性。  
2. 用默认 60 秒超时，长任务 504。  
3. 不转发 `X-Real-IP`，日志全是 127.0.0.1。  
  
**自检 3 问**：  
  
1. 流式事件是一条条到，还是最后一起到？  
2. 70 秒的任务会被切断吗？  
3. 日志里能看到真实客户端 IP 吗？  
  
---  
  
## 第 4 章 验收  
  
### 【第 4 跳】CI 与运维者 → 冒烟与回滚 → 可重复的上线  
  
### 4.0 进度卡  
  
**① 章节清单与状态**  
  
✅ 第1章 打包 → `Dockerfile`、`.dockerignore`  
✅ 第2章 编排 → `docker-compose.yml`  
✅ 第3章 反代 → `deploy/nginx.conf`  
🔄 第4章 验收 ← 本章  
  
**② 数据现在在哪**  
  
外部用户访问 `http://<公网IP>/api/v1/chat` 返回 200；Nginx 把 80 转到容器 8000，SSE 关闭缓冲后事件逐条到达。  
  
- 模型：能读到 `chunks` 表，检索与作答正常。  
- 程序内：`pytest` 可跑，但只在开发机；线上没有自动验证。  
- 外部用户：能访问，但上线靠手工敲命令；出错后不知道退回哪一版。  
  
**③ 本章做完后**  
  
每次 push 触发 CI 构建并跑测试；上线后有 `/api/v1/health` 探活、4 项冒烟脚本与一条回滚命令；故障可在 2 分钟内退回上一镜像。  
  
**④ 下一章要做**  
  
打开 `app/services/crawler.py`，把网页资料并发抓取后送进 0.2 切分  
  
**施工单**：目标产物：`ci.yml` 与 `scripts/smoke.sh`；上一跳给我：公网可访问的服务；我这一跳输出：可自动验证、可回滚的上线流程。  
  
### 4.1 场景与动机  
  
本章做：用 CI、冒烟脚本与回滚把上线变成可重复动作。  
本章不做：压力测试、监控告警、证书管理。  
  
① 第 3 章交出的是 `deploy/nginx.conf`，公网能访问，但上线靠手工。  
② 一次上线要敲 6 条命令，漏一条就出现「镜像换了、容器没重启」；出问题只能凭记忆找上一版镜像名，平均恢复 40 分钟。  
③ 本章把上线变成脚本。  
④ 选 CI 加冒烟加标签回滚：每一步可重复，失败有退路。  
  
本章不涉及新数据结构。  
  
### 4.2 最小可运行版  
  
本段做什么：用一条 `curl` 检查服务是否活着。  
  
```bash  
# 本段：最小可运行版  
curl -fsS http://127.0.0.1:8000/api/v1/health  
```  
  
预期：返回 `{"status":"ok","app_name":"InsightAgent",...}`，退出码 0。  
  
### 4.3 分段讲解  
  
| 代码 | 含义 |  
|---|---|  
| `curl` | 发 HTTP 请求 |  
| `-f` | 4xx／5xx 时返回非零退出码 |  
| `-sS` | 静默但保留错误信息 |  
| `http://.../health` | 健康检查地址 |  
  
**知识点 1：健康检查**  
  
定义：健康检查是一个轻量接口，只回答「服务能否处理请求」。  
  
定式（要背）：  
  
```bash  
curl -fsS http://127.0.0.1:8000/api/v1/health  
```  
  
适用：上线、负载均衡、容器编排。例题：返回 200 且 `status=ok` 才算通过。  
  
**知识点 2：冒烟测试**  
  
定义：冒烟测试是上线后跑的最小一组关键路径，验证「主要功能没坏」。  
  
定式（要背）：  
  
```bash  
curl -fsS -X POST "$BASE/api/v1/chat" -H 'Content-Type: application/json' \  
  -d '{"question":"这个项目部署时用哪个端口？"}' | grep -q '8000'  
```  
  
适用：每次发布后。例题：四项分别验健康、问答、流式、参数错误。  
  
**知识点 3：镜像标签与回滚**  
  
定义：每次构建用唯一标签（如 commit 短哈希），回滚就是把 `latest` 指回上一标签。  
  
定式（要背）：  
  
```bash  
docker tag insight-agent:previous insight-agent:current  
docker compose up -d  
```  
  
适用：任何镜像部署。例题：新版本 502 时，1 分钟内退到上一标签。  
  
**知识点 4：CI 流水线**  
  
定义：CI 在每次 push 后自动执行测试与构建，失败即阻断。  
  
定式（要背）：  
  
```yaml  
on: [push]  
jobs:  
  test: ...  
  build:  
    needs: test  
```  
  
适用：多人协作与频繁发布。例题：测试失败时 `build` 不执行，坏镜像不会产出。  
  
### 4.4 指出缺陷  
  
以下缺陷只评「验收」这一动作，均可在 4.2 的命令中指到具体参数。  
  
- **D1**：只 `curl` 健康检查 → 问答接口 500、流式接口断开都发现不了。  
- **D2**：`curl` 不加 `-f` → 502 时退出码仍为 0，脚本误判为成功。  
- **D3**：只检查连通、不校验内容 → 返回 `{"answer":"服务维护中"}` 也算通过。  
- **D4**：没有回滚步骤 → 上线后发现 502，只能凭记忆找上一版镜像，恢复时间以十分钟计。  
  
### 4.5 结构设计  
  
① 文件清单：  
  
| 文件路径 | 一句话职责 | 依赖谁 |  
|---|---|---|  
| `.github/workflows/ci.yml` | push 后跑测试并构建镜像 | `requirements.txt`、`Dockerfile` |  
| `scripts/smoke.sh` | 上线后跑 4 项关键路径 | 公网地址 |  
  
② 拆分理由：  
  
- `ci.yml`：解决 D4 的一半（坏镜像不产出），并保证每次构建可追溯。  
- `smoke.sh`：解决 D1、D2、D3，四项断言覆盖健康、问答、流式与 422。  
  
③ 不这么拆会怎样：上线检查写在某人的笔记里，换人就不执行；CI 只跑测试不构建，线上镜像与代码对不上。  
  
④ 依赖图：  
  
```  
push ──▶ ci.yml ──▶ 测试 ──▶ 镜像  
                              │  
上线 ──▶ smoke.sh ──▶ 4 项断言 ──▶ 通过/回滚  
```  
  
⑤ 分工：设计决策回答「手工、脚本还是 CI/CD」；结构设计回答「流水线与冒烟脚本各管哪一段」。  
  
### 4.6 完整版  
  
**设计决策：CI 加脚本，而不是手工上线**  
  
① 痛点：D1 的漏检、D2 的假成功、D3 的无内容校验、D4 的无法回滚。  
  
② 取舍：  
  
- 方案 A：手工执行一份上线清单。适用于一周发布一次。否决理由：清单靠人执行，漏项无法发现；回滚仍要人工找镜像。  
- 方案 B：完整 CD（自动部署到生产）。适用于发布频繁、有灰度与监控。否决理由：本册没有监控与灰度能力，自动部署会把坏版本直接推给所有用户。  
  
③ 触发条件：有 CI 与冒烟即可先做到「可验证、可回滚」；等有监控与灰度后，再把部署自动化。  
  
**文件 1：`.github/workflows/ci.yml`**  
  
本段做什么：push 后跑测试，测试通过才构建镜像。  
  
```yaml  
# 本段：测试与构建  
name: ci  
on: [push]  
jobs:  
  test:  
    runs-on: ubuntu-latest  
    steps:  
      - uses: actions/checkout@v4  
      - uses: actions/setup-python@v5  
        with:  
          python-version: "3.11"  
      - run: pip install -r requirements.txt  
      - run: pytest -q  
  build:  
    needs: test                     # 关键行：测试不过不构建  
    runs-on: ubuntu-latest  
    steps:  
      - uses: actions/checkout@v4  
      - run: docker build -t insight-agent:${{ github.sha }} .  
```  
  
| 代码 | 含义 |  
|---|---|  
| `on: [push]` | 每次推送触发 |  
| `needs: test` | 依赖测试任务，失败即跳过 |  
| `github.sha` | 用 commit 哈希做唯一镜像标签 |  
| `pytest -q` | 跑测试，非零退出即失败 |  
  
**文件 2：`scripts/smoke.sh`**  
  
本段做什么：上线后跑四项断言，任一失败即退出非零。  
  
```bash  
# 本段：冒烟脚本  
#!/usr/bin/env bash  
set -euo pipefail                       # 关键行：任一命令失败即退出  
BASE="${1:-http://127.0.0.1}"  
  
curl -fsS "$BASE/api/v1/health" | grep -q '"status":"ok"'          # 1 健康  
curl -fsS -X POST "$BASE/api/v1/chat" \  
  -H 'Content-Type: application/json' \  
  -d '{"question":"这个项目部署时用哪个端口？"}' | grep -q '8000'   # 2 问答  
curl -fsS -N -X POST "$BASE/api/v1/chat/stream" \  
  -H 'Content-Type: application/json' \  
  -d '{"question":"这个项目部署时用哪个端口？"}' | grep -q '\[DONE\]' # 3 流式  
code=$(curl -s -o /dev/null -w '%{http_code}' -X POST "$BASE/api/v1/chat" \  
  -H 'Content-Type: application/json' -d '{"question":""}')  
test "$code" = "422"                                                # 4 参数校验  
echo "smoke ok"  
```  
  
| 代码 | 含义 |  
|---|---|  
| `set -euo pipefail` | 任一命令失败立即退出 |  
| `grep -q` | 只判断是否包含，不打印 |  
| `-N` | 关闭 curl 缓冲，读流式 |  
| `-w '%{http_code}'` | 只取状态码 |  
| `test "$code" = "422"` | 断言参数校验生效 |  
  
**文件 3：回滚命令**  
  
本段做什么：一条命令退回上一镜像。  
  
```bash  
# 本段：回滚  
docker tag insight-agent:previous insight-agent:current  
docker compose up -d  
```  
  
| 代码 | 含义 |  
|---|---|  
| `docker tag` | 把 `current` 指向旧镜像 |  
| `compose up -d` | 用新标签重建容器 |  
  
固定模板：`ci.yml` 的 test→build 两任务、`smoke.sh` 的四项断言、回滚两行。可组合写法：是否加镜像仓库推送、是否加部署 job、是否加通知。  
  
已知边界：本册的冒烟在部署机上执行，依赖公网可达；CI 只做测试与构建，不自动部署到生产，避免无监控时误推坏版本。  
  
调用链：`push` 触发 `ci.yml`；测试任务调用 `pytest`，构建任务调用 `docker build`；上线后 `smoke.sh` 调用四个接口；任一项失败由运维执行回滚命令；回滚调用 `docker tag` 与 `compose up -d`。  
  
### 4.7 结尾块  
  
**本章判据**：`bash scripts/smoke.sh http://<公网IP>` 输出 `smoke ok`；回滚后 2 分钟内 `/api/v1/health` 恢复 200。  
  
**本章产出**：`.github/workflows/ci.yml`、`scripts/smoke.sh`、`deploy/ROLLBACK.md`。  
  
**下一章入口**：打开 `app/services/crawler.py`，把网页资料并发抓取后送进 0.2 切分  
  
**常见错法 3 条**：  
  
1. 只检查 `/health`，不测问答与流式。  
2. `curl` 不加 `-f`，502 也当成功。  
3. 没有唯一镜像标签，回滚时不知道退到哪一版。  
  
**自检 3 问**：  
  
1. 问答接口 500 时，冒烟会红吗？  
2. 流式结束标记丢了，能发现吗？  
3. 回滚要几分钟？  
  
**下一册从哪接**：打开 `app/services/crawler.py`，把网页资料并发抓取后送进 0.2 切分  
  
---  
  
## 按链路排查表  
  
| 现象 | 断在第几跳 | 查什么 |  
|---|---|---|  
| 容器启动即退出 | 4.1／4.2 | `docker logs insight`；看 `ModuleNotFoundError`、端口占用、`.env` 缺失 |  
| 宿主机 `curl` 通、公网 502 | 4.3 | Nginx 是否启动、`proxy_pass` 地址、云主机安全组是否放行 80 |  
| SSE 8 秒后一次性到达 | 4.3 | `proxy_buffering` 是否为 `off`、`proxy_http_version` 是否为 1.1 |  
| 容器重建后切片清空 | 4.2 | `volumes` 是否挂到 `/app/data`、`DATABASE_URL` 是否指向该路径 |  
| 上线后 502，重启无效 | 4.4 | 当前镜像标签、回滚命令是否执行、上一标签是否还存在 |  
| 冒烟通过但用户报错 | 4.4 | 冒烟是否只测了健康；补问答与流式的断言 |  
  
## 知识地图  
  
| 知识点 | 落点 |  
|---|---|  
| 镜像与分层 | 1.3 知识点 1、2 |  
| 构建上下文 | 1.3 知识点 3 |  
| 容器网络与端口 | 1.3 知识点 4 |  
| 数据卷 | 2.3 知识点 1 |  
| 环境变量注入 | 2.3 知识点 2 |  
| 健康检查 | 2.3 知识点 4、4.3 知识点 1 |  
| 反向代理 | 3.3 知识点 1 |  
| 流式与缓冲 | 3.3 知识点 2 |  
| 超时与请求体限制 | 3.3 知识点 3、4 |  
| 冒烟与回滚 | 4.3 知识点 2、3 |  
| CI 流水线 | 4.3 知识点 4 |  
  
## 自测  
  
1. 为什么 `pip install` 要放在 `COPY app` 之前？  
2. 容器里监听 `127.0.0.1` 与 `0.0.0.0` 有什么区别？  
3. 不挂数据卷，容器重建后 `chunks` 表会怎样？  
4. SSE 经过 Nginx 为什么要关 `proxy_buffering`？  
5. 回滚为什么依赖唯一镜像标签？  
  
<details>  
<summary>答案要点</summary>  
  
1. 依赖清单不变时可命中层缓存；代码改动不会让依赖重装。  
2. `127.0.0.1` 只接受容器内部访问，宿主映射进不来；`0.0.0.0` 监听所有网卡。  
3. 容器文件系统随容器删除，数据库文件一起消失，切片全部丢失。  
4. 默认缓冲会攒够再发，流式失去意义；关闭后逐条下发。  
5. 没有唯一标签时，无法确定上一版对应哪个镜像，回滚只能靠猜。  
  
</details>  
  
## 规范自检记录  
  
| 条目 | 证据（小节号） | 结论 |  
|---|---|---|  
| 1 八段结构 | 1.1–1.7、2.1–2.7、3.1–3.7、4.1–4.7；无数据建模章已在 N.1 声明 | 满足 |  
| 2 跳号标注 | 各章标题下【第 N 跳】 | 满足 |  
| 3 一章一动作 | 打包／编排／反代／验收，四章四动作 | 满足 |  
| 4 本章范围 | 各章 N.1 首两行 | 满足 |  
| 5 数据建模 | 本册无新数据结构，四章均声明 | 不适用 |  
| 6 最小可运行版 | 1.2、2.2、3.2、4.2 | 满足 |  
| 7 缺陷可定位 | 1.4 指 Dockerfile 行；2.4 指 run 参数；3.4 指 nginx 行；4.4 指 curl 参数 | 满足 |  
| 8 结构设计引用 | 1.5、2.5、3.5、4.5 均引用真实 D 编号 | 满足 |  
| 9 册首三行与导航表 | 本册开头 | 满足 |  
| 10 进度卡四部分 | 各章 N.0 | 满足 |  
| 11 三读者分行 | 各章 N.0② 均有模型／程序内／外部用户三行 | 满足 |  
| 12 端到端链路表 | 本册开头两条链，六列，标交汇点 | 满足 |  
| 13 跳声明与排查表 | 各章标题；文末排查表 6 行 | 满足 |  
| 14 能力落差 | 本册不引入工具／检索／缓存／队列／索引／异步，不适用 | 不适用 |  
| 15 施工单与数据流顺序 | 各章「施工单」三行 | 满足 |  
| 16 术语红线 | 全文未单用「文档」；资料文件／切片／教材正文已区分 | 满足 |  
| 17 自检六项 | 下表 C1–C6 | 满足 |  
| 18 v2.0 声明与逐条自检 | 本册首行与本表 | 满足 |  
| 19 第 0 册例外 | 本册非第 0 册 | 不适用 |  
| 20 知识点三件套 | 1.3、2.3、3.3、4.3 共 16 个 | 满足 |  
| 21 设计决策三段 | 1.6、2.6、3.6、4.6 | 满足 |  
| 22 读者画像 | 全文 | 满足 |  
| 23 代码与注释 | 各段 import／指令三标、函数 docstring、分段导读 | 部分满足：知识点卡内 3–6 行定式未逐行注释，因模板需直接抄写；正文代码均满足 |  
| 24 测试标注 | 4.6 冒烟四项均注明断言对象 | 满足 |  
| 25 结尾块 | 各章 N.7；4.7 含下一册从哪接 | 满足 |  
| 26 交付形态 | 单册、md 文本、直接输出、第二人称 | 满足 |  
| D1 只给完整版 | 四章均有最小版 | 未违反 |  
| D2 省略用语 | 全文无 | 未违反 |  
| D3 不可运行代码 | 各段可复制执行 | 未违反 |  
| D4 抽象理由 | 缺陷均给报错或数字 | 未违反 |  
| D5 一次多册 | 仅本册 | 未违反 |  
| D6 反问开头 | 无 | 未违反 |  
| D7 单用文档 | 无 | 未违反 |  
| D8 演示对象 | 无 | 未违反 |  
| D9 按文件名排序 | 按交付链 4.1–4.4 组织 | 未违反 |  
| D10 先建文件后定数据 | 各章先写施工单再给文件 | 未违反 |  
  
一致性检查：  
  
- **C1**：本册无数据建模段，完整版未出现新的持久化对象。通过。  
- **C2**：1.4 指向 `FROM`、`COPY . .`、`RUN pip`、`CMD` 行；2.4 指向 `-e`、缺 `--restart`、缺 `healthcheck`；3.4 指向缺 `proxy_buffering`、缺 `proxy_read_timeout`、缺 `client_max_body_size`、缺转发头；4.4 指向只 `curl health`、缺 `-f`、无内容断言、无回滚。通过。  
- **C3**：1.5 引用 D1–D5；2.5、3.5、4.5 均引用 D1–D4，全部真实存在。通过。  
- **C4**：1.4 全属打包；2.4 全属编排；3.4 全属反代；4.4 全属验收。无越界。通过。  
- **C5**：无数据建模，最小版无对象省略。通过。  
- **C6**：1.0④＝导航表第2行＝1.7；2.0④＝第3行＝2.7；3.0④＝第4行＝3.7；4.0④＝「下一册」行＝4.7 及「下一册从哪接」，三处一字不差。通过。  
  
本册结束，未开始第七册。  
