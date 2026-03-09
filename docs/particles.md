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

**Important:** Set all emitter properties **before** the game loop starts. The system allocates GPU resources on the first frame based on your property values.

## 2. Position & Spawn Area

```python
emitter.x = 400                  # emitter position (can be changed at runtime)
emitter.y = 300
emitter.xMin = -50               # spawn offset: particles appear randomly within
emitter.xMax = 50                # the rectangle (x+xMin, y+yMin) to (x+xMax, y+yMax)
emitter.yMin = -20
emitter.yMax = 20
```

Default spawn area is 0/0/0/0 — all particles spawn at the emitter's exact position.

## 3. Particle Lifetime

```python
emitter.particleLifetimeMin = 1.0    # minimum lifetime in seconds
emitter.particleLifetimeMax = 3.0    # maximum lifetime in seconds
```

Each particle gets a random lifetime within this range. The GPU automatically kills particles when they expire. Default: both 1.0 second.

## 4. Velocity

```python
emitter.velocityMin = 50            # pixels/second (minimum)
emitter.velocityMax = 200           # pixels/second (maximum)
emitter.velocityIncrease = -30      # acceleration in pixels/second²
emitter.velocityClamp0 = True       # if True, velocity stops at 0 (won't reverse)
```

Each particle gets a random velocity between min and max. `velocityIncrease` applies uniformly over time.

**Wiggle** — particles rhythmically speed up and slow down:

```python
emitter.velocityWiggle = 20         # amplitude in pixels/second
emitter.velocityWiggleInterval = 0.5 # seconds per full oscillation
```

## 5. Direction (Movement Angle)

```python
emitter.directionMin = 0            # degrees (0=right, 90=up, 180=left, 270=down)
emitter.directionMax = 360          # full circle spread
emitter.directionIncrease = 45      # degrees/sec — creates curved paths
```

Each particle's initial direction is randomized between min and max. When `directionIncrease ≠ 0`, particles follow curved paths computed via closed-form integrals — mathematically exact, no approximation. This is a fundamental velocity parameter, like `velocityIncrease` for speed.

**Emitter Direction Modifiers** — affects where NEW particles are aimed, not how existing ones move:

```python
emitter.emitterDirectionIncrease = 30     # degrees/sec — rotates the emission cone over time
emitter.emitterDirectionWiggle = 15       # degrees amplitude — wobbles the cone
emitter.emitterDirectionWiggleInterval = 2.0
```

Think of it like a sprinkler slowly rotating and wobbling.

### Direction Transformation

Direction transformers actively reshape particle trajectories, e.g. into specific geometric patterns. They are applied to the base path before any noise effects, so noise layers on top of the transformed path cleanly.

**Spiral** — particles follow expanding spiral paths. Each particle gets a random angular acceleration within the min/max range, creating outward-looping trajectories that cross their own path:

```python
emitter.directionSpiralMin = -2.0      # degrees/sec²
emitter.directionSpiralMax = 2.0       # degrees/sec²
```

With opposite signs for min and max, some particles spiral clockwise and others counter-clockwise, creating a fan of diverging spiral paths. Higher values produce tighter, faster loops. The spiral always expands outward — particles never collapse back to the spawn point. Speed remains constant regardless of loop size.

To make the spiral intensify over the emitter's lifetime (new particles spiral more than earlier ones):

```python
emitter.directionSpiralIncrease = 0.5  # degrees/sec added to both min and max per second
```

**Curl** — particles trace cycloid loops (spring/coil shapes) along their travel path. The loops follow the particle's current heading, so they ride naturally on top of curved paths created by `directionIncrease`:

```python
emitter.directionCurl = 360            # degrees/sec — loop rotation speed
emitter.directionCurlRadius = 30       # pixels — loop size
```

Higher curl speed produces faster, tighter loops. A larger radius makes wider coils. Combined with `directionIncrease`, the coils follow a curved path — creating helical patterns along arcs. The curl updates the particle's heading, so noise effects applied after it stay perpendicular to the looping path.

**Drift** — Drift/ Wind/ Blow effect: a per-particle constant drift velocity in world X/Y coordinates, independent of the particle's travel direction:

```python
emitter.driftXMin = -10       # pixels/sec
emitter.driftXMax = 10        # pixels/sec
emitter.driftYMin = 20        # pixels/sec
emitter.driftYMax = 20        # pixels/sec
```

When min equals max, all particles drift identically — useful for wind or gravity-like effects. When they differ, particles spread out over time. Drift works in world space, so `driftYMin = -20, driftYMax = -20` always pulls downward regardless of the particle's travel direction.

