# Color Converter: Getting Hover and Dark-Mode Variants From One HEX Code in Python, Node, and the Browser

Our designer handed me a single value for the new brand blue: `#5B8DEF`. That was the whole handoff.

No hover state, no disabled state, nothing for dark mode. I needed four more shades by Friday, so I did what most of us do first: I nudged the last two hex characters around in DevTools until something looked lighter.

On my calibrated monitor it looked fine. On the QA engineer's cheap panel the hover state was invisible against the button. That's when I stopped eyeballing hex codes and started converting them.

## Method 1: Python 3.11 with `colorsys`

`colorsys` ships with the standard library, so there is nothing to install. One thing to know before you start: the module speaks HLS, not HSL. The function is `rgb_to_hls`, and it returns `(h, l, s)` in that order.

```python
# color_variants.py — Python 3.11.1
import colorsys

def hex_to_rgb(value: str) -> tuple[int, int, int]:
    h = value.lstrip('#')
    if len(h) == 3:
        h = ''.join(c * 2 for c in h)
    n = int(h, 16)
    return (n >> 16) & 255, (n >> 8) & 255, n & 255

def rgb_to_hex(rgb: tuple[int, int, int]) -> str:
    return '#{:02X}{:02X}{:02X}'.format(*rgb)

def shift_lightness(value: str, delta: float) -> str:
    r, g, b = hex_to_rgb(value)
    h, l, s = colorsys.rgb_to_hls(r / 255, g / 255, b / 255)
    l2 = min(1.0, max(0.0, l + delta / 100))
    return rgb_to_hex(tuple(round(c * 255) for c in colorsys.hls_to_rgb(h, l2, s)))

if __name__ == '__main__':
    r, g, b = hex_to_rgb('#5B8DEF')
    h, l, s = colorsys.rgb_to_hls(r / 255, g / 255, b / 255)
    print('rgb  ', (r, g, b))
    print('hsl  ', [round(h * 360, 2), round(s * 100, 2), round(l * 100, 2)])
    for label, d in [('base', 0), ('hover', 8), ('dark-mode', -25)]:
        print(f'{label:10} {shift_lightness("#5B8DEF", d)}')
```

Real output from my terminal:

```
rgb   (91, 141, 239)
hsl   [219.73, 82.22, 64.71]
base       #5B8DEF
hover      #80A7F3
dark-mode  #124AB9
```

So `#5B8DEF` is `hsl(219.73, 82.22%, 64.71%)`. "Make it eight percent lighter" is now a number I can reproduce, and I can hand back three hex codes instead of three screenshots.

## Method 2: Node.js 22, hand-rolled

Node has no color module in the standard library. When the palette has to be generated during a frontend build, I write the two conversions myself.

```js
// color-variants.mjs — Node.js 22.13.1
const hexToRgb = hex => {
  let h = hex.replace('#', '').trim();
  if (h.length === 3) h = h.split('').map(c => c + c).join('');
  const n = parseInt(h, 16);
  return [(n >> 16) & 255, (n >> 8) & 255, n & 255];
};

const rgbToHsl = ([r, g, b]) => {
  r /= 255; g /= 255; b /= 255;
  const max = Math.max(r, g, b), min = Math.min(r, g, b);
  const l = (max + min) / 2, d = max - min;
  let h = 0, s = 0;
  if (d !== 0) {
    s = d / (1 - Math.abs(2 * l - 1));
    h = max === r ? ((g - b) / d) % 6 : max === g ? (b - r) / d + 2 : (r - g) / d + 4;
    h = (h * 60 + 360) % 360;
  }
  return [h, s * 100, l * 100];
};

const hslToRgb = (h, s, l) => {
  h = ((h % 360) + 360) % 360; s /= 100; l /= 100;
  const c = (1 - Math.abs(2 * l - 1)) * s;
  const x = c * (1 - Math.abs(((h / 60) % 2) - 1));
  const m = l - c / 2;
  return [[c,x,0],[x,c,0],[0,c,x],[0,x,c],[x,0,c],[c,0,x]][Math.floor(h / 60) % 6]
    .map(v => Math.round((v + m) * 255));
};

const toHex = a => '#' + a.map(v => v.toString(16).padStart(2, '0').toUpperCase()).join('');

const [h, s, l] = rgbToHsl(hexToRgb('#5B8DEF'));
console.log('base      ', toHex(hslToRgb(h, s, l)));
console.log('hover     ', toHex(hslToRgb(h, s, l + 8)));
console.log('dark-mode ', toHex(hslToRgb(h, s, l - 25)));
```

