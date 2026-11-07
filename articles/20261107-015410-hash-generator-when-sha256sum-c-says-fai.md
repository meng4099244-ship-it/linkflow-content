A vendor sent us a signed build of their SDK last month. Version 2.4.1, one tarball, one `checksums.txt`. Straightforward. I dropped both into CI, ran `sha256sum -c checksums.txt`, and got:

```
app-2.4.1.tar.gz: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
```

My first thought was a corrupted download. My second thought was worse: an attacker. It turned out to be neither. The tarball was fine, the checksum was fine, and the actual problems were the kind you only find by re-hashing the same file four different ways. Here's everything I tried, with the real outputs, so you can skip three of the four detours.

## Method 1: Python, done properly

Python 3.11.1 on the CI box. The job: hash the tarball, read the expected digest out of the manifest, compare. My first attempt was six lines and it exploded:

```python
# WRONG - do not run this on a binary
with open("app-2.4.1.tar.gz") as f:
    data = f.read()

import hashlib
print(hashlib.sha256(data).hexdigest())
```

```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x1f in position 0: invalid start byte
```

That `0x1f` is the gzip magic byte. `open()` defaults to text mode, and text mode tries to decode every byte as UTF-8 before it ever reaches `hashlib`. Two fixes: open in binary mode, and read in chunks so a 900 MB archive doesn't sit in RAM as one `bytes` object.

```python
# Python 3.11.1, standard library only
import hashlib
from pathlib import Path

def sha256_file(path: str) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:          # "rb", not "r"
        for chunk in iter(lambda: f.read(1024 * 1024), b""):
            h.update(chunk)
    return h.hexdigest()

digest = sha256_file("app-2.4.1.tar.gz")
manifest = Path("checksums.txt").read_text()

expected = next(
    line.split()[0] for line in manifest.splitlines()
    if "app-2.4.1.tar.gz" in line
)
print(digest)                             # 4f1b2c...e7a902 (64 hex chars)
print("match" if digest == expected.lower() else "MISMATCH")
```

This printed `match` while `sha256sum -c` kept saying `FAILED`. That contradiction is what sent me down the rabbit hole.

## Method 2: the shell, with the manifest itself as the suspect

GNU coreutils 8.32 on Ubuntu 22.04. When a per-file hash matches but `sha256sum -c` fails, stop hashing the tarball and start inspecting the manifest:

```bash
$ cat -A checksums.txt | head -2
4f1b2c9d...e7a902^M  app-2.4.1.tar.gz$
```

That `^M` is a carriage return. The vendor generated `checksums.txt` on Windows, so every line ends in CRLF. GNU `sha256sum` treats `app-2.4.1.tar.gz\r` as the filename, and my verbose run confirmed it:

```
sha256sum: app-2.4.1.tar.gz: No such file or directory
sha256sum: WARNING: 1 listed file could not be read
```

The quiet `FAILED` from earlier was actually a read failure, not a hash mismatch. The one-line fix on Linux:

```bash
tr -d '\r' < checksums.txt > checksums.fixed.txt
sha256sum -c checksums.fixed.txt
# app-2.4.1.tar.gz: OK
```

On macOS it fails even louder, because BSD `shasum` rejects the whole file:

```
shasum: standard input: no properly formatted checksum lines found
```

One more shell detail worth knowing: `sha256sum -c` resolves paths relative to the current directory, not relative to the manifest. Running it from the repo root when the manifest lists `dist/app-2.4.1.tar.gz` gives you the same "No such file or directory" again. `cd` into the directory that holds the artifact, or run `sha256sum -c ../checksums.fixed.txt` from inside `dist/`.

## Method 3: no scripts, just paste

After the third false alarm I stopped wanting to run anything at all. For a single file on a machine where I don't trust my own aliases, I now generate the digest with a browser-based hash generator like [this free SHA-256 hash tool](https://utilvy.com/en/tools/hash-generator/): drop the file in, read the digest, compare against the vendor's value by eye. It runs client-side, so the archive never leaves the machine, which matters more than it sounds — I'm not uploading an unreleased vendor build to some random hashing web service.

It computes MD5, SHA-1, SHA-256 and SHA-512 in one pass, which is handy when a vendor ships a manifest with mixed algorithms. The same site's [hash generator](https://utilvy.com/en/tools/hash-generator/) also takes pasted text, useful for hashing a small token or config snippet where `echo` vs `printf` quirks (a trailing newline changes everything) would otherwise bite. When I need to explain the process to a non-CLI teammate, this [browser-based checksum tool](https://utilvy.com/en/tools/hash-generator/) is the link I send.

For the record, the JavaScript way (if you must do it in Node instead of the browser) is `crypto.subtle.digest('SHA-256', arrayBuffer)` per the [MDN SubtleCrypto docs](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest), and the [Python `hashlib` reference](https://docs.python.org/3/library/hashlib.html) documents the chunked-update pattern I used above.

## The three pitfalls that actually cost me time

**1. Text-mode file reads.** The `UnicodeDecodeError` above looks like a corrupt file. It isn't. Any `open(path)` without `"rb"` will die on the first non-UTF-8 byte, and gzip's `0x1f 0x8b` header guarantees it. Binary hashes need binary reads, always.

**2. CRLF in the checksum manifest.** The failure message is misleading: `FAILED` on GNU `sha256sum` can mean "hash mismatch" *or* "file not readable". The `\r` glued to the filename causes the latter. `cat -A` the manifest before you distrust the tarball. And note `WARNING: 1 listed file could not be read` appears only with `-c`; the bare per-file run just exits 1 with no filename hint.

**3. Case-sensitive hex comparison.** Some vendors emit uppercase digests. `digest == expected` then reports MISMATCH for identical hashes. Compare with `.lower()` on both sides, and sanity-check the length is exactly 64 characters. I once chased a "mismatch" that was a 63-character paste with the last character lost to a terminal selection.

## Recap

| Check | Command / method | Gotcha |
|---|---|---|
| File digest | Python `hashlib` chunked, `"rb"` mode | text mode → `UnicodeDecodeError` |
| Manifest verify | `sha256sum -c` | CRLF manifest → false `FAILED` |
| macOS verify | `shasum -a 256 -c` | rejects CRLF outright |
| One-off check | browser hash tool | none, but eyeball the length |

The vendor has since regenerated their manifest on Linux. My CI has the `tr -d '\r'` guard anyway, because somebody else's Windows-generated checksum file is a matter of time.

More free browser-based utilities at [utilvy.com](https://utilvy.com/en/).