ZippyGame has a feature-rich, GPU-driven particle system.

The CPU sends a small spawn trigger for every particle, and the GPU handles the rest, with **zero per-frame CPU cost**.

Particles render on top of sprites.

## 1. Emitter Types

### Flow Emitter (Continuous)

```python
emitter = scene.addParticleFlow(
    "graphics/spark.png",       # particle texture (required)
    x=400, y=300,               # emitter position (default: window center)
    particles_per_second=200,   # emission rate (default: 50)
    life_time=-1,               # emitter duration in seconds, -1 = infinite (default: -1)
)
```

Particles spawn evenly distributed across frames — no frame-rate dependent bursting.

The emitter's lifetime can also be set after creation:

```python
emitter.lifeTime = 10              # emitter stops after 10 seconds (-1 = infinite)
```

### Burst Emitter

```python
burst = scene.addParticleBurst(
    "graphics/debris.png",      # particle texture (required)
    x=600, y=400,               # emitter position (default: window center)
    count=100,                  # particles per burst (default: 50)
    repeat=1,                   # 1 = once, N = N times, -1 = infinite, 0 = disabled
    interval=0.5,               # seconds between bursts (default: 1.0)
)
```

The first burst fires immediately. Subsequent bursts follow the interval.

**Time Offset** — shifts the burst schedule forward or backward in time:

```python
emitter.offset = 0.3              # delay: first burst fires at t=0.3 seconds
emitter.offset = -0.3             # advance: pretend 0.3 seconds already passed
```

With a positive offset, the emitter waits before firing its first burst. With a negative offset, the schedule is shifted as if time already elapsed — any bursts that would have fired in the virtual past are silently skipped, and the first actual burst is the next one on the shifted schedule. Skipped bursts do not count toward `repeat`.

Example with `offset = -0.3` and `interval = 0.5`: the virtual burst at t=0 is skipped. The next scheduled burst at t=0.5 fires at real time 0.2, then every 0.5 seconds after that.

**Important:** Set all emitter properties **before** the game loop starts. The system allocates GPU memory for particles on the first frame based on your settings.

## 2. Position & Spawn Area

```python
emitter.x = 400                  # emitter position (can be changed at runtime)
emitter.y = 300
```

Particles can spawn randomly within an offset area around the emitter position:

```python
emitter.xSpawn(min=-50, max=50)         # horizontal spawn offset range
emitter.ySpawn(min=-20, max=20)         # vertical spawn offset range
emitter.posSpawn(min_x=-50, max_x=50, min_y=-20, max_y=20)   # set both at once
```

Default spawn area is 0 — all particles spawn at the emitter's exact position.

## 3. Particle Lifetime

```python
emitter.lifetimeParticle(min=1.0, max=3.0)    # lifetime range in seconds
emitter.lifetimeParticle.min = 1.0             # set individually
emitter.lifetimeParticle.max = 3.0
```

Each particle gets a random lifetime within this range. The GPU automatically kills particles when they expire. Default: both 1.0 second.

## 4. Velocity

```python
emitter.velocity(min=50, max=200, increase=-30)   # set all at once
emitter.velocity.min = 50              # pixels/second (minimum)
emitter.velocity.max = 200             # pixels/second (maximum)
emitter.velocity.increase = -30        # acceleration in pixels/second² (default: 0)
emitter.velocity.clamp0 = True         # if True, velocity stops at 0 (won't reverse)
```

Each particle gets a random velocity between min and max. `increase` applies uniformly over time.

**Wiggle** — particles rhythmically speed up and slow down:

```python
emitter.velocity.wiggle = 20           # amplitude in pixels/second
emitter.velocity.wiggle.interval = 0.5 # seconds per full oscillation
```

Or set both at once:

```python
emitter.velocity.wiggle(amount=20, interval=0.5)
```

## 5. Direction (Movement Angle)

```python
emitter.direction(min=0, max=360, increase=45)   # set all at once
emitter.direction.min = 0              # degrees (0=right, 90=up, 180=left, 270=down)
emitter.direction.max = 360            # full circle spread
emitter.direction.increase = 45        # degrees/sec — creates curved paths (default: 0)
```

