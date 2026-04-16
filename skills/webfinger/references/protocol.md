# WebFinger Protocol Reference

Deep-dive reference for the protocols underlying the `webfinger` skill. Read
this when writing a WebFinger server, debugging an unusual JRD response,
implementing host-meta fallback, or reasoning about edge cases the client
library does not expose directly.

## Contents

1. [RFC 7033 — WebFinger](#rfc-7033--webfinger)
2. [JRD schema](#jrd-schema)
3. [RFC 7565 — `acct:` URI](#rfc-7565--acct-uri)
4. [RFC 6415 — host-meta fallback](#rfc-6415--host-meta-fallback)
5. [Common link relations](#common-link-relations)
6. [remoteStorage link properties](#remotestorage-link-properties)
7. [ActivityPub / Mastodon resolution flow](#activitypub--mastodon-resolution-flow)
8. [Security considerations](#security-considerations)

---

## RFC 7033 — WebFinger

WebFinger is a Standards-Track protocol (September 2013) for discovering
information about people or other entities on the Internet, identified by a URI,
using HTTPS over a well-known endpoint.

### Endpoint

The well-known URI suffix `webfinger` is registered under RFC 5785. The full
path component MUST be `/.well-known/webfinger`. The query string MUST contain
the target resource and MAY contain link-relation filters.

```http
GET /.well-known/webfinger?resource=acct%3Acarol%40example.com HTTP/1.1
Host: example.com
Accept: application/jrd+json
```

### Query parameters

| Name       | Cardinality          | Required | Behavior                                                                                       |
|------------|----------------------|----------|------------------------------------------------------------------------------------------------|
| `resource` | exactly once         | yes      | URI identifying the entity. If absent or malformed → `400 Bad Request`. If unknown → `404 Not Found`. |
| `rel`      | zero or more         | no       | Filters the `links` array to the requested relation types. Servers SHOULD support; if not, MUST ignore and return as if `rel` were absent. |

Both parameters and their values are percent-encoded per RFC 3986 §2.1. `=` and
`&` inside parameter values MUST also be percent-encoded.

The `resource` URI scheme is unconstrained: "WebFinger is neutral regarding the
scheme of such a URI: it could be an `acct` URI, an `http` or `https` URI, a
`mailto` URI, or some other scheme." (§4.5)

### Response

- **Media type**: `application/jrd+json` (registered in §10.2). A WebFinger
  resource MUST return a JRD as the representation if the client does not request
  another supported format via the HTTP `Accept` header.
- **HTTPS**: A WebFinger resource MUST be served over HTTPS. Clients MUST query
  using HTTPS only and MUST NOT fall back to HTTP. Certificates MUST be
  validated. Redirects MUST also be HTTPS, and the client MUST validate the new
  certificate.
- **CORS**: Public WebFinger servers MUST include the
  `Access-Control-Allow-Origin` header and SHOULD use the most permissive value:

  ```http
  Access-Control-Allow-Origin: *
  ```

  Intranet or sensitive servers SHOULD use a more restrictive policy.
- **Error semantics**:
  - Missing/malformed `resource` → `400`
  - Unknown `resource` → `404`
  - Servers MAY return `3xx` redirects (e.g. `307`) pointing at a third-party
    WebFinger host (RFC 7033 §7).

---

## JRD schema

A JSON Resource Descriptor is a JSON object with the following members
(RFC 7033 §4.4):

| Member       | Type                              | Required | Meaning                                                          |
|--------------|-----------------------------------|----------|------------------------------------------------------------------|
| `subject`    | string (URI)                      | SHOULD   | URI identifying the entity the JRD describes.                    |
| `aliases`    | array of strings (URIs)           | OPTIONAL | Other URIs identifying the same entity.                          |
| `properties` | object (URI → string \| null)     | OPTIONAL | Name/value pairs; names are URIs ("property identifiers").       |
| `links`      | array of link-relation objects    | OPTIONAL | Each describes one related resource.                             |

Notes:

- Clients MUST ignore unknown members (do not error on them).
- `subject` MAY differ from the requested `resource` — e.g. when a server
  prefers a canonical form or when the subject moved accounts. Trust the
  returned `subject` as authoritative for the response.

### Link-relation objects

Each entry in `links` is an object with these members (RFC 7033 §4.4.4):

| Member       | Type                                  | Required | Meaning                                                              |
|--------------|---------------------------------------|----------|----------------------------------------------------------------------|
| `rel`        | string (URI or registered token)      | MUST     | Compared via "Simple String Comparison" (RFC 3986 §6.2.1). Case-sensitive. |
| `type`       | string (media type, RFC 6838)         | OPTIONAL | Media type of the target.                                            |
| `href`       | string (URI)                          | OPTIONAL | URI of the target resource.                                          |
| `template`   | string (URI template with `{uri}`)    | OPTIONAL | Used in interaction relations (e.g. OStatus subscribe).              |
| `titles`     | object (lang-tag-or-`und` → string)   | OPTIONAL | Human-readable titles, keyed by RFC 5646 language tag or `und`.      |
| `properties` | object (URI → string \| null)         | OPTIONAL | Per-link metadata; names are URIs.                                   |

Examples of `titles`:

```json
"titles": {
  "en-us": "The Magical World of Steve",
  "fr":    "Le Monde Magique de Steve"
}
```

Order of `links` MAY indicate preference: when multiple links share the same
`rel`, the first is the user's preferred choice (§4.4.4). A JRD SHOULD NOT
contain more than one title with the same language tag inside one link object.

### Empty / minimal responses

A server MAY return a JRD with an empty `links` array or omit `links` entirely
when there are no links to return. An empty `subject`-only JRD is a valid
response to a successful lookup with no advertised services.

> **webfinger.js@3.0.3 caveat**: The library requires the parsed JRD to have a
> `links` property of type `object` and rejects with `WebFingerError: unknown
> response from server` otherwise. An RFC-valid subject-only JRD (no `links`
> field at all) will therefore fail. Servers targeted by webfinger.js clients
> should always emit `"links": []` rather than omitting the field.

---

## RFC 7565 — `acct:` URI

The `acct:` URI scheme (Standards Track, May 2015) identifies a user's account
at a service provider, independently of any interaction protocol. Defined ABNF
(RFC 7565 §7):

```
acctURI  = "acct" ":" userpart "@" host
userpart = unreserved / sub-delims
           0*( unreserved / pct-encoded / sub-delims )
```

- The string `acct:bob@example.com` identifies Bob's account at `example.com`,
  with **no implied protocol** for interaction.
- Unlike `mailto:`, `acct:` does **not** trigger email behavior. A click on
  `<a href='acct:bob@example.com'>` has no defined behavior — the URI is purely
  identifying.
- When the userpart itself contains an `@` (e.g. an email used as an account
  name at a different provider), the `@` MUST be percent-encoded:
  `acct:juliet%40capulet.example@shoppingsite.example`.
- At the time of writing, only WebFinger uses `acct:` in practice.

---

## RFC 6415 — host-meta fallback

`host-meta` (Standards Track, October 2011) predates WebFinger's
`/.well-known/webfinger` and was the original discovery surface. Modern clients
prefer WebFinger; host-meta is supported as a legacy fallback (in `webfinger.js`,
controlled by `uri_fallback: true`).

### Endpoint

```
GET /.well-known/host-meta
```

- Served as `application/xrd+xml` by default; servers SHOULD also offer JRD
  (`/.well-known/host-meta.json`).
- Document root is an `XRD` element. A `Subject` element SHOULD NOT appear at
  the host level; `Alias` is undefined and NOT RECOMMENDED.

### LRDD (Link-based Resource Descriptor Documents)

host-meta exposes resource-specific information via two mechanisms:

1. **Link templates** — a `<Link>` with a `template` attribute containing
   `{uri}` (and optionally other variables). The client substitutes the
   percent-encoded target URI to produce a per-resource link.
2. **LRDD documents** — a `<Link rel="lrdd" template="...{uri}..."/>` points at
   a per-resource descriptor. The client substitutes the resource URI and
   fetches the result, treating the response as the resource's descriptor.

Example host-meta document (RFC 6415 §1.1):

```xml
<?xml version='1.0' encoding='UTF-8'?>
<XRD xmlns='http://docs.oasis-open.org/ns/xri/xrd-1.0'>
  <Property type='http://protocol.example.net/version'>1.0</Property>
  <Link rel='copyright' href='http://example.com/copyright' />
  <Link rel='hub' template='http://example.com/hub' />
  <Link rel='lrdd'
        type='application/xrd+xml'
        template='http://example.com/lrdd?uri={uri}' />
</XRD>
```

### When `webfinger.js` uses it

With `uri_fallback: true`, webfinger.js tries each of these paths in order:

1. `/.well-known/webfinger?resource=acct:user@host`
2. `/.well-known/host-meta?resource=acct:user@host`
3. `/.well-known/host-meta.json?resource=acct:user@host`

The protocol used is `https://` for public hosts and `http://` for `localhost`
(the localhost selection is unconditional; `tls_only` does not gate it). When
`tls_only: false`, every URL above is retried over HTTP after the HTTPS attempt
fails. This catches deployments that have not migrated to RFC 7033 endpoints.

### Security note

"Applications utilizing the host-meta document where the authenticity of the
information is necessary MUST require the use of the HTTPS protocol and MUST NOT
produce a host-meta document using other means. In addition, such applications
MUST require that any redirection leading to the retrieval of a host-meta
document also utilize the HTTPS protocol." (RFC 6415 §5)

---

## Common link relations

| `rel` value                                               | Used for                                                                                       | Notes                                                                                  |
|-----------------------------------------------------------|------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| `self`                                                    | Fediverse: ActivityPub actor URI when paired with `type: application/activity+json`.            | IANA-registered (RFC 4287). **Not indexed** by webfinger.js — read from `result.object.links`. |
| `http://webfinger.net/rel/profile-page`                   | Human-readable profile page.                                                                   | Indexed by webfinger.js as `profile`. Usually `type: text/html`.                       |
| `http://webfinger.net/rel/avatar`                         | Avatar image URL.                                                                              | Indexed by webfinger.js as `avatar`.                                                   |
| `http://tools.ietf.org/id/draft-dejong-remotestorage`     | remoteStorage account discovery; `href` = storage root.                                        | Indexed by webfinger.js as `remotestorage`.                                            |
| `http://openid.net/specs/connect/1.0/issuer`              | OpenID Connect issuer URL for OIDC discovery.                                                  | Example in RFC 7033 §3.1. **Not indexed** — read from `result.object.links`.           |
| `http://ostatus.org/schema/1.0/subscribe`                 | OStatus / fediverse "remote follow" — uses `template` with `{uri}` placeholder.                | Used by Mastodon for remote-follow flows.                                              |
| `me`                                                      | IndieAuth-style identity link.                                                                 | webfinger.js folds this into the `profile` index alongside `profile-page`.             |

For relations not in webfinger.js's normalized index, always iterate
`result.object.links` directly:

```javascript
const issuer = result.object.links.find(
  (l) => l.rel === 'http://openid.net/specs/connect/1.0/issuer'
)?.href;
```

---

## remoteStorage link properties

remoteStorage (draft-dejong-remotestorage-26 §10) advertises exactly one
WebFinger link of the following shape:

```json
{
  "href": "<storage_root>",
  "rel":  "http://tools.ietf.org/id/draft-dejong-remotestorage",
  "properties": {
    "http://remotestorage.io/spec/version": "<storage_api>",
    "http://tools.ietf.org/html/rfc6749#section-4.2": "<auth-dialog>",
    "...": "..."
  }
}
```

| Property URI                                              | Meaning                                                                                  | Required          |
|-----------------------------------------------------------|------------------------------------------------------------------------------------------|-------------------|
| `http://remotestorage.io/spec/version`                    | Spec version, e.g. `"draft-dejong-remotestorage-26"`.                                    | Required          |
| `http://tools.ietf.org/html/rfc6749#section-4.2`          | OAuth 2.0 implicit-grant authorization endpoint URL, or `null` for Kerberos-only.        | Required          |
| `http://tools.ietf.org/html/rfc6750#section-2.3`          | `"true"` if bearer token via query parameter is supported, otherwise `null`.             | Optional          |
| `http://tools.ietf.org/html/rfc7233`                      | `"GET"` if Range GET requests are supported, otherwise `null`.                           | Optional          |
| `http://remotestorage.io/spec/web-authoring`              | Web-authoring FQDN, otherwise `null`.                                                    | Optional          |
| `http://tools.ietf.org/html/rfc6749#section-3.1`          | OAuth 2.0 authorization endpoint URL (PKCE, §10.1).                                      | Optional (PKCE)   |
| `http://tools.ietf.org/html/rfc6749#section-3.2`          | OAuth 2.0 token endpoint URL (PKCE, §10.1).                                              | Optional (PKCE)   |
| `http://tools.ietf.org/html/rfc7636`                      | Supported `code_challenge` methods. Servers MUST support `S256`.                          | Optional (PKCE)   |

Verbatim example (draft-dejong-remotestorage-26 §12.1):

```http
GET /.well-known/webfinger?resource=acct:michiel@michielbdejong.com HTTP/1.1
Host: michielbdejong.com
```

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Type: application/jrd+json

{
  "links":[{
    "href": "https://michielbdejong.com:7678/inbox",
    "rel": "post-me-anything"
  }, {
    "href": "https://michielbdejong.com/me.jpg",
    "rel": "avatar"
  }, {
    "href": "https://3pp.io:4439/storage/michiel",
    "rel": "http://tools.ietf.org/id/draft-dejong-remotestorage",
    "properties": {
      "http://remotestorage.io/spec/version": "draft-dejong-remotestorage-26",
      "http://tools.ietf.org/html/rfc6749#section-4.2": "https://3pp.io:4439/oauth/michiel",
      "http://tools.ietf.org/html/rfc6750#section-2.3": null,
      "http://tools.ietf.org/html/rfc7233": null,
      "http://remotestorage.io/spec/web-authoring": null
    }
  }]
}
```

---

## ActivityPub / Mastodon resolution flow

ActivityPub itself (W3C REC) does not define WebFinger; the W3C spec only
covers actor-ID dereferencing once the URI is known. The fediverse has
standardized on WebFinger to translate `@user@domain` handles to ActivityPub
actor URIs.

### Resolution algorithm

1. Parse `@user@domain` → `user`, `domain`.
2. Build the resource URI: `acct:user@domain`.
3. `GET https://{domain}/.well-known/webfinger?resource=acct%3A{user}%40{domain}`
   with `Accept: application/jrd+json`.
4. In the JRD `links`, find the entry where:
   - `rel === "self"`, AND
   - `type === "application/activity+json"` (or
     `application/ld+json; profile="https://www.w3.org/ns/activitystreams"`).
5. The `href` is the actor URI. Fetch it with
   `Accept: application/activity+json` to get the actor document.

> Because `self` is not in the webfinger.js indexed-link map, you must read it
> from `result.object.links` rather than `result.idx.links`.

### Verbatim Mastodon example

Request:

```
GET https://mastodon.social/.well-known/webfinger?resource=acct:gargron@mastodon.social
```

Response body:

```json
{
  "subject": "acct:Gargron@mastodon.social",
  "aliases": [
    "https://mastodon.social/@Gargron",
    "https://mastodon.social/users/Gargron"
  ],
  "links": [
    {
      "rel": "http://webfinger.net/rel/profile-page",
      "type": "text/html",
      "href": "https://mastodon.social/@Gargron"
    },
    {
      "rel": "self",
      "type": "application/activity+json",
      "href": "https://mastodon.social/users/Gargron"
    },
    {
      "rel": "http://ostatus.org/schema/1.0/subscribe",
      "template": "https://mastodon.social/authorize_interaction?uri={uri}"
    }
  ]
}
```

Implementation notes:

- `subject` may be returned with different casing than the request (`Gargron`
  vs `gargron`). Compare userpart case-insensitively when interoperating, and
  treat the returned `subject` as canonical.
- `aliases` typically contains both the human-friendly `/@user` URL and the
  canonical `/users/user` actor URL — either may dereference to the actor.
- The OStatus `subscribe` link uses `template` (not `href`); substitute the
  remote profile URI into `{uri}` to build a remote-follow URL.

---

## Security considerations

### From RFC 7033 §9

- **HTTPS** is REQUIRED. Clients MUST validate certificates and MUST NOT fall
  back to HTTP — including across redirects. Redirects MUST themselves be HTTPS.
- **CORS**: All security considerations of CORS apply. Public servers should
  use `Access-Control-Allow-Origin: *`; sensitive deployments MUST scope it.
- **Privacy / abuse**: WebFinger can be used to verify account existence
  (harvesting), correlate users across services, or fingerprint clients.
  Auto-querying based on user-supplied identifiers (e.g. an email address in a
  message) leaks the user's IP, client identity, and read-time to the queried
  host. Operators SHOULD apply rate limits.
- **Information reliability**: Servers cannot verify user-submitted data;
  clients MUST treat WebFinger data as untrusted input.
- **Access control**: Servers MAY return different JRDs based on
  authentication or network location.

### SSRF mitigations

WebFinger clients accept user-supplied hostnames and issue outbound HTTPS
requests, which is a textbook Server-Side Request Forgery vector: an attacker
can target internal IPs (`127.0.0.1`, `10.0.0.0/8`, `169.254.169.254` AWS IMDS,
etc.) by supplying an `acct:` URI whose host points at — or whose DNS resolves
to — such an address.

`webfinger.js` (and any production WebFinger client) MUST mitigate by:

- Resolving the hostname and rejecting any address in the loopback,
  link-local, private, broadcast, multicast, or IPv6 ULA ranges.
- Re-checking after **each** redirect (DNS rebinding defense).
- Enforcing `https://` and rejecting downgrades.
- Capping response size and applying connection / total timeouts.
- Limiting redirect chain length (webfinger.js: 3).
- Rejecting non-`application/jrd+json` (or non-JSON) responses rather than
  parsing arbitrary bytes.

`webfinger.js`'s defaults implement all of the above and follow the ActivityPub
"network access" guidance (block private/internal addresses, HTTPS only).

### Response size limits

RFC 7033 mandates no numeric upper bound, but unbounded responses are a DoS
vector. Real implementations cap at a few hundred kilobytes.

---

## Authoritative sources

- RFC 7033 — <https://datatracker.ietf.org/doc/html/rfc7033>
- RFC 7565 (`acct:` URI) — <https://datatracker.ietf.org/doc/html/rfc7565>
- RFC 6415 (host-meta) — <https://datatracker.ietf.org/doc/html/rfc6415>
- draft-dejong-remotestorage — <https://datatracker.ietf.org/doc/html/draft-dejong-remotestorage>
- Mastodon WebFinger docs — <https://docs.joinmastodon.org/spec/webfinger/>
- webfinger.js — <https://github.com/silverbucket/webfinger.js>
