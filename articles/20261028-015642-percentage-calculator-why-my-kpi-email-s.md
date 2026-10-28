# Percentage Calculator: Why My KPI Email Said +1.23% When Conversion Actually Grew 159%

Last Tuesday our weekly KPI email went out with a line reading "Conversion rate: +1.23%". Two hours later the growth lead replied, asking why we were celebrating a 1% lift on a release that had doubled checkout conversions.

Both numbers came out of the same query. One measured percentage points, the other measured relative change, and the email never said which. Cleaning that up cost me a morning and a rewrite of the report job.

Below is how I now handle the three percentage questions that show up on almost every dashboard: X% of Y, X is what percent of Y, and the percent change between two values.

## Method 1: Python 3.11.1 with Decimal

My report job used to run on floats. For anything involving money that is a bad idea, so the pricing half now uses `decimal`. The two conversion rows from that week are baked into the script below.

```python
# percentages.py  (Python 3.11.1)
from decimal import Decimal, ROUND_HALF_UP

def q(x: Decimal, places: str = "0.01") -> Decimal:
    return x.quantize(Decimal(places), rounding=ROUND_HALF_UP)

def pct_of(p, n):
    """X% of Y   ->   20% of 79.00"""
    return Decimal(n) * Decimal(p) / 100

def pct_share(part, whole):
    """X is what % of Y   ->   96 out of 12480"""
    return Decimal(part) * 100 / Decimal(whole)

def pct_change(old, new):
    """relative change from old to new"""
    if old == 0:
        return None          # no baseline, see pitfall 2
    return (Decimal(new) - Decimal(old)) / Decimal(old) * 100

week1 = {"visitors": 12480, "orders": 96}
week2 = {"visitors": 11930, "orders": 238}

r1 = pct_share(week1["orders"], week1["visitors"])
r2 = pct_share(week2["orders"], week2["visitors"])
print("week1 CR:", q(r1), "%")
print("week2 CR:", q(r2), "%")
print("change  :", q(pct_change(r1, r2), "0.1"), "% relative")
print("points  :", q(r2 - r1), "pp")

seat = Decimal("79.00")
print("20% of seat :", q(pct_of(20, seat)))     # the discount itself
after_20 = q(pct_of(80, seat))                  # 100% - 20% = what the customer pays
after_coupon = q(pct_of(85, after_20))          # then 15% off that discounted price
print("20% then 15%:", after_coupon)
print("flat 35% off:", q(pct_of(65, seat)))
```

Running it prints:

```
week1 CR: 0.77 %
week2 CR: 1.99 %
change  : 159.3 % relative
points  : 1.23 pp
20% of seat : 15.80
20% then 15%: 53.72
flat 35% off: 51.35
```

So 159.3% was the real lift, and the 1.23 in my email was percentage points. Both numbers are correct. Only one of them answered the question the growth lead was actually asking.

`Decimal` earns its keep on rounding, not on phantom digits. Ask Python for `round(2.675, 2)` and you get `2.67`. Ask for `Decimal("2.675").quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)` and you get `2.68`, which is the answer a human with a pencil would give.

