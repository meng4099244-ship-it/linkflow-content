Our marketing team ran a webinar in October and promised ten $200 vouchers to ten randomly chosen registrants. The export hit my inbox at 09:40, a CSV with 8,453 rows.

Legal added two conditions before I touched it. Nobody may predict the winners, and we have to reproduce the exact draw six months later if anyone contests it.

Those two conditions fight each other. Reproducible means a seed. A seed published before the draw lets anybody compute all ten names in advance. It took most of an afternoon to satisfy both, and the mistakes along the way are the interesting part.

## Method 1: Python 3.11.1 — draw now, publish the seed afterwards

The trick that settled it: mint a 64-bit seed from the OS entropy pool at the moment you run the draw, use it, then publish it *after* the winners go out. Until that instant nobody can know the outcome, because `secrets.randbits(64)` reads from `/dev/urandom`. Afterwards anyone with the CSV and the recorded seed replays the identical list.

```python
# draw_winners.py — Python 3.11.1
import csv, json, random, secrets
from datetime import datetime, timezone

rows = list(csv.DictReader(open("registrants.csv")))

seen = {}
for row in rows:                          # last registration wins
    seen[row["email"].strip().lower()] = row
pool = sorted(seen.values(), key=lambda r: int(r["registration_id"]))

seed = secrets.randbits(64)
rng = random.Random(seed)
winners = rng.sample(pool, 10)

print(f"rows {len(rows)} -> unique people {len(pool)}")
print(f"seed {seed}")
for n, w in enumerate(winners, 1):
    print(f"{n:2d}. {w['registration_id']:>5}  {w['email']}")

with open("draw-record.json", "w") as f:
    json.dump({"drawn_at": datetime.now(timezone.utc).isoformat(),
               "seed": seed,
               "pool_size": len(pool),
               "winners": [w["email"] for w in winners]}, f, indent=2)
```

My run:

```text
rows 8453 -> unique people 7912
seed 6803436325215821286
 1.  5218  hugo.1007@proton.me
 2.  2644  jonas.2949@fastmail.com
 3.  3899  omar.4274@fastmail.com
 4.  7623  ines.5768@acme.io
 5.  5788  grace.1486@outlook.com
 6.  3998  quentin.1796@outlook.com
 7.  5236  kira.3290@gmail.com
 8.  6944  chloe.4702@proton.me
 9.  7197  hugo.4107@proton.me
10.  6047  rosa.2557@proton.me
```

Two details worth copying. The dedupe runs before the draw, keyed on the lowercased email with the last registration winning, so someone who filled the form twice still gets one entry. And `random.sample` draws without replacement, unlike `random.choices`, which is pitfall 3.

Replaying six months later is short enough to paste into a dispute email:

```python
import json, random
record = json.load(open("draw-record.json"))
pool = build_pool("registrants.csv")       # same dedupe as above
print([w["email"] for w in random.Random(record["seed"]).sample(pool, 10)] == record["winners"])
```

Seeding also answers the "who verified this" question in a way that logging ten email addresses never does. If you would rather skip seeding, `secrets.choice(pool)` per winner is fine too, but keep the printed list as your evidence since there is no number to replay.

## Method 2: Node 22.13.1 — crypto.randomInt with a partial Fisher-Yates

