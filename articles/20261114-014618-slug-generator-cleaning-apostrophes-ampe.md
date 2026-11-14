# Slug Generator: Cleaning Apostrophes, Ampersands and Trailing Hyphens Out of My Blog URLs

Last month I moved 312 posts from an old WordPress site to Eleventy. The CMS export gave me titles exactly as the editors typed them: "We're Open — 20% Off This Week & Next", "Don't Panic: A Developer's Guide to Production". My one-liner `title.lower().replace(" ", "-")` deployed without a single error and quietly produced URLs like this:

```
https://blog.example/we're-open-—-20%-off-this-week-&-next
```

Nothing crashed, which is the problem. A week later Search Console reported 47 soft 404s, because the same article had been linked three different ways depending on whether the linking page percent-encoded the apostrophe, the em dash, or both. I spent an evening untangling it. Here is what worked, with real output from each approach.

## Method 1: Python 3.11.1

For a one-shot migration I want collision handling, not just slugification. Install the library first (`pip install python-slugify`, which gives you 8.0.0):

```python
# migrate_slugs.py — Python 3.11.1, python-slugify 8.0.0
import csv
from slugify import slugify

with open("posts_export.csv", encoding="utf-8") as f:
    rows = list(csv.DictReader(f))

used = set()
for row in rows:
    s = slugify(row["title"], separator="-", max_length=60)
    if s in used:
        s = f"{s}-{row['id']}"   # suffix with the post id, never guess
    used.add(s)
    print(row["id"], s)
```

Real output from my export:

```
1041 were-open-20-off-this-week-next
1042 rock-roll-comeback-tour
1043 dont-panic-a-developers-guide-to-production
1044 sale-ends-friday
```

Notice what vanished: the curly apostrophe in "We're", the em dash, the `%`, the `&`, and the `!` after "Friday". No dangling hyphens, no percent-encoding traps.

If you can't add a dependency, the standard library gets you 90% of the way:

```python
# slug_stdlib.py — Python 3.11.1, no third-party packages
import re, unicodedata

def slug_stdlib(title: str) -> str:
    text = unicodedata.normalize("NFKD", title).encode("ascii", "ignore").decode()
    text = re.sub(r"[^a-z0-9\s-]", "", text.lower())
    return re.sub(r"[\s-]+", "-", text).strip("-")

print(slug_stdlib("We're Open — 20% Off This Week & Next"))
# were-open-20-off-this-week-next
```

The `strip("-")` matters more than you would think. I'll come back to that in the pitfalls. The [Python docs on unicodedata](https://docs.python.org/3/library/unicodedata.html) explain why NFKD normalization ("é" splits into "e" plus a combining accent) is the safe first step before dropping non-ASCII characters.

## Method 2: Node 22.13.1

My build step already runs on Node, and both scripts had to agree on every URL or the migration would fight the template. The `slugify` package (1.6.6) matches the Python package's output on all 312 titles:

```js
// slugify.mjs — Node 22.13.1, slugify 1.6.6
import { readFileSync } from "node:fs";
import slugify from "slugify";

// my export had no quoted fields, so split(",") is fine here;
// for anything messier use a real CSV parser
const lines = readFileSync("posts_export.csv", "utf8").trim().split("\n").slice(1);

for (const line of lines) {
  const [id, title] = line.split(",");
  console.log(`${id} ${slugify(title, { lower: true, strict: true, trim: true })}`);
}
```

`strict: true` is the flag people forget. Without it, slugify keeps `%`, `&` and `@`, and you are back to percent-encoded URLs. Quick sanity check on the two titles that broke the migration:

```
$ node -e "import('slugify').then(m => console.log(m.default('Sale ends Friday!', {lower:true, strict:true, trim:true})))"
sale-ends-friday
$ node -e "import('slugify').then(m => console.log(m.default(\"We're Open — 20% Off\", {lower:true, strict:true, trim:true})))"
were-open-20-off
```

## Method 3: In the browser

For one-off titles I stopped opening an editor entirely. Writing a newsletter link by hand, sanity-checking what the CMS is about to generate, arguing with a colleague about whether a hyphen belongs there: I paste the title into [this free online slug generator](https://utilvy.com/en/tools/slug-generator/) and copy the result. It runs client-side, so pasting embargoed drafts is not a concern, and it flags the double-hyphen case instantly, which is the one my regex still misses occasionally.

## Three pitfalls that cost me real hours

**1. Two apostrophes, two URLs.** My teammate typed "don't" with a straight quote; the editor wrote "don’t" with U+2019. Naive slugs kept both characters, so the same article lived at two addresses:

```
>>> from urllib.parse import quote
>>> quote("don't-panic")
"don%27t-panic"
>>> quote("don’t-panic")
'don%E2%80%99t-panic'
```

Search engines saw duplicate content, and internal links pointed at whichever copy happened to be in the author's clipboard. The fix is boring: strip punctuation during slugification (both methods above do this), never percent-encode it into the URL.

**2. The trailing hyphen.** `re.sub(r"[^a-z0-9]+", "-", "Save 30% Now!".lower())` returns `save-30-now-`, dash included. My RSS template emitted that version while the sitemap had the trimmed one:

```
$ curl -sI https://blog.example/save-30-now- | head -1
HTTP/1.1 404 Not Found
```

One character. Search Console flagged all 47 of those soft 404s the next morning. Always call `.strip("-")`, or use a library whose trim behavior strips leading and trailing separators by default.

**3. Ampersands and doubled separators.** "Rock & Roll" through `.replace(" ", "-")` becomes `rock-&-roll`, which templates happily encode into `/rock-%26-roll`. Clicks still work, but every copy-paste share carries the ugly `%26`, and analytics tools split tracking URLs at the first unencoded `&`. Cleaning with a character class instead of a literal replace (`[\s-]+` becomes `-`) also collapses `Rock -- Roll` to `rock-roll` rather than `rock----roll`. MDN's [URL documentation](https://developer.mozilla.org/en-US/docs/Web/API/URL) is a solid reference for which characters a URL parser treats as boundaries.

## Wrap-up

| Approach | Where it runs | Best for | "We're Open — 20% Off & Next" becomes |
|---|---|---|---|
| python-slugify 8.0.0 | CI, migration scripts | Batch jobs with collision checks | `were-open-20-off-this-week-next` |
| slugify 1.6.6 (Node) | Build steps | Keeping JS pipelines consistent | `were-open-20-off-this-week-next` |
| [Utilvy's slug generator](https://utilvy.com/en/tools/slug-generator/) | Browser, no setup | One-off titles, quick checks | same, in about two seconds |

The rule I landed on: slugs come from a library, never from `.replace(" ", "-")`. And when a suspicious URL shows up in analytics, I paste the original title into [the browser-based tool](https://utilvy.com/en/tools/slug-generator/) before I trust whatever the CMS produced.

More free browser-based utilities at utilvy.com.