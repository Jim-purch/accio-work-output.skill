# workctl & accio-mcp 鉴权与数据手册

> 本文档是 accio-work-output skill 的技术参考手册，承接完整鉴权与取数知识：会话启动自检（GitHub 版本检查）完整流程、账号身份动态读取、workctl CLI 网关凭证注入（gateway-cli.json 字段说明）、accio-mcp-cli 通用 MCP 用法、三条授权文件关联关系、子账号数据获取与限制、常用 icbu 命令清单、故障排查速查与安全规范。适用于：需要拉取阿里国际站实时数据、排查 workctl / accio-mcp 报错（desktop_not_attached、401、参数缺失等）、或想了解账号鉴权机制时按需查阅。
>
> **验证状态**：文件路径已于 2026-08-21 在 macOS 实测核对（gateway-cli.json / state.json / plugins / skills / accio-mcp.mjs 均存在，workctl v0.1.33 已安装）；命令行为与坑点为 2026-07-27 Windows 环境实测沉淀，机制跨平台一致。

## 0. 路径约定

本账号在两类机器上的根路径（`{username}` / `{accountId}` 为占位符，实际值运行时解析，均不写入本 skill）：

| 平台 | 账号根目录 | 占位符说明 |
|------|-----------|-----------|
| macOS（本机） | `/Users/{username}/.accio/accounts/{accountId}/` | `{username}` = `whoami`；`{accountId}` = 账号 ID（发现命令见下） |
| Windows | `C:\Users\<USER>\.accio\accounts\<ID>\` | `<USER>` = `%USERNAME%`；`<ID>` = 账号根目录名 |

- 命令示例统一用 `$HOME` 展开（macOS/Linux bash 直接可用，Windows git-bash 同样支持），不把本机用户名硬编码进命令。
- bash / `node` 命令里正斜杠在 Windows 也能用，避免反斜杠转义问题。
- 账号 ID 运行时发现（实际值不写入本 skill，避免随公开仓库泄露）：`~/.accio/accounts/` 下的目录名即本账号 ID，本机通常仅一个（多个账号时取实际账号的目录名）。后续命令中的 `$ACC_ID` 均来自：

```bash
ACC_ID=$(ls "$HOME/.accio/accounts/" | head -1)
```

---

## 一、会话启动自检（GitHub 版本检查，完整流程）

> 每个工作会话开始时、交接给其他技能前，**静默**跑一次自检（不要把原始命令输出甩给用户——只报一行结论，有更新时附操作提示）。

**目的**：让本入口 skill 与 GitHub 源保持同步，确保 TOOMOTOO 销售流程始终跑在最新的鉴权流程、路由规则与 ICBU 命令清单上。

### 常量

| 名称 | 值 |
|------|-----|
| GitHub 仓库 | `https://github.com/Jim-purch/accio-work-output.skill.git` |
| 最新版 API | `https://api.github.com/repos/Jim-purch/accio-work-output.skill/releases/latest` |
| 远端版本来源 | 最新非预发布 Release 的 `tag_name` 字段（去掉前导 `v`，如 `v0.1.6` → `0.1.6`） |
| 本地版本来源 | 本 skill 已加载 frontmatter 的 `version:` 字段（加载时已在内存中——**勿从磁盘重读**） |

### 自检步骤

按序执行，成功一步即停：

> **为什么只信 GitHub（不读本地文件）**：新装机器上 skill 目录可能为空或文件未落地，从磁盘读 `SKILL.md` 取版本不可靠。本地版本加载 frontmatter 时已在内存；远端版本取 GitHub Releases 的 `tag_name`——Release 反映维护者实际发布的内容，比可能带未完成工作的 `main` 分支头更可靠。磁盘文件从不参与比对。

**第 1 步 — 从 GitHub Releases 拉最新已发布版本（唯一的网络调用）：**

```bash
# 远端版本——最新非预发布 Release。公开 API 未鉴权 60 次/小时，每会话一次足够。
# JSON 用 node 解析（本环境可用），比 sed 稳。解析/网络任何错误 → REMOTE_VER 为空 → 走第 2 步。
# 仓库无 Release 时返回 404 → 同样走第 2 步。
REMOTE_VER=$(curl -fsSL --max-time 8 \
  "https://api.github.com/repos/Jim-purch/accio-work-output.skill/releases/latest" \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{try{const t=JSON.parse(s).tag_name||'';process.stdout.write(t.replace(/^v/,''))}catch(e){process.exit(1)}})" \
  | head -1)
# LOCAL_VER = 本 skill 已加载 frontmatter 的 version:（文件顶部唯一硬编码版本），已在内存中，勿读盘。
# 语义比较（major.minor.patch）：
#   REMOTE_VER == LOCAL_VER → UP_TO_DATE
#   REMOTE_VER >  LOCAL_VER → BEHIND
#   REMOTE_VER <  LOCAL_VER → AHEAD（本地开发版比已发布版新）
```

