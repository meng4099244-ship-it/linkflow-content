Our billing webhook consumer started rejecting every retry last Friday. Nothing in our code had changed. The provider had quietly put a new egress proxy in front of their delivery service, and that proxy re-encoded the request body. Same JSON event, now wrapped in `Content-Transfer-Encoding: base64` with a hard fold every 76 characters.

My handler did the obvious thing:

```python
event = json.loads(base64.b64decode(request.body))
```

It raised `UnicodeDecodeError: 'utf-8' codec can't decode byte 0x8b in position 1: invalid start byte`. Byte `0x8b` is the second half of the gzip magic number, which tells you the payload was compressed before it was encoded. Two layers, and I had assumed one.

By the time I found it, the queue held 1,412 unprocessed events. Their retries kept arriving with the same body and my consumer kept returning 400, so the provider's dashboard showed the deliveries as failed with no useful reason attached.

Here is what I ended up shipping, and the three traps that cost me most of that afternoon.

## Method 1: a decoder that checks what it got

Python 3.11.1, standard library only:

```python
import base64, gzip, json

def decode_envelope(payload: str) -> dict:
    cleaned = "".join(payload.split())          # drop CRLF folding and stray spaces
    raw = base64.b64decode(cleaned, validate=True)
    if raw[:2] == b"\x1f\x8b":                  # gzip magic
        raw = gzip.decompress(raw)
    return json.loads(raw.decode("utf-8"))

blob = """H4sIAAAAAAACA6tWSi1LzStRslLKzCvLz0xO1StIzExR0lECElZAuZJ4w0CjcuNUoEhi
bn4pSKWJkYFBLQAxNK3vOAAAAA=="""

print(decode_envelope(blob))
# {'event': 'invoice.paid', 'id': 'evt_1Q2w3e', 'amount': 4200}
```

The `"".join(payload.split())` line is doing real work. `str.split()` with no argument splits on any whitespace, including `\r\n`, so the fold disappears before the decoder ever sees it.

Keep `validate=True`. It makes Python reject characters outside the alphabet instead of skipping them. Without it you get a decoder that quietly returns shorter bytes, and the failure surfaces three layers away as a JSON parse error.

Two things I log now, because they would have saved me the first hour: the length before and after cleaning, and the first four bytes in hex. When those disagree with what you expect, you know which layer is lying. `payload` arrived at 97 characters and cleaned to 96, which is the exact signature of a single CRLF fold. `raw[:4]` was `1f8b0800`, not `7b2265` (`{"e`), so the payload was never JSON text to begin with.

## Method 2: Node, where the decoder never complains

Node 22.13.1 is more forgiving, which is the problem. `Buffer.from(x, "base64")` ignores anything that is not a base64 character and does not throw. Append a stray `$` to a valid string and it still returns `{"ok":true}`, no warning. So validate before you decode:

```js
const { gunzipSync } = require("node:zlib");
const B64 = /^[A-Za-z0-9+/]*={0,2}$/;

function decodeEnvelope(payload) {
  const cleaned = payload.replace(/\s+/g, "");
  if (!B64.test(cleaned)) throw new Error("not standard base64: " + cleaned.slice(0, 16));
  if (cleaned.length % 4 !== 0) throw new Error("bad length: " + cleaned.length);
  let raw = Buffer.from(cleaned, "base64");
  if (raw[0] === 0x1f && raw[1] === 0x8b) raw = gunzipSync(raw);
  return JSON.parse(raw.toString("utf8"));
}
```

Run it against a URL-safe string and you get `Error: not standard base64: a-b_c`, which is exactly the message you want at the boundary instead of a stack trace in your event handler.

One caveat about the length check. `cleaned.length % 4 !== 0` catches truncated payloads, but not the common case where the sender dropped the `=` padding entirely. Unpadded strings whose length is already a multiple of four pass straight through, decode to the right bytes, and look fine. That is why the regex and the length check are separate lines: one guards the alphabet, the other guards the block alignment.

## Method 3: look inside the blob first

Before writing a single line of the above, I should have just looked at the payload. Paste the blob into [a browser-based Base64 decoder](https://utilvy.com/en/tools/base64/) and you see whether the output is JSON, gzip noise, or a PDF within two seconds. No install, nothing leaves the tab.

That is how I found the `0x8b`. Decoding to bytes and dumping the first sixteen of them in hex would also have worked, but the visual is faster when you are staring at an opaque string at 6pm. The page also handles the reverse direction, which matters when you are building a test fixture and need the exact string the provider would have sent.

## Three traps, with the output they produce

**1. Folded base64 plus a strict decoder.** Providers wrap at 76 columns because MIME says so, not because base64 needs it. Python tolerates the newlines by default, so a local test passes. Set `validate=True` and the same string fails:

```
binascii.Error: Only base64 data is allowed
```

GNU `base64 -d` and Node both accept folded input silently, so the bug lives in one service only. Normalize whitespace at the edge, once, and log the cleaned length.

**2. Base64 output is not text.** Decode a gzipped body and try to treat it as a string:

```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x8b in position 1: invalid start byte
```

Re-encoding the mangled string never gives you the original back. On my test payload, 70 bytes of gzip became 96 characters of base64. Decoded with `errors="replace"` and encoded back, those 70 bytes turned into 116, because 23 replacement characters take three bytes each. Check the magic bytes before decoding as UTF-8, and remember base64 has no checksum. A flipped character decodes cleanly into wrong bytes, so compare a SHA-256 if the payload matters.

**3. URL-safe alphabet and missing padding.** JWT segments and query parameters use `-` and `_` instead of `+` and `/`, and they drop the `=` padding. Python:

```
binascii.Error: Incorrect padding
```

Node does not error. `Buffer.from("a-b_c", "base64")` returns `6be6ff` because Node accepts `-` and `_` as well, so a URL-safe string sails through one service and blows up in the next. If a value comes from a URL, decode with `base64.urlsafe_b64decode`, or pass `altchars=b"-_"` and re-add padding as `s + "=" * (-len(s) % 4)`.

When I just need to confirm what a token segment holds, I paste it into [this online Base64 tool](https://utilvy.com/en/tools/base64/) and compare against the decoded output from my script. Two independent decoders agreeing is a cheap sanity check.

| Approach | Handles folding | Handles gzip | Fails loudly | Best for |
|---|---|---|---|---|
| Python `base64` + `validate=True` | after whitespace strip | manual magic check | yes | batch jobs, backfills |
| Node `Buffer` + regex guard | yes | `gunzipSync` | only with your own checks | services already on Node |
| Browser tool | paste as-is | shows bytes, not decompressed | visual | one-off inspection |

The fix that went to production was about fifteen lines: strip whitespace, validate the alphabet, check `\x1f\x8b`, decompress if present, then parse. Two days later I still keep [the same decoder running client-side](https://utilvy.com/en/tools/base64/) in a pinned tab for the last-mile check. The proxy is still folding at 76 characters and it no longer matters.

Background reading: [RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648) for the alphabet and padding rules, and [MDN's Base64 glossary entry](https://developer.mozilla.org/en-US/docs/Glossary/Base64) for the browser-side `btoa`/`atob` behavior.

More free browser-based utilities at [utilvy.com](https://utilvy.com).