A partner's billing webhook dropped a 4,087,349-byte file into our bucket at 2 a.m. One line. No spaces, no newlines, and my loader died on it immediately:

```text
json.decoder.JSONDecodeError: Expecting ',' delimiter: line 1 column 48130 (char 48129)
```

Line 1, column 48130. Technically correct, practically worthless. There is only one line and it is four megabytes wide. I spent the next twenty minutes holding the right arrow key in `less` before I found the real problem: a missing comma right after `"event_id":"evt_000154"`.

Here's the version of that process I use now. It takes about ninety seconds, and it also catches the two quieter bugs that were hiding behind the loud one.

## Method 1: Python 3.11.1 — trust `e.pos`, ignore `e.lineno`

`JSONDecodeError` carries a `pos` attribute: the absolute character offset into the string. `lineno` and `colno` are derived from it, and on a minified file they collapse into noise. Slice a window around `pos` instead of hunting for a line number.

```python
# locate.py — Python 3.11.1
import json, sys

path = sys.argv[1]
raw = open(path, encoding="utf-8").read()

try:
    json.loads(raw)
except json.JSONDecodeError as e:
    print(f"{type(e).__module__}.{type(e).__name__}: {e}")
    start = max(0, e.pos - 60)
    end = min(len(raw), e.pos + 60)
    print(f"\nchar {start}..{end} of {len(raw)}:")
    print(raw[start:end])
    print(" " * (e.pos - start) + "^")
    sys.exit(1)

print("valid JSON")
```

Against the broken file:

```text
json.decoder.JSONDecodeError: Expecting ',' delimiter: line 1 column 48130 (char 48129)

char 48069..48189 of 4087349:
37.36 Chrome/141.0.0.0","retry":1}},{"event_id":"evt_000154""user_id":100154,"email":"user154@example.com","plan":"team"
                                                            ^
```

`evt_000154""user_id`. Two quote marks touching, one comma short. The whole bug fits on one screen.

"Valid" is not the same as "correct", though, so I parse with two extra hooks before I trust anything:

```python
# check.py — Python 3.11.1
import json, sys
from collections import Counter

def no_dupes(pairs):
    counts = Counter(k for k, _ in pairs)
    dupes = sorted(k for k, n in counts.items() if n > 1)
    if dupes:
        raise ValueError(f"duplicate key(s): {dupes}")
    return dict(pairs)

def reject_constant(name):
    raise ValueError(f"non-standard JSON constant: {name}")

raw = open(sys.argv[1], encoding="utf-8").read()
try:
    data = json.loads(raw, object_pairs_hook=no_dupes, parse_constant=reject_constant)
except json.JSONDecodeError as e:
    start, end = max(0, e.pos - 60), min(len(raw), e.pos + 60)
    print(f"{e}\n\nchar {start}..{end} of {len(raw)}:")
    print(raw[start:end])
    print(" " * (e.pos - start) + "^")
    sys.exit(1)
except ValueError as e:
    print(f"rejected: {e}")
    sys.exit(1)

print(f"valid JSON, {len(raw)} bytes")
pretty = json.dumps(data, indent=2, ensure_ascii=False)
open(sys.argv[2], "w", encoding="utf-8").write(pretty)
print(f"wrote {len(pretty)} bytes to {sys.argv[2]} ({len(pretty)/len(raw):.1f}x)")
```

On the repaired payload:

```text
valid JSON, 4087350 bytes
wrote 5634358 bytes to pretty.json (1.4x)
```

Formatting inflates a file like this by roughly 40 percent. Fine for a scratch copy, worth remembering before you commit the pretty version by accident.

One caveat: `object_pairs_hook` runs a `Counter` over every object in the document, so on 13,000 nested records it is measurably slower than plain `json.loads`. I keep the plain parse on the CI happy path and switch to the strict one only when a delivery looks suspicious.

## Method 2: jq 1.7 and `cut`, no Python involved

Python isn't always in the container. jq usually is, or is one `apt-get install jq` away, and it fails loudly:

```text
$ jq . payload.json
jq: parse error: Expected separator between values at line 1, column 48138
$ echo $?
5
```

Look at the column: 48138, not 48130. jq counts from the start of the offending token, Python counts from the start of the buffer. Different origins, same neighbourhood, so don't waste time reconciling them. Exit code 5 means bad input, and under `set -e` it kills your script just as thoroughly as `grep` returning 1.

