---
name: accio-work-output
version: 0.1.0
github_repo: https://github.com/Jim-purch/accio-work-output.skill.git
priority: primary
domain: forklift-parts-foreign-trade-sales
description: >-
  PRIORITY entry-point skill for TOOMOTOO forklift-parts foreign-trade sales
  work inside this .accio account — invoke FIRST at the start of EVERY work
  session, before any other skill, even when the user just says hi / opens the
  workspace / asks any forklift-parts sales question. Three bootstrap jobs it
  runs on session start: (1) self-check this skill's version against GitHub
  (https://github.com/Jim-purch/accio-work-output.skill.git) and notify the
  user of updates; (2) route ALL generated deliverables into a `Work/`
  subdirectory of the account root — never scatter files at the top level;
  trigger whenever the user asks to create/save/export/generate/write any file
  or whenever a deliverable is about to be produced, even if the user did not
  say "Work"; (3) provide the Alibaba.com (ICBU) workctl CLI & accio-mcp
  authentication workflow for pulling real-time store data — trigger when the
  user needs Ali ICBU data access, gateway token injection, or hits workctl
  desktop_not_attached errors. After bootstrap, hand off to specialized sales
  skills (sales-negotiator, customer-voice-analyzer, company-research, etc.)
  while still enforcing the Work/ routing rule for every deliverable they
  produce.
---

# Accio Work Output Router (v0.1.0)

> **Priority / entry-point skill for TOOMOTOO forklift-parts foreign-trade
> sales.** This skill boots first at the start of every work session in this
> `.accio` account, self-checks for GitHub updates, then enforces the `Work/`
> routing rule and exposes the ICBU/workctl data layer that all downstream
> sales skills depend on.
>
> | Field | Value |
> |-------|-------|
> | Version | `0.1.0` |
> | GitHub repo | https://github.com/Jim-purch/accio-work-output.skill.git |
> | Priority | `primary` — invoke FIRST on every session start |
> | Domain | TOOMOTOO forklift-parts foreign-trade sales |
> | Last verified | 2026-07-27 |

## What this skill is for

You are operating inside a `.accio` account workspace (the directory under
`.accio/accounts/<accountId>`) bound to TOOMOTOO (Tianjin) International
Trading — an Alibaba.com (ICBU) cgs seller of forklift parts. This account
root is shared by many subsystems (`skills/`, `conversations/`, `agents/`,
`.workbuddy/`, memory files, etc.) and gets noisy fast if every generated
file is dropped at the top level.

This skill is the **priority entry point** for all forklift-parts
foreign-trade sales work here. It does three things, in order, on every
session start:

1. **Self-check GitHub updates** (see §"会话启动自检" below) — compare local
   `version` against the published repo and notify the user.
2. **Route every deliverable into `Work/`** — the single home for produced
   files at the account root.
3. **Expose the ICBU/workctl data layer** — so downstream sales skills can
   pull real-time store data (store diagnose, conversations, RFQ, products,
   ads, trade, logistics) without re-deriving the auth flow.

The rule: **any file you generate as a deliverable goes into `Work/`**, a
folder at the account root. That keeps the account root readable and makes it
trivial for the user to find, back up, or delete everything you produced.

This is not a feature the user has to remember to ask for — it is the default
behavior whenever you are producing work inside this account.

## When to trigger

This is the **priority / entry-point skill** for this account. Trigger it
FIRST, before any other skill, whenever any of these apply:

- **A new work session starts in this account** (the user opens the workspace,
  sends the first message of a session, asks anything forklift-parts-sales
  related, or just says hi). On this first trigger, run the **会话启动自检**
  flow below before doing anything else.
- The user asks to create / write / save / export / generate a file.
- You are about to produce a deliverable (report, docx, xlsx, pptx, pdf, html,
  script, dataset, image, video, archive, code output, etc.).
- You are about to call `Write`, `present_files`, or run a command that writes
  a user-facing artifact.
- The user needs Ali ICBU data access, gateway token injection, or hits a
  workctl `desktop_not_attached` error.

Do NOT trigger (and do not redirect) for system/runtime files that the platform
manages itself: `.workbuddy/` memory and logs, `automations/`, `conversations/`,
`agents/`, `skills/`, connector state, and anything the platform writes
automatically. Those stay where they belong.

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
| Local skill dir | `C:\Users\<USER>\.accio\accounts\<ID>\skills\accio-work-output` |
| GitHub repo URL | `https://github.com/Jim-purch/accio-work-output.skill.git` |
| Raw SKILL.md URL | `https://raw.githubusercontent.com/Jim-purch/accio-work-output.skill/main/SKILL.md` |
| Local version | the `version:` field in the local `SKILL.md` frontmatter (currently `0.1.0`) |

### Self-check procedure

Run these steps once per session, in order. Stop at the first step that
succeeds and skip the rest.

**Step 1 — git fetch (preferred, works when the local dir is a git clone).**

```bash
# Inside the local skill dir; <USER> and <ID> substituted at runtime.
SKILL_DIR="C:/Users/<USER>/.accio/accounts/<ID>/skills/accio-work-output"
if [ -d "$SKILL_DIR/.git" ]; then
  git -C "$SKILL_DIR" fetch --quiet origin main 2>/dev/null
  LOCAL=$(git -C "$SKILL_DIR" rev-parse HEAD 2>/dev/null)
  REMOTE=$(git -C "$SKILL_DIR" rev-parse origin/main 2>/dev/null)
  if [ -n "$LOCAL" ] && [ -n "$REMOTE" ]; then
    if [ "$LOCAL" = "$REMOTE" ]; then
      echo "UP_TO_DATE"
    elif git -C "$SKILL_DIR" merge-base --is-ancestor "$LOCAL" "$REMOTE" 2>/dev/null; then
      echo "BEHIND"
    else
      echo "DIVERGED"
    fi
  else
    echo "GIT_NO_REMOTE"
  fi
else
  echo "NO_GIT"
fi
```

**Step 2 — raw SKILL.md version compare (fallback when no local `.git`).**
Fetch only the remote `SKILL.md` frontmatter and compare the `version:` field
against the local one. Cheaper than a full clone; use when the skill folder
was installed by copy rather than `git clone`.

```bash
REMOTE_VER=$(curl -fsSL --max-time 8 \
  "https://raw.githubusercontent.com/Jim-purch/accio-work-output.skill/main/SKILL.md" \
  | sed -n 's/^version:[[:space:]]*//p' | head -1)
LOCAL_VER=$(sed -n 's/^version:[[:space:]]*//p' "$SKILL_DIR/SKILL.md" | head -1)
# Compare semver (major.minor.patch). If REMOTE_VER > LOCAL_VER → BEHIND.
```

**Step 3 — offline / network blocked.** If both steps above fail (no network,
GitHub unreachable, sandbox blocks the request), skip silently and proceed
with the local copy. Do not block the user's work.

### How to report to the user

After the check, surface the result to the user in **one short line** (Chinese,
since this account's primary user is a TOOMOTOO sales rep) plus an action
prompt only when an update exists:

| Result | What to tell the user |
|--------|----------------------|
| `UP_TO_DATE` | "✅ 已自检：本 skill 为最新版本 (v0.1.0)，与 GitHub 一致。" |
| `BEHIND` | "🔄 检测到 GitHub 有新版本 (本地 v0.1.0 → 远端 vX.Y.Z)。是否拉取更新？(`git -C <skill dir> pull origin main`)" — wait for user confirmation before pulling; never auto-overwrite a skill the user is actively editing. |
| `DIVERGED` | "⚠️ 本地 skill 与 GitHub 出现分叉（本地有未推送的修改）。如需同步请先 commit 本地改动或手动对比差异。" |
| `NO_GIT` / `GIT_NO_REMOTE` | Run Step 2. If Step 2 also fails, fall through to offline. |
| offline (Step 3) | "ℹ️ 未能连接 GitHub 检查更新（网络受限），本次会话使用本地 v0.1.0 继续。" |

### Rules

1. **Never auto-update without user confirmation.** A `BEHIND` result only
   *prompts*; it does not pull. The user may have local edits in flight.
2. **Never block the session on the self-check.** If the check takes more than
   ~8 s or fails, proceed with the local copy and mention the skip in one line.
3. **Run once per session, not per message.** Cache the result in memory for
   the rest of the session; don't re-fetch on every user turn.
4. **Don't leak sensitive paths or tokens** in the report line — show only the
   version numbers and the one-line `git pull` hint.
5. After reporting, immediately continue with the rest of the bootstrap
   (enforce `Work/` routing, expose the ICBU/workctl layer) and hand off to
   whatever the user actually asked for.

## Path convention (used throughout this skill)

All paths in this skill are written as **absolute paths** with two placeholders
that must be substituted to the actual values of the machine running the agent:

| Placeholder | Meaning | How to read at runtime |
|-------------|---------|------------------------|
| `<USER>` | Windows username of the account owner | `echo "$USERNAME"` in bash, or `%USERNAME%` in cmd |
| `<ID>` | Accio account ID | basename of the current working directory |

**Account root** and **Work dir** templates:

```
Account root:  C:\Users\<USER>\.accio\accounts\<ID>\
Work dir:      C:\Users\<USER>\.accio\accounts\<ID>\Work\
```

In bash / `node` commands, forward slashes work on Windows and avoid
backslash-escaping issues, so commands below use `C:/Users/<USER>/...` form.
Substitute `<USER>` and `<ID>` before running.

## How to route files

All deliverables go into the `Work\` folder at the account root. Resolve the
absolute target path directly (no helper script).

```
Work dir:  C:\Users\<USER>\.accio\accounts\<ID>\Work\
```

### Workflow

1. Compute the deliverable path: append your intended sub-path under `Work\`.
   Example: `reports\2026-07-27-summary.md` →
   `C:\Users\<USER>\.accio\accounts\<ID>\Work\reports\2026-07-27-summary.md`.
2. Ensure parent folders exist — the `Write` tool creates them automatically;
   if writing via a shell command, `mkdir -p` the parent first.
3. Write the file to that resolved absolute path (via the `Write` tool or a
   command).
4. Present it to the user normally with `present_files` — `present_files`
   accepts any absolute path, so the fact that it lives in `Work\` is
   transparent to the user.
5. If you create several related files, group them under a descriptive
   subfolder inside `Work\` (e.g. `Work\reports\`, `Work\scripts\`,
   `Work\exports\2026-07-27\`). Prefer a dated task folder for multi-file
   deliverables.

### Examples

- User: "帮我写一份竞品分析报告" → write to
  `C:\Users\<USER>\.accio\accounts\<ID>\Work\reports\competitor-analysis.md`, present it.
- User: "把这批数据导出成 CSV" → write to
  `C:\Users\<USER>\.accio\accounts\<ID>\Work\exports\users.csv`.
- User: "写个脚本批量处理日志" → write to
  `C:\Users\<USER>\.accio\accounts\<ID>\Work\scripts\log_tool.py`; note the
  location in your reply.

## Notes

- The `Work\` folder is for **user-facing deliverables only**. Never move or
  redirect the platform's own files into it.
- If you are outside `C:\Users\<USER>\.accio\accounts\<ID>\` for some reason,
  fall back to `.\Work` relative to the current directory so behavior stays
  predictable.
- Keep file names clear and, for dated material, prefixed with `YYYY-MM-DD` so
  the `Work\` folder stays sortable.

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

#### 3.2.3 鉴权（自动，无需手动注入 token）

与 workctl 不同，accio-mcp-cli **不需要手动 `export ACCIO_GATEWAY_TOKEN`**：

- 默认连本机 Accio Desktop 的 gateway（端口 4097）
- 凭证由 Accio Desktop 自动注入到 .mjs 进程
- 只要 Desktop 在跑，鉴权就自动完成

> ⚠️ **与 workctl 的对比**：workctl 走 `gateway-cli.json`（端口可能变、需手动注入 password）；accio-mcp-cli 走 Desktop 内部固定端口 4097 + 自动凭证。两套相互独立，凭证来源不同。

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

### 5.2 不可获取（权限限制）
- 子账号会话消息体：`list-conversation-msg` 主账号无权读（返回 `not in conversation`，需子账号登录态）
- 诊断聚合：`list-seller-*-dim-diag-data` 返回 null（T+2 未生成 / 小店无数据）
- 结论：无法量化单销售员的首响/回复时长/回复质量，仅能从会话级元数据推断

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

## 九、配套技能生态（skills/ 目录）

本账号 `skills/` 目录下并存一批配套技能，覆盖"平台工具 / 销售与客户 / 营销文案 / SEO 流量 / 内容社媒 / Skill 工程"几大类。它们与上面 workctl 拉到的国际站数据互补：**workctl 负责"取数"，配套技能负责"分析、创作、发布"，所有产物最后都按本 skill 第一部分路由到 `Work/`**。

### 9.0 协同范式

```
workctl icbu ...（取数）  ──┐
                          ├── 配套技能（分析/创作/发布）──→  Work/ 产物
外部搜索 / 社媒 / MCP（取数）──┘
```

典型链路：

| 取数 | 配套技能 | 产物 |
|------|---------|------|
| `crm store-diagnose-brief`（店铺诊断） | `competitor-deep-analysis` | `Work\reports\` 竞店对标报告 |
| `advisor data-advisor-shop-product`（商品效果） | `customer-voice-analyzer` | 选品/迭代建议 |
| `tm list-conversation`（客户会话） |
| `rfq rfq-aw-search`（RFQ 搜索） | `company-research` / `people-research` | 买家背调 + 跟进策略 |

### 9.1 平台工具与协作

| 技能 | 作用 | 备注 |
|------|------|------|
| `accio-work-output`（本 skill，**priority / entry-point**） | 会话启动自检 GitHub 更新；所有产物路由到 `Work/`；记录 workctl/网关鉴权机制 | **每次会话启动时首先触发**；产出任何文件时再次触发 |
| `accio-mcp-cli` | 用 CLI 发现/搜索/调用 MCP 工具（Gmail、Twitter、Notion、Square、Apify 等），鉴权自动 | 工作流：`toolkit → search → call`，避免 `list`（150+ 工具刷屏） |
| `lark-tools` | 用 `lark-cli` 操作飞书（文档/表格/Base/日历/任务/邮件/会议/IM） | 鉴权走 Connector UI（Settings → Connectors → Lark）；先 `lark-cli auth status` 再业务命令 |

### 9.2 销售与客户（与 workctl 客服 / RFQ 数据强相关）

| 技能 | 作用 |
|------|------|
| `sales-negotiator` | B2B 谈判策略：锚定、BATNA、定价、合同条款、多方谈判 |
| `customer-voice-analyzer` | 从评论里挖出画像/场景/优缺点/未满足需求/购买动机 6 维 |
| `company-research` | 用 Exa 调研公司：竞品、新闻、财务、LinkedIn |
| `people-research` | 用 Exa 找人：LinkedIn、专家、团队成员、公开 bio |
| `competitor-deep-analysis` | 多层情报 + 评论挖掘，找市场空白与差异化优势 |

### 9.3 营销与文案

| 技能 | 作用 |
|------|------|
| `copywriting` | 落地页/定价/产品/关于页等营销文案写作与改写 |
| `product-description-generator` | 生成 Amazon/Shopify/eBay/Etsy 的 SEO 产品描述 |
| `marketing-psychology` | 70+ 心智模型用于营销说服 |
| `marketing-ideas` | 140+ 营销打法（按类目） |
| `launch-strategy` | 分阶段发布、渠道、Product Hunt/早鸟/候补 |
| `cart-abandonment-recovery` | 弃购挽回邮件 + SMS 序列 |
| `ab-test-setup` | A/B 测试设计与假设 |

### 9.4 SEO 与流量

| 技能 | 作用 |
|------|------|
| `seo-keyword-research` | 高价值关键词 + 意图分类 + 主题簇 + GEO |
| `seo-page-audit` | 单页 SEO 体检，0-100 评分 + 优先级建议 |
| `seo-competitor-analysis` | 深度 SEO 竞品：关键词/外链/内容策略 |
| `ecommerce-seo-optimizer` | 电商页 meta 框架、JSON-LD、抓取管理 |
| `programmatic-seo-strategist` | 程序化 SEO：模板化长尾页 |
| `serp-ranking-analyzer` | SERP 深析：排名因子、意图、特征机会 |
| `etsy-seo-optimizer` | Etsy listing SEO（eRank 数据） |

### 9.5 内容与社媒

| 技能 | 作用 |
|------|------|
| `social-media-publisher` | 发 Instagram / X（图文、话题、@、多平台） |
| `image-prompt-guide` | AI 图片生成/编辑 prompt 与工具路由（含电商图集、白底、水印清理、HD 放大等） |
| `remotion` | 用 React 做视频 |

### 9.6 Skill 工程（管理 skills/ 本身）

| 技能 | 作用 |
|------|------|
| `skill-creator` | 创建/修改/评测 skill，优化 description 触发率 |
| `skill-finder` | 分层发现并安装 skill（内置目录 → skills.sh → web → ClawHub） |
| `skill-vetter` | 安装前安全审查（红旗、权限、可疑模式） |
| `self-improvement` | 把错误/纠正/反思写进每日日记，持续改进 |
| `plugin-create` | 脚手架/打包 plugin ZIP |

### 9.7 使用注意

1. 配套技能的 `description` 是触发主依据；不确定某能力是否被覆盖时，先用 `skill-finder` 搜，不要凭记忆猜。
2. 从外部源（ClawHub / GitHub）装新 skill 前，先用 `skill-vetter` 审一遍。
3. 各技能鉴权状态会变：workctl 看 `gateway-cli.json`、飞书看 `lark-cli auth status`、MCP 看 Connector UI —— **不要缓存旧 token**。
4. 任何配套技能产出的文件，都回到本 skill 的 `Work/` 路由规则（见第一部分）。
5. 技能清单会随 `C:\Users\<USER>\.accio\accounts\<ID>\skills\` 目录增减而变化，使用前用 `ls C:\Users\<USER>\.accio\accounts\<ID>\skills\` 或读 `C:\Users\<USER>\.accio\accounts\<ID>\skills\skills_config.json` 确认当前实际可用项，勿照搬本节清单。

---

*本 skill 只记录稳定的机制与方法。账号身份、workctl 版本、网关端口、子账号人员等运行时值请按各节指引动态读取；配套技能清单见第九节。如有 workctl 版本升级、网关机制变更或 skills/ 目录新增/移除技能，请同步更新本 skill。最近一次验证：2026-07-27。*
