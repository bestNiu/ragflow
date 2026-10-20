# RAGFlow 二开环境部署文档

> 本文档描述 `bestNiu/ragflow` 仓库 `dev-custom` 二开分支在**内网 Linux 主机**上的源码构建与 Docker Compose 部署方式，对外访问端口为 **8088**。
>
> | 项 | 值 |
> |---|---|
> | 文档版本 | v1.0 |
> | 更新日期 | 2026-10-20 |
> | 部署主机 | `192.168.0.211`（Ubuntu 24.04，8C/31G，Docker 29.1.3 + Compose 2.40.3） |
> | 部署目录 | `/home/ulyy/app/unionragFlow/ragflow` |
> | 分支 | `dev-custom`（基于 `main` @ `232b3f7`） |
> | 镜像 | `ragflow:go-local`（源码构建，非官方预编译镜像） |
> | 访问地址 | <http://192.168.0.211:8088> |

---

## 1. 架构与端口

### 1.1 运行拓扑

```
                     ┌──────────────────────────── 宿主机 192.168.0.211 ───────────────────────────┐
  浏览器 / SDK ──8088──▶ ragflow-cpu (nginx:80 → api:9380 / 静态前端)                            │
                9380──▶   ├── api_server      :9380   REST API / MCP(in-process)                  │
                9381──▶   ├── admin_server    :9381   管理 API                                   │
                          ├── ingestor               文档解析/入库（进程内）                       │
                          └── syncer                 数据源同步（进程内）                         │
                                   │                                                            │
             ┌─────────────────────┼─────────────────────┬──────────────┬───────────────┐      │
             ▼                     ▼                     ▼              ▼               ▼      │
        es01:9200            mysql:3306            minio:9000     kvrocks:6379      nats:4222   │
      (文档检索引擎)        (业务元数据库)         (对象存储)     (缓存/队列)      (消息队列)     │
             │                     │                     │              │               │      │
        ragflow_esdata01   ragflow_mysql_data   ragflow_minio_data  ragflow_kvrocks_data  ragflow_nats_data
```

所有容器同属 Compose 项目 `ragflow`、同一 bridge 网络 `ragflow_ragflow`。

### 1.2 端口清单

| 宿主端口 | 容器/服务 | 用途 | 对外暴露建议 |
|---|---|---|---|
| **8088** | `ragflow-cpu` → nginx:80 | **Web 控制台 + REST API（唯一对外入口）** | 允许内网访问 |
| 9380 | `ragflow-cpu` → api_server | 直连 API（绕过 nginx） | 建议仅本机/内网 |
| 9381 | `ragflow-cpu` → admin_server | 管理 API | 建议仅本机 |
| 1200 | `es01` | Elasticsearch | 仅内网 |
| 3306 | `mysql` | MySQL 8.0 | 仅内网 |
| 9000 / 9001 | `minio` | 对象存储 / 控制台 | 仅内网 |
| 6379 | `kvrocks` | Redis 协议缓存/队列 | 仅内网 |
| 4222 / 8222 | `nats` | 消息队列 / 监控 | 仅内网 |

> 443 未发布：镜像内 nginx 仅监听 80，没有 TLS vhost。需要 HTTPS 时请在前置反代（Nginx/Traefik）终止 TLS。
> 9382（独立 MCP 监听端口）默认未启用。

### 1.3 版本与资源

| 组件 | 版本/配置 |
|---|---|
| RAGFlow | 源码分支 `dev-custom`，Go 后端 + Vite 前端 |
| Python 依赖 | 镜像内只保留运行期依赖；DeepDoc 由 Go 进程内推理（CPU） |
| Elasticsearch | 8.11.3，`MEM_LIMIT=8G` |
| MySQL | 8.0.40 |
| MinIO（Silo 兼容版） | `pgsty/silo:RELEASE.2026-08-06T00-00-00Z` |
| Kvrocks | 2.16.0 |
| NATS | 2.14.2 |
| 镜像大小 | `ragflow:go-local` ≈ 1.15 GB |