**Drift Increase (per-particle)** — each particle accelerates its own drift over its lifetime, like gravity pulling harder the longer a particle has been alive:

```python
emitter.driftIncreaseX = 0       # pixels/sec² — per-particle X drift acceleration
emitter.driftIncreaseY = -50     # pixels/sec² — per-particle Y drift acceleration (e.g. gravity)
```

Every particle experiences the same acceleration curve from birth to death, regardless of when it was spawned.

**Drift Increase Global** — shifts the drift min/max range over the emitter's lifetime, so particles spawned later start with a stronger base drift:

```python
emitter.driftIncreaseGlobalX = 5    # pixels/sec added to driftXMin and driftXMax each second
emitter.driftIncreaseGlobalY = 0    # pixels/sec added to driftYMin and driftYMax each second
```

Useful for a wind that builds over time — early particles drift gently, later particles are born into a stronger wind.

Both can be combined: `driftIncreaseX/Y` makes each particle individually accelerate, while `driftIncreaseGlobalX/Y` ramps up the starting drift for newer particles.

### Direction Noise

Direction noise adds randomness and organic imperfection to particle paths.

**Wiggle** — smooth sine-wave oscillation perpendicular to the particle's current travel direction. Particles advance along their path while swaying side-to-side like a snake:

```python
emitter.directionWiggle = 20           # pixels — amplitude of lateral displacement
emitter.directionWiggleInterval = 1.0  # seconds per full oscillation
```

The wiggle follows the particle's *current* heading — if a particle is curving, spiraling, or curling, the wiggle stays perpendicular to the curve at every point. Each particle in a burst gets a unique phase offset so they don't oscillate in lockstep.

Additional wiggle options:

```python
emitter.directionWiggleZigzag = True   # sharp V-shaped bounces instead of smooth sine (default: False)
emitter.directionWiggleRandom = False  # all particles wiggle in sync (default: True = random phase)
```

Setting `directionWiggleRandom = False` makes all particles at the same spawn position follow the exact same sine/zigzag curve — useful for effects where synchronized movement is desired.

**Turbulence** — 'shakiness': displacement that adds organic randomness to particle paths:

```python
emitter.directionTurbulenceMin = 0.0   # pixels — per-particle minimum displacement
emitter.directionTurbulenceMax = 30.0  # pixels — per-particle maximum displacement
emitter.directionTurbulenceInterval = 1.0  # seconds per base oscillation
```

Each particle has a unique noise pattern. Good for adding organic imperfection and noise.

## 6. Scale

```python
emitter.scaleMin = 0.8              # overall scale multiplier (minimum)
emitter.scaleMax = 1.5              # overall scale multiplier (maximum)
emitter.scaleXMin = 1.0             # X scale / width multiplier (minimum)
emitter.scaleXMax = 1.0             # X scale / width multiplier (maximum)
emitter.scaleYMin = 0.5             # Y scale / height multiplier (minimum)
emitter.scaleYMax = 1.5             # Y scale / height multiplier (maximum)
```

Each particle gets random values for scale, scaleX, and scaleY independently. Final pixel size: `texture_width × scaleX × overall_scale` by `texture_height × scaleY × overall_scale`.

**Note:** Scale values are encoded as uint8 internally. Maximum representable value is 2.55.

**Relative Increase** — percentage of initial size per second:

```python
emitter.scaleIncrease = 50          # 50 = grow 50% per second, -50 = shrink 50% per second
emitter.scaleClamp0 = True          # default True — prevents negative scale
```

A value of 50 means: after 1 second, the particle is 50% larger than its starting size. Small and large particles grow proportionally.

**Wiggle:**

```python
emitter.scaleWiggle = 0.2           # scale units amplitude
emitter.scaleWiggleInterval = 0.8   # seconds per oscillation
```
Wiggles the scale back and forth. The particle grows and shrinks. If the scale is 0.5, a scaleWiggle of 0.5 will oscillate between 0.3 and 0.7.

### Convenient Sizing

Instead of calculating scale values to match a target pixel size you can resize the particles first while keeping the ratio:

```python
emitter.scaleToWidth(20)             # render at 20px wide, height preserves aspect ratio
emitter.scaleToHeight(30)            # render at 30px tall, width preserves aspect ratio
```

After calling one of these, setting the `scaleMin = 1.0` now means "20px wide" (or "30px tall").

## 7. Visual Rotation

```python
emitter.angleMin = 0                 # starting rotation in degrees (minimum)
emitter.angleMax = 360               # starting rotation in degrees (maximum)
emitter.rotationMin = -180           # rotates the angle in degrees/sec (minimum)
emitter.rotationMax = 180            # rotates the angle in degrees/sec (maximum)
emitter.rotationIncrease = 0         # rotation acceleration in degrees/sec²
emitter.rotationClamp0 = False       # if True, rotation speed can't go below 0
```