jq has no "show me the bytes" flag, so I hand the offset straight to `cut`:

```text
$ cut -c48070-48190 payload.json
37.36 Chrome/141.0.0.0","retry":1}},{"event_id":"evt_000154""user_id":100154,"email":"user154@example.com","plan":"team",
```

There it is again. Once the comma is back, jq does the part it's genuinely good at:

```text
$ jq -r '.events | length' fixed.json
13000
$ jq -r '.events[154] | "\(.event_id) \(.plan) \(.amount_cents)"' fixed.json
evt_000154 team 84450
```

For the formatted copy, `jq . fixed.json > pretty.json` gave me 5,634,359 bytes. One byte more than Python's `indent=2` output, because jq adds a trailing newline and `json.dumps` doesn't.

## Method 3: paste it into a browser tab

Both of the above need a shell and something installed, which is often more than the situation deserves. When I'm on a laptop that has nothing on it, or when the payload is a customer's data that I'd rather not write to disk at all, I open [a browser-based JSON formatter](https://utilvy.com/en/tools/json-formatter/) and paste the blob in.

It formats, validates, and minifies, all client-side, so a 4 MB payload never leaves the machine. That last part matters more than it sounds: these files are full of email addresses and IP addresses, and "paste it into a random website" is not a sentence I want in an incident retrospective. The error reporting points at the offending character too, which is most of what I needed from the Python script in the first place.

It also covers the direction I always forget: collapsing a pretty file back to one line before it goes into a request body or a CI variable. Minify is one click, which saves me from hand-editing newlines out of a multi-megabyte file.

## Three things that bit me

**1. `line 1 column N` on a minified file.** The line and column are real but unreadable. Use `e.pos`, or `cut -c`, and print ±60 characters of context. This single habit is what turned a twenty-minute hunt into ten seconds.

**2. Duplicate keys vanish without a complaint.** Nobody raises an error; the last value just wins:

```text
>>> json.loads('{"order_id":"A-1001","total":4200,"order_id":"A-1002"}')
{'order_id': 'A-1002', 'total': 4200}
```

jq does exactly the same thing:

```text
$ printf '%s' '{"order_id":"A-1001","total":4200,"order_id":"A-1002"}' | jq -c .
{"order_id":"A-1002","total":4200}
```

If that had been `amount_cents`, I'd have shipped a wrong number and never known. `object_pairs_hook` catches it; nothing else in my pipeline did.

**3. Python writes JSON that JavaScript refuses to read.** `json.loads` accepts `NaN` and `Infinity` by default, because Python has them and the grammar is relaxed about it. So the value survives the round trip and then breaks the consumer:

```text
>>> json.loads('{"discount":NaN,"total":4200}')
{'discount': nan, 'total': 4200}
>>> json.dumps(json.loads('{"discount":NaN,"total":4200}'))
'{"discount": NaN, "total": 4200}'
```

Hand that to Node 22.13.1 and you get:

```text
SyntaxError: Unexpected token 'N', "{"discount": NaN}" is not valid JSON
```

[RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259#section-6) has no `NaN` literal, and neither does [`JSON.parse`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse). `parse_constant` rejects it at the door. Related, same section of the spec: integers above 2^53 lose precision the moment JavaScript touches them. The same field parses as `12345678901234567890` in Python and `12345678901234567000` in Node. If you have order IDs that big, they should be strings.

## What I reach for

| Situation | Tool | Why |
| --- | --- | --- |
| Need the exact byte, plus dupe/NaN checks | Python 3.11.1 + `object_pairs_hook` | Only version that fails on silent data loss |
| Container with jq but no Python | jq 1.7 + `cut -c` | Fast validation, good querying, no dupe detection |
| Untrusted laptop or sensitive payload | [Utilvy's JSON formatter](https://utilvy.com/en/tools/json-formatter/) | Nothing to install, nothing uploaded |

The missing comma took twenty minutes the first time and ninety seconds the second. The difference wasn't a better parser. It was knowing which number in the error message to actually look at.

Worth saying out loud, though: I reach for [this online JSON formatter](https://utilvy.com/en/tools/json-formatter/) more often than either script, because pasting beats finding the file path. It's also the link I send teammates who drop a wall of broken JSON into Slack.

More free browser-based utilities at utilvy.com