宿主机要求：≥ 4C / 16G / 50G 可用磁盘（源码构建另需约 20–30 GB 构建缓存），`vm.max_map_count ≥ 262144`（本机已为 1048576，满足）。

---

## 2. 前置条件

```bash
# 1) Docker / Compose
docker --version            # >= 24.0
docker compose version      # >= v2.26.1
docker buildx version       # 需要 BuildKit

# 2) 内核参数（Elasticsearch 必需）
sysctl vm.max_map_count     # 期望 >= 262144
# 若不足（需 root）：
#   sudo sysctl -w vm.max_map_count=262144
#   echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf

# 3) 宿主 80/443 已被其它栈（dify）占用，本部署只用 8088，无需释放
ss -lntp | grep -E ':(8088|9380|9381|1200|3306|9000|9001|6379|4222|8222)\b'
```

> 本机容器**无法直连外网**（只有宿主上的代理 `127.0.0.1:7890` 可出网），因此构建阶段必须走代理，见第 3 节。

---

## 3. 获取代码与分支

```bash
mkdir -p ~/app/unionragFlow && cd ~/app/unionragFlow

# 私有仓库需带 PAT（示例，请替换为你自己的 token）
git clone --depth 1 \
  https://x-access-token:<YOUR_GITHUB_PAT>@github.com/bestNiu/ragflow.git ragflow

cd ragflow

# 切到二开分支（已包含本环境的构建/部署改造）
git fetch --depth 1 origin dev-custom:dev-custom
git checkout dev-custom

# 清理 git 中的 token，避免写入 .git/config
git remote set-url origin https://github.com/bestNiu/ragflow.git
git remote -v
```

> 浅克隆没有 tag，构建出的版本号是 commit hash（例如 `232b3f7`），这是 `RAGFLOW_DEV_MODE=true` 的原因之一。
> 若需完整历史：`git fetch --unshallow origin`。

本次二开分支的改动（`git log --oneline`）：

| 提交 | 内容 |
|---|---|
| `16bdb1e` | 不再发布无监听者的 443 端口 |
| `db3da23` | `build.sh` 代理修复；`.env` 8088 端口、独立项目名、随机密码、dev mode |

---

## 4. 构建镜像

### 4.1 构建命令（本机网络环境，走宿主代理）

```bash
cd ~/app/unionragFlow/ragflow

docker build --platform linux/amd64 --network host --progress=plain \
  --build-arg HTTP_PROXY=http://127.0.0.1:7890 \
  --build-arg HTTPS_PROXY=http://127.0.0.1:7890 \
  --build-arg http_proxy=http://127.0.0.1:7890 \
  --build-arg https_proxy=http://127.0.0.1:7890 \
  --build-arg NO_PROXY=localhost,127.0.0.1,::1,172.16.0.0/12,10.0.0.0/8 \
  --build-arg no_proxy=localhost,127.0.0.1,::1,172.16.0.0/12,10.0.0.0/8 \
  -f Dockerfile -t ragflow:go-local . 2>&1 | tee /tmp/ragflow-build.log
```

关键点：

| 参数 | 原因 |
|---|---|
| `--network host` | 让构建容器通过 `127.0.0.1:7890` 使用宿主代理 |
| `--build-arg HTTP(S)_PROXY` | apt / curl / npm / go mod / git 均走代理 |
| `--platform linux/amd64` | Go 后端只支持 Linux x86-64 |
| 保留 `.git` 目录 | Dockerfile 用 `git describe` 生成 `/ragflow/VERSION` |

构建耗时参考：首次约 15–25 分钟（主要花在 `infiniflow/ragflow_deps` 拉取、npm install、go mod download）；改动源码后重建约 2–5 分钟（层缓存）。

### 4.2 无代理 / 可直连外网的主机

```bash
docker build --platform linux/amd64 -f Dockerfile -t ragflow:go-local .
```

### 4.3 构建阶段的两个已知问题及处理

1. **apt 超时**（`Could not connect to archive.ubuntu.com`）
   容器直连被墙。处理：加 4.1 的代理参数 + `--network host`。

