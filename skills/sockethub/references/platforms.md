# Sockethub Platform Reference

## IRC Platform (`irc`)

**Type:** Persistent
**Package:** `@sockethub/platform-irc`

### Credentials

IRC supports token-based and password-based authentication. Tokens are the
preferred method -- they can be scoped, rotated, and revoked without changing
account passwords. The `password` and `token` fields are mutually exclusive;
providing both causes a schema validation error.

#### Token via SASL PLAIN (preferred)

Use for personal access tokens such as Libera.Chat NickServ tokens. When a
`token` is provided without an explicit `saslMechanism`, SASL PLAIN is used
automatically.

```javascript
sc.socket.emit('credentials', {
  '@context': sc.contextFor('irc'),
  type: 'credentials',
  actor: { id: 'mynick@irc.libera.chat' },
  object: {
    type: 'credentials',
    nick: 'mynick',
    server: 'irc.libera.chat',
    token: process.env.IRC_TOKEN,
    port: 6697,
    secure: true
  }
});
```

#### Token via SASL OAUTHBEARER

Use for OAuth 2.0 access tokens (RFC 7628), for example with chat.sr.ht.
The schema requires `token` when `saslMechanism` is `'OAUTHBEARER'`.

```javascript
sc.socket.emit('credentials', {
  '@context': sc.contextFor('irc'),
  type: 'credentials',
  actor: { id: 'mynick@chat.sr.ht' },
  object: {
    type: 'credentials',
    nick: 'mynick',
    server: 'chat.sr.ht',
    token: process.env.IRC_OAUTH_TOKEN,
    saslMechanism: 'OAUTHBEARER',
    port: 6697,
    secure: true
  }
});
```

#### Password via SASL PLAIN (legacy)

Traditional password authentication. Set `sasl: true` to enable SASL
negotiation with the server.

```javascript
sc.socket.emit('credentials', {
  '@context': sc.contextFor('irc'),
  type: 'credentials',
  actor: { id: 'mynick@irc.libera.chat' },
  object: {
    type: 'credentials',
    nick: 'mynick',
    server: 'irc.libera.chat',
    password: process.env.IRC_PASSWORD,
    port: 6697,
    secure: true,
    sasl: true
  }
});
```

#### No authentication

For servers that allow unauthenticated connections:

```javascript
sc.socket.emit('credentials', {
  '@context': sc.contextFor('irc'),
  type: 'credentials',
  actor: { id: 'mynick@irc.example.net' },
  object: {
    type: 'credentials',
    nick: 'mynick',
    server: 'irc.example.net',
    port: 6667
  }
});
```

#### Full credential object schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `'credentials'` |
| `nick` | string | Yes | IRC nickname |
| `server` | string | Yes | IRC server hostname |
| `token` | string | No | Personal access token or OAuth token (mutually exclusive with `password`) |
| `password` | string | No | Server password (mutually exclusive with `token`) |
| `username` | string | No | IRC username (defaults to `nick`) |
| `port` | number | No | Server port (default: 6667, or 6697 when `secure: true`) |
| `secure` | boolean | No | Use TLS (default: `true`) |
| `sasl` | boolean | No | Enable SASL authentication (auto-enabled when `token` or `password` is set) |
| `saslMechanism` | string | No | `'PLAIN'` (default) or `'OAUTHBEARER'` |
| `allowInvalidCert` | boolean | No | Default `false`. Sets `rejectUnauthorized: false`, i.e. **disables TLS certificate verification entirely** (enables MITM). Only for trusted self-signed networks; never in production |

**Validation rules:**
- `password` and `token` cannot both be present
- `saslMechanism: 'PLAIN'` requires either `password` or `token`
- `saslMechanism: 'OAUTHBEARER'` requires `token`

### Supported Actions

| Action | Description | Requires Target |
|--------|-------------|-----------------|
| `connect` | Connect to IRC server using stored credentials | No |
| `join` | Join a channel | Yes (room) |
| `leave` | Leave a channel | Yes (room) |
| `send` | Send a PRIVMSG to a channel or user (`/me ` prefix sends a CTCP ACTION, `/notice ` prefix sends a NOTICE; prefix is stripped) | Yes (room or person) |
| `update` | Update nick (`object.type: 'address'`) or topic (`object.type: 'topic'`) | Depends on update type |
| `query` | Query channel members (`object.type: 'attendance'`) or list channels (`object.type: 'channels'`, optional `object.name` mask) | Yes |
| `disconnect` | Disconnect from IRC server | No |

There is no `announce` verb with a handler: send a NOTICE via `send` with
content prefixed `/notice `. A regular message (PRIVMSG) to a channel requires
a prior `join`; otherwise the ack returns
`cannot send message to a channel of which you've not first joined.` A
`/notice` is sent raw without that check, but join anyway: `+n` channels drop
notices from non-members at the IRC server.

