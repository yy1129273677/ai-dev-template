# DEPLOY.md —— 部署手册（工作区通用模板）

> 干什么：把「这套东西怎么上服务器、怎么更新、怎么验收、出问题怎么查」写清楚。**复制到项目根目录后按占位符填写**。
> 从三个真实项目提炼：`AutomatedDispatchSystem`（整站 compose + dev compose）、`BackendManagementFramework`（本机构建推镜像）、`jisu-ai-customer`（自签 HTTPS + 离线镜像 + 详尽验收）。
>
> 配套：[`AGENTS.md`](./AGENTS.md)（工程约定）、[`DEV_WORKFLOW.md`](./DEV_WORKFLOW.md)（发布与回滚流程）、[`PROMPT_TEMPLATE.md`](./PROMPT_TEMPLATE.md)。
>
> 填写约定：`{{...}}` 是占位符；`〔可选〕` 标记的章节按项目实际情况保留或删除；**不适用的部署方式直接删掉，不要留三种让人猜**。

---

## 0. 这套东西是什么

`{{项目名}}`（{{一句话定位}}），部署形态：

| 容器 | 镜像 | 作用 | 对外端口 |
| ---- | ---- | ---- | -------- |
| `web` | 本地构建（Nginx） | 托管前端静态文件，反代 `/api`、`/uploads` | `{{WEB_PORT}}`（默认 8080） |
| `server` | 本地构建（Node {{版本}}） | {{框架}} API，启动前自动跑数据库迁移 | 内网 3000（不对外） |
| `db` | `pgvector/pgvector:pg16` | PostgreSQL 16 + pgvector | 内网 5432（不对外） |

```
浏览器 ──► Nginx(web:{{WEB_PORT}})
             ├── /            → 前端静态文件
             ├── /api/*       → server:3000
             └── /uploads/*   → server:3000
                                  │ Prisma / pg
                                  ▼
                            db:5432（volume 持久化）
```

数据持久化在 Docker volume：`{{DB_VOLUME}}`（数据库）、`{{UPLOAD_VOLUME}}`（上传文件）。

**第一次部署记住三件事**：根目录 `.env` 要改密码和密钥、数据库不对外暴露、部署后按第 7 节验收清单逐条过。

### 0.1 先选部署方式（四选一，删掉其余）

| 方式 | 适合场景 | 做法 | 缺点 |
| ---- | -------- | ---- | ---- |
| A. 服务器上 `compose build` | 服务器能正常拉基础镜像、配置够 | `docker compose up -d --build` | 构建慢、吃内存；国内拉镜像可能卡死 |
| B. 本机构建 → 传镜像 `docker load` | **2G 小机器 / 基础镜像拉不动**（BMF、jisu 都用这个） | 本机 `build` → `docker save` → `scp` → 服务器 `docker load` → `up -d` | 多一步传输，需要本机 Docker |
| C. 仓库内离线镜像 `*.tar.gz` | 目标机器完全断网 | 镜像 tar 随包携带，服务器 `docker load -i` 后 `up -d` | 仓库体积大（几百 MB），需定期更新 |
| D. 本地只起数据库（开发） | 日常开发 | `docker compose -f docker-compose.dev.yml up -d` | 不是部署，见 `server/README.md` |

---

## 1. 日常更新（最常用，先看这里）

### 1.1 标准两步走

```bash
# ① 本机：改完代码，提交并推送（便于回滚、服务器同步）
git add -A && git commit -m "fix: {{改了什么}}" && git push

# ② 服务器：拉代码 → 重建 → 重启 → 自检
cd {{PROJECT_DIR}} && git pull
docker compose up -d --build          # 只重建有变化的镜像
docker compose ps                     # 三个容器都 healthy/Up 才算成功
curl -fsS {{PUBLIC_BASE}}/api/health  # 期望 {"status":"ok",...}
```

〔可选〕把第②步写成脚本 `update.sh`：`git pull && docker compose up -d --build && 等健康检查 && 自检`，一条命令搞定（jisu 的做法）。