2. **`build.sh --go` 失败：libpcre2 下载失败**
   `build.sh` 原先用 `sudo apt-get install`，`sudo` 默认重置环境变量 → 代理丢失。
   已在 `dev-custom` 修复为：root 直接 `apt-get`，非 root 用 `sudo -E`。若使用旧分支，可手动打补丁：

   ```diff
   -        echo "sudo apt-get install -y libpcre2-dev"
   +        if [ "$(id -u)" -eq 0 ]; then
   +            echo "apt-get install -y libpcre2-dev"
   +        else
   +            echo "sudo -E apt-get install -y libpcre2-dev"
   +        fi
   ```

3. **构建完成后校验**

   ```bash
   docker images | grep ragflow:go-local       # 约 1.15GB
   docker run --rm --entrypoint cat ragflow:go-local /ragflow/VERSION   # 应输出 commit hash
   ```

---

## 5. 配置说明（`docker/.env`）

所有参数集中在 `docker/.env`，Compose 与容器 `env_file` 共用同一份。核心项：

| 变量 | 本环境值 | 说明 |
|---|---|---|
| `COMPOSE_PROJECT_NAME` | `ragflow` | 独立 Compose 项目，避免与宿主上的 `docker` 项目（dify）冲突 |
| `RAGFLOW_IMAGE` | `ragflow:go-local` | 使用第 4 节构建的本地镜像 |
| `SVR_WEB_HTTP_PORT` | **8088** | 对外 Web 端口（映射容器 nginx:80） |
| `SVR_HTTP_PORT` | `9380` | API 端口 |
| `ADMIN_SVR_HTTP_PORT` | `9381` | 管理 API 端口 |
| `RAGFLOW_DEV_MODE` | `true` | 分支构建版本号（commit hash）低于数据库迁移标记，开发模式跳过版本校验；**正式发布请置 `false` 并使用 tag 构建** |
| `DOC_ENGINE` | `elasticsearch` | 文档引擎 |
| `DB_TYPE` / `METADATA_DB_PROFILE` | `mysql` | 元数据库 |
| `DEVICE` | `cpu` | DeepDoc 推理设备 |
| `ES_PORT` | `1200` | ES 宿主机端口 |
| `EXPOSE_MYSQL_PORT` | `3306` | MySQL 宿主机端口 |
| `MINIO_PORT` / `MINIO_CONSOLE_PORT` | `9000` / `9001` | MinIO API / 控制台 |
| `KVROCKS_PORT` | `6379` | Kvrocks |
| `NATS_PORT` / 监控 | `4222` / `8222` | NATS |
| `MEM_LIMIT` | `8073741824`（≈8G） | ES 容器内存上限 |
| `ENABLE_REGISTER` | `1` | 允许自助注册（生产建议改 `0`） |
| `ALLOW_ANY_HOST` | `0` | 禁止数据库/连接器访问内网私网地址（保持 0） |
| `ADMIN_DEFAULT_PASSWORD` | `RagFlow@2026#Admin` | 见 6.2：Go 版启动时**不读取**该值，仅作记录 |

凭据（随机生成，已随分支提交；如需更换请同步修改后重建容器）：

| 变量 | 值 |
|---|---|
| `MYSQL_USER` / `MYSQL_PASSWORD` | `root` / `0aa84a27e5d3b799b6577bdca5fdb52c` |
| `ELASTIC_PASSWORD` | `4d85569d48547ca032aafb25bc5e1356` |
| `MINIO_USER` / `MINIO_PASSWORD` | `rag_flow` / `84536a0fca64d8fb24444227bc121d4c` |
| `REDIS_PASSWORD`（Kvrocks） | `9b5fe22f4489a406a9b42163a5b79079` |
| `CLICKHOUSE_PASSWORD`（未启用） | `2114e2295dd6671ba544261a6d74a9b3` |

修改后需重建容器生效：

```bash
cd docker && docker compose -f docker-compose.yml up -d --force-recreate
```

配置校验（不启动）：

```bash
cd docker && docker compose -f docker-compose.yml config --quiet && echo OK
```

