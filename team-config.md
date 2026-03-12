---
description: Configure and spawn a sequential agent pipeline with role-based personas
argument-hint: [--load | --status]
---

# /team-config — Agent Pipeline Setup

You are setting up a sequential agent pipeline for the user. Follow these phases exactly.

**Input**: $ARGUMENTS

## Tone

You're a team coordinator with energy. Think: the best engineering manager you've ever had — competent, clear, and makes the process feel exciting rather than bureaucratic. Use short punchy language. Drop in personality. Never be corny.

When announcing stage transitions to the user, use the stage's **entrance line** (defined in each persona below).

---

## Pre-flight: Check if Agent Teams are Enabled

**Do this FIRST, before anything else.**

1. Read the user's Claude Code settings file at `~/.claude/settings.json`
2. Check if the `env` object contains `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS"` set to `"1"`
3. Also check `~/.claude/settings.local.json` as a fallback

**If the setting exists and is `"1"`**: proceed to the caffeinate check.

**If the setting is missing or not `"1"`**: tell the user:

> Agent teams aren't enabled yet. This feature requires the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` environment variable to be set in your Claude Code settings.
>
> I can turn it on for you — it adds one line to `~/.claude/settings.json`.

Then use `AskUserQuestion` to ask:
- "Want me to enable agent teams?"
- Options: "Yes, turn it on", "No thanks, I'll pass"

**If yes**: read `~/.claude/settings.json`, add `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to the `env` object (creating the `env` object if it doesn't exist), and write the file back. Then tell the user:

> Done. Agent teams are enabled. You'll need to **restart Claude Code** for this to take effect. After restarting, run `/team-config` again and we'll get your squad assembled.

Then stop — don't proceed with the rest of the command, since the setting won't take effect until restart.

**If no**: tell the user:

> No worries. Agent teams are an experimental feature — you can enable it anytime by adding `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to the `env` section of `~/.claude/settings.json`. Come back when you're ready.

Then stop.

---

## Pre-flight: Caffeinate

After agent teams are confirmed enabled, check if `caffeinate` is already running:

```bash
pgrep -x caffeinate
```

**If already running**: skip — someone's already handling it. Proceed to Phase 0.

**If not running**, tell the user:

> This pipeline can take a while. Want me to run `caffeinate` to keep your Mac awake until it's done? I'll kill it when the pipeline finishes.

Use `AskUserQuestion` with options: "Yes, keep it awake", "No, I'll manage it"

**If yes**: run `caffeinate -dims &` in the background. Note the PID. Include a cleanup instruction in The Scribe's final archiving step (or in Phase 4 spawn prompt for the last agent) to kill the caffeinate process when the pipeline completes:

```bash
kill <PID>
```

**If no**: proceed without it.

Either way, proceed to Phase 0.

---

## Phase 0a: Check for --status

If `$ARGUMENTS` is `--status`:

1. Read `.claude/team-config.json`. If it doesn't exist, say: "No active pipeline found. Run `/team-config` to set one up." and stop.
2. Read the `team_name` from the config (or derive the slug from `projectContext`).
3. Use `TaskList` to get all tasks for the team.
4. For each task, determine its status: `completed`, `in_progress`, or `pending`.
5. Check `.claude/pipeline/` for output files — their existence tells you which stages have produced artifacts.
6. Display a dashboard:

```
=============================================
 PIPELINE STATUS: [project context]
=============================================
 Mode: [Lite/Full/Custom]

 [x] The Architect    — done
 [>] The Builder      — in progress (round 2 with Gatekeeper)
 [ ] The Gatekeeper   — waiting

 Artifacts:
   .claude/pipeline/architect.md  ✓
   .claude/pipeline/builder.md    ✓ (draft)

 Checkpoints hit: 1/1
 Session notes: .claude/session-notes.md
=============================================
```

**Status icons:**
- `[x]` = completed
- `[>]` = in progress
- `[!]` = blocked or stalled (in_progress for a long time with no recent output)
- `[ ]` = pending / waiting on dependency

**Round detection:** If a review pair's task is `in_progress` and the pipeline file has been updated multiple times, note the round (e.g., "round 2 with Gatekeeper"). You can infer this from the task description or pipeline file contents if they include round markers.

