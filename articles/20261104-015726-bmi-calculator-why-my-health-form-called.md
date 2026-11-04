# BMI Calculator: Why My Health Form Called a 190 lb User Obese (and the Unit Fixes That Stopped It)

Last month I shipped a health-check form for a corporate wellness dashboard. Two weeks later the clinic nurse called, annoyed. A tester who weighs 190 lb at 5'11" had been flagged "obese" by our form. Her own calculation said 26.5, overweight at worst, nowhere near obese.

Both numbers came from the same formula. The difference was what we fed into it. She had her numbers in five seconds on paper. We had a bug that took three days and three separate unit mistakes to pin down. Here is how I traced it, plus the three ways I now verify BMI results before anyone sees them.

## Method 1: A small Python script

Start with the metric formula, because metric removes one whole class of unit bugs. I keep this as `bmi.py` and run it on Python 3.11.1:

```python
# bmi.py — Python 3.11.1
def bmi(kg: float, cm: float) -> float:
    return kg / (cm / 100) ** 2

def category(value: float) -> str:
    if value < 18.5:
        return "underweight"
    if value < 25:
        return "healthy"
    if value < 30:
        return "overweight"
    return "obese"

print(f"{bmi(70, 175):.1f} {category(bmi(70, 175))}")
# 22.9 healthy

print(f"{bmi(86.2, 180):.1f} {category(bmi(86.2, 180))}")
# 26.6 overweight
```

The `cm / 100` conversion lives inside the function, so callers can pass centimeters straight from the form without thinking. The category cut-offs follow the [WHO BMI classification](https://www.who.int/news-room/fact-sheets/detail/obesity-and-overweight): under 18.5, 18.5 to 24.9, 25 to 29.9, then 30 and up.

When I tested the nurse's case in metric (86.2 kg at 180 cm), Python gave 26.6. No obesity anywhere. That told me the server math was fine and the bug was on the input path.

## Method 2: JavaScript for imperial users

Our American testers entered height in feet and inches, so the front end needed its own path. The standard constant is 703:

```js
// bmi.js — Node.js 22.13.1
const bmiImperial = (lb, feet, inches) =>
  (703 * lb) / (feet * 12 + inches) ** 2;

console.log(bmiImperial(190, 5, 11).toFixed(1));
// "26.5"
```

With feet and inches as separate arguments, `bmiImperial(190, 5, 11)` returns 26.496726839912714, which rounds to 26.5. That matches the nurse's number exactly.

One thing worth knowing if you ever wonder where 703 comes from: it is the conversion factor between the metric and imperial forms of the same equation. If you compute the weight in kilograms and the height in inches, you multiply by 703. I have seen codebases hardcode 730 or 700 from memory, which quietly skews every result by about one percent. Keep it as a named constant with a comment, not a magic number.

For quick sanity checks during code review I also re-run the numbers in [a free online BMI calculator](https://utilvy.com/en/tools/bmi-calculator/) rather than pasting code into a REPL every time. It runs entirely in the browser, shows the WHO category and the healthy weight range for the height you entered, and I keep it open in a second tab whenever I am comparing our backend against reference values.

## The three bugs that caused the mismatch

### 1. Centimeters entered as meters

The form field was labeled "Height (cm)". One tester typed `1.75`. Look what the metric function does with that:

```js
const bmi = (kg, cm) => kg / Math.pow(cm / 100, 2);
console.log(bmi(70, 175));  // 22.857142857142858
console.log(bmi(70, 1.75)); // 228571.42857142855
```

The second output is not a typo. Dividing 1.75 by 100 treats the man as 1.75 centimeters tall, and the BMI explodes to six digits. Our UI helpfully printed it as `228571.4` with no sanity check, because the front end trusted whatever arrived in the POST body. Nobody caught it during testing either, since every tester typed a plausible two-digit or three-digit value. I now reject any height above 260 or below 50 centimeters at the input handler; real people do not fall outside that range.

### 2. Decimal feet instead of feet plus inches

This was the actual "obese" bug. The imperial form had a single decimal field, and the tester typed `5.11` meaning five feet eleven inches. But 5.11 feet is 5.11 × 0.3048 = 1.5575 meters, not the 1.8034 meters of a real 5'11". Same 190 lb weight, two very different results:

```js
bmiImperial(190, 5, 11);     // 26.496726839912714 → 26.5, overweight
703 * 190 / (5.11 * 12) ** 2; // 35.52259084655942  → 35.5, obese
```

Nine points of BMI came from eleven inches hiding inside two decimal places. The fix was boring but effective: two separate integer fields for feet and inches, no decimal input allowed. Cross-checking the corrected form against the [browser-based BMI tool on Utilvy](https://utilvy.com/en/tools/bmi-calculator/) confirmed both paths now agree on 26.5 for every test case.

### 3. Display rounding vs. category boundary

One tester at 68 kg and 1.65 m produced raw BMI 24.977043158861342. The category function reads the raw value and says `healthy`, since 24.977 is below 25. But the display pipeline called `toFixed(1)` first:

```js
(68 / Math.pow(1.65, 2)).toFixed(1); // "25.0"
```

So the screen showed **"25.0 · healthy"**, which looks like a contradiction and generated a support ticket. Python behaves the same way: `round(24.977, 1)` returns `25.0`. The category must always come from the raw value, never the rounded one; if the displayed number sits within 0.1 of a boundary, I add a footnote like "rounded; category computed from the exact value". Categorizing from the rounded string would misclassify anyone sitting near 25.0 or 30.0.

If you would rather not maintain any of this, the [BMI calculator page](https://utilvy.com/en/tools/bmi-calculator/) handles both unit systems, applies the WHO categories from the unrounded value, and shows the healthy weight range alongside the score — handy for spot-checking your own implementation.

## Quick summary

| Method | Best for | Watch out for |
| --- | --- | --- |
| Python script (3.11.1) | Server-side validation, batch checks | Height must arrive in cm; guard the range |
| JavaScript (Node 22.13.1) | Imperial users in the browser | Split feet and inches; 703 is the constant |
| Browser-based BMI tool | One-off checks, QA cross-referencing | None. No unit input needed |

The lesson from the nurse's phone call: the formula is one line of code. The units, the input format, and the rounding are where the actual bugs live.

More free browser-based utilities at [utilvy.com](https://utilvy.com/en/).