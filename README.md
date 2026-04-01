# Claude Code Handoff Pattern

A lightweight mechanism for passing context between Claude Code sessions — solving the "short-term amnesia" problem that Memory files don't cover.

## The Problem

If you use Claude Code heavily with multiple agents/sessions, you've probably hit this:

1. You spend 30 minutes working on a task with Claude
2. Context gets long, you `/clear` or exit
3. Start a new session — Claude has **no idea** what just happened
4. You waste 5-10 minutes re-explaining, or Claude goes searching through files trying to reconstruct context

**Memory files** (long-term) work great for rules, patterns, and project state. But they don't capture "what we were doing 5 minutes ago."

**Compact** preserves some context but still occupies the context window and can cause attention drift.

There's a gap between working memory (context) and long-term memory (memory files). **Handoff fills that gap.**

## The Design

```
┌─────────────────────────────────────────────────┐
│              Information Lifecycle               │
├──────────┬──────────┬───────────┬───────────────┤
│ Context  │ Compact  │ Handoff   │ Memory        │
├──────────┼──────────┼───────────┼───────────────┤
│ Current  │ Current  │ Last      │ Permanent     │
│ convo    │ convo    │ session   │ knowledge     │
│          │ (shorter)│ snapshot  │               │
├──────────┼──────────┼───────────┼───────────────┤
│ Lost on  │ Lost on  │ Survives  │ Survives      │
│ clear    │ exit     │ exit,     │ forever       │
│          │          │ overwrit- │               │
│          │          │ ten next  │               │
├──────────┼──────────┼───────────┼───────────────┤
│ Auto     │ Auto     │ On-demand │ Auto (via     │
│ loaded   │ loaded   │ (no cost  │ MEMORY.md     │
│          │          │ if unread)│ index)        │
└──────────┴──────────┴───────────┴───────────────┘
```

Each layer has clear boundaries. No overlap.

## How It Works

### 1. Add the rule to CLAUDE.md

```markdown
## Memory Discipline
- Handoff before exit: before clear/exit/compact, write a handoff note to
  `memory/handoffs/<session>.md`. Overwrite previous content — handoff is
  a snapshot, not a log.
  - Format: `## What I just did` → `## Current status` → `## Next steps`
  - Keep under 20 lines. No frontmatter needed.
  - On-demand read: only load when explicitly asked ("what was X doing?")
- Pre-compact save: before compact/context reset, save any in-session
  learnings to memory AND write handoff. Compact without saving = amnesia.
```

See [`examples/claude-md-snippet.md`](examples/claude-md-snippet.md) for a copy-paste ready version with commentary.

### 2. Create the directory

```bash
mkdir -p ~/.claude/projects/<your-project>/memory/handoffs/
```

### 3. That's it

Before exiting, your agent writes something like:

```markdown
## What I just did
- Migrated data pipeline from cloud function to dedicated VM
- Set up cron jobs + webhook notifications
- Verified end-to-end flow

## Current status
Pipeline is live on VM. Cloud function decommissioned.

## Next steps
- Watch first automated run tomorrow morning
- API token expires in ~30 days — alert will fire
```

Next session, you say "check what was happening last time" → agent reads the file → instant context in 5 seconds instead of 5 minutes.

See [`examples/handoff-template.md`](examples/handoff-template.md) for the blank template and [`examples/handoff-real-example.md`](examples/handoff-real-example.md) for a real-world example.

## Naming: Bind to tmux Sessions, Not Agent Roles

If you run multiple sessions with the same agent (e.g., a "backend" agent in both a `backend-main` session and a `backend-jobs` session), naming handoffs by agent role creates conflicts — who should overwrite `backend.md`?

**Solution: name handoffs after the tmux session.**

```
memory/handoffs/
├── backend-main.md    # Backend agent in the "backend-main" tmux session
├── backend-jobs.md    # Backend agent in the "backend-jobs" tmux session
├── frontend.md        # Frontend agent in the "frontend" tmux session
└── devops.md          # DevOps agent in the "devops" tmux session
```

The agent detects its session name automatically:

```bash
tmux display-message -p '#S'
```

This eliminates the "prisoner's dilemma" — each session owns exactly one file, no coordination needed.

## Multi-Agent Collaboration

Handoffs aren't just for self-continuity. They enable **cross-session awareness**.

### Reading another agent's handoff

When you manage multiple agents via tmux, you often need one agent to know what another just did:

```
You: "check what backend-jobs was doing, then verify the deployment"

