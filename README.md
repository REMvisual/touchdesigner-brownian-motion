<p align="center">
  <img src="https://github.com/REMvisual/td-BrownianMotion/releases/download/v2.0.0/banner-1280x640.png" alt="Brownian Motion" width="100%">
</p>

[![Download Latest](https://img.shields.io/github/v/release/REMvisual/td-BrownianMotion?style=for-the-badge&label=Download&color=blue)](https://github.com/REMvisual/td-BrownianMotion/releases/latest)
[![Downloads](https://img.shields.io/github/downloads/REMvisual/td-BrownianMotion/total?style=for-the-badge)](https://github.com/REMvisual/td-BrownianMotion/releases)
![Views](https://komarev.com/ghpvc/?username=REMvisual-td-BrownianMotion&label=Views&color=brightgreen&style=for-the-badge)

Ornstein-Uhlenbeck brownian motion for TouchDesigner. A Script CHOP that generates smooth, mean-reverting procedural noise with rotation support and fractal detail — perfect for organic camera drift, floating objects, generative motion, and anything that needs to feel alive.

## What It Does

Unlike random noise, Ornstein-Uhlenbeck motion always pulls back toward center — so you get natural, bounded wandering instead of unbounded drift.

- **Exact OU math** — Gillespie 1996 analytical transition, zero discretization error
- **Implicit spring** — Klak-style filter, unconditionally stable at any timestep
- **Rotation noise** — independent OU process with its own speed and per-axis amplitude
- **1/f pink noise detail** — Voss-McCartney algorithm adds organic micro-texture
- **Per-axis range** — different min/max bounds per axis
- **Speed-independent smoothing** — texture changes without affecting perceived speed
- **Deterministic** — set a seed for repeatable motion
- **26 automated tests** — validated at extreme parameters

## Install

**Requires TouchDesigner 2025+**

1. **[Download the .tox](https://github.com/REMvisual/td-BrownianMotion/releases/latest)** from Releases
2. Drag into your TouchDesigner project
3. Wire the CHOP output to whatever needs motion

Outputs six channels: `tx`, `ty`, `tz`, `rx`, `ry`, `rz`.

## Parameters

### Motion

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Speed** | `1.0` | Time-scale multiplier. 1 = real-time, 5 = 5x faster. 0 = frozen (instant stop). Transitions are smoothed. |
| **Amplitude** | `1.0` | Output multiplier. 0 = silence, 1 = full range. |
| **Center Pull** | `2.0` | How strongly motion pulls back toward anchor (theta). High = tight orbit. Low = lazy wander. 0 = pure Brownian, no pull. |
| **Smoothing** | `0.5` | Spring filter. 1 = very smooth (~2s settling). 0 = minimal filtering. Controls texture, not speed. |
| **Detail** | `0.0` | Voss-McCartney 1/f pink noise micro-texture. 0 = off. Respects Smoothing. |
| **Detail Layers** | `3` | Fractal octaves for Detail (1-5). More = richer frequency content. |

### Range

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Range Min / Max** | `[-1, 1]` | Output bounds. Normalized OU state maps to this range. Set to [0, 100] and channels wander between 0 and 100. |
| **Per-Axis Range** | `Off` | Toggle per-axis min/max. On = each axis gets its own bounds. |
| **Range Min/Max X, Y, Z** | `[-1, 1]` | Per-axis bounds (visible when Per-Axis Range on). |

### Rotation

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Enable Rotation** | `Off` | Master toggle. When off, rx/ry/rz output zero. |
| **Rotation Speed** | `1.0` | Independent speed for rotation. Separate from position Speed. |
| **Pitch / Yaw / Roll** | `5 / 5 / 0` | Amplitude in degrees per axis. 0 disables that axis. Shares Center Pull and Smoothing with position. |

### Axes

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Affect X / Y / Z** | `All on` | Per-axis enable toggles. Disabled axes output zero. |

### Advanced

| Parameter | Default | Description |
|-----------|---------|-------------|
| **Anchor X / Y / Z** | `0` | Drift target in normalized [-1, 1] space. 0 = center of range. 0.5 = hang out in upper half. |
| **Seed** | `0` | 0 = unseeded (different every run). Any other value = deterministic, same motion after Reset. |
| **Independent Axes** | `On` | ON = each axis gets its own noise (3D wandering). OFF = shared noise (lockstep diagonal). |
| **Reset** | `Pulse` | Snap to anchor positions and re-seed. Fresh start. |

## The Math

Three-stage pipeline, all running without substep loops:

**1. Exact OU Transition** (Gillespie 1996)

```
decay = exp(-theta * dt)
X = mu + (X - mu) * decay + sigma * sqrt((1 - exp(-2*theta*dt)) / (2*theta)) * Z
```

Where `theta` is Center Pull, `mu` is Anchor, and `sigma = 0.55 * sqrt(2*theta)` is tuned so the stationary distribution fills ~80% of the [-1,1] range regardless of theta. Zero discretization error at any timestep.

**2. Voss-McCartney 1/f Detail** (applied before spring)

Multiple independent OU processes updating at halved rates (layer 0 every frame, layer 1 every 2 frames, layer 2 every 4, etc.), summed with halving amplitude. Produces true 1/f pink noise — the spectral profile humans perceive as "natural." Added to the OU state before filtering so Smoothing controls detail too.

**3. Implicit Spring Filter** (Klak-style)

```
vel = (vel - (pos - target) * omega^2 * dt) / (1 + omega * dt)^2
pos += vel * dt
```

Unconditionally stable at any timestep. Omega scales with `sqrt(speed)` so smoothing character is preserved at all speeds. Runs on real frame time (decoupled from simulation speed).

## License

[MIT](LICENSE) -- use it however you want.
