---
name: esyurl-qr-codes
description: "Generate QR codes with esyURL (https://esyurl.fyi): trackable QR codes that can be edited after printing, styled QR codes with dots, rounded corners, brand colours, a logo in the middle or a photo behind, as PNG or SVG, static QR codes for any text, one QR code for both the App Store and Google Play, and scan counts. Use when the user wants a QR code for a menu, flyer, poster, packaging, business card, event, app download or Wi-Fi, wants to add a logo to a QR code, needs a printed QR code to point somewhere new, or wants to know how many people scanned it."
---

# QR codes with esyURL

esyURL (https://esyurl.fyi) renders QR codes, plain or styled (dot modules,
rounded corner markers, colours, a logo in the middle, or a photo behind the
code). A QR code of an esyURL short link is **editable after printing**: change
the link's destination and every printed code follows. Scans are counted
apart from clicks.

## Signing in

This plugin connects the `esyurl` MCP server; use its tools. The first time a
tool is called, Claude Code asks the user to sign in to esyurl.fyi in the
browser (one-click account, email, or an existing key). If a tool answers
that authentication is needed, tell the user to run `/mcp`, pick `esyurl` and
authenticate. Never ask the user to paste a key into the chat.

## Pick the right kind

- **Trackable and editable (recommended for anything printed):** make a short
  link, then render its QR code. The image encodes `{shortUrl}?s=qr`.
- **Static, not tracked:** `render_qr_code` with `data` encodes any text (a
  URL, Wi-Fi string, vCard). Nothing is saved; it can't be changed later.

## QR code of a link

1. `create_short_link` with `targetUrl` and, for a styled code, `qrStyle`:

   ```json
   {"targetUrl": "https://example.com/menu", "slug": "menu",
    "qrStyle": {"modules": "dot", "eyeFrame": "rounded", "eyeBall": "circle", "dark": "#0a1630"}}
   ```

2. `render_qr_code` with the `linkId` (options: `size` 64–2048, `margin`
   0–16, `dark`/`light` hex, `ecl` L/M/Q/H). It returns the PNG, shown in the
   chat, plus a line with **download URLs** for PNG and SVG that work for an
   hour without a key.

To save a file, fetch the download URL and give the user the path:

```sh
curl -sf -o menu-qr.svg "https://esyurl.fyi/qr/<token>.svg"
curl -sf -o menu-qr.png "https://esyurl.fyi/qr/<token>.png?size=1024"
```

SVG is best for print. Don't paste base64 into the chat.

## Styles

Styled codes are free. Set `qrStyle` on the link (`update_short_link`
replaces the whole `qrStyle`); for a static code pass the same fields
directly to `render_qr_code`. All optional:

| Field | Values |
| --- | --- |
| `modules` | `square` `rounded` `dot` `row` `column` |
| `eyeFrame`, `eyeBall` | `square` `rounded` `circle` (corner markers) |
| `dark`, `light`, `eyeColor` | hex colours |
| `errorCorrectionLevel` (`ecl` for static codes) | `L` `M` `Q` `H` |
| `margin` | 0–16 |
| `image` | `{"mode": "logo", "data": "data:image/png;base64,...", "size"?: 0.1–0.3}` or `{"mode": "halftone", "data": ...}` |

- Logos go in the middle (error correction becomes H). Halftone draws the code
  as dots over a photo. png, jpeg, webp or gif, up to 2 MB, as a data URL
  (read the user's file and base64-encode it).
- To change other fields and keep the picture, send back the `imageId` the
  link shows instead of `data`.
- Dark on light, contrast at least 3:1 (rejected otherwise). Don't invert
  colours.
- A static code with an `image` has no download URL. For a file with a logo,
  put the logo in a link's `qrStyle` and render the link.

## Static QR code (any text)

`render_qr_code` with `data`, for example
`{"data": "WIFI:T:WPA;S:Cafe;P:secret;;", "modules": "rounded", "size": 800}`.
Save it from the download URL as above.

## One code for both app stores

Add redirect rules to the link with `update_short_link`; the printed code
stays the same:

```json
{"linkId": "...", "targetUrl": "https://example.com/app",
 "rules": [
   {"name": "iOS", "when": {"field": "os", "op": "eq", "value": "ios"}, "targetUrl": "https://apps.apple.com/app/id123456789"},
   {"name": "Android", "when": {"field": "os", "op": "eq", "value": "android"}, "targetUrl": "https://play.google.com/store/apps/details?id=com.example.app"}]}
```

Check it with `test_redirect_rules`. Rules can also match country, language,
device or time: see the `esyurl-short-links` skill.

## Scan counts

`get_short_link_visits` with the `linkId` (and `days`): `totals.qr` is scans,
`totals.direct` clicks, `totals.bot` link previews; `daily` per UTC day. Only
codes rendered by esyURL are counted.

## Changing a printed code

`update_short_link` with a new `targetUrl`: every printed copy now goes
there. Restyling (`qrStyle`) changes the image, so reprint for that.

## Reference

The live documentation wins over anything here:
https://esyurl.fyi/llms.txt (API, MCP tools, limits). Tool errors are JSON:
`{"error": {"code", "message", "details"?}}`; retry 5xx with backoff, don't
repeat a 4xx unchanged.