`REMOTE_VER` 非空 → 按下表报结论并结束。

**第 2 步 — 离线 / 网络受限**：第 1 步失败（无网络、GitHub 不可达、沙箱拦截、`REMOTE_VER` 为空）→ 静默跳过，用本地副本继续，不阻塞用户工作。

### 结果上报（一行中文 + 仅在有更新时附操作提示）

> 占位符 `{LOCAL_VER}` / `{REMOTE_VER}` 取自上述变量（`LOCAL_VER` = frontmatter `version:`，`REMOTE_VER` = Release `tag_name` 去掉 `v`）。禁止在消息里硬编码版本号。

| 结果 | 告知用户的内容 |
|------|---------------|
| `UP_TO_DATE` | 「✅ 已自检：本 skill 为最新版本 (v{LOCAL_VER})，与 GitHub 一致。」 |
| `BEHIND` | 「🔄 检测到 GitHub 有新版本 (本地 v{LOCAL_VER} → 远端 v{REMOTE_VER})。是否拉取更新？」——等用户确认后再更新，绝不自动覆盖用户可能正在编辑的 skill。更新方式：skill 目录是 `git clone` 则 `git -C <skill dir> pull origin main`；否则从仓库重新复制/安装。 |
| `AHEAD` | 「ℹ️ 本地版本 (v{LOCAL_VER}) 高于 GitHub 已发布版本，可能是本地开发版，无需更新。」 |
| 离线（第 2 步） | 「ℹ️ 未能连接 GitHub 检查更新（网络受限），本次会话使用本地 v{LOCAL_VER} 继续。」 |

### 规则

1. **绝不未经用户确认自动更新**：`BEHIND` 只提示不拉取（用户可能有未提交的本地修改）。
2. **绝不让自检阻塞会话**：超过约 8 秒或失败，用本地副本继续并一行带过。
3. **每会话只跑一次**（不是每条消息）：结果缓存在内存里，本轮会话后续不再重复请求。
4. **不在报告行泄露敏感路径或 token**：只给版本号与一行更新提示。
5. 上报后立即继续 bootstrap 的其余部分（暴露 ICBU/workctl 层），再交接用户实际要做的事。

## 二、账号身份信息（动态读取，勿硬编码）

账号身份会随绑定/变更而变化，**使用时从 OAuth 状态文件实时读取**：

| 读取项 | 来源（绝对路径） | 说明 |
|--------|------|------|
| login_id / ali_id / serviceType | `/Users/{username}/.accio/accounts/{accountId}/connectors/data/alibaba/state.json` | 账号唯一标识与服务类型 |
| 授权状态 / 授权时间 | 同上 `state.json` | status / authorizedAt 字段 |
| 已安装插件 | `/Users/{username}/.accio/accounts/{accountId}/plugins/installed/` | 目录名即插件名 |

```bash
# 动态读取当前账号身份（输出 JSON，含 login_id/ali_id/serviceType/status 等；$ACC_ID 定义见 §0）
node -p "JSON.parse(require('fs').readFileSync('$HOME/.accio/accounts/$ACC_ID/connectors/data/alibaba/state.json'))"
```

公司主体固定：Toomotoo (Tianjin) International Trading（阿里国际站 cgs 卖家，主营叉车配件）。

## 三、workctl CLI 鉴权（已验证可用 ✅）

### 3.1 工具版本与网关地址（动态读取）

workctl 版本与网关地址会随升级/重启变化，**使用时动态读取，勿硬编码**：

```bash
workctl --version          # 查 workctl 版本（本机实测 v0.1.33）
workctl health             # 查网关 backend 状态与连通性
# 网关地址（url 字段）见 /Users/{username}/.accio/accounts/{accountId}/.accio/runtime/gateway-cli.json
```

### 3.2 凭证获取（核心，最关键的一步）

主 Agent 沙箱默认**没有** `ACCIO_GATEWAY_TOKEN` 环境变量，直接跑 workctl 动态命令会报 `desktop_not_attached`。用 `gateway-cli.json` 的 password 注入：

