# accio-work-output

> **Priority / entry-point skill for TOOMOTOO forklift-parts foreign-trade
> sales** — boots first at the start of every work session inside the
> `.accio` account workspace, self-checks for GitHub updates, and exposes the
> Alibaba.com (ICBU) workctl data layer for all downstream sales skills.

| Field | Value |
|-------|-------|
| Status | public release |
| Priority | `primary` — invoke FIRST on every session start |
| Domain | TOOMOTOO (Tianjin) International Trading — forklift parts (ICBU cgs seller) |
| GitHub repo | https://github.com/Jim-purch/accio-work-output.skill.git |
| Last verified | 2026-08-21 |

---

## What this skill does

`accio-work-output` is the **entry-point skill** of the TOOMOTOO forklift-parts
foreign-trade sales workspace. `SKILL.md` is kept intentionally thin (~100
lines): it carries only what must be resident in every session, and delegates
all details to the `reference/` library for on-demand consultation.

The two bootstrap jobs:

1. **Self-check GitHub updates** — compare the local `version` against the
   published repo and notify the user in one line.
2. **Expose the ICBU/workctl data layer** — the auth flow, command list, and
   known pitfalls for pulling real-time store data via `workctl icbu …` and
   `accio-mcp-cli`. Downstream sales skills consume this instead of
   re-deriving the auth flow each time.

After bootstrap, the skill hands off to the matching companion skills —
discovered dynamically in the account's `skills/` directory at runtime, never
assumed from a fixed list — and routes methodology questions to the
`reference/` playbooks (also enumerated dynamically).

## The gateway credential (the one path that matters)

All real-time ICBU data flows through the Accio gateway, whose CLI credential
lives at:

```
/Users/{username}/.accio/accounts/{accountId}/.accio/runtime/gateway-cli.json
```

`{username}` is the machine's username (`whoami`); `{accountId}` is the Accio account ID (`ls ~/.accio/accounts/` — usually a single directory). The literal account ID is deliberately never spelled out in this public repo; both placeholders resolve at runtime. The file's `url` field is
the gateway address; its `password` field is the value of
`ACCIO_GATEWAY_TOKEN`. The gateway rewrites this file on every restart, so it
must be read fresh each time — never cached, never hardcoded. Without the
token injection, workctl dynamic commands fail with `desktop_not_attached`.

## Repository layout

| Path | Purpose |
|------|---------|
| `SKILL.md` | Thin entry-point router (~100 lines): triggers, the two bootstrap jobs, the gateway credential path, and the reference index |
| `reference/workctl-icbu-鉴权与数据手册.md` | Full auth & data handbook: self-check procedure, account identity, workctl CLI auth, `gateway-cli.json` field reference, accio-mcp-cli usage, sub-account limits, command cookbook, troubleshooting |
| `reference/配套技能与文档发现流程.md` | Dynamic discovery flows for companion skills (`skills/`) and methodology docs (`reference/`) |
| `reference/业务谈判-销售全流程.md` | Sales negotiation playbook: first reply, FABE, quoting, price reduction, follow-ups, regional negotiation |
| `reference/平台运营-店铺与流量管理.md` | Platform operations playbook: storefront, listings, pricing, P4P (ops/owner-focused) |

Every `reference/` doc opens with a `>` summary block stating its coverage —
that block is the matching key when the agent enumerates the library.

## Session-start self-check

On the first message of every session, the skill silently checks whether the
local copy is in sync with GitHub (latest non-prerelease Release `tag_name`
vs the frontmatter `version:`), and reports one short line. It never
auto-pulls — a newer remote only *prompts* for confirmation. Full procedure:
`reference/workctl-icbu-鉴权与数据手册.md` §一.

## Output rule

Generated files (reports, exports, scripts) are written to the agent's
default working directory — the skill imposes no special output folder, since
a hard-coded folder may be unwritable on some setups.

## Installation

### Option A — clone (recommended, makes pulling updates easier)

```bash
# macOS: {username} = whoami; {accountId} = ls ~/.accio/accounts/ (usually one directory)
ACC_ID=$(ls "$HOME/.accio/accounts/" | head -1)
SKILL_DIR="$HOME/.accio/accounts/$ACC_ID/skills/accio-work-output"
git clone https://github.com/Jim-purch/accio-work-output.skill.git "$SKILL_DIR"
```

On Windows, target `C:\Users\<USER>\.accio\accounts\<ID>\skills\accio-work-output\`
instead (`<USER>` = `%USERNAME%`, `<ID>` = account root directory name).

### Option B — copy

Copy this folder (or just `SKILL.md` + `reference/`) into the skills
directory above. The self-check will fall back to offline mode.

### Verify

Open a new work session in the account and check that the agent reports
`✅ 已自检：本 skill 为最新版本，与 GitHub 一致。` on the first message.

## Updating

```bash
ACC_ID=$(ls "$HOME/.accio/accounts/" | head -1)
SKILL_DIR="$HOME/.accio/accounts/$ACC_ID/skills/accio-work-output"
git -C "$SKILL_DIR" pull origin main
```

The self-check will prompt you when a new version is available — confirm
before pulling, especially if you have local edits.

## Contributing

This skill is purpose-built for the TOOMOTOO account and is published
primarily so the agent can self-check for updates. If you fork it for another
ICBU seller account:

1. Replace the TOOMOTOO-specific positioning in the frontmatter `description`
   and the intro.
2. Re-verify the workctl / accio-mcp auth flow on the new account — the
   mechanism is stable, but runtime values (gateway port, sub-account list,
   plugin set) are account-specific.
3. Bump the `version` field in `SKILL.md` frontmatter.

## License

Internal use only. Distributed to TOOMOTOO sales workspaces via the
`.accio/accounts/<ID>/skills/` convention. Not for redistribution outside
the company.
