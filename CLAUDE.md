# Brownian Motion — TouchDesigner Script CHOP

Ornstein-Uhlenbeck brownian motion Script CHOP for TouchDesigner 2025+.
Pure Python OU engine with critically-damped spring filter, per-axis range, headless-tested.

## TouchDesigner Context

This is a TouchDesigner project. When working on this codebase:
- Always apply `/touchdesigner` skill knowledge (TD Python Bible) automatically
- For deep API work, also load `/touchdesigner-advanced`
- TD 2025+ is required — uses `CookLevel.ALWAYS`, 2025.x `absTime` behavior
- Script CHOP callbacks: `onSetupParameters`, `onCook`, `onPulse`, `onGetCookLevel`
- Custom param naming: Uppercase first letter, no underscores (e.g., `Peraxisrange`, `Rangeminx`)
- Never use `scriptOp.store()`/`fetch()` in `onCook` — causes cook dependency loops. Use module-level dict instead.
- Never set `par.enable` in `onCook` — also causes cook loops. Use `parGroup.enableExpr` in `onSetupParameters` instead (TD 2025+).
- GitHub repo: `REMvisual/touchdesigner-brownian-motion`
- UE5 reference implementation: `C:\Standalone\browniannoise_unreal\BrownianMotion\`

## Project Structure

- `src/brownian_motion.py` — Single-file: BrownianMotion class + TD Script CHOP callbacks
- `tests/test_brownian.py` — 20 headless pytest tests (extreme speed/range/edge cases)
- `Brownian Motion-V*.tox` — Prebuilt TD component (paste script into callbacks DAT)
- `README.md` — User-facing docs with parameter descriptions

## Dev Workflow

- Edit `src/brownian_motion.py`, test headlessly with `python -m pytest tests/ -v`
- Paste into TD Script CHOP callbacks DAT, hit Setup Parameters
- Export .tox from TD for releases
- `src/` and `tests/` are gitignored — only .tox, README, LICENSE go to GitHub

## Key Architecture Decisions

- **Single file**: class + callbacks in one file so TD only needs one paste
- **Module-level `_instances` dict**: avoids `scriptOp.store()`/`fetch()` cook loop
- **`CookLevel.ALWAYS`**: forces per-frame cooking (requires TD 2025+)
- **Decoupled spring**: OU runs at speed-scaled time, spring at real frame time
- **sqrt(speed) spring scaling**: keeps motion smooth at high speeds without lagging

## Beads Task Tracking Integration

Beads is our **persistent memory system**. Every piece of work gets a bead — no exceptions. Beads are how we know what happened in past sessions, what's in progress, and what's been completed. Without beads, context is lost between sessions.

### MANDATORY: Every Piece of Work Gets a Bead

**This is NOT optional. Before writing ANY code, a bead MUST exist.**

| Trigger | Action |
|---------|--------|
| User describes a plan or idea to implement | `bd create` immediately, set to `in_progress` |
| User says "do this", "fix this", "add this" | `bd create` immediately, set to `in_progress` |
| User says "do this later" / "add to backlog" | `bd create` with appropriate priority, leave as `open` |
| Starting a multi-step plan | `bd create` an epic + child tasks for each step |
| Bug report or issue discovered | `bd create --type=bug` |
| About to commit with no active bead | **STOP** — create a bead first, then close it with the commit |
| Session start | Run `bd ready` and `bd list --status=in_progress` to restore context |

### The Golden Rule

**No code without a bead. No commit without a bead ID.**

If you realize you've been working without a bead, create one immediately and backfill.

### Opening Beads

1. Check for existing related beads: `bd list --status=open` and `bd list --status=in_progress`
2. If a matching bead exists, use it: `bd update <id> --status in_progress`
3. If not, create one: `bd create --title="Description" --type=<task|bug|feature|epic> --priority=<0-4>`
4. Set to in-progress: `bd update <id> --status in_progress`

### Closing Beads

Close beads when:
- User confirms work is done ("done", "looks good", "approved", "ship it")
- User requests a commit (the commit itself means the work is complete)
- A bug fix is verified working

```bash
bd close <id> --reason "Completed — <brief summary>"
```

### Commits MUST Reference Beads

**Every git commit message MUST include the bead ID.** Format:

```
<type>: <description> (bd-<id>)
```

### Using Beads as Memory

**CRITICAL: Beads are your primary context source for TASK STATE.** When you need to understand:
- What happened in past sessions → `bd list` and `bd show <id>`
- Why something was built a certain way → check the bead description/notes
- What's currently in progress → `bd list --status=in_progress`
- What's blocked or waiting → `bd blocked` and `bd list --status=open`

At session start, ALWAYS check beads before asking the user to re-explain context.

### Integration with OpenViking

If this project has OpenViking configured (`ov.conf` exists):
- **Beads** tracks task state (what needs doing, what's done, dependencies)
- **OpenViking** tracks session memory (decisions, patterns, fixes, reasoning)
- When starting work on a bead, use `/memory-recall <bead title or topic>` to pull relevant past context

### Session Workflow

1. **Session Start**: `bd prime` runs automatically → then check `bd list --status=in_progress` for active work
2. **Before coding**: Ensure a bead exists for the work
3. **During work**: Update notes if design decisions are made: `bd update <id> --notes "..."`
4. **On commit**: Include `(bd-<id>)` in message, close bead if work is done
5. **Session End**: `bd sync`

### Beads Setup & Validation

If the user asks to **set up, update, validate, or check beads**, **invoke the `/beads` skill** — do NOT manually run `bd` commands for setup/validation.
