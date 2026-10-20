Our editor filed a story called *"Größe Straße — 5 Tips"* on a Tuesday. By Wednesday morning the page was 404ing, and the URL read `/blog/groe-strae-5-tips`. Half the street name had evaporated.

Slugs look like the simplest part of a CMS until the first non-English headline arrives. Mine is a Python 3.11.1 backend with a Node 22.13.1 build step, and the two disagreed about that string in completely different ways. I lost an afternoon to it. The cause was Unicode, which has more edge cases than any regex I've ever written.

A slug is the readable tail of a URL. `/blog/grosse-strasse-5-tips` beats `/blog/18473` for humans and crawlers alike. The rules are strict, though: lowercase, ASCII, hyphens for spaces, nothing a URL parser would read as syntax. Characters like `?`, `#` and `%` carry meaning in a URL, so leaving them in is asking for trouble. MDN's [percent-encoding](https://developer.mozilla.org/en-US/docs/Glossary/Percent-encoding) page lists exactly which ones.

## Method 1: Python 3.11, standard library only

This is the version running in my CMS today. No dependencies.

```python
# slug.py — Python 3.11.1, standard library only
import re
import unicodedata

def slugify(title: str) -> str:
    text = unicodedata.normalize("NFKD", title.casefold())
    text = "".join(c for c in text if not unicodedata.combining(c))
    text = text.encode("ascii", "ignore").decode("ascii")
    text = re.sub(r"[^a-z0-9]+", "-", text)
    return text.strip("-")

if __name__ == "__main__":
    for t in ["Größe Straße — 5 Tips", "Café & Crème", "Release 🚀 v2.0 is out!"]:
        print(f"{t!r:32} -> {slugify(t)!r}")
```

Run it and you get:

```
'Größe Straße — 5 Tips'          -> 'grosse-strasse-5-tips'
'Café & Crème'                   -> 'cafe-creme'
'Release 🚀 v2.0 is out!'         -> 'release-v2-0-is-out'
```

Three steps, and the order is not negotiable. `casefold()` first, then NFKD normalization splits `ö` into `o` plus a combining diaeresis, then I drop the combining marks and discard whatever is still not ASCII.

If you would rather not maintain that, [`python-slugify`](https://pypi.org/project/python-slugify/) 9.1.3 does the same job and adds real transliteration:

```python
from slugify import slugify

slugify("Größe Straße — 5 Tips")            # 'grosse-strasse-5-tips'
slugify("你好世界 Hello World")                # 'ni-hao-shi-jie-hello-world'
slugify("你好世界 Hello", allow_unicode=True)   # '你好世界-hello'
```

That pinyin output is the one thing my stdlib version cannot do. Handed a Chinese headline it returns `hello-world` and silently drops four characters. I keep both around: stdlib for the API, the library for the import script that pulls in our Chinese-language archive. The normalization rules behind all of this live in [Unicode TR #15](https://unicode.org/reports/tr15/), if you want the source of truth instead of my summary.

## Method 2: a shell function, and Node 22

For renaming a batch of markdown files I don't write a script. I use `iconv`, which has been transliterating since long before I started doing this:

```bash
slug() {
  printf '%s' "$1" \
    | iconv -f utf-8 -t ascii//TRANSLIT \
    | tr '[:upper:]' '[:lower:]' \
    | sed -E 's/[^a-z0-9]+/-/g; s/^-+|-+$//g'
}

slug "Größe Straße — 5 Tips"   # grosse-strasse-5-tips
slug "Café & Crème"            # cafe-creme
```

`iconv` turns the em dash into `--`, and the `sed` call collapses that run into a single hyphen. Worth knowing: `//TRANSLIT` is a glibc feature. On Alpine with musl it quietly does nothing, which cost me a confused hour on a CI box last year.

Node 22.13.1 ships no built-in slugify, and the obvious `.normalize("NFKD")` one-liner is wrong. It leaves `ß` alone, `ß` is not ASCII, so it becomes a hyphen and you get `gro-e-stra-e-5-tips`. Map `ß` to `ss` first:

```js
// Node 22.13.1 — no dependencies
const slugify = (title) =>
  title
    .toLowerCase()
    .replace(/ß/g, "ss")
    .normalize("NFKD")
    .replace(/[\u0300-\u036f]/g, "")
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-+|-+$/g, "");

console.log(slugify("Größe Straße — 5 Tips")); // grosse-strasse-5-tips
console.log(slugify("Café & Crème"));          // cafe-creme
```

## Method 3: skip the code

Most of the time I'm not writing code at all. I'm looking at a headline in a draft doc and I want the slug now, before I paste it into the CMS.

That's when I open [a free online slug generator](https://utilvy.com/en/tools/slug-generator/). Paste the title, choose hyphen or underscore, copy the result. It handles the Unicode cases above and keeps non-Latin scripts when you want them kept, which my stdlib script cannot. Everything runs in the browser, so an embargoed headline never touches somebody else's server.

## Three ways this bites you

**1. `lower()` is not `casefold()`.** They differ on exactly one character that matters here, and it's `ß`. `lower()` leaves it alone; `casefold()` expands it to `ss`. Push "Größe Straße" through both:

```
ascii-ignore + lower()    : groe strae  5 tips
ascii-ignore + casefold() : grosse strasse  5 tips
```

In the first line the `ß` is deleted outright, because it isn't ASCII and NFKD does not decompose it. That is precisely how I shipped `/blog/groe-strae-5-tips`.

**2. A CJK headline can produce an empty slug.** Every character in 你好世界 is non-ASCII, so an ASCII-only path wipes all of it:

```python
>>> slugify("你好世界")
''
```

Your template then renders `https://example.com/blog/`, and every Chinese post collapses onto the index. Guard against it: fall back to transliteration or a surrogate ID when the result comes back empty.

**3. Two different titles, one slug.** Diacritics are not identity:

```python
>>> slugify("Café & Crème")
'cafe-creme'
>>> slugify("Cafe  Creme!!")
'cafe-creme'
```

If the `slug` column is `UNIQUE`, the second insert dies:

```
sqlite3.IntegrityError: UNIQUE constraint failed: posts.slug
```

Append a counter or a short hash on collision. And keep the slug stable afterwards, because a slug that changes when someone fixes a typo breaks every link already pointing at it.

## Which one to reach for

| Situation | Pick | Why |
|---|---|---|
| Backend generating slugs at write time | Python stdlib, `casefold()` then NFKD | No dependency, deterministic, three lines |
| Importing non-Latin archives | `python-slugify` 9.1.3 | Real transliteration, `你好世界` becomes `ni-hao-shi-jie` |
| Renaming files or a CI step | `iconv //TRANSLIT` plus `sed` | One pipe, glibc only |
| Build step in JavaScript | Node 22 with an explicit `ß` to `ss` rule | The naive NFKD one-liner yields `gro-e-stra-e-5-tips` |
| One headline, right now | [this browser-based tool](https://utilvy.com/en/tools/slug-generator/) | Zero setup, nothing uploaded |

Before you trust any of these, feed them a German street name, a Chinese headline, and a title containing an emoji. Those three inputs caught every bug I had. When I need a quick second opinion on a tricky title, [utilvy's slug generator](https://utilvy.com/en/tools/slug-generator/) is usually the fastest check.

More free browser-based utilities at utilvy.com