Last February I inherited the billing code for a small SaaS. The marketing page promises a 14-day trial. A customer signed up on March 1, 2026, and her card got hit at 00:00 UTC on March 15 — which was 7 p.m. on March 14 in New York. Her ticket said, in full: "I counted. Day 14 was yesterday."

She was right. Our trial gate treated the trial as the half-open interval [signup, signup + 14 days), expiring at March 15 00:00 UTC. She counted the way humans do: signup day is day 1, so her day 14 was March 14. Both readings are defensible. The real bug was that three parts of our codebase picked different conventions and nobody had written one down. Fixing it meant answering one boring question with total precision: given two dates, how many days are between them — and does "between" include the endpoints?

Here's the toolkit I settled on.

## Method 1: Python 3.11.1, standard library only

I stopped guessing and computed both numbers side by side:

```python
# days_between.py — Python 3.11.1, stdlib only
from datetime import date

def breakdown(start: date, end: date) -> dict:
    days = (end - start).days          # half-open: counts [start, end)
    months = (end.year - start.year) * 12 + (end.month - start.month)
    if end.day < start.day:
        months -= 1
    return {
        "exclusive_days": days,       # how long the interval ran
        "inclusive_days": days + 1,   # how many calendar dates it touched
        "weeks_and_days": f"{days // 7}w {days % 7}d",
        "whole_months": months,
    }

print(breakdown(date(2026, 3, 1), date(2026, 3, 15)))
```

Output:

```text
{'exclusive_days': 14, 'inclusive_days': 15, 'weeks_and_days': '2w 0d', 'whole_months': 0}
```

`date(2026, 3, 15) - date(2026, 3, 1)` gives a `timedelta(days=14)`. Add one and you get the 15 calendar dates a daily-charge report sees. Both numbers now go into every log line our billing cron prints. The month arithmetic needs the `if end.day < start.day` guard, by the way — without it, March 1 to March 15 reports as one whole month, which finance will notice before you do.

Then we wrote the convention into the billing README: every interval is half-open, `[start, end)`. Trials expire at the instant `start + 14 days`. Any copy that says "14 days including today" gets translated to `days + 1` at the display layer, never in the ledger. The [datetime module docs](https://docs.python.org/3/library/datetime.html) cover the arithmetic itself; the convention is yours to pick and document.

## Method 2: Node.js 22.13.1, normalize everything through UTC

Our reminder service is TypeScript on Node 22.13.1, and it had a second opinion about dates. The classic trap: `new Date("2026-03-15")` — a date-only string — parses as UTC midnight per the spec ([MDN explains the string formats here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#date_time_string_format)), while `new Date(2026, 2, 1)` parses in the server's local zone. Mix the two and the math goes soft. The fix is boring and total: pin both sides to UTC before subtracting.

```javascript
// days-between.js — Node.js 22.13.1
function daysBetween(startISO, endISO) {
  const a = new Date(`${startISO}T00:00:00Z`); // force UTC
  const b = new Date(`${endISO}T00:00:00Z`);
  return Math.round((b - a) / 86400000);
}
console.log(daysBetween("2026-03-01", "2026-03-15")); // 14
```

To see how bad the mixed version gets, run it in two timezones:

```bash
$ TZ=America/New_York node -e "const a=new Date('2026-03-15'),b=new Date(2026,2,1);console.log((a-b)/86400000, Math.floor((a-b)/86400000))"
13.791666666666666 13
$ TZ=Asia/Shanghai node -e "const a=new Date('2026-03-15'),b=new Date(2026,2,1);console.log((a-b)/86400000, Math.floor((a-b)/86400000))"
14.333333333333334 14
```

Identical code, identical input, `Math.floor` says 13 in New York and 14 in Shanghai. Once both instants are pinned to midnight UTC, the difference is a whole number of days and `Math.round` has nothing to round.

## Method 3: when someone just needs the number

Most date-range questions here come from finance, not engineers. They have a spreadsheet of contract start and end dates and want days, weeks, months. For them I stopped pasting Python snippets into chat and started sending [the days-between-dates calculator on utilvy](https://utilvy.com/en/tools/days-between-dates/). You drop in two dates and it returns the gap in days, plus weeks, whole months and total hours — everything finance asks for in the same breath.

I use it as a third opinion too. After the March incident, every billing change gets cross-checked three ways: the Python breakdown, the Node function, and a ten-second check in [this browser-based date difference tool](https://utilvy.com/en/tools/days-between-dates/). It runs entirely client-side, so pasting contract dates into it cleared our security review without a ticket. That matters more than it sounds; half the reason people hand-compute dates is that pasting customer data into a random web app is a bad idea.

## Three pitfalls that cost me real time

**1. The +1 nobody writes down.** `(b - a).days` for March 1 → March 15 is 14. The number of calendar dates the interval touches is 15. Our dashboard said "14 days" while the daily report counted 15 rows, and the discrepancy shipped for months. Pick a convention, name it in code ("exclusive_days"), and force the other spelling through an explicit `+ 1` with a comment.

**2. Date-only strings are UTC, constructors are local.** The two-TZ output above is the smoking gun. If your CI box runs in UTC and your prod box doesn't, the same expression produces different day counts in each — which is exactly the kind of bug that survives a green test suite.

**3. Same-timezone subtraction lies across DST.** This one cost me an afternoon. In Python, two aware datetimes sharing the same `tzinfo` object subtract wall-clock style — CPython skips the offset conversion when `tzinfo` is the identical object. Across the US spring-forward on March 8, 2026:

```python
>>> from datetime import datetime
>>> from zoneinfo import ZoneInfo
>>> ny = ZoneInfo("America/New_York")
>>> s = datetime(2026, 3, 7, 12, 0, tzinfo=ny)
>>> e = datetime(2026, 3, 9, 12, 0, tzinfo=ny)
>>> e - s
datetime.timedelta(days=2)
```

Two days? The clocks say 47 hours — March 8 is only 23 hours long in New York. Convert both sides to UTC first and the truth appears:

```python
>>> from datetime import timezone
>>> e.astimezone(timezone.utc) - s.astimezone(timezone.utc)
datetime.timedelta(days=1, seconds=82800)
```

One day plus 82,800 seconds: 47 hours. Any day count built on `total_seconds() / 86400` inherits this across a DST boundary, so divide only after both sides are UTC.

## Cheat sheet

| Question | Answer for 2026-03-01 → 2026-03-15 |
|---|---|
| `(end - start).days` (Python) | 14 |
| Calendar dates touched | 15 |
| Node, UTC-pinned | `Math.round((b - a) / 86400000)` → 14 |
| No-code check | [days-between-dates calculator](https://utilvy.com/en/tools/days-between-dates/) → 14 days / 2 weeks |

The customer got a credit and we shipped the convention doc the same week. Intervals are `[start, end)` everywhere, the `+ 1` lives only in reports, and timezones get normalized before any subtraction. Not clever. But nobody has filed a "you charged me a day early" ticket since.

More free browser-based utilities at [utilvy.com](https://utilvy.com).