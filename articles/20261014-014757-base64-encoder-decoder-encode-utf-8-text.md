# Base64 Encoder / Decoder: Encode UTF-8 Text Without Corrupting Unicode

Last month I had to place a small JSON object inside an environment variable used by a CI job. The object contained a customer name in Chinese and a check mark. My first attempt looked fine in a local test, but the decoded value on the runner contained mojibake. The mistake was not Base64 itself; I had converted characters to bytes differently on the two sides.

Base64 is a byte-to-text encoding. It is handy when a transport accepts plain ASCII but the payload contains Unicode text or binary data. It is not compression, and it is definitely not encryption. Here are the three approaches I now use, from a repeatable script to a quick browser check.

Before encoding, I write down both sides of the boundary: “JSON text encoded as UTF-8” on the producer and “Base64 decoded to UTF-8 text” on the consumer. That sounds fussy, but it catches a real class of bugs. A JavaScript string, a UTF-8 byte sequence, and the Base64 text made from those bytes are three different values. If a database driver, shell, or HTTP form changes one stage, decoding may succeed while the final text is still wrong.

Base64 also makes data larger. Every three input bytes become four ASCII characters, plus optional padding. A 10-byte UTF-8 value therefore produces 16 Base64 characters in the example below. That overhead is acceptable for a short configuration value; it is a poor reason to embed a 40 MB video in JSON.

## Method 1: encode UTF-8 safely with Node.js

I tested this example with Node.js 22.13.1. Node's `Buffer` API makes the byte encoding explicit, which is the detail that prevents most Unicode bugs.

Save this as `base64-demo.mjs`:

```javascript
// Node.js 22.13.1
const input = "中文 ✓";

const utf8Bytes = Buffer.from(input, "utf8");
const encoded = utf8Bytes.toString("base64");
const decoded = Buffer.from(encoded, "base64").toString("utf8");

console.log({
  input,
  utf8Bytes: utf8Bytes.length,
  encoded,
  decoded,
  roundTripOK: decoded === input,
});
```

Run it with:

```bash
node base64-demo.mjs
```

The output is:

```text
{
  input: '中文 ✓',
  utf8Bytes: 10,
  encoded: '5Lit5paHIOKckw==',
  decoded: '中文 ✓',
  roundTripOK: true
}
```

The string has four visible characters plus a space, yet it occupies 10 UTF-8 bytes. Base64 works on those 10 bytes, not on JavaScript's character count. I keep the explicit `"utf8"` arguments because they document the contract even though UTF-8 is Node's default here.

For files, do not read arbitrary content as UTF-8 first. Preserve the bytes:

```javascript
// Node.js 22.13.1
import { readFile, writeFile } from "node:fs/promises";

const original = await readFile("input.png");
const encoded = original.toString("base64");
await writeFile("input.png.b64", encoded, "ascii");

const restored = Buffer.from(encoded, "base64");
await writeFile("restored.png", restored);
```

The official [Node.js Buffer documentation](https://nodejs.org/api/buffer.html) lists the supported encodings and explains why a `Buffer` is the right abstraction for binary payloads.

## Method 2: use Python for strict validation

When I am cleaning data rather than writing an application, Python is often faster to reach for. This version was tested with Python 3.11.1 and uses `validate=True`, so unexpected characters fail instead of being silently ignored.

Save it as `b64.py`:

```python
# Python 3.11.1
import argparse
import base64

parser = argparse.ArgumentParser()
parser.add_argument("mode", choices=["encode", "decode"])
parser.add_argument("value")
args = parser.parse_args()

if args.mode == "encode":
    raw = args.value.encode("utf-8")
    print(base64.b64encode(raw).decode("ascii"))
else:
    raw = base64.b64decode(args.value, validate=True)
    print(raw.decode("utf-8"))
```

A round trip is easy to inspect:

```bash
python3 b64.py encode "中文 ✓"
# 5Lit5paHIOKckw==

python3 b64.py decode "5Lit5paHIOKckw=="
# 中文 ✓
```

For a binary file, replace the text conversion with `Path(...).read_bytes()` and write the decoded bytes with `write_bytes()`. That avoids an accidental text decode before Base64 even starts.

## Method 3: use an online encoder for one-off values

Scripts are best when the operation belongs in a build, test, or migration. They are overkill when I only need to inspect a short API token payload or decode one value from a log.

For that case, I use [a free online Base64 encoder and decoder](https://utilvy.com/en/tools/base64/). I can paste UTF-8 text, switch between encoding and decoding, and copy the result without installing a package. The page runs in the browser, so the text is processed client-side rather than uploaded.

I also use [this browser-based Base64 tool](https://utilvy.com/en/tools/base64/) as a second implementation during debugging. If my script and the page produce the same encoded text, I can usually stop blaming Base64 and look for a transport issue such as trimming, URL parsing, or a mismatched character encoding.

## Common pitfalls I have actually hit

### 1. Calling `btoa()` directly on Unicode

`btoa()` expects a binary string whose code points fit in one byte. In Node.js 22.13.1, this fails:

```javascript
btoa("中文 ✓");
```

```text
InvalidCharacterError: Invalid character
```

In browser code, convert the text to UTF-8 bytes first or use an approach based on `TextEncoder`. MDN's [`btoa()` reference](https://developer.mozilla.org/en-US/docs/Web/API/Window/btoa) includes the byte-oriented pattern. Do not “fix” this by dropping the non-ASCII characters.

### 2. Losing the padding

Some systems remove trailing `=` characters. Python's strict decoder exposes the problem immediately:

```python
base64.b64decode("aGVsbG8", validate=True)
```

```text
binascii.Error: Incorrect padding
```

Restore the required padding only when the producer's contract says padding may be omitted. Blindly appending `=` can hide a truncated value. I first check the expected Base64 variant and the payload length.

### 3. Mixing Base64 and Base64url

Standard Base64 uses `+` and `/`; Base64url uses `-` and `_`, often without padding. JWT sections, for example, use Base64url. Feeding one variant to a strict decoder for the other can produce an error or the wrong bytes.

In Node.js, I make the choice visible:

```javascript
Buffer.from(data).toString("base64");    // standard
Buffer.from(data).toString("base64url"); // URL-safe
```

I never replace characters by hand unless I am implementing a documented protocol.

## Quick comparison

| Method | Best for | Unicode-safe step | Main trade-off |
|---|---|---|---|
| Node.js `Buffer` | Apps and CI scripts | `Buffer.from(text, "utf8")` | Requires Node.js |
| Python `base64` | Data cleanup and strict checks | `text.encode("utf-8")` | Text and binary paths differ |
| Online tool | One-off inspection | Handled in the browser | Not ideal for automation |

My rule is simple: define the bytes before encoding, use the matching Base64 variant, and verify a round trip with a known value. When I need a quick independent check, the [UTF-8-safe Base64 converter](https://utilvy.com/en/tools/base64/) saves me from writing another throwaway command.

More free browser-based utilities at [utilvy.com](https://utilvy.com).