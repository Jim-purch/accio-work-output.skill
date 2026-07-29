# accio-work-output

> **Priority / entry-point skill for TOOMOTOO forklift-parts foreign-trade
> sales** — boots first at the start of every work session inside the
> `.accio` account workspace, self-checks for GitHub updates, enforces the
> `Work/` deliverable routing rule, and exposes the Alibaba.com (ICBU)
> workctl data layer for all downstream sales skills.

| Field | Value |
|-------|-------|
| Status | initial public release |
| Priority | `primary` — invoke FIRST on every session start |
| Domain | TOOMOTOO (Tianjin) International Trading — forklift parts (ICBU cgs seller) |
| GitHub repo | https://github.com/Jim-purch/accio-work-output.skill.git |
| Last verified | 2026-07-28 |

---

## What this skill does

`accio-work-output` is the **entry-point skill** of the TOOMOTOO forklift-parts
foreign-trade sales workspace. It is intentionally small in scope: it does
three bootstrap jobs at the start of every work session, then hands off to
the matching companion skills — discovered dynamically in the account's
`skills/` directory at runtime, never assumed from a fixed list (see
`SKILL.md` §九) — while continuing to enforce one global rule for every file
those downstream skills produce.

The three bootstrap jobs:

1. **Self-check GitHub updates** — compare the local `version` against the
   published repo and notify the user in one line (see
   [Session-start self-check](#session-start-self-check)).
2. **Route every deliverable into `Work/`** — the single home for produced
   files at the account root. The account root stays clean; everything the
   agent produces is easy to find, back up, or delete in one place.
3. **Expose the ICBU/workctl data layer** — the auth flow, command list, and
   known pitfalls for pulling real-time store data via `workctl icbu …` and
   `accio-mcp-cli`. Downstream sales skills consume this instead of
   re-deriving the auth flow each time.

After bootstrap, the skill stays resident for the rest of the session: any
time any skill (or the user) is about to write a deliverable, this skill's
`Work/` routing rule applies.

## Why a "priority / entry-point" skill?

The TOOMOTOO account has 20+ cooperating skills installed under `skills/`
(the set changes over time — the agent enumerates the directory and matches
each skill's `description` at runtime rather than relying on a hardcoded
list). Without an
explicit entry point, the agent has to guess which skill to load first, and
the ICBU auth flow + `Work/` routing rule get re-derived or skipped entirely.
This skill makes the ordering explicit: **it loads first, every session**, so
the auth flow and routing rule are always established before any sales work
begins.

## Session-start self-check

On the first message of every session, the skill silently checks whether the
local copy is in sync with GitHub. It tries two steps in order and stops at
the first that works:

1. **GitHub Releases API** — fetches the latest non-prerelease release's
   `tag_name` (e.g. `v0.1.6`), strips the leading `v`, and compares semver
   against the local frontmatter `version:` field. This reflects what the
   maintainer actually shipped, rather than the raw `main` branch head which
   may carry unfinished work.
2. **Offline / network blocked** — skip silently, proceed with the local copy.

The user sees one short line:

- `✅ 已自检：本 skill 为最新版本，与 GitHub 一致。`
- `🔄 检测到 GitHub 有新版本。是否拉取更新？` — only
  *prompts*; never auto-pulls. The user may have local edits in flight.
- `ℹ️ 未能连接 GitHub 检查更新（网络受限），本次会话使用本地版本继续。`

Rules: run once per session (not per message), never block the session on the
check, never auto-update without user confirmation, never leak sensitive paths
or tokens in the report line. Full procedure lives in `SKILL.md` →
*会话启动自检（Session-start self-check）*.

## The `Work/` routing rule

All deliverables go into:

```
C:\Users\<USER>\.accio\accounts\<ID>\Work\
```

Substitute `<USER>` (Windows username) and `<ID>` (account ID = basename of
the account root) at runtime. Inside `Work\`, group related files under
descriptive subfolders (`Work\reports\`, `Work\scripts\`, `Work\exports\…\`).
For dated material, prefix file names with `YYYY-MM-DD` so the folder stays
sortable.

Do **not** redirect the platform's own runtime files (`.workbuddy/`,
`automations/`, `conversations/`, `agents/`, `skills/`, connector state)
into `Work\` — those stay where the platform put them. `Work\` is for
**user-facing deliverables only**.

## ICBU data layer (summary)

The skill carries the full, field-tested auth flow for pulling real-time
store data on this account. Highlights:

- **Account**: Toomotoo (Tianjin) International Trading, Alibaba.com cgs
  seller, forklift parts. Account identity (`login_id` / `ali_id` /
  `serviceType`) is read dynamically from
  `connectors\data\alibaba\state.json` — never hardcoded.
- **workctl CLI** (verified working ✅): the gateway token is injected from
  `.accio\runtime\gateway-cli.json`'s `password` field into
  `ACCIO_GATEWAY_TOKEN` at the start of each shell. Without this injection,
  dynamic commands fail with `desktop_not_attached`.
- **accio-mcp-cli** (verified working ✅): a Node script
  (`accio-mcp.mjs`) shipped with Accio Desktop; auth is automatic via the
  Desktop's internal gateway. Workflow: `toolkit → search → call`. Use
  `--json-file` to dodge Windows shell quoting hell.
- **Ali MCP interface** (currently unavailable ❌): not exposed via
  connector-proxy; `mcpServers={}`. Use the workctl path instead.
- **Available `workctl icbu` subcommands**: `advisor`, `crm`, `product`,
  `tm`, `rfq`, `trade`, `ads`, `logistics`, `member`, `cco`, `storefront`.
- **Known pitfalls**: `-f json` not recognized at subcommand level,
  `-o <path>` mis-parses on Windows (use `> file` redirect), TM diagnosis
  needs composite params, old ads API is deprecated (use HATEOAS), data is
  T+2, main account can't read sub-account conversation bodies.

The full command list, field-by-field `gateway-cli.json` reference, the
three-file auth relationship, sub-account limits, and a troubleshooting
matrix live in `SKILL.md` → *阿里国际站 workctl & accio-mcp 鉴权经验*.

## Installation

### Option A — clone (recommended, makes pulling updates easier)

```bash
# <USER> and <ID> are the Windows username and Accio account ID
SKILL_DIR="C:/Users/<USER>/.accio/accounts/<ID>/skills/accio-work-output"
git clone https://github.com/Jim-purch/accio-work-output.skill.git "$SKILL_DIR"
```

### Option B — copy

Copy this folder (or just `SKILL.md`) into
`C:\Users\<USER>\.accio\accounts\<ID>\skills\accio-work-output\`. The
self-check will fall back to Step 2 (offline / network blocked).

### Verify

Open a new work session in the account and check that the agent reports
`✅ 已自检：本 skill 为最新版本，与 GitHub 一致。` on the first message.

## Updating

To pull a newer version published on GitHub:

```bash
SKILL_DIR="C:/Users/<USER>/.accio/accounts/<ID>/skills/accio-work-output"
git -C "$SKILL_DIR" pull origin main
```

The self-check will prompt you when a new version is available — confirm
before pulling, especially if you have local edits to `SKILL.md`.

## Contributing

This skill is purpose-built for the TOOMOTOO account and is published
primarily so the agent can self-check for updates. If you fork it for another
ICBU seller account:

1. Replace the TOOMOTOO-specific positioning in the frontmatter `description`
   and the `## What this skill does` intro.
2. Re-verify the workctl / accio-mcp auth flow on the new account — the
   mechanism is stable, but runtime values (gateway port, sub-account list,
   plugin set) are account-specific.
3. Bump the `version` field in `SKILL.md` frontmatter.

## License

Internal use only. Distributed to TOOMOTOO sales workspaces via the
`.accio/accounts/<ID>/skills/` convention. Not for redistribution outside
the company.