### 1.2 只改前端（后端完全不碰）—— 三种速度

| 做法 | 速度 | 适用 |
| ---- | ---- | ---- |
| 重建 web 镜像 `docker compose up -d --build web` | 中 | 前端代码改了，走正常流程 |
| 本机构建 → 覆盖静态文件 | **最快** | 只改页面/样式；把 `client/dist/**` scp 到服务器挂载目录，浏览器刷新即可（jisu 的 `deploy-dist.ps1`） |
| 服务器上重新 `npm run build` | 慢 | 不推荐（服务器装前端依赖占资源） |

> Vite 每次构建产物带 hash，旧文件不会自动消失；长期只换静态文件要定期清理 `assets/`（jisu 的 `prune-dist.sh` 思路：保留 index.html 引用的文件 + 最近 N 个历史版本，其余删除）。

### 1.3 什么时候必须重建镜像？

| 改动 | 要不要重建 |
| ---- | ---------- |
| 前端页面/样式 | 重建 `web`（或按 1.2 换静态文件） |
| 后端代码 | 必须重建 `server` |
| `prisma/schema.prisma` + 迁移文件 | 必须重建 `server`（启动时自动 `migrate deploy`） |
| 依赖 / 基础镜像 / Dockerfile | 重建对应镜像（锁文件要一起提交） |
| 只改 `docker-compose.yml` 或根 `.env` | 不用重建，`docker compose up -d` 重建容器 |
| 只改 `server/.env` | 重启 `server` 容器 |

### 1.4 一键发布脚本（方式 B 的落地）

```powershell
# 本机（项目根目录）
.\deploy-images.ps1                  # 构建前后端镜像 → 上传 → 服务器导入 → 重启 → 自检
.\deploy-images.ps1 -Only web        # 只更新前端
.\deploy-images.ps1 -SkipBuild       # 不重新构建，直接上传现有镜像

# 服务器（项目根目录）
bash update.sh                        # 拉代码 → 重建 → 重启 → 自检
```

> 为什么脚本化：`docker save/load + scp` 手动做容易漏步骤；脚本里把「等健康检查 → curl 自检 → 失败即停」固化了。

---

## 2. 服务器准备

### 2.1 选型

- 1 核 2G 起步（前端 + 后端 + PG 同机够用；视频/大文件场景再加内存和磁盘）
- 磁盘 40G 起（镜像 + 数据库 + 上传文件）
- 系统：Ubuntu 22.04 / Debian 12 / 阿里云 Linux 均可
- **不要用 2G 机器在服务器上 build 前端**：`npm install` + vite build 容易 OOM，用方式 B

### 2.2 安全组 / 防火墙

| 端口 | 是否对公网开放 | 用途 |
| ---- | -------------- | ---- |
| `{{WEB_PORT}}`（80/8080） | ✅ | 网站入口 |
| `{{HTTPS_PORT}}`（443） | 〔可选〕✅ | HTTPS |
| 3000（server） | ❌ | 只允许 Nginx 内网访问 |
| 5432（db） | ❌ | 只在 Docker 网络内，绝不对外 |
| 22（SSH） | ⚠️ 仅限自己的 IP | 运维 |

> compose 里数据库**不要写 `ports:`**，需要本地调试再看，调试完删掉。

### 2.3 安装 Docker

```bash
curl -fsSL https://get.docker.com | sh        # 国内慢就先换镜像源
sudo usermod -aG docker $USER                 # 退出重新登录生效
docker --version && docker compose version    # 本手册用 v2 语法（docker compose，中间空格）
```