Each particle's initial direction is randomized between min and max. When `direction.increase ≠ 0`, particles follow curved paths computed via closed-form integrals — mathematically exact, no approximation. This is a fundamental velocity parameter, like `velocity.increase` for speed.

**Emission Cone Modifiers** — affects where NEW particles are aimed, not how existing ones move:

```python
emitter.cone.direction = 40                  # constant offset in degrees (rotates the whole cone)
emitter.cone.direction.increase = 30         # degrees/sec — rotates the emission cone over time
emitter.cone.direction.wiggle = 15           # degrees amplitude — wobbles the cone
emitter.cone.direction.wiggle.interval = 2.0
```

Think of it like a sprinkler slowly rotating and wobbling. `cone.direction` applies a fixed rotation to the emission angle, while `cone.direction.increase` rotates it continuously over time.

### Direction Transformation

Direction transformers actively reshape particle trajectories, e.g. into specific geometric patterns. They are applied to the base path before any noise effects, so noise layers on top of the transformed path cleanly.

**Spiral** — particles follow expanding spiral paths. Each particle gets a random angular acceleration within the min/max range, creating outward-looping trajectories that cross their own path:

```python
emitter.direction.spiral(min=-2.0, max=2.0)       # degrees/sec²
emitter.direction.spiral.min = -2.0
emitter.direction.spiral.max = 2.0
```

With opposite signs for min and max, some particles spiral clockwise and others counter-clockwise, creating a fan of diverging spiral paths. Higher values produce tighter, faster loops. The spiral always expands outward — particles never collapse back to the spawn point. Speed remains constant regardless of loop size.

To make the spiral intensify over the emitter's lifetime (new particles spiral more than earlier ones):

```python
emitter.direction.spiral.increase = 0.5   # degrees/sec added to both min and max per second
```

**Curl** — particles trace cycloid loops (spring/coil shapes) along their travel path. The loops follow the particle's current heading, so they ride naturally on top of curved paths created by `direction.increase`:

```python
emitter.direction.curl = 360              # degrees/sec — loop rotation speed
emitter.direction.curl.radius = 30        # pixels — loop size
```

Or set both at once:

```python
emitter.direction.curl(speed=360, radius=30)
```

Higher curl speed produces faster, tighter loops. A larger radius makes wider coils. Combined with `direction.increase`, the coils follow a curved path — creating helical patterns along arcs. The curl updates the particle's heading, so noise effects applied after it stay perpendicular to the looping path.

**Drift** — Drift/ Wind/ Blow effect: a per-particle constant drift velocity in world X/Y coordinates, independent of the particle's travel direction:

```python
emitter.drift.x(min=-10, max=10)          # pixels/sec
emitter.drift.y(min=20, max=20)           # pixels/sec
emitter.drift.x.min = -10
emitter.drift.x.max = 10
emitter.drift.y.min = 20
emitter.drift.y.max = 20
```

When min equals max, all particles drift identically — useful for wind or gravity-like effects. When they differ, particles spread out over time. Drift works in world space, so `drift.y(min=-20, max=-20)` always pulls downward regardless of the particle's travel direction.

**Drift Increase (per-particle)** — each particle accelerates its own drift over its lifetime, like gravity pulling harder the longer a particle has been alive:

```python
emitter.drift.x.increase = 0          # pixels/sec² — per-particle X drift acceleration
emitter.drift.y.increase = -50        # pixels/sec² — per-particle Y drift acceleration (e.g. gravity)
```

Every particle experiences the same acceleration curve from birth to death, regardless of when it was spawned.

**Drift Increase Global** — shifts the drift min/max range over the emitter's lifetime, so particles spawned later start with a stronger base drift:

```python
emitter.drift.world.x.increase = 5    # pixels/sec added to drift X min and max each second
emitter.drift.world.y.increase = 0    # pixels/sec added to drift Y min and max each second
```

Useful for a wind that builds over time — early particles drift gently, later particles are born into a stronger wind.

Both can be combined: `drift.x.increase` / `drift.y.increase` makes each particle individually accelerate, while `drift.world.x.increase` / `drift.world.y.increase` ramps up the starting drift for newer particles.

### Direction Noise

