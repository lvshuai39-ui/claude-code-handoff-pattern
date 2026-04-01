# CLAUDE.md Snippet — Handoff Rules

Copy this into your project's `CLAUDE.md` under your Memory Discipline section.

```markdown
## Memory Discipline

# ... your existing memory rules ...

- Handoff before exit: before clear/exit/compact, write a handoff note to
  `memory/handoffs/<session>.md`. File name = tmux session name (use
  `tmux display-message -p '#S'` to detect). Overwrite previous content —
  handoff is a snapshot, not a log.
  - Format: `## What I just did` (bullet list) → `## Current status`
    (1-2 lines) → `## Next steps` (bullet list)
  - Keep under 20 lines. No frontmatter needed.
  - On-demand read: when user says "check what X was doing" or "pick up
    where we left off", read `memory/handoffs/<session>.md`
- Pre-compact save: before compact/context reset, save any in-session
  learnings to memory AND write handoff. Compact without saving = amnesia.
```

## Why these two rules work together

`Pre-compact save` ensures long-term lessons survive (→ memory files).
`Handoff before exit` ensures short-term operational state survives (→ handoff files).

They fire at the same trigger point (clear/exit/compact) but write to different places for different purposes.
```
