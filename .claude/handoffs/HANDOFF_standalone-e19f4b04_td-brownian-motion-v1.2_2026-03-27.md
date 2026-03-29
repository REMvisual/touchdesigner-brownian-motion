# TouchDesigner Brownian Motion Script CHOP — v1.2.1 Released

**Date:** 2026-03-27
**Status:** COMPLETED
**Bead(s):** none (beads initialized at end of session, no issues created yet)
**Epic:** none
**Chain:** `standalone-e19f4b04` seq `1`
**Parent:** `none — first in chain`
**Prior chain:** none — first in chain

---

## The Goal

Port the UE5 Ornstein-Uhlenbeck Brownian Motion plugin math to a TouchDesigner Script CHOP, add per-axis range support, validate the math with headless tests at extreme parameters, and release publicly on GitHub. The UE5 plugin at `C:\Standalone\browniannoise_unreal\BrownianMotion\` is the reference implementation. The TD version is a single-file Script CHOP for the `REMvisual/touchdesigner-brownian-motion` GitHub repo.

## Where We Are

- **v1.2.1 released** on GitHub: https://github.com/REMvisual/touchdesigner-brownian-motion/releases/tag/v1.2.1
- **Single-file architecture**: `src/brownian_motion.py` contains BrownianMotion class + TD Script CHOP callbacks (395 lines)
- **20 headless pytest tests** all passing (3.2-6.4s runtime): `tests/test_brownian.py` (418 lines)
- **Per-axis range** feature added: `Peraxisrange` toggle with per-axis min/max params
- **Decoupled spring filter**: OU runs at speed-scaled time, spring runs at real frame time
- **sqrt(speed) spring scaling**: smooth motion at high speeds without lagging behind
- **Module-level `_instances` dict**: avoids `scriptOp.store()`/`fetch()` cook dependency loop
- **`CookLevel.ALWAYS`**: forces per-frame cooking (TD 2025+ required)
- **`absTime.frame` read in onCook**: creates per-frame cook dependency
- **.tox exported**: `Brownian Motion-V1.2.0.tox` attached to GitHub release
- **README rewritten**: block-style parameter docs, math explanation, architecture section
- **Project initialized**: beads + OpenViking + CLAUDE.md with TD context
- **gitignored**: `src/`, `tests/`, `__pycache__/`, `.pytest_cache/` — only .tox, README, LICENSE in git
- **Confirmed working in TD 2025** (Build 2025.x). Does NOT work in TD 2023 (Build 2023.12230) — `CookLevel` not supported.

## What We Tried (Chronological)

1. **Initial architecture: separate core + callbacks files** — Created `brownian_core.py` and `script_chop_callbacks.py`. User said "we only want one right?" → merged into single `brownian_motion.py`. Cleaner for TD (one paste into callbacks DAT).

2. **`scriptOp.store()`/`fetch()` for state** — Used TD storage to persist BrownianMotion instance between cooks. Caused cook dependency loop (writing to storage during cook triggers recook). Fix: module-level `_instances = {}` dict keyed by `scriptOp.id`.

3. **`absTime.stepSeconds` for dt without time dependency** — Removed `absTime` ref to fix cook loop but then nothing triggered per-frame recooking. Fix: kept `absTime.frame` read (creates dependency) + `CookLevel.ALWAYS`.

4. **No `CookLevel.ALWAYS`** — CHOP only cooked when params changed, not every frame. Fix: added `onGetCookLevel` returning `CookLevel.ALWAYS`.

5. **Spring inside OU substep loop** — At high speeds, spring was chasing 20+ random jumps per frame. Result: jittery output at speed>5. Fix: decoupled spring from OU loop — OU runs all substeps first, then spring runs separately at real frame time.

6. **No speed scaling on spring** — After decoupling, spring couldn't keep up with fast OU evolution. Fix: `sqrt(speed)` scaling on spring omega. At speed=10, spring is 3.2x stiffer (not 10x linear or 1x none).

7. **Linear speed scaling on spring (UE5 approach)** — `omega * speed` made spring transparent at high speeds. Tried `1 + ln(speed)` but still had spring inside loop. Final answer: decoupled spring + sqrt(speed).

8. **`par.enable` in `onCook`** — Setting parameter enable state during cook caused cook dependency loop (modifying param properties triggers recook). Fix: removed from onCook entirely. Tried `par.enableExpr` in `onSetupParameters` but `enableExpr` is a `ParGroup` property, not `Par` — failed silently and broke setup. Final: removed enable toggling entirely for now.

9. **TD 2023 compatibility** — Script CHOP wouldn't animate in TD 2023 (Build 2023.12230). `CookLevel.ALWAYS` not supported in pre-2025 builds. Added "Requires TouchDesigner 2025+" to README.

## Key Decisions

- **Single file, not module + wrapper**: User preference. One paste into TD, tests import the class directly. `src/` and `tests/` are gitignored — GitHub only has .tox + README.
- **Decoupled spring from OU**: The spring is a display-side low-pass filter, not physics. Running it at real time (not speed-scaled) keeps output smooth regardless of OU speed. This DIVERGES from the UE5 implementation which runs spring inside the substep loop.
- **sqrt(speed) over linear or log**: Linear makes spring transparent at high speeds. Log was tried but insufficient. Sqrt is the sweet spot — tested against all 20 tests.
- **Module-level dict over TD storage**: `_instances[scriptOp.id]` avoids cook loop. Survives across cooks. Only risk: if Script CHOP is deleted, stale entry remains (harmless).
- **No per-axis enable toggling (deferred)**: `par.enable` in onCook and `par.enableExpr` in setup both caused issues. Cosmetic feature — deferred to future version.
- **Default range [-1, 1] not [-100, 100]**: TD users typically want normalized output to map downstream. Changed from UE5's [-100, 100] cm default.
- **TD 2025+ requirement**: `CookLevel.ALWAYS` and `absTime.stepSeconds` behavior require 2025. Not worth backward compat hacks.

## Evidence & Data

### Git Release History

| Tag | Date | Description |
|-----|------|-------------|
| v1.0.0 | pre-session | Initial .tox + README |
| v1.1.0 | 2026-03-27 | Per-axis range, reworked OU engine, detailed param docs |
| v1.2.0 | 2026-03-27 | Decoupled spring, cook loop fix, per-axis enable |
| v1.2.1 | 2026-03-27 | TD 2025+ requirement, clean release |

### Test Suite (20 tests, all passing)

| # | Test | Validates |
|---|------|-----------|
| 1 | range_coverage | OU explores >40% of [-1,1] per axis (5000 steps) |
| 2 | speed_scaling | speed=5 covers 1.5x+ distance vs speed=1 |
| 3 | axis_isolation | disabled axes = zero output |
| 4 | per_axis_range | X[-50,50] Y[-200,200] Z[-10,10] respected, Y spread > X |
| 5 | uniform_range | per_axis=False gives same bounds to all |
| 6 | deterministic_seed | same seed = identical 1000-step playback |
| 7 | extreme_speed_high | speed=100, 1000 steps, all finite, clamped [-1,1] |
| 8 | extreme_speed_zero | frozen after speed smoother settles |
| 9 | extreme_range_large | [-10000, 10000] no overflow |
| 10 | extreme_range_tiny | [-0.001, 0.001] stays in bounds |
| 11 | extreme_range_asymmetric | [-10, 500] mean near midpoint 245 |
| 12 | zero_reversion | pure random walk, still clamped |
| 13 | high_reversion | mean abs < 0.7, not stuck at boundary |
| 14 | roughness_smooth | roughness=0 delta < roughness=1 delta |
| 15 | roughness_raw | roughness=1: smoothed == OU state |
| 16 | center_bias | center=0.8: mean > 0.3 |
| 17 | amplitude_zero | output = 0 |
| 18 | amplitude_scaling | 2x amplitude = 2x output |
| 19 | no_nan_inf | 10K steps, random extreme params, all finite |
| 20 | reset | state zeroed |

### Spring Scaling Comparison

| Speed | Linear (UE5) | sqrt (final) | Effect |
|-------|-------------|-------------|--------|
| 1 | 1x | 1x | Same |
| 5 | 5x | 2.2x | Noticeably smoother |
| 10 | 10x | 3.2x | Smooth, tracks well |
| 100 | 100x | 10x | Still smooth, not transparent |

### TD Version Compatibility

| TD Version | Build | Works? | Issue |
|------------|-------|--------|-------|
| 2025 | 2025.x | Yes | Full functionality |
| 2023 | 2023.12230 | No | `CookLevel.ALWAYS` not supported |

### OU Math (ported from UE5)

```
dx = theta * (mu - x) * dt + sigma * sqrt(dt) * N(0,1)
sigma = 0.55 * sqrt(2 * theta)  [if theta > 0, else 0.55]
spring: omega = 2 * exp(roughness * 3.2) * sqrt(speed)
damping = 2 * omega  [critical damping]
substep: max 1/120s per step
```

### Cook Loop Causes Found

| Cause | Mechanism | Fix |
|-------|-----------|-----|
| `scriptOp.store()` in onCook | Writing storage triggers recook | Module-level `_instances` dict |
| `par.enable = ...` in onCook | Modifying param properties triggers recook | Removed (deferred) |
| `par.enableExpr` in onSetupParameters | `enableExpr` is on ParGroup not Par, failed silently | Removed |

## Code Analysis

### BrownianMotion class (lines 14-190)
- `__init__(seed)`: Random seed, 3-element state lists, Box-Muller spare
- `step(dt, speed, reversion, roughness, center, independent_axes)`: OU substeps then spring filter
- `map_offset(amplitude, range_min, range_max, affect, per_axis_range, axis_ranges)`: [-1,1] → output
- `_box_muller()`: Gaussian noise via Box-Muller transform with spare caching
- `reset()`: Zero all state

### TD Callbacks (lines 200-395)
- `_instances = {}`: Module-level state dict, keyed by `scriptOp.id`
- `onSetupParameters`: 4 pages (Motion, Range, Axes, Advanced), 20+ custom params
- `onGetCookLevel`: Returns `CookLevel.ALWAYS`
- `onCook`: Read params, step simulation, map output, write 3 channels (tx/ty/tz)
- `onPulse`: Handle Reset pulse via `_instances[key].reset()`

### UE5 Reference (C:\Standalone\browniannoise_unreal\BrownianMotion\)
- `BrownianMotionComponent.cpp`: 935 lines, StepOU at line 627, MapOffset at line 400
- Has collision avoidance (ProbeRepulsion, SweepClamp) — NOT ported to TD
- Has Sequencer integration (Perlin-based, not OU) — NOT ported to TD
- Has blend in/out system — NOT ported to TD

## Files Changed

### Source code
- `src/brownian_motion.py` — Full OU engine + TD Script CHOP callbacks (395 lines)
- `src/__init__.py` — Empty package init

### Tests
- `tests/test_brownian.py` — 20 headless pytest tests (418 lines)
- `tests/__init__.py` — Empty package init

### Documentation
- `README.md` — Block-style parameter docs, math section, architecture, TD 2025+ requirement
- `CLAUDE.md` — Project context with TD skill triggers, architecture decisions, beads section

### Config
- `.gitignore` — src/, tests/, __pycache__/, .pytest_cache/, ov.conf, .openviking/, .dolt/, *.db, .beads credential key
- `.claude/settings.local.json` — bd + python permissions

### Releases
- `Brownian Motion-V1.2.0.tox` — Prebuilt TD component (attached to GitHub release)

### Project infra
- `.beads/` — Beads task tracking (Dolt backend)
- `ov.conf` — OpenViking config (gitignored)
- `.openviking/memory/` — OV state directory (gitignored)
- `AGENTS.md` — Auto-generated by beads

## User Feedback & Preferences

- **"we only want one right?"** — User wants single-file architecture. No separate core + wrapper.
- **High speed jitter complaint** — User noticed spring was transparent at high speeds. Led to decoupled spring + sqrt scaling.
- **"varying aspects" at high speed with variable ranges** — Expected behavior (different ranges = different output magnitudes). User accepted after explanation.
- **"it only moves when I move parameters"** — Led to `CookLevel.ALWAYS` discovery.
- **Wanted comment block for nodegraph** — User moved the docstring out of the script into a TD Annotate/comment. Wanted compact but detailed parameter descriptions.
- **"the git should only have the tox and the readme"** — Clean public repo. Dev files gitignored.
- **Direct style** — User doesn't need hand-holding. "YES" means do it.
- **Version naming** — User exports .tox with version suffix: `Brownian Motion-V1.2.0.tox`
- **TD 2023 compatibility** — User tested in 2023, found it broken, accepted 2025+ requirement
- **"make an init for this project"** — Wants full project setup (beads + OV) for released projects
- **"make sure our init knows we're in TD"** — CLAUDE.md should reference TD skills and conventions

## Where We're Going

1. **Per-axis enable toggling** — `parGroup.enableExpr` (not `par.enableExpr`) in `onSetupParameters` should work in TD 2025+. Needs testing.
2. **TD 2023 fallback** — Could add a `try: CookLevel.ALWAYS` / `except: absTime.frame` pattern for backward compat. Low priority.
3. **Additional features from UE5** — Blend in/out, collision avoidance, Sequencer integration. All optional.
4. **Re-export .tox for v1.2.1** — Current GitHub release has v1.2.0 .tox but v1.2.1 tag. User should re-export and upload.
5. **Backport spring changes to UE5 plugin** — The decoupled spring + sqrt scaling might benefit the UE5 version too.

## Risks & Blockers

- **`parGroup.enableExpr` untested** — Might not work on Script CHOP custom params. Need to verify in TD 2025.
- **UE5 spring divergence** — TD version now has different spring behavior than UE5. Could cause confusion if someone uses both.

## Open Questions

- Should the TD version support TD 2023 with a fallback path?
- Should `enableExpr` use `parGroup` access or is there a cleaner TD 2025+ API for conditional param visibility?

## Quick Start for Next Session

```bash
# Restore context
cd C:\Standalone\touchdesigner-brownian-motion

# Key files to read first
cat src/brownian_motion.py    # Full implementation (395 lines)
cat tests/test_brownian.py    # Test suite (418 lines)
cat CLAUDE.md                 # Project context + TD conventions

# UE5 reference
cat C:\Standalone\browniannoise_unreal\BrownianMotion\Source\BrownianMotion\Private\BrownianMotionComponent.cpp

# Run tests
python -m pytest tests/test_brownian.py -v

# GitHub repo
# https://github.com/REMvisual/touchdesigner-brownian-motion

# Next action: Test parGroup.enableExpr for per-axis range toggle in TD 2025
```