Direction noise adds randomness and organic imperfection to particle paths.

**Wiggle** — smooth sine-wave oscillation perpendicular to the particle's current travel direction. Particles advance along their path while swaying side-to-side like a snake:

```python
emitter.direction.wiggle = 20              # pixels — amplitude of lateral displacement
emitter.direction.wiggle.interval = 1.0    # seconds per full oscillation
```

The wiggle follows the particle's *current* heading — if a particle is curving, spiraling, or curling, the wiggle stays perpendicular to the curve at every point. Each particle in a burst gets a unique phase offset so they don't oscillate in lockstep.

Additional wiggle options:

```python
emitter.direction.wiggle.zigzag = True     # sharp V-shaped bounces instead of smooth sine (default: False)
emitter.direction.wiggle.random = False    # all particles wiggle in sync (default: True = random phase)
```

Setting `direction.wiggle.random = False` makes all particles at the same spawn position follow the exact same sine/zigzag curve — useful for effects where synchronized movement is desired.

**Turbulence** — 'shakiness': displacement that adds organic randomness to particle paths:

```python
emitter.direction.turbulence(min=0, max=30)          # pixels — per-particle displacement range
emitter.direction.turbulence.interval = 1.0           # seconds per base oscillation
```

Each particle has a unique noise pattern. Good for adding organic imperfection and noise.

## 6. Scale

```python
emitter.scale(min=0.8, max=1.5)            # overall scale multiplier range
emitter.scale.width(min=1.0, max=1.0)      # X scale / width multiplier range
emitter.scale.height(min=0.5, max=1.5)     # Y scale / height multiplier range
```

Each particle gets random values for scale, width, and height independently. Final pixel size: `texture_width × width × overall_scale` by `texture_height × height × overall_scale`.

**Note:** Scale values are encoded as uint8 internally. Maximum representable value is 2.55.

**Scale Increase** — adds to the scale every second, regardless of initial size:

```python
emitter.scale.increase = 50            # 50 = add 0.5 to scale per second, -50 = subtract 0.5 per second
emitter.scale.clamp0 = True            # default True — prevents negative scale
```

A value of 50 adds 0.5 to the scale every second. A particle starting at scale 0.0 with `scale.increase = 50` grows to 0.5 after one second, 1.0 after two seconds. This is absolute, not proportional — all particles grow at the same rate regardless of their starting size.

When `scale.clamp0 = False`, scale is allowed to go negative, which flips/mirrors the particle texture. This creates a smooth inversion as the particle passes through zero size.

**Wiggle:**

```python
emitter.scale.wiggle = 50              # percent amplitude — 50 = ±50% of current scale
emitter.scale.wiggle.interval = 0.8    # seconds per oscillation
```

Wiggles the scale back and forth as a percentage of the particle's current size (after `scale.increase` is applied). A particle at current scale 1.0 with `scale.wiggle = 50` oscillates between 0.5 and 1.5. A particle at current scale 0.4 with the same wiggle oscillates between 0.2 and 0.6 — the wiggle is always proportional.

### Convenient Sizing

Instead of calculating scale values to match a target pixel size you can resize the particles first while keeping the ratio:

```python
emitter.scaleToWidth(20)             # render at 20px wide, height preserves aspect ratio
emitter.scaleToHeight(30)            # render at 30px tall, width preserves aspect ratio
```

After calling one of these, setting `scale(min=1.0, max=1.0)` now means "20px wide" (or "30px tall").

## 7. Visual Rotation

```python
emitter.angle(min=0, max=360)              # starting rotation in degrees
emitter.rotation(min=-180, max=180)        # spin speed in degrees/sec
emitter.rotation.increase = 0             # rotation acceleration in degrees/sec² (default: 0)
emitter.rotation.clamp0 = False           # if True, rotation speed can't go below 0
```

Each particle gets a random starting angle and random rotation speed. Visual rotation is independent from movement direction.

**Angle Wiggle** — particles rock back and forth:

```python
emitter.angle.wiggle = 30              # degrees amplitude
emitter.angle.wiggle.interval = 1.0
```

**Auto Angle** — automatically orient each particle to face its movement direction:

```python
emitter.angle.auto = True              # orient to movement direction (default: False)
```

When enabled, the particle's visual rotation follows its instantaneous heading instead of using the static per-particle starting angle from `angle(min=, max=)`. This means particles always "point" the way they're moving — through curves, spirals, curls, and any other path shape.

By default, the auto angle includes the influence of noise effects (direction wiggle, turbulence, drift). To follow only the major path (direction, direction.increase, spiral, curl) and ignore noise:

```python
emitter.angle.auto.followNoise = False   # ignore wiggle/turbulence/drift for heading (default: True)
```

With `followNoise = True` (default), a wiggling particle visually rocks as it sways. With `followNoise = False`, the particle points along the smooth underlying path and the wiggle only affects position.

Rotation, rotation clamp, and angle wiggle all still compose on top of the auto heading — so you can add deliberate spin or wobble even with auto angle enabled.

**Angle Offset** — apply a constant rotation offset in degrees:

```python
emitter.angle.offset = -90             # degrees (default: 0)
```

Useful when a particle texture isn't oriented the way you need at angle 0. For example, if your arrow texture points upward but your particles travel rightward, `angle.offset = -90` corrects it. Works in both auto and non-auto modes.

## 8. Color

```python
emitter.color = ["yellow", "red", "black"]     # up to 8 color stops
```

Particles transition through the color array over their lifetime. Accepts named colors or RGB/RGBA tuples (0–255):

```python
emitter.color = ["white"]                                    # constant white
emitter.color = ["yellow", "red", "black"]                   # fire fade
emitter.color = [(255, 200, 0), (255, 0, 0), (0, 0, 0)]    # same, with tuples
emitter.color = ["white", (34, 230, 30, 130), (0, 0, 0, 0)] # green fade to transparent
```

**Named colors:** white, black, red, green, blue, yellow, orange, purple, cyan, magenta, gray/grey.
Additionally, ZippyGame offers 873 names you can call by name. Check out [Colors](colors.md)

## 9. Alpha

```python
emitter.alpha = [1.0, 0.0]          # up to 8 stops (0.0=transparent, 1.0=opaque)
```

Transitions over the particle's lifetime. Multiplied with color alpha.

```python
emitter.alpha = [1.0, 1.0]              # constant opacity (default)
emitter.alpha = [1.0, 0.0]              # fade out
emitter.alpha = [0.0, 1.0, 1.0, 0.0]   # fade in, hold, fade out
```

**Wiggle** — flickering/blinking with per-particle random phase:

```python
emitter.alpha.wiggle = 20              # percent amplitude — 20 = ±0.2 alpha
emitter.alpha.wiggle.interval = 0.5    # seconds per oscillation
```

A particle at alpha 0.8 with `alpha.wiggle = 20` oscillates between 0.6 and 1.0. The result is always clamped to the 0.0–1.0 range.

## 10. Blend Mode

```python
emitter.blendMode("normal")      # standard alpha blending (smoke, dust, debris)
emitter.blendMode("additive")    # bright glow (fire, sparks, magic, lasers)
mode = emitter.blendMode()       # returns current mode (no argument = getter)
```

## 11. Greyscale Base

```python
emitter.greyscaleBase = True     # converts texture to greyscale before applying colors
```

Useful when your particle texture has its own colors that conflict with the color array. A red texture tinted blue looks dark/black. With `greyscaleBase = True`, the texture is neutralized first, so any tint applies cleanly.

## 12. Tint Pulse

A secondary color effect that creates traveling waves of color through the particle cloud.

**Basic setup:**

```python
emitter.tintPulse.color = ["red", "yellow", "blue"]   # up to 8 color stops
emitter.tintPulse = 0.8                                 # strength: 0.0=disabled, 1.0=full intensity
emitter.tintPulse.interval = 2.0                        # seconds per oscillation
```

The tint target progresses through the color array over the particle's lifetime, while each particle periodically pulses toward and away from the target.

**Wave Travel:**

```python
emitter.tintPulse.waveSpeed = 3.7      # positive = outward, negative = inward
```

Controls how fast the tint bands travel through the particle cloud.

**Color Cycling:**