---

## 6. 启动与验证

### 6.1 启动

```bash
cd ~/app/unionragFlow/ragflow/docker
docker compose -f docker-compose.yml up -d
docker compose ps
```

首次启动顺序：MySQL 健康 → 迁移（migrate）→ syncer → admin → api → ingestor → nginx。
API 就绪通常需要 30–90 秒，首次初始化数据库更久。

### 6.2 验证

```bash
# 1) 容器状态（5 个依赖应为 healthy）
docker compose ps

# 2) 健康检查（经 nginx，8088）
curl -s http://127.0.0.1:8088/api/v1/system/healthz
# 期望：{"db":"ok","redis":"ok","doc_engine":"ok","storage":"ok","message_queue":"ok","status":"ok"}

# 3) 前端页面
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.0.211:8088/     # 200

# 4) 管理 API
curl -s http://127.0.0.1:9381/healthz

# 5) 服务日志
docker compose logs -f --tail=200 ragflow-cpu
```

登录：<http://192.168.0.211:8088> → `admin@ragflow.io` / `RagFlow@2026#Admin`
管理端（服务状态 / 用户管理 / 配置查看）：<http://192.168.0.211:8088/admin> → 同一超管账号
（nginx 已将 `/api/v1/admin` 反代到 9381，因此管理端只需 8088 一个入口。）

#### 关于超管密码（重要）

Go 版 `InitDefaultAdmin` 的初始密码被硬编码为 `admin`，`.env` 中的 `ADMIN_DEFAULT_PASSWORD` 在启动阶段**不生效**。本环境已在首次启动后将该账号密码改为 `RagFlow@2026#Admin`（写入 `rag_flow.user.password`，算法为 werkzeug scrypt；登录时前端对密码做 `base64` 后 RSA 加密，服务端按 `base64(password)` 校验）。

用命令行验证密码（注意传 base64，而不是明文）：

```bash
curl -s -X POST http://127.0.0.1:8088/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d "{\"email\":\"admin@ragflow.io\",\"password\":\"$(printf '%s' 'RagFlow@2026#Admin' | base64)\"}"
# 返回 "code":0 即成功
```

重置超管密码（忘记密码 / 新建环境时）：

```bash
cd ~/app/unionragFlow/ragflow/docker

# 1) 生成 werkzeug 兼容 scrypt 哈希（密码先 base64）
HASH=$(python3 - <<'EOF'
import base64, hashlib, os
pw = base64.b64encode(b"RagFlow@2026#Admin")
salt = base64.b64encode(os.urandom(12)).decode()
h = hashlib.scrypt(pw, salt=salt.encode(), n=32768, r=8, p=1, dklen=64, maxmem=128*1024*1024)
print(f"scrypt:32768:8:1${salt}${h.hex()}")
EOF
)

# 2) 写入数据库（密码从 .env 读取，勿硬编码）
MYSQL_PWD=$(grep -E '^MYSQL_PASSWORD=' .env | cut -d= -f2)
docker exec -i ragflow-mysql-1 mysql -uroot -p"$MYSQL_PWD" \
  -e "update rag_flow.user set password='$HASH' where email='admin@ragflow.io';"

# 3) 按 6.2 的 curl 验证
```

也可在前端管理端（<http://192.168.0.211:8088/admin> → 用户管理 → 修改密码）修改。

### 6.3 模型接入（union AI gateway）

本环境通过 `https://ai-gateway.union-laboratory.com`（OpenAI 兼容网关）提供模型能力，已为普通账号
`chengs@unionclin.com` 的租户完成配置。

#### 6.3.1 前置：容器必须能出网

宿主与容器都**没有**公网直连路由，只有宿主上的 HTTP 代理 `127.0.0.1:7890` 可出网。RAGFlow 进程在容器内，
必须显式注入代理，否则「添加模型 / 校验连接 / 拉取模型列表 / 实际推理」全部表现为超时或失败：

