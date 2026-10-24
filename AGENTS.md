# AGENTS.md —— 工作区通用工程约定（AI 协作模板）

> 适用范围：本工作区（`E:\study\project`）下的所有项目，包括
> `AutomatedDispatchSystem/`（陪易玩）、`BackendManagementFramework/`（后台管理框架）、`jisu-ai-customer/`（AI 客服）。
>
> **优先级**：项目内自己的 `AGENTS.md`（尤其是第二部分业务约定） > 本文件 > 一般默认习惯。
> 本文件只做**通用基线**，不写任何项目的业务规则。
>
> 配套文件：
>
> - [`PROMPT_TEMPLATE.md`](./PROMPT_TEMPLATE.md)：怎么给 AI 写需求（场景化提示词模板）
> - [`DEV_WORKFLOW.md`](./DEV_WORKFLOW.md)：从需求到上线的开发流程与质量门禁
> - [`DEPLOY.md`](./DEPLOY.md)：通用部署手册模板（复制到项目根后按占位符填写）
>
> 来源：从上述三个真实项目的约定与踩坑中提炼（详见 §12 对照表）。

---

## 0. 怎么用这份模板

| 场景 | 做法 |
| ---- | ---- |
| 新建项目 | 复制 §1–§11 到项目根的 `AGENTS.md`，按 §13 填写第二部分（业务定制）；本文件留在工作区根做基线 |
| 已有项目 | 用 §1–§11 逐条对照，缺什么补什么；第二部分缺失就补（BMF/ADS 都有，jisu 目前没有） |
| 改技术栈 | 先写 ADR（`docs/adr/`），再改项目 `AGENTS.md` §1，最后才动代码（见 §1.3） |
| 交付验收 | 过 §14「交付前自检清单」+ `DEV_WORKFLOW.md` 的质量门禁表 |

> 铁律：**不要为了"统一"去改项目已定技术栈**。统一的是约定与流程，不是选型。

---

## 1. 工程形态与目录（三项目共同基线）

### 1.1 双包结构

前后端是**两个独立 npm 包**，放在同一个仓库里：

```
<项目>/
├── client/            # 前端（独立 npm 包）
├── server/            # 后端（独立 npm 包）
├── docs/              # 需求 / 设计 / ADR / 术语 / 图
├── AGENTS.md          # 本约定（通用部分 + 本项目业务部分）
├── DEPLOY.md          # 唯一部署文档
├── docker-compose.yml # 整站部署编排
└── .env.example       # 环境变量模板（真实 .env 不提交）
```

不引入 monorepo 工具链（pnpm workspace / turbo），不把两个包合成一个。根目录只放开发期工具、编排和文档。

### 1.2 目录基线

| 路径 | 放什么 | 不放什么 |
| ---- | ------ | -------- |
| `client/` | 页面、组件、请求封装、样式、前端测试 | 业务规则真相（以后端为准） |
| `server/` | 接口、Service、Prisma schema/迁移/seed、后端测试 | 前端构建产物 |
| `docs/requirements/` | **需求原文**（只增不改，编号 `001-YYYYMMDD.md`） | 拆解、方案、实现说明 |
| `docs/design/` | 需求拆解确认、架构、数据模型、接口、页面流程、状态机、安全 | 需求原文 |
| `docs/adr/` | 架构决策记录（为什么这么选、什么条件下改选别的） | 使用手册 |
| `docs/glossary.md` | 术语表（防止 AI 和人各叫各的） | — |
| `docs/diagrams/` | 手绘/截图/流程图 | — |
| 根 `DEPLOY.md` | 部署方式 + 注意事项（唯一部署文档） | 子包怎么跑 |
| 包内 `README.md` | 这个包怎么跑、注意什么 | 部署命令 |

> 三项目均已采用「双包 + docs + DEPLOY.md」；jisu 目前没有 `docs/` 和 `AGENTS.md`，属于待补齐项。

