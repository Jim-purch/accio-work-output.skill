---
name: accio-work-output
version: 0.1.6
github_repo: https://github.com/Jim-purch/accio-work-output.skill.git
priority: primary
domain: forklift-parts-foreign-trade-sales
description: >-
  PRIORITY entry-point skill for TOOMOTOO forklift-parts foreign-trade sales
  work inside this .accio account — invoke FIRST at the start of EVERY work
  session, before any other skill, even when the user just says hi / opens the
  workspace / asks any forklift-parts sales question. Two bootstrap jobs it
  runs on session start: (1) self-check this skill's version against GitHub
  (https://github.com/Jim-purch/accio-work-output.skill.git) and notify the
  user of updates; (2) provide the Alibaba.com (ICBU) workctl CLI & accio-mcp
  authentication workflow for pulling real-time store data — trigger when the
  user needs Ali ICBU data access, gateway token injection, or hits workctl
  desktop_not_attached errors. After bootstrap, hand off to the matching
  companion skills dynamically discovered in this account's skills/ directory
  (never assume a fixed skill list — enumerate and match by description), and
  consult this skill's own reference/ library of sales-method playbooks when
  the task touches negotiation, first-reply, follow-up scripts, quoting,
  price reduction, customer reactivation, or the sales process (enumerate
  reference/ and match by each doc's header summary — never assume a fixed
  doc list). Generated files are written to the agent's default working
  directory — this skill imposes no special output folder.
---

# Accio Work Output Router

> **Priority / entry-point skill for TOOMOTOO forklift-parts foreign-trade
> sales.** This skill boots first at the start of every work session in this
> `.accio` account, self-checks for GitHub updates and exposes the
> ICBU/workctl data layer that all downstream sales skills depend on.
>
> | Field | Value |
> |-------|-------|
> | Version | see frontmatter `version:` at the top of this file (single source of truth) |
> | GitHub repo | https://github.com/Jim-purch/accio-work-output.skill.git |
> | Priority | `primary` — invoke FIRST on every session start |
> | Domain | TOOMOTOO forklift-parts foreign-trade sales |
> | Last verified | 2026-07-28 |

## What this skill is for

You are operating inside a `.accio` account workspace (the directory under
`.accio/accounts/<accountId>`) bound to TOOMOTOO (Tianjin) International
Trading — an Alibaba.com (ICBU) cgs seller of forklift parts.

This skill is the **priority entry point** for all forklift-parts
foreign-trade sales work here. It does two things, in order, on every
session start:

1. **Self-check GitHub updates** (see §"会话启动自检" below) — compare local
   `version` against the published repo and notify the user.
2. **Expose the ICBU/workctl data layer** — so downstream sales skills can
   pull real-time store data (store diagnose, conversations, RFQ, products,
   ads, trade, logistics) without re-deriving the auth flow.

In addition, this skill ships a **`reference/` library of sales-method
playbooks** (see §十). When a task matches a playbook's scenarios —
negotiation, first-reply, follow-up scripts, quoting, price reduction,
customer reactivation, sales-team enablement — enumerate `reference/`,
read the matching playbook, then combine its methods with live workctl data
and the dynamically discovered companion skills (§九) to give a complete,
context-specific answer.

Generated files (reports, exports, scripts, etc.) are written to the agent's
default working directory — no special output folder is enforced, since a
hard-coded folder may be unwritable on some setups.

## When to trigger

This is the **priority / entry-point skill** for this account. Trigger it
FIRST, before any other skill, whenever any of these apply:

- **A new work session starts in this account** (the user opens the workspace,
  sends the first message of a session, asks anything forklift-parts-sales
  related, or just says hi). On this first trigger, run the **会话启动自检**
  flow below before doing anything else.
- The user needs Ali ICBU data access, gateway token injection, or hits a
  workctl `desktop_not_attached` error.

## 会话启动自检（Session-start self-check）

> At the start of every work session, before handing off to any other skill,
> run this self-check **silently** (do not dump raw command output to the
> user — only report the conclusion in one or two lines, plus an action
> prompt if an update is available).

**Purpose**: keep this entry-point skill in sync with its GitHub source so the
TOOMOTOO sales workflow always runs on the latest auth flow, routing rules,
and ICBU command list.

### Constants

| Name | Value |
|------|-------|
| GitHub repo URL | `https://github.com/Jim-purch/accio-work-output.skill.git` |

### Self-check procedure

Run these steps once per session, in order. Stop at the first step that
succeeds and skip the rest.

> **Why GitHub-only (no local-file read):** On a freshly installed machine
> the skill directory (`.../skills/accio-work-output/`) may be empty or its
> files not yet materialized, so reading the on-disk `SKILL.md` to get the
> local version is unreliable. The local version is already known from this
> skill's loaded frontmatter (`version:` field — in the agent's memory at
> load time), and the remote version is fetched from the GitHub public repo.
> GitHub is the single source of truth for "latest"; the on-disk local file
> is never read for the check.

