---
name: webfinger
description: >-
  Discover user services (remoteStorage endpoints, ActivityPub actors, profile pages, avatars,
  OpenID Connect issuers) from an email-like address using the WebFinger protocol (RFC 7033). Use when
  building apps that resolve acct: URIs, look up user metadata from `user@domain`, integrate with
  remoteStorage, Mastodon or other fediverse software, implement OpenID Connect issuer discovery, or
  work with the webfinger.js client library.
license: MIT
metadata:
  author: silverbucket
---

# webfinger.js

A modern TypeScript client library for the WebFinger protocol (RFC 7033). Resolves
an email-like address (`user@domain`) to a JSON Resource Descriptor (JRD) that
points at the user's services — remoteStorage, ActivityPub actor, profile page,
avatar, OpenID Connect issuer, and so on. Works in browsers and Node.js with zero
runtime dependencies and SSRF protection enabled by default.

## When to Use

Invoke when you need to:

- Resolve a fediverse handle (`@user@domain`) to an ActivityPub actor URI
- Look up a user's remoteStorage server, OAuth endpoints, and storage-spec version
- Discover a user's profile page, avatar, or other public services from `user@domain`
- Implement OpenID Connect issuer discovery (RFC 7033 §3.1)
- Build any app that takes an `acct:` URI and dereferences it to service endpoints
- Add user discovery to a remoteStorage.js, Mastodon, or PeerTube integration
- Validate that a WebFinger record is well-formed before pointing infrastructure at it
- Debug a JRD response or work around the host-meta legacy fallback

## Quick Start

### Install

```bash
npm install webfinger.js
# or
bun add webfinger.js
# or
yarn add webfinger.js
```

CDN (browser):

```html
<script src="https://cdn.jsdelivr.net/npm/webfinger.js@3/dist/webfinger.min.js"></script>
```

### Basic Lookup

```javascript
import WebFinger from 'webfinger.js';

const wf = new WebFinger({
  tls_only: true,        // HTTPS only (default; do not disable on the public web)
  uri_fallback: true,    // Try host-meta(.json) if /.well-known/webfinger 404s
  request_timeout: 10000 // ms
});

const result = await wf.lookup('user@example.org');

// Indexed view: only links/properties webfinger.js knows how to normalize
console.log('Name:',   result.idx.properties.name);          // string | undefined
console.log('Avatar:', result.idx.links.avatar?.[0]?.href);  // string | undefined

// Raw JRD: always available for any rel/property
console.log('Subject:', result.object.subject);
console.log('All links:', result.object.links);
```

The result has two views. `result.idx` is a curated, normalized view across a small
set of well-known relations. `result.object` is the raw JRD straight from the
server — use it for any rel webfinger.js doesn't know about (notably `self` for
ActivityPub).

## Examples

### Example 1: Resolve a fediverse handle to an ActivityPub actor

The fediverse uses WebFinger to map `@user@domain` to an actor URI, but the
ActivityPub `self` link lives only in the raw JRD — webfinger.js does not index
`self` into `result.idx.links`. Read it from `result.object.links`.

```javascript
import WebFinger from 'webfinger.js';

const wf = new WebFinger({ tls_only: true, uri_fallback: true });

async function resolveActor(handle) {
  // Accept '@Gargron@mastodon.social' or 'Gargron@mastodon.social'
  const addr = handle.replace(/^@/, '');
  const { object: jrd } = await wf.lookup(addr);

  // ActivityPub servers may advertise either MIME type for the actor link.
  const isAPType = (t) =>
    t === 'application/activity+json' ||
    (typeof t === 'string' &&
      t.startsWith('application/ld+json') &&
      t.includes('https://www.w3.org/ns/activitystreams'));

  const self = jrd.links?.find((l) => l.rel === 'self' && isAPType(l.type));
  if (!self?.href) throw new Error(`No ActivityPub actor for ${addr}`);

  const actor = await fetch(self.href, {
    headers: { Accept: 'application/activity+json' }
  }).then((r) => r.json());

  return actor; // ActivityPub Actor object
}

const actor = await resolveActor('@Gargron@mastodon.social');
console.log(actor.id, actor.preferredUsername, actor.inbox);
```

### Example 2: Discover a user's remoteStorage endpoint

