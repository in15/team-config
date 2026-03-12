# /team-config

One command to assemble an agent pipeline with role-based personas for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

**[View the presentation](https://in15.github.io/team-config)**

## What is this?

A single file you drop into any repo's `.claude/commands/` directory. Run `/team-config` in Claude Code, and it:

1. **Asks what you're building** — interactive setup via quick questions
2. **Assembles a squad** — 8 specialized agent personas with distinct voices
3. **Runs a pipeline** — each stage hands off to the next, sequentially
4. **Pauses where you want** — configurable human checkpoints between stages
5. **Documents everything** — a Scribe agent produces a concise decision log
6. **Saves your config** — reload next time with `/team-config --load`

## The Squad

| # | Codename | Role |
|---|----------|------|
| 1 | **The Architect** | Turns your idea into a spec no one can misread |
| 2 | **The Skeptic** | Pokes holes in the spec before anyone writes code |
| 3 | **The Test Smith** | TDD — writes failing tests from the approved spec |
| 4 | **The Test Critic** | Makes sure the tests actually prove something |
| 5 | **The Builder** | Writes the minimum code to make every test green |
| 6 | **The Gatekeeper** | Reviews the code like production depends on it |
| 7 | **The Judge** | Final verdict — did we build what we promised? |
| 8 | **The Scribe** | Watches everything, documents decisions and tradeoffs |

## The Pipeline

```
Architect → Skeptic → Test Smith → Test Critic → Builder → Gatekeeper → Judge
                                                                          ↓
                                                                    Decision Log
                                                                   (from The Scribe)
```

Each stage produces output that feeds the next. The Scribe observes in parallel and writes a session log designed to be absorbed in 60 seconds.

## Quick Start

### 1. Copy the command file

```bash
# From your repo root
mkdir -p .claude/commands
curl -o .claude/commands/team-config.md \
  https://raw.githubusercontent.com/in15/team-config/main/team-config.md
```

Or just copy `team-config.md` into `.claude/commands/` manually.

### 2. Run it

```
/team-config
```

That's it. The command walks you through setup and launches the pipeline.

### 3. Reload next time

```
/team-config --load
```

Shows your saved lineup, lets you confirm or tweak, then spawns.

## What gets generated

When you run `/team-config`, it creates:

| File | Purpose |
|------|---------|
| `.claude/agents/team-*.md` | Agent persona files (one per role) |
| `.claude/team-config.json` | Saved pipeline config for reuse |
| `.claude/session-notes.md` | The Scribe's decision log |

Agent files and config are meant to be **committed** — shared with your team so everyone uses the same personas.

## Customization

- **Pick your stages** — use all 8 or just the ones you need
- **Set checkpoints** — choose which stages pause for your approval
- **Add custom roles** — define your own personas with a codename and description
- **Extend personas** — edit the generated `.claude/agents/team-*.md` files directly

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with team/agent support
- That's it. No dependencies, no build step, no config files to set up.

## License

MIT
