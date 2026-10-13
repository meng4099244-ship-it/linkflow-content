I inherited a Go service in March that returned JSON like this:

```json
{
  "user_id": 8821,
  "created_at": "2026-03-04T11:20:00Z",
  "profile_image_url": "https://cdn.example.com/a.png",
  "is_2fa_enabled": true
}
```

Our frontend is TypeScript. Every component wanted `userId`, `createdAt`, `profileImageUrl`. Forty-plus endpoints, all snake_case, all needing to cross that boundary.

My first instinct was find-and-replace in the editor. That lasted about ninety seconds before I hit `is_2fa_enabled`.

Someone on the team suggested `camelcase-keys`, an npm package that does the conversion at runtime. I tried it and backed out. It walks every response object on every request, which is measurable once a list endpoint returns 500 rows, and it pushes failures into production instead of build time. I wanted the rename to happen once, in a migration script I could diff and revert.

## Method 1: a Python script that walks the JSON (Python 3.11)

Renaming top-level keys is easy. The annoying part is nested objects and arrays of objects, so I wrote a recursive walker.

Before trusting it with 40 endpoints, I pinned the weird cases in a test file. Three assertions caught two bugs:

```python
# test_recase.py — pytest 9.0.2
from recase import snake_to_camel

def test_plain():
    assert snake_to_camel("created_at") == "createdAt"

def test_acronym_stays_shouted():
    assert snake_to_camel("parse_xml_http_response") == "parseXMLHTTPResponse"

def test_leading_digit_segment():
    assert snake_to_camel("is_2fa_enabled") == "is2FAEnabled"
```

The middle one failed on my first attempt because `xml` wasn't in the allowlist yet. That is exactly the kind of bug you want to find in a test and not in a staging payload.

```python
# recase.py — Python 3.11.1
import json
import re
import sys

# Acronyms we refuse to split apart.
KEEP_UPPER = {"2fa", "http", "xml", "api", "ip", "utc"}

def snake_to_camel(key: str) -> str:
    parts = [p for p in key.split("_") if p]
    if not parts:
        return key
    head = parts[0]
    out = head if head in KEEP_UPPER else head.lower()
    for p in parts[1:]:
        out += p.upper() if p in KEEP_UPPER else p.capitalize()
    return out

def walk(node):
    if isinstance(node, dict):
        return {snake_to_camel(k): walk(v) for k, v in node.items()}
    if isinstance(node, list):
        return [walk(v) for v in node]
    return node

if __name__ == "__main__":
    data = json.load(sys.stdin)
    json.dump(walk(data), sys.stdout, indent=2, ensure_ascii=False)
    print()
```

Run it against the sample above:

```console
$ echo '{"user_id":8821,"is_2fa_enabled":true,"profile_image_url":"x"}' | python3 recase.py
{
  "userId": 8821,
  "is2FAEnabled": true,
  "profileImageUrl": "x"
}
```

The `KEEP_UPPER` set is the whole point. Without it, `is_2fa_enabled` collapses to `is2faEnabled`.

My first version listed `id` and `url` in there too. That was wrong, and the output told me so immediately: `userID` and `profileImageURL`, when the TypeScript interfaces said `userId` and `profileImageUrl`. Id and URL are acronyms in prose but ordinary Pascal segments in this codebase's naming. So the list holds only the terms that are *always* shouted: `HTTP`, `XML`, `API`, `IP`, `UTC`, `2FA`.

Nested objects and arrays come along for free:

```console
$ echo '{"user_id":1,"items":[{"order_id":9,"ship_utc_ts":"2026-01-01"}],"parse_xml_http_response":true}' | python3 recase.py
{
  "userId": 1,
  "items": [
    {
      "orderId": 9,
      "shipUTCTs": "2026-01-01"
    }
  ],
  "parseXMLHTTPResponse": true
}
```

Note I used `p.capitalize()`, not `p.title()`. That distinction matters, and I'll show you why in the gotchas.

## Method 2: Node, for when the repo is already JS (Node 22.13.1)

If the conversion has to live in the build pipeline next to the TypeScript, doing it in Python means adding a toolchain. Node is already there.

