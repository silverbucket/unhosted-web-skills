---
name: webfinger
description: Discover user services (remoteStorage endpoints, ActivityPub actors, profile pages, avatars,
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
<script src="https://cdn.jsdelivr.net/npm/webfinger.js@3/dist/webfinger.js"></script>
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
// PKCE-capable servers (draft-26 §10.1) also advertise:
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

Errors from `lookup()` are always instances of `WebFingerError` (with an optional
`status` for HTTP responses). Errors from `lookupLink()` come in two shapes: a
`WebFingerError` from the underlying lookup, **or** a plain string (`'unsupported
rel X'`, `'no links found with rel="X"'`) for rel-validation failures. Plain
strings are not `Error` instances, so an `instanceof Error` check will miss them.

```javascript
import WebFinger, { WebFingerError } from 'webfinger.js';

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
| `tls_only`                 | `true`  | Require HTTPS for non-localhost hosts. **Localhost is always reached over HTTP**, regardless of this setting (webfinger.js@3.0.3 unconditionally selects `http://` for `localhost` / `localhost.localdomain`). When `false`, the library will also retry over HTTP for any host if the HTTPS request fails. |
| `uri_fallback`             | `false` | If `/.well-known/webfinger` fails, also try `/.well-known/host-meta` and `/.well-known/host-meta.json`.    |
| `request_timeout`          | `10000` | Per-request timeout in milliseconds.                                                                       |
| `allow_private_addresses`  | `false` | Disable the SSRF blocklist. **Development only** — never set `true` in production.                         |

Setting `uri_fallback: true` is recommended when querying older infrastructure
(some self-hosted remoteStorage and OStatus deployments still serve only
host-meta). It costs at most two extra HTTP round-trips on miss.

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

Accepts a bare address (`user@domain`) or an `acct:` URI; bare addresses are
treated as `acct:` automatically. Resolves to a `WebFingerResult`; rejects with
`WebFingerError`.

> **Caveat for non-`acct:` URIs**: webfinger.js@3.0.3 concatenates the address
> into the query string verbatim (`'?resource=' + uri + address`) without
> percent-encoding. Any `&`, `?`, `=`, or other reserved character in a passed
> URI corrupts the request. For example, `lookup('https://example.com/p?x=1&y=2')`
> emits `…?resource=https://example.com/p?x=1&y=2`, which the server parses as
> `resource=https://example.com/p?x=1` plus a stray `y=2` parameter. Stick to
> bare `user@domain` or `acct:user@domain` until the library escapes the value.

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
> - The `camlistore` rel is in the allowed list but a typo in webfinger.js
>   v3.0.3's internal mapping (`camilstore` vs `camlistore`) prevents links from
>   landing in the indexed view. Avoid relying on it.

### `WebFingerError`

```typescript
class WebFingerError extends Error {
  name: 'WebFingerError';
  status?: number; // set for HTTP-origin errors (e.g. 404, 5xx)
}
```

Exported as a **named** export alongside the default class:

```javascript
import WebFinger, { WebFingerError } from 'webfinger.js';
```

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

- **HTTPS for public hosts.** `tls_only: true` is the default; redirects to
  public hosts must also be HTTPS. The library will not downgrade an HTTPS
  request to HTTP unless `tls_only: false` is set explicitly. Note: `localhost`
  is a deliberate exception — it is always reached over HTTP, even with
  `tls_only: true`. Pair this with `allow_private_addresses: true` for any
  local-server testing.
- **Private and internal addresses are blocked** by default — including DNS
  resolution. Blocked ranges:
  - Loopback: `127.0.0.0/8`, `::1`, `localhost`, `localhost.localdomain`
  - Private IPv4: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
  - Link-local: `169.254.0.0/16`, `fe80::/10`
  - Multicast: `224.0.0.0/4`, `ff00::/8`
  - Reserved: `240.0.0.0/4`
  - IPv6 ULA: `fc00::/7`
- **Redirects are capped** at 3 hops. Each redirect target is re-validated
  against the blocklist (mitigates DNS-rebinding).
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
Accept: application/jrd+json
```

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
- **Node.js**: 18+ (the library uses the global `fetch`; no `engines` is
  declared but 18 is the effective floor).
- **Bun**: supported as a first-class target.
- **Module formats**:
  - ESM: `import WebFinger from 'webfinger.js'` (default export)
  - CJS: `const WebFinger = require('webfinger.js'); new WebFinger({ ... })`
    (the class is the module export; `WebFinger.default` also works via a
    compatibility shim)

## Requirements

- Browser with `fetch` (any modern browser) **or** Node.js 18+ / Bun
- npm package: `webfinger.js` (current major: `3.x`)
- Zero runtime dependencies

## Related Packages

- `webfinger.js` — Client library (this skill)
- `remotestoragejs` — Uses WebFinger for storage discovery; see the
  `remotestorage` skill in this marketplace
- `@sockethub/server` — Uses ActivityStreams; complements WebFinger-based
  fediverse work; see the `sockethub` skill in this marketplace
