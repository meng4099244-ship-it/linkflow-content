You've been there: an API returns a 400-line JSON blob, your editor won't highlight it, and the error is somewhere around line 217. Or a teammate pastes config into Slack and swears "it's valid JSON", while your parser disagrees.

Here are three ways to format and validate JSON — from terminal to browser — and when each one makes sense.

## Method 1: Python (already on your machine)

If you have Python installed, you already have a JSON formatter:

```python
# pretty-print a file
import json

with open("data.json") as f:
    data = json.load(f)

print(json.dumps(data, indent=2, sort_keys=True))
```

Validation comes free: if the file is broken, `json.load()` raises a `JSONDecodeError` with the exact line and column — I've found this more precise than some IDE plugins.

One-liner version for quick checks:

```bash
python3 -m json.tool data.json
```

## Method 2: Node.js

Node has no built-in pretty printer, but it's one line of code:

```bash
node -e "console.log(JSON.stringify(require('./data.json'), null, 2))"
```

Or inside a script:

```javascript
const fs = require("fs");
const raw = fs.readFileSync("data.json", "utf8");
try {
  const data = JSON.parse(raw);
  console.log(JSON.stringify(data, null, 2));
} catch (e) {
  console.error("Invalid JSON:", e.message);
}
```

## Method 3: In the browser (no install, no upload)

Sometimes you don't want to touch the terminal — you're reviewing a snippet from a chat, or you're on a machine where you'd rather not run unknown files. That's where a browser-based formatter shines.

I use [a free online JSON formatter](https://utilvy.com/en/tools/json-formatter/) for this: paste the text, it validates and pretty-prints instantly with syntax highlighting, and because everything runs client-side, the data never leaves your browser. That last part matters more than people realize — plenty of API responses contain tokens, customer data, or keys you shouldn't paste into a random server.

The same site has sibling tools I keep pinned, like [a Base64 encoder/decoder](https://utilvy.com/en/tools/base64/) and a URL encoder — handy when you're debugging JWT payloads or query strings alongside your JSON.

## Common mistakes worth knowing

**1. Trailing commas.** Valid in JavaScript objects, fatal in JSON:

```json
{
  "name": "Ada",
  "role": "engineer",   // ← this comma breaks strict JSON parsers
}
```

**2. Single quotes.** JSON strings use double quotes only. `'{"a": 1}'` is not JSON, it's a JavaScript expression that happens to look like one.

**3. Unescaped control characters.** Copy-pasted text from PDFs or logs often contains invisible characters like U+00A0. A validator will catch them; your eyes usually won't.

**4. Assuming JSON5 or JSONC.** Config files for VS Code or tsconfig allow comments (`jsonc`), but the JSON you send over an API does not. Stripping comments before validating saves an hour of confusion.

## Which method should you use?

| Situation | Best pick |
|---|---|
| Scripting, CI pipelines, large files | Python / Node |
| Quick manual check with sensitive data | [Browser-based formatter](https://utilvy.com/en/tools/json-formatter/) |
| Debugging alongside another encoding (Base64, URL) | Browser toolchain |

For day-to-day debugging I've settled on the browser for anything small and the terminal for anything batch. The exact split matters less than always validating before you ship — a broken JSON file caught locally costs nothing; caught in production, it costs a pager alert.

---

*More free browser-based developer utilities at [utilvy.com](https://utilvy.com/en/) — everything runs client-side, no signup.*