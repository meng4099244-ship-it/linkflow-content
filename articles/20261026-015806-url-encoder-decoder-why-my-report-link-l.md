I shipped a "copy view link" button for our internal reporting page last month. It produced links that looked like this:

```
https://reports.acme.com/view?redirect=https://app.acme.com/dash?tab=sales&range=30d&q=café
```

Two question marks. Three ampersands. One accented character. Pasted into Slack, the link half worked: people landed on the default tab, `range` silently reset to 7 days, and the search box came back empty.

Nothing crashed, which is what made it annoying. The URL was ambiguous, not invalid. Every `&` after the first one belonged to whoever parsed it first.

## See what the server actually received

Before changing code I wanted the damage in front of me. Python's `parse_qs` splits a query string roughly the way most web frameworks do:

```python
from urllib.parse import parse_qs

qs = "redirect=https://app.acme.com/dash?tab=sales&range=30d&q=café"
print(parse_qs(qs))
```

```
{'redirect': ['https://app.acme.com/dash?tab=sales'],
 'range': ['30d'],
 'q': ['café']}
```

Three parameters where I intended two. My `redirect` value got cut at the first `&`, and `range` — a key I never sent at the top level — arrived as its own parameter. The dashboard read `redirect`, saw a URL with no `range` in it, and fell back to defaults. One missing escape explained the entire afternoon.

## Method 1: build the query string in Python 3.11.1

Stop concatenating. `urlencode` escapes each value so the delimiters inside a value stop acting like delimiters:

```python
from urllib.parse import urlencode, parse_qs

params = {
    "redirect": "https://app.acme.com/dash?tab=sales&range=30d",
    "q": "café",
}
query = urlencode(params)
print(query)
print(parse_qs(query))
```

```
redirect=https%3A%2F%2Fapp.acme.com%2Fdash%3Ftab%3Dsales%26range%3D30d&q=caf%C3%A9
{'redirect': ['https://app.acme.com/dash?tab=sales&range=30d'], 'q': ['café']}
```

Back to two parameters, and the round trip is clean. `?` became `%3F`, `&` became `%26`, `/` became `%2F`. The `é` turned into `%C3%A9` because the string is UTF-8 encoded first and each byte gets its own escape.

For a single value, `quote` is the primitive underneath `urlencode`. Note the default, though — I come back to it in the pitfalls below:

```python
from urllib.parse import quote

value = "https://app.acme.com/dash?tab=sales&range=30d"
print(quote(value))           # https%3A//app.acme.com/dash%3Ftab%3Dsales%26range%3D30d
print(quote(value, safe=""))   # https%3A%2F%2Fapp.acme.com%2Fdash%3Ftab%3Dsales%26range%3D30d
```

[RFC 3986 §2.1](https://datatracker.ietf.org/doc/html/rfc3986#section-2.1) is worth reading once if you want to know where the `%` convention comes from, and why the set of characters you must escape differs between a path segment and a query value.

## Method 2: let the URL object do it in Node.js 22.13.1

In JavaScript I stopped hand-encoding years ago. `URL` and `URLSearchParams` keep the escaping in one place and won't let you skip a value:

```js
const u = new URL("https://reports.acme.com/view");
u.searchParams.set("redirect", "https://app.acme.com/dash?tab=sales&range=30d");
u.searchParams.set("q", "café");

console.log(u.href);
console.log(new URL(u.href).searchParams.get("redirect"));
```

```
https://reports.acme.com/view?redirect=https%3A%2F%2Fapp.acme.com%2Fdash%3Ftab%3Dsales%26range%3D30d&q=caf%C3%A9
https://app.acme.com/dash?tab=sales&range=30d
```

The same bytes Python produced. That mattered more than it sounds: the two services had been disagreeing about this link for weeks.

If you only need a raw value, `encodeURIComponent` is the function, not `encodeURI`. `encodeURI` assumes you handed it a whole URL and deliberately leaves `&`, `?`, and `#` alone, which is the opposite of what you want for a value. MDN's page on [encodeURIComponent](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent) lists what each one escapes.

## Method 3: when you just need one string encoded right now

Most of the time I'm not writing code for this. I'm looking at a broken link somebody sent me, or checking what a value *should* look like before I paste it into a config file. Opening a REPL and importing `urllib.parse` is more ceremony than the job deserves.

For that I paste the value into [a free online URL encoder](https://utilvy.com/en/tools/url-encoder/), take the escaped string, and drop it where it needs to go. It runs in the browser, so nothing I paste leaves my machine — which matters when the value contains an internal hostname or a signed token.

## Three ways I've broken this since

**1. `quote()` leaves `/` alone by default.** The signature is `quote(string, safe='/')`. That's reasonable when you're encoding a whole path and wrong when you're encoding a single value that happens to contain slashes:

```python
print(quote("a/b?x=1&y=2"))           # a/b%3Fx%3D1%26y%3D2
print(quote("a/b?x=1&y=2", safe=""))  # a%2Fb%3Fx%3D1%26y%3D2
```

The first line bit me when a filename parameter containing a slash was read as an extra path segment and the route 404'd. Pass `safe=""` whenever the thing you're encoding is one value.

**2. `+` and `%20` are not the same thing.** `urlencode` calls `quote_plus`, so a space becomes `+`. Legal in a query string, mandatory in `application/x-www-form-urlencoded` bodies, and not a space anywhere else. Python's `unquote` won't convert it:

```python
from urllib.parse import unquote, unquote_plus

print(unquote("file=Q3+report.pdf"))       # file=Q3+report.pdf
print(unquote_plus("file=Q3+report.pdf"))  # file=Q3 report.pdf
```

Node behaves the same way: `new URLSearchParams({file: "Q3 report.pdf"}).toString()` returns `file=Q3+report.pdf`. Fine for query values. Building a path? Use `%20` — `quote(v, safe="")` or `encodeURIComponent`.

**3. Double encoding is silent.** Encode an already-encoded string once more and `%20` turns into `%2520`. Nothing raises; the value simply never matches:

```python
print(quote(quote("Q3 report.pdf")))  # Q3%2520report.pdf
```

Node returns `Q3%2520report.pdf` from `encodeURIComponent(encodeURIComponent(...))` as well. Decoding handles the two languages differently, too — JavaScript is loud, Python is quiet:

```js
decodeURIComponent("%E0%A4%A"); // URIError: URI malformed
```

I decode once, print the result, and look for a leftover `%`. If there is one, something upstream already encoded it.

## Which one to reach for

| Situation | Use | Watch out for |
| --- | --- | --- |
| Building a query string in Python | `urlencode(dict)` | Spaces become `+`; pass `quote_via=quote` for `%20` |
| One value, Python | `quote(v, safe="")` | Default `safe='/'` leaves slashes unescaped |
| Building a URL in Node | `new URL()` + `searchParams.set()` | Don't `encodeURI` the value first |
| One value, JavaScript | `encodeURIComponent` | `encodeURI` leaves `&`, `?`, `#` alone |
| Inspecting or fixing a link by hand | [this browser-based percent-encoding tool](https://utilvy.com/en/tools/url-encoder/) | Decode once, then look for a leftover `%` |

The rule I keep coming back to is boring: encode at the moment you insert a value, and never encode a string you didn't build yourself.

Since the fix, the only URL bugs left in my inbox are the ones people type by hand. For those, [the encoder I keep bookmarked](https://utilvy.com/en/tools/url-encoder/) beats opening a REPL.

More free browser-based utilities at utilvy.com