```
base       #5B8DEF
hover      #80A7F3
dark-mode  #124AB8
```

Look at that last line. Python said `#124AB9`, Node says `#124AB8`. Same math, one unit apart, because rounding lands in a different place in each implementation. Invisible on screen, but any test comparing hex strings across languages needs a tolerance.

## Method 3: a free online color converter

Most of the time I'm not in a build step. I'm on a call, the designer says "try it ten percent lighter," and opening a REPL is slower than just looking at it.

I paste the value into [a free online color converter](https://utilvy.com/en/tools/color-converter/) and read HEX, RGB and HSL side by side with a live preview. Drag the lightness slider, read off the hex, paste it into the stylesheet. Ten seconds, nothing to install, and the palette never leaves my browser.## Three things that bit me

**1. `colorsys` returns HLS, and unpacking it as HSL washes the color out**

This one cost me an afternoon:

```python
h, s, l = colorsys.rgb_to_hls(0.357, 0.553, 0.937)   # wrong names
# h=219.73  s=64.71  l=82.22
```

The second value is lightness, not saturation. Feed that back with `+8` on what you assume is lightness and `#5B8DEF` comes out as `#D6E1F6`, a pale ghost of the brand blue. If your "slightly lighter" shade looks washed out, check the unpack order before anything else.

**2. Three-digit shorthand breaks naive parsers**

`#5BE` is valid CSS and means `#55BBEE`, but a parser that slices two characters at a time disagrees:

```
>>> tuple(int('5BE'[i:i+2], 16) for i in (0, 2, 4))
ValueError: invalid literal for int() with base 16: ''
```

Node fails quietly instead, which is worse:

```js
> parseInt('5BE', 16)
1470
// (1470 >> 16) & 255 = 0, (1470 >> 8) & 255 = 5, 1470 & 255 = 190  →  #0005BE
```

`#0005BE` is a near-black navy. Your brand-blue button renders almost black and nobody notices until staging. Expand the shorthand before parsing.

**3. The HSL numbers you write down are not the hex you started with**

Round `hsl(219.73, 82.22%, 64.71%)` to values a designer can say out loud, `hsl(220, 82%, 65%)`, then convert back:

```
hsl(220, 82%, 65%) → #5D8DEF
```

Two steps off in the red channel from `#5B8DEF`. The drift is one-time: six more round trips stay at `#5D8DEF`. Keep the hex as the source of truth and treat HSL as a working value you throw away.

Same trap, different direction. Lightening lightness by 8 gives `#80A7F3`, while `color-mix(in srgb, #5B8DEF 92%, white)` gives `#6896F0`. Both are "eight percent lighter." They are not the same color. Pick one rule for the design system and write it down; the syntax is covered in [MDN's `<color>` reference](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value) and the [CSS Color Module Level 4 spec](https://www.w3.org/TR/css-color-4/).

## Which one I reach for

| Situation | Tool | What I get from `#5B8DEF` |
|---|---|---|
| Live on a call, answer needed now | [this browser-based color tool](https://utilvy.com/en/tools/color-converter/) | HEX / RGB / HSL with preview |
| Generating a palette in CI | Python 3.11 `colorsys` | hover `#80A7F3`, dark `#124AB9` |
| Palette baked into a frontend build | Node 22 hand-rolled | hover `#80A7F3`, dark `#124AB8` |
| Theming directly in CSS | `color-mix()` | `#6896F0` at 8% white |

The hex is the contract. Everything else is a view of it.

When I just need all three notations in front of me, the [HEX/RGB/HSL converter I keep bookmarked](https://utilvy.com/en/tools/color-converter/) beats writing a script.

More free browser-based utilities at utilvy.com