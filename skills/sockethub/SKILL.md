---
name: sockethub
description: Use when building with Sockethub, ActivityStreams messaging,
  browser-to-IRC/XMPP gateways, RSS/Atom fetching, metadata previews, or
  multi-protocol chat clients. Also use whenever the user mentions Sockethub
  directly, wants browser JavaScript to talk to IRC or XMPP, or needs
  protocol-specific auth hidden behind one consistent message format.
license: LGPL-3.0
metadata:
  author: sockethub
---

# Sockethub

A protocol gateway for the web. It lets browser or server-side JavaScript
talk to IRC, XMPP, feeds, and metadata endpoints through one
ActivityStreams-based API.

## Core Guidance

- Prefer `sc.contextFor(platform)` over hand-writing context arrays unless the
  user explicitly needs the raw URLs.
- Prefer `await sc.ready()` before sending anything. The client queues messages
  before readiness, but ready-first is easier to reason about.
- Treat credentials as runtime secrets. Never hardcode, commit, paste into
  logs, or echo back real passwords or tokens. Use placeholders or
  environment-backed variables in examples (e.g. `process.env.IRC_TOKEN`).
- Prefer tokens over passwords when the platform supports them. IRC has an
  explicit `token` field. XMPP does not -- it accepts a `password` field, and
  some deployments allow a token string in that field as a compatibility mode.
- Always show the ack callback form of `socket.emit()` when wiring connect or
  credentials flows. Sockethub returns callback payloads containing `error`
  for rejected actions.
- Call `sc.clearCredentials()` when switching users or performing explicit
  logout so the client does not replay stale credentials on reconnect.

## When to Use

Invoke when you need to:

- Connect a web application to IRC networks or XMPP servers
- Build a browser-based chat client for multiple protocols
- Fetch and parse RSS/Atom feeds from JavaScript
- Send messages across different messaging platforms
- Create protocol-agnostic messaging applications
- Handle real-time bidirectional communication with legacy protocols
- Generate metadata previews for shared URLs
- Explain Sockethub credential flows, connection lifecycle, or ActivityStreams
  message shapes

## Architecture

Sockethub runs as a multi-process system:

- **Server process**: Express + Socket.IO hub handling validation, encryption, and routing
- **Platform workers**: Isolated processes per protocol (IRC, XMPP, Feeds, Metadata)
- **Redis/BullMQ**: Job queue for encrypted message routing between server and workers
- **IPC**: Platform-to-client messages bypass Redis for low latency

Each client Socket.IO connection gets a unique session with its own encryption key.
Credentials are encrypted at rest in Redis via `@sockethub/crypto`. Platform crashes
are isolated -- one worker failing does not affect others.

**Platform types:**
- **Persistent** (IRC, XMPP): Maintain long-lived connections, require authentication,
  auto-reconnect on failure
- **Stateless** (Feeds, Metadata): No persistent connections, always ready, ideal for
  HTTP-based operations

See `references/architecture.md` for detailed internals.

## Quick Start

### Server Setup

For packaged usage:

```bash
npm install -g sockethub
sockethub --examples
```

For current `master` or local development:

```bash
git clone https://github.com/sockethub/sockethub.git
cd sockethub
bun install
bun run dev
```

Sockethub listens on `http://localhost:10550` by default and requires Redis.

Or use Docker Compose:

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  sockethub:
    image: sockethub/sockethub:alpha
    ports: ["10550:10550"]
    environment:
      REDIS_URL: redis://redis:6379
    depends_on: [redis]
```

```bash
docker compose up -d
```

### Client Connection

```javascript
import SockethubClient from '@sockethub/client';
import { io } from 'socket.io-client';

const socket = io('http://localhost:10550', { path: '/sockethub' });
const sc = new SockethubClient(socket, { initTimeoutMs: 5000 });

// Listen for readiness and errors
sc.socket.on('ready', (info) => {
  console.log('Sockethub ready:', info.reason, info.platforms);
});
sc.socket.on('init_error', (e) => {
  console.warn('Sockethub init failed:', e.error);
});

// Wait for schema registry to load (messages sent before ready() are queued)
await sc.ready();

// Listen for incoming messages
sc.socket.on('message', (msg) => console.log('Received:', msg));
```

## Connection Flow

Every persistent platform (IRC, XMPP) follows the same three-step pattern:
**set credentials, connect, then operate**. Skipping a step or reordering them
will fail. Stateless platforms (Feeds, Metadata) skip directly to operate.

### Step 1: Set Credentials

Credentials are sent via the dedicated `credentials` event (not `message`).
The server encrypts and stores them for the session. Each credential is keyed
by the `actor.id`, so you can have multiple identities per platform.

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

### Step 2: Connect

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'connect',
  actor: { id: 'mynick@irc.libera.chat', type: 'person' }
});
```

### Step 3: Operate