```yaml
# docker/docker-compose.yml → services.ragflow-cpu.environment
- HTTP_PROXY=${RAGFLOW_HTTP_PROXY:-}
- HTTPS_PROXY=${RAGFLOW_HTTPS_PROXY:-}
- NO_PROXY=${RAGFLOW_NO_PROXY:-}
- http_proxy=${RAGFLOW_HTTP_PROXY:-}
- https_proxy=${RAGFLOW_HTTPS_PROXY:-}
- no_proxy=${RAGFLOW_NO_PROXY:-}
```

```bash
# docker/.env
RAGFLOW_HTTP_PROXY=http://host.docker.internal:7890
RAGFLOW_HTTPS_PROXY=http://host.docker.internal:7890
RAGFLOW_NO_PROXY=localhost,127.0.0.1,::1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,mysql,minio,es01,kvrocks,nats,clickhouse,tei
```

> `host.docker.internal` 由 compose 的 `extra_hosts: host.docker.internal:host-gateway` 映射到宿主。
> `NO_PROXY` 里的私网段与服务名保证 MySQL / MinIO / ES / Kvrocks / NATS 等内部流量**不走**代理。

自检：

```bash
docker exec ragflow-ragflow-cpu-1 sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" https://ai-gateway.union-laboratory.com/v1/models \
   -H "Authorization: Bearer $RAGFLOW_API_KEY"'
# 期望 200；502/超时说明代理或 NO_PROXY 配错
```

#### 6.3.2 网关能力（已实测）

| 能力 | 路径 | 说明 |
|---|---|---|
| 模型列表 | `GET /v1/models` | 71 个模型 |
| 对话 | `POST /v1/chat/completions` | OpenAI 兼容 |
| 向量 | `POST /v1/embeddings` | `text-embedding-v4`，默认 **1024 维**，**单请求最多 10 条** |
| 重排 | `POST /v1/rerank` | Jina/Cohere 风格，`qwen3-rerank` |

#### 6.3.3 已完成的配置

| 项 | 值 |
|---|---|
| Provider | `OpenAI-API-Compatible` |
| Instance | `union-gateway` |
| Base URL | `https://ai-gateway.union-laboratory.com/v1`（**必须带 `/v1`**） |
| API Key | 见 `tenant_model_instance.api_key`（部署时写入） |
| 对话模型（默认） | `deepseek-v4.1-flash-1017`（另含 `qwen3.8-max`） |
| 向量模型（默认） | `text-embedding-v4` |
| 重排模型（默认） | `qwen3-rerank` |
| 视觉模型 | `qwen3-vl-plus`（chat + vision） |

向量模型的 `tenant_model.extra` 必须带维度/批大小元数据，否则调用报
`input embedding max dimension is nil`：

```json
{"max_tokens": 8192, "max_dimension": 1024, "dimensions": [1024], "max_batch_size": 10}
```

同时 `docker/.env` 中 `EMBEDDING_BATCH_SIZE=10`（默认 16 会被网关以 400 拒绝）。

#### 6.3.4 验证

```bash
BASE=http://127.0.0.1:8088
B=$(printf '%s' '<普通用户密码>' | base64)
TOK=$(curl -s -D - -o /dev/null -X POST $BASE/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d "{\"email\":\"chengs@unionclin.com\",\"password\":\"$B\"}" \
  | grep -i '^authorization:' | sed 's/^[Aa]uthorization: //' | tr -d '\r')

# 1) 租户默认模型
curl -s -H "Authorization: $TOK" $BASE/api/v1/users/me/models

# 2) 对话
curl -s -X POST $BASE/api/v1/chat/to_model -H "Authorization: $TOK" \
  -H 'Content-Type: application/json' \
  -d '{"provider_name":"OpenAI-API-Compatible","instance_name":"union-gateway","model_name":"deepseek-v4.1-flash-1017","question":"你好"}'

# 3) 向量（注意 dimension=1024）
curl -s -X POST $BASE/api/v1/embeddings -H "Authorization: $TOK" \
  -H 'Content-Type: application/json' \
  -d '{"provider_name":"OpenAI-API-Compatible","instance_name":"union-gateway","model_name":"text-embedding-v4","texts":["hello"],"dimension":1024}'

# 4) 重排
curl -s -X POST $BASE/api/v1/rerank -H "Authorization: $TOK" \
  -H 'Content-Type: application/json' \
  -d '{"provider_name":"OpenAI-API-Compatible","instance_name":"union-gateway","model_name":"qwen3-rerank","query":"什么是RAG","documents":["RAG是检索增强生成","今天天气不错"]}'
```

