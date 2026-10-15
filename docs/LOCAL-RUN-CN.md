# 在本机把 DeskcommCRM 跑起来（开发路径）

这不是客户上线用的 `hostgator-setup-kit/install.sh`。那条路要域名、Supabase 云项目和安装脚本。这里是 README「Desenvolvimento」和 [`SETUP.md`](SETUP.md) 的开发路径：Node 在本机，`pnpm dev` 起 Next.js。

下面的命令都在仓库根目录执行。密钥只放 `.env.local`。这个文件被 `.gitignore` 的 `.env*` 忽略，不要提交。

2026-10-15 在一台 Ubuntu 24.04 云 VM（Node v22.14.0、pnpm 9.15.9、Docker 29）上量过：没有 Wall 的生产密钥，首页、登录页和健康检查可以起来。智能体回复、真 WhatsApp 配对、发信仍然要你自己的账号。

## 1. 装依赖

```bash
node -v          # 需要 v22（.nvmrc 是 22）
corepack enable
corepack prepare pnpm@9.15.9 --activate
pnpm install --frozen-lockfile
```

`pnpm typecheck` 在默认堆上会在大约 4 GB 处被 Node 杀掉（`Ineffective mark-compacts near heap limit`）。加大堆之后通过：

```bash
NODE_OPTIONS='--max-old-space-size=8192' pnpm typecheck
pnpm lint
```

`pnpm test:unit` 要在**还没有** `.env.local` 的时候跑。Vitest 会把 `.env.local` 读进进程。文件里一旦有 `UPSTASH_REDIS_REST_URL`，限流测试就不再走内存计数器；一旦有 `NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321`，会把「另一台 Supabase」的用例认成自己的库。

## 2. 本地数据库：用 baseline，不要 `supabase db push`

历史 migration 里 `0001`–`0009` 和 `0013` 是 `SELECT 1;` 空壳。`supabase db push` 会成功退出，库却是空的。安装用的 schema 是 `supabase/baseline.sql`。

本仓库的 `scripts/local-supabase.sh start` 会先把 `supabase/migrations` 挪开，再 `supabase start`，然后在库里创建扩展并灌入 baseline。失败时脚本会把 migrations 目录放回去。

```bash
bash scripts/local-supabase.sh start
npx supabase status -o json
```

状态 JSON 里要用的字段：`API_URL`（本机是 `http://127.0.0.1:54321`）、`ANON_KEY`、`SERVICE_ROLE_KEY`、`DB_URL`（`127.0.0.1:54322`）。Studio 在 `http://127.0.0.1:54323`。

如果 `supabase start` 停在 `Initialising schema...` 并且报 `error running container: exit 1`，先看容器之间能不能互通。这台 VM 上 `iptables-legacy` 的 `FORWARD` 策略是 `DROP`，Realtime 连不上 Postgres，15 秒后整栈被拆掉。当时的处理：

```bash
sudo iptables-legacy -P FORWARD ACCEPT
docker network create nettest
docker run -d --name a --network nettest alpine:3.20 sleep 30
docker run --rm --network nettest alpine:3.20 ping -c 1 a
```

这台 VM 没有 systemd，`systemctl enable docker` 不会把 dockerd 拉起来。需要的话在前台之外自己起 `containerd` 和 `dockerd`，并把 `/var/run/docker.sock` 交给当前用户（`sudo chmod 666 /var/run/docker.sock`，或重新登录以吃到 `docker` 组）。

### 用 Supabase 云项目代替本地 Docker

这是要你自己账号的一步。在 SQL 编辑器或 `psql` 里，**先**扩展、**再** baseline：

```bash
psql "$SUPABASE_DB_URL" -v ON_ERROR_STOP=1 -c \
  'create extension if not exists vector with schema public;
   create extension if not exists citext with schema public;
   create extension if not exists pg_trgm with schema public;'
psql "$SUPABASE_DB_URL" -v ON_ERROR_STOP=1 -f supabase/baseline.sql
```

连接串用 Session pooler 的 URI，不要用只走 IPv6 的 Direct connection。Storage 里建私有 bucket `whatsapp-media`。

## 3. `.env.local` 填写顺序

`pnpm dev`（`NODE_ENV=development`）**缺了下面三项就起不来**，`lib/env.ts` 里它们是 `requiredAlways`，并且 URL 必须是合法 `http(s)`：