Pass an ack callback to learn the outcome of each action; treat any payload
with an `error` property as a failure.

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'join',
  actor: { id: 'mynick@irc.libera.chat', type: 'person' },
  target: { id: '#sockethub@irc.libera.chat', type: 'room' }
}, (result) => {
  if (result?.error) {
    console.error('Join failed:', result.error);
    return;
  }
  // Channel is joined; safe to query attendance, render UI, etc.
});
```

## Authentication

Both IRC and XMPP support token-based authentication, which is the preferred
approach. Tokens avoid transmitting reusable passwords and can be scoped or
revoked independently.

### IRC

IRC supports three auth methods. `password` and `token` are mutually
exclusive -- providing both causes a schema validation error.

- **Token via SASL PLAIN** (preferred): set `token` to a personal access token
  (e.g. Libera.Chat NickServ). SASL PLAIN is used automatically.
- **Token via SASL OAUTHBEARER**: set `token` and `saslMechanism: 'OAUTHBEARER'`
  for OAuth 2.0 bearer tokens (RFC 7628), e.g. chat.sr.ht.
- **Password via SASL PLAIN** (legacy): set `password` and `sasl: true`.

### XMPP

XMPP uses a single `password` field for all authentication. For deployments
that accept bearer-style tokens in the SASL PLAIN password slot, pass the
token string as `password`. Dedicated token SASL mechanisms (ejabberd X-OAUTH2,
Prosody OAUTHBEARER, Prosody X-TOKEN, SASL2 FAST) are not implemented.

See `references/platforms.md` for full credential schemas, validation rules,
and per-platform code examples.

## Examples

### Example 1: IRC -- Connect, join, and send a message

```javascript
import SockethubClient from '@sockethub/client';
import { io } from 'socket.io-client';

const socket = io('http://localhost:10550', { path: '/sockethub' });
const sc = new SockethubClient(socket);
await sc.ready();

const actorId = 'mynick@irc.libera.chat';

// 1. Set credentials (token-based)
sc.socket.emit('credentials', {
  '@context': sc.contextFor('irc'),
  type: 'credentials',
  actor: { id: actorId },
  object: {
    type: 'credentials',
    nick: 'mynick',
    server: 'irc.libera.chat',
    token: process.env.IRC_TOKEN,
    port: 6697,
    secure: true
  }
});

// 2. Connect
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'connect',
  actor: { id: actorId, type: 'person' }
});

// 3. Join channel
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'join',
  actor: { id: actorId, type: 'person' },
  target: { id: '#sockethub@irc.libera.chat', type: 'room' }
});

// 4. Send a message (with acknowledgement callback)
sc.socket.emit('message', {
  '@context': sc.contextFor('irc'),
  type: 'send',
  actor: { id: actorId, type: 'person' },
  target: { id: '#sockethub@irc.libera.chat', type: 'room' },
  object: { type: 'message', content: 'Hello from Sockethub!' }
}, (ack) => {
  if (ack?.error) console.error('Send failed:', ack.error);
});
```

### Example 2: XMPP -- Send a direct message

```javascript
const actorId = 'user@jabber.org';

sc.socket.emit('credentials', {
  '@context': sc.contextFor('xmpp'),
  type: 'credentials',
  actor: { id: actorId },
  object: {
    type: 'credentials',
    userAddress: actorId,
    password: process.env.XMPP_TOKEN,
    resource: 'web'
  }
});

sc.socket.emit('message', {
  '@context': sc.contextFor('xmpp'),
  type: 'connect',
  actor: { id: actorId, type: 'person' }
});

sc.socket.emit('message', {
  '@context': sc.contextFor('xmpp'),
  type: 'send',
  actor: { id: actorId, type: 'person' },
  target: { id: 'friend@jabber.org', type: 'person' },
  object: { type: 'message', content: 'Hello from Sockethub!' }
});
```

### Example 3: Fetch RSS feed

Stateless platforms need no credentials or connect step.

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('feeds'),
  type: 'fetch',
  actor: { id: 'https://example.com/feed.rss' }
});

// Response is a Collection of Create activities with feed entries
sc.socket.on('message', (msg) => {
  if (msg['@context']?.some(c => c.includes('/feeds/'))) {
    msg.object.items.forEach(entry => {
      console.log(entry.object.title, entry.object.url);
    });
  }
});
```

### Example 4: Fetch URL metadata

```javascript
sc.socket.emit('message', {
  '@context': sc.contextFor('metadata'),
  type: 'fetch',
  actor: { id: 'https://example.com/article' }
});

sc.socket.on('message', (msg) => {
  if (msg['@context']?.some(c => c.includes('/metadata/'))) {
    console.log(msg.object.title, msg.object.description, msg.object.image);
  }
});
```

## CLI Reference

