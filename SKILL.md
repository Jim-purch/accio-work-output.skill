---
name: accio-work-output
version: 0.2.0
github_repo: https://github.com/Jim-purch/accio-work-output.skill.git
priority: primary
domain: forklift-parts-foreign-trade-sales
description: >-
  PRIORITY entry-point skill for TOOMOTOO forklift-parts foreign-trade sales
  work inside this .accio account — invoke FIRST at the start
  of EVERY work session, before any other skill, even when the user just says
  hi / opens the workspace / asks any forklift-parts sales question. Two
  bootstrap jobs on session start: (1) self-check this skill's version
  against GitHub (https://github.com/Jim-purch/accio-work-output.skill.git)
  and notify the user of updates; (2) expose the Alibaba.com (ICBU) workctl
  auth layer — the gateway token lives in
  /Users/{username}/.accio/accounts/{accountId}/.accio/runtime/gateway-cli.json
  (its password field = ACCIO_GATEWAY_TOKEN; both placeholders resolve at runtime); trigger when the user needs Ali
  ICBU data access, gateway token injection, or hits workctl
  desktop_not_attached errors. After bootstrap, hand off to companion skills
  discovered dynamically in the account skills/ directory (never a fixed
  list) and consult this skill's reference/ library on demand (enumerate it,
  match by each doc's header summary). Generated files go to the agent's
  default working directory.
---

# Accio Work Output Router

> **Priority / entry-point skill for TOOMOTOO forklift-parts foreign-trade
> sales.** Boots first at every session start in this `.accio` account,
> self-checks for GitHub updates, and exposes the ICBU/workctl data layer.
> 本 skill 只保留会话常驻必需的最小信息；完整鉴权、命令清单与流程细节一律放
> `reference/` 按需调研（见文末索引表）。

| Field | Value |
|-------|-------|
| Version | frontmatter `version:`（唯一事实源，加载时已在内存，勿重读磁盘） |
| GitHub repo | https://github.com/Jim-purch/accio-work-output.skill.git |
| Domain | TOOMOTOO (Tianjin) International Trading — 叉车配件，ICBU cgs 卖家 |
| Last verified | 2026-08-21 |

## When to trigger

- **本账号新会话开始**（首条消息、任何叉车配件外贸销售问题、甚至只是打招呼）→ 先跑下面两个 bootstrap，再做别的。
- 用户需要阿里国际站数据、网关 token 注入、或遇到 workctl `desktop_not_attached` 报错 → 直接看「核心凭证」一节。

## Bootstrap 1 — 会话启动自检（GitHub 版本检查）

静默执行，只向用户报一行中文结论：

```bash
REMOTE_VER=$(curl -fsSL --max-time 8 \
  "https://api.github.com/repos/Jim-purch/accio-work-output.skill/releases/latest" \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{try{const t=JSON.parse(s).tag_name||'';process.stdout.write(t.replace(/^v/,''))}catch(e){process.exit(1)}})" \
  | head -1)
```

- `LOCAL_VER` = 本 skill 已加载 frontmatter 的 `version:`（内存中已知，勿重读磁盘）。
- 版本一致 → 「✅ 已自检：本 skill 为最新版本 (v{LOCAL_VER})，与 GitHub 一致。」；远端更新 → 一行提示并等用户确认，**绝不自动拉取**；网络失败 → 静默跳过，用本地版继续。
- 每会话只跑一次；绝不让自检阻塞会话（>8s 即放弃）。完整上报矩阵与规则见 `reference/workctl-icbu-鉴权与数据手册.md` §一。

## Bootstrap 2 — 核心凭证：gateway-cli.json（本 skill 最重要的一条路径）

> **`/Users/{username}/.accio/accounts/{accountId}/.accio/runtime/gateway-cli.json`**
>
> `{username}` = 本机用户名（`whoami`）；`{accountId}` = 账号 ID（`ls ~/.accio/accounts/`，通常仅一个目录，取目录名——实际 ID 不写入本 skill，避免随公开仓库泄露）。该文件是连 Accio 网关的通行证：`url` 字段 = 网关地址，`password` 字段 = `ACCIO_GATEWAY_TOKEN` 的值。**网关每次重启都会重写此文件（password/端口可能变），必须每次实时读取，禁止缓存旧值、禁止把字段值硬编码进任何 skill 或产物。**

主 Agent 沙箱默认没有 `ACCIO_GATEWAY_TOKEN`，未注入时 workctl 动态命令一律报 `desktop_not_attached`。每次新开 shell 先注入再验证：

```bash
ACC_ID=$(ls "$HOME/.accio/accounts/" | head -1)   # {accountId} 运行时发现（通常仅一个目录）
export ACCIO_GATEWAY_TOKEN=$(node -p "JSON.parse(require('fs').readFileSync('$HOME/.accio/accounts/$ACC_ID/.accio/runtime/gateway-cli.json')).password")
workctl health   # 期望 backend=pass, version=pass
```

注入后 `workctl icbu <子命令>` 全部可用：`advisor / crm / product / tm / rfq / trade / ads / logistics / member / cco / storefront`（店铺诊断、经营大盘、会话、RFQ、订单、广告、物流、子账号等实时数据）。

## Bootstrap 之后 — 动态交接（清单不固化）

1. **配套技能**（账号 `skills/` 目录，负责分析/创作/发布）：调用前必须实时枚举目录、读各技能 `description` 匹配后再调用，禁止凭记忆猜技能名。流程与命令见 `reference/配套技能与文档发现流程.md`。
2. **方法文档**（本 skill 的 `reference/`，负责"怎么谈、怎么说"）：同样先枚举、读各文档开头摘要块匹配场景后再读全文，结合实时数据输出，不照抄模板。

## reference/ 索引（按需调研；以目录实际内容为准，新文档按摘要块匹配）

| 文档 | 何时查阅 |
|------|---------|
| `workctl-icbu-鉴权与数据手册.md` | 拉国际站实时数据、workctl / accio-mcp 鉴权机制与命令清单、gateway-cli.json 字段、子账号数据限制、`desktop_not_attached` / 401 / T+2 等故障排查、自检完整流程 |
| `配套技能与文档发现流程.md` | 调用配套技能前、查方法文档前的动态发现流程与命令 |
| `业务谈判-销售全流程.md` | 首回三要素、FABE、报价/降价/催单、已读不回8连击、客户分层与区域化谈判 |
| `平台运营-店铺与流量管理.md` | 店铺装修、商品发布、价格布局、P4P 推广（运营/老板专用，非销售分内工作） |

## 安全与输出

1. `gateway-cli.json` 含 password 密钥，勿明文外传、勿写入任何产物；本 skill 只记录机制不记明文。
2. 客户数据仅存本地不外传；不在搜索查询中暴露内部数据（成本价、客户列表）。
3. 生成文件写入 agent 默认工作目录，本 skill 不强制特殊输出目录。

---

*运行时值（账号身份、workctl 版本、网关端口、子账号人员）一律动态读取；技能与文档清单不固化，一律动态枚举后按需查阅。最近一次验证：2026-08-21（macOS，文件路径实测核对）。*