**Step 1 — fetch remote version from GitHub (the only network call).**

Fetch only the remote `SKILL.md` frontmatter and read its `version:` field.
The local version is the `version:` field of **this skill's own frontmatter**
(known in-memory from loading this skill — **do NOT** re-read it from the
on-disk `SKILL.md` via `sed`/shell, which breaks on empty new-install dirs).

```bash
# Remote version — from the GitHub public repo (raw). No <USER>/<ID> needed.
REMOTE_VER=$(curl -fsSL --max-time 8 \
  "https://raw.githubusercontent.com/Jim-purch/accio-work-output.skill/main/SKILL.md" \
  | sed -n 's/^version:[[:space:]]*//p' | head -1)
# LOCAL_VER = the version: field of THIS skill's loaded frontmatter
# (the single hard-coded version at the top of this file) — already in the
# agent's memory; do NOT read from disk.
# Compare semver (major.minor.patch):
#   REMOTE_VER == LOCAL_VER → UP_TO_DATE
#   REMOTE_VER >  LOCAL_VER → BEHIND
#   REMOTE_VER <  LOCAL_VER → AHEAD (local dev copy newer than published)
```

If `curl` returns a non-empty `REMOTE_VER`, report the comparison result
(table below) and stop.

**Step 2 — offline / network blocked.** If Step 1 fails (no network, GitHub
unreachable, sandbox blocks the request, or `REMOTE_VER` is empty), skip
silently and proceed with the local copy. Do not block the user's work.

### How to report to the user

