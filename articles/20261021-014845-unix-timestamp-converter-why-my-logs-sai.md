# Unix Timestamp Converter: Why My Logs Said 1970-01-21 (and the Seconds vs Milliseconds Fix)

I spent a solid forty minutes last Tuesday staring at a billing page that claimed every subscription in our system had expired on 1970-01-21.

Not 1970-01-01, the date everyone learns to recognize instantly. 1970-01-21, a much more specific flavour of wrong. That particular wrongness has exactly one cause: a millisecond timestamp got handed to code that expected seconds.

The API was fine. My frontend was not.

If you have ever debugged a date that landed in 1970, or a "today" that showed up eight hours early, this is the checklist I wish I had taped to my monitor. Three ways to convert a Unix timestamp, then three traps I keep stepping in.

## Method 1: Python 3.11.1

Server side, `datetime.fromtimestamp()` is the whole job. The one argument that matters is the second one.

```python
# Python 3.11.1
import datetime

ts = 1761000000

utc = datetime.datetime.fromtimestamp(ts, datetime.timezone.utc)
print(utc.isoformat())
# 2025-10-20T22:40:00+00:00

print(datetime.datetime.fromtimestamp(ts).isoformat())
# 2025-10-21T06:40:00   <- same instant, my laptop is UTC+8
```

Skip the timezone argument and you get a naive datetime silently rendered in local time. That is where off-by-eight-hours bugs are born.

Going the other direction, build an aware datetime and read `.timestamp()`:

```python
# Python 3.11.1
import datetime

d = datetime.datetime(2026, 10, 21, 0, 0, 0, tzinfo=datetime.timezone.utc)
print(int(d.timestamp()))
# 1792540800

print(datetime.datetime.now(datetime.timezone.utc).isoformat())
# 2026-10-21T01:47:19.575814+00:00
```

I always store UTC and format at the edge. The [Python datetime docs](https://docs.python.org/3/library/datetime.html#datetime.datetime.fromtimestamp) spell out the naive-vs-aware rules better than I can, and they are worth reading once.

## Method 2: Node.js 22.13.1

JavaScript's `Date` constructor takes milliseconds. Full stop. It does not guess, and it does not warn.

```javascript
// Node.js 22.13.1
const seconds = 1761000000;

console.log(new Date(seconds).toISOString());
// 1970-01-21T09:10:00.000Z   <- wrong, this is my bug from Tuesday

console.log(new Date(seconds * 1000).toISOString());
// 2025-10-20T22:40:00.000Z   <- correct
```

Backwards is `Date.now()` (milliseconds) or `Math.floor(Date.now() / 1000)` when the backend wants seconds:

```javascript
// Node.js 22.13.1
console.log(Date.now());
// 1792547248983

console.log(Math.floor(Date.now() / 1000));
// 1792547248
```

MDN's [Date](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date) page is the reference I keep open, especially the section on date-string parsing, which never behaves the way people assume.

When I am already in a terminal, GNU `date` is faster than opening an editor:

```bash
TZ=UTC date -d @1761000000 '+%Y-%m-%d %H:%M:%S %Z'
# 2025-10-20 22:40:00 UTC

TZ=UTC date -d "2026-10-21 01:47:19" +%s
# 1792547239
```

## Method 3: paste it into a browser

Sometimes I am reading a stranger's log line and do not want to boot anything. A [free online Unix timestamp converter](https://utilvy.com/en/tools/timestamp-converter/) handles both directions in one input box, and shows UTC and local time side by side, which is the part I actually care about when the number came from a server in another region.

I keep [this browser-based timestamp converter](https://utilvy.com/en/tools/timestamp-converter/) bookmarked for exactly that. Nothing is uploaded, the arithmetic happens in the page, and I can hand the link to a support engineer who has never opened a REPL.

## Three traps worth memorizing

**1. Seconds versus milliseconds.** Symptoms: a date in January 1970, or a year around 57000. In Node, `new Date(1761000000)` returns `1970-01-21T09:10:00.000Z`. In Python, passing the millisecond value straight in does not degrade gracefully, it throws:

```
>>> datetime.datetime.fromtimestamp(1761000000000)
ValueError: year 57773 is out of range
```

That traceback is a gift. It means milliseconds, and the fix is to divide by 1000.

**2. Timezone drift on round trips.** A date-only string is parsed as UTC, but a date-time string without an offset is parsed as local. On Node 22.13.1 with `TZ` unset on a UTC+8 box:

```javascript
new Date('2026-10-21').toISOString()
// 2026-10-21T00:00:00.000Z
new Date('2026-10-21T00:00:00').toISOString()
// 2026-10-20T16:00:00.000Z
```

Python mirrors the trap in reverse. `datetime.datetime(2026, 10, 21, 0, 0).timestamp()` returned `1792512000.0` on my machine, while the same value marked UTC returned `1792540800.0`. Eight hours apart, because a naive datetime is interpreted in local time. Set `TZ` explicitly in CI, or your tests pass locally and fail in the container.

**3. The 2038 ceiling and 32-bit storage.** `2147483647` is `2038-01-19T03:14:07Z`. Store timestamps in a 64-bit column or as `bigint`, because a signed 32-bit int wraps one second later. Postgres `integer` and legacy MySQL `INT` columns are the usual offenders, and they will not complain until they do.

I also stopped trusting shell date syntax across machines. GNU wants `date -d @1761000000`; macOS and BSD want `date -r 1761000000`. Run the GNU form on a BSD box and you get:

```
date: 1761000000: No such file or directory
```

Same command, same number, completely different parsing. Any script that might run on a laptop needs the branch.

| Task | Python 3.11.1 | Node.js 22.13.1 | Shell |
| --- | --- | --- | --- |
| Epoch to UTC | `datetime.fromtimestamp(ts, timezone.utc)` | `new Date(ts * 1000)` | `TZ=UTC date -d @$ts` |
| Date to epoch | `int(aware_dt.timestamp())` | `Math.floor(Date.now() / 1000)` | `TZ=UTC date -d "$str" +%s` |
| Unit | seconds, float | milliseconds | seconds |
| Main trap | naive datetime reads local | seconds treated as ms | BSD `date -r` vs GNU `date -d` |

Two habits fixed this class of bug for me: I name every timestamp field with its unit (`expires_at_s`, `created_ms`), and I never let a naive datetime cross a function boundary. When I do need to eyeball a number fast, the [timestamp converter at utilvy](https://utilvy.com/en/tools/timestamp-converter/) shows both directions at once, which is usually enough to confirm whether I am off by a factor of 1000.

More free browser-based utilities at utilvy.com