# Word Counter: Why `wc -w` Says 564 Words When My Newsletter Limit Is 500

I write a weekly engineering newsletter with a hard 500-word cap. Last Tuesday I finished a Redis postmortem, ran `wc -w` on the draft, and got 564.

Sixty-four words over. I started cutting a paragraph about our rollback steps.

Then our editor opened the same file and her CMS meter read 468. Same file, same day, two numbers 96 words apart. Neither of us was lying, and we were both measuring something real.

The draft is Markdown: YAML front matter, two fenced code blocks, inline `code` spans, a bulleted follow-up list, a blockquote, and a Chinese summary section near the bottom. `wc` knows about exactly none of that.

## Method 1: strip the markup first (Python 3.11.1)

The only reliable way I've found is to remove everything that isn't prose, then count what's left. Here is the whole script, runnable as-is:

```python
"""Step-by-step Markdown word count. Python 3.11.1"""
import re

FRONT  = re.compile(r"\A---\n.*?\n---\n", re.S)
FENCE  = re.compile(r"^```.*?^```", re.S | re.M)
INLINE = re.compile(r"`[^`]*`")
LINK   = re.compile(r"\[([^\]]*)\]\([^)]*\)")
BARE   = re.compile(r"https?://\S+")
BULLET = re.compile(r"^\s*[-*+]\s+(?:\[[ xX]\]\s*)?", re.M)
HEAD   = re.compile(r"^\s*#{1,6}\s*", re.M)
QUOTE  = re.compile(r"^\s*>\s*", re.M)
EMPH   = re.compile(r"[*_]{1,3}")
WORD   = re.compile(r"[A-Za-z0-9][A-Za-z0-9'\-]*")
CJK    = re.compile(r"[぀-ヿ㐀-䶿一-鿿豈-﫿]")

raw = open("draft.md", encoding="utf-8").read()

steps = []
t = raw
steps.append(("0. raw file", t))
t = FRONT.sub("", t);  steps.append(("1. drop YAML front matter", t))
t = FENCE.sub("", t);  steps.append(("2. drop fenced code blocks", t))
t = INLINE.sub("", t); steps.append(("3. drop inline `code`", t))
t = LINK.sub(r"\1", t); t = BARE.sub("", t)
steps.append(("4. unwrap links, drop bare URLs", t))
t = BULLET.sub("", t); t = HEAD.sub("", t); t = QUOTE.sub("", t)
steps.append(("5. drop list/heading/quote markup", t))
t = EMPH.sub("", t);   steps.append(("6. drop emphasis asterisks", t))

for label, text in steps:
    print(f"{label:<36} wc-w={len(text.split()):>4}  chars={len(text):>5}")

final = steps[-1][1]
latin = len(WORD.findall(final))
cjk = len(CJK.findall(final))
print("\nlatin words         :", latin)
print("CJK characters      :", cjk)
print("prose total         :", latin + cjk)
print("CJK as ~2 chars/word:", latin + round(cjk / 2))
print("reading time @200wpm:", f"{(latin + cjk) / 200:.2f} min")
```

On my draft:

```
0. raw file                          wc-w= 564  chars= 3363
1. drop YAML front matter            wc-w= 543  chars= 3223
2. drop fenced code blocks           wc-w= 501  chars= 2960
3. drop inline `code`                wc-w= 487  chars= 2812
4. unwrap links, drop bare URLs      wc-w= 487  chars= 2736
5. drop list/heading/quote markup    wc-w= 468  chars= 2683
6. drop emphasis asterisks           wc-w= 468  chars= 2677

latin words         : 467
CJK characters      : 95
prose total         : 562
CJK as ~2 chars/word: 515
reading time @200wpm: 2.81 min
```

Step 4 is my favourite: the token count did not move at all, yet 76 characters vanished. `[runbook](https://wiki.acme.run/ratelimit)` becomes `runbook`, so the anchor text stays and the URL goes. A URL was never a word.

Step 6 also moves nothing. `*paying*` loses its asterisks and stays one token. That step exists to make the word regex honest, not to change the count.

The real answer for my newsletter: **467 Latin words**. Under the cap, with room for one more short paragraph. I put the rollback steps back.

## Method 2: hand it to ICU (Node.js 22.13.1)

If you would rather not maintain regexes, `Intl.Segmenter` does the splitting using the same Unicode rules your browser uses.

```js
// Node.js v22.13.1
import { readFileSync } from "node:fs";

const raw = readFileSync("draft.md", "utf8");

const body = raw
  .replace(/^---\n[\s\S]*?\n---\n/, "")
  .replace(/^```[\s\S]*?^```/gm, "")
  .replace(/`[^`]*`/g, "")
  .replace(/^[-*+]\s+(?:\[[ xX]\]\s*)?/gm, "")
  .replace(/^#{1,6}\s*/gm, "")
  .replace(/^\s*>\s*/gm, "");

function count(text, locale = "en-US") {
  const s = new Intl.Segmenter(locale, { granularity: "word" });
  let all = 0, wordLike = 0;
  for (const p of s.segment(text)) {
    all += 1;
    if (p.isWordLike) wordLike += 1;
  }
  return { all, wordLike };
}

console.log("raw split(/\\s+/)      :", raw.trim().split(/\s+/).length);
console.log("body split(/\\s+/)     :", body.trim().split(/\s+/).length);
console.log("segmenter en all      :", count(body).all);
console.log("segmenter en wordLike :", count(body).wordLike);
```

```
raw split(/\s+/)      : 564
body split(/\s+/)     : 468
segmenter en all      : 1124
segmenter en wordLike : 540
```

Two numbers there need explaining. `all` is 1124 because Segmenter emits punctuation and whitespace as segments too; you almost never want that one. And `wordLike` is 540, which is 73 *more* than Python's 467 on identical text, because ICU splits hyphenated identifiers and clock times:

| token | segments | wordLike | split as |
| --- | --- | --- | --- |
| `429s` | 1 | 1 | `429s` |
| `12:00:59` | 5 | 3 | `12` `:` `00` `:` `59` |
| `allkeys-lru` | 3 | 2 | `allkeys` `-` `lru` |
| `maxmemory-policy` | 3 | 2 | `maxmemory` `-` `policy` |
| `don't` | 1 | 1 | `don't` |
| `1.8 ms` | 3 | 2 | `1.8` ` ` `ms` |

Neither 467 nor 540 is wrong. My Python regex treats `maxmemory-policy` as one word because I put `-` inside the character class; ICU treats it as two. Pick one and write it down, because the difference on a technical draft is 15%.

## Method 3: paste it into a browser tab

Most days I am not writing a build step. I am checking one draft, once.

So I paste the body into [a free online word counter](https://utilvy.com/en/tools/word-counter/) and read the number off the page. It gives words, characters, sentences and reading time as you type, and it never uploads the file — which matters when the draft is an incident postmortem with customer IDs in it. I cross-check the word figure against my script once, then trust the sentence count, which is the one number I refuse to regex myself.

If the draft has a Chinese section, I run it through [this browser-based tool](https://utilvy.com/en/tools/word-counter/) a second time with the CJK block selected, because no single "word" definition covers both scripts. See pitfall 1.

## Three things that bit me

### 1. `wc -w` thinks a Chinese sentence is one word

Chinese does not use spaces between words. Take one line from my draft:

```python
cjk = "我们在灰度环境复现了这个问题：限流键没有设置过期时间，导致老用户被永久封禁。"
print(len(cjk))            # 38
print(len(cjk.split()))    # 1
```

Thirty-eight characters, one "word" as far as `wc` is concerned. `Intl.Segmenter` with `zh-Hans` segments the same string into 23 word-like units, which is much closer to how a reader experiences it. My compromise is 2 characters ≈ 1 word, giving 515 instead of 562, and I say so in the byline note.

### 2. Character counts change with the locale

I ran `wc -m` twice on the same file:

```
$ LC_ALL=C       wc -m draft.md
3585 draft.md
$ LC_ALL=C.UTF-8 wc -m draft.md
3363 draft.md
```

Under the `C` locale `wc -m` reports bytes, not characters. The 222-byte gap is the Chinese and emoji. If a CI job asserts on character count, pin `LC_ALL=C.UTF-8` in the job or you will chase a number that moves between your laptop and the runner.

### 3. Read the file as text, explicitly

Both of these are real mistakes I made in the same afternoon:

```
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 2841: ordinal not in range(128)
TypeError: b.trim is not a function
```

The first is `open("draft.md")` with no `encoding=` under a POSIX locale; Python 3.11 picked `ANSI_X3.4-1968` and died on the first Chinese byte. The second is `readFileSync("draft.md")` with no encoding — you get a `Buffer`, and `Buffer` has no `.trim()`. Pass `encoding="utf-8"` and `"utf8"` respectively, and stop trusting defaults.

Emoji have the same problem in miniature. The family emoji at the end of my draft, `👨‍👩‍👧`, is 8 UTF-16 code units, 5 code points, and 1 grapheme:

```
fam.length (UTF-16)  : 8
[...fam].length      : 5
grapheme segments    : 1
```

## Which number to use

| Question | Use | My draft |
| --- | --- | --- |
| Is it under the 500-word cap? | Latin words, markup stripped | 467 |
| How long to read? | prose words ÷ 200 wpm | 2 min 49 sec |
| Does it fit the SMS/DB limit? | UTF-8 bytes | 3,585 |
| How long is the Chinese section? | characters ÷ 2 | 95 chars ≈ 48 words |
| Rough sanity check | `wc -w` | 564 (includes code) |

The lesson I keep relearning: a word count is a definition, not a measurement. Say which definition you used, and the 96-word gap between you and your editor stops being an argument.

If you need the number once, [the word counter on utilvy](https://utilvy.com/en/tools/word-counter/) is faster than any script I've written. If you need it every commit, the Python pipeline above is what runs in our CI.

Segmentation rules and the `isWordLike` flag are documented on [MDN's Intl.Segmenter page](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter); the `wc` flags and their locale behaviour are in the [GNU coreutils manual](https://www.gnu.org/software/coreutils/manual/html_node/wc-invocation.html).

More free browser-based utilities at utilvy.com