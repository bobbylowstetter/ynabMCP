# ynabMCP

Personal project scaffold that gives Claude Code (and Cloud Routines) read access to
[YNAB](https://www.ynab.com) via the community [`@maro-org/ynab-mcp`](https://www.npmjs.com/package/@maro-org/ynab-mcp)
server, plus Notion access for pulling budget data into a workspace.

No custom server code lives here — this repo is just the project-scoped MCP config
(`.mcp.json`) and setup notes.

## Why a PAT, not OAuth

Composio's shared YNAB OAuth app is capped at 25 authorized users and is often already
maxed out ("restricted and unable to be authorized"). For a single-user pipeline like
this, a YNAB Personal Access Token sidesteps OAuth entirely and is the simpler option.

## Setup

### 1. Generate a YNAB Personal Access Token

YNAB → Account Settings → Developer Settings → New Token.

### 2. Make the token available to Claude Code

`.mcp.json` (committed, safe) references `${YNAB_API_TOKEN}` — Claude Code expands
this from your shell environment at connect time. It does **not** auto-load a `.env`
file, so pick one:

- Copy `.env.example` to `.env`, fill in the token, then before running Claude Code:
  ```bash
  set -a && source .env && set +a
  ```
- Or export it permanently in your shell profile (`~/.zshrc`):
  ```bash
  export YNAB_API_TOKEN="your-pat-here"
  ```

`.env` is gitignored — only `.env.example` is tracked.

### 3. Verify the MCP servers connect

From this project directory:

```bash
claude mcp list
```

You should see `ynab` and `notion` both connected. `notion` uses Claude Code's
existing OAuth session (`claude.ai`/Notion), so no token is needed for it.

### 4. Read-only for now

`.mcp.json` sets `YNAB_READ_ONLY=true` — Claude can read transactions, balances, and
categories, and use `suggest_transaction_categories`, but every write tool
(`update_transactions`, etc.) is blocked.

**To enable labeling later**, flip the flag:

```jsonc
"YNAB_READ_ONLY": "false"
```

in `.mcp.json`, then `claude mcp list` again to confirm. Writes are logged with undo
entries (`list_undo_history` / `undo_operations`) if auto-categorization gets
something wrong.

### 5. Cloud Routine (optional, for a daily pipeline)

Routines are project-scoped, so once `.mcp.json` lives here, a Routine pointed at this
project picks up both MCP servers automatically. Example prompt:

> Pull today's transactions and category balances from YNAB (read-only) and update the
> corresponding Notion pages in [workspace/page].

## Security note

Never commit a literal token into `.mcp.json` or anywhere else in this repo — always
go through `${YNAB_API_TOKEN}` expansion and keep the real value in your shell
environment or the gitignored `.env`.
