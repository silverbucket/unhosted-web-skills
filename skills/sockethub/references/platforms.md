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
| `send` | Send a message to a channel or user | Yes (room or person) |
| `update` | Update nick or topic | Depends on update type |
| `query` | Query channel or user information | Yes |
| `announce` | Send an announcement | Yes |
| `disconnect` | Disconnect from IRC server | No |

### IRC-Specific Behavior

- Actor IDs follow the pattern `nick@server` (e.g., `mynick@irc.libera.chat`)
- Channel targets use `#channel@server` (e.g., `#sockethub@irc.libera.chat`)
- Incoming messages are converted via `@sockethub/irc2as` translator
- Nick changes emit `update` messages with `object.type: 'address'`
- Topic changes emit `update` messages with `object.type: 'topic'`
- User join/leave events emit corresponding `join`/`leave` messages
- Presence changes (away, back) emit messages with `object.type: 'presence'`
- Channel user lists come as `attendance` type objects
- Messages starting with `/me` are sent as `object.type: 'me'` (IRC action)
- Messages starting with `/NOTICE` are sent as IRC NOTICE commands

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

XMPP uses a single `password` field for all authentication. For servers that
accept bearer-style tokens in the SASL PLAIN password slot, pass the token
string as `password`. This is common practice for many XMPP servers.

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
    password: process.env.XMPP_TOKEN,  // accepts token or password
    resource: 'web'
  }
});
```

#### Full credential object schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `'credentials'` |
| `userAddress` | string | Yes | Full JID (e.g., `user@jabber.org`) |
| `password` | string | Yes | Account password or bearer token |
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
| `make-friend` | Accept a roster subscription | Yes (person) |
| `remove-friend` | Remove a roster subscription | Yes (person) |
| `query` | Query roster or room info | Optional |
| `disconnect` | Disconnect from XMPP server | No |

### XMPP-Specific Behavior

- Messages to rooms (`target.type: 'room'`) are sent as `groupchat` type
- Messages to users (`target.type: 'person'`) are sent as `chat` type
- Presence updates support values: `away`, `chat`, `dnd`, `xa`, `offline`, `online`
- Message correction is supported via `object['xmpp:replace']` with the original message id
- Unrecoverable errors (auth failures, policy violations) require manual reconnection
- Recoverable errors (network issues) trigger automatic reconnection

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

// Accept friend request
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

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('feeds'),
  type: 'fetch',
  actor: { id: 'https://blog.example.com/feed.xml' }
});
```

### Response Format

The response is a Collection of Create activities:

```javascript
{
  '@context': [...],
  type: 'fetch',
  actor: { id: 'https://blog.example.com/feed.xml' },
  object: {
    type: 'collection',
    items: [
      {
        type: 'create',
        object: {
          type: 'message',
          title: 'Article Title',
          url: 'https://blog.example.com/article',
          content: 'Article summary...',
          published: '2026-03-15T10:00:00Z'
        }
      }
    ]
  }
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

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('metadata'),
  type: 'fetch',
  actor: { id: 'https://example.com/article' }
});
```

### Response Format

```javascript
{
  '@context': [...],
  type: 'fetch',
  actor: { id: 'https://example.com/article' },
  object: {
    type: 'message',
    title: 'Page Title',
    description: 'Page description from OG tags',
    image: 'https://example.com/og-image.jpg',
    url: 'https://example.com/article'
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