remoteStorage advertises a single link with the rel
`http://tools.ietf.org/id/draft-dejong-remotestorage`. webfinger.js normalizes
that long URI to the short alias `'remotestorage'`, so `lookupLink` works
directly. The `properties` object on the link carries the spec version and the
OAuth endpoints (with PKCE additions in newer servers).

```javascript
import WebFinger from 'webfinger.js';

const wf = new WebFinger({ tls_only: true });

const storage = await wf.lookupLink('user@5apps.com', 'remotestorage');

const storageRoot = storage.href;
const props       = storage.properties ?? {};

const version  = props['http://remotestorage.io/spec/version'];
const oauthUrl = props['http://tools.ietf.org/html/rfc6749#section-4.2'];
// PKCE-capable servers (draft-27 §10.1) also advertise:
const authEp   = props['http://tools.ietf.org/html/rfc6749#section-3.1'];
const tokenEp  = props['http://tools.ietf.org/html/rfc6749#section-3.2'];
const pkce     = props['http://tools.ietf.org/html/rfc7636']; // code_challenge methods

console.log({ storageRoot, version, oauthUrl, authEp, tokenEp, pkce });
```

### Example 3: Look up profile page and avatar

Both rels are normalized into `result.idx.links` under the short keys `profile`
and `avatar`. Each is an array — there can legitimately be more than one — so
use optional chaining and treat the first entry as preferred (RFC 7033 §4.4.4
says link order MAY indicate preference).

```javascript
import WebFinger from 'webfinger.js';

const wf = new WebFinger({ tls_only: true, uri_fallback: true });
const result = await wf.lookup('alice@example.org');

const profile = result.idx.links.profile?.[0]?.href;
const avatar  = result.idx.links.avatar?.[0]?.href;
const name    = result.idx.properties.name; // only if JRD has http://packetizer.com/ns/name

console.log({ name, profile, avatar });
```

### Example 4: Batch lookups with `Promise.allSettled`

WebFinger requests are independent and benefit from concurrency. Use
`Promise.allSettled` so one bad address doesn't fail the whole batch.

```javascript
import WebFinger from 'webfinger.js';

const wf = new WebFinger({ tls_only: true, request_timeout: 8000 });

async function lookupMany(addresses) {
  const settled = await Promise.allSettled(addresses.map((a) => wf.lookup(a)));
  return settled.map((res, i) => ({
    address: addresses[i],
    ok: res.status === 'fulfilled',
    jrd: res.status === 'fulfilled' ? res.value.object : null,
    error: res.status === 'rejected' ? res.reason : null
  }));
}

const results = await lookupMany([
  'Gargron@mastodon.social',
  'nobody@invalid.example',
  'user@5apps.com'
]);
```

### Example 5: Error handling

Errors come in three shapes from webfinger.js@3.1.0:

1. **`WebFingerError`** (with an optional `status` for HTTP responses) — the
   library's own failures: bad address, SSRF block, HTTP 4xx/5xx from the
   server, malformed JRD, request timeout.
2. **Raw `TypeError`** — propagated verbatim from the runtime's `fetch()` for
   transport failures (DNS failure, TLS handshake failure, network unreachable,
   and — in browsers — CORS rejection). The library does not wrap these.
3. **Plain string** (only from `lookupLink`) — `'unsupported rel <rel>'` or
   `'no links found with rel="<rel>"'`. Not an `Error` instance, so
   `instanceof Error` misses them.

A robust handler covers all three:

```javascript
import WebFinger, { WebFingerError } from 'webfinger.js';
// CJS / CDN builds do not expose WebFingerError — use
// err.name === 'WebFingerError' instead of instanceof there.

const wf = new WebFinger({ tls_only: true });

try {
  const link = await wf.lookupLink('nobody@example.com', 'remotestorage');
  console.log(link.href);
} catch (err) {
  if (err instanceof WebFingerError) {
    if (err.status === 404) {
      console.warn('No WebFinger record for that address.');
    } else {
      console.error(`WebFinger failed (${err.status ?? 'no status'}): ${err.message}`);
    }
  } else if (err instanceof TypeError) {
    // Transport-level failure from fetch: DNS, TLS, network, or CORS.
    console.error('Network failure:', err.message);
  } else if (typeof err === 'string') {
    // 'unsupported rel <rel>' or 'no links found with rel="<rel>"'
    console.warn('lookupLink rejection:', err);
  } else {
    throw err;
  }
}
```

