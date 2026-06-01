# OpenWebUI Claude Code Pipe

Run [Claude Code](https://docs.claude.com/en/docs/claude-code/overview)'s agent loop from inside [Open WebUI](https://github.com/open-webui/open-webui) chats, via the [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python).

This is an Open WebUI **Pipe** that exposes Claude Code as a selectable model. Each chat gets its own isolated workspace directory; agent turns within the same chat resume the same Claude Code session, so context (files, prior tool calls) carries forward.

## Features

- **Full Claude Code agent loop** — Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch (configurable allowlist)
- **Per-chat workspaces** — each `chat_id` gets a sandboxed working directory that persists across turns
- **Dual auth** — bring your own Anthropic **API key** (pay-per-token) *or* a **Claude Pro/Max OAuth token** (bills against your subscription)
- **Streaming UI** — tool calls render inline with previews; generated images/PDFs/CSVs surface as artifacts in the chat
- **Configurable valves** — model, permission mode, tool allowlist, max turns, workspace root, setting sources (`CLAUDE.md`)

## Requirements

- Open WebUI (any recent version with the Pipes/Functions framework)
- Python deps (auto-installed by Open WebUI from the file header):
  - `claude-agent-sdk>=0.1.60`
  - `anthropic>=0.40.0`
- The `claude` CLI must be available on the host running Open WebUI's Python backend (the SDK shells out to it). Install via `npm install -g @anthropic-ai/claude-code`.

## Installation

1. In Open WebUI, go to **Workspace → Functions → +** (or **Admin Panel → Functions**).
2. Paste the contents of [`claude_agent_pipe.py`](./claude_agent_pipe.py) into the editor.
3. Save and enable the function.
4. Open the function's **Valves** and configure auth (one of):
   - `ANTHROPIC_API_KEY` — standard pay-per-token billing
   - `CLAUDE_CODE_OAUTH_TOKEN` — generate on a machine with a browser via `claude setup-token`; bills against your Pro/Max/Team subscription
5. A new model named **Claude Code** will appear in the model picker.

## Configuration (Valves)

| Valve | Default | Description |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | *(env)* | Anthropic API key. Falls back to the backend's env var. |
| `CLAUDE_CODE_OAUTH_TOKEN` | *(empty)* | Claude subscription OAuth token. Takes priority over the API key when set. |
| `MODEL` | `claude-haiku-4-5` | Claude model ID (e.g. `claude-haiku-4-5`, `claude-sonnet-4-6`, `claude-opus-4-7`). |
| `PERMISSION_MODE` | `bypassPermissions` | `default`, `acceptEdits`, `bypassPermissions`, `plan`, or `dontAsk`. |
| `ALLOWED_TOOLS` | `Read,Write,Edit,Bash,Glob,Grep,WebSearch,WebFetch` | Comma-separated tools auto-approved without prompting. |
| `WORKDIR_ROOT` | `/tmp/claude-agent-pipe` | Root directory for per-chat workspaces. |
| `MAX_TURNS` | `30` | Max agent turns per user message. `0` disables the cap. |
| `SETTING_SOURCES` | *(empty)* | Comma-separated filesystem setting sources to load: `user`, `project`, `local`. Empty = none (isolated baseline). See below. |

## Persistent context via `CLAUDE.md` (`SETTING_SOURCES`)

By default the pipe passes `setting_sources=[]` to the SDK, so **no** filesystem
settings are loaded: each chat starts from a clean baseline and does **not**
inherit the backend user's `~/.claude/` or the workdir's `.claude/`. This is the
safe default for shared deployments.

If you run a single-user/homelab instance and want persistent environmental
context (e.g. a host inventory or standing instructions in
`~/.claude/CLAUDE.md`) without re-explaining it every chat, set the valve:

| Value | Loads |
| --- | --- |
| *(empty)* | Nothing — isolated baseline (default). |
| `user` | `~/.claude/CLAUDE.md` **and** `~/.claude/settings.json`. |
| `user,project,local` | Above plus the workdir's `.claude/settings.json` and `.claude/settings.local.json`. |

Each token maps to one source — `user` → `~/.claude/`, `project` →
`<workdir>/.claude/settings.json`, `local` → `<workdir>/.claude/settings.local.json`.
Unknown tokens are dropped.

> [!WARNING]
> **Settings sources load more than `CLAUDE.md`.** A loaded `settings.json` can
> define **hooks that execute shell commands**, permission grants, env vars, and
> MCP servers — for *every chat*, under the backend user's identity, with the
> pipe's default `bypassPermissions` mode. Only enable `SETTING_SOURCES` on an
> instance you fully trust and control. **Do not enable it on multi-user or
> public deployments** — it breaks per-chat isolation and lets host config
> influence (or run code in) every user's session. There is no way to load
> `CLAUDE.md` *without* also loading `settings.json` from the same source; that
> coupling is in Claude Code, not this pipe.

### Does this apply to the sandboxed pipe?

Not directly. [`claude_agent_pipe_sandboxed.py`](./claude_agent_pipe_sandboxed.py)
doesn't use `setting_sources` at all — it shells the `claude` CLI inside an
open-terminal sandbox with a **per-chat `CLAUDE_CONFIG_DIR`** that's created
fresh each chat, so there's no host `~/.claude/` to inherit and nothing to
disable. Persistent context there is meant to come from mechanisms already built
for isolation:

- **Workspace Model system prompt** — appended on every turn (`--append-system-prompt`); the natural place for standing instructions.
- **Baking into the image** — skills are already vendored into the sandbox image at build time; a `CLAUDE.md` or settings can be baked the same way (under the per-chat `CLAUDE_CONFIG_DIR` layout) if you want file-based context.

Because the sandbox isolates the agent and a proxy holds the credentials, the
security tradeoff is far milder there — but the `setting_sources` valve itself
has nothing to act on, so it's intentionally **not** added to the sandboxed pipe.

## Auth notes

When both auth methods are present, the OAuth token wins and the API key is unset before invoking the SDK so it can't override.

Per Anthropic's terms: a Claude subscription is for personal use — **don't re-offer subscription auth to other end users** through a shared Open WebUI deployment. For multi-user setups, use API keys.

## License

MIT