### 1.3 技术栈：三项目实际选型矩阵（不要擅自替换）

| 项目 | 形态 | 前端 | 后端 | 数据库 | 权限模型 | 实时/交互 | 部署 |
| ---- | ---- | ---- | ---- | ------ | -------- | --------- | ---- |
| **BackendManagementFramework** | 多租户后台管理框架（PC） | Vue 3 + Vite + **Element Plus** + Pinia + vue-router + axios + ECharts | NestJS 11 + Prisma 6 + **class-validator** + Passport JWT | PostgreSQL + pgvector | 多租户 `systemId` + RBAC 权限码 + 动态表格/路由 | 无（表单/表格为主） | compose + 离线镜像 tar.gz |
| **AutomatedDispatchSystem（陪易玩）** | 陪玩接单移动端（H5） | **uni-app**（Vue 3 + Vite + TS）+ Pinia + 自研组件 | NestJS 11 + Prisma 6 + 全局守卫 + 手写校验 | PostgreSQL 16 + pgvector | 单租户 + 三角色权限矩阵 | 轮询（3s 消息 / 10s 通知） | 整站 compose + 开发仅 db 的 dev compose |
| **jisu-ai-customer** | AI 客服（PC Web） | Vue 3 + Vite + Element Plus | **Express + LangChain** + pg | PostgreSQL + pgvector | 单管理员 Cookie 会话 | 流式输出（SSE） | compose + 自签 HTTPS + 离线镜像 |

三条结论：

1. **同类选型优先复用本项目既有栈**（后台类项目用 Element Plus 那套，移动端用 uni-app 那套，AI/RAG 类用 Express+LangChain 那套）。
2. **换栈必须有 ADR**，并说明迁移成本与失败模式。
3. **共同底线不变**：TypeScript、Prisma（有 ORM 时）、PostgreSQL、JWT/会话、Docker Compose、Prettier。

---

## 2. 通用命令速查

```bash
# ---------- 工作区根 / 项目根（格式化，优先 npx，不往仓库里装 node_modules）----------
npx prettier@3.7.4 --write .   # 整个仓库格式化
npx prettier@3.7.4 --check .   # 只检查不改

# ---------- 前端（client/）----------
npm install
npm run dev            # 本地开发
npm run typecheck      # vue-tsc --noEmit（桌面端）/ 类型检查
npm test               # vitest（只测纯函数）
npm run build          # 构建（build 前必须能过 typecheck）

# ---------- 后端（server/）----------
npm install
npm run start:dev      # 本地开发（写完接口先看 Swagger）
npm run typecheck      # tsc --noEmit
npm test               # jest --runInBand（不连数据库）
npm run build          # prisma generate + nest build

# ---------- 数据库（有 Prisma 的项目）----------
npm run prisma:generate
npm run prisma:migrate:deploy     # 应用已有迁移（不改数据）
npm run prisma:migrate:diff       # 离线生成增量 SQL → 人工检查 → 粘进新迁移
npm run seed                      # 初始化/演示数据（生产慎用）
```

> 例外：老项目若根目录已有 `package.json`（只装 Prettier），保留它的 `npm run format` 也可以，但**不要再往根目录加业务依赖**。

---

## 3. 硬性规则（安全与数据底线，违反即回滚）