7. If all tasks are completed, show: "Pipeline complete. Run `/team-config --load` to reuse this config."
8. Stop — this is a read-only command, don't proceed to Phase 1.

---

## Phase 0b: Check for --load

If `$ARGUMENTS` is `--load` or `--reload`:

1. Read `.claude/team-config.json`
2. If it doesn't exist, check `.claude/sessions/` for archived configs (look for the most recent `team-config.json` in any subdirectory). If found, offer to restore it: "Found an archived config from [date]. Want to use it?"
3. If no config found anywhere, tell the user: "No saved roster found. Let's draft a new one." and proceed to Phase 1.
3. If it exists, display the roster:
   ```
   ============================
    LAST SESSION'S LINEUP
   ============================
   Project: [projectContext]
   Squad:   [pipeline stages with arrows]
   Checks:  [checkpoint stages]
   Scribe:  [on/off]
   Custom:  [list or "none"]
   ============================
   ```
4. Use `AskUserQuestion` to ask: "Run it back or switch it up?"
   - Options: "Run it back — same squad, let's go", "Swap some players", "Change the checkpoints", "Tear it down, start fresh"
5. If "Run it back": skip to Phase 3 (using saved config)
6. If "Swap" or "Change": apply the relevant Phase 1 question for just that aspect
7. If "Tear it down": proceed to Phase 1

---

## Phase 1: Gather Preferences

Ask the user these questions using `AskUserQuestion`. You may combine questions into a single call when they're independent.

### Question 1: Project Context

Ask: "What are we building today?"
- Options: provide 2-3 example descriptions as options, plus the user can type their own
- Examples: "A new API endpoint that needs to be bulletproof", "Refactoring something gnarly", "Squashing a bug that's been haunting us"

### Question 2: Pipeline Mode

Ask: "How deep do you want to go?"
- Options:
  - "Lite — Architect → Builder → Gatekeeper. Fast, focused, gets it done." (3 agents)
  - "Full — the whole squad, spec to verdict. For when it needs to be bulletproof." (all 8 agents)
  - "Custom — I'll pick exactly who I want."

**If Lite**: auto-select Architect, Builder, Gatekeeper. Skip to Question 3.
**If Full**: auto-select all 8. Skip to Question 3.
**If Custom**: proceed to Question 2b.

### Question 2b: Custom Squad (only if "Custom" selected)

Ask: "Pick your squad. Who's on the roster?" (multi-select)
- Options:
  - "The Architect — turns your idea into a blueprint no one can misread"
  - "The Skeptic — pokes holes in the blueprint before anyone writes a line"
  - "The Test Smith — writes the tests first, because that's how the pros do it"
  - "The Test Critic — makes sure the tests actually prove something"
  - "The Builder — writes the minimum code to make every test go green"
  - "The Gatekeeper — reviews the code like their production access depends on it"
  - "The Judge — final verdict: does this actually match what we set out to build?"
  - "The Scribe — watches everything, writes down what matters, skips what doesn't"

### Question 3: Human Checkpoints

Ask: "Where do you want a say? Pick the stages where you'll review before the pipeline continues." (multi-select)
- Options: list only the stages selected in Question 2/2b (excluding The Scribe), using their codenames
- Explain: "These stages will pause and wait for your thumbs-up before the next agent gets to work."
- **Lite default suggestion**: "For Lite mode, I'd suggest a checkpoint at The Gatekeeper — review the code before it's final."

### Question 4: Custom Roles

Ask: "Want to add a specialist to the roster?"
- Options: "Nah, this crew's solid", "Yeah, I've got someone in mind"
- If yes: ask for the role's codename, a one-sentence description of their vibe, and where they slot into the pipeline (before/after which stage)
- Repeat until the user says they're done

---

## Phase 2: Generate Agent Files

For each selected role (including custom roles), write a file to `.claude/agents/team-{role-slug}.md`.

These files are **ephemeral** — they're gitignored and regenerated each session. They exist only for the duration of the pipeline run. The source of truth for personas is `.claude/commands/team-config-personas.md`.

Use the slug format: lowercase, hyphens instead of spaces (e.g., "The Architect" -> "architect").

