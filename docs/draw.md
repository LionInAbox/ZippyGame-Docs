ZippyGame can draw 2D shapes directly to the screen — circles, rectangles, lines, ellipses, and polygons.

Shape drawing is **immediate-mode**: you call draw functions every frame (typically inside `constantLoop` or `constantLoopAfterDraw`), and all shapes are batched internally and rendered in a single GPU pass. No shape objects to manage, no retained state. Call it, it draws, it's gone next frame.

Shapes render on top of sprites and particles.

## 1. Drawing a Point

```python
scene.drawPoint(x, y, radius, color)
```

```python
scene.drawPoint(400, 300, 5, (255, 255, 255))
```

Draws a filled circle. This is the simplest draw call — a convenience shortcut for `drawCircle` with no outline.

## 2. Circles

```python
scene.drawCircle(
    x, y,                          # center position (required)
    radius,                        # radius in pixels (required)
    color=(255, 255, 255, 255),    # RGBA 0-255 (default: white, fully opaque)
    thickness=0,                   # 0 = filled, 1-255 = outline width in pixels
)
```

```python
# Filled circle
scene.drawCircle(400, 300, 50, (255, 0, 0, 255))

# Outline circle
scene.drawCircle(400, 300, 50, (0, 255, 0), thickness=2)
```

## 3. Rectangles

```python
scene.drawRect(
    x, y,                          # center position (required)
    width, height,                 # dimensions in pixels (required)
    color=(255, 255, 255, 255),    # RGBA 0-255
    thickness=0,                   # 0 = filled, 1-255 = outline width
    corner_radius=0,               # rounded corners in pixels (0 = sharp)
    angle=0,                       # rotation in degrees
)
```

Position is the **center** of the rectangle, not the top-left corner.

```python
# Filled rectangle
scene.drawRect(400, 300, 200, 100, (0, 100, 255))

# Outlined rectangle
scene.drawRect(400, 300, 200, 100, (255, 255, 0), thickness=3)

# Rounded corners
scene.drawRect(400, 300, 200, 100, (255, 255, 255), corner_radius=15)

# Rotated 45 degrees
scene.drawRect(400, 300, 200, 100, (255, 128, 0), angle=45)

# All combined
scene.drawRect(400, 300, 200, 100, (0, 255, 200), thickness=2, corner_radius=10, angle=30)
```

## 4. Squares

```python
scene.drawSquare(
    x, y,                          # center position (required)
    size,                          # side length in pixels (required)
    color=(255, 255, 255, 255),    # RGBA 0-255
    thickness=0,                   # 0 = filled, 1-255 = outline width
    corner_radius=0,               # rounded corners in pixels
    angle=0,                       # rotation in degrees
)
```

Convenience wrapper for `drawRect` with equal width and height.

```python
scene.drawSquare(640, 360, 80, (255, 255, 255))
scene.drawSquare(640, 360, 80, (255, 0, 0), thickness=2, corner_radius=8)
```

## 5. Ellipses

```python
scene.drawEllipse(
    x, y,                          # center position (required)
    rx, ry,                        # horizontal and vertical radii (required)
    color=(255, 255, 255, 255),    # RGBA 0-255
    thickness=0,                   # 0 = filled, 1-255 = outline width
)
```

```python
# Filled ellipse
scene.drawEllipse(400, 300, 100, 50, (200, 100, 255))

# Outline ellipse
scene.drawEllipse(400, 300, 100, 50, (255, 255, 0), thickness=2)
```

## 6. Lines

```python
scene.drawLine(
    x1, y1,                        # start point (required)
    x2, y2,                        # end point (required)
    color=(255, 255, 255, 255),    # RGBA 0-255
    thickness=1,                   # line width in pixels (default: 1)
)
```

Lines have rounded endpoints.

```python
scene.drawLine(100, 100, 700, 500, (255, 255, 255))
scene.drawLine(100, 100, 700, 500, (255, 0, 0), thickness=4)
```

## 7. Polygons

```python
scene.drawPolygon(
    vertices,                      # list of (x, y) tuples (required, 3+ points)
    color=(255, 255, 255, 255),    # RGBA 0-255
    fill=True,                     # True = filled, False = outline only
    thickness=1,                   # outline width in pixels (only when fill=False)
)
```

Vertices can be in any winding order — clockwise or counter-clockwise. Both convex and concave polygons are supported.

```python
# Filled triangle
scene.drawPolygon(
    [(400, 500), (300, 300), (500, 300)],
    color=(0, 255, 100)
)

# Outlined hexagon
import math
cx, cy, r = 400, 300, 80
hexagon = [(cx + r * math.cos(a), cy + r * math.sin(a))
           for a in [math.radians(i * 60) for i in range(6)]]
scene.drawPolygon(hexagon, color=(255, 200, 0), fill=False, thickness=2)

# Concave polygon (star, arrow, L-shape — any shape works)
star = [(400, 450), (420, 370), (500, 350), (440, 310),
        (460, 230), (400, 280), (340, 230), (360, 310),
        (300, 350), (380, 370)]
scene.drawPolygon(star, color=(255, 50, 50))
```