#### 6.3.5 在界面上自行添加模型（新租户参考）

1. 用**普通账号**登录 <http://192.168.0.211:8088>（超管会被 403 拦截，见 8 节）；
2. 右上角头像 → **模型提供商** → 添加提供商 → 选择 `OpenAI-API-Compatible`；
3. 实例名填 `union-gateway`，API Key 填网关 Key，Base URL 填 `https://ai-gateway.union-laboratory.com/v1`；
4. 保存后点「校验连接」，在模型列表中勾选需要的模型（向量模型要填维度 1024）；
5. 回到模型提供商首页，把默认对话/向量/重排模型切到刚添加的模型。

> 说明：API Key 以明文保存在 `tenant_model_instance.api_key`（上游设计如此），数据库备份需按敏感数据管理。

---

## 7. 日常运维

### 7.1 状态与日志

```bash
cd ~/app/unionragFlow/ragflow/docker

docker compose ps                       # 状态
docker compose logs -f ragflow-cpu      # 主服务（api/admin/ingestor/syncer 共用）
docker compose logs -f mysql es01       # 依赖
docker stats --no-stream                # 资源占用

# 容器内落盘日志（syncer 等）
sudo ls -l ragflow-logs/                # 目录由容器内 root 创建
```

### 7.2 启停与重建

```bash
docker compose stop                     # 停止（保留容器）
docker compose start                    # 启动
docker compose restart ragflow-cpu      # 仅重启主服务
docker compose down                     # 删除容器与网络，保留数据卷
docker compose down -v                  # ⚠️ 连数据卷一起删除（数据全丢）
```

### 7.3 分支代码更新后重新部署

```bash
cd ~/app/unionragFlow/ragflow
git fetch origin dev-custom && git checkout dev-custom && git pull --ff-only

# 重新构建（命令同 4.1）
docker build --platform linux/amd64 --network host \
  --build-arg HTTP_PROXY=http://127.0.0.1:7890 --build-arg HTTPS_PROXY=http://127.0.0.1:7890 \
  --build-arg http_proxy=http://127.0.0.1:7890 --build-arg https_proxy=http://127.0.0.1:7890 \
  -t ragflow:go-local .

cd docker
docker compose up -d --force-recreate ragflow-cpu
docker compose logs -f --tail=100 ragflow-cpu
```

数据库迁移由 `entrypoint.sh` 在容器启动时自动执行（migrate 阶段）。**升级前务必先备份**（见 7.4）。

> 若报数据库版本不匹配：确认 `.env` 中 `RAGFLOW_DEV_MODE=true`；正式发布应用 tag 构建并置 `false`。

### 7.4 备份与恢复

```bash
cd ~/app/unionragFlow/ragflow/docker
BK=~/backup/ragflow-$(date +%F-%H%M); mkdir -p "$BK"

# 1) MySQL 全库
MYSQL_PWD=$(grep -E '^MYSQL_PASSWORD=' .env | cut -d= -f2)
docker exec ragflow-mysql-1 mysqldump -uroot -p"$MYSQL_PWD" \
  --single-transaction --routines --triggers --databases rag_flow > "$BK/mysql.sql"

# 2) 对象存储 + 向量索引（停服后整卷打包最稳妥）
docker compose stop ragflow-cpu
docker run --rm -v ragflow_minio_data:/data -v "$BK":/backup alpine \
  tar czf /backup/minio.tar.gz -C /data .
docker run --rm -v ragflow_esdata01:/data -v "$BK":/backup alpine \
  tar czf /backup/esdata.tar.gz -C /data .
docker compose start ragflow-cpu

# 3) 配置文件
cp .env "$BK/.env"; cp docker-compose.yml "$BK/"
```

