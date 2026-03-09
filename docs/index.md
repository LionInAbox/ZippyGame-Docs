# ZippyGame Documentation

A high-performance 2D game framework built on OpenGL instanced rendering.
Capable of rendering **200,000+ sprites at 80 FPS** with per-sprite rotation, tinting, parallax, and transparency — plus a GPU-driven particle system with 100,000+ particles at near-zero CPU cost.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Quick Start](#2-quick-start)
3. [Creating a Game](#3-creating-a-game)
4. [Scenes](#4-scenes)
5. [Sprites](#5-sprites)
6. [The Image Handle](#6-the-image-handle)
7. [Angles](#7-angles)
8. [Camera](#8-camera)
9. [Parallax](#9-parallax)
10. [Tinting](#10-tinting)
11. [Particle System](#11-particle-system)
12. [Bulk Numpy Access](#12-bulk-numpy-access)
13. [Configuration Defaults](#13-configuration-defaults)
14. [Utilities](#14-utilities)
15. [Using Pyglet Alongside ZippyGame](#15-using-pyglet-alongside-zippygame)
16. [Project Structure](#16-project-structure)
17. [Performance Notes](#17-performance-notes)

---

## 1. Requirements

```
pip install pyglet numpy Pillow
```

Minimum OpenGL 3.3. The framework automatically detects your GPU capabilities and selects the fastest available rendering backend.

---

## 2. Quick Start

```python
from zippyGame import *

game = newGame(title="My Game")
scene = game.addScene("level 1")

hero = scene.addImage("graphics/hero.png", x=400, y=300)
hero.alpha = 200

game.run()
```

That's it — a window opens, your sprite appears, and the game loop runs.

---

## 3. Creating a Game

```python
game = newGame(
    width=1280,              # window width in pixels (default: 1280)
    height=720,              # window height in pixels (default: 720)
    title="My Game",         # window title
    background_color="black" # see below for options
)
```

**Background color** accepts a preset string or a custom RGBA tuple:

```python
# Presets:
background_color="black"       # (0.08, 0.08, 0.12, 1.0)
background_color="white"       # (1.0, 1.0, 1.0, 1.0)
background_color="dark_gray"   # (0.15, 0.15, 0.15, 1.0)

# Custom:
background_color=(0.2, 0.0, 0.3, 1.0)   # dark purple
```
For a wider range of colors check out ZippyGame's 873  [colors](colors.md)

---

## 4. Scenes

A scene holds all sprites, particles, textures, and rendering state for one level or screen.

```python
scene = game.addScene("level 1", max_image_count=10000)
```

**`max_image_count`** is how many sprites this scene can hold. It determines how much memory is allocated up front. The default is 10,000. If you plan to use more sprites, set it higher:

```python
scene = game.addScene("battle", max_image_count=300000)
```

Memory cost is small — roughly 36 bytes per slot (CPU arrays + static data). 300,000 slots is about 10 MB. However, the number directly affects GPU buffer sizes and upload bandwidth, so don't set it to millions if you only need thousands.

**Important: all `addImage()` calls within a scene must currently use the same texture file.** This is the single-texture constraint for sprites. A texture atlas system to remove this limitation is planned. Particle emitters have their own textures and are not affected by this constraint.

---

## 5. Sprites

```python
img = scene.addImage(
    "graphics/bug.png",    # path to image file (required)
    x=400,                 # x position, default: window center
    y=300,                 # y position, default: window center
    width=64,              # sprite width in pixels, default: texture width
    height=64,             # sprite height in pixels, default: texture height
    alpha=255,             # opacity 0-255, default: 255 (fully opaque)
    angle=0,               # rotation in degrees (0-360), default: 0
    flip_x=False,          # horizontal flip
    flip_y=False,          # vertical flip
    tint_r=255,            # tint red 0-255, default: 255
    tint_g=255,            # tint green 0-255, default: 255
    tint_b=255,            # tint blue 0-255, default: 255
    tint_blend=0,          # tint mode 0-255, default: 0 (see Tinting)
    parallax_x=1.0,        # horizontal parallax, default: 1.0 (see Parallax)
    parallax_y=1.0,        # vertical parallax, default: 1.0
)
```

Every parameter except `image_path` is optional. `addImage()` returns an **Image handle** you can use to modify the sprite later.

You can call `addImage()` before or after `game.run()`. Sprites are usable immediately.

**Scaling sprites to a target size** while preserving aspect ratio:

```python
tex_w, tex_h = getImageSize("graphics/bug.png")
target_size = 70
if tex_w > tex_h:
    w = float(target_size)
    h = float(target_size) * tex_h / tex_w
else:
    h = float(target_size)
    w = float(target_size) * tex_w / tex_h

img = scene.addImage("graphics/bug.png", width=w, height=h)
```

**Maximum sprite dimensions:** width and height are stored as 12-bit integers internally, so the maximum is 4095 pixels per dimension.

---

## 6. The Image Handle

`addImage()` returns an Image object — a lightweight handle (~64 bytes) that reads and writes directly into the scene's internal arrays. No data is duplicated.

### Dynamic Properties

These update instantly with no GPU overhead (uploaded automatically each frame):

```python
img.x = 500.0          # float — horizontal position
img.y = 200.0          # float — vertical position
img.angle = 45         # float — rotation in degrees (0-360, wraps automatically)
img.alpha = 128        # int 0-255 — opacity
img.flip_x = True      # bool — horizontal flip
img.flip_y = False     # bool — vertical flip
img.visible = True     # bool — visibility (stub, for future use)
```

### Static Properties

These update the GPU static buffer when changed. Setting them is still fast, but each change triggers a small GPU upload for that sprite's slot:

```python
img.width = 100        # int 0-4095 — sprite width in pixels
img.height = 50        # int 0-4095 — sprite height in pixels
img.tint_r = 255       # int 0-255 — tint red
img.tint_g = 0         # int 0-255 — tint green
img.tint_b = 0         # int 0-255 — tint blue
img.tint_blend = 128   # int 0-255 — tint blend mode
img.parallax_x = 0.5   # float — horizontal parallax factor
img.parallax_y = 0.5   # float — vertical parallax factor
img.sprite_type = 0    # int 0-255 — reserved for texture atlas (future)
```

### Convenience Methods

These batch multiple changes into a single GPU update:

```python
img.setPosition(500, 200)          # set x and y together
img.setSize(100, 50)               # set width and height together
img.setTint(255, 0, 0, blend=128)  # set r, g, b and optionally blend
img.setParallax(0.5, 0.5)          # set parallax x and y together
```

Use convenience methods when changing multiple static properties at once — they issue one GPU update instead of several.

---

## 7. Angles

Sprite angles use **degrees** (0–360). Setting `img.angle = 45` rotates 45 degrees. Values wrap automatically — 370 degrees becomes 10 degrees.

```python
img.angle = 0          # no rotation
img.angle = 90         # quarter turn
img.angle = 180        # half turn
img.angle = 270        # three-quarter turn
```

Internally, angles are stored as uint16 (0–65535) for GPU efficiency. The `angle` property converts automatically. Precision is approximately 0.005 degrees.

**Conversion helpers** for working with radians or the internal uint16 format:

```python
from zippyGame import *

a = degrees_to_angle(90)       # → 16384 (uint16)
a = radians_to_angle(3.14159)  # → 32768 (uint16)
d = angle_to_degrees(16384)    # → 90.0
r = angle_to_radians(16384)    # → 1.5708...
```

For bulk numpy operations, use the constants directly:

```python
import numpy as np
from zippyGame.utilities import _RAD_TO_ANGLE

radians = np.array([0.0, 1.57, 3.14])
angles_u16 = (radians * _RAD_TO_ANGLE).astype(np.uint16)
```

---

## 8. Camera

Each scene has a camera that controls what part of the world is visible:

```python
scene.camera_x = 500.0
scene.camera_y = 200.0
```

Moving the camera shifts all sprites and particles on screen. The camera position is uploaded to the GPU once per frame as a uniform — zero per-sprite cost.

**Keyboard-controlled camera example:**

```python
from pyglet.window import key

keys = key.KeyStateHandler()
game.window.push_handlers(keys)

def update_camera(dt):
    speed = 300 * dt
    if keys[key.LEFT]:   scene.camera_x -= speed
    if keys[key.RIGHT]:  scene.camera_x += speed
    if keys[key.UP]:     scene.camera_y += speed
    if keys[key.DOWN]:   scene.camera_y -= speed

pyglet.clock.schedule(update_camera)
```

---

## 9. Parallax

Parallax controls how fast a sprite scrolls relative to the camera. This creates a depth illusion — distant backgrounds scroll slowly, nearby foregrounds scroll fast.

```python
img = scene.addImage("cloud.png", parallax_x=0.5, parallax_y=0.5)
```

**Parallax values:**

| Value | Effect |
|-------|--------|
| 0.0 | Fixed to screen (HUD, UI elements) |
| 0.5 | Half-speed scrolling (distant background) |
| 1.0 | Normal scrolling (default — moves with camera) |
| 1.5 | Faster than camera (close foreground) |
| 2.0+ | Extreme foreground |

The full range is **-6.35 to 6.4** in steps of 0.05. Negative values scroll in the opposite direction of the camera.

Parallax has zero per-frame cost — it's computed in the vertex shader from static data that's already on the GPU.

**Creating a parallax scene:**

```python
# Far background — barely moves
for i in range(20):
    scene.addImage("star.png", x=random.uniform(0, 1280),
                   y=random.uniform(0, 720), parallax_x=0.1, parallax_y=0.1)

# Mid background — moves at half speed
for i in range(50):
    scene.addImage("cloud.png", x=random.uniform(0, 1280),
                   y=random.uniform(0, 720), parallax_x=0.5, parallax_y=0.5)

# Normal world sprites
for i in range(200):
    scene.addImage("tree.png", x=random.uniform(0, 2000),
                   y=random.uniform(0, 720))  # default 1.0

# Close foreground — moves faster
for i in range(10):
    scene.addImage("leaf.png", x=random.uniform(0, 1280),
                   y=random.uniform(0, 720), parallax_x=1.5, parallax_y=1.5)
```

---

## 10. Tinting

Every sprite can be tinted with a color. The tint system has two modes controlled by `tint_blend`:

**Multiply mode** (`tint_blend=0`, default): Multiplies the sprite's color by the tint color. White (255, 255, 255) has no effect. Dark tints darken the sprite. Good for color grading and shadows.

**Replace mode** (`tint_blend=255`): Completely replaces the sprite's color with the tint, preserving only the alpha channel. Good for silhouettes and solid color fills.

**Blend mode** (any value 1–254): Mixes between multiply and replace. `tint_blend=128` gives a 50/50 mix.

```python
# Red tint in multiply mode — darkens non-red parts
enemy = scene.addImage("bug.png", tint_r=255, tint_g=80, tint_b=80, tint_blend=0)

# Blue solid overlay — sprite becomes a blue silhouette
frozen = scene.addImage("bug.png", tint_r=50, tint_g=50, tint_b=255, tint_blend=255)

# Gentle green tint — half blend
poisoned = scene.addImage("bug.png", tint_r=100, tint_g=255, tint_b=100, tint_blend=100)

# Change tint at runtime:
enemy.setTint(255, 255, 0, blend=128)   # flash yellow
enemy.tint_r = 255                       # or change individual channels
```

Tinting has zero per-frame cost — it's computed in the fragment shader from static data.

---

## 11. Bulk Numpy Access

For high-performance scenarios (100k+ sprites), you can bypass Image handles and write directly to the scene's numpy arrays. Both approaches write to the same underlying data.

```python
n = scene.count   # number of active sprites

# Move all sprites:
scene.cpu_positions[:n, 0] += velocities_x * dt    # x positions
scene.cpu_positions[:n, 1] += velocities_y * dt    # y positions

# Set all sprites to half opacity:
scene.cpu_alpha[:n] = 128

# Rotate all sprites:
scene.cpu_angles_u16[:n] += rotation_deltas

# Set flags directly:
scene.cpu_flags[:n] = 0   # clear all flags
```

**Available arrays:**

| Array | Shape | Dtype | Content |
|-------|-------|-------|---------|
| `scene.cpu_positions` | (capacity, 2) | float32 | x, y positions |
| `scene.cpu_angles_u16` | (capacity,) | uint16 | rotation angles |
| `scene.cpu_alpha` | (capacity,) | uint8 | opacity values |
| `scene.cpu_flags` | (capacity,) | uint8 | bit flags (flip_x, flip_y, visible) |
| `scene.static_data` | (capacity,) | structured | packed_size, tint, parallax |

Always slice to `[:n]` where `n = scene.count` — slots beyond `count` are unused.

---

## 12. Configuration Defaults

All defaults live in `defaults.py` and can be modified before creating scenes:

```python
from zippyGame import defaults

# Scene capacity:
defaults.max_image_count = 10000   # default max sprites per scene

# Sprite defaults (applied to every new addImage call):
defaults.alpha       = 255        # fully opaque
defaults.tint_r      = 255        # white = no tint
defaults.tint_g      = 255
defaults.tint_b      = 255
defaults.tint_blend  = 0          # multiply mode
defaults.parallax_x  = 1.0        # normal scrolling
defaults.parallax_y  = 1.0
defaults.flip_x      = False
defaults.flip_y      = False

# Rendering:
defaults.FORCE_TIER  = 0          # 0=auto, 1=persistent mapped, 2=orphaning, 3=subdata
defaults.NUM_BUFFER_SECTIONS = 3  # triple buffering (Tier 1 only)
defaults.MOVE_SPRITES = True      # benchmark mode: random sprite movement

# Particle defaults:
defaults.particle_pps         = 50     # default particles per second
defaults.particle_velocity    = 100.0  # default velocity in pixels/sec
defaults.particle_directionMin = 0     # default direction range (degrees)
defaults.particle_directionMax = 360
defaults.particle_lifetimeMin  = 1.0   # default lifetime range (seconds)
defaults.particle_lifetimeMax  = 1.0
```

**`MOVE_SPRITES`** is a benchmark flag. When `True`, all sprites get random velocities and bounce around the screen. Set it to `False` for actual game development — your sprites will stay where you put them.

**`FORCE_TIER`** overrides automatic GPU backend selection. Leave at 0 unless you're debugging rendering issues:
- **Tier 1** (GL 4.4+): Persistent mapped buffers with triple buffering. Fastest, zero CPU/GPU stalls.
- **Tier 2** (GL 3.3+): Buffer orphaning with mapped writes. Fast, driver handles synchronization.
- **Tier 3** (GL 3.3+): Classic `glBufferSubData`. Simplest, may cause GPU stalls.

---

## 13. Utilities

```python
from zippyGame import *

# Get image dimensions without loading into GPU:
width, height = getImageSize("graphics/hero.png")

# Pick a random item from a list:
texture = pick(["a.png", "b.png", "c.png"])

# Angle conversion (see Angles section):
a = degrees_to_angle(45)
a = radians_to_angle(1.57)
d = angle_to_degrees(a)
r = angle_to_radians(a)
```

---

## 14. Using Pyglet Alongside ZippyGame

ZippyGame is built on pyglet. You have full access to the pyglet window and its event system:

```python
game = newGame(title="My Game")
scene = game.addScene("main")

# Access the pyglet window directly:
game.window    # pyglet.window.Window instance
game.width     # window width
game.height    # window height

# Register keyboard handlers:
from pyglet.window import key
keys = key.KeyStateHandler()
game.window.push_handlers(keys)

# Schedule recurring updates:
def my_update(dt):
    if keys[key.SPACE]:
        print("Space pressed!")

pyglet.clock.schedule(my_update)

# Start the game loop:
game.run()
```

**Important:** `game.run()` calls `pyglet.app.run()` internally with `interval=0` (uncapped frame rate, vsync off). The game loop drives everything from there.

---

## 15. Project Structure

```
zippyGame/
    __init__.py
    _imports.py          — shared imports (numpy, pyglet, etc.)
    game.py              — newGame class, window, scene management
    renderer.py          — shaders, GL backends, InstancedRenderer
    defaults.py          — configuration defaults
    utilities.py         — angle helpers, getImageSize, pick
    context.py           — global game/scene references
    classes/
        objects.py       — Image class, parallax conversion
        scene.py         — scene class, addImage, update loop
        particles.py     — ParticleSystem, ParticleEmitter
    templates/
        sceneTemplate.py — scene lifecycle hooks (future)
```

---

## 16. Performance Notes

### Sprites

The renderer uses **instanced rendering** with a two-VBO architecture. Per sprite, only 12 bytes are uploaded to the GPU each frame (position, angle, alpha, flags). Static data like size, tint, and parallax is uploaded once and only updated when you change it.

At 200,000 sprites, the dynamic upload bandwidth is about 144 MB/s at 60 FPS (this may vary based on your hardware). The framework automatically selects the fastest upload path your GPU supports.

Things that are free (zero per-frame cost): tinting, parallax, sprite flipping, static size changes.

Things that cost per-frame: position changes, angle changes, alpha changes — but these are uploaded in bulk as a single contiguous buffer, not individually.

Sprites outside the visible window are automatically culled and not sent to the GPU.

### Particles

The particle system has near-zero CPU overhead — only spawning new particles costs CPU time (vectorized numpy batch generation + 1-2 GL upload calls per spawn batch). Once spawned, particles have zero per-frame CPU cost.

GPU vertex shader cost is ~15 trig calls per alive particle. Dead ring buffer slots early-exit with 2 comparisons, no trig. All direction transformers (spiral, curl, drift) and direction noise (wiggle, turbulence) are guarded by uniform-based checks — zero cost when disabled. Tint pulse features are similarly guarded.

**The main bottleneck is fillrate** (fragment shader). Cost scales with `particle_count × average_screen_coverage × overdraw_layers`. Large overlapping particles with `scaleIncrease` are the most expensive scenario. If frame rate drops during particle effects, reduce particle size or count before reducing features.

---

## Complete Example

```python
from zippyGame import *
from pyglet.window import key

# --- Setup ---
game = newGame(title="Parallax Demo", background_color="black")
defaults.MOVE_SPRITES = False
scene = game.addScene("demo", max_image_count=1000)

path = "graphics/player.png"
tex_w, tex_h = getImageSize(path)

# --- Background layer (slow parallax) ---
for i in range(50):
    scene.addImage(path, x=random.uniform(0, 1280), y=random.uniform(0, 720),
                   width=20, height=20, alpha=80, parallax_x=0.3, parallax_y=0.3)

# --- Main sprites ---
for i in range(200):
    scene.addImage(path, x=random.uniform(0, 1280), y=random.uniform(0, 720),
                   width=40, height=40)

# --- Foreground (fast parallax) ---
for i in range(20):
    scene.addImage(path, x=random.uniform(0, 1280), y=random.uniform(0, 720),
                   width=60, height=60, alpha=150, parallax_x=1.5, parallax_y=1.5)

# --- Tinted hero ---
hero = scene.addImage(path, x=640, y=360, width=80, height=80,
                      tint_r=255, tint_g=200, tint_b=50, tint_blend=100)

# --- Particle trail ---
trail = scene.addParticleFlow("graphics/spark.png", x=640, y=360, particles_per_second=100)
trail.scaleToWidth(10)
trail.particleLifetimeMax = 0.8
trail.velocityMin = 10
trail.velocityMax = 40
trail.color = ["white", "yellow", (255, 100, 0, 0)]
trail.blendMode("additive")

# --- Camera controls ---
keys = key.KeyStateHandler()
game.window.push_handlers(keys)

def update(dt):
    speed = 300 * dt
    if keys[key.LEFT]:   scene.camera_x -= speed
    if keys[key.RIGHT]:  scene.camera_x += speed
    if keys[key.UP]:     scene.camera_y += speed
    if keys[key.DOWN]:   scene.camera_y -= speed

pyglet.clock.schedule(update)
game.run()
```