### IRC-Specific Behavior

- Actor IDs follow the pattern `nick@server` (e.g., `mynick@irc.libera.chat`)
- Channel targets use `#channel@server` (e.g., `#sockethub@irc.libera.chat`)
- Incoming messages are converted via `@sockethub/irc2as` translator
- Nick changes emit `update` messages with `object.type: 'address'`
- Topic changes emit `update` messages with `object.type: 'topic'`
- User join/leave events emit corresponding `join`/`leave` messages
- MODE changes emit `update` messages with `object.type: 'presence'` carrying
  a `role` (`owner`, `admin`, `participant`, or `member`); IRC AWAY status is
  not translated
- Channel user lists come as `attendance` type objects (with per-nick roles)
- Incoming CTCP ACTION is delivered with `object.type: 'me'`; incoming NOTICE
  as a plain `message`
- Sockets that submit identical credentials (non-empty `password` or `token`)
  for the same actor share one platform instance. Anonymous credentials are
  session-private; a second anonymous connection with the same nick fails with
  the IRC server's own error, e.g. `unable to connect to server: nicknames unavailable`.

### Example: Send a channel message

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'send',
  actor: { id: 'mynick@irc.libera.chat', type: 'person' },
  target: { id: '#sockethub@irc.libera.chat', type: 'room' },
  object: { type: 'message', content: 'Hello channel!' }
}, (ack) => {
  if (ack?.error) console.error('Send failed:', ack.error);
});
```

### Example: Send a NOTICE

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'send',
  actor: { id: 'mynick@irc.libera.chat', type: 'person' },
  target: { id: '#sockethub@irc.libera.chat', type: 'room' },
  object: { type: 'message', content: '/notice Deploy starting in 5 minutes' }
}, (ack) => {
  if (ack?.error) console.error('Notice failed:', ack.error);
});
```

### Example: Change nickname

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'update',
  actor: { id: 'oldnick@irc.libera.chat', type: 'person' },
  target: { id: 'newnick@irc.libera.chat', type: 'person', name: 'newnick' },
  object: { type: 'address' }
});
```

### Example: Query channel members

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'query',
  actor: { id: 'mynick@irc.libera.chat', type: 'person' },
  target: { id: '#sockethub@irc.libera.chat', type: 'room', name: '#sockethub' },
  object: { type: 'attendance' }
});
```

---

## XMPP Platform (`xmpp`)

**Type:** Persistent
**Package:** `@sockethub/platform-xmpp`

### Credentials

XMPP uses a single `password` field for all authentication. Some deployments
accept a bearer-style token in the SASL PLAIN password slot as a compatibility
mode — pass the token string as `password` only when the server explicitly
supports it.

Dedicated token SASL mechanisms (ejabberd X-OAUTH2, Prosody OAUTHBEARER,
Prosody community X-TOKEN, SASL2 FAST) are not implemented by this client.

```javascript
sc.socket.emit('credentials', {
  '@context': sc.contextFor('xmpp'),
  type: 'credentials',
  actor: { id: 'user@jabber.org' },
  object: {
    type: 'credentials',
    userAddress: 'user@jabber.org',
    password: process.env.XMPP_PASSWORD,  // or a token, if your server accepts it
    resource: 'web'
  }
});
```

#### Full credential object schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `'credentials'` |
| `userAddress` | string | Yes | Full JID (e.g., `user@jabber.org`) |
| `password` | string | Yes | Account password (or token if the server explicitly supports it) |
| `resource` | string | Yes | Resource identifier (e.g., `'web'`, `'phone'`) |
| `server` | string | No | Override server hostname (extracted from `userAddress` if omitted) |
| `port` | number | No | Override server port (appended to service URL) |

### Supported Actions

| Action | Description | Requires Target |
|--------|-------------|-----------------|
| `connect` | Connect to XMPP server | No |
| `join` | Join a MUC (multi-user chat) room | Yes (room) |
| `leave` | Leave a MUC room | Yes (room) |
| `send` | Send a message | Yes (person or room) |
| `update` | Update presence or status | No |
| `request-friend` | Send a roster subscription request | Yes (person) |
| `make-friend` | Intended to accept a subscription, but currently sends the same `presence type="subscribe"` as `request-friend` | Yes (person) |
| `remove-friend` | Remove a roster subscription | Yes (person) |
| `query` | Service discovery against `target.id`: `object.type: 'attendance'` (default, disco#items) or `'room-info'` (disco#info). No roster query. | Yes |
| `disconnect` | Disconnect from XMPP server | No |

### XMPP-Specific Behavior

- Messages to rooms (`target.type: 'room'`) are sent as `groupchat` type
- Messages to users (`target.type: 'person'`) are sent as `chat` type
- Presence updates support values: `away`, `chat`, `dnd`, `xa`, `offline`, `online`
- Message correction is supported via `object['xmpp:replace']` with the original message id
- Unrecoverable errors (auth failures, policy violations) require re-issuing
  `connect`; stored credentials remain available (sliding TTL, default 7 days)
- Recoverable errors (network issues) trigger automatic reconnection
- Connection errors arrive on `message` as
  `{ type: 'connect', error, object: { type: 'connect', status: 'reconnecting' | 'failed', condition } }`

### Example: Send a direct message

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('xmpp'),
  type: 'send',
  actor: { id: 'user@jabber.org', type: 'person' },
  target: { id: 'friend@jabber.org', type: 'person' },
  object: { type: 'message', content: 'Hello!' }
});
```

### Example: Update presence

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('xmpp'),
  type: 'update',
  actor: { id: 'user@jabber.org' },
  object: {
    type: 'presence',
    presence: 'away',
    content: 'Be right back'
  }
});
```