### 2.4 镜像加速（国内机器必做，否则 `docker pull` 大概率拉不动）

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{ "registry-mirrors": ["https://{{你的专属加速地址}}.mirror.aliyuncs.com"] }
EOF
sudo systemctl restart docker
```

> 加速地址在阿里云控制台「容器镜像服务 → 镜像加速器」里拿，**账号专属**；公共加速源经常不可用。
> 坑：`docker build`（buildkit）拉镜像可能卡死，但单独 `docker pull <image>` 往往很快——先 pull 再 build 会稳很多。

### 2.5 npm 镜像 / 离线构建

- 项目包内放 `.npmrc` 指向 `registry.npmmirror.com`（容器里 `npm ci` 也走国内源；海外部署删掉即可）。
- 服务器构建前 `docker pull node:{{版本}}-alpine`、`docker pull nginx:{{版本}}-alpine` 预热基础镜像。

### 2.6 时区与时钟

容器默认 UTC；数据库存 UTC、前端按本地时区展示即可。需要容器也用东八区：compose 加 `TZ=Asia/Shanghai`。证书/令牌校验依赖系统时间，`date` 不对先 `ntpdate`。

---

## 3. 把代码放到服务器上

```bash
# 方式一：git clone / pull（推荐，便于回滚与 update.sh）
git clone {{仓库地址}} {{PROJECT_DIR}} && cd {{PROJECT_DIR}}

# 方式二：本机打包上传（没仓库时；排除依赖和产物）
tar --exclude=node_modules --exclude=dist --exclude=uploads -czf {{项目}}.tar.gz .
scp {{项目}}.tar.gz root@{{SERVER_IP}}:/opt/ && ssh root@{{SERVER_IP}} \
  "mkdir -p {{PROJECT_DIR}} && tar xzf /opt/{{项目}}.tar.gz -C {{PROJECT_DIR}}"
```

> 分支注意：如果 `main` 落后于开发分支，明确写清「服务器上要 `git checkout {{分支}}`」，别让人拉到旧代码（jisu 踩过）。

---

## 4. 环境变量（两个文件，分工不同）

### 4.1 根目录 `.env`（compose 读，**必须配**）

```bash
cp .env.example .env && vim .env
```

| 变量 | 怎么填 |
| ---- | ------ |
| `POSTGRES_PASSWORD` | 强密码（数据库虽不对外，别偷懒） |
| `JWT_SECRET` / `SESSION_SECRET` | 32 字节以上随机串（见 4.3） |
| `ADMIN_PASSWORD` | 管理员初始密码，别用示例值 |
| `CORS_ORIGIN` | 真实访问地址，如 `http://{{SERVER_IP}}:{{WEB_PORT}}`；**生产不配后端拒绝启动** |
| `WEB_PORT` | 对外端口 |
| 〔可选〕`TZ` / `NODE_ENV` | 生产保持默认（`production`）；容器验收可临时 `development` |

### 4.2 `server/.env`（业务密钥，服务器上也可以只走 compose 注入）

| 变量 | 说明 |
| ---- | ---- |
| `DATABASE_URL` | compose 内部用 `db:5432`；本地开发用 `127.0.0.1:{{DB_HOST_PORT}}` |
| `JWT_SECRET` / `JWT_EXPIRES_IN` | 与根 `.env` 保持一致（或直接由 compose 注入） |
| `SMS_FIXED_CODE` | 固定验证码：**仅非生产生效**，生产环境接口直接报错 |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | seed 创建的管理员 |
| `UPLOAD_DIR` / `MAX_*` | 上传目录与大小上限 |

> 原则：**根 `.env` 只放 compose 需要的（数据库口令、密钥、端口）；`server/.env` 只给本地后端用**。生产环境变量通过 compose `environment` 注入，不要把 `server/.env` 传上服务器。

### 4.3 生成强密钥

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### 4.4 缺配置必须拒绝启动（刻意行为）

生产缺 `CORS_ORIGIN`、`JWT_SECRET`、`POSTGRES_PASSWORD` 时，compose 或后端应**直接失败**，不要用空值兜底。

---

## 5. 构建并启动

### 方式一：服务器上构建（简单）

```bash
docker compose up -d --build
docker compose ps                 # 三个都 Up，db/server healthy
docker compose logs -f server     # 看到「后端已启动」再继续
```

首次构建 5–10 分钟（取决于网络），之后有缓存会快。

### 方式二：本机构建 → 传镜像 → 服务器导入（2G 机器推荐）

