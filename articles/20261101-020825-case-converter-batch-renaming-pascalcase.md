Last Tuesday we agreed on a file naming rule. Components move to kebab-case, so `ChatWidget.tsx` becomes `chat-widget.tsx`, and the old PascalCase files disappear.

The trigger was mundane. Two teammates on macOS kept hitting broken checkouts because `DateRange.tsx` and `daterange.tsx` are two files in CI on Linux, but a single file on their laptops. Same repo, two views of it.

Eleven components. Sounds like one slow afternoon.

I spent most of that afternoon on three problems that have nothing to do with converting case: imports pointing at old paths, Git refusing to notice half my renames, and `XMLHttpRequestPanel.tsx` coming out of my first script as `xmlhttprequestpanel.tsx`.

Here is what worked, in Python 3.11.1 and Node v22.13.1, plus the shortcut for small batches.

## Method 1: a Python 3.11.1 script that keeps Git history

Plain `mv` loses the fact that a rename happened. `git mv` keeps it. My script runs `git mv` per file, then rewrites every import that points at an old name.

```python
import re
import subprocess
import sys
from pathlib import Path

SPLITTER = re.compile(r"(?<=[a-z0-9])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])")

# Digits next to acronyms: the splitter cannot guess these.
OVERRIDES = {
    "UserAvatar2FA": "user-avatar-2fa",
    "OAuth2Callback": "oauth2-callback",
}

root = Path(sys.argv[1])
plan = {}
for src in sorted(root.glob("*.tsx")):
    target = root / f"{OVERRIDES.get(src.stem) or '-'.join(p.lower() for p in SPLITTER.split(src.stem))}.tsx"
    if target.name != src.name:
        plan[src] = target

for src, dst in plan.items():
    subprocess.run(["git", "mv", str(src), str(dst)], check=True)
    print(f"{src.name:26} -> {dst.name}")

edited = 0
for path in root.glob("*.tsx"):
    text = path.read_text(encoding="utf-8")
    for old, new in ((s.stem, d.stem) for s, d in plan.items()):
        text = re.sub(rf"([./]){old}(?=\")", rf"\g<1>{new}", text)
    if text != path.read_text(encoding="utf-8"):
        path.write_text(text, encoding="utf-8")
        edited += 1

print(f"\nrenamed {len(plan)} files, rewrote imports in {edited} files")
```

Run it against the component folder, then check that nothing still refers to the old paths.

```bash
$ python3 rename_components.py src
ChatWidget.tsx             -> chat-widget.tsx
DateRange.tsx              -> date-range.tsx
Grid2Col.tsx               -> grid2-col.tsx
HTTPClientErrors.tsx       -> http-client-errors.tsx
InvoiceLineItemV2.tsx      -> invoice-line-item-v2.tsx
OAuth2Callback.tsx         -> oauth2-callback.tsx
PDFViewer.tsx              -> pdf-viewer.tsx
SideNavBar.tsx             -> side-nav-bar.tsx
UserAvatar2FA.tsx          -> user-avatar-2fa.tsx
Utils.tsx                  -> utils.tsx
XMLHttpRequestPanel.tsx    -> xml-http-request-panel.tsx

renamed 11 files, rewrote imports in 11 files

$ grep -rn 'from "./[A-Z]' src/*.tsx || echo "(none)"
(none)
```

Every import got rewritten, so the grep comes back clean, and `git mv` keeps the batch as 11 renames rather than 11 deletions plus 11 untracked files.

