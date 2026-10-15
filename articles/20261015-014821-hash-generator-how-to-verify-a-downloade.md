I used to skip the checksum.

A vendor page would print a SHA-256 next to the download link, and I'd click, install, and move on. That habit ended the day a release tarball unpacked without a single error and then segfaulted on startup. The archive on my disk was 41 MB. The real one is 118 MB. A proxy in front of the CDN had cut the transfer, and nothing between the CDN and my filesystem noticed.

Four seconds with `sha256sum` would have caught it. Now I verify first, using one of three methods depending on what I already have open.

## Method 1: Python 3.11 and hashlib

For files I want a streaming read. `hashlib.sha256(open(path).read())` is fine for a config file and rude for a 4 GB disk image.

This is the script I keep at `~/bin/sha256file.py`, running on Python 3.11.1:

```python
#!/usr/bin/env python3
"""Print the SHA-256 of one or more files, reading in 1 MiB chunks."""
import hashlib
import sys

CHUNK = 1024 * 1024

def sha256_file(path: str) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:          # binary mode, not "r"
        for chunk in iter(lambda: f.read(CHUNK), b""):
            h.update(chunk)
    return h.hexdigest()

if __name__ == "__main__":
    for p in sys.argv[1:]:
        print(f"{sha256_file(p)}  {p}")
```

The two-column output is deliberate. It matches what GNU `sha256sum` prints, so I can paste the result into a `CHECKSUMS` file and diff it against the vendor's list.

```
$ python3 sha256file.py release-2.4.1.tar.gz
3917ea2ec94345943fd55c88fc021982508fce43c1963d0619d454fcb9d49291  release-2.4.1.tar.gz
```

For CI I don't want output, I want a verdict:

```python
EXPECTED = "3917ea2ec94345943fd55c88fc021982508fce43c1963d0619d454fcb9d49291"
got = sha256_file("release-2.4.1.tar.gz")
if got == EXPECTED:
    print("OK")
else:
    raise SystemExit(f"MISMATCH\n  expected {EXPECTED}\n  got      {got}")
```

The rest of the algorithm list is in [Python's hashlib documentation](https://docs.python.org/3/library/hashlib.html). MD5, SHA-1, SHA-256 and SHA-512 are all available, and `hashlib.new("sha512")` picks one by name at runtime.

One thing that bit me: the available set depends on the OpenSSL build underneath. On OpenSSL 3.0.13, `hashlib.new("md4")` raises `ValueError: unsupported hash type md4`, while the same call on an older machine worked fine. If your script has to run on both, read `hashlib.algorithms_available` rather than assuming.

## Method 2: the shell, or Node when JS is already running

On a Linux box the quickest path is the one already installed. With GNU coreutils:

```
$ sha256sum -c CHECKSUMS.txt
release-2.4.1.tar.gz: OK
```

And when the file is wrong:

```
$ sha256sum -c CHECKSUMS.txt
release-2.4.1.tar.gz: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
$ echo $?
1
```

That exit code is the entire reason I put this in a pipeline instead of eyeballing the hex. Non-zero fails the step.

When the job is already Node (I'm on v22.13.1), `node:crypto` does it without shelling out:

```js
import { createHash } from "node:crypto";
import { createReadStream } from "node:fs";

const hash = createHash("sha256");
for await (const chunk of createReadStream("release-2.4.1.tar.gz")) {
  hash.update(chunk);
}
console.log(hash.digest("hex"));
// 3917ea2ec94345943fd55c88fc021982508fce43c1963d0619d454fcb9d49291
```

Node 22 lets you `for await` straight over a read stream, so there is no `on("data")` plumbing to get wrong. For a short string it collapses to one line:

```js
createHash("sha256").update("hello").digest("hex");
// '2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824'
```

## Method 3: hashing a string in the browser

Large files are the hard case. Short strings are the everyday one: a webhook body I need to fingerprint, a JSON config whose value I suspect changed, a token I'd rather not paste into a shell where it lands in history.

For those I paste the text into a [browser-based hash generator](https://utilvy.com/en/tools/hash-generator/) and get MD5, SHA-1, SHA-256 and SHA-512 back at once. No install, no REPL warm-up, and the text stays in the tab instead of going to someone else's server. That last part is why I'll use it for a production token and not for a random paste bin.

My most frequent use is webhook debugging. A provider signs the request body with HMAC-SHA256, and before I touch the signature I hash the raw body to confirm the bytes are what I think I received. About half the time the mismatch turns out to be a trailing newline my own logging added.

If you need the same thing inside your own app, MDN's [SubtleCrypto.digest()](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest) is the browser-native route, though it's async and hands you an ArrayBuffer to convert yourself.

## Three ways this bites you

**Your shell adds a byte you didn't ask for.** `echo` appends a newline, `printf` doesn't. The vendor hashed the file bytes; you hashed the bytes plus `\n`:

```
$ echo 'hello' | sha256sum
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  -
$ printf 'hello' | sha256sum
2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824  -
```

**Line endings are bytes too.** A Windows checkout with `core.autocrlf=true` rewrites every `\n` as `\r\n`, and the hash follows:

```
$ printf 'a\r\nb\r\n' | sha256sum
58055bdcc73787eb88c78d36f0b4939e9c5dc1c3ad17e25cc85a6833cf1a0cab  -
$ printf 'a\nb\n' | sha256sum
911169ddaaf146aff539f58c26c489af3b892dff0fe283c1c264c65ae5aa59a2  -
```

Python has the same trap: open with `"r"` instead of `"rb"` on Windows and the newline translation happens before your first `update()` call.

**A matching MD5 tells you nothing about an attacker.** MD5 and SHA-1 are still good at catching a truncated download and worthless if someone wanted to hand you a different file that hashes identically. SHA-1 collisions stopped being academic years ago. Anything touching security gets SHA-256.

And a hash is not encryption. `sha256(password)` is not password storage, use bcrypt or argon2.

## Which one to reach for

| Situation | Use | Why |
|---|---|---|
| Large file on Linux | `sha256sum -c` | Fastest, exit code fits CI |
| Large file in a Python job | `hashlib` with chunked reads | No subprocess, bounded memory |
| Large file in a Node job | `node:crypto` + read stream | Stays in the language you're in |
| Short string on someone else's laptop | this [free hash tool](https://utilvy.com/en/tools/hash-generator/) | Nothing to install, nothing uploaded |
| Passwords or tokens at rest | bcrypt / argon2 | Hashing is one-way, not secret |

I still check every download before I run it. Four seconds, and the one time it fails it hands back an afternoon. The 41 MB tarball cost me a morning of bisecting a library version that was never the problem. On machines I don't own, the [hash page on utilvy](https://utilvy.com/en/tools/hash-generator/) is what I open.

More free browser-based utilities at utilvy.com