```javascript
// recase.mjs — Node 22.13.1
import { readFileSync } from "node:fs";

const KEEP_UPPER = new Set(["2fa", "http", "xml", "api", "ip", "utc"]);

const toCamel = (key) =>
  key
    .split("_")
    .filter(Boolean)
    .map((p, i) => {
      if (KEEP_UPPER.has(p)) return i === 0 ? p : p.toUpperCase();
      return i === 0 ? p.toLowerCase() : p[0].toUpperCase() + p.slice(1).toLowerCase();
    })
    .join("");

const walk = (n) =>
  Array.isArray(n)
    ? n.map(walk)
    : n && typeof n === "object"
      ? Object.fromEntries(Object.entries(n).map(([k, v]) => [toCamel(k), walk(v)]))
      : n;

const input = JSON.parse(readFileSync(0, "utf8"));
console.log(JSON.stringify(walk(input), null, 2));
```

```console
$ node recase.mjs < payload.json
{
  "userId": 8821,
  "createdAt": "2026-03-04T11:20:00Z",
  "profileImageUrl": "https://cdn.example.com/a.png",
  "is2FAEnabled": true
}
```

One caveat worth knowing: `toLowerCase()` is locale-independent in JavaScript, so it's safe here. Reach for `toLocaleUpperCase()` instead and you invite trouble, because Turkish dotless `i` and German `ß` resolve differently per locale. MDN has the details on [String.prototype.toUpperCase](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/toUpperCase).

Wiring this into `package.json` as a `prebuild` step meant the renamed payload was committed to the repo, not computed on every request. Same output, zero runtime cost.

## Method 3: when you just have one blob of text

The scripts above are for whole JSON documents. Most of my day isn't like that. Often I have a single column list from a database migration, or a block of constant names, and I need it flipped once and then never again.

Installing a dependency for that is silly. I open [a free online case converter](https://utilvy.com/en/tools/case-converter/), paste the list, and pick the target style: camelCase, PascalCase, SCREAMING_SNAKE_CASE, kebab-case, whatever the house style needs.

Concretely, last week I had 60-odd Postgres column names from `\d+ orders` that needed to become a TypeScript interface. Pasting that block and hitting convert took maybe fifteen seconds. Writing a throwaway script for it would have taken longer than the migration itself.

What sold me on it: everything runs client-side. Those API payloads had real customer emails in them, and I'd rather not paste that into someone else's server. It's the same reason I don't use online JSON formatters for production dumps.

For quick one-off conversions like renaming a batch of CSS custom properties, [this browser-based case tool](https://utilvy.com/en/tools/case-converter/) beats typing `python3 recase.py` and then remembering where I saved it.

## Three gotchas that cost me real time

**1. `str.title()` mangles apostrophes.** I used it in my first draft. Python's `title()` uppercases the letter after *any* non-letter, not just underscores:

```python
>>> "don't stop".title()
"Don'T Stop"
>>> "user_id".title()
"User_Id"
```

Neither is what you want. `capitalize()` only touches the first character, which is why the final script uses it. The same behavior is documented in the [Python standard types reference](https://docs.python.org/3/library/stdtypes.html#str.title).

**2. Uppercasing can change string length.** German ß expands:

```python
>>> len("straße")
6
>>> "straße".upper()
'STRASSE'
>>> len("straße".upper())
7
```

Same in JS: `"straße".toUpperCase()` returns `"STRASSE"`. If you uppercase a column name and then compare lengths, or use it as a fixed-width identifier, you get silent drift. I hit this on a billing export where a customer's street field was part of a hash key.

**3. Acronym round-trips are lossy.** The naive regex everyone writes first:

```javascript
"parseXMLHTTPResponse".replace(/([A-Z])/g, (m) => "_" + m.toLowerCase());
// => 'parse_x_m_l_h_t_t_p_response'
```

Going camel → snake → camel does *not* return you to the original:

```python
>>> re.sub(r"(?<!^)(?=[A-Z])", "_", "parseXMLHTTPResponse").lower()
'parse_x_m_l_h_t_t_p_response'
```

Once `XMLHTTP` is exploded you cannot reconstruct it without a dictionary. That is the entire reason for the `KEEP_UPPER` allowlist. A short list of terms I know appear in this codebase beats any clever heuristic, because a heuristic has to guess and I don't have to.

## Picking an approach

| Situation | Use | Why |
| --- | --- | --- |
| Whole JSON payloads, repeated runs | Python script (Method 1) | Recursive, testable, no browser |
| Already inside a JS/TS repo | Node script (Method 2) | No extra toolchain |
| One-off list of names | [the case converter on utilvy.com](https://utilvy.com/en/tools/case-converter/) | No install, data stays local |
| Anything with acronyms | Any of the above + an allowlist | Round-trips stay reversible |

The rule I settled on: automate the bulk rename, keep an explicit acronym list, and never trust a converter you haven't tested on `is_2fa_enabled`.

More free browser-based utilities at utilvy.com