The splitter does the real work. `(?<=[a-z0-9])(?=[A-Z])` cuts between a lowercase letter or digit and an uppercase one. `(?<=[A-Z])(?=[A-Z][a-z])` cuts before the last capital of an acronym run, which is what turns `XMLHttp` into `XML-Http`. Both are zero-width assertions, documented in the [Python `re` reference](https://docs.python.org/3/library/re.html).

Two names refuse to cooperate. `UserAvatar2FA` splits into `user-avatar2-fa` and `OAuth2Callback` into `o-auth2-callback`, because the digit rule cannot tell whether a digit attaches forward (`2FA`) or backward (`OAuth2`). I gave up on solving that inside the regex and pinned both names in `OVERRIDES`.

## Method 2: the same batch rename in Node v22.13.1

No Python on the machine? The logic ports over directly.

```js
import { execFileSync } from "node:child_process";
import { readdirSync, readFileSync, writeFileSync } from "node:fs";
import path from "node:path";

const root = process.argv[2] ?? "src";
const OVERRIDES = new Map([
  ["UserAvatar2FA", "user-avatar-2fa"],
  ["OAuth2Callback", "oauth2-callback"],
]);
const SPLIT = /(?<=[a-z0-9])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])/;
const toKebab = (stem) =>
  OVERRIDES.get(stem) ?? stem.split(SPLIT).map((p) => p.toLowerCase()).join("-");

const candidates = readdirSync(root, { encoding: "utf8" })
  .filter((name) => name.endsWith(".tsx"))
  .sort();

const mapping = new Map();
for (const name of candidates) {
  const stem = name.slice(0, -4);
  const target = `${toKebab(stem)}.tsx`;
  if (target === name) continue;
  execFileSync("git", ["mv", path.join(root, name), path.join(root, target)]);
  mapping.set(stem, target.slice(0, -4));
  console.log(`${name.padEnd(26)} -> ${target}`);
}

let edited = 0;
for (const name of readdirSync(root, { encoding: "utf8" })) {
  if (!name.endsWith(".tsx")) continue;
  const file = path.join(root, name);
  const before = readFileSync(file, "utf8");
  let after = before;
  for (const [from, to] of mapping) {
    after = after.replaceAll(new RegExp(`([./])${from}(?=")`, "g"), `$1${to}`);
  }
  if (after !== before) {
    writeFileSync(file, after);
    edited += 1;
  }
}

console.log(`\nrenamed ${mapping.size} files, rewrote imports in ${edited} files`);
```

`String.prototype.split` accepts lookbehind in V8, so both assertions work unchanged ([MDN covers the lookbehind syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Lookbehind_assertion)). The output matched Python line for line: 11 renames, 11 files rewritten, zero leftover imports.

Spot-check the odd names against the [Utilvy case converter](https://utilvy.com/en/tools/case-converter/) before trusting either script. If it disagrees with your expectation, the script is usually the wrong one.

## Method 3: convert names in a browser tab

Two or three files? Skip the script entirely. Paste the component names into [a free case converter](https://utilvy.com/en/tools/case-converter/), pick kebab-case, copy the result, and pair them up by hand.

It handles whole lists in one paste, so ten names are no slower than one. It runs locally in the tab, which matters on an unreleased branch.

Above a handful of files, go back to a script. The browser gives you the new names, but someone still has to rewrite eleven import blocks, and a web page cannot do that part.

## Three things that cost me the afternoon

**Straight lowercasing destroys acronym boundaries**

My very first line was this:

```python
>>> "XMLHttpRequestPanel".lower()
'xmlhttprequestpanel'
```

Same for `HTTPClientErrors` turning into `httpclienterrors` and `PDFViewer` into `pdfviewer`. Every word boundary vanishes, and there is no way back from a flattened string. [This browser-based tool](https://utilvy.com/en/tools/case-converter/) keeps those boundaries intact, which makes it a decent oracle when you want to argue with your own regex. Split first, lowercase second.

**Renaming the file is only half the job, and the failure is silent**

My first rewrite used a guard I thought was clever:

```python
text = re.sub(rf'(?<!/){old}(?=")', new, text)
```

Every import path in a TypeScript file starts with `./`. The guard rejected all of them, and the script printed:

```
renamed 11 files, rewrote imports in 0 files
```

Eleven renames, zero rewrites, exit code 0. Nothing broke visibly until the next build. Use a capture group instead of a lookbehind, then grep for `from "./[A-Z]` before you commit.

**Case-insensitive filesystems hide case-only renames**

`Utils.tsx` to `utils.tsx` changes nothing but case. Fine on Linux, invisible on macOS and Windows, where Git is configured with `core.ignorecase`. Here is the reproduction on an ext4 disk:

```bash
$ git config core.ignorecase true
$ mv Utils.tsx utils.tsx
$ git status --short
 D Utils.tsx
```

Only a deletion. `git ls-files` still lists `Utils.tsx`, while `import Utils from "./utils"` points at a file Git no longer tracks. Flip the flag and Git sees both halves:

```bash
$ git config core.ignorecase false
$ git status --short
 D Utils.tsx
?? utils.tsx
```

On macOS, step through a temp name: `git mv Utils.tsx tmp-utils.tsx && git mv tmp-utils.tsx utils.tsx`. Or do the whole batch in CI where the filesystem takes casing seriously.

## Pick your approach

| Scenario | Approach | Handled 11 files in |
| --- | --- | --- |
| Whole repo to migrate | Python 3.11.1 script + `git mv` | one run, imports rewritten |
| Python unavailable | Node v22.13.1 equivalent | one run, identical output |
| Two or three names | Browser-based kebab conversion | under a minute |

Verify in this order: grep for leftover PascalCase imports, then read `git status` on a case-sensitive filesystem, then commit.

More free browser-based utilities at utilvy.com