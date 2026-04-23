# nshrt — Nostr Short Links

Serverless, client-side URL shortener for Nostr identifiers. Encoding is deterministic — the same input always produces the same short URL. No server, no database, no tracking — the short code *is* the data.

**Live:** [https://njmp.to](https://njmp.to)

## How it works

Nostr identifiers (npub, nevent, naddr, note) are bech32-encoded bytes. nshrt strips the bech32 wrapper and re-encodes the raw bytes in **base62** (characters `0–9`, `a–z`, `A–Z`).

A 32-byte pubkey shrinks from `npub1…` (63 chars) to ~43 chars. When someone visits a short link, the JS router decodes the code back to the full identifier and redirects to `njump.me`.

**URL scheme:**

| Path | Resolves to |
|---|---|
| `/p/<base62>` | `npub` — public key |
| `/n/<base62>` | `note` — event (simple) |
| `/e/<base62>` | `nevent` — event |
| `/a/<base62>` | `naddr` — parameterized replaceable event |

> **Note:** relay hints embedded in `nevent` and `naddr` identifiers are dropped during encoding — only the event ID, pubkey, kind, and d-tag are preserved. Links may not resolve if the target event is only available on obscure relays.

---

## Generating nshrt links in your client

Share buttons don't need to call any API. The encoding is pure math — you can construct the short URL entirely client-side using the raw values you already have.

### Base62 encoding

All link types reduce to: encode a byte array as base62. Here's the reference implementation:

```js
const B62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

function bytesToBase62(bytes) {
  let digits = [0];
  for (const byte of bytes) {
    let carry = byte;
    for (let i = 0; i < digits.length; i++) {
      carry += digits[i] * 256;
      digits[i] = carry % 62;
      carry = Math.floor(carry / 62);
    }
    while (carry > 0) {
      digits.push(carry % 62);
      carry = Math.floor(carry / 62);
    }
  }
  let result = "";
  for (let i = 0; i < bytes.length && bytes[i] === 0; i++) result += "0";
  return result + digits.reverse().map(d => B62[d]).join("");
}

function hexToBytes(hex) {
  const bytes = new Uint8Array(hex.length / 2);
  for (let i = 0; i < bytes.length; i++)
    bytes[i] = parseInt(hex.slice(i * 2, i * 2 + 2), 16);
  return bytes;
}
```

### `/p/` — profile link (npub)

Input: hex pubkey (32 bytes)

```js
function profileLink(pubkeyHex) {
  return `https://njmp.to/p/${bytesToBase62(hexToBytes(pubkeyHex))}`;
}
```

### `/n/` — note link (note)

Input: hex event id (32 bytes)

```js
function noteLink(eventIdHex) {
  return `https://njmp.to/n/${bytesToBase62(hexToBytes(eventIdHex))}`;
}
```

### `/e/` — event link (nevent)

Same payload as `/n/`, but resolves to a `nevent` identifier. Use `/e/` when the viewer might need relay context; use `/n/` for simplicity.

```js
function eventLink(eventIdHex) {
  return `https://njmp.to/e/${bytesToBase62(hexToBytes(eventIdHex))}`;
}
```

### `/a/` — addressable event link (naddr)

Input: `kind` (integer), hex pubkey (32 bytes), `d` tag (string)

The payload is `kind (4 bytes big-endian) ‖ pubkey (32 bytes) ‖ d-tag (UTF-8 bytes)`.

```js
function naddrLink(kind, pubkeyHex, dTag) {
  const kindBytes = [
    (kind >>> 24) & 0xff,
    (kind >>> 16) & 0xff,
    (kind >>> 8) & 0xff,
    kind & 0xff,
  ];
  const pubkeyBytes = Array.from(hexToBytes(pubkeyHex));
  const dTagBytes = Array.from(new TextEncoder().encode(dTag));
  const payload = new Uint8Array([...kindBytes, ...pubkeyBytes, ...dTagBytes]);
  return `https://njmp.to/a/${bytesToBase62(payload)}`;
}
```

### Example usage

```js
// Share a profile
const shareUrl = profileLink("7b3b8e..."); // → https://njmp.to/p/7Xk9mQr…

// Share a note
const shareUrl = noteLink("3a1f9c..."); // → https://njmp.to/n/1dKpRqZ…

// Share a long-form article (kind 30023)
const shareUrl = naddrLink(30023, "7b3b8e...", "my-article-slug");
```

---

## Self-hosting

nshrt is a single `index.html`. Deploy it on any static host and configure your server to serve `index.html` for all routes:

**Netlify**

```
# _redirects
/*  /index.html  200
```

**Caddy**

```
# Caddyfile
try_files {path} /index.html
```

**Vercel**

```json
// vercel.json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

**nginx**

```nginx
# nginx.conf
location / {
  try_files $uri /index.html;
}
```

The `BASE_URL` is derived from `window.location.origin` at runtime, so links will automatically use your domain.

---

## Contributing

Issues and pull requests are welcome. Since nshrt is a single `index.html` with no build step, contributions are straightforward — just edit the file and open a PR.

Things that would be useful:

- Support for additional Nostr identifier types
- Improvements to the encoding/decoding logic
- UI or accessibility improvements

---

## Support

If nshrt is useful to you [⚡ Zap this note](nostr:nevent1qqsdmw0m9v8hwmpfrsa2ulg9fudhm0mueqastu5687v9pr0dunxs02qpzamhxue69uhhyetvv9ujuurjd9kkzmpwdejhgtczyr8gjtqms9a54u6xe3vnmwjkmpuwhzkrv9pzpnkfja0usny3yw6fvqcyqqqqqqgjaju5r) or just [send some sats](lightning:sats@liberspace.org) to sats@liberspace.org