```powershell
# 本机（PowerShell，项目根目录）
docker compose build
docker save {{server镜像}}:latest -o server.tar
docker save {{web镜像}}:latest -o web.tar
scp server.tar web.tar root@{{SERVER_IP}}:/opt/

# 服务器
docker load -i /opt/server.tar && docker load -i /opt/web.tar
cd {{PROJECT_DIR}} && docker compose up -d
```

### 方式三：仓库内离线镜像（断网机器）

```bash
# 本机
docker save postgres:16 client:latest server:latest | gzip > {{项目}}-images.tar.gz
# 服务器
docker load -i {{项目}}-images.tar.gz && docker compose up -d
```

---

## 6. 首次初始化（只做一次）

```bash
# 1) 迁移（server 容器启动时已自动跑 migrate deploy；手动确认一次）
docker compose exec server npx prisma migrate deploy

# 2) 种子数据（游戏/价目表/管理员/示例数据）
docker compose exec server npx ts-node prisma/seed.ts

# 3) 项目特有初始化〔按需保留〕
#    - 知识库/语料入库：docker compose exec server npm run ingest
#    - 业务数据迁移脚本：docker compose exec server npm run migrate:00X
#    - 初始化 SQL：docker compose exec -T db psql -U {{DB_USER}} -d {{DB_NAME}} < initdb/init.sql
```

> seed/迁移脚本要能**重复跑**（已迁移输出「无需迁移」），并在文档里写明「生产跑之前先备份」。

---

## 7. 验收清单（部署完逐条 curl，别跳）

```bash
BASE={{PUBLIC_BASE}}          # 例：http://1.2.3.4:8080

# ① 静态页面 200，登录页能打开
curl -s -o /dev/null -w "%{http_code}\n" $BASE/

# ② 健康检查
curl -s $BASE/api/health
#   期望：{"status":"ok",...}（若含 db/pgvector 字段，应都为 true）

# ③ 未登录调业务接口 → 401（最关键：说明接口真锁上了）
curl -s -o /dev/null -w "%{http_code}\n" -X POST $BASE/api/{{受保护接口}} \
  -H "Content-Type: application/json" -d '{}'
#   期望：401

# ④ 错误密码/验证码 → 401
curl -s -o /dev/null -w "%{http_code}\n" -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" -d '{"username":"admin","password":"wrong"}'
#   期望：401

# ⑤ 正确登录 → 200 + token
TOKEN=$(curl -s -X POST $BASE/api/auth/login -H "Content-Type: application/json" \
  -d '{"username":"{{ADMIN_USER}}","password":"{{ADMIN_PASSWORD}}"}' | jq -r .token)
#   期望：拿到非空 token

# ⑥ 带 token 调业务接口 → 200
curl -s $BASE/api/auth/me -H "Authorization: Bearer $TOKEN"
#   期望：200

# ⑦ 越权 → 403（用低权限账号调管理员接口）

# ⑧ 静态资源 gzip 生效（小带宽的关键）
curl -s -I -H "Accept-Encoding: gzip" $BASE/assets/{{某个js}} | grep -i content-encoding
#   期望：content-encoding: gzip

# ⑨ 上传与回显：上传一个文件 → 返回 url → 浏览器能打开

# ⑩ 〔自签 HTTPS〕-k 忽略证书只看服务
curl -sk -o /dev/null -w "%{http_code}\n" https://{{SERVER_IP}}/api/health
```

项目特有验收〔按需保留/删除〕：

- 核心业务链路冒烟：`API_BASE=$BASE/api node server/test/smoke.mjs`（会创建测试账号，验收后可清理）
  > 生产模式下脚本会提示「业务链路跳过（生产模式）」：固定验证码被禁用是设计行为，不是故障。
- 多租户：用两个不同 `systemId` 的账号验证数据隔离（BMF）。
- 配置驱动：用示例 TCID 取渲染配置（BMF）。
- 流式输出：`curl -N` 看是否逐字返回（jisu 的 SSE）。
- 知识库：提问一次确认能检索到内容（jisu）。

---

## 8. 端口与防火墙（部署后核对）

