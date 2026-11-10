# CSV to JSON Converter: When One Quoted Comma Broke 4,112 Supplier Records

Every quarter I pull two supplier price lists into our internal pricing API. Supplier A exports comma-separated CSV from an old Windows ERP. Supplier B sends semicolon-separated files with German decimal commas and a byte-order mark nobody warned me about. The API only speaks JSON.

For three quarters my sync script was fourteen lines of Node. It worked, which is the worst thing a parser can do. Then supplier A shipped the full catalog instead of the usual 400-row delta, and 4,112 of 12,417 rows landed with the wrong number of fields. The API accepted them all, because nothing in JSON complains when `amount` becomes a string holding half a company name.

This is the version of that script I run now, plus two other routes I keep around when I'm debugging at 11 pm.

## Method 1: Python 3.11.1 standard library only

The [csv module](https://docs.python.org/3/library/csv.html) has shipped with Python since 2.3 and handles [RFC 4180](https://www.rfc-editor.org/rfc/rfc4180) quoting, but only once you tell it which delimiter you have and refuse to continue when a row doesn't line up. My old version skipped both steps. It did `line.split(',')`, which on `1001,"Acme, Inc.",1200.00` returns four cells:

```
[ '1001', '"Acme', ' Inc."', '1200.00' ]
```

```python
# python3.11 csv_to_json.py feed.csv > feed.json
import csv, json, sys

def rows(path):
    with open(path, newline="", encoding="utf-8-sig") as f:
        sample = f.read(4096)
        f.seek(0)
        try:
            dialect = csv.Sniffer().sniff(sample, delimiters=",;\t")
        except csv.Error:
            dialect = csv.excel
        for lineno, row in enumerate(csv.DictReader(f, dialect=dialect), start=2):
            if None in row:                       # more cells than headers
                raise SystemExit(f"line {lineno}: extra fields {row[None]}")
            if None in row.values():              # fewer cells than headers
                raise SystemExit(f"line {lineno}: missing field")
            yield row

data = list(rows(sys.argv[1]))
json.dump(data, sys.stdout, ensure_ascii=False, indent=2)
print(f"{len(data)} rows ok", file=sys.stderr)
```

Three details carry the weight here. `newline=""` stops Python from translating `\r\n` inside quoted fields, which is how addresses with line breaks stay one value. `encoding="utf-8-sig"` eats the BOM that supplier B's export tool writes. And the `None in row` check is the only thing standing between a misaligned row and a silent bad price.

On the real feed, Python 3.11.1 on Ubuntu 22.04:

```
$ python3 csv_to_json.py feed.csv > feed.json
12417 rows ok
```

That's 0.10 seconds of wall clock, and the awkward record comes through intact:

```json
{"sku": "SKU-10097", "product": "Bracket, \"heavy\" 97", "supplier": "Acme Parts", "price": "33,60", "stock": "64"}
```

Note that `price` is the string `"33,60"`, not a number. That's deliberate. Parsing decimal commas is a business decision, and I'd rather do it in one place with `Decimal` than let a library guess.

## Method 2: pandas 3.0.0, if you already have it loaded

When the file is already heading into analysis, one call does the same job. The default arguments do not.

```python
import pandas as pd

df = pd.read_csv(
    "feed.csv",
    sep=None,               # guess , ; or tab
    engine="python",        # the C engine cannot guess a separator
    dtype=str,              # keep 01234 and 34,10 exactly as written
    keep_default_na=False,  # empty cell stays "", not NaN
    encoding="utf-8-sig",   # strip the BOM
)
df.to_json("feed.json", orient="records", force_ascii=False, indent=2)
```

Leave the arguments off and you get this on line 98 of the same file:

```
pandas.errors.ParserError: Error tokenizing data. C error: Expected 2 fields in line 98, saw 3
```

Line 98 is the first row with a quoted comma in the product name. pandas was reading the file as comma-delimited and finding two columns, so the third comma looked like a broken row. `sep=None` with `engine="python"` sniffs semicolons correctly and returns shape `(12417, 5)` with the same JSON as Method 1, byte for byte, for the record above.

## Method 3: paste it into a browser tool

Half my CSV problems are one-off: a colleague's export, a log dump, a file I need to eyeball before trusting it. For those I use [a browser-based CSV to JSON converter](https://utilvy.com/en/tools/csv-json-converter/). Paste the file, pick the delimiter (comma, semicolon or tab), leave "First row is header" checked, and the JSON appears on the right with copy and download buttons.

The parser follows RFC 4180, so quoted delimiters, doubled quotes and line breaks inside a field behave the way they should. Since it runs client-side, customer exports and invoice lists never leave the machine, which matters when the file is somebody else's data and I have no business uploading it to a random server.

I also use [the same tool in reverse](https://utilvy.com/en/tools/csv-json-converter/) when an API returns an array of objects and finance wants a spreadsheet. Nested objects and arrays are written as compact JSON inside the cell rather than dropped, and keys missing from a row become empty cells.

For anything that runs on a schedule, keep the script. A web page can't be put in cron, and [this converter](https://utilvy.com/en/tools/csv-json-converter/) is a debugging and one-off instrument, not a pipeline stage.

## Three traps that don't throw

**1. The BOM that becomes part of your column name.** Supplier B's file starts with `EF BB BF`. Read it as plain UTF-8 and the header is not `sku`:

```python
>>> list(csv.DictReader(open("bom.csv", newline="", encoding="utf-8")).fieldnames)
['\ufeffid', 'name']
>>> row["id"]
KeyError: 'id'
```

pandas does the same thing: `df.columns[0]` is `'\ufeffsku'`, and `df["sku"]` raises `KeyError: 'sku'` on a frame that visibly has a sku column. Fix: `utf-8-sig`, both places.

**2. Leading zeros gone before you notice.** A ZIP code of `01234` read with default dtypes becomes `1234`, dtype `int64`. SKUs, phone numbers and account IDs have the same problem, and the damage shows up two systems downstream. `dtype=str` in pandas, or just don't cast in the stdlib script.

**3. An unclosed quote that eats the rest of the file.** A three-line file with one stray `"` at the start of a field:

```python
>>> list(csv.reader(open("unclosed.csv", newline="")))
[['id', 'name'], ['1', 'Alice\n2,Bob,extra\n']]
```

Two rows instead of three. No exception, no warning. The parser treated everything after the quote as one long value. This is why I compare `len(data)` against `wc -l` minus one, and why the `None in row` guard exists: extra cells land under the key `None` (`{'id': '1003', 'customer': 'Acme', 'amount': ' Inc.', None: ['42']}`), which is easy to miss unless you check for it.

| Route | Handles quoted commas | Keeps BOM out | Keeps leading zeros | Good for |
| --- | --- | --- | --- | --- |
| Python `csv` + `json` | yes, with the right dialect | yes, `utf-8-sig` | yes, no casting by default | scheduled syncs, strict row checks |
| pandas 3.0.0 | yes, `sep=None` + python engine | yes, `encoding="utf-8-sig"` | yes, `dtype=str` | files already going into analysis |
| Browser tool | yes, RFC 4180 | yes, pasted text is clean | yes, values stay strings | one-off files, quick inspection |

The rule I ended up with: a parser that silently produces JSON is not a parser I trust. Count the rows, assert the field count, and compare one record against the source file by hand the first time. The sync job now fails the build when the row count moves more than 5% without a matching change in the supplier's header, which caught a truncated export in September before it reached pricing.

More free browser-based utilities at utilvy.com