After the check, surface the result to the user in **one short line** (Chinese,
since this account's primary user is a TOOMOTOO sales rep) plus an action
prompt only when an update exists:

> Placeholders `{LOCAL_VER}` / `{REMOTE_VER}` below are filled from the
> self-check variables above (`LOCAL_VER` = this skill's frontmatter
> `version:`, `REMOTE_VER` = the value fetched from GitHub). Never hard-code
> the version number in these messages — it must always come from the single
> source of truth at the top of this file.

| Result | What to tell the user |
|--------|----------------------|
| `UP_TO_DATE` | "✅ 已自检：本 skill 为最新版本 (v{LOCAL_VER})，与 GitHub 一致。" |
| `BEHIND` | "🔄 检测到 GitHub 有新版本 (本地 v{LOCAL_VER} → 远端 v{REMOTE_VER})。是否拉取更新？" — wait for user confirmation before updating; never auto-overwrite a skill the user is actively editing. To update: if the skill dir is a `git clone`, run `git -C <skill dir> pull origin main`; otherwise re-copy / re-install the skill from the repo. |
| `AHEAD` | "ℹ️ 本地版本 (v{LOCAL_VER}) 高于 GitHub 已发布版本，可能是本地开发版，无需更新。" |
| offline (Step 2) | "ℹ️ 未能连接 GitHub 检查更新（网络受限），本次会话使用本地 v{LOCAL_VER} 继续。" |

### Rules

1. **Never auto-update without user confirmation.** A `BEHIND` result only
   *prompts*; it does not pull. The user may have local edits in flight.
2. **Never block the session on the self-check.** If the check takes more than
   ~8 s or fails, proceed with the local copy and mention the skip in one line.
3. **Run once per session, not per message.** Cache the result in memory for
   the rest of the session; don't re-fetch on every user turn.
4. **Don't leak sensitive paths or tokens** in the report line — show only the
   version numbers and the one-line update hint.
5. After reporting, immediately continue with the rest of the bootstrap
   (expose the ICBU/workctl layer) and hand off to whatever the user
   actually asked for.

## Path convention (used throughout this skill)

All paths in this skill are written as **absolute paths** with two placeholders
that must be substituted to the actual values of the machine running the agent:

| Placeholder | Meaning | How to read at runtime |
|-------------|---------|------------------------|
| `<USER>` | Windows username of the account owner | `echo "$USERNAME"` in bash, or `%USERNAME%` in cmd |
| `<ID>` | Accio account ID | basename of the current working directory |

**Account root** template:

```
Account root:  C:\Users\<USER>\.accio\accounts\<ID>\
```

In bash / `node` commands, forward slashes work on Windows and avoid
backslash-escaping issues, so commands below use `C:/Users/<USER>/...` form.
Substitute `<USER>` and `<ID>` before running.

---

# 阿里国际站 workctl & accio-mcp 鉴权经验

> 本账号已绑定阿里国际站卖家账号（公司主体 Toomotoo (Tianjin) International Trading，cgs 卖家）。
>
> **重要原则**：账号身份（login_id / ali_id / serviceType）、workctl 版本号、网关端口、子账号数量与人员等**会变动的运行时值不硬编码在本 skill 里，使用时动态读取**。本 skill 只记录稳定的机制、命令名、路径、坑点与排查方法。
>
> **验证状态**：workctl 路径已验证可用 ✅；阿里 MCP 接口（accio-mcp）当前不可用 ❌（仅记录结论）；通用 accio-mcp-cli 可用 ✅。最近一次验证 2026-07-27。

## 一、账号身份信息（动态读取，勿硬编码）

账号身份会随账号绑定/变更而变化，**使用时从 OAuth 状态文件实时读取**：

| 读取项 | 来源（绝对路径） | 说明 |
|--------|------|------|
| login_id / ali_id / serviceType | `C:\Users\<USER>\.accio\accounts\<ID>\connectors\data\alibaba\state.json` | 账号唯一标识与服务类型 |
| 授权状态 / 授权时间 | `C:\Users\<USER>\.accio\accounts\<ID>\connectors\data\alibaba\state.json` | status / authorizedAt 字段 |
| 已安装插件 | `C:\Users\<USER>\.accio\accounts\<ID>\plugins\installed\` 目录 | 目录名即插件名 |

```bash
# 动态读取当前账号身份（输出 JSON，含 login_id/ali_id/serviceType/status 等）
# 把 <USER> 和 <ID> 替换为本机实际值
node -p "JSON.parse(require('fs').readFileSync('C:/Users/<USER>/.accio/accounts/<ID>/connectors/data/alibaba/state.json'))"
```

公司主体固定：Toomotoo (Tianjin) International Trading（阿里国际站 cgs 卖家，主营叉车配件）。

## 二、workctl CLI 鉴权（已验证可用 ✅）

### 2.1 工具版本与网关地址（动态读取）

workctl 版本与网关地址会随升级/重启变化，**使用时动态读取，勿硬编码**：

```bash
workctl --version          # 查 workctl 版本（如 v0.1.x）
workctl health             # 查网关 backend 状态与连通性
# 网关地址（url 字段）见 C:\Users\<USER>\.accio\accounts\<ID>\.accio\runtime\gateway-cli.json
```

### 2.2 凭证获取（核心，最关键的一步）

主 Agent 沙箱默认**没有** `ACCIO_GATEWAY_TOKEN` 环境变量，直接跑 workctl 动态命令会报 `desktop_not_attached`。但可以用 `gateway-cli.json` 的 password 注入：

```bash
# 把 <USER> 和 <ID> 替换为本机实际值
export ACCIO_GATEWAY_TOKEN=$(node -p "JSON.parse(require('fs').readFileSync('C:/Users/<USER>/.accio/accounts/<ID>/.accio/runtime/gateway-cli.json')).password")
```

注入后 workctl 动态命令（`icbu` 子命令）全部可用。

### 2.3 gateway-cli.json 字段说明

路径：`C:\Users\<USER>\.accio\accounts\<ID>\.accio\runtime\gateway-cli.json`（网关重启会重写整个文件）

| 字段 | 含义 | 是否会变 |
|------|------|---------|
| url | 网关 HTTP 地址（如 `http://localhost:40xx`） | 重启可能变 |
| wsUrl | WebSocket 地址 | 随 url 变 |
| authMode | 认证模式（basic） | 稳定 |
| username | 网关用户名 | 见文件 |
| password | 网关密钥（= `ACCIO_GATEWAY_TOKEN` 的值） | 重启可能变 |
| relayPort | 中继端口 | 会变 |
| pid / createdAt | 进程 ID / 创建时间 | 每次重启必变 |

> ⚠️ **重要**：网关重启后 `gateway-cli.json` 会重写（pid/createdAt 必变，password/relayPort 也可能变），使用时务必**实时读取原文件**，不要缓存旧值。具体字段值不硬编码于本 skill。

### 2.4 连通性验证

```bash
# 注入凭证后
workctl health
# 期望返回：backend=pass, version=pass（auth/config/network/cache 为 skip，属正常）
```

### 2.5 可用 icbu 子命令一览

| 子命令 | 用途 |
|--------|------|
| `advisor` | 经营大盘、流量画像、渠道流量、地域流量、商品效果、访客明细 |
| `crm` | 店铺诊断（store-diagnose-brief）、服务周报 |
| `product` | 商品发品/编辑、AI图片/视频、类目推断、选品 |
| `tm` | TM 会话/消息、客服诊断、买家背调、接待策略 |
| `rfq` | RFQ 商机搜索、详情、报价详情 |
| `trade` | 订单、违规查询、风险扫描、供应商验证 |
| `ads` | 广告诊断、投放管理（部分旧 API 已废弃，见坑点） |
| `logistics` | 物流运费查询、线路推荐、订单列表 |
| `member` | 子账号管理（list 子账号列表） |
| `cco` | 知识库（平台规则、市场准入、物流知识） |
| `storefront` | AI 旺铺（网页创建/编辑/预览/发布） |

### 2.6 已知坑点（实测踩雷）

| # | 坑点 | 表现 | 解决 |
|---|------|------|------|
| 1 | `-f json` 在子命令层级不识别 | `unknown shorthand flag: 'f'` | 去掉 `-f json`，默认输出即 JSON |
| 2 | `-o <path>` 在 Windows 下路径解析异常 | `create output file: open ... 报错` | 改用 bash `> file.json` 重定向 |
| 3 | TM 诊断需复合参数 | `missing required parameter: buyerType/dateType/queryDate/start_time` | `list-seller-*-dim-diag-data` 需同时传 buyerType + dateType + start_time 等 |
| 4 | 广告账户诊断旧 API 已废弃 | `code 410, This API is deprecated` | 迁移至 HATEOAS：`list --entityType=diagnosis`（省略 campaignId 做账户级） |
| 5 | 数据 T+2 延迟 | 最新可查日为前天 | 经营大盘等聚合数据最新为 T-2 |
| 6 | 子账号会话消息无权读 | `not in conversation` | 主账号非子账号会话成员，需子账号登录态 |

## 三、Accio MCP 鉴权机制

Accio Desktop 自带两套 MCP 能力，鉴权与可用性完全不同：

| 能力 | 工具 | 鉴权方式 | 可用性 |
|------|------|---------|--------|
| 阿里国际站 MCP 接口 | accio-mcp（connector-proxy 内置） | 走 connector OAuth | ❌ 当前不可用 |
| 通用 MCP 工具（Gmail/Twitter/Notion 等） | accio-mcp-cli（Node 脚本） | Desktop 自动注入 | ✅ 可用 |

### 3.1 阿里 MCP 接口（不可用 ❌，仅记录）

- 阿里 MCP 工具**未通过 connector-proxy 暴露**
- 插件 `connectors.json` 的 `mcpServers={}`（空配置）
- `ListMcpResources` 仅返回 Ardot Editor，无阿里相关资源
- `alibaba-chat-and-analysis` 子代理需 `sessions_spawn` 调度，主 Agent 工具集无此能力
- **结论**：当前无法通过 MCP 直接调用阿里接口，必须走 workctl icbu 路径（见第二节）

### 3.2 accio-mcp-cli 鉴权机制（通用 MCP，可用 ✅）

> 本节源自 `accio_mcp_cli_proxy.py` 的实测封装经验。accio-mcp-cli 能调 Gmail / Twitter / Notion / Square / Apify（含 Instagram/Facebook/TikTok/YouTube/Reddit/1688）/ Google Workspace / GitHub / Composio（Figma/HubSpot/Intercom）等远程 MCP 工具。

#### 3.2.1 本体位置（关键：不在 PATH）

accio-mcp-cli **不是独立可执行命令**，实际是一个 Node 脚本 `accio-mcp.mjs`，藏在 Accio Desktop 安装目录里：

```
默认路径: C:\Users\<USER>\AppData\Local\Programs\Accio\resources\accio-mcp-cli\accio-mcp.mjs
```

定位优先级（高 → 低）：

1. 显式传入路径
2. 环境变量 `ACCIO_MCP_CLI_PATH`
3. 上面默认路径
4. `shutil.which('accio-mcp-cli')`（若用户手动加入了 PATH）

**前提**：Accio Desktop 必须已安装且在运行 —— 否则 .mjs 文件不存在或 gateway 不通。

#### 3.2.2 运行方式

.mjs 必须用 node 跑（Node.js ≥ 18）：

```bash
node "C:\Users\<USER>\AppData\Local\Programs\Accio\resources\accio-mcp-cli\accio-mcp.mjs" <子命令>
```

#### 3.2.3 鉴权（注入 token）

accio-mcp-cli **需要手动 `export ACCIO_GATEWAY_TOKEN`**：

- 默认连本机 gateway（端口 4097）
- 凭证注入到 .mjs 进程

> ⚠️ **与 workctl 的对比**：workctl 走 `gateway-cli.json`（端口可能变、需手动注入 password）；accio-mcp-cli 走内部固定端口 4097 + 凭证。

#### 3.2.4 命令模式

| 命令 | 用途 |
|------|------|
| `toolkit [kw]` | 浏览 toolkit（无 kw 列全部概览，有 kw 列该 toolkit 下所有工具） |
| `search <kw>` | 按关键字全文搜工具（匹配 name/description/toolkit/service） |
| `call <tool> [--json-file path \| --key val ...] [--server <srv>] [--raw]` | 调用工具 |
| `list` | 列出全部工具（150+，慎用，会刷屏） |
| `server list / tools <name> / test <name> / add / remove` | 自定义 MCP server 管理 |

#### 3.2.5 调用流程（toolkit → search → call）

```bash
# 把 <USER> 替换为本机实际用户名
MJS="C:/Users/<USER>/AppData/Local/Programs/Accio/resources/accio-mcp-cli/accio-mcp.mjs"

# 1) 浏览某 toolkit
node "$MJS" toolkit alibaba

# 2) 按关键字搜工具
node "$MJS" search twitter

# 3) 调用工具（参数走 --json-file 避免 Windows 引号地狱）
echo '{"productId":"16XXXXXXXXXXX","queryType":"trunk","componentList":["attr"]}' > /tmp/args.json
node "$MJS" call product_query_information --json-file /tmp/args.json --raw
```

#### 3.2.6 响应解析（套娃 JSON 坑）

accio-mcp-cli 的 MCP 响应是多层套娃，必须逐层解开：

| 层 | 结构 |
|----|------|
| 1（外层 MCP） | `{content: [{type:"text", text:"<payload>"}], isError: false}` |
| 2（text 内容） | 可能是 JSON 字符串，需再 `JSON.parse` 一次 |
| 3（accio 风格） | `{success, errorCode, errorMsg, data}`，其中 `data` 可能仍是 JSON 字符串需再 parse |

要点：

- `isError=true` → 工具调用失败，看 `errorMsg`
- `--raw` 输出原始 MCP JSON；不加 `--raw` 时 `list/search/toolkit` 直接 pretty 文本
- 解析时逐层尝试，最深层无法解析则当字符串返回

#### 3.2.7 已知坑点

| # | 坑点 | 解决 |
|---|------|------|
| 1 | Windows shell 引号地狱（JSON 参数带引号被吞） | 用 `--json-file <path>` 传 JSON，别用 `--json '{...}'` 内联 |
| 2 | accio-mcp-cli 正常退出时偶发 Node UV async 断言警告到 stderr，returncode 非 0 | 只要 stdout 非空就视为成功；仅 stdout 空 且 rc≠0 才报错 |
| 3 | `list` 返回 150+ 工具刷屏 | 用 `toolkit`/`search` 缩小范围 |
| 4 | 找不到 accio-mcp.mjs | 确认 Accio Desktop 已安装启动；或设 `ACCIO_MCP_CLI_PATH` 环境变量；或加入 PATH |
| 5 | 鉴权失败 / 401 | 重启 Accio Desktop（凭证由 Desktop 注入） |
| 6 | Node 未安装或版本低 | 装 Node.js ≥ 18 |

#### 3.2.8 推荐封装

直接调 .mjs 较繁琐（路径定位 / Node / 响应解析 / 引号），建议封装一层 Python 代理屏蔽这些细节，对外暴露：

- `proxy.toolkit("alibaba")` / `proxy.search("twitter")` / `proxy.call(tool, params_dict)`
- 自动处理 .mjs 路径定位、`--json-file` 传参、多层 JSON 解套、退出码容错

参考实现：`accio_mcp_cli_proxy.py`（已存在于本地 `workctl-exports/` 目录）。配套技能 `accio-mcp-cli` 的 SKILL.md 也记录了相同工作流，可对照阅读。

## 四、三条授权文件的关联关系

| # | 文件 | 路径（绝对路径） | 作用 |
|---|------|------|------|
| 1 | 阿里 OAuth 状态 | `C:\Users\<USER>\.accio\accounts\<ID>\connectors\data\alibaba\state.json` | 账号身份凭证（ali_id/login_id/serviceType/status，值动态读取） |
| 2 | 插件 connector 配置 | `C:\Users\<USER>\.accio\accounts\<ID>\plugins\installed\alibaba-com-seller-assistant\connectors\connectors.json` | OAuth 适配声明（mcpServers={}，oauth adapter=mcp-oauth2） |
| 3 | 网关 CLI 凭证 | `C:\Users\<USER>\.accio\accounts\<ID>\.accio\runtime\gateway-cli.json` | 连网关的通行证（url/password 见该文件，值动态读取） |

**关联逻辑**：
- workctl 动态命令 / 阿里 MCP 调用都走网关（url 见 `gateway-cli.json`）
- 网关 Basic Auth 凭证在文件 3
- workctl 用 `ACCIO_GATEWAY_TOKEN` 环境变量取 token（health 实测）
- **OAuth（文件 1 账号身份）+ 网关凭证（文件 3 通行证）配合即可拉实时数据**，文件 2 仅是适配声明

## 五、子账号数据获取与限制

### 5.1 可获取（主账号权限）
- 子账号列表：`workctl icbu member list`（账号数量与人员会变，**动态查询，勿硬编码**）
- 各子账号会话列表：`workctl icbu tm list-conversation --aliId=<子账号aliId>`（返回最近 20 条会话，含 lastMessageTime/unread/买家国家/标签；子账号 aliId 从 member list 结果取）

### 5.2 子账号会话消息体（可获取，必须用对接口）
- ❌ `list-conversation-msg`（按 conversationId 点查）：校验会话 membership，主账号非子账号会话成员，返回 `not in conversation`——此路不通，但**不代表无权读取**
- ✅ `list-conversation-msg-time-range`（按时间范围查）：接受可选参数 `selfAliId`（"查询对象aliId"，integer）。主账号 token 传入子账号 aliId 即可读该子账号会话消息体（限近 30 天）。
- 推荐用法：直接走 `workctl workflow chat-analysis fetch-messages`（内部自动把会话 `sellerAliId` 映射为 `list-conversation-msg-time-range` 的 `selfAliId`，多会话并行、按 3 段时间窗拉取）；详见 alibaba-chat-and-analysis 子代理 `msg-history.md`
- 结论：首响时长/回复时长/回复质量**可量化**——消息体可读且带 `senderAliId`+`timestamp`，可按子账号拆分计算

### 5.3 诊断聚合（受限）
- `list-seller-acct-dim-diag-data`：仅主账号 aliId 可用，子账号 aliId 无权访问账号维度诊断（官方 schema 明示）

## 六、常用数据获取命令清单（可直接复制）

```bash
# ===== 第一步：注入网关凭证（每次新开 shell 都要做） =====
# 把 <USER> 和 <ID> 替换为本机实际值
export ACCIO_GATEWAY_TOKEN=$(node -p "JSON.parse(require('fs').readFileSync('C:/Users/<USER>/.accio/accounts/<ID>/.accio/runtime/gateway-cli.json')).password")

# ===== 第二步：验证连通性 =====
workctl health

# ===== 店铺运营数据 =====
workctl icbu crm store-diagnose-brief                              # 店铺诊断（近5周趋势）
workctl icbu advisor data-advisor-shop-summary                     # 经营大盘 + 同行对比
workctl icbu advisor data-advisor-shop-flow-profile                # 流量地域画像
workctl icbu advisor data-advisor-shop-channel                    # 渠道流量
workctl icbu advisor data-advisor-shop-region                     # 地域流量
workctl icbu advisor data-advisor-shop-product                    # 商品效果（默认20条）
workctl icbu advisor data-advisor-visitor-detail                  # 访客明细

# ===== 客服与沟通 =====
workctl icbu tm list-conversation --aliId=<aliId>                 # 某账号会话列表
workctl icbu tm list-seller-acct-dim-diag-data --buyerType=1 --dateType=2 --queryDate=YYYYMMDD  # 主账号客服诊断
workctl icbu tm list-seller-shop-dim-diag-data --buyerType=1 --dateType=2 --queryDate=YYYYMMDD  # 店铺客服诊断

# ===== 商品与广告 =====
workctl icbu product get-score --productId=<id>                   # 商品质量分
workctl icbu ads list --entityType=diagnosis                      # 广告诊断（HATEOAS 新接口）

# ===== 交易与风控 =====
workctl icbu trade list-shop-violation-result                     # 店铺违规
workctl icbu trade shop-risk-diagnosis                            # 店铺风险扫描
workctl icbu trade list-trade-list-mcp                           # 交易合同列表

# ===== RFQ 商机 =====
workctl icbu rfq rfq-aw-search --keywords="forklift parts"        # RFQ 搜索

# ===== 子账号管理 =====
workctl icbu member list                                          # 子账号列表（动态查询）
```

## 七、故障排查速查

| 现象 | 原因 | 解决 |
|------|------|------|
| `desktop_not_attached` | 未注入 ACCIO_GATEWAY_TOKEN | 执行 2.2 的 export 命令 |
| health pass 但 schema 返回 401 | 用了假 token 或 token 过期 | 重新从 gateway-cli.json 读真实 password |
| `missing required parameter` | 漏传必填参数 | 看错误信息的 next_action，补齐参数 |
| 命令报 `unknown shorthand flag` | 用了 `-f` 等子命令不支持的 flag | 去掉 flag，用默认输出 |
| 输出文件创建失败 | `-o` 路径解析问题 | 改用 `> file` 重定向 |
| 数据为 null | T+2 未生成 或 小店无数据 | 换更早的日期重试 |

## 八、安全注意事项

1. `gateway-cli.json` 含 password 密钥，**勿明文外传**，本 skill 仅记录机制不记明文
2. 客户档案数据仅存储在本地，不外传
3. 不在搜索查询中暴露 TOOMOTOO 的内部数据（成本价、客户列表）
4. 网关重启后凭证失效，需重新读取 gateway-cli.json
5. 涉及客户数据的查询与归档带"负责销售"过滤，不跨销售共享客户信息

## 九、配套技能生态（skills/ 目录，动态发现）

本账号 `skills/` 目录下并存一批配套技能，与上面 workctl 拉到的国际站数据互补：**workctl 负责"取数"，配套技能负责"分析、创作、发布"**。

> ⚠️ **核心原则：技能清单不固化在本 skill 里。** `skills\` 目录会随时增删技能，本节**不维护静态清单**。每次需要调用配套技能前，必须先按 9.2 的流程**实时枚举目录、读取各技能的 `description`**，再挑选匹配项调用。凭记忆或历史清单直接猜技能名 = 违规。

### 9.0 协同范式

```
workctl icbu ...（取数）         ──┐
                                 ├── 配套技能（分析/创作/发布）──→  产物
外部搜索 / 社媒 / MCP（取数）     ──┤
reference/ 方法文档（学方法，见第十节）──┘
```

典型链路（技能仅按职能描述，实际名字以动态发现结果为准）：

| 取数 | 配套技能职能 | 产物 |
|------|---------|------|
| `crm store-diagnose-brief`（店铺诊断） | 竞品/店铺深度分析 | 竞店对标报告 |
| `advisor data-advisor-shop-product`（商品效果） | 评论挖掘 / 客户之声分析 | 选品/迭代建议 |
| `tm list-conversation`（客户会话） | 谈判策略 / 跟进话术（话术方法先查 reference/，见第十节） | 跟进策略 + 话术 |
| `rfq rfq-aw-search`（RFQ 搜索） | 公司背调 / 人物背调 | 买家背调 + 跟进策略 |

### 9.1 技能目录（唯一事实源）

| 项 | 路径 / 说明 |
|----|------------|
| 技能根目录 | `C:\Users\<USER>\.accio\accounts\<ID>\skills\`（每个子目录 = 一个技能，目录名即技能名） |
| 辅助索引 | `C:\Users\<USER>\.accio\accounts\<ID>\skills\skills_config.json`（若存在可作参考；**以目录实际内容为准**） |
| 单技能元数据 | `skills\<技能名>\SKILL.md` 的 frontmatter：`name` + `description`（触发匹配的主依据） |
| 本 skill | `skills\accio-work-output\`（priority / entry-point，每次会话启动时首先触发） |

### 9.2 动态发现与调用流程（必须按序执行）

**第 1 步 — 枚举当前实际安装的技能：**

```bash
# 把 <USER> 和 <ID> 替换为本机实际值；列出 skills/ 下所有技能目录
ls "C:/Users/<USER>/.accio/accounts/<ID>/skills/"
```

**第 2 步 — 批量读取各技能的 name + description**（一条命令拿到全量触发依据；已处理多行折叠 description）：

```bash
node -e "
const fs=require('fs'),path=require('path');
const root='C:/Users/<USER>/.accio/accounts/<ID>/skills';
for(const d of fs.readdirSync(root)){
  const p=path.join(root,d,'SKILL.md');
  if(!fs.existsSync(p)) continue;
  const fm=(fs.readFileSync(p,'utf8').match(/^---\r?\n([\s\S]*?)\r?\n---/)||[])[1];
  if(!fm){console.log(d+' :: (no frontmatter)');continue;}
  let name=d,desc='',cur='';
  for(const l of fm.split(/\r?\n/)){
    const kv=l.match(/^(\w+):\s*(.*)$/);
    if(kv){cur=kv[1];if(cur==='name')name=kv[2].trim();if(cur==='description')desc=kv[2].replace(/^>-?\s*/,'').trim();}
    else if(cur==='description'&&/^\s+\S/.test(l))desc+=' '+l.trim();
  }
  console.log(name+' :: '+desc.trim().slice(0,300));
}"
```

**第 3 步 — 匹配与调用：** 把当前任务需求与第 2 步输出的各技能 `description` 比对，选最匹配的 1–2 个；命中后再读该技能 `SKILL.md` 全文按其流程执行。

### 9.3 使用注意

1. **禁止凭记忆/历史清单猜技能名**——技能随时增删，先跑 9.2 第 1–2 步拿到当前真实清单，再决定调用谁。
2. 技能的 `description` 是触发匹配的主依据；逐个读 `SKILL.md` 全文成本高，只在命中后读。
3. 动态发现后仍无匹配技能时，若目录中存在 `skill-finder`，用它按"内置目录 → skills.sh → web → ClawHub"分层搜索安装新技能；从外部源安装前先过 `skill-vetter`（若存在）安全审查。
4. 各技能鉴权状态会变：workctl 看 `gateway-cli.json`、MCP 看 Connector UI —— **不要缓存旧 token**。

## 十、参考文档库（reference/，方法学习）

本 skill 自带 `reference/` 目录，存放业务方法类参考文档（销售流程、谈判话术、运营 SOP 等）。它与第九节的配套技能互补：**配套技能负责"怎么做"（工具与流程），reference/ 负责"怎么谈、怎么说"（方法论与话术）**。遇到匹配场景时，先到 reference/ 取方法学习，再结合 workctl 实时数据与动态发现的配套技能，给用户更完备的解答。

> ⚠️ **与第九节同一原则：文档清单不固化在本 skill 里。** `reference/` 会随时增删文档，本节**不维护静态清单**。每次需要查方法前，必须先按 10.2 流程**实时枚举目录、读各文档开头摘要块**，再决定读哪份。凭记忆直接猜文档名 = 违规。

### 10.1 目录与匹配依据

| 项 | 路径 / 说明 |
|----|------------|
| 参考文档根目录 | `<本 skill 所在目录>\reference\`（与本 SKILL.md **同级**；每个 `.md` = 一份方法文档。skill 安装/移动到任何位置，`reference/` 都跟随本 skill 目录，不写死绝对路径） |
| 匹配依据 | 每份文档**开头的摘要块**（`>` 引用，写明覆盖范围与适用场景）——新增文档必须带此摘要块 |
| 当前收录（示例，以目录实际内容为准） | 《业务谈判-销售全流程》：线索清洗与背调、首回三要素、FABE 多次沟通、报价/降价/催单、已读不回8连击、客户分层与区域化谈判、素材管理与团队赋能；《平台运营-店铺与流量管理》：店铺装修与线上表达、商品发布与内容运营、价格布局与分析、流量推广（P4P/自营销）、询盘分配——**非销售分内工作，运营/老板专用** |

### 10.2 查阅流程（遇到匹配场景时按序执行）

**第 1 步 — 枚举目录：**

```bash
# <本 skill 所在目录> = 本 SKILL.md 被加载的目录（随账号/机器而变，运行时取实际加载路径，勿写死）
ls "<本 skill 所在目录>/reference/"
```

**第 2 步 — 读摘要匹配场景：** 逐个读候选文档开头 20–40 行的摘要块，与当前任务场景比对，选最匹配的 1 份（不匹配则不强行引用）。

**第 3 步 — 读全文取方法：** 命中后读该文档相关章节全文，抽取适用的模型、原则、话术。

**第 4 步 — 结合输出（不照抄模板）：** 把取到的方法与以下两者结合，按当前客户/产品上下文定制输出：
- workctl 实时数据（客户会话、RFQ、店铺诊断等，见第二、六节）
- 第九节动态发现的配套技能（背调、话术生成、素材制作等）

### 10.3 典型触发场景（对照当前收录文档）

| 用户场景 | 应查的方法 |
|---------|-----------|
| 新客户询盘怎么回 / 首次触达 | 首回三要素、首回错误示范 |
| 写产品介绍 / 跟客户讲产品 | FABE 模型（先想客群再写 B）、说人话四原则 |
| 客户嫌贵 / 要降价 | 报价十六字方针、降价台阶（策略性/互利型） |
| 客户已读不回 / 要跟进话术 | 已读不回8连击（专业版/温情版）、表情包策略、TM 群发激活 |
| 催款 / 逼单 | 迂回催单、给出条件、营造紧迫、情绪价值 |
| 特定区域客户怎么谈（印度/中东/非洲/欧美…） | 客户分层（L1-L4）、区域化谈判、佣金谈判 |
| 线索值不值得跟 / 客户背调 | 线索清洗标准、背调信息清单与方法 |
| 销售团队培训 / 话术素材沉淀 | 素材双阵地、新员工卡点培养、投屏检查法 |
| 店铺装修 / 顶展金品 / 实力标签 / P4P / 价格布局 / 商品发布优化 | 《平台运营-店铺与流量管理》（运营/老板专用，非销售分内工作） |

> 上表仅对应当前已收录文档；`reference/` 新增文档后，以其摘要块为准重新匹配。

### 10.4 维护规则

1. 新增方法文档直接放进 `reference/`，无需改本 skill（动态枚举）。
2. 每份文档开头必须有摘要块（`>` 引用），写明覆盖范围与适用场景——这是 10.2 第 2 步匹配的唯一依据。
3. reference/ 只放"方法"，不放"数据"：客户名单、成交价、账号身份等运行时数据一律走 workctl 实时拉取或本地受控存储，不沉淀进参考文档。

---

*本 skill 只记录稳定的机制与方法。账号身份、workctl 版本、网关端口、子账号人员等运行时值请按各节指引动态读取；配套技能不在本 skill 固化清单，一律按第九节动态发现流程实时枚举后再调用；业务方法文档放 `reference/` 并按第十节动态枚举后按需查阅。如有 workctl 版本升级或网关机制变更，请同步更新本 skill。最近一次验证：2026-07-28。*