1. `NEXT_PUBLIC_SUPABASE_URL` — 本地填 `http://127.0.0.1:54321`
2. `NEXT_PUBLIC_SUPABASE_ANON_KEY` — status JSON 的 `ANON_KEY`
3. `SUPABASE_SERVICE_ROLE_KEY` — status JSON 的 `SERVICE_ROLE_KEY`（绕过 RLS，只放服务器）

开发模式下，其余「生产才必填」的项空着也能启动（`required()` 在非 production 里默认成空字符串）。`next build` 在 `NEXT_PHASE=phase-production-build` 时会给缺的项填占位符，所以构建可以没有云密钥。`next start` / 生产镜像会重新要求真值。

建议按这个顺序补，每一项都可以后补：

| 顺序 | 变量 | 没有它时 |
|---|---|---|
| 4 | `SUPABASE_DB_URL` | 直接连 Postgres 的路由不用；本地填 status 的 `DB_URL` |
| 5 | 本地随机密钥 | 见下面的 `openssl`。生产模式缺了会拒绝启动 |
| 6 | Redis | 不填时 `/api/v1/health` 的 redis 是 `degraded`。本地可以用下面的容器，不必开 Upstash 账号 |
| 7 | 一个 AI 密钥 | 进程能起，日志会警告没有 `AI_GATEWAY_API_KEY` / `ANTHROPIC_API_KEY` / `OPENROUTER_API_KEY` / `OPENAI_API_KEY`。智能体要回复，至少填一个，或之后在「IA › Credenciais」里登记 |
| 8 | WAHA | 不填时健康检查里 waha 是 `degraded`。要收 WhatsApp 再做 |
| 可空 | `SENTRY_DSN`、`RESEND_API_KEY`、SMTP、Nuvemshop、Google 日历 | 监控、发信、电商、Google 日历按钮会缺；页面仍在 |

本地随机密钥（每条各跑一次，不要互相复用，不要写进 git）：

```bash
openssl rand -hex 32
openssl rand -base64 32
```

对应关系：

```bash
# hex 32
INTERNAL_SECRET=
INTERNAL_CRON_SECRET=
NUVEMSHOP_OAUTH_ENCRYPTION_KEY=
IMPERSONATE_COOKIE_SECRET=
LGPD_SIGNING_KEY=
WAHA_HMAC_SECRET=

# base64 32（AI_CRED_AES_KEY 必须是 32 字节的 base64）
CPF_ENCRYPTION_KEY=
WAHA_BYO_ENCRYPTION_KEY=
AI_CRED_AES_KEY=
```

`CPF_ENCRYPTION_KEY` 还要进数据库的 `private.app_secrets`（名字 `cpf_key`），否则联系人存得下、CPF 存不下。`install.sh` 会自己种这行。手工安装见 [`SETUP.md`](SETUP.md) 第 9 节的 `insert`。

地址：

```bash
NEXT_PUBLIC_APP_URL=http://127.0.0.1:3000
NEXT_PUBLIC_ADMIN_URL=http://127.0.0.1:3000
WAHA_WEBHOOK_BASE_URL=http://127.0.0.1:3000
```

### 本地 Redis（代替 Upstash）

`docker-compose.local.yml` 里的 `srh` 镜像是 `hiett/serverless-redis-http`，digest `sha256:5b0bb9239fce53abf87b2018a7a0deb9ec7bd900c5360738fe5fbeeb426f9150`。令牌自己生成，容器和 `.env.local` 用同一个值：

```bash
SRH_TOKEN="$(openssl rand -hex 24)"
docker network create deskcomm-local
docker run -d --name deskcomm-redis --network deskcomm-local redis:7-alpine \
  redis-server --save "" --appendonly no
docker run -d --name deskcomm-srh --network deskcomm-local -p 127.0.0.1:8079:80 \
  -e SRH_MODE=env \
  -e SRH_TOKEN="$SRH_TOKEN" \
  -e SRH_CONNECTION_STRING=redis://deskcomm-redis:6379 \
  hiett/serverless-redis-http@sha256:5b0bb9239fce53abf87b2018a7a0deb9ec7bd900c5360738fe5fbeeb426f9150
```

```bash
UPSTASH_REDIS_REST_URL=http://127.0.0.1:8079
UPSTASH_REDIS_REST_TOKEN=<同一个 SRH_TOKEN>
```

### 本地 WAHA