```python
emitter.tintPulse.color.speed = 10     # cycles the color mapping over time
```

The color mapping shifts continuously. A value of 10 means a full cycle every 10 seconds. Negative values cycle in reverse.

**Shape Deformation:**

```python
emitter.tintPulse.jagged = 3.0         # high-frequency angular noise (sharp bumps)
emitter.tintPulse.warped = 5.0         # 2D spatial noise (organic blob shapes)
emitter.tintPulse.areaIncrease = 0.5   # pulse bands grow wider toward edges
```

- **Jagged** creates small sharp bumps around the pulse rings, giving it a heptagonic star-like shape (using multi-octave sin waves). Creates noise at high values.
- **Warped** creates smooth, amoeba-like deformations using 2D spatial interference patterns based on particle world positions. This produces the most organic-looking effects.
- **Area Increase** makes the tint effect stronger on older particles further from the emitter.

All three stack independently and can be combined.

## 13. Debug Tools

```python
count = emitter.aliveCount()    # returns number of currently alive particles
```

Call manually when needed — zero cost when not called. The number represents alive particles, not total ring buffer capacity.

## 14. Emitter Playback Controls — Pause, Stop, Play

Emitters can be paused and stopped independently.

**Pause / Unpause** — freezes/resumes all particle animation:

```python
emitter.pause()          # freeze all particles in place (position, scale, rotation, alpha — everything)
emitter.unpause()        # resume animation from exactly where it left off
```

When paused, the emitter's internal clock stops. No particles move, no particles die, no new particles spawn. Unpausing seamlessly resumes — particles pick up mid-animation with no visible jump.

**Stop / Play** — controls whether new particles are spawned:

```python
emitter.stop()           # stop spawning new particles; existing particles continue to death
emitter.play()           # resume spawning
```

When stopped, no new particles are created and global emitter updates (spiral increase, global drift increase, cone direction) freeze. But existing particles finish their lifecycle, animate, and die normally.

**Combining both** — the two flags are independent, creating four possible states:

| paused | stopped | Particles | Spawning | Global Updates |
|--------|---------|-----------|----------|----------------|
| no     | no      | animate   | yes      | advance        |
| no     | yes     | animate   | no       | frozen         |
| yes    | no      | frozen    | no       | frozen         |
| yes    | yes     | frozen    | no       | frozen         |

This lets you do things like:

```python
# Freeze the scene, and control spawning:
emitter.stop()           # stop spawning
emitter.pause()          # freeze everything
emitter.unpause()        # existing particles animate to death, but no new ones spawn
emitter.play()           # resume spawning

# Or: stop, freeze, activate spawn, then unfreeze:
emitter.stop()
emitter.pause()
emitter.play()           # cleared the stop flag, but still paused — nothing visible happens
emitter.unpause()        # now everything resumes, including spawning
```

## 15. Motion Blur

Motion blur creates smooth, path-following trails behind particles using a persistent FBO that fades over time. Particles leave behind an afterimage that naturally follows any curved, spiraling, or wiggling path.

```python
emitter.motionBlur = 5              # trail half-life (0=off, 1=short, 10=1sec, 30=long)
```

The value controls how long trails linger. It represents a half-life: `motionBlur = 10` means trails fade to half brightness in 1 second. Lower values produce short, snappy trails; higher values produce long, lingering ones.

**Performance:** Motion blur adds 2–3 fullscreen quad passes per emitter per frame (fade, composite, optional clamp). This cost is constant regardless of particle count — 500k particles cost the same as 100. It does not increase particle overdraw.

## 16. Motion Stretch

Motion stretch elongates each particle's sprite dimensions along its movement direction, creating a speed-streak effect.

```python
emitter.motionStretch = 10          # stretch factor (0=off, 10=moderate, 50=extreme)
```

The stretch amount is proportional to the particle's current speed with a factor 10. A particle moving at 200 px/s with `motionStretch = 10` gets stretched by 200 pixels total. Static particles are unaffected.

**Performance note:** Motion stretch increases the pixel area of every particle, which can significantly reduce framerate. For path-following blur effects, `motionBlur` is much more efficient.

## 17. Sub-Emitters

