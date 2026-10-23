The ticket said the dock scale and the dashboard disagreed by about a kilogram. It was a Friday, the shipment was already on a truck, and the customs form listed a weight our own API had produced.

I pulled the rows. The supplier sent `154.3 lb` per carton, ten cartons. Our importer stored `69.99 kg` each. The forwarder's sheet said `69.99`. So far, so good.

The importer was innocent. A colleague wrote it in 2019 against the 1959 definitions and left a comment explaining the exact factors. The label service was the one that recomputed weight for the print job, and it had its own helper, added three years later by someone who never saw that comment. Two code paths, two answers, one shipment.

The helper looked like this:

```python
kg = lb / 2.2
```

For 1,000 lb that returns `454.5454545454545`. The correct answer is `453.59237`. That's 0.95 kg of phantom cargo per half ton, and on a 40-pallet order it grows into something a customs officer notices.

The fix wasn't a smarter library. It was using the definition instead of a memory.

## Method 1: Python 3.11.1 with the exact factors

The pound and the inch are not approximations. Since the 1959 agreement they're defined against the kilogram and the metre: 1 lb = 0.45359237 kg exactly, 1 in = 2.54 cm exactly. Those digits are the definition, not a rounded conversion — [NIST SP 811](https://www.nist.gov/pubs/sp/811/final) lists them as exact.

So the patch stayed small:

```python
# convert.py - Python 3.11.1, standard library only
from decimal import Decimal, ROUND_HALF_UP

LB_TO_KG = Decimal("0.45359237")
IN_TO_CM = Decimal("2.54")

def lb_to_kg(lb: str) -> Decimal:
    return Decimal(lb) * LB_TO_KG

def in_to_cm(inch: str) -> Decimal:
    return Decimal(inch) * IN_TO_CM

def f_to_c(f: str) -> Decimal:
    return (Decimal(f) - Decimal("32")) * Decimal("5") / Decimal("9")

def show(value: Decimal, places: int = 2) -> str:
    q = Decimal(1).scaleb(-places)
    return str(value.quantize(q, rounding=ROUND_HALF_UP))

if __name__ == "__main__":
    print("1000 lb  ->", show(lb_to_kg("1000"), 3), "kg")
    print("154.3 lb ->", show(lb_to_kg("154.3"), 2), "kg")
    print("70 in    ->", show(in_to_cm("70"), 1), "cm")
    print("350 F    ->", show(f_to_c("350"), 1), "C")
```

```
$ python3 convert.py
1000 lb  -> 453.592 kg
154.3 lb -> 69.99 kg
70 in    -> 177.8 cm
350 F    -> 176.7 C
```

Two details in there matter more than they look. Inputs arrive as strings, because `Decimal(154.3)` starts from a float and inherits its noise. And `ROUND_HALF_UP` is deliberate: the default for `quantize` is `ROUND_HALF_EVEN`, which sends 69.985 to 69.98. Our invoices get checked by people with desk calculators, and people round halves up.

## Method 2: Node.js 22.13.1, rounding only at the edge

The Node service renders the shipping label, so it needs the same numbers. I didn't install a units package. Three constants cover our whole catalogue.

```js
// convert.mjs - Node.js 22.13.1, no dependencies
const LB_TO_KG = 0.45359237;
const IN_TO_CM = 2.54;

const lbToKg = (lb) => lb * LB_TO_KG;
const inToCm = (inch) => inch * IN_TO_CM;
const fToC = (f) => (f - 32) * 5 / 9;

const fmt = (n, d = 2) =>
  new Intl.NumberFormat("en-US", {
    minimumFractionDigits: d,
    maximumFractionDigits: d,
  }).format(n);

console.log(fmt(lbToKg(1000), 3), "kg");   // 453.592 kg
console.log(fmt(lbToKg(154.3), 2), "kg");  // 69.99 kg
console.log(fmt(inToCm(70), 1), "cm");     // 177.8 cm
console.log(fmt(fToC(350), 1), "C");       // 176.7 C
console.log(lbToKg(1000));                 // 453.59237
```

Keep the float through the whole calculation and format once, at render. Round per row and the errors don't cancel, they stack in whichever direction the rounding fell. [Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat) also gives you the thousands separator for 12,500 kg pallet totals without any string surgery.

## Method 3: the browser tab I open during a support call

Not every conversion happens in a repo. Last week a customer called about an oven profile their US partner had sent in Fahrenheit. I had their sheet open, no terminal, and a person waiting on the line.

I typed 350 into the temperature tab of this [free online unit converter](https://utilvy.com/en/tools/unit-converter/) and read 176.7 °C. Same for the carton size: 70 in came back as 177.8 cm. Then the carton weight, 154.3 lb, which gave 69.99 kg and matched the label in my hand.

That last check is the only one I care about mid-call. Three tabs: length, weight, temperature. The swap button flips direction when someone hands you kilos and wants pounds back.

What I actually like is that it runs client-side. Supplier price sheets and carton weights don't get uploaded anywhere, which is the reason I stopped pasting them into the first converter Google offered me. When you need the number and not a code path, [this browser-based tool](https://utilvy.com/en/tools/unit-converter/) is quicker than opening a REPL, and it agrees with the scripts above on every value I checked.

## Three mistakes I've made myself

**Dividing by 2.2.** Everyone learns this one. The error is 0.095%, which sounds harmless until a 20-tonne container is off by 19 kg and the bill of lading no longer matches.

```python
>>> 1000 / 2.2
454.5454545454545
>>> 1000 * 0.45359237
453.59237
```

**Forgetting the offset in Fahrenheit.** Fahrenheit isn't a scaled Celsius, it's shifted. Multiply first and you're 17.8 degrees too hot:

```python
>>> 350 * 5 / 9
194.44444444444446
```

A 350 °F oven is 176.7 °C. The first number browns a cake in twelve minutes; the second one bakes it.

**Rounding twice.** A CSV import summed ten cartons in float and wrote `699.8930269100002` into the table. The same sum in `Decimal` gave `699.893026910`. Neither is wrong, but they don't match, and the reconciliation report flagged it every night for a week. Pick one precision, one rounding mode, and convert at the boundary only.

And one that's cheap to avoid: this converter does length, weight, and temperature. Volume isn't in it, and US versus imperial gallons differ by 20%. Don't stretch a length factor to cover it.

## Summary

| Approach | Use it for | Watch out for |
|---|---|---|
| Python `Decimal` + exact factors | invoices, CSV import, anything audited | pass strings, not floats; set the rounding mode explicitly |
| Node.js constants + `Intl.NumberFormat` | labels, dashboards, anything rendered | round once, at output |
| [Online converter](https://utilvy.com/en/tools/unit-converter/) | one-off checks, support calls, locked-down machines | length, weight, temperature only |

The scale was right the whole time. The arithmetic was a memory of a number instead of the number itself.

We ended up with one function, `to_metric()`, called from three places, and a test file holding eight cases copied out of real shipment rows. The `2.2` helper is deleted. Nobody has reopened that ticket.

More free browser-based utilities at utilvy.com