1. **不放宽标识符校验**：动态表名/列名是 PostgreSQL 唯一不能参数化的地方，必须走统一校验函数 + 注入用例（BMF 的 `identifiers.ts` 就是范例；没有这功能的项目不要新造）。
2. **数据值不拼 SQL**：值一律参数化（Prisma `sql` / `$queryRaw` 的模板参数），只有校验过的标识符才能进 SQL 文本。
3. **不开放「原始 SQL WHERE」给用户**：复杂查询用结构化条件组。
4. **不用 `prisma migrate dev`**：会连库、可能重置数据；新增迁移用离线 `migrate diff` + 人工检查。
5. **单测不连数据库**：纯函数/规则测试；需要真库的 e2e 放 `server/test/`，独立脚本跑。
6. **权限判断必须在后端**：前端隐藏按钮只是体验；新接口加 `@RequirePermissions` / 角色守卫，并同步权限清单。
7. **多租户必须带租户条件**：项目如果有 `systemId`/`tenantId`，任何查询漏它就是跨租户泄露（BMF 属这类）。
8. **响应字段白名单**：Prisma 显式 `select`；禁止返回 `passwordHash`、`token`、`secret`、`DATABASE_URL`、完整手机号等敏感字段；不该给对方看的（如陪玩看老板手机号）一律打码或不返回。
9. **上传必须四重校验**：扩展名/MIME 白名单 + 文件头魔数 + 大小上限 + UUID 文件名；上传目录只读、不执行。
10. **测试后门只限非生产**：固定验证码、调试接口、默认口令在 `NODE_ENV=production` 必须失效（宁可启动报错，也不要静默放行）。
11. **危险操作先确认**：删库、覆盖数据、重置仓库、改系统配置、`git push --force`，必须明确得到用户同意。
12. **密钥不进代码**：`.env` 只提交 `.env.example`；发现硬编码密钥先报告再处理。

---

## 4. 代码风格（硬约束）

1. **风格由 `.prettierrc.json` 唯一定义**（三项目一致：printWidth 120、2 空格、双引号、分号、尾逗号 all、LF、Vue script/style 不额外缩进）。想改先改这个文件。
2. **编辑器 formatOnSave + Prettier 扩展**；提交前先格式化，避免"提交时又重排一遍"。
3. **不私自引入 ESLint**：三项目目前都没接 lint 链，统一靠 typecheck + 单测 + 约定扫描测试。
4. **SFC 块顺序**：`<template>` → `<script setup>` → `<style>`（BMF/ADS 都是硬约束）。
5. **样式**：桌面端 `Less` + `variables.less` 统一定义颜色/圆角/阴影，不写死十六进制；移动端（uni-app）尺寸用 `rpx`。
6. **类型检查在构建前**：前端 `build` 前先 `typecheck`；后端交付前 `tsc --noEmit` 必过。

---

## 5. 接口与 Swagger 规范（新增接口必须遵守）

> Swagger 只认声明式元数据，不认 TS 类型（编译后擦除）。不写装饰器 = 文档里没有请求体。

1. controller 类上 `@ApiTags("中文分组")`；需要登录加 `@ApiBearerAuth()`；公开接口加 `@Public()`（或项目对应机制），并在 summary 标注「公开接口」。
2. **每个路由方法必须有 `@ApiOperation({ summary })`**：一句中文说清干什么；有副作用/权限/状态机限制时补 `description`。
3. 路径参数 `@ApiParam`、query 参数 `@ApiQuery`，`required` 必须和实际可选性一致。
4. 请求体必须是 DTO 类：字段 `@ApiProperty` / `@ApiPropertyOptional`，中文业务含义 + **真实可用的 example**（禁止 `"string"` 占位）。
5. 复杂对象数组用 `type: Object` + description 说明结构，不为文档硬造嵌套 DTO。
6. 文档描述必须与源码校验一致（数量上限、枚举、必填）。
7. 自检：`npx tsc --noEmit -p tsconfig.json`。
8. 参考实现：BMF 的 `table-config` / `auth` 模块、ADS 的 `auth` / `orders` / `audits` 模块。

---

## 6. 后端安全基线