Sub-emitters turn every particle into its own emitter — particles that spawn particles. The entire system runs on the GPU via compute shaders with zero per-frame CPU cost.

### Setup

Create two emitters: the **main emitter** whose particles become spawn points, and the **template emitter** whose settings define what the sub-particles look like and how they behave.

```python
# Template: defines sub-particle appearance and behavior
sparks = scene.addParticleFlow("graphics/spark.png", particles_per_second=30)
sparks.scaleToWidth(4)
sparks.velocity(min=20, max=60)
sparks.direction(min=0, max=360)
sparks.lifetimeParticle(min=0.2, max=0.5)
sparks.alpha = [1.0, 0.0]
sparks.color = ["yellow", "orange", "red"]

# Main emitter: each particle spawns sub-particles using the template's settings
trail = scene.addParticleFlow("graphics/soft_circle.png", particles_per_second=50)
trail.velocity(min=100, max=200)
trail.direction(min=60, max=120)
trail.lifetimeParticle(min=2.0, max=3.0)
trail.subEmitter(sparks)
```

Calling `subEmitter()` does two things: it connects the template's settings to the main emitter's compute shader, and it automatically makes the template inert — no GPU memory allocated, no spawning, no drawing. The template exists only as a configuration source.

### Options

```python
emitter.subEmitter(template, keepEmitter=False, hideMainParticles=False)
```

**`keepEmitter`** — by default, the template emitter becomes inactive. Set `keepEmitter=True` to let it continue to exist and emit particles independently:

```python
trail.subEmitter(sparks, keepEmitter=True)    # sparks also emits on its own at sparks.x, sparks.y
```

**`hideMainParticles`** — hides the main emitter's own particles. Only the sub-particles are drawn. The main particles still exist internally (the compute shader needs their positions), they just aren't rendered:

```python
trail.subEmitter(sparks, hideMainParticles=True)    # trail's particles are invisible, only sparks are visible
```

This is useful when the main emitter serves purely as a path generator — you want particles following curved or spiraling paths, but only the sub-particles spawned along those paths should be visible.

### What the Template Controls

The template emitter's settings split into two roles:

**Spawn parameters** — determine how sub-particles are born. These are read by the compute shader when creating each sub-particle:

- `velocity(min=, max=)` — initial speed range
- `direction(min=, max=)` — initial direction range
- `cone.direction` — direction offset (including increase and wiggle)
- `lifetimeParticle(min=, max=)` — lifetime range
- `scale(min=, max=)`, `scale.width(min=, max=)`, `scale.height(min=, max=)` — size ranges
- `angle(min=, max=)` — starting visual rotation range
- `rotation(min=, max=)` — spin speed range
- `xSpawn`, `ySpawn`, `posSpawn` — spawn offset area
- `particles_per_second` (flow mode) or `count`, `interval`, `repeat` (burst mode)

**Animation parameters** — determine how sub-particles behave after spawning. These are used by the vertex shader when drawing:

- All velocity, direction, scale, rotation increase/wiggle/clamp settings
- Direction spiral, curl, turbulence, drift
- Color array, alpha array, tint pulse
- Blend mode, greyscale base, motion stretch
- Angle auto, angle offset, angle wiggle

### Flow vs Burst Sub-Emitters

The template's emitter type determines how sub-particles spawn from each parent particle:

**Flow template** — each alive parent particle continuously spawns sub-particles at the template's `particles_per_second` rate:

```python
sparks = scene.addParticleFlow("graphics/spark.png", particles_per_second=20)
# Each parent spawns ~20 sub-particles per second for as long as it lives
```

**Burst template** — each parent particle fires bursts of sub-particles at regular intervals based on its own age:

```python
sparks = scene.addParticleBurst("graphics/spark.png", count=10, interval=0.5, repeat=-1)
# Each parent fires 10 sub-particles every 0.5 seconds
```

### Pause Behavior

