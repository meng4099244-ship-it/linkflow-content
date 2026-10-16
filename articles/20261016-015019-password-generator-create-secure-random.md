I needed a fresh database password for a staging service, but the usual "pick a phrase and mutate a few letters" routine felt wrong. Humans repeat patterns. We capitalize the first character, put `!` at the end, and reuse the same substitutions across accounts.

For this job I wanted 20 characters, at least one lowercase letter, uppercase letter, digit, and symbol, plus randomness from the operating system rather than a predictable pseudo-random seed. Here are the three approaches I actually use, depending on where I am working.

Before generating anything, I inspect the target form. One payroll portal I dealt with capped passwords at 16 characters and rejected `_`, even though its reset page listed that symbol as valid. Writing down the exact length and allowed set first saves a surprising amount of regenerate-and-retry work.

## Method 1: Generate and validate a password in Python 3.11.1

Python’s [`secrets` documentation](https://docs.python.org/3.11/library/secrets.html) explicitly recommends the module for passwords and authentication tokens. It draws from the strongest randomness source supplied by the operating system. That is the key difference from `random`, which is built for simulation.

Save this as `password_generator.py`:

```python
#!/usr/bin/env python3
"""Generate a password with at least one character from each class.
Tested with Python 3.11.1.
"""

import argparse
import secrets
import string

SYMBOLS = "!@#$%^&*_-+="
GROUPS = (string.ascii_lowercase, string.ascii_uppercase,
          string.digits, SYMBOLS)
ALPHABET = "".join(GROUPS)


def secure_shuffle(items: list[str]) -> None:
    """Shuffle in place with Fisher-Yates and a cryptographic RNG."""
    for i in range(len(items) - 1, 0, -1):
        j = secrets.randbelow(i + 1)
        items[i], items[j] = items[j], items[i]


def generate_password(length: int) -> str:
    if length < len(GROUPS):
        raise ValueError(
            "length must be at least 4 to include every character class"
        )

    chars = [secrets.choice(group) for group in GROUPS]
    chars.extend(
        secrets.choice(ALPHABET)
        for _ in range(length - len(chars))
    )
    secure_shuffle(chars)
    return "".join(chars)


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("length", nargs="?", type=int, default=20)
    args = parser.parse_args()
    print(generate_password(args.length))


if __name__ == "__main__":
    main()
```

Run it with the desired length:

```text
$ python3.11 password_generator.py 20
U7mEnc8SDgvJM#@QYZY1
```

Your result will differ; that is expected. The script first takes one character from each required group, fills the remaining positions from the full alphabet, then shuffles with `secrets.randbelow()`. Without the final shuffle, the first four positions would reveal the character-class order.

## Method 2: Use Node.js 22.13.1 and `node:crypto`

On JavaScript projects I use `crypto.randomInt()` rather than `Math.random()`. The structure is deliberately similar to the Python version, so it is easy to review during a security check.

Save this as `password_generator.js`:

```javascript
#!/usr/bin/env node
const { randomInt } = require("node:crypto");

const groups = [
  "abcdefghijklmnopqrstuvwxyz",
  "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "0123456789",
  "!@#$%^&*_-+=",
];
const alphabet = groups.join("");

function pick(chars) {
  return chars[randomInt(chars.length)];
}

function secureShuffle(items) {
  for (let i = items.length - 1; i > 0; i--) {
    const j = randomInt(i + 1);
    [items[i], items[j]] = [items[j], items[i]];
  }
}

function generatePassword(length = 20) {
  if (!Number.isInteger(length) || length < groups.length) {
    throw new RangeError("length must be an integer of at least 4");
  }

  const chars = groups.map(pick);
  while (chars.length < length) chars.push(pick(alphabet));
  secureShuffle(chars);
  return chars.join("");
}

const length = process.argv[2] === undefined ? 20 : Number(process.argv[2]);
console.log(generatePassword(length));
```

My 24-character test produced:

```text
$ node password_generator.js 24
GbQ$uAIt*#mPj-eR46@8!iUm
```

In browser code, the equivalent primitive is `crypto.getRandomValues()`. MDN describes it as a source of [cryptographically strong random values](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues). Do not swap it for `Math.random()` just because the latter needs fewer keystrokes.

## Method 3: Generate one in the browser

When I am on a locked-down machine, creating and reviewing a script is more work than the password itself. I open a [free online password generator](https://utilvy.com/en/tools/password-generator/), choose the length and character groups, generate the password, and copy it directly into my password manager.

The page runs client-side, so the generated value is not sent away for processing. I still avoid screenshots, chat messages, and clipboard-history tools. For a database credential, I usually set 24 characters and enable all four groups; for a site with a restrictive policy, I match its allowed symbols instead of repeatedly regenerating at random.

I paste the result into the password and confirmation fields before saving it in the vault. That small sequence catches trimmed spaces, accidental double-click copies, and forms that silently truncate long input. I also clear the clipboard after the secret is stored.

The [browser-based password tool](https://utilvy.com/en/tools/password-generator/) is also handy for showing a teammate why length and randomness matter more than turning `password` into `P@ssw0rd!`. It exposes the choices without making anyone install Python or Node.

## Three mistakes I have seen in real scripts

### 1. Using a repeatable random generator

A seeded generator is useful in tests, but disastrous for credentials. This Python 3.11.1 example produces the same "password" twice:

```text
$ python3.11 repeatable.py
OhbVrpoiVgRV
OhbVrpoiVgRV
```

The bug is not subtle: both runs call `random.seed(42)`. Even without an explicit seed, Python’s `random` and JavaScript’s `Math.random()` are not designed to resist prediction. Use `secrets`, `node:crypto`, or Web Crypto.

### 2. Asking for a length that cannot satisfy the policy

Four required groups need at least four positions. I prefer a hard failure to silently dropping a requirement:

```text
$ python3.11 password_generator.py 3
ValueError: length must be at least 4 to include every character class
```

There is a second trap here. Picking one character from each group and stopping at eight characters technically passes many forms, but it leaves far less search space than a 20- or 24-character result. Policy compliance is not the same as strength.

### 3. Pasting symbols into a shell without quoting

A generated password can be strong and still be mangled during deployment. Bash expands `$HOME` and treats `&` as a control operator:

```text
$ printf '%s\n' K$HOME&9q
bash: line 1: 9q: command not found
K/root
```

Quoting preserves the actual value:

```text
$ printf '%s\n' 'K$HOME&9q'
K$HOME&9q
```

For production, I put secrets into the platform’s secret store rather than a shell command or `.env` file committed to Git. The [local password generator](https://utilvy.com/en/tools/password-generator/) solves generation; safe storage and rotation are still my responsibility.

## Quick comparison

| Method | Best use | Secure randomness | Setup |
|---|---|---:|---:|
| Python 3.11 `secrets` | Automation and backend scripts | Yes | Python only |
| Node.js 22 `node:crypto` | JavaScript projects and CLIs | Yes | Node only |
| Utilvy password generator | One-off generation on restricted machines | Browser crypto, client-side | None |

For repeatable automation, I keep the Python or Node script in a private operations repository. For a one-off credential, the browser route is faster. In both cases, I generate a unique password, store it in a password manager or secret vault, and never try to memorize it.

More free browser-based utilities at [utilvy.com](https://utilvy.com/en/).