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
Architect ←→ Skeptic → Test Smith ←→ Test Critic → Builder ←→ Gatekeeper → Judge
    ↑            ↓          ↑              ↓           ↑            ↓          ↓
    └── iterate ─┘          └── iterate ───┘           └── iterate ─┘    Decision Log
                                                                        (from The Scribe)
```

Each stage feeds the next. But it's not a one-way conveyor belt — **producer-reviewer pairs iterate until the reviewer approves.**

### Review Loops

Three pairs go back and forth until the work is right:

| Producer | Reviewer | They iterate on |
|----------|----------|----------------|
| Architect | Skeptic | The spec — is it clear, complete, and bulletproof? |
| Test Smith | Test Critic | The tests — do they actually prove the spec? |
| Builder | Gatekeeper | The code — is it correct, secure, and shippable? |

The reviewer can send work back as many times as needed. No artificial limits.

### Fresh Eyes Final Pass

After the in-session reviewer approves, a **brand new reviewer** with zero prior context does a cold read. They've never seen the iterations, the discussions, or the compromises. They just read the final artifact and ask: *does this hold up?*

- **If clean**: pipeline advances
- **If they catch something**: it goes back to the producer for one more fix

This mirrors real teams: your primary reviewer iterates with you, then someone else skims it before merge.

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