Agent reads memory/handoffs/backend-jobs.md → sees the migration is
done → knows to verify the cron jobs, not start from scratch
```

### Real workflow example

```
tmux session: backend-jobs →  Finishes deploying a new service
                               Writes handoff before exit

tmux session: backend-main →  You say "look at backend-jobs's handoff,
                               check if the first run succeeded"
                               Agent reads backend-jobs.md → knows
                               exactly what to check

tmux session: lead         →  You say "what did the backend sessions
                               do today?"
                               Agent reads all handoffs in the directory
                               → gives you a 30-second status briefing
```

### The directory _is_ the team's status board

```bash
ls memory/handoffs/
# backend-main.md  backend-jobs.md  frontend.md  devops.md

# Quick status check across all sessions:
head -3 memory/handoffs/*.md
```

Any agent can read any handoff. The naming convention (tmux session name) makes it unambiguous who wrote what.

## Failure Modes & Fallbacks

### Agent forgets to write handoff

This is the most likely failure. Mitigations:

1. **Pre-compact hook** — If you use Claude Code's `PreCompact` hook, you can add a reminder. But the simplest defense is coupling it with an existing rule: "Pre-compact save" (save memory + write handoff) fires at the same trigger point, so it's one checklist item, not two.

2. **Stale handoff is still useful** — If the agent exits without updating, the previous handoff remains. It's outdated but still better than nothing — it tells you what the session was _last_ known to be doing.

3. **Graceful degradation** — If no handoff exists, the new session falls back to normal behavior: reading memory files, checking git log, scanning recent file changes. Handoff is an accelerator, not a dependency.

### Compact fires but handoff doesn't get written

The `Pre-compact save` rule in CLAUDE.md covers this: it tells the agent to save both memory _and_ handoff before compact. If you want belt-and-suspenders, add it to your PreCompact hook:

```json
{
  "hooks": {
    "PreCompact": [{
      "hooks": [{
        "type": "command",
        "command": "echo 'REMINDER: Write handoff before compact'",
        "timeout": 3
      }]
    }]
  }
}
```

### Handoff conflicts with memory

They can't conflict because they serve different purposes:
- Memory: "Database migrations need epoch filtering" (permanent lesson)
- Handoff: "I was debugging the migration, got to step 3" (ephemeral state)

If you find yourself putting lessons in handoffs, that's a memory. If you find yourself putting session state in memory, that's a handoff.

## Why Not Just Use Memory?

| | Memory | Handoff |
|---|---|---|
| Content | Rules, patterns, lessons | "What I was doing" |
| Lifecycle | Permanent (accumulates) | Ephemeral (overwritten) |
| Written when | Learning something reusable | Leaving a session |
| Loaded | Always (via MEMORY.md index) | Only when asked |
| Context cost | Constant (index always loaded) | Zero (until requested) |

They complement each other. Memory answers "what do we know?" — Handoff answers "what were we doing?"

## File Structure

```
your-project/
└── memory/
    ├── MEMORY.md              # Long-term memory index (auto-loaded)
    ├── some_memory_file.md    # Permanent knowledge
    └── handoffs/              # Session handoffs (on-demand)
        ├── backend-main.md
        ├── backend-jobs.md
        └── frontend.md
```

## FAQ

**Will agents actually remember to write handoffs?**
Same enforcement as any CLAUDE.md rule — it's loaded into every session. In practice, handoffs are simpler than memory (3 sections, fill-in-the-blank), so compliance is higher. Coupling it with the existing "pre-compact save" rule means it's not an extra thing to remember — it's part of the same exit checklist.

**Won't handoff files pile up?**
One file per tmux session, overwritten each time. If you have 6 sessions, you have 6 small files. They don't grow.

**Can I auto-load handoffs on session start?**
You could, but I'd recommend against it. The whole point is zero context cost until needed. If you auto-load, it's just another thing competing for attention in the context window.

**What if I don't use tmux?**
Use any stable session identifier — terminal tab name, a manual name you pass as an env var, or even just a hardcoded name per project. The key insight is: one file per workspace, overwritten not appended.

---

Built while running 5+ Claude Code sessions in parallel. The pattern emerged from a real operational pain point, not from theory.