| 检查 | 命令/动作 | 期望 |
| ---- | --------- | ---- |
| 对外只开 web | 云控制台安全组 | 只有 `{{WEB_PORT}}`/443 对公网 |
| 数据库未暴露 | `docker compose ps` 看 db 的 PORTS | 只有 `5432/tcp` 或 `127.0.0.1:...`，没有 `0.0.0.0:5432` |
| 反代路径正确 | 浏览器/curl `/api/health` | 200，不是 404 |
| HTTPS〔可选〕 | `curl -k https://...` | 200；证书 SAN 含 IP/域名 |

---

## 9. 日常运维

```bash
# 看日志（排查第一步）
docker compose logs -f server
docker compose logs --tail=200 web

# 重启 / 停止 / 启动
docker compose restart server
docker compose stop && docker compose up -d

# 数据库备份（建议 crontab 每天一次，保留 7 天，异地保存）
docker compose exec -T db pg_dump -U {{DB_USER}} {{DB_NAME}} | gzip > backup-$(date +%F).sql.gz

# 恢复（会覆盖现有数据，先确认备份可用）
gunzip -c backup-2026-10-22.sql.gz | docker compose exec -T db psql -U {{DB_USER}} -d {{DB_NAME}}

# 清理构建垃圾镜像 / 悬空卷（先看清楚再删）
docker image prune -f
docker system df

# 清理历史静态产物（Vite hash 文件堆积时）
bash prune-dist.sh          # 保留 index.html 引用的 + 最近 5 个历史版本
```

---

## 10. 常见故障对照

| 现象 | 原因 | 处理 |
| ---- | ---- | ---- |
| `server` 一直重启 | 缺 `CORS_ORIGIN`/`JWT_SECRET`/`POSTGRES_PASSWORD`；`DATABASE_URL` 错 | `docker compose logs server` 看报错，补 `.env` |
| 502 Bad Gateway | server 没起来 / 健康检查未过 | `docker compose ps`、看 server 日志 |
| 页面能开但接口 404 | Nginx 反代路径缺 `/api/` | 检查 `proxy_pass` 尾部 |
| 接口 401 | token 过期 / 没带 header / 密钥变了 | 重新登录；确认 `JWT_SECRET` 没被换 |
| 新增接口 401 但已登录 | 全局前缀/路由路径不一致 | 对比前端请求路径与 controller 前缀 |
| 上传 413 | 超过 Nginx `client_max_body_size` | 调大 nginx.conf 后重建 web |
| 迁移报 failed migrations | 迁移 SQL 执行失败被记录 | 修 SQL → `npx prisma migrate resolve --rolled-back 000X` → `migrate deploy` |
| db 密码改了但连不上 | volume 里还是旧密码 | 用旧密码，或备份后删 volume 重建（会丢数据） |
| `docker build` 卡在拉镜像 | buildkit 拉取慢/被墙 | 先 `docker pull` 基础镜像；或用方式 B 本机构建 |
| 时间显示差 8 小时 | 容器 UTC | 前端按本地时区展示；或 compose 加 `TZ=Asia/Shanghai` |
| 磁盘满 | 镜像/历史镜像/上传文件堆积 | `docker system df`、清理旧静态产物与孤儿上传 |
| 服务器 2G 构建时 OOM | 服务器上 build 前端 | 改成本机构建传镜像（方式 B） |
| PowerShell 脚本被执行策略拒绝 | E 盘文件被当「远程」，RemoteSigned 拒签 | 用 .cmd 包一层 `-ExecutionPolicy Bypass`（BMF 的 `deploy.cmd`） |

---

## 11. 上线前安全检查清单

- [ ] `JWT_SECRET` / `ADMIN_PASSWORD` / `POSTGRES_PASSWORD` 全部是强随机值，不是示例值
- [ ] `CORS_ORIGIN` 是真实地址（不配应拒绝启动）
- [ ] 数据库端口未对公网
- [ ] 上传目录只读、不执行；大小限制生效
- [ ] 未登录调业务接口返回 401；越权返回 403
- [ ] 固定验证码/调试接口在生产环境已失效
- [ ] HTTPS〔可选〕已配好，或明确记录「本次不启用及原因」
- [ ] 数据库已备份且恢复流程演练过
- [ ] 日志能看到请求错误与关键动作，且不含密码/令牌
- [ ] 服务器时间正确、磁盘有空间

