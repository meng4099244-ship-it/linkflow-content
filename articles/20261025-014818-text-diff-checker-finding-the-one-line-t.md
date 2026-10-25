# Text Diff Checker: Finding the One Line That Broke Staging in Python, Shell, and Your Browser

Staging started returning 502s at 09:40 on a Thursday. The API pods were fine: three replicas, zero restarts, p99 latency normal. The nginx config in front of them had been "synced from prod" the night before, so I pulled both files onto my laptop and ran the first command every developer reaches for.

`diff -u prod.conf staging.conf` came back with all ten lines flagged. Ten out of ten. That output told me nothing, and it was also misleading. Exactly one line had really changed.

Twenty minutes later I had the reason, and it was boring: whoever edited the staging copy did it on Windows.

## Method 1: Compare the files yourself with Python 3.11

When the answer has to come out of a script (a CI guard, a pre-commit hook, a "did the generated file drift" check), I use `difflib`. It ships with the language, so there is nothing to install.

```python
#!/usr/bin/env python3
"""Print only the lines that actually changed between two text files."""
import difflib
import sys
from pathlib import Path


def load(path):
    # Normalise CRLF on purpose, so a file saved on Windows
    # does not masquerade as a full rewrite.
    return Path(path).read_text(encoding="utf-8").replace("\r\n", "\n").split("\n")


def diff(old_path, new_path):
    old, new = load(old_path), load(new_path)
    sm = difflib.SequenceMatcher(None, old, new, autojunk=False)
    for tag, i1, i2, j1, j2 in sm.get_opcodes():
        if tag == "equal":
            continue
        print(f"@@ {tag}: old[{i1 + 1}:{i2}] -> new[{j1 + 1}:{j2}]")
        for line in old[i1:i2]:
            print("- " + line)
        for line in new[j1:j2]:
            print("+ " + line)
    print(f"similarity: {sm.ratio():.4f}")


if __name__ == "__main__":
    diff(sys.argv[1], sys.argv[2])
```

Run it (Python 3.11.1):

```
$ python3 diff_two.py prod.conf staging.conf
@@ replace: old[8:8] -> new[8:8]
-     proxy_read_timeout 30s;
+     proxy_read_timeout 5s;
similarity: 0.9091
```

There it is. `30s` became `5s`, and our slowest upstream handler takes about 7 seconds on a cold cache. That was the 502.

Two lines in that script carry their weight. I replace `\r\n` before splitting so Windows line endings stop looking like a rewrite. And `autojunk=False` disables a heuristic I do not want anywhere near config files: by default difflib treats elements that repeat often in a sequence of 200+ items as junk and refuses to match on them. Good for prose, quietly wrong for generated YAML full of repeated keys. The behavior is documented in the [difflib docs](https://docs.python.org/3/library/difflib.html).

## Method 2: Let git do the work

If the files live in a repository, `git diff` beats `diff`, mostly because of its flags:

```
$ git diff --no-index --ignore-cr-at-eol -U1 prod.conf staging.conf
diff --git a/prod.conf b/staging.conf
index b7bacd9..1719920 100644
--- a/prod.conf
+++ b/staging.conf
@@ -5,6 +5,6 @@ upstream api {
 server {
     listen 443 ssl;
     server_name api.example.com;
-    proxy_read_timeout 30s;
+    proxy_read_timeout 5s;
     proxy_pass http://api;
 }
```

`--no-index` makes git compare two plain files even when neither is tracked. `--ignore-cr-at-eol` drops the carriage return at the end of each line, which is precisely my staging problem, and `-U1` cuts the context down to one line so a 400-line file does not bury the change. On GNU diffutils 3.10 the equivalent is `diff -u --strip-trailing-cr`; the full flag list lives in the [git-diff documentation](https://git-scm.com/docs/git-diff).

## Method 3: When you only need the answer

Most days I do not want to write a script or remember which flag order matters. I keep a [free online text diff checker](https://utilvy.com/en/tools/text-diff/) open in a tab: paste the old text on the left, the new text on the right, and it highlights added and removed lines immediately. It runs entirely in the browser, which matters when the text is a config file containing internal hostnames I am not supposed to paste into a random SaaS box.

That covers the awkward cases too: two snippets copied out of a Slack thread, a JSON payload from a bug report against the one from staging, a contract sent over email as text. When I am on a machine where installing Python is not an option, [this browser-based diff tool](https://utilvy.com/en/tools/text-diff/) is the whole workflow.

## Three ways this bites back

**1. CRLF makes every line look changed.** Here is the real output, with `cat -A` so the carriage return is visible:

```
$ diff -u prod.conf staging.conf | cat -A
@@ -1,10 +1,10 @@
-upstream api {$
+upstream api {^M$
-    server 10.0.1.11:8080;$
+    server 10.0.1.11:8080;^M$
```

`diff -q` compresses that into `Files prod.conf and staging.conf differ` and stops there. Python hides the same difference for the opposite reason: `open("staging.conf").readline()` returns `'upstream api {\n'`, because text mode translates CRLF on the way in. So Python said one line, `diff` said ten lines, and both were answering different questions.

**2. `diff` exits 1 when files differ.** That status means "differences found", not "something broke":

```
$ bash -c 'set -e; diff -q prod.conf staging.conf; echo "reached next step"'
Files prod.conf and staging.conf differ
$ echo $?
1
```

"reached next step" never printed. In a CI step with `set -e`, that aborts the job before your deploy logic runs; guard it with `if ! diff -q a b; then ...`. One more caution: `diff -w` ignores all whitespace, so I reach for it on prose and JSON but never on YAML or Python, where an indentation change is the change.

**3. Diffs are order-sensitive.** Same three lines, different order:

```
$ diff -u o1.txt o2.txt
@@ -1,3 +1,3 @@
-gamma
 alpha
 beta
+gamma
```

Sorting first is the usual fix, and process substitution does it in bash. Not in `sh`:

```
$ sh -c 'diff -u <(sort o1.txt) <(sort o2.txt)'
sh: 1: Syntax error: "(" unexpected
$ echo $?
2
```

That is the default shell in Alpine-based CI images, so the same one-liner works in your terminal and dies in the pipeline. One last marker worth knowing: files with no trailing newline print `\ No newline at end of file` in the hunk, and that line alone accounts for a surprising share of "changed" reports.

## What I actually use

| Situation | What I reach for | Why |
| --- | --- | --- |
| Answer needed inside a script or CI check | Python 3.11 `difflib`, `autojunk=False` | Standard library, CRLF handled the way I decide |
| Files are in a repo, or I want flag control | `git diff --no-index --ignore-cr-at-eol -U1` | Best coverage of whitespace and line-ending noise |
| Two blobs pasted from tickets, Slack, email | The [diff utility on utilvy](https://utilvy.com/en/tools/text-diff/) | No install, nothing leaves the browser |
| Quick yes/no in a shell loop | `diff -q`, remembering exit 1 is not an error | Fastest, and harmless once you guard the status |

The 502 took me twenty minutes to explain and four seconds to fix. The interesting part was never the timeout value. It was that `diff` and Python disagreed about the same two files, and I had to know why before I could trust either answer.

More free browser-based utilities at [utilvy.com](https://utilvy.com/en/)