```bash
ACC_ID=$(ls "$HOME/.accio/accounts/" | head -1)   # 账号 ID 运行时发现（§0；同一 shell 已定义可跳过）
export ACCIO_GATEWAY_TOKEN=$(node -p "JSON.parse(require('fs').readFileSync('$HOME/.accio/accounts/$ACC_ID/.accio/runtime/gateway-cli.json')).password")
```

注入后 workctl 动态命令（`icbu` 子命令）全部可用。

### 3.3 gateway-cli.json 字段说明

路径：`/Users/{username}/.accio/accounts/{accountId}/.accio/runtime/gateway-cli.json`（网关重启会重写整个文件）

| 字段 | 含义 | 是否会变 |
|------|------|---------|
| url | 网关 HTTP 地址（如 `http://localhost:40xx`） | 重启可能变 |
| wsUrl | WebSocket 地址 | 随 url 变 |
| authMode | 认证模式（basic） | 稳定 |
| username | 网关用户名 | 见文件 |
| password | 网关密钥（= `ACCIO_GATEWAY_TOKEN` 的值） | 重启可能变 |
| relayPort | 中继端口 | 会变 |
| pid / createdAt | 进程 ID / 创建时间 | 每次重启必变 |

> ⚠️ **重要**：网关重启后 `gateway-cli.json` 会重写（pid/createdAt 必变，password/relayPort 也可能变），使用时务必**实时读取原文件**，不要缓存旧值。具体字段值不硬编码于任何 skill。

### 3.4 连通性验证

```bash
# 注入凭证后
workctl health
# 期望返回：backend=pass, version=pass（auth/config/network/cache 为 skip，属正常）
```

### 3.5 可用 icbu 子命令一览

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

### 3.6 已知坑点（实测踩雷）

| # | 坑点 | 表现 | 解决 |
|---|------|------|------|
| 1 | `-f json` 在子命令层级不识别 | `unknown shorthand flag: 'f'` | 去掉 `-f json`，默认输出即 JSON |
| 2 | `-o <path>` 输出文件路径解析异常（Windows 尤甚） | `create output file: open ... 报错` | 改用 bash `> file.json` 重定向 |
| 3 | TM 诊断需复合参数 | `missing required parameter: buyerType/dateType/queryDate/start_time` | `list-seller-*-dim-diag-data` 需同时传 buyerType + dateType + start_time 等 |
| 4 | 广告账户诊断旧 API 已废弃 | `code 410, This API is deprecated` | 迁移至 HATEOAS：`list --entityType=diagnosis`（省略 campaignId 做账户级） |
| 5 | 数据 T+2 延迟 | 最新可查日为前天 | 经营大盘等聚合数据最新为 T-2 |
| 6 | 子账号会话消息无权读 | `not in conversation` | 主账号非子账号会话成员，需子账号登录态（见§六绕法） |

## 四、Accio MCP 鉴权机制

Accio Desktop 自带两套 MCP 能力，鉴权与可用性完全不同：

| 能力 | 工具 | 鉴权方式 | 可用性 |
|------|------|---------|--------|
| 阿里国际站 MCP 接口 | accio-mcp（connector-proxy 内置） | 走 connector OAuth | ❌ 当前不可用 |
| 通用 MCP 工具（Gmail/Twitter/Notion 等） | accio-mcp-cli（Node 脚本） | Desktop 自动注入 | ✅ 可用 |

### 4.1 阿里 MCP 接口（不可用 ❌，仅记录）

- 阿里 MCP 工具**未通过 connector-proxy 暴露**
- 插件 `connectors.json` 的 `mcpServers={}`（空配置）
- `ListMcpResources` 仅返回 Ardot Editor，无阿里相关资源
- `alibaba-chat-and-analysis` 子代理需 `sessions_spawn` 调度，主 Agent 工具集无此能力
- **结论**：当前无法通过 MCP 直接调用阿里接口，必须走 workctl icbu 路径（见§三）

### 4.2 accio-mcp-cli 鉴权机制（通用 MCP，可用 ✅）

> 本节源自 `accio_mcp_cli_proxy.py` 的实测封装经验。accio-mcp-cli 能调 Gmail / Twitter / Notion / Square / Apify（含 Instagram/Facebook/TikTok/YouTube/Reddit/1688）/ Google Workspace / GitHub / Composio（Figma/HubSpot/Intercom）等远程 MCP 工具。

#### 4.2.1 本体位置（关键：不在 PATH）

accio-mcp-cli **不是独立可执行命令**，实际是一个 Node 脚本 `accio-mcp.mjs`，藏在 Accio Desktop 安装目录里：

