# Word Counter for Writers: Counting Words, Characters and Reading Time in Python, Shell, and the Browser

## Three tools, three different numbers

I shipped a newsletter draft last Tuesday at 1,097 words. Or so my editor claimed.

An hour later the editor pinged me: "this is 1,140 words, our limit is 800." I opened the same file in Google Docs and got 1,142. Neither number matched mine.

That is not a bug. It is three different definitions of the word "word" quietly disagreeing with each other.

The 800 limit was real, by the way. Their mail client truncates anything longer, so every word past the line is a word nobody ever reads. Once you know which definition your publisher uses, you stop arguing and start shipping.

Here is the sample file I use to test any counting tool. Save it as `draft.txt` and every number below is reproducible on your machine.

```text
I shipped the newsletter draft last Tuesday and the editor pinged me an hour later: "this is 1,140 words, our limit is 800."

That was the third time that month. My editor counts words in Google Docs, I count them in VS Code, and the two numbers never match. Docs says 812, my status bar says 797.

The gap comes from hyphenated words and the em dash — things like "state-of-the-art" or "well — actually". Docs splits on the dash, my editor does not.
```

## Method 1: a Python script you actually keep

`wc` is fine for a quick check, but it cannot tell me reading time, and it counts Markdown fences as prose. So I wrote a script I now carry between projects.

Versions used here: **Python 3.11.1** (CPython, Linux x86_64).

```python
#!/usr/bin/env python3
"""wordcount.py - count words, characters, sentences and reading time."""
import re
import sys
import unicodedata

WORD_RE = re.compile(r"[^\s\u200b]+")
SENTENCE_RE = re.compile(r"[^.!?\n]+[.!?]+", re.MULTILINE)
WPM = 200  # average adult reading speed for English prose


def normalise(text: str) -> str:
    # NFC folds "e" + U+0301 into "é", so one accented letter counts once
    return unicodedata.normalize("NFC", text)


def strip_markup(text: str) -> str:
    text = re.sub(r"```.*?```", "", text, flags=re.DOTALL)  # fenced code
    text = re.sub(r"`[^`\n]+`", "", text)                   # inline code
    text = re.sub(r"!?\[([^\]]*)\]\([^)]*\)", r"\1", text)  # links -> anchor
    text = re.sub(r"^#{1,6}\s*", "", text, flags=re.MULTILINE)
    return text


def count(path: str) -> dict:
    raw = open(path, encoding="utf-8").read()
    text = normalise(strip_markup(raw))
    words = [w for w in WORD_RE.findall(text) if any(c.isalnum() for c in w)]
    minutes = len(words) / WPM
    return {
        "words": len(words),
        "chars": len(text),
        "chars_no_space": len(re.sub(r"\s", "", text)),
        "sentences": len(SENTENCE_RE.findall(text)),
        "minutes": minutes,
        "read_time": f"{int(minutes)}:{int(round((minutes % 1) * 60)):02d}",
        "bytes": len(raw.encode("utf-8")),
    }


if __name__ == "__main__":
    if len(sys.argv) != 2:
        sys.exit("usage: wordcount.py FILE")
    try:
        r = count(sys.argv[1])
    except UnicodeDecodeError as exc:
        sys.exit(f"UnicodeDecodeError: {exc}\nHint: this file is not UTF-8. "
                 "Try: iconv -f latin1 -t utf-8 FILE")
    print(f"words           {r['words']}")
    print(f"characters      {r['chars']}")
    print(f"characters (no spaces)  {r['chars_no_space']}")
    print(f"sentences       {r['sentences']}")
    print(f"reading time    {r['read_time']}  ({r['minutes']:.2f} min @ {WPM} wpm)")
    print(f"utf-8 bytes     {r['bytes']}")
```

Run it:

```bash
python3 wordcount.py draft.txt
```

```text
words           82
characters      451
characters (no spaces)  365
sentences       6
reading time    0:25  (0.41 min @ 200 wpm)
utf-8 bytes     455
```

The `strip_markup` step matters more than I expected. On a short Markdown post with one Python code block, `wc -w` reported 47 words while the script reported 36. Eleven phantom words, all of them `def`, `parse`, `return`. If you bill clients per word, that difference is invoice noise you do not want.

The `normalise` step is the other half. Without NFC normalisation, `"e" + U+0301` reads as two characters and my character count drifts upward on any file with pasted French or Spanish text.

## Method 2: one line in the shell

Sometimes I just want a number and I do not want to find the script. GNU coreutils 9.4 is on this box, and `wc` plus `awk` is enough.

```bash
awk '{n+=NF} END{printf "words: %d\nreading time: %d:%02d\n", n, n/200, (n*60/200)%60}' draft.txt
```

```text
words: 84
reading time: 0:25
```

Pure shell if you prefer no `awk`:

```bash
words=$(wc -w < draft.txt)
printf 'words: %s\nreading time: %d:%02d\n' "$words" $((words/200)) $(((words*60/200)%60))
```

Both print 84. My Python script printed 82.

Same file, different answer. That discrepancy is the whole point of this article, and I get to it in the gotchas below.

If you live in Node instead, v22.13.1 gives you the same 82:

```bash
node -e 'const fs=require("fs");const t=fs.readFileSync("draft.txt","utf8").normalize("NFC");const w=t.split(/\s+/).filter(x=>/[A-Za-z0-9]/.test(x));console.log(`words: ${w.length}`);'
```

The `normalize("NFC")` call there is the JavaScript twin of Python's `unicodedata.normalize`; MDN documents it under [`String.prototype.normalize()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/normalize).

