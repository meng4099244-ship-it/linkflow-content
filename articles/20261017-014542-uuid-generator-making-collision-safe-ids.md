# UUID Generator: Making Collision-Safe IDs in Python, Node.js, and Your Browser

## The problem I actually had

Three worker boxes were inserting rows into the same Postgres table. Auto-increment IDs collided the moment I replayed a batch, and the integer keys leaked row counts to anyone who noticed `?id=1042` in a URL. I needed identifiers I could mint anywhere, offline, without asking a coordinator for permission.

That is the job UUIDs were built for. Here are the three ways I generate them.

## Method 1: Python's stdlib (3.11.1)

Nothing to install. `uuid.uuid4()` pulls 122 random bits from `os.urandom`, which is the OS CSPRNG, not the `random` module.

```python
import uuid

def make_batch(n: int) -> list[str]:
    return [str(uuid.uuid4()) for _ in range(n)]

def is_v4(value: str) -> bool:
    try:
        u = uuid.UUID(value)
    except ValueError:
        return False
    return u.version == 4 and u.variant == uuid.RFC_4122

batch = make_batch(3)
print(batch)
print("all v4?", all(is_v4(x) for x in batch))
print("unique?", len(set(make_batch(1000))) == 1000)
```

Real output from my terminal:

```text
['a70bd4da-f889-41cc-b9a7-d7b44105d020', '93a3e77d-f44b-4705-a113-2d2e8271f411', '89ee394b-50f3-491e-b514-10bad9d6e6d5']
all v4? True
unique? True
```

Three details worth keeping.

`uuid.UUID(...)` also accepts the braced and dash-free forms, so `uuid.UUID('3f2504e04f8911d39a0c0305e82c3301')` parses cleanly into `3f2504e0-4f89-11d3-9a0c-0305e82c3301`. Handy when IDs arrive from a system that strips punctuation.

`u.int.bit_length()` returned 124 rather than 128 on my sample, because leading zeros are real. Never assume fixed width if you slice the hex by hand.

And if you are writing these to a database, keep them in a native `uuid` column where one exists. Postgres stores a UUID in 16 bytes; the 36-character text form costs 37 bytes per row plus index bloat, which adds up fast across a few million rows.

## Method 2: Node.js and the shell

Node has shipped this in core since 14.17. On 22.13.1:

```js
const { randomUUID } = require('node:crypto');

const batch = Array.from({ length: 3 }, () => randomUUID());
console.log(batch);

const isV4 = (s) => /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i.test(s);
console.log('all v4?', batch.every(isV4));
```

```text
[ '180e423f-6c22-491c-b0ea-349eb60d6537',
  'c5ed1348-a69f-441f-9083-9aae964a37a6',
  '0e47aadf-b916-4376-8c76-eb9f9b046d9b' ]
all v4? true
```

The version nibble sits at index 14 and the variant at index 19, which is what that regex pins down. MDN's [Crypto.randomUUID()](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) page is what I send people to when they ask whether the call is safe, and it is the same call in the browser. One browser caveat from that page: the Web Crypto version is only exposed in a secure context, so a page served over plain HTTP on a LAN address will find `crypto.randomUUID` undefined. Node has no such restriction.

When I am in a shell and need one immediately, I read it out of the kernel:

```bash
cat /proc/sys/kernel/random/uuid
# dab6db66-bcea-44eb-88ae-cd61419fe654
```

`uuidgen` does the same job on machines with util-linux installed. The slim container I was working in did not have it, so `/proc` is the more portable habit.

## Method 3: when I just need a pile of them

For a script, method 1 wins. For seeding a test fixture, filling 200 rows of a CSV, or handing an ID to someone who does not have Python open, I use [a free online UUID generator](https://utilvy.com/en/tools/uuid-generator/): type a count, click once, copy the column. It runs client-side, so the identifiers never travel to a server, which matters if you are generating IDs for a customer import file.

Before I paste a few hundred of them into a migration, I spot-check the batch for version and duplicates. [The UUID page I keep bookmarked](https://utilvy.com/en/tools/uuid-generator/) gives me a clean column I can diff against without standing up a scratch script on a machine I do not control.

## Three things that bit me

**1. `random` is not a CSPRNG.** I once assembled a "UUID" from `random.getrandbits(128)` because I wanted reproducible values in a test. It was reproducible in production too:

```python
random.seed(42); uuid.UUID(int=random.getrandbits(128), version=4)
# bdd640fb-0667-4ad1-9c80-317fa3b1799d
random.seed(42); uuid.UUID(int=random.getrandbits(128), version=4)
# bdd640fb-0667-4ad1-9c80-317fa3b1799d
```

Same value twice. `Math.random()` in JavaScript fails the same way, and arguably worse, because V8's PRNG state can be recovered from a handful of outputs. Reach for `secrets` or `crypto`, and hardcode a literal if a test needs a fixed ID.

**2. v1 leaks your MAC address.** `uuid.uuid1()` embeds the host MAC and a timestamp. Mine printed a node field of `0x5e96561a893d`. Anyone holding that ID can recover when it was minted and on which NIC. Fine for internal trace IDs, not fine in a public URL.

**3. v4 sorts badly.** Random primary keys scatter inserts across the B-tree and carry no chronological meaning whatsoever. I generated four IDs three milliseconds apart and asked whether creation order matched lexicographic order: v4 answered `False`. [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html) added UUIDv7 for exactly this, and it is about eight lines of Python:

```python
import os, time, uuid

def uuid7() -> uuid.UUID:
    ms = int(time.time() * 1000)
    b = bytearray(os.urandom(16))
    b[0:6] = ms.to_bytes(6, "big")     # 48-bit timestamp first
    b[6] = (b[6] & 0x0F) | 0x70        # version 7
    b[8] = (b[8] & 0x3F) | 0x80        # RFC variant
    return uuid.UUID(bytes=bytes(b))
```

The same ordering test now returns `True`, and you can read the creation time back out of the first six bytes. One caveat I hit straight away: inside a single millisecond the tail is still random, so ordering only holds at millisecond granularity. If you need strict ordering within one millisecond, put a counter in the random bits instead of trusting `os.urandom`.

## What I reach for

| Situation | What I use | Why |
| --- | --- | --- |
| App code, Python 3.11 | `uuid.uuid4()` | stdlib, CSPRNG-backed, zero dependencies |
| App code, Node 22 | `crypto.randomUUID()` | core module, same guarantee |
| One ID in a shell | `cat /proc/sys/kernel/random/uuid` | no package, no script |
| DB primary key at scale | UUIDv7 | sorts by time, keeps indexes tight |
| Bulk IDs for a fixture or CSV | [this browser-based generator](https://utilvy.com/en/tools/uuid-generator/) | no setup, nothing uploaded |
| Anything security-sensitive | never v1, never `random` | v1 leaks the MAC; `random` is predictable |

Most days the honest answer is `uuid.uuid4()` and moving on with your life. The interesting part starts when the IDs need to sort, or when they end up in a URL where a v1 would quietly hand out your hardware address.

More free browser-based utilities at utilvy.com