## Configuration

All four options are independent; pass any subset.

| Option                     | Default | Purpose                                                                                                    |
|----------------------------|---------|------------------------------------------------------------------------------------------------------------|
| `tls_only`                 | `true`  | Require HTTPS for non-localhost hosts. **Localhost is always reached over HTTP**, regardless of this setting (webfinger.js@3.1.0 unconditionally selects `http://` for `localhost` / `localhost.localdomain`). When `false`, the library will also retry over HTTP for any host if the HTTPS request fails. |
| `uri_fallback`             | `false` | If `/.well-known/webfinger` fails, also try `/.well-known/host-meta` and `/.well-known/host-meta.json`. See note below — webfinger.js parses all three paths as JRD (JSON), so legacy XML host-meta responses will not succeed. |
| `request_timeout`          | `10000` | Timeout in milliseconds **per HTTP attempt** — every redirect hop, fallback URI and protocol retry gets a fresh budget. Worst-case wall time ≈ timeout × (redirects + 1) × URIs × protocols. |
| `allow_private_addresses`  | `false` | Disable the SSRF blocklist. **Development only** — never set `true` in production.                         |

Setting `uri_fallback: true` is useful only against deployments that serve a
JSON body at `/.well-known/host-meta` or `/.well-known/host-meta.json` — for
example, a server that has not migrated to the RFC 7033 WebFinger path but
already offers JRD. webfinger.js@3.1.0 does not parse XML; a classic XML
host-meta response (`application/xrd+xml`) rejects with
`WebFingerError: invalid json` and the fallback does not help. Each fallback
miss costs one extra HTTP round-trip.

## API Reference

### Constructor

```typescript
new WebFinger(cfg?: Partial<WebFingerConfig>)
```

`WebFingerConfig` has exactly the four fields above; unknown fields are ignored.

### `lookup(address)`

```typescript
lookup(address: string): Promise<WebFingerResult>
```

Accepts a bare address (`user@domain`), an `acct:` URI, or any full URI with
a scheme (`https://example.com/p?x=1`); bare addresses are treated as `acct:`
automatically. Since 3.1.0 the resource is percent-encoded before it goes into
the query string (reserved characters such as `&`, `#`, `?`, `+`, `=` are
escaped; `:`, `@` and `/` stay literal), internationalized hostnames are
converted to A-labels (punycode) and the userpart is NFC-normalized — so
`jürgen@müller.example` is sent as
`acct:j%C3%BCrgen@xn--mller-kva.example`. Rejects with `WebFingerError` for
library-level failures and with raw `TypeError` for `fetch` failures — see the
error-handling notes below.

**Error propagation:**

- Library-level errors (bad address, SSRF block, HTTP 4xx/5xx, malformed JRD,
  timeout) reject with `WebFingerError` (see below).
- Transport-level `fetch` failures (DNS failure, TLS handshake failure,
  network unreachable, CORS rejection in the browser) are **not** wrapped —
  they propagate as the raw `TypeError` thrown by the runtime's `fetch`
  implementation. A robust caller must catch both shapes.

### `lookupLink(address, rel)`

```typescript
lookupLink(address: string, rel: string): Promise<LinkObject>
```

Convenience wrapper around `lookup()` that returns the first link with the given
indexed rel. The `rel` argument must be one of the short aliases below — passing
the full URI form (e.g. `'http://webfinger.net/rel/avatar'`) does **not** work.

| Allowed `rel`     | Matches JRD link rel                                                                  |
|-------------------|---------------------------------------------------------------------------------------|
| `avatar`          | `http://webfinger.net/rel/avatar`                                                     |
| `profile`         | `http://webfinger.net/rel/profile-page`, `me`                                         |
| `remotestorage`   | `http://tools.ietf.org/id/draft-dejong-remotestorage`, `remotestorage`, `remoteStorage` |
| `vcard`           | `vcard`                                                                               |
| `blog`            | `blog`, `http://packetizer.com/rel/blog`                                              |
| `updates`         | `http://schemas.google.com/g/2010#updates-from`                                       |
| `share`           | `http://www.packetizer.com/rel/share`                                                 |
| `camlistore`      | `https://camlistore.org/rel/server` (broken — see gotcha below)                       |