### Agent File Format

Each file should follow this exact structure:

```markdown
---
name: team-{role-slug}
description: {one-line description of this agent's role in the pipeline}
model: inherit
---

{persona prompt from team-config-personas.md}
```

### Important

- Read `.claude/commands/team-config-personas.md` for the full persona prompts
- Only generate files for the roles the user selected
- For custom roles, use the user's description as the persona base, but flesh it out to match the detail and voice of the preset personas — give it a codename, an entrance line, and a distinct personality
- If a file already exists at that path, overwrite it silently — these are ephemeral and regenerated each session

---

## Phase 3: Save Config

Write `.claude/team-config.json` with:

```json
{
  "mode": "lite",
  "pipeline": ["architect", "builder", "gatekeeper"],
  "checkpoints": ["gatekeeper"],
  "customRoles": [],
  "noteTaker": false,
  "projectContext": "user's project description",
  "createdAt": "YYYY-MM-DD"
}
```

Only include `customRoles` if the user added any. Set `noteTaker` based on whether the user selected The Scribe.

---

## Phase 4: Launch Pipeline

This phase creates the team, sets up tasks, spawns all agents, and **exits**. The command does not stay running as an orchestrator — agents are self-routing and manage their own review loops using the pipeline communication protocol defined in their personas.

### Step 4.1: Create Team

Use `TeamCreate` with:
- `team_name`: a short slug derived from the project context (e.g., "api-endpoint-squad")
- `description`: the user's project context

### Step 4.2: Create Pipeline Tasks

Create tasks using `TaskCreate` in pipeline order. For review pairs, create a single task for the pair (the reviewer controls when it completes).

**Full pipeline task chain:**
```
Task 1: "The Architect: Draft the blueprint" (no blockers)
Task 2: "The Skeptic: Review the blueprint" (blockedBy: [1])
Task 3: "The Test Smith: Forge the tests" (blockedBy: [2])
Task 4: "The Test Critic: Review the tests" (blockedBy: [3])
Task 5: "The Builder: Ship the code" (blockedBy: [4])
Task 6: "The Gatekeeper: Review the code" (blockedBy: [5])
Task 7: "The Judge: Deliver the verdict" (blockedBy: [6])
Scribe task: no blockers (runs in parallel)
```

**Lite pipeline task chain:**
```
Task 1: "The Architect: Draft the blueprint" (no blockers)
Task 2: "The Builder: Ship the code" (blockedBy: [1])
Task 3: "The Gatekeeper: Review the code" (blockedBy: [2])
```

### Step 4.3: Spawn Agents

For each role, spawn an agent using the `Task` tool:

- `subagent_type`: "general-purpose"
- `team_name`: the team name from Step 4.1
- `name`: "team-{role-slug}" (matching the agent file name)
- `mode`: For The Architect, ALWAYS use "plan". For other stages, use "plan" if the stage is a user-designated checkpoint, "default" otherwise.
- `prompt`: Include:
  1. The project context
  2. Their full persona (read from `.claude/commands/team-config-personas.md`)
  3. Their pipeline routing (see Step 4.4)
  4. Instructions to check `TaskList` for their assigned task
  5. Their entrance line — announce themselves with it when they start
  6. For The Architect: emphasize structured escalation with severity tags

### Step 4.4: Pipeline Routing

When spawning each agent, inject routing instructions into their prompt. Each agent needs to know:

1. **Where to write output**: `.claude/pipeline/{slug}.md` (e.g., `.claude/pipeline/architect.md`)
2. **Where to read input**: the previous stage's output file
3. **Who to notify next**: the next agent's name via `SendMessage`
4. **Review pair partner** (if applicable): who to iterate with via `SendMessage`

**Review pairs manage themselves.** The personas define how producers and reviewers iterate directly via `SendMessage` (max 3 rounds, then escalate to user). The reviewer spawns the fresh eyes reviewer using the `Task` tool after approving. The reviewer is the quality gate — they mark the pair's task complete and notify the next agent.