On a $40.53 cart, 15% comes out as `6.0795` in floats and `6.08` once quantized. One cent is nothing until the invoice runs to four thousand lines. The [decimal module docs](https://docs.python.org/3/library/decimal.html) are worth ten minutes if you have never picked a rounding mode on purpose.

## Method 2: Node.js 22.13.1 with Intl.NumberFormat

The API behind the dashboard is Node, so the same three helpers live there. I format with `Intl.NumberFormat` rather than `toFixed`, partly because `toFixed` hands back a string and partly because it rounds in ways I did not expect. Pitfall 3 covers that.

```js
// percentages.mjs  (Node.js v22.13.1)
const pctOf = (p, n) => (n * p) / 100;
const pctShare = (part, whole) => (part / whole) * 100;
const pctChange = (from, to) => (from === 0 ? null : ((to - from) / from) * 100);

const nf2 = new Intl.NumberFormat("en-US", { minimumFractionDigits: 2, maximumFractionDigits: 2 });
const nf1 = new Intl.NumberFormat("en-US", { minimumFractionDigits: 1, maximumFractionDigits: 1 });

const rows = [
  { week: "2026-W42", visitors: 12480, orders: 96 },
  { week: "2026-W43", visitors: 11930, orders: 238 },
];

const cr = (r) => pctShare(r.orders, r.visitors);
for (const r of rows) {
  console.log(`${r.week}  ${r.orders}/${r.visitors}  CR = ${nf2.format(cr(r))}%`);
}

const [a, b] = rows;
console.log("lift   :", `${nf1.format(pctChange(cr(a), cr(b)))}% relative`);
console.log("points :", `${nf2.format(cr(b) - cr(a))} pp`);
console.log("20% of 79:", nf2.format(pctOf(20, 79)));
console.log("seat     :", nf2.format(pctOf(80, 79)));
```

Output:

```
2026-W42  96/12480  CR = 0.77%
2026-W43  238/11930  CR = 1.99%
lift   : 159.3% relative
points : 1.23 pp
20% of 79: 15.80
seat     : 63.20
```

Node agrees with Python here, which is the point of writing both. [MDN's Intl.NumberFormat page](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat) is the reference when you need locale separators, since `53.72` prints as `53,72` across much of Europe.

## Method 3: a browser tool when you only need the number

Most days I am not writing a report job. I am on a call and someone asks what 238 out of 11,930 is as a percent, and I need it before the call ends.

For that I keep a free online [percentage calculator](https://utilvy.com/en/tools/percentage-calculator/) open in a tab. It covers the three modes I actually use, and it runs entirely client-side, so pasting revenue figures into it does not send them anywhere. No signup either.

I reach for it on the pricing questions as well. Typing `79 * 0.8 * 0.85` into a search box is slower than using [this browser-based tool](https://utilvy.com/en/tools/percentage-calculator/), which gives me 53.72 in about two seconds.

## Three things that bit me

**1. Percentage points are not percent.** Going from 0.77% to 1.99% is +1.23 pp and +159.3% relative. My email printed "+1.23%" with no unit, which is why it read as a rounding blip. Now I print both and spell out `pp`. If you can only show one, show the relative figure, because that is the one people argue about.

**2. A zero baseline crashes Python and lies in JavaScript.** A newly launched segment had zero orders the previous week, so the cron job died on a Monday:

```
ZeroDivisionError: division by zero
```

The Node version does not crash. It returns `Infinity`, and for a segment that was empty in both weeks it returns `NaN`:

```
lift(0,12) = Infinity
lift(0,0)  = NaN
```

Then it gets quieter. `JSON.stringify({lift: NaN})` produces `{"lift":null}`, so the dashboard drew a tidy 0% instead of failing loudly. Guard the baseline, return `null`, and render "n/a" in the UI.

**3. Discounts compound, and rounded numbers drift.** Finance told a customer that a 20% plan discount plus a 15% coupon meant 35% off. Compounding gives 32.0%, so a $79.00 seat costs $53.72 rather than $51.35. That gap is $2.37 per seat, or $1,137.60 across 480 seats. I ran the compounding through [the percentage page at Utilvy](https://utilvy.com/en/tools/percentage-calculator/) before I sent the correction, just to be sure I had not inverted it.

Rounding drifts too. A three-way revenue split stored as 33.33% per party totals `99.99`, and the missing hundredth belongs to whoever reconciles the ledger at month end. In Node, `Math.round(1.005 * 100) / 100` returns `1` because `1.005 * 100` is `100.49999999999999`. Round once at the end, never row by row.

## Which approach for which job

| Situation | Use | Watch out for |
|---|---|---|
| Money, invoices, discounts | Python 3.11.1 with `Decimal` | Quantize once, at the end |
| API and dashboard formatting | Node 22.13.1 with `Intl.NumberFormat` | `toFixed` rounding surprises |
| A one-off answer on a call | browser-based percentage tool | Paste, never upload |
| Any "change" figure | relative %, with `pp` next to it | Zero baseline returns `null`, not 0 |

The bug in that KPI email was never the arithmetic. It was that I shipped a number without its unit. Once the report printed "159.3% relative (1.23 pp)", nobody had to ask which one it was.

More free browser-based utilities at utilvy.com