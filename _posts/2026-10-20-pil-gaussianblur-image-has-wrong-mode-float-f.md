---
title: "PIL GaussianBlur \"ValueError: image has wrong mode\" on a float ('F') image from numpy — and why converting to 'L' can zero your data"
description: >-
  Image.fromarray() on a float32 or float64 numpy array gives a mode 'F'
  image, and Pillow's GaussianBlur, BoxBlur, UnsharpMask and BLUR all raise
  "ValueError: image has wrong mode" on it. The usual workaround (scale to
  8-bit 'L') turned a map of small values into all zeros and wrapped -0.1 to
  231. Reproduced with Pillow 12.3.0, with a numpy-only Gaussian blur that
  keeps the floats.
tags: [python, pillow, numpy, images]
---

You build a mask or a heat map in numpy as floats, turn it into a Pillow image
with `Image.fromarray()`, and ask for a blur. Pillow raises
**`ValueError: image has wrong mode`**. The image is mode `'F'` (32-bit float),
and Pillow's Gaussian blur doesn't support it. The common fix, scaling to an
8-bit `'L'` image, makes the error go away, but in my tests it also turned
small values into zeros and a negative value into 231, with no warning.

Below: the error reproduced, which modes and filters fail, what the 8-bit
workaround does to float data, and a short numpy Gaussian blur that works on
the floats directly.

The versions I ran everything with:

```console
$ python3 --version
Python 3.12.13
$ python3 -c 'import PIL, numpy; print(PIL.__version__, numpy.__version__)'
12.3.0 2.5.3
```

Every script below sits in one folder and runs in the order shown.

## Reproduce

```python
# repro.py — a float mask from numpy becomes an 'F' image
import numpy as np
from PIL import Image, ImageFilter

mask = np.zeros((100, 100), dtype=np.float32)
mask[30:70, 30:70] = 1.0
img = Image.fromarray(mask)
print("mode:", img.mode)
img.filter(ImageFilter.GaussianBlur(3))
print("blurred")
```

```console
$ python3 repro.py
mode: F
Traceback (most recent call last):
  File ".../repro.py", line 9, in <module>
    img.filter(ImageFilter.GaussianBlur(3))
  File ".../site-packages/PIL/Image.py", line 1462, in filter
    return self._new(filter.filter(self.im))
  File ".../site-packages/PIL/ImageFilter.py", line 206, in filter
    return image.gaussian_blur(xy)
ValueError: image has wrong mode
```

(I shortened the paths to `...` and removed the `^^^^` marker lines.)

`float32` gave mode `F`, and `GaussianBlur` failed inside
`image.gaussian_blur`. The message doesn't name the mode or the filter.

## Which dtypes and filters

`Image.fromarray` picks the mode from the array's dtype. Then I tried five
filters on an empty image of each mode:

```python
# modes.py — which numpy dtypes give which mode, and which filters accept it
import numpy as np
from PIL import Image, ImageFilter

for dt in ["uint8", "uint16", "int32", "float32", "float64"]:
    print(f"{dt:8s} -> {Image.fromarray(np.zeros((4, 4), dtype=dt)).mode}")

filters = {
    "GaussianBlur": ImageFilter.GaussianBlur(2),
    "BoxBlur": ImageFilter.BoxBlur(2),
    "UnsharpMask": ImageFilter.UnsharpMask(2),
    "BLUR": ImageFilter.BLUR,
    "MedianFilter": ImageFilter.MedianFilter(3),
}
for mode in ["L", "RGB", "I;16", "I", "F"]:
    im = Image.new(mode, (20, 20))
    row = []
    for name, f in filters.items():
        try:
            im.filter(f)
            row.append(f"{name} ok")
        except ValueError as e:
            row.append(f"{name} FAIL ({e})")
    print(f"{mode:5s}", "; ".join(row))
```

```console
$ python3 modes.py
uint8    -> L
uint16   -> I;16
int32    -> I
float32  -> F
float64  -> F
L     GaussianBlur ok; BoxBlur ok; UnsharpMask ok; BLUR ok; MedianFilter ok
RGB   GaussianBlur ok; BoxBlur ok; UnsharpMask ok; BLUR ok; MedianFilter ok
I;16  GaussianBlur FAIL (image has wrong mode); BoxBlur FAIL (image has wrong mode); UnsharpMask FAIL (image has wrong mode); BLUR ok; MedianFilter FAIL (image has wrong mode)
I     GaussianBlur FAIL (image has wrong mode); BoxBlur FAIL (image has wrong mode); UnsharpMask FAIL (image has wrong mode); BLUR ok; MedianFilter ok
F     GaussianBlur FAIL (image has wrong mode); BoxBlur FAIL (image has wrong mode); UnsharpMask FAIL (image has wrong mode); BLUR FAIL (image has wrong mode); MedianFilter ok
```

So in this Pillow version, `GaussianBlur`, `BoxBlur` and `UnsharpMask` fail
on `F`, `I` and `I;16`, not only on floats: an image made from a `uint16` or
`int32` array hits the same error. `BLUR` works on `I` and `I;16` but not on
`F`. `MedianFilter` works on `F`.

## The usual workaround, and what it did to my data

Scale to 0–255, make an `'L'` image, blur, scale back:

```python
# via_l.py — the usual workaround: scale to 8-bit 'L', blur, scale back
import numpy as np
from PIL import Image, ImageFilter

def blur_via_l(a, radius):
    l = Image.fromarray((a * 255).astype(np.uint8))
    return np.asarray(l.filter(ImageFilter.GaussianBlur(radius)), dtype=np.float32) / 255

mask = np.zeros((100, 100), dtype=np.float32)
mask[30:70, 30:70] = 1.0
print("0..1 mask, max after blur:", blur_via_l(mask, 3).max())

heat = np.zeros((100, 100), dtype=np.float32)
heat[30:70, 30:70] = 0.002              # small values, e.g. a probability map
print("0.002 map, max after blur:", blur_via_l(heat, 3).max())

values = np.array([0.5, 1.0, 1.2, -0.1], dtype=np.float32)
print("1.2 and -0.1 as uint8:", (values * 255).astype(np.uint8))
```

