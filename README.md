# /team-config

One command to assemble an agent pipeline with role-based personas for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

**[View the presentation](https://in15.github.io/team-config)**

## What is this?

A command you drop into any repo's `.claude/commands/` directory. Run `/team-config` in Claude Code, and it:

1. **Checks your setup** — verifies agent teams are enabled, offers to turn it on
2. **Keeps your Mac awake** — optional `caffeinate` for long pipelines
3. **Asks what you're building** — interactive setup via quick questions
4. **Lets you pick a mode** — Lite (3 agents), Full (8 agents), or Custom
5. **Assembles a squad** — specialized agent personas with distinct voices
6. **Runs the pipeline** — spawns agents sequentially, manages review loops, reports progress
7. **Saves your config** — reload next time with `/team-config --load`

## Pipeline Modes

| Mode | Agents | Best for |
|------|--------|----------|
| **Lite** (default) | Architect, Builder, Gatekeeper | Simple tasks, fast iteration |
| **Full** | All 8 roles | Complex projects that need to be bulletproof |
| **Custom** | You pick | When you know exactly what you need |

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
Architect <-> Skeptic -> Test Smith <-> Test Critic -> Builder <-> Gatekeeper -> Judge
    |            |           |              |            |            |           |
    +-- iterate -+           +-- iterate ---+            +-- iterate -+     Decision Log
                                                                          (The Scribe)
```

### Review Loops (max 3 rounds)

Producer-reviewer pairs iterate until the reviewer approves:

| Producer | Reviewer | They iterate on |
|----------|----------|----------------|
| Architect | Skeptic | The spec — clear, complete, bulletproof? |
| Test Smith | Test Critic | The tests — do they actually prove the spec? |
| Builder | Gatekeeper | The code — correct, secure, shippable? |

If still unresolved after 3 rounds, you're asked to intervene: override, give guidance, or allow one more round.

### Fresh Eyes Final Pass

After the reviewer approves, a **new reviewer instance** with zero prior context does a cold read. They've never seen the iterations or compromises — just the final artifact.

- **If clean**: pipeline advances
- **If issues found**: goes back to the producer for one more fix

### Architect's Escalation

The Architect runs in plan mode and surfaces every question, concern, and decision point. Questions are batched by topic and tagged by severity:

- **[Blocker]** — can't proceed, asked immediately
- **[Decision]** — multiple valid approaches, batched with related decisions
- **[FYI]** — notable observations, batched at section boundaries

## How It Works

The command drives the pipeline **sequentially** — it spawns one agent at a time, waits for it to complete, reads the output, then spawns the next. For review pairs, it manages the back-and-forth loop and spawns fresh eyes reviewers automatically.

This is simple and reliable:
- No complex inter-agent messaging
- No experimental primitives that might not work
- You see progress as each stage completes
- Checkpoints pause and wait for your approval

## Quick Start

### 1. Copy the files

```bash
mkdir -p .claude/commands
curl -o .claude/commands/team-config.md \
  https://raw.githubusercontent.com/in15/team-config/main/team-config.md
curl -o .claude/commands/team-config-launch.md \
  https://raw.githubusercontent.com/in15/team-config/main/team-config-launch.md
curl -o .claude/commands/team-config-personas.md \
  https://raw.githubusercontent.com/in15/team-config/main/team-config-personas.md
```

### 2. Run it

```
/team-config
```

The command walks you through setup and runs the pipeline.

### 3. Reload next time

```
/team-config --load
```

Shows your saved lineup, lets you confirm or tweak, then runs.

## What gets generated

| File | Lifetime | Purpose |
|------|----------|---------|
| `.claude/agents/team-*.md` | Ephemeral (gitignored) | Agent persona files, regenerated each session |
| `.claude/pipeline/*.md` | Ephemeral (gitignored) | Stage outputs and handoff artifacts |
| `.claude/team-config.json` | Persistent | Saved pipeline config for `--load` |
| `.claude/session-notes.md` | Archived | The Scribe's decision log |
| `.claude/sessions/` | Archived | Past configs and notes |

Agent personas are **ephemeral** — gitignored and regenerated each session. The source of truth lives in `team-config-personas.md`.

## Customization

- **Pick your mode** — Lite for speed, Full for rigor, Custom for control
- **Set checkpoints** — choose which stages pause for your approval
- **Add custom roles** — define your own personas with a codename and description
- **Tweak personas** — edit `team-config-personas.md` to change agent behavior

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with agent teams enabled
- The command checks for `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` and offers to enable it

## License

MIT