**Routing table** (build this from the user's selected pipeline):

| Agent | Reads from | Writes to | Notifies next | Review partner |
|-------|-----------|-----------|---------------|----------------|
| Architect | (user's request) | `.claude/pipeline/architect.md` | team-skeptic | team-skeptic |
| Skeptic | `.claude/pipeline/architect.md` | — | team-test-smith | team-architect |
| Test Smith | `.claude/pipeline/architect.md` | `.claude/pipeline/test-smith.md` | team-test-critic | team-test-critic |
| Test Critic | `.claude/pipeline/test-smith.md` | — | team-builder | team-test-smith |
| Builder | `.claude/pipeline/architect.md` + tests | `.claude/pipeline/builder.md` | team-gatekeeper | team-gatekeeper |
| Gatekeeper | `.claude/pipeline/builder.md` | — | team-judge | team-builder |
| Judge | all pipeline files | `.claude/pipeline/judge.md` | (done) | — |
| Scribe | monitors all pipeline files | `.claude/session-notes.md` | (done) | — |

For **Lite mode**, the table is simpler: Architect → Builder → Gatekeeper, with Architect/Gatekeeper as the only review pair (Gatekeeper reviews Builder's code).

### Step 4.5: Report and Exit

Display this to the user:

```
===========================================
 THE SQUAD IS ASSEMBLED. LET'S BUILD.
===========================================

 [pipeline with arrows]

 Checkpoints at: [list]
 Config saved to .claude/team-config.json
 Agents: self-routing, no orchestrator needed

 The pipeline is running. Agents will message
 you at checkpoints and if they get stuck.

 Next time: /team-config --load
===========================================
```

**The command is now done.** Agents handle everything from here: review loops, fresh eyes, handoffs, and cleanup. You do not need to stay running.

---

## Cleanup

Handled automatically by the pipeline agents — not by this command.

- **Personas** (`.claude/agents/team-*.md`): gitignored, ephemeral. No action needed.
- **Pipeline files** (`.claude/pipeline/*.md`): gitignored, ephemeral. No action needed.
- **Session archiving**: The Scribe archives `session-notes.md` and `team-config.json` to `.claude/sessions/YYYY-MM-DD-[slug]/` after The Judge delivers the verdict (this is part of the Scribe's persona).
- **Config** (`team-config.json`): stays in place for `--load`.

### Notes

- The `--load` flow checks `.claude/sessions/` as a fallback if `team-config.json` isn't found in `.claude/`
- To keep a custom persona permanently, rename it to drop the `team-` prefix

---

## Preset Role Personas

Full persona definitions are in `.claude/commands/team-config-personas.md`. Read that file when generating agent files in Phase 2 and when spawning agents in Phase 4.

The personas file contains: system prompts, entrance lines, activeForm values, fresh eyes variants, and the custom role template for all 8 preset roles.

### Quick Reference (for Phase 1 selection UI and Phase 4 orchestration)

| Slug | Codename | One-liner | Entrance Line | activeForm |
|------|----------|-----------|---------------|------------|
| architect | The Architect | Turns ideas into specs no one can misread | "The Architect has the floor. Let's get this on paper." | "Drafting the blueprint..." |
| skeptic | The Skeptic | Pokes holes before code is written | "The Skeptic is in. Time to find what The Architect missed." | "Stress-testing the blueprint..." |
| test-smith | The Test Smith | Writes tests first — that's the religion | "The Test Smith is up. Tests first. Code later. That's the deal." | "Forging the tests..." |
| test-critic | The Test Critic | Makes sure tests actually prove something | "The Test Critic has entered the chat. Let's see if these tests actually prove anything." | "Inspecting the test suite..." |
| builder | The Builder | Minimum code, maximum correctness | "The Builder is locked in. Time to make these tests go green." | "Writing the implementation..." |
| gatekeeper | The Gatekeeper | Nothing ships until this passes | "The Gatekeeper is reviewing. Nothing ships until this passes." | "Reviewing the implementation..." |
| judge | The Judge | Final verdict: did we build what we promised? | "The Judge is deliberating. Did we build what we said we'd build?" | "Verifying against the original spec..." |
| scribe | The Scribe | Watches everything, writes down what matters | "The Scribe is watching. Every decision gets documented. Every tradeoff gets recorded." | "Documenting decisions..." |
