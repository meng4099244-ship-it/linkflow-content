Last quarter our billing service pushed 412 invoices to the bank's batch API and 412 came back rejected. Same root cause on every one: the "Amount in words" line. We were generating it with a one-liner that turned `1078.10` into "one thousand and seventy-eight point one dollars". The bank wanted cheque style — "US Dollars One Thousand and Seventy-Eight and Cents Ten Only" — and its validator refuses anything that doesn't match the figure exactly. Two weeks of printing costs, one very annoyed accountant, and a support ticket later, I sat down and did this properly. Here are the three approaches that work, and the three bugs that cost me the afternoon.

## Method 1: Python 3.11 with num2words

I tested this on Python 3.11.8 with num2words 0.5.14 ([docs on PyPI](https://pypi.org/project/num2words/)). The key move is to never let the library see a float. Split the money value into an integer dollars part and an integer cents part first, then convert each separately:

```python
# pip install num2words==0.5.14  (Python 3.11.8)
from decimal import Decimal
from num2words import num2words

def amount_in_words(amount: str, currency: str = "US Dollars") -> str:
    d = Decimal(amount).quantize(Decimal("0.01"))
    dollars, cents = divmod(int(d * 100), 100)
    text = f"{num2words(dollars, lang='en').title()} {currency}"
    if cents:
        text += f" And Cents {num2words(cents, lang='en').title()}"
    return f"{text} Only"

print(amount_in_words("1078.10"))
print(amount_in_words("21646.15"))
print(amount_in_words("500"))
```

Output:

```
US Dollars One Thousand And Seventy-Eight And Cents Ten Only
US Dollars Twenty-One Thousand, Six Hundred And Forty-Six And Cents Fifteen Only
US Dollars Five Hundred Only
```

Feeding the string `"1078.10"` into `Decimal` (not the float `1078.10`) is what makes this exact. `num2words` itself handles the grammar — the "and" placement, hyphens in "forty-six" — which is the part I always get wrong by hand. If you want zero of this code and just need a one-off amount spelled out, a [free online number-to-words converter](https://utilvy.com/en/tools/number-to-words/) does the same thing in the browser with nothing to install.

## Method 2: Node.js with number-to-words

The Node side is where I lost the most time. On Node 22.13.1 with `number-to-words` 1.2.5 ([npm](https://www.npmjs.com/package/number-to-words)), the obvious call is wrong in the quietest possible way:

```js
const n2w = require('number-to-words');
console.log(n2w.toWords(1078.10));
// "one thousand, seventy-eight"
```

No error, no warning. The cents are just gone. The package converts only the integer part, and since `1078.10 === 1078.1` in JavaScript, there's not even a trailing zero left to notice. My first invoice batch from the Node service printed amounts with no cents at all, and nobody spotted it for three days.

The fix is to split on the string, not the number:

```js
// Node 22.13.1, number-to-words 1.2.5
const n2w = require('number-to-words');

function amountInWords(amount, currency = 'US Dollars') {
  const [dollars, cents] = Number(amount).toFixed(2).split('.');
  let text = `${n2w.toWords(Number(dollars))} ${currency}`;
  if (cents !== '00') {
    text += ` and ${n2w.toWords(Number(cents))} cents`;
  }
  return text + ' only';
}

console.log(amountInWords(1078.10));
// "one thousand, seventy-eight US Dollars and ten cents only"
```

`toFixed(2)` is what preserves the cents; splitting the resulting string keeps `10` intact instead of `0.1`. For a quick sanity check when I don't want to open an editor, I paste the amount into [this browser-based tool](https://utilvy.com/en/tools/number-to-words/) and compare its output against the server's.

## Three bugs that will bite you anyway

**1. `int(amount * 100)` truncates.** This was the original 412-invoice bug, in different clothes. In Python:

```pycon
>>> repr(1078.10 * 100)
'107809.99999999999'
>>> int(1078.10 * 100)
107809
```

So the invoice said "Cents Nine" on a $1,078.10 charge. The float `1078.10` isn't exactly representable in binary, and `int()` truncates instead of rounding. `Decimal("1078.10")` from a string, or `round(1078.10 * 100)`, both give 107810.

**2. Plain decimal conversion reads like a coordinate, not money.** Calling `num2words(12340.56)` without `to='currency'` on Python 3.11.8 returns `"twelve thousand, three hundred and forty point five six"`. I've also seen rendered contracts that said "one thousand, five" for 1005 because someone hand-rolled a converter without the "and". Neither is wrong English exactly — [the conventions are genuinely messy](https://en.wikipedia.org/wiki/English_numerals) — but banks pick one and enforce it. Match their sample, don't improvise.

**3. Negative amounts produce words no bank accepts.** `num2words(-500)` returns `"minus five hundred"`; `number-to-words` gives the same `"minus five hundred"` on Node. A refund processed as a negative invoice amount will render a legal line containing "minus", which the validator rejects. Handle refunds as credit notes with their own wording — take `abs()` only if your bank's format actually expects it.

## Wrapping up

| Situation | What I use now |
| --- | --- |
| Batch generation in the billing service | Python 3.11 + num2words 0.5.14, input as `Decimal` from a string |
| Node microservice | number-to-words 1.2.5, split via `toFixed(2).split('.')` |
| One-off amount, or verifying output | [Number to Words on utilvy](https://utilvy.com/en/tools/number-to-words/) |

The pattern that fixed all 412 rejections: treat money as a string until the last moment, split dollars and cents as integers, and let a maintained library handle the grammar. The same discipline applies to the other formatting chores around invoices — dates, timezones, even the [word count limits our newsletter tool checks](https://utilvy.com/en/tools/word-counter/) — but amounts in words are the one the bank actually bounces.

More free browser-based utilities at utilvy.com.