When `fill=False`, the polygon outline is drawn as a series of connected line segments with rounded joins at each vertex.

## 8. Color

All draw functions accept color as an RGB or RGBA tuple with values 0–255:

```python
(255, 0, 0)        # red, fully opaque (alpha defaults to 255)
(255, 0, 0, 255)   # red, fully opaque (explicit)
(255, 0, 0, 128)   # red, half transparent
(255, 255, 255, 0) # fully transparent
```

The alpha channel is optional. If you pass 3 values, alpha defaults to 255 (fully opaque).

## 9. Where to Draw

Draw calls work in `constantLoop`, `constantLoopAfter`, and `constantLoopAfterDraw`. All shapes are collected during the frame and rendered together at the end:

```python
from zippyGame import *
from zippyGame.context import *

def constantLoop(dt):
    scene.drawCircle(400, 300, 50, (255, 0, 0))
    scene.drawLine(0, 0, 800, 600, (255, 255, 255), thickness=2)

def constantLoopAfterDraw(dt):
    scene.drawRect(640, 50, 200, 30, (0, 0, 0, 180))  # HUD background
```

Because shapes are batched and flushed at the end of the frame, they always render on top of sprites and particles regardless of which hook you call them from. Draw order between shapes themselves follows call order — shapes drawn first appear behind shapes drawn later.

## 10. Camera

Shapes follow the scene camera by default, just like sprites. If `scene.camera_x` or `scene.camera_y` are set, shapes move with the camera.

## 11. Performance

The shape renderer is built on the same instanced rendering architecture as the sprite system. Per-frame cost is 2 draw calls and 2 buffer uploads total, regardless of how many shapes you draw.

Per-shape cost is one numpy row write (~32 bytes). 10,000 shapes = 320 KB of upload per frame — negligible for any modern GPU.

Circles, rectangles, rounded rects, ellipses, and lines are all rendered via signed distance functions in a single instanced draw call. The fragment shader produces analytically anti-aliased edges at any scale with no MSAA cost.

Filled polygons are tessellated into triangles on the CPU and drawn in a second draw call. Convex polygons use a simple fan (fastest). Concave polygons use ear-clipping — O(n²) per polygon, which is fine for typical game polygons under ~100 vertices.

Polygon outlines are decomposed into line segments and rendered through the SDF path — no tessellation needed.

GPU resources for the shape renderer are allocated lazily on the first draw call. If you never draw shapes, no GPU memory is used.

## 12. Limits

The default capacity is 16,384 SDF shapes (circles, rects, lines, ellipses) and 65,536 triangle vertices (filled polygons) per frame. Shapes beyond the capacity are silently dropped.

Maximum outline thickness is 255 pixels. Maximum rounded corner radius is 255 pixels.

## 13. Example — HUD Elements

```python
from zippyGame import *
from zippyGame.context import *

player_health = 80   # out of 100

def constantLoopAfterDraw(dt):
    # Health bar background
    scene.drawRect(120, 680, 200, 20, (40, 40, 40))
    # Health bar fill
    bar_width = player_health * 2
    scene.drawRect(20 + bar_width / 2, 680, bar_width, 20, (0, 200, 50))
    # Health bar border
    scene.drawRect(120, 680, 200, 20, (255, 255, 255), thickness=1)
```

## 14. Example — Debug Visualization

```python
from zippyGame import *
from zippyGame.context import *
import math

player = scene.addImage("graphics/player.png", x=400, y=300)

def constantLoopAfterDraw(dt):
    # Show hitbox
    scene.drawCircle(player.x, player.y, 30, (0, 255, 0, 100), thickness=1)

    # Show velocity direction
    angle = math.radians(player.angle)
    dx = math.cos(angle) * 60
    dy = math.sin(angle) * 60
    scene.drawLine(player.x, player.y, player.x + dx, player.y + dy,
                   (255, 255, 0), thickness=1)
```

## 15. Example — Geometric Patterns

```python
from zippyGame import *
from zippyGame.context import *
import math

t = 0

def constantLoop(dt):
    global t
    t += dt

    cx, cy = game.width / 2, game.height / 2

    # Rotating ring of circles
    for i in range(12):
        a = math.radians(i * 30 + t * 60)
        x = cx + math.cos(a) * 150
        y = cy + math.sin(a) * 150
        scene.drawCircle(x, y, 15, (100, 200, 255, 200))

    # Pulsing center square
    size = 60 + math.sin(t * 3) * 20
    scene.drawSquare(cx, cy, size, (255, 100, 50), corner_radius=8, angle=t * 45)

    # Connecting lines
    for i in range(12):
        a1 = math.radians(i * 30 + t * 60)
        a2 = math.radians((i + 1) * 30 + t * 60)
        x1 = cx + math.cos(a1) * 150
        y1 = cy + math.sin(a1) * 150
        x2 = cx + math.cos(a2) * 150
        y2 = cy + math.sin(a2) * 150
        scene.drawLine(x1, y1, x2, y2, (100, 200, 255, 80), thickness=1)
```
