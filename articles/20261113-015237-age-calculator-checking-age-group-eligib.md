Last August I got pulled into our soccer club's registration mess. The league rule is simple: U-12 means the player was still 11 on August 1, 2026. Not 11 today — 11 on cutoff day. The previous coordinator had checked ages the week before the deadline, eyeballed the leftovers, and moved on. Two kids ended up in the wrong bracket. One of them, born 2014-07-28, was 11 when the list was reviewed but 12 on cutoff day. Protest filed, bracket redone, coordinator embarrassed.

I rebuilt the check as a script and keep it in the repo now. The roster lives in a shared spreadsheet, the league publishes its cutoff in writing, and the whole verification is now a thirty-second job that runs before anyone signs anything. Here are three ways to do the same thing, from most repeatable to fastest for a single case.

## Method 1: a small Python script

Python 3.11.1, standard library only. "Age at a date" is the same borrow logic as "age today", just against a fixed cutoff: you start with the year difference and subtract one whenever the birthday falls after the date you're measuring at. The only twist is that the "current date" is a parameter instead of `date.today()`:

```python
# check_eligibility.py — Python 3.11.1, standard library only
import csv
from datetime import date

CUTOFF = date(2026, 8, 1)
MAX_AGE = 11  # U-12 = still 11 on cutoff day

def age_at(dob: date, at: date) -> int:
    years = at.year - dob.year
    if (at.month, at.day) < (dob.month, dob.day):
        years -= 1
    return years

with open("registrations.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        y, m, d = (int(p) for p in row["dob"].split("-"))
        age = age_at(date(y, m, d), CUTOFF)
        status = "OK" if age <= MAX_AGE else "NOT eligible"
        print(f"{row['player']}: born {row['dob']} -> {age} at cutoff, {status}")
```

The relevant slice of the real output:

```text
Lucas M.: born 2014-07-28 -> 12 at cutoff, NOT eligible
Priya S.: born 2014-08-05 -> 11 at cutoff, OK
```

One subtlety: `(at.month, at.day) < (dob.month, dob.day)` compares tuples on purpose. My first version used `dob.replace(year=at.year)` and a player born 2000-02-29 blew up with `ValueError: day is out of range for month`, because Feb 29 doesn't exist in 2026. Tuple comparison never builds an invalid date, so leap-day birthdays pass through. The [datetime module docs](https://docs.python.org/3/library/datetime.html) have the details if you want them.

## Method 2: Node.js

The same logic in Node 22.13.1, no dependencies. I use this when the roster lands as a spreadsheet export and the rest of our tooling is already JavaScript:

```js
// check.mjs — Node 22.13.1, no dependencies
import { readFileSync } from "node:fs";

const cutoff = new Date(2026, 7, 1); // 0-indexed: 7 = August
const maxAge = 11;

function ageAt(dob, at) {
  let years = at.getFullYear() - dob.getFullYear();
  const beforeBirthday =
    at.getMonth() < dob.getMonth() ||
    (at.getMonth() === dob.getMonth() && at.getDate() < dob.getDate());
  if (beforeBirthday) years--;
  return years;
}

const rows = readFileSync("registrations.csv", "utf8").trim().split("\n").slice(1);
for (const row of rows) {
  const [player, dobStr] = row.split(",");
  const [y, m, d] = dobStr.split("-").map(Number);
  const age = ageAt(new Date(y, m - 1, d), cutoff);
  console.log(`${player}: born ${dobStr} -> ${age} at cutoff`);
}
```

That `m - 1` is doing load-bearing work. More on that in a second. For everything else the Date object can do to you, [MDN's Date reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date) is the page I keep open.

## Method 3: the browser, for one-off cases

The script path earns its keep at roster scale. At registration-desk scale — one parent, one birth certificate, one disputed bracket — opening a terminal is theater. I open [this browser-based age calculator](https://utilvy.com/en/tools/age-calculator/), type the date of birth and the "as of" date, and read out years/months/days plus total days and hours. The years/months/days breakdown matters more than I expected: "eleven and a half" lands very differently than "eleven, turning twelve tomorrow" when you're explaining a decision to a parent. It runs fully client-side, nothing is uploaded, which matters when the input is a child's birth date. Two or three contested cases per season is exactly what [a free online age calculator](https://utilvy.com/en/tools/age-calculator/) is for; 214 rows is what the script is for.

## Three traps I hit so you don't have to

**Trap 1: dateutil's argument order.** My favorite shortcut betrayed me once. I swapped the arguments, and the eligibility check silently passed for the one player it should have flagged:

```python
>>> from datetime import date
>>> from dateutil.relativedelta import relativedelta
>>> r = relativedelta(date(2014, 7, 20), date(2026, 8, 1))
>>> r
relativedelta(years=-12, days=-12)
>>> r.years > 11
False
```

`years=-12`, no exception, no warning — and `r.years > 11` is False, so a 12-year-old stayed in the U-12 list. `relativedelta(cutoff, dob)` is the correct order; the [relativedelta docs](https://dateutil.readthedocs.io/en/stable/relativedelta.html) define it as a subtraction, and the direction is on you.

**Trap 2: JavaScript months start at zero.** A colleague's version built DOBs with `new Date(2013, 8, 2)` for an August 2 birthday. Watch what that actually constructs:

```text
> new Date(2013, 8, 2).toString()
"Mon Sep 02 2013 00:00:00 GMT+0800 (China Standard Time)"
```

September 2. In that case the age number happened to come out the same, because September 2 sits on the same side of the August 1 cutoff as August 2 does. That's exactly what makes the bug nasty: the eligibility column looks right, nothing fails, and the wrong birthday just quietly travels into the league card. The parent caught it on paper. The fix is boring and non-negotiable: subtract one from any month you read out of a human-facing source, and print the parsed date back once so a human sees what the machine thinks.

**Trap 3: unpadded dates in spreadsheets.** Someone hand-edited our export and one cell said `2014-8-2`. One cell, two failures. Python 3.11's `date.fromisoformat` — friendlier since 3.11, but still strict about this — rejects it:

```text
ValueError: Invalid isoformat string: '2014-8-2'
```

The string comparison didn't catch it either, because `'2014-8-2' < '2014-08-01'` evaluates to `False` — `'8'` sorts after `'0'`, so the unpadded date silently sorted as *after* the cutoff. I now normalize every date cell through a parse-and-reformat step before any comparison, and rows that still look off get a ten-second spot check with [the age calculator](https://utilvy.com/en/tools/age-calculator/) before I trust them.

## Which one to pick

| Approach | Best for | Cost |
|---|---|---|
| Python script | Full rosters, repeatable audits | ~20 lines, one-time setup |
| Node script | When data already lives in JS tooling | Same logic, mind the `m - 1` |
| Browser calculator | Single disputed case at a desk | Zero setup, nothing uploaded |

The rule underneath all three never changes: age at a cutoff date means counting completed years between the DOB and the cutoff, borrowing a year when the birthday hasn't happened yet, and never comparing dates as strings.

More free browser-based utilities at [utilvy.com](https://utilvy.com).