Same job with zero dependencies. [`crypto.randomInt(min, max)`](https://nodejs.org/api/crypto.html#cryptorandomintmin-max-callback) draws from the CSPRNG and handles rejection sampling internally, so unlike `crypto.randomBytes(4).readUInt32BE() % n` it does not skew the low end of the range.

```js
// drawWinners.mjs — Node 22.13.1
import { readFileSync } from "node:fs";
import { randomInt } from "node:crypto";

const lines = readFileSync("registrants.csv", "utf8").trim().split("\n").slice(1);
const byEmail = new Map();
for (const line of lines) {
  const [id, email] = line.split(",");
  byEmail.set(email.trim().toLowerCase(), id);       // last one wins
}
const pool = [...byEmail.entries()].sort((a, b) => Number(a[1]) - Number(b[1]));

function pick(pool, k) {
  const remaining = pool.slice();
  const out = [];
  for (let i = 0; i < k; i++) {
    const j = randomInt(i, remaining.length);        // crypto-backed, no modulo bias
    [remaining[i], remaining[j]] = [remaining[j], remaining[i]];
    out.push(remaining[i]);
  }
  return out;
}

console.log(`rows: ${lines.length} unique: ${pool.length}`);
for (const [n, [email, id]] of pick(pool, 10).entries()) {
  console.log(`${String(n + 1).padStart(2)}. ${id.padStart(5)}  ${email}`);
}
```

On the same CSV:

```text
rows: 8453 unique: 7912
 1.  1514  farid.7145@gmail.com
 2.  7382  sven.4378@acme.io
 3.  7162  kira.3630@gmail.com
 4.  7934  quentin.4656@outlook.com
 5.  3338  quentin.3296@outlook.com
 6.  1595  tara.3459@fastmail.com
 7.  8364  ana.7180@gmail.com
 8.  4519  dmitri.1763@acme.io
 9.  4791  grace.8866@outlook.com
10.  6326  grace.6346@outlook.com
```

Note that `pick` consumes only `k` values and then stops, so the cost stays flat whether the pool holds thousands of rows or millions.

What Node will not give you is a seedable, reproducible CSPRNG stream. `Math.random()` is seeded once per context and cannot be rewound, so replaying an old draw in JS means rolling your own PRNG on top of a `randomInt`-generated seed, or simply doing the draw elsewhere.

## Method 3: when you just want ten numbers right now

Plenty of the time nobody needs a script. Meetup raffle with 200 tickets, splitting a workshop into four groups live on a call, seeding a few fixture values while writing tests. I open [this browser-based random picker](https://utilvy.com/en/tools/random-number-generator/), set the range to 1 through 7912, switch on no-duplicates, ask for 10, and copy them into the sheet.

Everything runs client-side, so the registrant list never travels to a server. That matters more than it sounds: most giveaway exports contain emails you have no permission to push through somebody else's API. Dedupe and sorting are checkboxes there rather than lines you can forget, which makes the third pitfall impossible on that path.

It also covers the boring asks: seat numbers, ticket stubs, ten integers between 1 and 100 to pad a test fixture. For those I use [the random number generator at utilvy](https://utilvy.com/en/tools/random-number-generator/) and never open an editor.

## Pitfalls I actually hit

### 1. The old results page leaked its seed in 10 ms

Before rewriting anything I read the script marketing had been using for previous draws. It was a MINSTD generator seeded from the clock:

```js
let state = BigInt(Date.now());
const next = () => {
  state = (state * 48271n) % 2147483647n;      // Lehmer / MINSTD
  return Number(state) / 2147483647;
};
```

Once three consecutive outputs are visible anywhere, whether on a results page or inside sequential promo codes, the seed falls out by brute force over plausible millisecond values:

```text
observed: [ 3285, 2106, 5622 ]
recovered seed (epoch ms): 1793239041039  in 10.1 ms
```

Ten milliseconds, and every later number from that page was already determined when it was written. MDN puts it plainly: [`Math.random()` does not provide cryptographically secure random numbers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random). `randomInt` and `secrets` do.

### 2. Modulo bias is real, and usually far smaller than you fear

The classic shape:

```python
idx = int.from_bytes(os.urandom(1), "big") % 100
```

A byte spans 0 to 255, which is not a multiple of 100, so 0 through 55 are reachable three ways and 56 through 99 two ways. Across 200,000 draws the low group totalled 131,253 against an expected 131,250, averaging 2,344 hits per slot against 1,562. The busiest slot (46) came up 2,461 times and the quietest (92) only 1,456. That gap is 1.5001x, precisely 3/2.

Scale it to the real pool and it nearly disappears. `2**32 % 8453` leaves a remainder of 6,449, so the first 6,449 people get 508,100 chances each while everyone else gets 508,099. An edge of 0.000197%.

My takeaway after measuring both ends: call the library function because it costs nothing, and stop suspecting `%` of rigging a draw once the pool climbs past a few hundred thousand.

### 3. Ten winners can be nine people

My first draft used `random.choices(pool, k=10)`, which samples with replacement. Over 10,000 simulated draws, 55 of them announced the same email twice. The same test with `random.sample` on the deduped pool produced zero.

Reach for a set to fix it and Python refuses outright:

```text
>>> random.sample(set(emails), 10)
TypeError: Population must be a sequence.  For dicts or sets, use sorted(d).
```

Set support was deprecated in 3.9 and removed in 3.11, and yes, the double space after the period is in the real message. Sort first and it behaves:

```python
random.sample(sorted(set(emails)), 10)
```

The same trap exists one step upstream. My file had 8,453 rows for 7,912 people, so drawing from rows lets even a correct winner list hand two entries to one registrant. Dedupe the people before you draw them.

## Short version

| Goal | Use | Skip |
| --- | --- | --- |
| Unpredictable and replayable | `secrets.randbits(64)` seed, published after the draw | any seed derived from the clock |
| Genuinely distinct winners | dedupe first, then `random.sample` | `random.choices`, or a bare `set` passed to `sample` |
| Unbiased range | `secrets.randbelow`, `crypto.randomInt` | rolling your own `%` |
| Numbers during a call | [a picker that dedupes before it draws](https://utilvy.com/en/tools/random-number-generator/) | uploading the CSV to someone else's API |

More free browser-based utilities at utilvy.com