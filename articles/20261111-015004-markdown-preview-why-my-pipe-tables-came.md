# Markdown Preview: Why My Pipe Tables Came Out as Plain Text (and How I Ship Clean HTML Now)

Last month I had to publish a changelog through our company CMS. It accepts raw HTML only. My source was Markdown, written against a GitHub wiki where every table, fence and blockquote rendered perfectly.

Pasted into the CMS, the tables collapsed into something like this:

```text
| Feature | Supported |
| --- | --- |
| Tables | Yes |
```

No error message. No warning. Just pipes sitting in a paragraph, live on production for about twenty minutes before a colleague asked why the changelog looked like ASCII art.

The annoying part: our CMS does have a preview tab. I checked it. The preview rendered the raw HTML I had pasted, which was *correct* HTML, for the wrong content. The failure was upstream, in the conversion I never really did. My editor's Markdown plugin showed a nice table, so I assumed the file contained something a browser could use. It didn't. GitHub had been doing the conversion for me all along.

That afternoon I stopped pasting blind. I built a small render-then-check pipeline on my machine, and later found a faster shortcut for one-off posts. Here's what worked, and the three mistakes I made on the way.

## Method 1: Python with the `markdown` package

My first attempt used Python 3.11.1 with markdown 3.10.1 (`pip install markdown`). The core call is one function:

```python
# md2html.py — Python 3.11.1, markdown 3.10.1
import sys
import markdown

src = open(sys.argv[1], encoding="utf-8").read()

html = markdown.markdown(
    src,
    extensions=["tables", "fenced_code", "nl2br"],
    output_format="html",
)

print(html)
```

Run it with `python md2html.py changelog.md`. The `extensions` list is the entire story of my broken table, and I'll come back to that in the pitfalls section. With it in place, a pipe table comes out as real markup:

```html
<table>
<thead>
<tr>
<th>Feature</th>
<th>Supported</th>
</tr>
</thead>
```

Two smaller notes from a week of using this daily. First, always open the source with an explicit encoding. I skipped it once on a Windows box and a curly apostrophe in a release note killed the script with `UnicodeDecodeError: 'charmap' codec can't decode byte 0x9d in position 214`. Second, if your output feeds a template that already wraps content in `<html>` and `<body>`, leave `output_format` alone; the default produces a plain fragment, which is what CMSes want.

The [extension list in the official docs](https://python-markdown.github.io/extensions/) is worth a skim. `tables` handles pipe syntax, `fenced_code` handles triple-backtick blocks, and `nl2br` controls line-break behaviour that GitHub treats differently from raw Markdown.

## Method 2: Node with `marked` plus a sanitizer

For the JS side of our build I used Node 22.13.1 with marked 12.0.2 and sanitize-html 2.13.0. marked renders GFM tables out of the box, so the script is shorter:

```js
// md2html.mjs — Node 22.13.1, marked 12.0.2, sanitize-html 2.13.0
import { readFileSync } from "node:fs";
import { marked } from "marked";
import sanitizeHtml from "sanitize-html";

const md = readFileSync(process.argv[2], "utf8");
const raw = marked.parse(md, { gfm: true });
const clean = sanitizeHtml(raw, {
  allowedTags: sanitizeHtml.defaults.allowedTags.concat(["img"]),
});
console.log(clean);
```

The sanitizer is not optional decoration. I tried the script without it, fed it a changelog that contained an inline HTML snippet, and marked handed `<script>alert(document.cookie)</script>` straight through into the output. The [DOMPurify project](https://github.com/cure53/DOMPurify) documents this class of problem well if you want the full picture; on the server, sanitize-html does the same job.

## Method 3: the browser, when it's a one-off

Installing packages for a single blog post felt heavy. For those cases I now paste the Markdown into [this browser-based Markdown preview tool](https://utilvy.com/en/tools/markdown-preview/), read the rendered panel, and hit Copy HTML when it looks right. It supports the GitHub-style subset most README files use: headings, tables, fenced code, blockquotes, links and images.

Two details make it fit my CMS workflow. The tool escapes raw HTML typed into the source and strips links that use `javascript:` or `data:` URLs, so the copied output is safe to paste into a template without a second sanitizing pass.

The other detail is privacy. Everything renders client-side, so I've used it for internal release notes that shouldn't touch any server at all. When I need a quick sanity check on someone else's README snippet, [a free online Markdown previewer like this one](https://utilvy.com/en/tools/markdown-preview/) beats spinning up a sandbox repo just to see a table.

## Three pitfalls that cost me real time

**1. Forgetting the `tables` extension.** Without it, Python-Markdown renders my table as one flat paragraph, which is the exact ASCII-art output that hit production:

```html
<p>| Feature | Supported |
| --- | --- |
| Tables | Yes |</p>
```

GitHub hides this because GFM enables tables everywhere by default. Worse, the extension name is singular: I once passed `"table"`, which fails at import time with `ModuleNotFoundError: No module named 'table'`. Unhelpful, but at least loud.

**2. Assuming renderers strip raw HTML.** They don't. Python-Markdown 3.10.1 kept `<script>alert(document.cookie)</script>` and `<img src=x onerror=alert(1)>` intact in its output, and marked 12.0.2 did the same. If any input comes from humans you don't trust — comments, support tickets, Confluence exports — a sanitizer is mandatory, not a nice-to-have.

**3. Single newlines vanishing.** In GitHub issues and comments, pressing Enter once starts a new line. In Python-Markdown, it doesn't:

```html
<p>First line of my changelog
Second line, meant to be a separate row
Third line</p>
```

All three lines collapse into one paragraph in the browser. Adding `nl2br` to the extensions list converts each newline to a `<br />`, which matches what people expect when they paste from a text editor.

## What I actually do now

| Situation | What I reach for |
| --- | --- |
| Recurring build step | Python `markdown` 3.10.1 or `marked` 12.0.2 + sanitize-html |
| Untrusted input in the source | Always a sanitizer, no exceptions |
| One-off post for the CMS | [utilvy.com/en/tools/markdown-preview/](https://utilvy.com/en/tools/markdown-preview/), then Copy HTML |
| Debugging a table that won't render | Check the extension list first, the syntax second |

The biggest lesson wasn't technical. Renderers differ, and "it looked fine on GitHub" is never evidence that it will look fine anywhere else.

More free browser-based utilities at [utilvy.com](https://utilvy.com/en/).