> **Gotchas for `lookupLink`:**
>
> - Any other rel rejects with the **plain string** `'unsupported rel <rel>'` —
>   not a `WebFingerError`, not even an `Error` instance. Catch with
>   `typeof err === 'string'`.
> - When the lookup succeeds but no matching link exists, it rejects with the
>   plain string `'no links found with rel="<rel>"'`.
> - For non-indexed relations (notably `self` for ActivityPub, or
>   `http://openid.net/specs/connect/1.0/issuer` for OIDC), call `lookup()` and
>   read `result.object.links` directly.
> - The `camlistore` rel is in the allowed list but a long-standing typo in
>   webfinger.js's internal mapping (`camilstore` vs `camlistore`, still
>   present as of v3.1.0) prevents links from landing in the indexed view.
>   Avoid relying on it.

### `WebFingerError`

```typescript
class WebFingerError extends Error {
  name: 'WebFingerError';
  status?: number; // set for HTTP-origin errors (e.g. 404, 5xx)
}
```

Exported as a **named** export alongside the default class — **ESM only**:

```javascript
import WebFinger, { WebFingerError } from 'webfinger.js';
```

The CJS build (`require('webfinger.js')`) and the CDN/UMD bundle export the
`WebFinger` class alone with nothing attached, so `WebFingerError` is not
reachable there. Check `err.name === 'WebFingerError'` instead of
`instanceof` in those environments.

`WebFingerError` covers the library's own error paths. **It does not cover
transport failures** — a `fetch()` that throws `TypeError` (DNS, TLS, network
unreachable, CORS) propagates raw. See Example 5 for the three-shape catch
pattern.

### Result shape

```typescript
type WebFingerResult = {
  object: JRD;           // raw JRD from the server
  idx: {
    links: { [shortRel: string]: LinkObject[] };
    properties: Record<string, unknown>;
  };
};

type JRD = {
  subject?: string;
  links: Array<Record<string, unknown>>;
  properties?: Record<string, unknown>;
  error?: string;
};

type LinkObject = {
  href: string;
  rel: string;
  type?: string;
  properties?: Record<string, string | null>;
  [key: string]: unknown;
};
```

There is no `result.json` field — the raw JRD lives at `result.object`. Only one
property is normalized into `result.idx.properties`: the namespaced `name`
(`http://packetizer.com/ns/name`). Anything else must be read from
`result.object.properties`.

## Security

webfinger.js is built around an "ActivityPub-grade" SSRF posture and ships safe
defaults. Treat the defaults as the baseline; only relax them with intent.

- **HTTPS for public hosts only.** `tls_only: true` is the default. For
  non-localhost hosts, webfinger.js issues the request over HTTPS and (with
  defaults) does not retry over HTTP; set `tls_only: false` to opt into an
  HTTP retry after HTTPS failure. **Gap:** the redirect handler never checks
  the target's scheme, so an HTTPS→HTTP redirect is followed even with
  `tls_only: true`, although RFC 7033 §4.2 requires clients to reject it.
  **`tls_only` does not cover `localhost`**: for `localhost` /
  `localhost.localdomain` the library selects HTTP unconditionally, regardless
  of this flag. Reaching a local WebFinger server therefore also requires
  `allow_private_addresses: true`.
- **Private and internal addresses are blocked** by default — including DNS
  resolution. Blocked ranges:
  - Loopback / unspecified: `127.0.0.0/8`, `0.0.0.0/8`, `::1`, `::`,
    `localhost`, `localhost.localdomain`
  - Private IPv4: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
  - Link-local: `169.254.0.0/16`, `fe80::/10`
  - Multicast: `224.0.0.0/4`, `ff00::/8`
  - Reserved: `240.0.0.0/4`
  - IPv6 ULA: `fc00::/7`
  - Embedded IPv4 forms of any blocked IPv4 address: IPv4-mapped
    `::ffff:0:0/96` (3.0.6, GHSA-fwvw-2r88-m4cv), IPv4-compatible `::/96`,
    and the NAT64 well-known prefix `64:ff9b::/96` (3.1.0)
