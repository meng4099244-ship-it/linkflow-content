# Age Calculator: Getting Exact Age in Years, Months and Days from a Birth Date in Python, Node.js, and Your Browser

## The 18+ gate that let a 17-year-old through

I was wiring an age gate for a lending product a few weeks ago. Read the birth date off the form, compare it with today, block anyone under 18 years, 0 months, 0 days. Three lines of Python:

```python
def age_years(birth: date, today: date) -> int:
    return (today - birth).days // 365
```

Every test I wrote passed. Then QA filed a ticket containing one date: `2008-10-20`. On `2026-10-19` that function returns `18`, and the applicant is 17 years, 11 months, 29 days old. One day short, and the gate waved them through.

Age looks like arithmetic, but it is not. A year is 365 or 366 days depending on where you land, and a month is anywhere from 28 to 31, so any formula built on division is wrong somewhere in the calendar. Here is what I replaced it with, in Python and Node, plus the shortcut I reach for when I only need one number.

## Method 1: subtract with borrowing (Python 3.11.1)

Do it the way you would on paper. Subtract the components, and when the day count goes negative, borrow from the month. The one thing you need from the standard library is the length of the month you are borrowing from, and that is exactly what [`calendar.monthrange`](https://docs.python.org/3/library/calendar.html) returns.

```python
import calendar
from datetime import date


def days_in_prev_month(d: date) -> int:
    """How many days the month before `d` has (handles the January wrap)."""
    if d.month == 1:
        return calendar.monthrange(d.year - 1, 12)[1]
    return calendar.monthrange(d.year, d.month - 1)[1]


def exact_age(birth: date, today: date) -> tuple[int, int, int]:
    if birth > today:
        raise ValueError(f"birth date {birth} is after {today}")
    years = today.year - birth.year
    months = today.month - birth.month
    days = today.day - birth.day

    if days < 0:                      # borrow a month
        months -= 1
        days += days_in_prev_month(today)

    if months < 0:                    # borrow a year
        years -= 1
        months += 12

    return years, months, days


if __name__ == "__main__":
    today = date(2026, 10, 19)
    for birth in [date(1990, 6, 15), date(2008, 10, 20), date(2000, 2, 29)]:
        y, m, d = exact_age(birth, today)
        print(f"{birth}  {y}y {m}m {d}d   total days: {(today - birth).days}")
```

Output:

```text
1990-06-15  36y 4m 4d   total days: 13275
2008-10-20  17y 11m 29d   total days: 6573
2000-02-29  26y 7m 20d   total days: 9729
```

The middle line is the one that broke my gate, and it now says `17y 11m 29d`.

Keep the January branch in `days_in_prev_month`. If you let `d.month - 1` reach zero you get `IllegalMonthError: bad month number 0; must be 1-12`, and it only shows up in production during the first week of January.

## Method 2: the same borrow in Node.js 22.13.1

Node gives you less help here than you would expect. `Temporal` is still not switched on in v22.13.1, so `typeof Temporal` prints `undefined` and there is no flag to enable it. The borrowing logic gets written by hand.

I also parse the ISO strings myself instead of building `Date` objects. The reason is in the traps section below.

```js
"use strict";
// Exact age in years, months, days. Node.js v22.13.1, no dependencies.

function parseISODate(iso) {
  const [y, m, d] = iso.split("-").map(Number);
  return { year: y, month: m, day: d };
}

function daysInMonth(year, month) {
  return new Date(Date.UTC(year, month, 0)).getUTCDate(); // day 0 = last day of prev month
}

function exactAge(birthISO, todayISO) {
  const b = parseISODate(birthISO);
  const t = parseISODate(todayISO);

  let years = t.year - b.year;
  let months = t.month - b.month;
  let days = t.day - b.day;

  if (days < 0) {
    months -= 1;
    const prevYear = t.month === 1 ? t.year - 1 : t.year;
    const prevMonth = t.month === 1 ? 12 : t.month - 1;
    days += daysInMonth(prevYear, prevMonth);
  }
  if (months < 0) {
    years -= 1;
    months += 12;
  }
  return { years, months, days };
}

const today = "2026-10-19";
for (const birth of ["1990-06-15", "2008-10-20", "2000-02-29"]) {
  const a = exactAge(birth, today);
  console.log(`${birth}  ${a.years}y ${a.months}m ${a.days}d`);
}
```

```text
1990-06-15  36y 4m 4d
2008-10-20  17y 11m 29d
2000-02-29  26y 7m 20d
```

Same three answers as the Python version, which is the cross-check I wanted before trusting either one.

## Method 3: when you just need the number once

Most days I do not need age inside a program at all. I need it once, while staring at a record that looks wrong or a date someone pasted into Slack. Opening a REPL for that is silly.

So I keep [a free online age calculator](https://utilvy.com/en/tools/age-calculator/) open in a tab. Paste the birth date, pick the reference date, and it hands back years, months and days plus the totals in days, hours and minutes. No install, and the input stays in the browser, which matters when the birth date belongs to a real customer rather than a fixture in my test file.

## Three traps I hit

**1. Dividing by 365, or by 365.25.** They fail in opposite directions. With birth `2008-10-20` and today `2026-10-19`:

```text
days = 6573 | //365 = 18 | int(/365.25) = 17
```

`//365` claims 18, and the true age is 17 years, 11 months, 29 days. Now flip it, for a child born `2025-10-19` who turns one today:

```text
days = 365 | //365 = 1 | int(/365.25) = 0
```

Here `//365` is right and 365.25 says the child is zero. I scanned a 40-year window: `//365` disagreed with the real age on 105 birth dates, and 365.25 disagreed on 20. Neither constant is safe.

**2. `new Date('2000-02-29')` is not a local date.** Date-only ISO strings are parsed as UTC midnight. Read the fields back with local getters in a negative-offset timezone and you get the day before:

```text
$ TZ=America/New_York node tz.js
getUTCDate(): 29
getDate()   : 28
toString()  : Mon Feb 28 2000 19:00:00 GMT-0500 (Eastern Standard Time)
```

Under `TZ=UTC` both getters print `29`. That is why the two implementations above work on year/month/day integers pulled out of the string. MDN covers the parsing rule under [date time string format](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#date_time_string_format).

**3. February 29 has no anniversary.** The obvious "swap in the current year" trick dies immediately:

```python
>>> date(2000, 2, 29).replace(year=2026)
ValueError: day is out of range for month
```

The borrow version survives it, returning `26y 7m 20d` for `2000-02-29` on `2026-10-19`, because the month it borrows is September with 30 days. Whether a leap-day baby turns 18 on February 28 or March 1 is a legal question rather than a calendar one, so settle it with whoever owns the policy before you ship.

## Which approach fits

| Situation | Use |
| --- | --- |
| Age inside a Python service | `exact_age()` above, standard library only |
| Age inside a Node service | the same borrow logic, fields parsed with `.split("-")` |
| One-off check while debugging | [this browser-based tool](https://utilvy.com/en/tools/age-calculator/) |
| Anything with legal weight | borrow logic plus a written rule for Feb 29 |
| Never | `(today - birth).days // 365` |

## Wrapping up

Age is a calendar question wearing an arithmetic costume. Subtract years, months and days with borrowing instead of dividing, keep birth dates as plain calendar dates rather than instants in time, and let [the online age calculator](https://utilvy.com/en/tools/age-calculator/) handle the cases where you only need one answer. Those twenty lines replaced a three-line bug that would have let minors into a lending flow.

More free browser-based utilities at utilvy.com