| 项 | 要求 |
| -- | ---- |
| 认证 | 全局守卫默认保护；放行必须显式声明；JWT 有效期合理（7 天级别），角色变更可接受延迟 |
| 授权 | 权限码集中定义在一处（`permissions.ts`），角色→权限矩阵可单测；新权限同步文档表格 |
| 输入 | 静态结构用 DTO；动态字段必须白名单过滤 + 类型转换；不要开 `whitelist:true` 除非全部 DTO 都有校验装饰器 |
| 响应 | 显式 `select` / 手工白名单；敏感字段不返回 |
| 限流 | `@nestjs/throttler`：全局 120 次/分钟/IP；验证码、登录、上传、导出单独收紧；探活豁免 |
| 安全头 | helmet 开启；CSP 由 Nginx 管 |
| CORS | 生产没配 `CORS_ORIGIN` 必须拒绝启动；`origin:true` 只允许非生产 |
| 上传 | 见 §3.9；Nginx 只读静态托管 |
| 日志 | 审核/下单/结单/导出等关键动作留结构化日志；不含密码/令牌/完整手机号 |
| 备份 | 生产库定时 `pg_dump`，保留 7 天，并做恢复演练 |
| 自检三问 | 这个端点有守卫吗？输入校验/白名单吗？返回字段安全吗？回答否必须说明原因与风险 |

---

## 7. 数据库与迁移

```bash
# 改表结构的标准流程（禁止 migrate dev）
npx prisma validate
npm run prisma:generate
npm run prisma:migrate:diff > prisma/migrations/000X_名字/migration.sql
# 人工检查 SQL（DROP/改类型/索引），然后
npm run prisma:migrate:deploy
```

- 迁移文件**进 git**；`0.001` 类初始迁移里可以带 `CREATE EXTENSION IF NOT EXISTS vector`。
- 金额用整数（元或分）+ **下单时快照单价**，不要用浮点；改价不影响历史数据。
- 状态字段用 enum，并画出状态机（`docs/design/` 里要有迁移表）。
- seed 只用于开发/首次部署；跑 seed 前确认不会覆盖生产人工数据。
- 默认账号写在 `AGENTS.md` 第二部分 + `.env.example`，不写死在代码里。

---

## 8. 测试与质量门禁

| 层 | 测什么 | 放哪 | 是否连库 |
| -- | ------ | ---- | -------- |
| 后端单测 | 价格/上限/排序/权限矩阵/校验器/状态机纯函数 | `server/src/**/*.spec.ts` | ❌ |
| 约定扫描 | 每个路由有 `@ApiOperation`、受保护 controller 有 `@ApiBearerAuth`、不用 `any` 接 `@Body()` | `controller-conventions.spec.ts` | ❌ |
| 前端单测 | 格式化/校验/纯计算 | `client/tests/*.spec.ts` | ❌ |
| e2e 冒烟 | 注册→审核→核心流程→权限兜底 | `server/test/*.mjs`（独立脚本） | ✅ |
| 手工验收 | 部署后用 curl 过验收清单 | `DEPLOY.md` | ✅ |

**交付门禁（缺一不可）**：

```bash
cd server && npm run typecheck && npm test
cd client && npm run typecheck && npm test && npm run build
npx prettier@3.7.4 --check .
node server/test/smoke.mjs          # 起了后端+库时跑
```

> e2e 脚本必须**可重复跑**：用时间戳生成测试手机号/账号，不要依赖固定数据（ADS 踩过这个坑）。

---

## 9. 文档组织

- **需求原文只放 `docs/requirements/`，不改**；编号 `001-YYYYMMDD.md`。
- **需求拆解与确认放 `docs/design/`**：一句话理解、角色权限、功能逐条拆解、业务规则、数据模型、接口、页面、状态机、需求缺口、待拍板问题（Q1..Qn）、验收对照。
- **ADR 放 `docs/adr/`**：技术栈、实时方案、前端形态等"以后一定有人问为什么"的决定。
- **`DEPLOY.md` 放根目录，是唯一部署文档**；包内 README 只讲怎么跑、注意什么。
  > 工作区根目录的 [`DEPLOY.md`](./DEPLOY.md) 是通用部署手册模板（日常更新/服务器准备/环境变量/构建/首次初始化/验收 curl/运维/故障/踩坑），项目内复制后按占位符填写。