- **DNS-resolution check** (Node.js / Bun only): hostnames are resolved and
  every resulting IP is checked against the blocklist, so `localtest.me` →
  `127.0.0.1` is rejected. Since 3.1.0 the check also goes through
  `dns.lookup` (getaddrinfo), catching `/etc/hosts` aliases such as
  `ip6-localhost`. The `dns` module is loaded via `process.getBuiltinModule`
  (3.0.5); on Node.js older than 20.16 that function is missing and the DNS
  check is **silently skipped** — only literal IPs and hostnames are blocked.
- **Host canonicalization** (v3.0.4+): alternative loopback encodings such as
  `2130706433`, `0x7f000001`, and `127.1` are canonicalized before the blocklist
  check, so they cannot be used to bypass SSRF protection.
- **Redirects are capped** at 3 hops. Each redirect target is re-validated
  against the blocklist using the same canonicalization pipeline as the initial
  lookup (mitigates DNS-rebinding).
- **Per-attempt timeouts** (v3.0.4+): `request_timeout` is enforced via
  `AbortController` on each HTTP attempt, so stalled response bodies no longer
  hang the promise past the configured bound. Each redirect hop, fallback URI
  and protocol retry gets a fresh budget, so a single `lookup()` can take
  roughly timeout × (redirects + 1) × URIs × protocols in the worst case.
- **Not covered by the library:** no response-size cap (the body is read in
  full), and the `Content-Type` is only logged, not enforced — any body that
  parses as JSON is accepted. Add those limits yourself if you need them.
- **`allow_private_addresses: true` is for local development only.** Use it when
  testing against a local WebFinger server; never in production.
- **Browser deployments** rely on the browser's own CORS and same-origin
  enforcement. The DNS-resolution check above only runs in Node.js / Bun.

## Protocol Basics

A WebFinger request is a single GET against `/.well-known/webfinger` on the
host portion of the address:

```http
GET /.well-known/webfinger?resource=acct%3Aalice%40example.com HTTP/1.1
Host: example.com
Accept: application/jrd+json, application/json
```

(`application/jrd+json, application/json` is what webfinger.js sends; RFC 7033
only requires that the server return JRD when no other format is requested.)
The response is a JSON Resource Descriptor (JRD): a `subject`, optional
`aliases` and `properties`, and an array of `links`. Each link has at minimum a
`rel` and usually an `href` (or a `template` with a `{uri}` placeholder for
interaction relations like OStatus subscribe). Servers MUST return CORS
`Access-Control-Allow-Origin: *` for public records and MUST serve over HTTPS.

For the full RFC-level reference — JRD schema, `titles`, `properties`,
`aliases`, host-meta fallback, `acct:` URI syntax, common link relations,
ActivityPub resolution flow with verbatim Mastodon example, and remoteStorage
property names — see `references/protocol.md`.

## Environment support

- **Browsers**: ESM via bundlers, or the CDN script tag above.
- **Node.js**: `>=22.12.0` (declared in `engines`). The DNS-resolution SSRF
  check depends on `process.getBuiltinModule`; on older Node it is silently
  skipped, leaving only the literal-address blocklist.
- **Bun**: supported as a first-class target.
- **Module formats**:
  - ESM: `import WebFinger, { WebFingerError } from 'webfinger.js'` (default
    export plus named `WebFingerError`; the types `WebFingerConfig`, `JRD`,
    `WebFingerResult` and `LinkObject` are also exported from
    `dist/webfinger.d.ts`)
  - CJS: `const WebFinger = require('webfinger.js'); new WebFinger({ ... })`
    (the class is the module export; `WebFinger.default` also works via a
    compatibility shim; `WebFingerError` is **not** exported)
  - CDN/UMD: exposes only `window.WebFinger`

## Requirements

- Browser with `fetch` (any modern browser) **or** Node.js 22.12+ / Bun
- npm package: `webfinger.js` (current major: `3.x`)
- Zero runtime dependencies

## Related Packages

- `webfinger.js` — Client library (this skill)
- `remotestoragejs` — Uses WebFinger for storage discovery; see the
  `remotestorage` skill in this marketplace
- `@sockethub/server` — Uses ActivityStreams; complements WebFinger-based
  fediverse work; see the `sockethub` skill in this marketplace
