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

## Phase 4: Run Pipeline

**You stay running and drive the pipeline sequentially.** No fire-and-forget. You spawn one agent at a time using the `Agent` tool, wait for it to finish, read its output, then spawn the next.

### Pipeline structure

The pipeline is a sequence of **stages**. Some stages are **review pairs** (producer + reviewer). You handle the iteration loop yourself.

**Full pipeline:**
```
1. Architect  ←→  Skeptic (review pair)
2. Test Smith ←→  Test Critic (review pair)
3. Builder    ←→  Gatekeeper (review pair)
4. Judge (solo)
5. Scribe (solo, runs last)
```

**Lite pipeline:**
```
1. Architect (solo)
2. Builder ←→ Gatekeeper (review pair)
```

### 4.1: Running a solo stage

1. Announce the stage to the user (use the entrance line from the quick reference)
2. Spawn the agent with the `Agent` tool:
   - `subagent_type`: "general-purpose"
   - `mode`: "plan" for The Architect (always) and user-designated checkpoints. "default" otherwise.
   - `prompt`: Include the project context, their full persona (from `team-config-personas.md`), input file paths, and instruction to write output to `.claude/pipeline/{slug}.md`
3. Wait for the agent to complete
4. Confirm output file exists. Briefly tell the user what happened (1-2 sentences, not a wall of text).
5. Move to the next stage.

### 4.2: Running a review pair

A review pair has a **producer** and a **reviewer**. They iterate until the reviewer approves (max 3 rounds).

**Round 1:**
1. Announce the producer. Spawn producer agent → wait → read output from `.claude/pipeline/{producer-slug}.md`
2. Announce the reviewer. Spawn reviewer agent with the producer's output file as input → wait → read reviewer's verdict

**If reviewer approves (verdict contains "APPROVED" or "SOLID"):**
3. Spawn a **fresh eyes** reviewer — a new agent with no memory of prior rounds. Use the fresh eyes prompt variant from `team-config-personas.md`. Pass only the final artifact, not the review history.
4. If fresh eyes confirms ("CONFIRMED"): stage is done. Move to next stage.
5. If fresh eyes finds issues: run one more producer round (see "sent back" below), then fresh eyes re-checks. If still issues after that, ask the user what to do.

**If reviewer sends it back (verdict contains "SENT BACK", "NEEDS WORK", or "BLOCKED"):**
3. Show the user a 1-line summary of what the reviewer flagged.
4. Spawn the producer again with: the reviewer's feedback + their previous output + instruction to address the feedback and update `.claude/pipeline/{producer-slug}.md`
5. Spawn the reviewer again with the updated output.
6. Repeat. **Max 3 rounds.** If round 3 and still not approved, ask the user:
   - "The reviewer still has concerns after 3 rounds. Options: (a) Override and move on, (b) Give guidance, (c) One more round"

### 4.3: Checkpoint stages

If a stage is a user-designated checkpoint, **pause after the agent completes** and show the user the output summary. Ask: "Good to proceed?" before moving to the next stage. Use `mode: "plan"` for the agent.

### 4.4: The Scribe

If The Scribe is in the pipeline, spawn it **after the last stage completes** (not in parallel). Give it:
- The project context
- Paths to all pipeline output files (`.claude/pipeline/*.md`)
- Instruction to read each file and produce `.claude/session-notes.md`
- Instruction to archive: copy `session-notes.md` and `team-config.json` to `.claude/sessions/YYYY-MM-DD-[slug]/`

### 4.5: Pipeline status updates

Between stages, show a brief progress line:

```
[2/3] Builder is up next...
```

or for review pairs:

```
[1/3] Architect ←→ Skeptic — round 2...
```

### 4.6: Completion

When all stages are done:

1. If caffeinate was started, kill it: `kill <PID>`
2. Display:

```
===========================================
 DONE. Here's what happened:
===========================================
 [one-line summary per stage]

 Session notes: .claude/session-notes.md
 Config saved: .claude/team-config.json
 Reload: /team-config --load
===========================================
```

---

## Agent prompt template

When spawning each agent, use this structure:

```
**Project**: {project context}

**Your role**: {entrance line}

**Input**: Read {input file path(s)}

**Output**: Write your final output to {output file path}

{full persona from team-config-personas.md}

{for review rounds: "The reviewer sent back your work with this feedback: {feedback}. Address each point, update your output file, and re-submit."}
```

Keep prompts focused. Don't include routing tables or pipeline communication protocols — you're handling the coordination, not the agents.

---

## Cleanup

- **Personas** (`.claude/agents/team-*.md`): gitignored, ephemeral.
- **Pipeline files** (`.claude/pipeline/*.md`): gitignored, ephemeral.
- **Session archiving**: handled by The Scribe (or by you if Scribe isn't in the pipeline — just copy config + notes to `.claude/sessions/`).
- **Config**: stays for `--load`.