Each particle gets a random starting angle and random rotation speed. Visual rotation is independent from movement direction.

**Wiggle** — particles rotate back and forth:

```python
emitter.rotationWiggle = 30          # degrees amplitude
emitter.rotationWiggleInterval = 1.0
```

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
emitter.alphaWiggle = 0.2               # amplitude (0.0–1.0)
emitter.alphaWiggleInterval = 0.5       # seconds per oscillation
```

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
emitter.tintPulseColor = ["red", "yellow", "blue"]   # up to 8 color stops
emitter.tintPulseStrength = 0.8                        # 0.0=disabled, 1.0=full intensity
emitter.tintPulseInterval = 2.0                        # seconds per oscillation
```

The tint target progresses through the color array over the particle's lifetime, while each particle periodically pulses toward and away from the target.

**Wave Travel:**

```python
emitter.tintPulseWaveSpeed = 3.7      # positive = outward, negative = inward
```

Controls how fast the tint bands travel through the particle cloud.

**Color Cycling:**

```python
emitter.tintPulseColorSpeed = 10      # cycles the color mapping over time
```

The color mapping shifts continuously. A value of 10 means a full cycle every 10 seconds. Negative values cycle in reverse.

**Shape Deformation:**

```python
emitter.tintPulseJagged = 3.0         # high-frequency angular noise (sharp bumps)
emitter.tintPulseWarped = 5.0         # 2D spatial noise (organic blob shapes)
emitter.tintPulseAreaIncrease = 0.5   # pulse bands grow wider toward edges
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

When stopped, no new particles are created and global emitter updates (spiral increase, global drift increase, emitter direction) freeze. But existing particles finish their lifecycle, animate, and die normally.

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

## 15. Particle Example — Fire Effect

```python
fire = scene.addParticleFlow("graphics/soft_circle.png", x=640, y=200, particles_per_second=300)
fire.scaleToWidth(40)
fire.particleLifetimeMin = 0.5
fire.particleLifetimeMax = 1.5
fire.velocityMin = 30
fire.velocityMax = 80
fire.directionMin = 60
fire.directionMax = 120
fire.color = ["yellow", "orange", "red", (50, 0, 0)]
fire.alpha = [0.8, 0.6, 0.0]
fire.scaleIncrease = -30
fire.blendMode("additive")
```

## 16. Particle Example — Explosion Burst

```python
boom = scene.addParticleBurst("graphics/spark.png", x=400, y=400, count=200, repeat=1)
boom.scaleToWidth(8)
boom.particleLifetimeMin = 0.3
boom.particleLifetimeMax = 0.8
boom.velocityMin = 200
boom.velocityMax = 600
boom.rotationMin = -360
boom.rotationMax = 360
boom.alpha = [1.0, 0.0]
boom.blendMode("additive")
```

## 17. Particle Example — Spiraling Magic Effect

```python
magic = scene.addParticleFlow("graphics/soft_circle.png", x=640, y=360, particles_per_second=150)
magic.scaleToWidth(12)
magic.particleLifetimeMin = 1.5
magic.particleLifetimeMax = 3.0
magic.velocityMin = 40
magic.velocityMax = 100
magic.directionMin = 0
magic.directionMax = 360
magic.directionSpiralMin = -3.0
magic.directionSpiralMax = 3.0
magic.directionTurbulenceMin = 5.0
magic.directionTurbulenceMax = 15.0
magic.directionTurbulenceInterval = 0.8
magic.directionWiggle = 8
magic.directionWiggleInterval = 0.5
magic.color = ["cyan", "purple", "blue", (0, 0, 40)]
magic.alpha = [0.0, 0.8, 0.8, 0.0]
magic.scaleIncrease = -20
magic.blendMode("additive")
```

## 18. Particle Example — Spring Coils

```python
spring = scene.addParticleFlow("graphics/soft_circle.png", x=640, y=500, particles_per_second=100)
spring.scaleToWidth(6)
spring.particleLifetimeMin = 2.0
spring.particleLifetimeMax = 3.0
spring.velocityMin = 60
spring.velocityMax = 80
spring.directionMin = 80
spring.directionMax = 100
spring.directionCurl = 400
spring.directionCurlRadius = 25
spring.directionWiggle = 4
spring.directionWiggleInterval = 0.3
spring.directionWiggleZigzag = True
spring.color = ["white", "cyan", (0, 100, 255)]
spring.alpha = [1.0, 0.8, 0.0]
spring.blendMode("additive")
```

---