```console
$ python3 via_l.py
0..1 mask, max after blur: 1.0
0.002 map, max after blur: 0.0
1.2 and -0.1 as uint8: [127 255  50 231]
```

- For a mask that really is 0..1 and where 256 levels are enough, it works.
- **A map whose values are all below 1/255 came back as all zeros.**
  `0.002 * 255` is 0.51, and `astype(np.uint8)` truncates it to 0 before the
  blur runs.
- **Out-of-range values don't clip; they wrap.** On this Mac, 1.2 became 50
  (306 − 256) and -0.1 became 231. NumPy printed no warning. Use
  `np.clip(a, 0, 1)` first if you go this way, and accept that anything above
  1 is lost.

## The fix: blur the floats directly

A Gaussian blur is two 1-D convolutions, one along the rows and one along the
columns, so numpy alone is enough:

```python
# gauss.py — Gaussian blur for float images ('F' mode or float arrays), numpy only
import numpy as np
from PIL import Image

def gaussian_blur_float(img, sigma):
    a = np.asarray(img, dtype=np.float32)
    r = max(1, int(3 * sigma + 0.5))           # cover +-3 sigma
    x = np.arange(-r, r + 1, dtype=np.float32)
    k = np.exp(-(x * x) / (2 * sigma * sigma))
    k /= k.sum()                               # weights sum to 1
    p = np.pad(a, r, mode="edge")              # repeat edge pixels
    # separable: blur rows, then columns
    p = np.apply_along_axis(lambda v: np.convolve(v, k, mode="valid"), 1, p)
    p = np.apply_along_axis(lambda v: np.convolve(v, k, mode="valid"), 0, p)
    out = p.astype(np.float32)
    return Image.fromarray(out) if isinstance(img, Image.Image) else out
```

```python
# fixed.py — blur the float data directly
import numpy as np
from PIL import Image
from gauss import gaussian_blur_float

heat = np.zeros((100, 100), dtype=np.float32)
heat[30:70, 30:70] = 0.002
out = gaussian_blur_float(Image.fromarray(heat), 3)
a = np.asarray(out)
print("mode:", out.mode, "max:", a.max(), "min:", a.min())
print("sum before:", heat.sum(), "after:", a.sum())

values = np.zeros((20, 20), dtype=np.float32)
values[5:15, 5:15] = [[1.2]] * 10
values[0, 0] = -0.1
v = gaussian_blur_float(values, 2)
print("out-of-range input, min/max after blur:", v.min(), v.max())
```

```console
$ python3 fixed.py
mode: F max: 0.002 min: 0.0
sum before: 3.2000003 after: 3.1999998
out-of-range input, min/max after blur: -0.035835568 1.1685065
```

The small values survived, the total stayed the same here (the values sit far from the border; near the edges the repeated edge pixels can change the sum),
the result is still a mode `F` image, and values outside 0..1 stayed outside
it instead of wrapping.

Is it the same blur as Pillow's? I compared the two on an 8-bit image, where
Pillow's `GaussianBlur(3)` works:

```python
# compare.py — gaussian_blur_float vs Pillow's GaussianBlur on an 8-bit image
import numpy as np
from PIL import Image, ImageFilter
from gauss import gaussian_blur_float

rng = np.random.default_rng(0)
a = (rng.random((120, 160)) * 255).astype(np.uint8)
a[40:80, 60:100] = 255
pil = np.asarray(Image.fromarray(a).filter(ImageFilter.GaussianBlur(3)), dtype=np.float32)
mine = gaussian_blur_float(a.astype(np.float32), 3)
d = np.abs(pil - mine)
print("mean abs diff:", d.mean())
print("max abs diff, 10px from the edges:", d[10:-10, 10:-10].max())
print("max abs diff, whole image:", d.max())
```

```console
$ python3 compare.py
mean abs diff: 0.63691396
max abs diff, 10px from the edges: 2.710846
max abs diff, whole image: 25.185959
```

Away from the edges they differ by at most 2.7 out of 255. The large
differences (up to 25) are within 10 pixels of the border. I didn't check
exactly why; `gauss.py` repeats the edge pixel (`mode="edge"` in `np.pad`), and
Pillow's blur is built differently (box blurs), so the border is where they
diverge. If your output has to match Pillow's
pixel for pixel, this isn't a drop-in replacement.

SciPy has `scipy.ndimage.gaussian_filter` for numpy arrays. I didn't test it
here.

## Checklist

- `ValueError: image has wrong mode` from `GaussianBlur`, `BoxBlur` or
  `UnsharpMask`: check `img.mode`. In Pillow 12.3.0 they fail on `F`, `I`
  and `I;16`.
- `Image.fromarray` gives `F` for `float32` and `float64`, `I` for `int32`,
  and `I;16` for `uint16`.
- Scaling to `'L'` works for 0..1 masks where 256 levels are enough. It
  truncates values below 1/255 to 0, and out-of-range values wrap unless you
  `np.clip` first.
- To keep float precision, blur the array in numpy (separable Gaussian
  above). Expect small differences from Pillow's blur, larger near the
  edges.
- `MedianFilter` already works on `F`. `BLUR` works on `I` and `I;16`, but
  not on `F`.

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.12.13, Pillow 12.3.0
and NumPy 2.5.3 (versions from the commands at the top). I didn't test other
Pillow versions; which modes each filter accepts may differ between them.*