```
macOS:   /Applications/Accio.app/Contents/Resources/accio-mcp-cli/accio-mcp.mjs
Windows: C:\Users\<USER>\AppData\Local\Programs\Accio\resources\accio-mcp-cli\accio-mcp.mjs
```

定位优先级（高 → 低）：

1. 显式传入路径
2. 环境变量 `ACCIO_MCP_CLI_PATH`
3. 上面默认路径
4. `shutil.which('accio-mcp-cli')`（若用户手动加入了 PATH）

**前提**：Accio Desktop 必须已安装且在运行——否则 .mjs 文件不存在或 gateway 不通。

#### 4.2.2 运行方式

.mjs 必须用 node 跑（Node.js ≥ 18）：

```bash
# macOS
node "/Applications/Accio.app/Contents/Resources/accio-mcp-cli/accio-mcp.mjs" <子命令>
```

#### 4.2.3 鉴权（注入 token）

accio-mcp-cli **需要手动 `export ACCIO_GATEWAY_TOKEN`**：

- 默认连本机 gateway（端口 4097）
- 凭证注入到 .mjs 进程

> ⚠️ **与 workctl 的对比**：workctl 走 `gateway-cli.json`（端口可能变、需手动注入 password）；accio-mcp-cli 走内部固定端口 4097 + 凭证。

#### 4.2.4 命令模式

| 命令 | 用途 |
|------|------|
| `toolkit [kw]` | 浏览 toolkit（无 kw 列全部概览，有 kw 列该 toolkit 下所有工具） |
| `search <kw>` | 按关键字全文搜工具（匹配 name/description/toolkit/service） |
| `call <tool> [--json-file path \| --key val ...] [--server <srv>] [--raw]` | 调用工具 |
| `list` | 列出全部工具（150+，慎用，会刷屏） |
| `server list / tools <name> / test <name> / add / remove` | 自定义 MCP server 管理 |

#### 4.2.5 调用流程（toolkit → search → call）

```bash
MJS="/Applications/Accio.app/Contents/Resources/accio-mcp-cli/accio-mcp.mjs"

# 1) 浏览某 toolkit
node "$MJS" toolkit alibaba

# 2) 按关键字搜工具
node "$MJS" search twitter

# 3) 调用工具（参数走 --json-file，避免 shell 引号地狱——Windows 必须如此，macOS 也推荐）
echo '{"productId":"16XXXXXXXXXXX","queryType":"trunk","componentList":["attr"]}' > /tmp/args.json
node "$MJS" call product_query_information --json-file /tmp/args.json --raw
```

#### 4.2.6 响应解析（套娃 JSON 坑）

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

#### 4.2.7 已知坑点

| # | 坑点 | 解决 |
|---|------|------|
| 1 | Windows shell 引号地狱（JSON 参数带引号被吞） | 用 `--json-file <path>` 传 JSON，别用 `--key val` 内联 |
| 2 | accio-mcp-cli 正常退出时偶发 Node UV async 断言警告到 stderr，returncode 非 0 | 只要 stdout 非空就视为成功；仅 stdout 空 且 rc≠0 才报错 |
| 3 | `list` 返回 150+ 工具刷屏 | 用 `toolkit`/`search` 缩小范围 |
| 4 | 找不到 accio-mcp.mjs | 确认 Accio Desktop 已安装启动；或设 `ACCIO_MCP_CLI_PATH` 环境变量；或加入 PATH |
| 5 | 鉴权失败 / 401 | 重启 Accio Desktop（凭证由 Desktop 注入） |
| 6 | Node 未安装或版本低 | 装 Node.js ≥ 18 |

#### 4.2.8 推荐封装

直接调 .mjs 较繁琐（路径定位 / Node / 响应解析 / 引号），建议封装一层 Python 代理屏蔽这些细节，对外暴露：

- `proxy.toolkit("alibaba")` / `proxy.search("twitter")` / `proxy.call(tool, params_dict)`
- 自动处理 .mjs 路径定位、`--json-file` 传参、多层 JSON 解套、退出码容错

参考实现：`accio_mcp_cli_proxy.py`（已存在于本地 `workctl-exports/` 目录）。配套技能 `accio-mcp-cli` 的 SKILL.md 也记录了相同工作流，可对照阅读。

## 五、三条授权文件的关联关系

