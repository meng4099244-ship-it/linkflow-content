# Remove Duplicate Lines From a 48,000-Line CSV Without Losing the Original Order

Last month I had to merge two exports from our billing system into one import file. The first export had 31,207 customer email addresses, the second had 17,706, and a few thousand lines overlapped. The CRM import endpoint rejects the whole file if it contains a single repeated email, so deduplication had to happen before upload.

There was a second constraint that tripped me up: row order drove import priority. Whichever line appeared first had to stay. So `sort -u`, the command half of my brain reached for, was off the table from the start. Here are three approaches that do work, tested on Ubuntu 22.04 with Python 3.11.1 and GNU awk 5.1.1.

## Method 1: Python, streaming with a set

The obvious one-liner is `list(set(lines))`. I've written it myself under deadline pressure. Don't. A set iterates in arbitrary hash order, so the output is a shuffled file, and the CRM import would run in whatever order the hash table happened to land on.

What you want is a `seen` set plus a counter:

```python
# dedupe.py -- Python 3.11.1, no third-party dependencies
import sys

def main(path: str) -> int:
    seen: set[str] = set()
    kept, dropped = 0, 0
    with open(path, encoding="utf-8", newline="") as f:
        for line in f:
            line = line.rstrip("\n")
            if line in seen:
                dropped += 1
                continue
            seen.add(line)
            print(line)
            kept += 1
    print(f"kept {kept}, dropped {dropped}", file=sys.stderr)
    return 0

if __name__ == "__main__":
    sys.exit(main(sys.argv[1]))
```

Run it like this:

```
$ python3 dedupe.py emails_merged.csv > emails_clean.csv
kept 45710, dropped 3203
```

Reading line by line means the file never sits in memory as one blob. The `seen` set held about 45,000 short strings, roughly 6 MB on my machine. I've pushed a 2 GB access log through the same shape of script by printing to `stdout` and redirecting; it took a few minutes but finished. Membership checks against a set are O(1) on average, which is why this stays fast as the file grows. The same pattern exists in JavaScript as a [`Set`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set), so porting the script is a ten-minute job if your pipeline is Node-based.

## Method 2: awk, one line

When I just need the result and don't want to write a file, GNU awk does it in a single command:

```
$ awk '!seen[$0]++' emails_merged.csv > emails_clean.awk.csv
```

`$0` is the entire line. The expression `seen[$0]++` returns 0 the first time a line shows up (falsy, so awk prints it) and counts upward after that. First occurrence wins, original order preserved. Same semantics as the Python script. The gawk manual has a whole chapter on how arrays and their uninitialized elements behave, worth a read if this idiom feels too magical.

One honest caveat: awk accumulates every line it has seen in memory, same as the Python set. For a 40 GB file on a 4 GB box neither will end well, and you'd reach for `sort` with a stable flag or a disk-based approach. For anything under a few hundred megabytes it's fine.

## Method 3: paste it into a browser tool

Sometimes the input arrives through Slack as a pasted list of a few thousand lines, from a teammate on Windows who has no intention of opening a terminal. For those, I use [a free online duplicate line remover](https://utilvy.com/en/tools/remove-duplicate-lines/). Paste, click, copy the result back out.

[This browser-based tool](https://utilvy.com/en/tools/remove-duplicate-lines/) deletes duplicate lines while keeping the first occurrence and the original order, which is the same semantics as the two methods above. The processing happens client-side in the page, so the paste never leaves the machine. With customer email addresses involved, that's the part I actually care about, and it's faster than explaining to a colleague how to run awk in PowerShell.

## Three pitfalls that cost me time

### 1. CRLF line endings make duplicates invisible

The second export came out of a Windows service. Every line ended in `\r\n`, and my `rstrip("\n")` left the carriage return in place. On screen the lines looked identical; in memory they weren't:

```python
>>> open("second_export.csv", newline="").readline()
'maria@example.com\r\n'
>>> "maria@example.com\r" == "maria@example.com"
False
```

First run reported `dropped 3203`. After I changed the strip to `line.rstrip("\r\n")`, the same file reported `dropped 3847`. Those extra 644 "duplicates" were the same addresses, invisible until the repr showed the `\r`.

### 2. A UTF-8 BOM protects the first line from matching

The export was saved as "UTF-8 with BOM", which many spreadsheet tools do by default. The very first line carries a three-byte prefix that repr makes easy to spot:

```python
>>> open("emails.csv", encoding="utf-8").readline()
'\ufeffalice@example.com\n'
```

`'\ufeffalice@example.com'` never equals `'alice@example.com'`, so if that address appears again later, it survives as a duplicate. Opening the file with `encoding="utf-8-sig"` strips the BOM on read and the problem disappears. I now check the first byte of any vendor CSV before writing the script.

### 3. `sort -u` reorders rows even when you think it won't

A teammate "fixed" one of these files with `sort -u` in a cron job. Dedup worked. The import priority did not, because the output came back sorted, not in original order. Locale changes the collateral damage, too:

```
$ printf 'Zebra\napple\nBanana\n' | sort
apple
Banana
Zebra
$ printf 'Zebra\napple\nBanana\n' | LC_ALL=C sort
Banana
Zebra
apple
```

Same input, same machine, two different outputs depending on `LC_ALL`. The GNU [coreutils sort documentation](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html) covers the field and locale behavior in detail. When order matters, dedupe with awk or the streaming script instead, and save `sort -u` for cases where sorted output is what you wanted anyway.

## Which one to use

| Method | Command | Keeps first-occurrence order | Where it runs |
| --- | --- | --- | --- |
| Python set | `python3 dedupe.py in.csv > out.csv` | Yes | Python 3.11, streaming |
| awk | `awk '!seen[$0]++' in.csv` | Yes | GNU awk 5.1, one line |
| `sort -u` | `sort -u in.csv` | No | coreutils, order destroyed |
| Browser tool | paste into [the remove-duplicate-lines utility](https://utilvy.com/en/tools/remove-duplicate-lines/) | Yes | utilvy.com, nothing installed |

For files in the gigabytes I use the Python script. For quick shell work, awk. For a pasted list from a colleague, the browser tool. All three preserve the first occurrence, and that's usually the only thing the downstream system actually checks.

More free browser-based utilities at [utilvy.com](https://utilvy.com).