When the main emitter is paused, sub-particle spawning stops. Existing sub-particles also freeze (they share the main emitter's time clock). Unpausing resumes both seamlessly.

### Performance

Sub-emitters use a GPU compute shader to read parent particle positions and write sub-particle data into a GPU buffer. No data transfers between CPU and GPU. The compute dispatch and memory barrier add a small constant cost per frame per sub-emitter, independent of particle count. Drawing sub-particles costs the same as drawing the same number of regular particles.

The sub-particle buffer is pre-allocated based on the maximum expected count (parent capacity × sub-particles per parent × lifetime), capped at ~3.7 million particles. When no sub-emitters are configured, the compute shader is never compiled and zero overhead is added.

**Requires OpenGL 4.3+** (compute shader support). The system checks the GL version and raises a clear error if the hardware doesn't support it.

## 18. Particle Example — Fire Effect

```python
# Fire Emitter:
# Setup:
fire = scene.addParticleFlow("graphics/fire_particle.png", particles_per_second=230) # Load Particle Graphic and basic setup
fire.greyscaleBase = True # Make base graphic grey for perfect coloring
fire.y = 0 # Start at the bottom of the screen
fire.scaleToWidth(35) # Scale down to 35 pixel width and corresponding height

# Speed:
fire.velocity(519, 634, -170)
fire.velocity.clamp0 = True # particles can't start moving backwards

# Direction:
dirBuffer = 15
fire.direction(90 + dirBuffer, 90 - dirBuffer) # 90 +- 15 degrees
fire.direction.wiggle = 40 # flicker effect
fire.direction.wiggle.interval = 0.7

# Rotation:
fire.angle.wiggle = 90 # flicker effect
fire.angle.wiggle.interval = 0.2
fire.angle.auto = True # auto rotate particles towards movement direction

# Scale:
fire.scale(1.2, 1.7, -20)
fire.scale.wiggle = 30 # flicker effect
fire.scale.wiggle.interval = 0.3

# Particles:
fire.lifetimeParticle(0.8, 1.5)
fire.posSpawn(-20, 20, -10,10) # Spawn area

# Colors:
fire.color = ["white", color.orangePeel, color.redSalsa, color.redPurple]
fire.alpha = [0, 1, 1, 1, 1, 0.2, 0.2, 0]
fire.blendMode("additive")

# Emitter cone:
fire.cone.direction.wiggle = 10 # flame movement effect
fire.cone.direction.wiggle.interval = 0.4

# Smoothing with blur:
fire.blur = 0.8
fire.motionStretch = 0.4
```

## 19. Particle Example — Explosion Burst

```python
boom = scene.addParticleBurst("graphics/spark.png", x=400, y=400, count=200, repeat=1)
boom.scaleToWidth(8)
boom.lifetimeParticle(min=0.3, max=0.8)
boom.velocity(min=200, max=600)
boom.rotation(min=0, max=360)
boom.alpha = [1.0, 0.0]
boom.blendMode("additive")
```

## 20. Particle Example — Spiraling Magic Effect

```python
magic = scene.addParticleFlow("graphics/bug.png", x=640, y=360, particles_per_second=150)
magic.scaleToWidth(42)
magic.lifetimeParticle(min=6.5, max=13.0)
magic.velocity(min= 250, max=470)
magic.direction(min=0, max=0)
magic.direction.spiral(min=560.0, max=560.0)
magic.direction.turbulence(min=5.0, max=15.0)
magic.direction.turbulence.interval = 0.8
magic.direction.wiggle = 8
magic.direction.wiggle.interval = 0.5
magic.color = ["cyan", "purple", "blue", (0, 0, 40)]
magic.alpha = [0.0, 0.8, 0.8, 0.0]
magic.scale.increase = -20
magic.blendMode("additive")
```

## 21. Particle Example — Spring Coils

```python
spring = scene.addParticleFlow("graphics/soft_circle.png", x=640, y=300, particles_per_second=100)
spring.scaleToWidth(6)
spring.lifetimeParticle(min=4.0, max=6.0)
spring.velocity(min=60, max=80)
spring.direction(min=80, max=100)
spring.direction.curl = 400
spring.direction.curl.radius = 25
spring.direction.wiggle = 4
spring.direction.wiggle.interval = 0.3
spring.direction.wiggle.zigzag = True
spring.color = ["white", "cyan", (0, 100, 255)]
spring.alpha = [1.0, 0.8, 0.0]
spring.blendMode("additive")
```

---