- **交付至少 3 个 md**：根（DEPLOY）+ client README + server README；`docs/README.md` 做索引。
- 改代码必须同步文档：需求变更 → 原文 + 拆解文档 + 项目 AGENTS 第二部分业务规则 + 权限清单。

---

## 10. 部署约定

- **compose 分层**：`docker-compose.yml` 整站（db + server + web）；`docker-compose.dev.yml` 本地只起数据库（开发时前后端用 npm 跑）。
- **env 拆分**：根 `.env` 给 compose（数据库口令、端口、密钥）；`server/.env` 给后端本地开发。真实 env 不提交。
- **健康检查**：db 用 `pg_isready`，server 用 `wget /api/health`，web 依赖 server。
- **国内环境**：配 Docker 镜像加速；包内 `.npmrc` 指向 npmmirror；2G 小机器用"本地构建镜像→传镜像"的方式（jisu 的做法）。
- **服务器可选自签 HTTPS**（没有域名时）：jisu 的 `deploy/ip.crt|ip.key` + Nginx 443 是一份现成参考。
- **部署方式与脚本命名必须和已有项目保持一致**：服务器放完整仓库（git clone）；镜像本机构建后上传导入（`deploy-images.ps1` + `deploy.cmd`），服务器端 `update.sh` 只做「拉代码 + 用已上传镜像重建 + 自检」；端口写死在 compose 里。**不要每个项目另发明一套流程**（参考 BMF 8081+3001 / jisu 8080 / 陪易玩 8082+3002）。- **`DEPLOY.md` 必备章节**：日常更新（两步走/只改前端/何时重建）→ 服务器准备（镜像加速）→ 代码上传 → 环境变量 → 构建启动（两种方式）→ 首次初始化（迁移+seed）→ **验收清单（curl）** → 运维（日志/重启/备份/清理）→ 常见故障对照 → 实战踩坑记录。
- **验收最关键一条**：未登录调业务接口必须 401（说明接口真的锁上了）。

---

## 11. AI 协作规则（人机分工）

**人负责**：需求决策、验收标准、危险操作确认、上线/回滚决定、凭据与密钥。
**AI 负责**：读代码、给方案对比、实现、补测试、写文档、跑验证命令、报告风险。

AI 必须遵守：

1. **先读后写**：没读过相关代码/文档不写方案；读旧代码先复述理解的架构与数据流。
2. **先问后改**：逻辑变更、跨文件改动、删改代码、新增依赖、需求有歧义 —— 先说明"改哪里/怎么改/为什么/影响面/备选方案"，等确认。
3. **先方案后代码**：多方案对比 + 推荐 + 什么条件下改推另一个。
4. **最小改动**：不顺手重构无关代码，不擅自引入依赖。
5. **不确定就说不确定**：不编造 API/版本/参数/路径；先查源码或文档。
6. **不擅自 git 提交/推送**，除非明确要求（要求时也要先确认 `.gitignore` 与敏感文件）。
7. **改完自证**：给出执行的命令 + 结果，不写"应该能跑"。
8. **报告格式**：做了什么 / 为什么这么做 / 影响哪些文件 / 怎么验 / 有什么风险或遗留。

---

## 12. 三项目对照表（提炼出的可复用资产）