## Method 3: paste it into a browser tab

The script is great until I am on a locked-down laptop, reviewing someone else's copy in a browser, with no terminal and no permission to install anything.

That is when I open [a free online word counter](https://utilvy.com/en/tools/word-counter/) and paste the text in. It gives me words, characters, sentences and reading time in one panel, and it updates as I type.

That is why I reach for it with work copy. Nothing gets uploaded, so I can paste a draft that is still under NDA without emailing legal first. It matters most when the text is an unreleased product announcement.

Two jobs it handles for me:

- Checking a meta description against a hard limit. One I wrote last week came out at 116 characters, safely under the 155 that most search snippets show before they truncate
- Sanity-checking a colleague's submitted draft without asking them to install Python

When I need the [browser-based word counter](https://utilvy.com/en/tools/word-counter/) to agree with my script, I paste plain text only. Feed it raw Markdown and you get the fence problem all over again.

## Four things that will bite you

**1. Punctuation-only tokens inflate the count.** My sample has two em dashes (U+2014). `wc -w` counts each as a word, so it says 84. The Python filter drops any token with no alphanumeric character, so it says 82.

Neither is wrong. They answer different questions. If your publisher counts like `wc`, quote 84.

**2. Bytes are not characters.** On the same file, `wc -m` reports 451 and `wc -c` reports 455. Four extra bytes, because each em dash is three bytes in UTF-8.

I hit this the annoying way while checking a 280-character social post. I ran `wc -c`, saw a number above the limit, and trimmed six words that did not need trimming. The platform counted characters. Use `-m`, not `-c`.

**3. A non-UTF-8 file crashes the whole thing.** Files exported from older Windows tools love Latin-1. Mine did:

```text
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 3: invalid continuation byte
Hint: this file is not UTF-8. Try: iconv -f latin1 -t utf-8 FILE
```

Convert first, then count:

```bash
iconv -f latin1 -t utf-8 latin1.txt > latin1-fixed.txt
python3 wordcount.py latin1-fixed.txt
```

After conversion the same file reports 3 words, 18 characters, 22 UTF-8 bytes. Before it, I got a traceback and no number at all.

**4. Abbreviations break sentence counting.** My `SENTENCE_RE` is a naive regex, not a real segmenter. Feed it `"Dr. Smith arrived at 5 p.m. He left after 10 min. I stayed."` and it returns 5 matches for 3 sentences:

```text
'Dr.'
'Smith arrived at 5 p.'
'm.'
'He left after 10 min.'
'I stayed.'
```

I do not trust the sentence field on anything with abbreviations in it. The word and character numbers stay fine; treat sentences as a rough signal.

## Which one when

| Situation | Use | Gives you |
| --- | --- | --- |
| Billing or hard limits, need repeatability | `wordcount.py` (Python 3.11) | Words, chars, sentences, reading time, bytes, markup stripped |
| Quick check in a terminal you already have | `wc -w` + `awk` | Words and reading time, one line |
| Locked-down machine, someone else's copy | [browser word counter](https://utilvy.com/en/tools/word-counter/) | All four metrics, no install, nothing uploaded |
| Text with accents or combined marks | Either, but normalise to NFC first | Stable character counts |

One last thing I got wrong for a year: 200 wpm is a rate for English prose. Apply it to Chinese or Japanese text and the reading time is nonsense, because those scripts are not delimited by spaces and the whole word-segmentation model changes. Unicode's own rules for this live in [UAX #29](https://unicode.org/reports/tr29/), which is worth a skim if you ever build a counter for CJK.

Pick one method, run it on the same sample file every time, and the 1,097-versus-1,142 argument disappears.

More free browser-based utilities at utilvy.com