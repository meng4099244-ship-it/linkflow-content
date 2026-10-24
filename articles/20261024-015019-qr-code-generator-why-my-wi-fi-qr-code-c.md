# QR Code Generator: Why My Wi-Fi QR Code Connected Nobody (and the Escaping Fix That Did)

I printed forty stickers for our office guest network last month. Each one carried a QR code, a hand-drawn Wi-Fi glyph, and the words "Scan to connect." Thirty-eight worked. On the other two, the phone raised a join sheet for a network called `Guest` with an empty password field, failed to connect, and the visitor ended up typing the password by hand — the exact thing the sticker existed to prevent.

The QR codes rendered fine. The string I fed into them was wrong.

## What actually lives inside a Wi-Fi QR code

A Wi-Fi QR code isn't a link. It's a short text record, and both iOS and Android recognise the shape:

```
WIFI:T:WPA;S:GuestNet;P:hunter2!;;
```

`T` is the security type, `S` the SSID, `P` the password, and the double semicolon closes the record. [ZXing documented this format](https://github.com/zxing/zxing/wiki/Barcode-Contents) years ago and every scanner still follows it. One stray character and the join sheet either never appears or shows the wrong credentials.

I had one stray character.

## Method 1 — Python and segno

[segno](https://segno.readthedocs.io/en/stable/) is the QR library I reach for now. Pure Python, zero dependencies, and it writes PNG without Pillow. Everything below runs on Python 3.11.1 with segno 1.6.6.

```python
# pip install segno==1.6.6
from segno import helpers

qr = helpers.make_wifi(ssid="GuestNet", password="hunter2!", security="WPA")
print(qr.version, qr.error, qr.mask)      # 3 M 2
qr.save("guest-wifi.png", scale=10, border=4)
```

Version 3, error correction level M, mask 2. With the 4-module quiet zone the image is 37×37 modules and lands on disk as a 500-byte PNG.

Building that payload by hand is where I broke things:

```python
ssid, password = "Lab;Net", "pa:ss;word\\1"

print(f"WIFI:T:WPA;S:{ssid};P:{password};;")
# WIFI:T:WPA;S:Lab;Net;P:pa:ss;word\1;;

print(helpers.make_wifi_data(ssid=ssid, password=password, security="WPA"))
# WIFI:T:WPA;S:Lab\;Net;P:pa\:ss\;word\\1;;
```

Look at the second line. `;`, `:`, `,` and `\` inside a value must be escaped with a backslash, otherwise the parser thinks the field ended early. My SSID contained a semicolon, so the scanner read `S:Lab` and dropped the rest. That's how a visitor got offered `Guest` with no password.

Owning the escaping yourself is fifteen lines, and it matches segno character for character:

```python
def wifi_payload(ssid, password, security="WPA"):
    def esc(value):
        for ch in ("\\", ";", ",", ":"):
            value = value.replace(ch, "\\" + ch)
        return value

    return f"WIFI:T:{security};S:{esc(ssid)};P:{esc(password)};;"

assert wifi_payload(ssid, password) == helpers.make_wifi_data(
    ssid=ssid, password=password, security="WPA"
)   # passes
```

## Method 2 — Node.js and the qrcode package

Node 22.13.1, qrcode 1.5.4. Same payload, different syntax:

```js
// npm install qrcode@1.5.4
const QRCode = require("qrcode");

const payload = "WIFI:T:WPA;S:GuestNet;P:hunter2!;;";

const info = QRCode.create(payload, { errorCorrectionLevel: "M" });
console.log(info.version, info.modules.size);   // 3 29

await QRCode.toFile("guest-wifi.png", payload, {
  errorCorrectionLevel: "H",
  scale: 10,
  margin: 4,
});
```

`create()` is the part worth knowing. It returns the chosen version and module count without touching the filesystem, which is how I check a size before sending anything to a printer. The same payload at level H moves to version 4 and 33 modules.

For a quick sanity check I render to the terminal and scan it with a second phone:

```js
console.log(await QRCode.toString(payload, { type: "terminal", small: true }));
```

```
▄▄▄▄▄▄▄  ▄▄▄   ▄  ▄▄ ▄▄▄▄▄▄▄
█ ▄▄▄ █ ▄▀ ▀█ ▄▀  ▄▄█ █ ▄▄▄ █
█ ███ █ █▀▀▄▀▀█▀██▄█  █ ███ █
█▄▄▄▄▄█ █ █ █ █▀█▀▄ ▄ █▄▄▄▄▄█
```

## Method 3 — nothing to install

Sometimes I'm on a locked-down laptop where `pip install` isn't an option, or I need one code in the next thirty seconds. That's when I open [a free QR code generator](https://utilvy.com/en/tools/qr-generator/) in the tab that's already open, paste the payload, and download the PNG.

It fits this particular job for three reasons. The encoding happens in the browser, so a Wi-Fi password never travels to somebody else's server. The PNG comes down without a watermark. And the code is static, so it keeps working long after the tab is closed — no account, no expiry, no renewal. I used it for a conference badge run where the payloads were vCards, twenty of them, and it never asked me to sign in.

For one or two codes, [this browser-based QR tool](https://utilvy.com/en/tools/qr-generator/) beats writing a script. I stopped writing one-off scripts for this about a year ago.

## Three things that cost me an afternoon

**1. Unescaped separators.** Covered above, and it's the one that actually shipped to print. `WIFI:T:WPA;S:Lab;Net;` parses as SSID `Lab`. If your SSID or password can contain `;`, `,`, `:` or `\` — ours can — escape them, or hand the job to a helper that does.

**2. Error correction quietly changes the size, and pinning the version fails loudly.** The same 43-character URL gives you version 3 (29×29 modules) at level M and version 5 (37×37) at level H. Ask for a version that can't hold your data and you get two very different messages for the same mistake:

```
segno.encoder.DataOverflowError: The provided data does not fit into version "3"Proposal: version 27
```

```
Error: The chosen QR Code version cannot contain this amount of data.
Minimum version required to store current data is: 27.
```

While you're there, don't set `border=0` or `margin: 0` to squeeze out space. The quiet zone is 4 modules by spec, and scanners start failing on glossy stickers without it.

**3. Non-ASCII payloads depend on the scanner's guess.** `"会议室 WiFi 密码：guest2026"` is 21 characters and 33 UTF-8 bytes. Encode it as plain bytes and a scanner that assumes Latin-1 hands back:

```
ä¼è®®å®¤ WiFi å¯ç ï¼guest2026
```

Passing `eci=True` to `segno.make_qr()` writes an ECI designator that says "this is UTF-8", and the same scanner reads it correctly. When I can't control which scanner someone uses, I keep the payload ASCII and put the Chinese in the printed text beside the code.

## Which one I reach for

| Situation | What I use |
| --- | --- |
| Batch generation in CI | Python + segno, because `make_wifi` handles escaping |
| Already inside a Node build step | qrcode 1.5.4, with `QRCode.create()` to check version first |
| One code, right now, on a work laptop | the [online QR generator](https://utilvy.com/en/tools/qr-generator/) |
| Non-ASCII content | either library, with ECI enabled |
| Anything going on glossy print | level H, quiet zone ≥ 4 modules, ≥ 300 px output |

The boring lesson printing taught me: the picture is the easy half. What breaks a QR code in the real world is the string, the blank space around it, and the encoding nobody wrote down.

More free browser-based utilities at utilvy.com