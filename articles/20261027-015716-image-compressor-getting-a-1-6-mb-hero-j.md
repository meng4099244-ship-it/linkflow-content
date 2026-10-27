Our marketing site scored 41 on mobile Lighthouse last month. The biggest line item wasn't JavaScript. It was one hero photo: 2400×1600 pixels, 1,709,690 bytes, sitting at the top of the homepage and pushing LCP to 4.8s on a throttled 4G profile.

I had three constraints going in. No build step, because the CMS just takes uploads. No shipping customer photos to a third-party API — legal shot that down in about four seconds. And whatever I produced had to survive a designer looking at it full-screen.

Here's what I tried, in order.

## Method 1: Pillow, in a loop over the upload folder

Python 3.11 with Pillow 12.1.0. The naive version is three lines and gets you most of the way:

```python
from PIL import Image

im = Image.open("hero.jpg")
im.save("hero-q75.jpg", quality=75)
```

That took 1,709,690 bytes down to 500,812. A 71% cut, no visible damage at thumbnail size. But at full width on a 27-inch display, the gradients in the sky banded badly.

The version I kept does two extra things: it caps the long edge first, and it turns on the flags everyone forgets.

```python
from PIL import Image
from pathlib import Path

MAX_W = 1600

def shrink(src: str, dst: str, quality: int = 82) -> None:
    with Image.open(src) as im:
        im = im.convert("RGB")                      # kills alpha and CMYK surprises
        if im.width > MAX_W:
            h = round(im.height * MAX_W / im.width)
            im = im.resize((MAX_W, h), Image.LANCZOS)
        im.save(
            dst,
            "JPEG",
            quality=quality,
            optimize=True,      # rebuilds the Huffman tables, ~5-8% for free
            progressive=True,   # renders in passes, better perceived load
            subsampling=0,      # 4:4:4, avoids colour bleeding on red text
        )

for p in Path("uploads").glob("*.jpg"):
    shrink(str(p), f"out/{p.name}")

print("hero.jpg ->", Path("out/hero.jpg").stat().st_size, "bytes")
```

Here's what each change bought me:

| what I changed | bytes |
| --- | --- |
| original | 1,709,690 |
| `quality=75`, nothing else | 500,812 |
| `quality=82` + `optimize` + `progressive` | 667,825 |
| resized to 1600px wide, then `quality=82` | **197,471** |

Resizing did more work than the quality slider did. Dropping 2400px to 1600px removes 55% of the pixels before the encoder ever sees them, and on a page where the image renders in a 1200px container, nobody can tell.

## Method 2: sharp on Node 22

If your pipeline is already Node, sharp 0.35.5 (libvips 8.18.7) is faster and its mozjpeg build beats Pillow's encoder on the same quality number.

```bash
npm install sharp
```

```js
import sharp from "sharp";
import { statSync } from "node:fs";

const WIDTH = 1600;

async function pipeline(input, output) {
  await sharp(input)
    .rotate()                                  // bakes EXIF orientation into pixels
    .resize({ width: WIDTH, withoutEnlargement: true })
    .jpeg({ quality: 82, progressive: true, mozjpeg: true })
    .toFile(output);
  console.log(output, statSync(output).size, "bytes");
}

await pipeline("hero.jpg", "hero-1600.jpg");
await sharp("hero.jpg").resize({ width: 1600 }).webp({ quality: 80 }).toFile("hero.webp");
await sharp("hero.jpg").resize({ width: 1600 }).avif({ quality: 50 }).toFile("hero.avif");
```

Same source image, same quality setting, different encoder:

- `q82` + mozjpeg, no resize: 424,452 bytes (Pillow's plain q82 was 667,825)
- `webp` q80 at 1600px: 142,526 bytes
- `avif` q50 at 1600px: 42,818 bytes

AVIF at 42 KB is the number that made the designer stop arguing. Support is fine in 2026 (check the [image format table on MDN](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types) for the current list), but I still ship it inside a `<picture>` element with WebP and JPEG fallbacks. "Fine" doesn't cover every embedded browser in the wild.

One thing that surprised me: sharp's WebP at q80 came out at 142,526 bytes while Pillow's came out at 116,320. Same nominal quality, 26 KB apart. Encoder implementations disagree about what `quality=80` means, so treat it as a dial to tune, not a spec.

## Method 3: when you just need one image smaller

Both of the above assume you can run code. Half the time I can't — I'm on a laptop with no repo cloned, or a colleague in marketing is asking why her screenshot won't upload to the CMS.

For those, I use [a browser-based image compressor](https://utilvy.com/en/tools/image-compressor/). You drop the file in, drag a quality slider, and watch the size update. The whole thing runs client-side, which matters for me: nothing gets uploaded, so it passes the same legal review that killed the API idea.

It's also the fastest way to answer "what quality number do we actually need?" I'll push the slider until the artifact shows up, read the number, then hard-code that number into the Python script. Guessing a quality value and iterating in code is a slow loop; dragging a slider with live preview is a fast one. If you want a quick sanity check on a single hero image before a deploy, [that same online tool](https://utilvy.com/en/tools/image-compressor/) is what I reach for, and it handles WebP output too, which the CMS accepts now.

## Three things that bit me

**1. PNGs with transparency explode on save.** My script ran over a folder, hit a logo with an alpha channel, and died:

```
OSError: cannot write mode RGBA as JPEG
```

The fix is the `convert("RGB")` line in Method 1. Note that it fills transparent pixels with black, not white — if your logo is dark-on-transparent, flatten it against white first or you'll get a black box.

**2. Re-compressing the same file doesn't shrink it, it just degrades it.** I saved the output back over the input six times at `quality=82` and measured RMSE against the original:

```
pass 1: size= 696216  RMSE vs original=6.098
pass 2: size= 695064  RMSE vs original=6.245
pass 3: size= 695070  RMSE vs original=6.326
pass 4: size= 694987  RMSE vs original=6.387
pass 5: size= 694961  RMSE vs original=6.442
pass 6: size= 694971  RMSE vs original=6.486
```

Size barely moved: 1,245 bytes across five extra passes, while error kept climbing. A file that stops getting smaller will happily keep getting worse. Always write to a new path and never overwrite your source.

**3. EXIF orientation vanishes silently.** Phones store rotation as a tag, not as rotated pixels. Save without passing it through and the tag is gone:

```
after naive save: exif bytes=20  orientation=None
```

The photo was 32 bytes of EXIF with `Orientation=6` before, empty after. Every vertical phone photo now renders on its side. `ImageOps.exif_transpose()` in Python and `.rotate()` in sharp both bake the rotation into the pixels before encoding, which is what you want.

## Summary

| approach | best for | result on my 2400×1600 test image |
| --- | --- | --- |
| Pillow, resize + q82 | batch jobs, existing Python tooling | 197,471 bytes (JPEG) |
| sharp, mozjpeg | Node pipelines, best JPEG bytes | 424,452 bytes, no resize |
| sharp, AVIF q50 | modern browsers, smallest file | 42,818 bytes |
| [the Utilvy image compressor](https://utilvy.com/en/tools/image-compressor/) | one-offs, no install, private files | slider-driven, instant |

My rule now: cap the long edge before touching quality, always write to a new file, and let the browser pick AVIF when it can. That hero went from 1.7 MB to 42 KB, and LCP dropped to 1.9s. [More on why that metric moved is on web.dev](https://web.dev/articles/optimize-lcp).

More free browser-based utilities at utilvy.com