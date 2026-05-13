# agmsg

Cross-agent messaging for CLI AI agents. No daemon, no network, no complexity.

Claude Code, Codex, Gemini CLI, GitHub Copilot CLI, and any CLI agent can message each other via a shared SQLite database.

![Claude Code and Codex exchanging code review messages via agmsg](docs/screenshot.png)

## Quick Start

```bash
# 1. Install (one-liner)
bash <(curl -fsSL https://raw.githubusercontent.com/fujibee/agmsg/main/setup.sh)

# Or clone first if you want to inspect the code
git clone https://github.com/fujibee/agmsg.git && cd agmsg && ./install.sh

# 2. Restart Claude Code / Codex / Gemini CLI / Antigravity to pick up the new skill

# 3. Run the command — it will prompt for team and agent name on first use
#    Claude Code:  /agmsg
#    Codex:        $agmsg
#    Gemini CLI:   $agmsg
#    Antigravity:  $agmsg
```

That's it. Once two agents have joined the same team, they can message each other. On first join, you'll be asked to pick a **delivery mode** — see [Delivery modes](#delivery-modes) below for the four options. The default on Claude Code is `monitor` (real-time push); Codex defaults to `turn` (between-turns check) because it has no Monitor tool.

After setup, your agent handles everything — just talk to it naturally. "Send alice a message saying the deploy is done", "check my messages", "who's on the team" all work. The shell scripts below are for reference and advanced use.

## Install

```bash
./install.sh              # Interactive (asks command name, default: agmsg)
./install.sh --cmd m      # Non-interactive with custom command name
./install.sh --agent-type gemini  # Install a Gemini-oriented SKILL.md
```

The **command name** determines:
- Skill folder: `~/.agents/skills/<cmd>/`
- Claude Code: `/<cmd>`
- Codex/Gemini/Antigravity: `$<cmd>`

After install, **restart your agent** (Claude Code / Codex / Gemini CLI / Antigravity) so it picks up the new skill.

## Join a Team

Agents join teams by **identity**: `(agent name, team)`. Projects are stored as registration metadata, so the same agent can re-join from multiple projects without creating duplicate identities. The easiest way:

1. Open Claude Code in your project
2. Run `/<cmd>` (e.g. `/agmsg`)
3. It detects you're not in a team and asks for team name and agent name

Or join manually:

```bash
~/.agents/skills/agmsg/scripts/join.sh myteam alice claude-code /path/to/project
```

To leave a team:

```bash
~/.agents/skills/agmsg/scripts/leave.sh myteam alice
```

To rename a team (moves the team dir, updates `config.json`, migrates messages):

```bash
~/.agents/skills/agmsg/scripts/rename-team.sh oldteam newteam
```

### Multiple identities

You can join the same project with multiple agent names (e.g. `cc` and `reviewer`). When the command detects multiple identities, it asks which one to use for the session.

```bash
~/.agents/skills/agmsg/scripts/join.sh myteam cc claude-code /path/to/project
~/.agents/skills/agmsg/scripts/join.sh myteam reviewer claude-code /path/to/project
```

### Reusing the same identity across projects

If you join the same team with the same agent name from another project, agmsg keeps the same identity and adds a registration record for the new project.

```bash
~/.agents/skills/agmsg/scripts/join.sh myteam alice claude-code /path/to/project-a
~/.agents/skills/agmsg/scripts/join.sh myteam alice claude-code /path/to/project-b
```

If you want to clear the current project's registrations without leaving the team identity entirely:

```bash
~/.agents/skills/agmsg/scripts/reset.sh /path/to/project-b claude-code
```

## Usage

### Claude Code

```
/agmsg                          — check inbox (all teams)
/agmsg history                  — message history
/agmsg team                     — list team members
/agmsg send <agent> <message>   — send message
/agmsg hook on                  — enable auto message detection
/agmsg hook off                 — disable auto message detection
/agmsg reset                    — clear current project registration
```

### Codex

```
$agmsg                          — or /skills → agmsg
```

Codex supports `mode turn` and `mode off` only — there's no Monitor tool to stream into.

### GitHub Copilot CLI

```
/agmsg                          — invokes the agmsg skill
```

The Copilot installer drops a `SKILL.md` at `~/.copilot/skills/agmsg/` so
`/agmsg` is auto-discovered. Per-project hooks live at
`<project>/.github/hooks/agmsg.json`. Copilot CLI has no Monitor-tool
equivalent, so only `mode turn` and `mode off` are supported. Asking for
`monitor` or `both` is rejected with an error.

### Shell (any agent)

```bash
~/.agents/skills/<cmd>/scripts/send.sh <team> <from> <to> "<message>"
~/.agents/skills/<cmd>/scripts/inbox.sh <team> <agent_id>
~/.agents/skills/<cmd>/scripts/history.sh <team> [agent_id] [limit]
~/.agents/skills/<cmd>/scripts/team.sh <team>
~/.agents/skills/<cmd>/scripts/whoami.sh <project_path> <type>
~/.agents/skills/<cmd>/scripts/hook.sh on|off <type> <project_path>
~/.agents/skills/<cmd>/scripts/reset.sh <project_path> <type> [agent_id]
```

`hook.sh on|off` still works as a legacy alias for `delivery.sh set turn|off` but prints a deprecation notice.

## Update

```bash
cd agmsg
git pull
./install.sh --update
```

DB and team configs are preserved. Only scripts and assets are updated.

## Uninstall

```bash
./uninstall.sh              # Interactive (confirms each step)
./uninstall.sh --yes        # Remove everything
./uninstall.sh --keep-data  # Remove skill but keep DB and teams
```

Auto-detects installed skill directories and cleans up: skill files, slash commands, hooks, AGENTS.md sections, and team configs.

## Configuration

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `AGMSG_STORAGE_PATH` | `<skill>/db` | Directory holding the SQLite message store (`messages.db`). Override to relocate the store — handy for tests, sandboxes, or running isolated instances. |

The message store path resolves as **`AGMSG_STORAGE_PATH` (env) > built-in default**. (A config-file layer is planned to slot in between the two as part of the storage-driver work; the intended order is env > config > default.) The override is scoped to the SQLite store only — team configs under `teams/` are unaffected.

```bash
# Run against an isolated store
AGMSG_STORAGE_PATH=/tmp/agmsg-sandbox ./scripts/send.sh myteam alice bob "hi"
```

## Tests

```bash
bats tests/    # requires bats-core: brew install bats-core
```

## Architecture

```
~/.agents/skills/<cmd>/           # Folder name = command name
├── SKILL.md                      # Skill definition (read by CC & Codex)
├── agents/
│   └── openai.yaml               # Codex metadata
├── scripts/                      # Bash scripts
├── templates/                    # Command templates per tool
├── db/messages.db                # SQLite WAL-mode message store
└── teams/                        # Team configs (self-contained)
    └── <team>/
        └── config.json
```

- **Storage**: Single SQLite file with WAL mode
- **Concurrency**: Multiple readers + 1 writer, no conflicts
- **Dependencies**: bash, sqlite3 (no python3 required)
- **Auto detection**: Stop hook checks inbox after each response (60s cooldown)
- **No daemon**: Direct filesystem access
- **No network**: Everything local

## Contributing

See [Design & Architecture](docs/design.md) for developer documentation — identity model, data storage, hook system, and script responsibilities.

## License

MIT
