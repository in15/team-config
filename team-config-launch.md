# /team-config — Launch Instructions

This file is read by `/team-config` after Phase 1 (user preferences) is complete. Do not use this file directly.

---

## Phase 2: Generate Agent Files

For each selected role, write a file to `.claude/agents/team-{role-slug}.md`.

These are **ephemeral** (gitignored, regenerated each session). Slug format: lowercase, hyphens (e.g., "The Architect" → "architect").

### Agent File Format

```markdown
---
name: team-{role-slug}
description: {one-line description}
model: inherit
---

{persona prompt from team-config-personas.md}
```

- Read `.claude/commands/team-config-personas.md` for full persona prompts
- Only generate files for selected roles
- For custom roles, flesh out the user's description to match preset persona detail — give it a codename, entrance line, and personality
- Overwrite existing files silently

---

## Phase 3: Save Config

Write `.claude/team-config.json`:

```json
{
  "mode": "lite|full|custom",
  "pipeline": ["architect", "builder", "gatekeeper"],
  "checkpoints": ["gatekeeper"],
  "customRoles": [],
  "noteTaker": false,
  "projectContext": "user's project description",
  "createdAt": "YYYY-MM-DD"
}
```

---

## Phase 4: Launch Pipeline

Creates the team, sets up tasks, spawns agents, and **exits**. Agents self-route from here.

### 4.1: Create Team

`TeamCreate` with a slug derived from the project context (e.g., "api-endpoint-squad").

### 4.2: Create Pipeline Tasks

`TaskCreate` in pipeline order with `addBlockedBy` for sequencing.

**Lite:**
```
Task 1: Architect (no blockers)
Task 2: Builder (blockedBy: [1])
Task 3: Gatekeeper (blockedBy: [2])
```

**Full:**
```
Task 1: Architect (no blockers)
Task 2: Skeptic (blockedBy: [1])
Task 3: Test Smith (blockedBy: [2])
Task 4: Test Critic (blockedBy: [3])
Task 5: Builder (blockedBy: [4])
Task 6: Gatekeeper (blockedBy: [5])
Task 7: Judge (blockedBy: [6])
Scribe: no blockers (parallel)
```

### 4.3: Spawn Agents

For each role, use the `Task` tool:

- `subagent_type`: "general-purpose"
- `name`: "team-{role-slug}"
- `mode`: "plan" for The Architect (always) and checkpoint stages. "default" otherwise.
- `prompt`: project context + full persona (from `team-config-personas.md`) + routing (see 4.4) + entrance line

### 4.4: Pipeline Routing

Inject into each agent's prompt:

1. **Write output to**: `.claude/pipeline/{slug}.md`
2. **Read input from**: previous stage's output file
3. **Notify next**: next agent via `SendMessage`
4. **Review partner**: who to iterate with (if applicable)

**Review pairs self-manage**: max 3 rounds via `SendMessage`, then escalate to user. Reviewer spawns fresh eyes after approving. Reviewer marks task complete and notifies next.

**Full routing table:**

| Agent | Reads from | Writes to | Notifies | Review partner |
|-------|-----------|-----------|----------|----------------|
| Architect | (request) | pipeline/architect.md | team-skeptic | team-skeptic |
| Skeptic | pipeline/architect.md | — | team-test-smith | team-architect |
| Test Smith | pipeline/architect.md | pipeline/test-smith.md | team-test-critic | team-test-critic |
| Test Critic | pipeline/test-smith.md | — | team-builder | team-test-smith |
| Builder | blueprint + tests | pipeline/builder.md | team-gatekeeper | team-gatekeeper |
| Gatekeeper | pipeline/builder.md | — | team-judge | team-builder |
| Judge | all pipeline files | pipeline/judge.md | (done) | — |
| Scribe | monitors all | session-notes.md | (done) | — |

**Lite routing**: Architect → Builder → Gatekeeper. Gatekeeper reviews Builder's code.

If caffeinate PID was noted, include in the last agent's prompt: `kill <PID>` when pipeline completes.

### 4.5: Report and Exit

```
===========================================
 THE SQUAD IS ASSEMBLED. LET'S BUILD.
===========================================
 [pipeline with arrows]
 Checkpoints at: [list]
 Config saved to .claude/team-config.json
 Agents: self-routing, no orchestrator needed
 Check progress: /team-config --status
 Next time: /team-config --load
===========================================
```

**Done.** Agents handle everything from here.

---

## Cleanup

Handled by agents, not this command.

- **Personas** (`.claude/agents/team-*.md`): gitignored, ephemeral.
- **Pipeline files** (`.claude/pipeline/*.md`): gitignored, ephemeral.
- **Session archiving**: The Scribe archives to `.claude/sessions/YYYY-MM-DD-[slug]/` after Judge's verdict.
- **Config**: stays for `--load`.