恢复（示例）：

```bash
docker compose down
docker run --rm -v ragflow_minio_data:/data -v ~/backup/xxx:/backup alpine \
  sh -c 'rm -rf /data/* && tar xzf /backup/minio.tar.gz -C /data'
docker run --rm -v ragflow_esdata01:/data -v ~/backup/xxx:/backup alpine \
  sh -c 'rm -rf /data/* && tar xzf /backup/esdata.tar.gz -C /data'
docker compose up -d          # 等 mysql 健康后导入
docker exec -i ragflow-mysql-1 mysql -uroot -p"$MYSQL_PWD" < ~/backup/xxx/mysql.sql
docker compose restart ragflow-cpu
```

### 7.5 磁盘清理

```bash
docker system df
docker builder prune              # 清理构建缓存（下次重建变慢）
docker image prune                 # 清理悬空镜像
```

---

## 8. 故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 构建时 apt/curl 大量 `connection timed out` | 容器无法直连外网 | 使用 4.1 的代理参数 + `--network host` |
| `libpcre2-8 not found ... apt-get` 下载失败 | `sudo` 重置了代理变量 | 使用含 `sudo -E` / root 直接 apt 的 `build.sh`（本分支已修复） |
| `go mod download` 偶发失败 | 模块代理连接被重置 | Dockerfile 已内置 5 次重试；仍失败则重跑构建 |
| ES 容器反复重启 / `max virtual memory areas too low` | `vm.max_map_count` 过小 | `sudo sysctl -w vm.max_map_count=262144`（需 root） |
| 启动即退出，日志提示 DB 版本不匹配 | 分支版本号低于数据库迁移标记 | `.env` 设 `RAGFLOW_DEV_MODE=true` |
| 8088 打不开 | 容器未就绪 / 端口被占 | `docker compose ps`、`ss -lntp \| grep 8088`、`docker compose logs ragflow-cpu` |
| nginx 返回 502 | api_server 尚在启动 | 等待 `/api/v1/system/healthz` 返回 200；`docker compose logs -f ragflow-cpu` |
| 访问 8443 失败 | 镜像 nginx 不监听 443 | 预期行为；需要 TLS 请前置反代 |
| `admin@ragflow.io` 登录失败 | 密码是 Go 启动时的硬编码默认值 | 按 6.2 重置密码；命令行验证时密码需 base64 |
| 模型配置页 / 模型相关接口 403 | 用了超管账号（`is_superuser=1`）| 超管只能进 `/admin`；用普通账号登录主界面 |
| 添加模型时连接超时 / 拉不到模型列表 | 容器没有出网代理 | 按 6.3.1 配 `RAGFLOW_HTTP_PROXY` + `NO_PROXY`，重建容器 |
| 模型接口 404 | Base URL 少了 `/v1` | 填 `https://ai-gateway.union-laboratory.com/v1` |
| `input embedding max dimension is nil` | `tenant_model.extra` 缺维度/批大小元数据 | 补 `max_dimension/dimensions/max_batch_size`（见 6.3.3） |
| embeddings 返回 400 `InternalError.Algo` | `text-embedding-v4` 单批最多 10 条 | `.env` 设 `EMBEDDING_BATCH_SIZE=10`，模型 extra `max_batch_size: 10` |
| 页面所有接口 403 但 token 正常 | 前端还持有超管会话 | 退出后用普通账号重新登录 |
| `permission denied: docker/ragflow-logs/...` | 日志目录由容器内 root 创建 | 用 `sudo` 查看或 `sudo chown -R $USER docker/ragflow-logs` |
| `docker compose up` 提示容器名冲突 | 与其它栈同名（如 dify 的 `docker` 项目） | 本环境已通过 `COMPOSE_PROJECT_NAME=ragflow` 隔离 |
| 磁盘不足 | 构建缓存 + 镜像占用 | 见 7.5，预留 ≥ 20 GB |

排查通用命令：