| # | 文件 | 路径（绝对路径） | 作用 |
|---|------|------|------|
| 1 | 阿里 OAuth 状态 | `/Users/{username}/.accio/accounts/{accountId}/connectors/data/alibaba/state.json` | 账号身份凭证（ali_id/login_id/serviceType/status，值动态读取） |
| 2 | 插件 connector 配置 | `/Users/{username}/.accio/accounts/{accountId}/plugins/installed/alibaba-com-seller-assistant/connectors/connectors.json` | OAuth 适配声明（mcpServers={}，oauth adapter=mcp-oauth2） |
| 3 | 网关 CLI 凭证 | `/Users/{username}/.accio/accounts/{accountId}/.accio/runtime/gateway-cli.json` | 连网关的通行证（url/password 见该文件，值动态读取） |

**关联逻辑**：

- workctl 动态命令 / 阿里 MCP 调用都走网关（url 见 `gateway-cli.json`）
- 网关 Basic Auth 凭证在文件 3
- workctl 用 `ACCIO_GATEWAY_TOKEN` 环境变量取 token（health 实测）
- **OAuth（文件 1 账号身份）+ 网关凭证（文件 3 通行证）配合即可拉实时数据**，文件 2 仅是适配声明

## 六、子账号数据获取与限制

### 6.1 可获取（主账号权限）

- 子账号列表：`workctl icbu member list`（账号数量与人员会变，**动态查询，勿硬编码**）
- 各子账号会话列表：`workctl icbu tm list-conversation --aliId=<子账号aliId>`（返回最近 20 条会话，含 lastMessageTime/unread/买家国家/标签；子账号 aliId 从 member list 结果取）

### 6.2 子账号会话消息体（可获取，必须用对接口）

- ❌ `list-conversation-msg`（按 conversationId 点查）：校验会话 membership，主账号非子账号会话成员，返回 `not in conversation`——此路不通，但**不代表无权读取**
- ✅ `list-conversation-msg-time-range`（按时间范围查）：接受可选参数 `selfAliId`（"查询对象aliId"，integer）。主账号 token 传入子账号 aliId 即可读该子账号会话消息体（限近 30 天）
- 推荐用法：直接走 `workctl workflow chat-analysis fetch-messages`（内部自动把会话 `sellerAliId` 映射为 `list-conversation-msg-time-range` 的 `selfAliId`，多会话并行、按 3 段时间窗拉取）；详见 alibaba-chat-and-analysis 子代理 `msg-history.md`
- 结论：首响时长/回复时长/回复质量**可量化**——消息体可读且带 `senderAliId`+`timestamp`，可按子账号拆分计算

### 6.3 诊断聚合（受限）

- `list-seller-acct-dim-diag-data`：仅主账号 aliId 可用，子账号 aliId 无权访问账号维度诊断（官方 schema 明示）

## 七、常用数据获取命令清单（可直接复制）

```bash
# ===== 第一步：注入网关凭证（每次新开 shell 都要做） =====
ACC_ID=$(ls "$HOME/.accio/accounts/" | head -1)   # 账号 ID 运行时发现（§0；同一 shell 已定义可跳过）
export ACCIO_GATEWAY_TOKEN=$(node -p "JSON.parse(require('fs').readFileSync('$HOME/.accio/accounts/$ACC_ID/.accio/runtime/gateway-cli.json')).password")

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

## 八、故障排查速查

| 现象 | 原因 | 解决 |
|------|------|------|
| `desktop_not_attached` | 未注入 ACCIO_GATEWAY_TOKEN | 执行 §3.2 的 export 命令 |
| health pass 但 schema 返回 401 | 用了假 token 或 token 过期 | 重新从 gateway-cli.json 读真实 password |
| `missing required parameter` | 漏传必填参数 | 看错误信息的 next_action，补齐参数 |
| 命令报 `unknown shorthand flag` | 用了 `-f` 等子命令不支持的 flag | 去掉 flag，用默认输出 |
| 输出文件创建失败 | `-o` 路径解析问题 | 改用 `> file` 重定向 |
| 数据为 null | T+2 未生成 或 小店无数据 | 换更早的日期重试 |

## 九、安全注意事项

1. `gateway-cli.json` 含 password 密钥，**勿明文外传**，skill 只记录机制不记明文
2. 客户档案数据仅存储在本地，不外传
3. 不在搜索查询中暴露 TOOMOTOO 的内部数据（成本价、客户列表）
4. 网关重启后凭证失效，需重新读取 gateway-cli.json
5. 涉及客户数据的查询与归档带"负责销售"过滤，不跨销售共享客户信息

---

*本文档只记录稳定的机制与命令。账号身份、workctl 版本、网关端口、子账号人员等运行时值按各节指引动态读取。如 workctl 升级或网关机制变更，同步更新本文档。*
