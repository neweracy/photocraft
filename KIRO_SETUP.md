# Kiro setup for PhotoCraft

Personal setup for driving PhotoCraft's MCP server from Kiro. This file is intentionally
separate from `README.md`. It documents one contributor's local Kiro wiring and is not part of
PhotoCraft's product docs.

## What this gives you

- The `photocraft` MCP server registered in Kiro, so an agent can read and edit documents through
  the same command registry the UI uses.
- A Bedrock model verified reachable, with the exact id to use.
- A `web-search` server for research, plus a steering file that enforces PhotoCraft's clean-room
  asset/licence rules.
- A fork + `upstream` remote and a global SessionStart hook that keeps the local checkout and the
  installed MCP binary up to date with upstream.

## 1. The MCP server

PhotoCraft exposes MCP through the CLI: `photocraft-cli mcp`. Headless mode owns its own engine and
does file I/O, which is what Kiro uses:

```sh
photocraft-cli mcp --headless --automation-read-root <dir> --automation-write-root <dir>
```

Each root grants one direction of filesystem access; an omitted root denies it. Roots must be
absolute (a relative root resolves from the client's launch directory).

### Build / install

The binary is installed on PATH via cargo, so it survives `cargo clean` (a symlink into
`target/release` does not):

```sh
# LTO off + 2 jobs avoids a Windows out-of-commit-memory failure (os error 1455) on this machine.
CARGO_PROFILE_RELEASE_LTO=off CARGO_PROFILE_RELEASE_DEBUG=0 \
  cargo install --path apps/photocraft-cli --locked -j 2
```

Installed to `D:/rust/cargo/bin/photocraft-cli.exe` (this machine's `CARGO_HOME` is `D:\rust\cargo`).

### Read-only vs mutating tools

PhotoCraft's own `crates/automation/src/server/conventions.rs` is the source of truth. The read-only
tools (safe to auto-approve) are:

```
command_list  doc_inspect  session_list  render_preview
doc_render_preview  ui_inspect  ui_screenshot  jobs_list
```

Everything else can change state: `command_run`, `command_batch`, `doc_new`, `doc_open`, `doc_save`,
`doc_place`, `doc_close`, `doc_select`, `doc_export_layers`, `doc_export_data_sets`. These are left
to require confirmation.

### Register it in Kiro

Add this to `~/.kiro/settings/mcp.json` under `mcpServers` (also add it to a workspace
`.kiro/settings/mcp.json` if you want it only in some projects). Only the read-only tools are
auto-approved:

```json
"photocraft": {
  "command": "D:/rust/cargo/bin/photocraft-cli.exe",
  "args": [
    "mcp",
    "--headless",
    "--automation-read-root",
    "d:/code/photocraft",
    "--automation-write-root",
    "d:/code/photocraft"
  ],
  "disabled": false,
  "autoApprove": [
    "command_list",
    "doc_inspect",
    "session_list",
    "render_preview",
    "doc_render_preview",
    "ui_inspect",
    "ui_screenshot",
    "jobs_list"
  ]
}
```

> This machine denies agent writes to `~/.kiro/settings/mcp.json`, so the block above is pasted by
> hand. Everything else here is wired automatically.

### Smoke test (what a working server looks like)

```sh
# A read call: list filters (read-only).
photocraft-cli mcp --headless --automation-read-root d:/code/photocraft \
  --automation-write-root d:/code/photocraft   # then from the client: command_list {"filter":"filter."}
```

Through the MCP client the loop is: `doc_new` → `command_run` (an edit) → `doc_inspect` to confirm
the history grew → `command_run {"id":"edit.undo"}` to undo → `doc_inspect` to confirm it shrank.

## 2. Bedrock model

Verified this account reaches Bedrock with a live `converse` call (account `104702104957`, IAM user
`Kwaku`, region `us-east-1`).

**Use the inference-profile id, not the bare foundation-model id.** Newer Claude models on Bedrock
can only be invoked through a profile; the bare `anthropic.claude-*` id throws `ValidationException`
on invoke.

**Exact model id to use:**

```
us.anthropic.claude-sonnet-4-6
```

Other enabled sonnet profiles: `us.anthropic.claude-sonnet-5`, `us.anthropic.claude-sonnet-5-5`,
`us.anthropic.claude-sonnet-4-5-20250929-v1:0` (and `global.` variants). List current ones with:

```sh
aws bedrock list-inference-profiles --region us-east-1
```

## 3. web-search + clean-room steering

The `web-search` MCP server (DuckDuckGo) is already registered and enabled; the agent uses it for
research. `.kiro/steering/photocraft-agent.md` enforces PhotoCraft's `AGENTS.md` clean-room rule:
research for behaviour and specs, but **compose features from built-in commands and never download,
import, or commit third-party assets**. Fonts come from `storytold/craft-fonts`, never committed.

## 4. Fork, upstream, and auto-update

- Fork: `https://github.com/neweracy/photocraft` (origin).
- Upstream: `storytold/photocraft` (fetch only; push URL set to `DISABLE` so upstream can never be
  pushed to by mistake).

```sh
git remote -v
# origin    https://github.com/neweracy/photocraft.git (fetch/push)
# upstream  https://github.com/storytold/photocraft.git (fetch)
# upstream  DISABLE (push)
```

- `.kiro/scripts/auto-update.sh`: fetches `upstream`, and only if HEAD is on `main` with a clean
  tree and no merge in progress, fast-forwards or merges, then rebuilds the CLI. It never pushes,
  never resolves conflicts (aborts on conflict), and runs detached. Try it safely with `--dry-run`.
- `~/.kiro/hooks/photocraft-auto-update.json`: a global SessionStart hook that runs the script.

```sh
# See what it would do without changing anything:
"C:/Program Files/Git/bin/bash.exe" d:/code/photocraft/.kiro/scripts/auto-update.sh --dry-run
```

Log: `.kiro/logs/auto-update.log` (that folder carries its own `.gitignore`, so logs are never
committed).

## Notes / risks

- The auto-update hook rebuilds on every session start when upstream has new commits. It is
  conservative (refuses on a dirty tree or non-`main` branch, aborts on conflict), but a bad
  upstream commit could still break the build; the merge is kept and logged so you can inspect it.
- Keep secrets out of committed files. `~/.kiro/settings/mcp.json` on this machine holds several API
  tokens in plaintext; that file is user-level config and is not part of this repo.
