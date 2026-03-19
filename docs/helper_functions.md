# Helper Functions

A quick reference for the utility functions available in the framework.

---

## getImageSize(path)

Returns the **width** and **height** (in pixels) of an image file on disk — without loading it into the framework as a game asset. Useful when you need to know an image's resolution ahead of time, for example to calculate layout, scale sprites, or validate assets before use.

```python
w, h = getImageSize("assets/player.png")
```

**Parameters**

| Name | Type | Description |
|------|------|-------------|
| `path` | `str` | File path to the image |

**Returns:** A tuple `(width, height)` in pixels.

---

## mouseX()

Returns the current **x-coordinate** of the mouse cursor in screen space. Updated every frame automatically.

```python
x = mouseX()
```

**Returns:** `int` — the horizontal mouse position.

---

## mouseY()

Returns the current **y-coordinate** of the mouse cursor in screen space. Updated every frame automatically.

```python
y = mouseY()
```

**Returns:** `int` — the vertical mouse position.

---

## mousePos()

Returns full mouse coordinates.

```python
x, y = mousePos()
```

**Returns:** A tuple `(x, y)`.

---

## loadImage(path)

Loads an image from disk and returns a standard **pyglet image object**. This is not used inside the framework's own rendering pipeline — it exists so you can load an image for pixel-level manipulation with the functions below (e.g. reading and modifying individual pixels, then saving the result).

```python
img = loadImage("assets/texture.png")
```

**Parameters**

| Name | Type | Description |
|------|------|-------------|
| `path` | `str` | File path to the image |

**Returns:** A `pyglet.image.AbstractImage`.

---

## getPixels(pyglet_image)

Extracts all pixel data from a pyglet image and returns it as a **numpy array**, where each row is one pixel in **RGBA** format (values 0–255).

You can loop through individual pixels and set their values directly:

```python
pixels = getPixels(img)

for p in pixels:
    if p[3] < 100: # if alpha is < 100
        p[0], p[1], p[2], p[3] = 255, 0, 0, 255  # set every pixel to red
```

Or run fast bulk operations using numpy vectorization:

```python
pixels = getPixels(img)

# Create a boolean mask: True for every pixel where alpha < 100
mask = pixels[:, 3] < 100

# Apply new RGBA values only to the pixels where mask is True
pixels[mask] = [255, 0, 0, 255]
```

**Parameters**

| Name | Type | Description |
|------|------|-------------|
| `pyglet_image` | pyglet image | An image loaded via `loadImage()` |

**Returns:** A numpy `ndarray` of shape `(n_pixels, 4)` with dtype `uint8`.

---

## pixelsToImage(pixel_array, width, height)

Converts a numpy pixel array (as returned by `getPixels`) back into a **pyglet image object**. This is the reverse of `getPixels` — use it after you've finished modifying pixel data and want to turn it back into an image you can save or use.

```python
new_img = pixelsToImage(pixels, width=64, height=64)
```

**Parameters**

| Name | Type | Description |
|------|------|-------------|
| `pixel_array` | `ndarray` | A numpy array of shape `(n_pixels, 4)`, dtype `uint8` |
| `width` | `int` | Width of the image in pixels |
| `height` | `int` | Height of the image in pixels |

**Returns:** A `pyglet.image.ImageData`.

---

## saveImage(pyglet_image, file_name, path)

Saves a pyglet image to disk. You provide the file name and the folder path separately — the function joins them for you.

```python
saveImage(new_img, "output.png", "assets/generated")
# saves to: assets/generated/output.png
```

**Parameters**

| Name | Type | Description |
|------|------|-------------|
| `pyglet_image` | pyglet image | The image to save |
| `file_name` | `str` | Name of the output file (e.g. `"result.png"`) |
| `path` | `str` | Directory to save into |