```
sockethub [options]

Options:
  --port <number>        Server port (default: 10550, env: PORT)
  --host <string>        Server host (default: localhost, env: HOST)
  --config, -c <path>    Path to sockethub.config.json
  --examples             Enable example pages at /examples
  --info                 Display runtime information
  --redis.url <url>      Redis connection URL (env: REDIS_URL)
  --platforms <list>     Comma-separated enabled platforms
  --rateLimit <number>   Max events per second per client (default: 100)

Environment: PORT, HOST, REDIS_URL, LOG_LEVEL,
  SOCKETHUB_PLATFORM_HEARTBEAT_INTERVAL_MS, SOCKETHUB_PLATFORM_TIMEOUT_MS
```

## Supported Platforms

| Platform | Type | Actions | Description |
|----------|------|---------|-------------|
| IRC (`irc`) | Persistent | connect, join, leave, send, update, query, disconnect | Internet Relay Chat |
| XMPP (`xmpp`) | Persistent | connect, join, leave, send, update, request-friend, make-friend, remove-friend, query, disconnect | Extensible Messaging and Presence |
| Feeds (`feeds`) | Stateless | fetch | RSS 2.0, Atom 1.0, RSS 1.0/RDF |
| Metadata (`metadata`) | Stateless | fetch | Open Graph and page metadata extraction |

See `references/platforms.md` for per-platform details and credential schemas.

## ActivityStreams Message Format

All messages follow ActivityStreams 2.0 structure with Sockethub context.
Use `sc.contextFor(platform)` to build the `@context` array automatically
from server-provided metadata:

```javascript
{
  '@context': sc.contextFor('irc'),         // builds the three-element array
  type: 'send',                             // action type
  actor: { id: 'user@server', type: 'person' },  // who is acting
  target: { id: 'room@server', type: 'room' },   // target of action
  object: { type: 'message', content: '...' }     // payload
}
```

Providing `@context` manually is supported, but `contextFor()` is recommended
because it derives context URLs from the server's schema registry and stays
correct across versions. See `references/schema-validation.md` for the full
context URL structure.

**Valid object types:** `message`, `me`, `credentials`, `attendance`, `presence`,
`topic`, `address`

If `actor` is provided as a string, Sockethub expands it using any previously
saved ActivityObject with the same id. If none exists, it expands to `{ id }`.

See `references/schema-validation.md` for full schema details.

## API Reference

### SockethubClient

```javascript
import SockethubClient from '@sockethub/client';

// Constructor options
const sc = new SockethubClient(socket, {
  initTimeoutMs: 5000,      // Schema registry load timeout
  maxQueuedOutbound: 100,   // Max queued messages before ready()
  maxQueuedAgeMs: 30000     // Max age for queued messages
});

// Initialize -- messages sent before ready() are queued and flushed afterward
await sc.ready();

// Properties
sc.socket           // Underlying Socket.IO instance
sc.ActivityStreams   // ActivityStreams helper library

// Methods
sc.contextFor('irc')   // Build canonical @context array for a platform
sc.clearCredentials()  // Remove stored credentials

// Events
sc.socket.on('message', handler)         // Incoming platform messages
sc.socket.on('connect', handler)         // Socket.IO transport connected
sc.socket.on('connect_error', handler)   // Socket.IO connection failed
sc.socket.on('disconnect', handler)      // Socket.IO transport disconnected
sc.socket.on('ready', handler)           // Schema registry loaded; client is ready
sc.socket.on('init_error', handler)      // Client initialization failed
sc.socket.on('client_error', handler)    // Client-side validation error
sc.socket.emit('message', activity, cb)  // Send ActivityStreams message; cb(result) on response
sc.socket.emit('credentials', creds, cb) // Set platform credentials; cb(result) on response
```

Treat any ack callback payload containing an `error` property as failure.
Not every inbound `message` is a response to your last request — platforms
also push asynchronously (e.g. incoming IRC `PRIVMSG`, XMPP presence).

## Session Sharing

Persistent platforms can attach multiple sockets to one running platform
instance when they target the same actor and the credentials pass share
validation. Share is allowed only when credentials include a non-empty
`password` or `token`. Username-only anonymous credentials are not shareable
and may return `username already in use`.

## What to Avoid

- Do not present Sockethub as a direct browser-to-IRC/XMPP library. It is a
  server-side protocol gateway with a Socket.IO client API.
- Do not recommend committing credentials to config files or source code.
- Do not imply XMPP supports a first-class `token` field. Upstream does not.
- Do not lead with raw context URLs when `sc.contextFor(...)` is enough.
- Do not omit Redis when explaining server startup.
- Do not ignore callback or `failed`-event error paths.
- If the user pastes a real secret, redact it in any echoed examples or
  summaries.

## Requirements

- Bun >= 1.2.4 (or Node.js 18+)
- Redis >= 6.0

## Related Packages

`@sockethub/client`, `@sockethub/server`, `@sockethub/schemas`,
`@sockethub/activity-streams`, `@sockethub/crypto`, `@sockethub/data-layer`,
`@sockethub/platform-irc`, `@sockethub/platform-xmpp`,
`@sockethub/platform-feeds`, `@sockethub/irc2as`