---

## 12. 实战踩坑记录

> 每次部署遇到的坑写进来：**不写下来，下次还会踩**。

| 日期 | 项目 | 现象 | 根因 | 修法 | 防回归 |
| ---- | ---- | ---- | ---- | ---- | ------ |
| | | | | | |

已沉淀的通用坑（可直接复用）：

1. **镜像加速源不可靠** → 必须用账号专属地址；`docker pull` 单独执行比 buildkit 稳。
2. **2G 机器在服务器 build 前端会 OOM / 卡死** → 本机构建 → `save/scp/load`。
3. **迁移文件带 BOM** → PostgreSQL 报 `syntax error at or near ""`；用无 BOM UTF-8 写入。
4. **Prisma 在 alpine 缺 openssl** → Dockerfile 里 `apk add --no-cache openssl`。
5. **`class-validator` 未安装却注册了 ValidationPipe** → 启动直接报错；要么装齐，要么不注册（本项目用手写校验）。
6. **前端请求库不支持某个 HTTP 方法**（如 `uni.request` 不支持 PATCH）→ 接口统一用 GET/POST/PUT/DELETE。
7. **本机与服务端口冲突**（如本机已有 PostgreSQL 5432）→ 容器映射到 5435，生产不对外。
8. **冒烟脚本跑第二次就红** → 固定测试账号导致断言不成立；用时间戳生成账号，脚本幂等。
9. **Vite 旧 hash 产物越堆越多** → `prune-dist.sh` 保留 index.html 引用的 + 最近 N 个版本。
10. **`main` 分支落后于开发分支** → 服务器拉到旧代码；在 DEPLOY 里写清分支，或统一用发布分支。
11. **PowerShell 执行策略拒签本地 ps1** → E 盘被视为远程；用 `.cmd` wrapper 加 `-ExecutionPolicy Bypass`（只影响单次）。
12. **只换静态文件后页面白屏** → 旧 index.html 引用的 assets 被清掉了；清理脚本必须保留当前引用。

---

## 附 A：三项目部署方式对照

| 项目 | 部署方式 | 特殊点 | 验收重点 |
| ---- | -------- | ------ | -------- |
| **BackendManagementFramework** | 本机构建镜像 → 上传 → 服务器导入（`deploy-images.ps1` / `deploy.cmd`） | 多租户初始化脚本、示例 TCID 链路、`systemId` 隔离 | 登录 → 拿子系统列表 → 用 TCID 取渲染配置 |
| **AutomatedDispatchSystem（陪易玩）** | 服务器 `compose up -d --build`；开发用 `docker-compose.dev.yml` 只起库 | 启动自动迁移、seed 游戏/管理员、生产禁用固定验证码、有 41 项冒烟 | 未登录 401、下单链路、结单弹框、权限兜底 |
| **jisu-ai-customer** | 本机构建传镜像或服务器 build；**自签 HTTPS**；`update.sh` / `deploy-dist.ps1` / `prune-dist.sh` | 无域名用 IP 证书、离线镜像搬运、首次知识库入库、流式输出 | 登录 200、未登录 401、静态 gzip、HTTPS、流式逐字返回 |

## 附 B：这份手册的写作要求（给 AI）

1. **命令可直接复制执行**，不要写「自行调整」这种模糊话；占位符统一 `{{...}}`。
2. **验收必须可执行**：每条 curl 给期望结果（状态码/字段）。
3. **首次部署与日常更新分开写**，日常更新放最前面（最常用）。
4. **不要写三种部署方式却不说明何时用哪种**；不适用的删掉。
5. 踩坑记录写「现象/根因/修法/防回归」，不要只写「注意 XX」。
6. 密钥、IP、密码一律占位符，**真实值绝不进仓库**。