```bash
docker compose ps
docker compose logs --tail=300 ragflow-cpu
docker inspect ragflow-ragflow-cpu-1 --format '{{.State.Status}} {{.State.ExitCode}}'
curl -sv http://127.0.0.1:8088/api/v1/system/healthz
```

---

## 9. 安全建议

1. **依赖端口不要暴露到公网**：MySQL/ES/MinIO/Kvrocks/NATS 当前发布在 `0.0.0.0`。仅内网使用时建议二选一：
   - 在 `docker/docker-compose-base.yml` 的 `ports` 前加 `127.0.0.1:` 绑定；或
   - 用防火墙限制来源网段。
2. **修改默认凭据**：本环境已随机生成（见第 5 节）；若仓库可能转公开，请先轮换 `docker/.env` 中全部密码并重新提交。
3. **关闭自助注册**：`ENABLE_REGISTER=0`（当前为 `1`）。
4. **TLS**：在前面挂 Nginx/Traefik，`proxy_pass http://192.168.0.211:8088`。
5. **生产模式**：使用 release tag 构建并设 `RAGFLOW_DEV_MODE=false`。
6. **PAT 轮换**：部署时使用的 GitHub PAT 应视为已泄露，使用后立即在 GitHub 设置中撤销并重新生成；不要在 `.git/config` 中长期保存 token。
7. 管理 API（9381）无额外鉴权门槛（仅 token 鉴权），不要直接对外。

---

## 10. 附录

### 10.1 本分支相对 `main` 的改动

```bash
cd ~/app/unionragFlow/ragflow
git diff main --stat
```

| 文件 | 改动 |
|---|---|
| `build.sh` | pcre2 安装命令在 root 下直接执行、否则 `sudo -E`，保证代理变量不被清除 |
| `docker/.env` | `COMPOSE_PROJECT_NAME=ragflow`、`SVR_WEB_HTTP_PORT=8088`、`RAGFLOW_IMAGE=ragflow:go-local`、`RAGFLOW_DEV_MODE=true`、随机凭据、停止发布 443 |
| `docker/docker-compose.yml` | 注释掉 `${SVR_WEB_HTTPS_PORT}:443` 映射（nginx 无 TLS 监听）；为 `ragflow-cpu` 注入出网代理变量 |
| `docker/.env` | 新增 `RAGFLOW_HTTP_PROXY/HTTPS_PROXY/NO_PROXY`；`EMBEDDING_BATCH_SIZE=10`（网关向量接口限制） |
| `DEPLOY.md` | 本文档 |

### 10.2 命令速查

```bash
# 目录
cd ~/app/unionragFlow/ragflow              # 代码
cd ~/app/unionragFlow/ragflow/docker       # 部署

# 构建（走代理）
docker build --platform linux/amd64 --network host \
  --build-arg HTTP_PROXY=http://127.0.0.1:7890 --build-arg HTTPS_PROXY=http://127.0.0.1:7890 \
  --build-arg http_proxy=http://127.0.0.1:7890 --build-arg https_proxy=http://127.0.0.1:7890 \
  -t ragflow:go-local .

# 部署 / 状态 / 日志 / 停止
docker compose -f docker-compose.yml up -d
docker compose ps
docker compose logs -f --tail=200 ragflow-cpu
docker compose down

# 健康检查
curl -s http://127.0.0.1:8088/api/v1/system/healthz

# 进入容器
docker exec -it ragflow-ragflow-cpu-1 bash

# MySQL 客户端
docker exec -it ragflow-mysql-1 mysql -uroot -p"$(grep -E '^MYSQL_PASSWORD=' .env | cut -d= -f2)"
```

### 10.3 变更记录

| 日期 | 版本 | 说明 |
|---|---|---|
| 2026-10-20 | v1.1 | 新增 6.3 模型接入：容器出网代理、union AI gateway 配置（对话/向量/重排）、向量维度与批大小限制、验证命令；补充 5 条模型相关故障排查 |
| 2026-10-20 | v1.0 | 初次部署：源码构建 `ragflow:go-local`，Compose 项目 `ragflow`，对外 8088；记录代理构建、pcre2 修复、超管密码处理 |