| 维度 | BMF | ADS（陪易玩） | jisu | 通用做法 |
| ---- | --- | ------------- | ---- | -------- |
| 工程结构 | client + server + docs + compose | 同左 + dev compose | client + server + deploy + initdb | 双包 + docs + DEPLOY |
| 约定文件 | AGENTS 两部分（含"常见业务改动怎么做"） | AGENTS 两部分（含术语铁律、权限清单） | 无 | **补 AGENTS 两部分** |
| 需求文档 | requirements 001–004 + 整理版 | requirements 原文 + design 拆解 | 无 | 原文进 requirements，拆解进 design |
| 设计文档 | context/domain/flows/states/rules/failure/security + ADR + glossary + diagrams | 拆解/架构/数据/接口/页面/状态机/安全 + ADR | 无 | design 分层 + ADR + glossary |
| 安全 | 多租户 systemId、RBAC、动态标识符校验、class-validator | 单租户三角色、权限矩阵、手写校验、上传四重校验、限流 | Cookie 会话、htpasswd、自签 HTTPS | 认证+授权+输入+响应+限流+上传+备份 |
| 测试 | jest 单测 + 约定扫描（推测） | 71 单测 + 11 前端测试 + 41 项冒烟 | 无 | 单测不连库 + e2e 脚本 + 约定扫描 |
| 部署 | compose + 离线镜像 + deploy 脚本 | compose + dev compose + 运维/验收/踩坑 | compose + 自签 HTTPS + 离线镜像 + 详细验收 | 统一 DEPLOY.md 结构 |
| 最值得抄的 | 业务改动 SOP、权限码清单、动态表格校验 | 需求拆解→确认、状态机文档、冒烟脚本、防并发 | 部署手册详尽度、HTTPS/离线镜像、验收 curl | 三者互补，见 §9/§10 |

---

## 13. 项目自己的 `AGENTS.md` 第二部分要写什么

每个项目在自己根 `AGENTS.md` 里追加（新建项目可整段复制后填写）：

```markdown
## 8. 项目定位与关键决策
- 一句话：谁用、解决什么问题、核心流程是什么
- 名字与功能是否一致（不一致就改名或标注二期）
- 需求基线：原文路径 + 拆解确认文档路径（有歧义时以拆解文档为准）

## 9. 术语表
| 原文叫法 | 代码名 | 含义 | 注意 |
（同一角色多个叫法、同一词多个含义必须在这里统一）

## 10. 数据库 / 迁移 / seed 命令与默认账号
- 本地库怎么起（compose 文件、端口、库名、账号）
- 迁移/seed/生成命令
- 默认账号（管理员、测试角色）与生效条件

## 11. 业务规则与权限清单
- 业务硬性规则（状态机、上限、可见性、排序、金额）
- 权限码表（权限码 / 含义 / 允许角色）

## 12. 需求 002 之后的约定
- 新需求编号规则、文档放哪、合并前要更新哪些文档

## 13. 常见业务改动怎么做（SOP）
- 加一个权限码：改哪几个文件、同步哪张表
- 加一张表/字段：迁移流程 + 响应白名单 + 测试
- 加一个页面/接口：controller/DTO/Swagger/权限/前端请求封装
- 加一个定时任务/消息：入口、幂等、失败重试

## 14. bug 与踩坑记录
| 日期 | 现象 | 根因 | 修法 | 防回归 |
（踩过的坑不写下来，下次还会踩）

## 15. 目录结构（本项目实际）
```

---

## 14. 交付前自检清单

- [ ] `server`: `npm run typecheck` + `npm test` 全绿
- [ ] `client`: `npm run typecheck` + `npm test` + `npm run build` 全绿
- [ ] `npx prettier@3.7.4 --check .` 全绿
- [ ] 改了 schema：迁移文件人工检查过，`migrate deploy` 通过，无 `migrate dev`
- [ ] 新接口：有守卫、有输入校验/白名单、有 Swagger 元数据、响应字段安全
- [ ] 新权限码：`permissions.ts` 与文档权限表同步
- [ ] e2e 冒烟通过（含权限兜底：未登录 401、越权 403）
- [ ] `DEPLOY.md` 的验收清单能照着跑通（至少本地 compose 起一遍）
- [ ] 需求变更：原文 + 拆解文档 + 项目 AGENTS 业务规则 + 权限表同步
- [ ] 文档：新增/修改的 md 已按 `docs/README.md` 索引归档
- [ ] 敏感信息：无密钥/口令/连接串进代码或 git（`git status` 确认 `.env` 被忽略）
