# UUID Generator: Why My Lookup API Rejected Every ID Pasted From Support Emails

Last month our support team filed the same bug three times in one week. A customer copies their order ID from an email, pastes it into the admin lookup, and the API answers with a 400. The ID looks perfectly fine to everyone involved.

I pulled the failing values from the request logs. One was `550E8400-E29B-41D4-A716-446655440000`, wrapped in braces because that is how Outlook renders identifiers copied from certain calendar-style footers. Another came from a PDF invoice with `urn:uuid:` stuck in front. A third carried an invisible trailing newline from the PDF text layer.

All three were the same UUID. None of them were the same *string*. That distinction cost us a week of finger-pointing, so here is the fix I landed, plus the traps I hit on the way.

## Method 1: Parse and Canonicalize in Python (3.11.1)

My first instinct was a regex. That was the wrong first instinct, because [the standard library already handles the messy formats](https://docs.python.org/3.11/library/uuid.html) — I just wasn't storing the result. Python 3.11.1's `uuid.UUID()` accepts more than the canonical form, and `str()` on the parsed object always emits the lowercase, hyphenated form defined in RFC 9562 (the successor to RFC 4122):

```python
# Python 3.11.1
import uuid

def normalize(raw: str) -> str:
    try:
        u = uuid.UUID(raw.strip())
    except ValueError as e:
        raise ValueError(f"not a valid UUID: {e}") from e
    return str(u)

print(normalize("{550E8400-E29B-41D4-A716-446655440000}"))
print(normalize("urn:uuid:550e8400-e29b-41d4-a716-446655440000"))
print(normalize("550e8400e29b41d4a716446655440000"))
print(normalize("550e8400-e29b-41d4-a716-44665544000"))  # one hex digit short
```

Output, straight from my terminal:

```text
550e8400-e29b-41d4-a716-446655440000
550e8400-e29b-41d4-a716-446655440000
550e8400-e29b-41d4-a716-446655440000
ValueError: badly formed hexadecimal UUID string
```

Three visually different inputs, one canonical output. The truncated one dies with `ValueError: badly formed hexadecimal UUID string`, which is the exact message that had been reaching our customers as a raw 400 body. Wrapping it and returning a human sentence was half the fix. The other half was storing `normalize(raw)` in the database instead of the raw pasted value, so lookups compare canonical strings from that point on.

## Method 2: Strict Validation in Node (22.13.1)

Our lookup service is Node, and it disagreed with Python in an instructive way. With the `uuid` package (14.0.3), `validate()` only accepts the canonical hyphenated form — case-insensitive, but nothing else:

```javascript
// Node 22.13.1, uuid 14.0.3
const { validate, version } = require("uuid");

console.log(validate("550e8400-e29b-41d4-a716-446655440000"));  // true
console.log(validate("550E8400-E29B-41D4-A716-446655440000"));  // true
console.log(validate("{550e8400-e29b-41d4-a716-446655440000}")); // false
console.log(validate("urn:uuid:550e8400-e29b-41d4-a716-446655440000")); // false
console.log(validate("550e8400e29b41d4a716446655440000"));      // false
```

Same UUID that Python cheerfully parsed: `false` in Node, three times. So the two services disagreed about what counts as "valid", and only one of them was ever fed pasted IDs. The Node-side fix was to strip the decorations first, then validate, then compare lowercase:

```javascript
function normalize(raw) {
  let s = raw.trim().replace(/^\{|\}$/g, "").replace(/^urn:uuid:/i, "");
  if (!validate(s)) throw new Error(`not a valid UUID: ${raw}`);
  return s.toLowerCase();
}
```

If you need to confirm the ID is actually a v4 and not some other variant, `version()` tells you — it returned `4` for our order IDs.

When I just need a fresh, clean test ID instead of untangling a mangled one, I grab one from [a free online UUID generator](https://utilvy.com/en/tools/uuid-generator/). It runs entirely in the browser, so the test fixtures never leave my machine, and I can copy a batch of ten without writing a script.

## Method 3: Quick Checks in the Browser

The newsletter crew on our team doesn't code, and they still needed to answer "is this ID real or did the PDF mangle it?". Sending them to a REPL was a non-starter. This [browser-based UUID tool](https://utilvy.com/en/tools/uuid-generator/) became their shortcut: paste, eyeball the length and hyphen pattern against a freshly generated reference, and regenerate a clean one when the original turned out to be damaged. Zero installs, and nothing gets uploaded anywhere — the comparison happens client-side.

## Three Traps I Hit

**1. Python accepts more than it should, and you will store the raw input.** `uuid.UUID()` happily parsed the braces, the `urn:uuid:` prefix, and the hyphen-less 32-hex form in my tests. If you only call it as a validity check and keep the original string, your database now holds three spellings of one ID, and `WHERE id = '...'` misses two of them. I proved it locally: `db == pasted` was `False` while `uuid.UUID(db) == uuid.UUID(pasted)` was `True`. Always store `str(u)`, not the input.

**2. The nil UUID breaks naive version checks.** A customer once sent us `00000000-0000-0000-0000-000000000000` as a "test" value. In Python, `uuid.UUID(...).version` returns `None` for it, and our check `if u.version >= 4` blew up with `TypeError: '>=' not supported between instances of 'NoneType' and 'int'`. Node's `version()` returned `0` for the same input instead of throwing, so the two runtimes even failed differently. Guard for `None` before any numeric comparison.

**3. Trailing whitespace fails in both runtimes.** A newline picked up from a PDF copy is enough: Python raised `ValueError: badly formed hexadecimal UUID string` and Node's `validate()` returned `false` for an ID with a single `\n` at the end. Neither runtime trims for you. My `normalize()` functions strip whitespace before anything else, and the 400s from copy-paste garbage dropped to zero the week we shipped that.

## Wrap-Up

| Problem | Fix | Where it bit us |
| --- | --- | --- |
| Decorated IDs (braces, `urn:uuid:`, no hyphens) | Parse then store `str(u)` | Python accepted all three; Node accepted none |
| Nil UUID in version checks | Handle `version is None` | Python `TypeError`, Node returned `0` |
| Trailing newline from PDF copy | `.strip()` / `.trim()` first | Both runtimes rejected the ID |

One rule summarizes the whole episode: never compare, store, or log a UUID as the user gave it to you. Canonicalize at the boundary. And when you need clean IDs for fixtures instead of forensics, the [UUID generator page](https://utilvy.com/en/tools/uuid-generator/) is faster than opening an editor.

More free browser-based utilities at [utilvy.com](https://utilvy.com/en/).