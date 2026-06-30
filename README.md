# lin

**Swap the Linear MCP account bound to a Claude Code project folder — per folder, like `use-gh`.**

`lin` is a single self-contained bash script. You register your Linear API keys once, then run `lin use <name>` inside any project folder to stamp that folder's `.mcp.json` with the right key. Claude Code reads `.mcp.json` on launch, so the next time you start `claude` in that folder it talks to Linear as that account.

Everything is **per-folder**. The only global state is the account registry (`~/.config/lin/accounts`) — the pool of keys you add. `lin` never touches `~/.claude.json` or any global Claude config.

---

## Why

The Claude/Cursor "Connect Linear" button uses an OAuth browser flow that never gives you a copyable key, so it can't be swapped per project. `lin` uses **static personal API keys** in the MCP `Authorization` header instead, which means you can point different folders at different Linear accounts (e.g. personal vs. client vs. work) and switch with one command.

---

## Requirements

- **bash** and **curl**
- **python3** — strongly recommended. Without it, `lin` still binds a single account per folder, but loses: the `lin projects` listing, multi-account folder sets, account-name detection in `lin status`, and the `--team`/`--project` scope feature.
- **Claude Code** — the consumer of the `.mcp.json` files this writes.

---

## Install

Clone the repo and put `lin` somewhere on your `PATH`.

```bash
git clone https://github.com/roco55/lin-cli.git
cd lin-cli

# Option A — symlink into a bin dir already on your PATH
ln -s "$PWD/lin" ~/.local/bin/lin     # or /usr/local/bin/lin

# Option B — copy it
cp lin ~/.local/bin/lin

chmod +x ~/.local/bin/lin              # if not already executable
```

Confirm it's working:

```bash
lin help
```

### Tab-completion (optional)

```bash
# zsh — add to ~/.zshrc, then restart your shell
source <(lin completion zsh)

# bash — add to ~/.bashrc, then restart your shell
source <(lin completion bash)
```

### Getting a Linear API key

Linear → **Settings → Account → Security & Access → "New API key"**
<https://linear.app/settings/account/security>

This static key is separate from the OAuth "Connect Linear" browser flow — that flow gives you no copyable key, which is exactly why per-folder swapping needs a static one.

---

## Quick start

```bash
# 1. Register your accounts once (stored in ~/.config/lin/accounts, chmod 600)
lin add personal lin_api_xxxxxxxxxxxxxxxx
lin add client   lin_api_yyyyyyyyyyyyyyyy

# 2. (optional) See an account's real teams & projects, live from Linear
lin projects client

# 3. Inside a project folder, bind it to an account
cd ~/projects/some-client-app
lin use client

# 4. Restart `claude` in that folder — it now talks to Linear as "client"
```

---

## Commands

| Command | What it does |
|---|---|
| `lin add <name> <key>` | Register (or overwrite) an account in the global registry. |
| `lin rm <name>` | Remove an account from the registry. |
| `lin list` | List registered account names. |
| `lin projects <name>` | List that account's real teams & projects, fetched live from Linear. |
| `lin use <name> [more...]` | Make `<name>` the **active** account in the current folder. Adds it to the folder's set if new and promotes it to the top. |
| `lin use <name> --team "<Team>"` | …and set a default Linear **team** for this folder (written to `CLAUDE.md`). |
| `lin use <name> --project "<Project>"` | …and set a default Linear **project** for this folder (written to `CLAUDE.md`). |
| `lin status` | Show this folder's active account, its full set, and any team/project scope. |
| `lin completion <bash\|zsh>` | Print shell completion to source. |
| `lin help` | Full help. |

### Switching accounts in a folder

A folder's account **set only grows**. Add several at once, then switch between them by re-running `lin use`:

```bash
lin use personal client     # set = {personal, client}, active = personal
lin use client              # active = client  (set unchanged)
lin status                  # shows active + the rest of the set
```

---

## How it works

- **Registry** — `lin add` writes `name<TAB>key` lines into `~/.config/lin/accounts` (created `chmod 600`). This is the only global state and the source for tab-completion.

- **Binding a folder** — `lin use` writes a `.mcp.json` in the current directory with a `linear` HTTP MCP server pointing at `https://mcp.linear.app/mcp`, carrying `Authorization: Bearer <key>`. It preserves any other MCP servers already in the file. It also records the folder's account set in a non-MCP `_lin` block (Claude Code ignores unknown top-level keys) so re-running `use` and tab-completion know this folder's options.

- **Taking effect** — Claude Code reads `.mcp.json` when it starts, so a bind takes effect the **next time you launch `claude`** in that folder.

- **Team/project scope** — `--team` / `--project` write a managed, marker-delimited block into the folder's `CLAUDE.md` telling Claude to default Linear operations to that team/project. The block is idempotent — re-running updates it in place.

- **Listing** — `lin projects` queries Linear's GraphQL API (`https://api.linear.app/graphql`) with the same key. Teams and projects are fetched as two flat queries and grouped client-side, because a single deeply-nested query exceeds Linear's complexity cap.

---

## Security

- Account keys live in `~/.config/lin/accounts`, created `chmod 600` (owner read/write only).
- Each folder's `.mcp.json` holds a **live key** in plaintext. When `lin use` runs inside a git repo, it auto-appends `.mcp.json` to that repo's `.gitignore` so the key isn't committed. The `.mcp.json` is also written `chmod 600`.
- Never commit a `.mcp.json` that contains a real key. (This repo's own `.gitignore` already excludes it.)

---

## License

MIT