### Example: Manage roster

```javascript
// Send friend request
sc.socket.emit('message', {
  '@context': sc.contextFor('xmpp'),
  type: 'request-friend',
  actor: { id: 'user@jabber.org' },
  target: { id: 'newcontact@jabber.org' }
});

// Accept friend request (caveat: currently sends presence type="subscribe",
// identical to request-friend, rather than "subscribed")
sc.socket.emit('message', {
  '@context': sc.contextFor('xmpp'),
  type: 'make-friend',
  actor: { id: 'user@jabber.org' },
  target: { id: 'newcontact@jabber.org' }
});
```

---

## Feeds Platform (`feeds`)

**Type:** Stateless
**Package:** `@sockethub/platform-feeds`

### Supported Actions

| Action | Description |
|--------|-------------|
| `fetch` | Fetch and parse an RSS/Atom feed URL |

Supports RSS 2.0, Atom 1.0, and RSS 1.0/RDF formats. Uses `podparse` for parsing.

### Example: Fetch a feed

Use `actor.type: 'feed'` (the client fills in `person` otherwise). The optional
`object` accepts only `since` (RFC 3339 date-time; entries published at or
after it) and `limit` (integer >= 1, applied after `since`, feed order kept).

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('feeds'),
  type: 'fetch',
  actor: { id: 'https://blog.example.com/feed.xml', type: 'feed' },
  object: { since: '2026-01-01T00:00:00Z', limit: 10 }
});
```

### Response Format

The response is a single top-level `collection` whose `items` are `post`
activities (one per entry):

```javascript
{
  id: '...',                    // request id, or null
  '@context': [...],
  type: 'collection',
  summary: 'Example Blog',      // feed title, or 'Unknown Feed'
  totalItems: 10,
  items: [
    {
      type: 'post',
      actor: { id: 'https://blog.example.com/feed.xml', type: 'feed', name: 'Example Blog', link: '...' },
      object: {
        type: 'article',          // or 'note'
        id: 'https://blog.example.com/article',
        title: 'Article Title',
        brief: 'Short summary...',
        content: 'Full content...',
        contentType: 'html',      // or 'text'
        url: 'https://blog.example.com/article',
        published: '2026-03-15T10:00:00Z',
        datenum: 1773568800000
      }
    }
  ]
}
```

---

## Metadata Platform (`metadata`)

**Type:** Stateless
**Package:** `@sockethub/platform-metadata`

### Supported Actions

| Action | Description |
|--------|-------------|
| `fetch` | Extract Open Graph and page metadata from a URL |

Uses `open-graph-scraper` to extract OG tags, page titles, descriptions, and images.

### Example: Fetch page metadata

Use `actor.type: 'website'`. No `object` is accepted on the request.

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('metadata'),
  type: 'fetch',
  actor: { id: 'https://example.com/article', type: 'website' }
});
```

### Response Format

The response echoes the `fetch` with a `page` object. `image` is an array
(from open-graph-scraper), so read `msg.object.image?.[0]?.url`.

```javascript
{
  '@context': [...],
  type: 'fetch',
  actor: { id: 'https://example.com/article', type: 'website' },
  object: {
    type: 'page',
    title: 'Page Title',
    name: 'Site Name',
    description: 'Page description from OG tags',
    image: [{ url: 'https://example.com/og-image.jpg', width: 1200, height: 630, alt: '...' }],
    url: 'https://example.com/article',
    language: 'en',
    favicon: 'https://example.com/favicon.ico',
    charset: 'utf-8',
    video: { url: '...', thumbnail: '...', width: 0, height: 0, duration: 0 }  // optional
  }
}
```

---

## Dummy Platform (`dummy`)

**Type:** Stateless
**Package:** `@sockethub/platform-dummy`

An echo/testing platform for development and integration testing. Echoes back
any message sent to it. Useful for verifying Sockethub server setup without
needing external services.

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('dummy'),
  type: 'echo',
  actor: { id: 'test-user' },
  object: { type: 'message', content: 'echo test' }
});
```