`docker-compose.yml` 的开发服务用的是浮动标签 `devlikeapro/waha:noweb`（仓库注释写的是 Plus；2026-10-15 拉下来的进程自报 CORE `2026.9.2`）。生产安装钉的是另一张镜像，不要把下面的观测当成生产契约。

这次量到的行为：请求头 `X-Api-Key` 必须和容器环境变量 `WAHA_API_KEY` 的字符串相同。把头设成 SHA512、容器里放的也是这份 SHA512 时，`/api/sessions` 返回 200；头是明文、容器里是哈希时返回 401。所以本地若要健康检查里的 waha 为 `ok`，两边都放同一条明文：

```bash
WAHA_KEY="$(openssl rand -hex 24)"
docker run -d --name deskcomm-waha -p 127.0.0.1:3030:3000 \
  --add-host=host.docker.internal:host-gateway \
  -e "WAHA_API_KEY=${WAHA_KEY}" \
  -e WHATSAPP_DEFAULT_ENGINE=NOWEB \
  -e WAHA_DASHBOARD_ENABLED=true \
  -e "WHATSAPP_HOOK_URL=http://host.docker.internal:3000/api/v1/webhooks/waha" \
  -e "WHATSAPP_HOOK_HMAC_KEY=${WAHA_HMAC_SECRET}" \
  devlikeapro/waha:noweb
```

```bash
WAHA_API_BASE_URL=http://127.0.0.1:3030
WAHA_API_KEY=<同一个 WAHA_KEY>
```

`/ping` 不鉴权。配对二维码要手机，这一步没有做。

## 4. 启动并核对

```bash
pnpm dev --hostname 127.0.0.1 --port 3000
```

另一个终端：

```bash
curl -sS -o /dev/null -w 'home:%{http_code}\n' http://127.0.0.1:3000/
curl -sS -o /dev/null -w 'login:%{http_code}\n' http://127.0.0.1:3000/login
curl -sS -D - -o /dev/null http://127.0.0.1:3000/app
curl -sS http://127.0.0.1:3000/api/v1/health
```

这台 VM 上的结果：`/` 200，`/login` 200，`/app` 307 到 `/login?next=%2Fapp`。三项 Supabase 密钥、本地 Redis、以及与容器一致的 WAHA 明文都填上之后，health 为 `healthy`（supabase、redis、waha 都是 `ok`）。日志里仍会警告没有 AI 密钥。

生产构建（同样不需要云上的 AI 密钥；`.env.local` 在的话 Next 会读它）：

```bash
NODE_OPTIONS='--max-old-space-size=8192' pnpm build
```

## 5. 第一个能登录的人

本地库灌完 baseline 之后**没有**业务用户。`/login` 打得开，没有账号可进。自托管第一个主人用仓库里的脚本（见 `scripts/bootstrap-owner.ts` 文件头）：

```bash
OWNER_EMAIL='you@example.com' OWNER_PASSWORD='换成你自己的强密码' \
  OWNER_ORG_NAME='Minha Empresa' \
  npx tsx scripts/bootstrap-owner.ts
```

脚本读 `.env` / `.env.local`。这一步会在本地 Auth 里创建用户，密码不要提交。我们没有跑这个脚本，也没有走完 `/signup`。

## 6. 还要你本人提供的东西

按你想用到的深度，而不是按「不填就起不来」：

1. **云上 Supabase**（若不用第 2 节的本地 Docker）：Project URL、anon key、service_role key、Session pooler URI。免费项目有项目数量上限。
2. **至少一个 AI 提供商的密钥**：`ANTHROPIC_API_KEY`，或 `OPENAI_API_KEY`，或 `OPENROUTER_API_KEY`，或 `AI_GATEWAY_API_KEY`。不填，页面在，智能体不回复。听语音、给知识库做 embedding 还要 OpenAI 或你在「IA › Credenciais」里登记的密钥。
3. **WhatsApp**：手机打开「已关联的设备」，在产品里扫 WAHA 的二维码。本地这张 CORE 镜像没有完成配对，也没有发出去一条消息。
4. **邮件**（邀请、找回密码）：`RESEND_API_KEY` + `RESEND_FROM_EMAIL`，或 SMTP 那一组。都不填时，邀请链接留在屏幕上。
5. **可选**：Sentry DSN、Nuvemshop、Google 日历 / Google Ads、自建 Jitsi。空着就少对应按钮，不挡启动。

不要把任何一项写进 git。核对：

```bash
git check-ignore -v .env.local
git status --short
```
