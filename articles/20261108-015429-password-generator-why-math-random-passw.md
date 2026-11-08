# Password Generator: Why Math.random() Passwords Get Cracked (and What to Use Instead)

Two years ago I shipped a password reset flow where the temporary password came from `Math.random().toString(36).slice(2, 14)`. Twelve characters, mixed letters and digits. It looked random. Our pen tester asked for two consecutive reset emails, pasted both tokens into a script, and reconstructed the generator's internal state in under a minute. Every "random" password our server could produce was now a short list for him.

That day taught me the difference between *unpredictable-looking* and *unpredictable*. I have rewritten that generator three times since, for three different employers, and the bug keeps coming back in code review because `Math.random()` is one line long and looks completely innocent. Here is how I generate passwords now, in Python, in Node, and when I am too lazy to write code at all.

## Why Math.random() fails at this

`Math.random()` is a pseudorandom number generator, not a secure one. V8 (Chrome and Node.js) uses an algorithm called xorshift128+ that keeps 128 bits of internal state. The state evolves deterministically from one call to the next, so a few consecutive outputs are enough to solve for it offline. This is not a theoretical concern; the technique is well documented and the tools to do it are public. The MDN page for [Math.random()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random) says it outright: it should not be used for anything security-related.

The fix is always the same idea: read randomness from the operating system's CSPRNG instead of a PRNG with recoverable state.

## Method 1: Python's secrets module (Python 3.11.1)

Python makes this almost boring. The `secrets` module was added exactly for this job, and `secrets.choice()` picks a character uniformly.

```python
# Python 3.11.1
import math
import secrets
import string

ALPHABET = string.ascii_letters + string.digits  # 62 symbols

def gen_password(length: int = 16) -> str:
    return "".join(secrets.choice(ALPHABET) for _ in range(length))

for _ in range(3):
    print(gen_password(16))

print("entropy:", round(16 * math.log2(len(ALPHABET)), 1), "bits")
```

Output from my machine:

```text
p65Q27UkKzPhpNwe
qUPeJyfHVhF5333I
Br13QDK4tsv2zeYl
entropy: 95.3 bits
```

Ninety-five bits is well beyond brute-force reach. The [secrets documentation](https://docs.python.org/3/library/secrets.html) recommends the same pattern for tokens, reset links, and anything else an attacker might guess.

## Method 2: Node.js crypto.randomInt (Node 22.13.1)

In Node, reach for `node:crypto`. I use `crypto.randomInt()` per character. It performs rejection sampling internally, so every index is equally likely, which matters more than most people think (see pitfall 2).

```js
// Node 22.13.1 (CommonJS)
const crypto = require("node:crypto");
const ALPHABET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

function genPassword(length = 16) {
  return Array.from({ length }, () =>
    ALPHABET[crypto.randomInt(ALPHABET.length)]
  ).join("");
}

for (let i = 0; i < 3; i++) console.log(genPassword(16));
```

Output:

```text
qC3nGqZj5HZOwvkw
PCh5EqjcXuVvUokb
9eLoGcWlrzZX3nKB
```

In the browser the equivalent is `crypto.getRandomValues()`, which fills a typed array from the same class of OS entropy. Never reach for `Math.random()` as a fallback there either.

## Method 3: when I just need one password, right now

Half the time I am not building a flow. I am setting up a database user or rotating an API credential, and I want one strong password in the next five seconds. Writing a script for that is overhead, and pasting something from a code snippet on the internet is how the modulo-bias bug from pitfall 1 survives into 2026. This is where I open [a free online password generator](https://utilvy.com/en/tools/password-generator/), set length and character sets, and copy the result. [This browser-based tool](https://utilvy.com/en/tools/password-generator/) generates everything locally in JavaScript, so the password never leaves my machine, which is the whole point of a password generator in the first place. If you are auditing your own habits, the [password generator](https://utilvy.com/en/tools/password-generator/) is also a fast way to compare what a real 20-character password looks like against whatever your old script produced.

## Three pitfalls that quietly weaken "random" passwords

**1. Modulo bias when mapping bytes to characters.** A common shortcut is `charset[randomBytes(1)[0] % charset.length]`. With 62 characters, `256 % 62 = 8`, so the first 8 symbols get hit 5 times per 256 bytes while the rest get 4. I measured it with 620,000 single bytes in Python 3.11.1: index 0 landed 1.957% of the time versus 1.613% expected. That is roughly 21% more likely than its neighbors, on every single character of every password. `secrets.choice()` and `crypto.randomInt()` both avoid this; hand-rolled `%` does not.

**2. Slicing Base64 strings.** `crypto.randomBytes(12).toString("base64")` gives 16 characters of 96-bit entropy, which sounds great until you look at it:

```text
+KQcdtawbjaxhglg   <- full string
+KQcdtawbj         <- slice(0, 10): about 60 bits left
```

Cutting to 10 characters throws away a third of the entropy you paid for. The full string also carries `+` and `/`, which break URLs and some CSV imports, and a 12-byte input always ends in exactly one `=` pad. If someone pastes `os.urandom(12)` output around, they probably shipped padding and bias together.

**3. Requiring "complexity rules" on a strong generator.** Forcing a digit, a symbol, and an uppercase letter into a 12-character password shrinks the effective search space and trains users to predict where the symbol sits (usually the end). A uniform 16-character alphanumeric password at 95.3 bits beats a rule-mangled 12-character one at roughly 71 bits. Length beats punctuation.

## Summary

| Method | Entropy (16 chars) | Best for |
|---|---|---|
| Python `secrets.choice()` | ~95 bits | Server-side token generation |
| Node `crypto.randomInt()` | ~95 bits | Node services, unbiased per-char picks |
| `Math.random()` tricks | Recoverable state | Nothing. Stop. |

The pattern is simple: OS entropy in, uniform selection, no truncation games. Everything else is decoration.

More free browser-based utilities at [utilvy.com](https://utilvy.com/en/).