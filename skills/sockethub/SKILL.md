---
name: sockethub
description: Bridges web applications to messaging protocols (IRC, XMPP, RSS/Atom) via
  ActivityStreams. Use when building chat clients, connecting to IRC channels, sending
  XMPP messages, fetching RSS feeds, or creating protocol-agnostic messaging applications.
license: LGPL-3.0
metadata:
  author: sockethub
---

# Sockethub

A polyglot messaging gateway that translates ActivityStreams messages to
protocol-specific formats, enabling browser JavaScript to communicate with IRC,
XMPP, and feed services through a unified API.

## When to Use

Invoke when you need to:

- Connect a web application to IRC networks or XMPP servers
- Build a browser-based chat client for multiple protocols
- Fetch and parse RSS/Atom feeds from JavaScript
- Send messages across different messaging platforms
- Create protocol-agnostic messaging applications
- Handle real-time bidirectional communication with legacy protocols
- Generate metadata previews for shared URLs

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

```bash
# Install globally
bun add -g sockethub@alpha

# Start server (requires Redis)
sockethub --port 10550
```

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
await sc.ready(); // loads schema registry; messages sent before ready() are queued up to configured limits

// Listen for messages
sc.socket.on('message', (msg) => console.log(msg));

// Send ActivityStreams message
sc.socket.emit('message', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/irc/v1.jsonld'
  ],
  type: 'send',
  actor: { id: 'myuser@irc.libera.chat', type: 'person' },
  target: { id: '#channel@irc.libera.chat', type: 'room' },
  object: { type: 'message', content: 'Hello!' }
});
```

## Examples

### Example 1: Connect to IRC and join a channel

```javascript
// Set credentials
sc.socket.emit('credentials', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/irc/v1.jsonld'
  ],
  type: 'credentials',
  actor: { id: 'mynick@irc.libera.chat' },
  object: {
    type: 'credentials',
    nick: 'mynick',
    server: 'irc.libera.chat',
    port: 6697,
    secure: true
  }
});

// Connect
sc.socket.emit('message', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/irc/v1.jsonld'
  ],
  type: 'connect',
  actor: { id: 'mynick@irc.libera.chat', type: 'person' }
});

// Join channel
sc.socket.emit('message', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/irc/v1.jsonld'
  ],
  type: 'join',
  actor: { id: 'mynick@irc.libera.chat', type: 'person' },
  target: { id: '#sockethub@irc.libera.chat', type: 'room' }
});
```

### Example 2: Send XMPP message

```javascript
// Set XMPP credentials
sc.socket.emit('credentials', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/xmpp/v1.jsonld'
  ],
  type: 'credentials',
  actor: { id: 'user@jabber.org' },
  object: {
    type: 'credentials',
    userAddress: 'user@jabber.org',
    password: 'secret',
    resource: 'web'
  }
});

// Connect and send
sc.socket.emit('message', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/xmpp/v1.jsonld'
  ],
  type: 'connect',
  actor: { id: 'user@jabber.org', type: 'person' }
});

sc.socket.emit('message', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/xmpp/v1.jsonld'
  ],
  type: 'send',
  actor: { id: 'user@jabber.org', type: 'person' },
  target: { id: 'friend@jabber.org', type: 'person' },
  object: { type: 'message', content: 'Hello from Sockethub!' }
});
```

### Example 3: Fetch RSS feed

```javascript
sc.socket.emit('message', {
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/feeds/v1.jsonld'
  ],
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
  '@context': [
    'https://www.w3.org/ns/activitystreams',
    'https://sockethub.org/ns/context/v1.jsonld',
    'https://sockethub.org/ns/context/platform/metadata/v1.jsonld'
  ],
  type: 'fetch',
  actor: { id: 'https://example.com/article' }
});

// Response contains Open Graph data
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
  --public.host <host>   Public-facing hostname (reverse proxy)
  --public.port <port>   Public-facing port
  --public.path <path>   Public-facing base path
  --rateLimit <number>   Max events per second per client (default: 100)
  --sentry.dsn <dsn>     Sentry DSN for error reporting

Environment Variables:
  PORT                                      Server port
  HOST                                      Server hostname
  REDIS_URL                                 Redis connection string
  LOG_LEVEL                                 Logging verbosity (debug, info, warn, error)
  SOCKETHUB_PLATFORM_HEARTBEAT_INTERVAL_MS  Worker heartbeat interval
  SOCKETHUB_PLATFORM_TIMEOUT_MS             Worker timeout threshold
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

All messages follow ActivityStreams 2.0 structure with Sockethub context:

```javascript
{
  '@context': [
    'https://www.w3.org/ns/activitystreams',         // AS2 base
    'https://sockethub.org/ns/context/v1.jsonld',    // Sockethub base
    'https://sockethub.org/ns/context/platform/irc/v1.jsonld'  // Platform
  ],
  type: 'send',                                // Action type
  actor: { id: 'user@server', type: 'person' }, // Who is acting
  target: { id: 'room@server', type: 'room' }, // Target of action
  object: { type: 'message', content: '...' }  // Payload
}
```

**Valid object types:** `message`, `me`, `credentials`, `attendance`, `presence`,
`topic`, `address`

If `actor` is provided as a string, Sockethub expands it using any previously
saved ActivityObject with the same id. If none exists, it expands to `{ id }`.

The client automatically injects `@context` if missing (via `contextFor(platform)`),
but explicitly providing it is recommended.

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

// Initialize -- recommended to await before sending messages
// Messages sent before ready() are queued (up to the above limits) and flushed afterward
await sc.ready();

// Properties
sc.socket           // Underlying Socket.IO instance
sc.ActivityStreams   // ActivityStreams helper library

// Methods
sc.clearCredentials()  // Remove stored credentials

// Events
sc.socket.on('message', handler)      // Incoming platform messages
sc.socket.on('completed', handler)    // Job completion acknowledgement
sc.socket.on('failed', handler)       // Job failure notification
sc.socket.on('connect', handler)      // Socket.IO transport connected
sc.socket.on('disconnect', handler)   // Socket.IO transport disconnected
sc.socket.emit('message', activity)   // Send ActivityStreams message
sc.socket.emit('credentials', creds)  // Set platform credentials
```

### IRC Credentials

```javascript
{
  type: 'credentials',
  nick: 'nickname',
  server: 'irc.libera.chat',
  port: 6697,
  secure: true,
  password: 'optional',
  sasl: false
}
```

### XMPP Credentials

```javascript
{
  type: 'credentials',
  userAddress: 'user@domain',
  password: 'secret',
  resource: 'device-identifier'
}
```

## Requirements

- Bun >= 1.2.4 (or Node.js 18+)
- Redis >= 6.0

## Related Packages

- `@sockethub/client` - Browser/Node client library
- `@sockethub/server` - Core server implementation
- `@sockethub/schemas` - Schema registry and validation
- `@sockethub/activity-streams` - ActivityStreams utilities
- `@sockethub/crypto` - Session credential encryption
- `@sockethub/data-layer` - Redis/BullMQ data abstraction
- `@sockethub/logger` - Structured Winston logging
- `@sockethub/platform-irc` - IRC platform
- `@sockethub/platform-xmpp` - XMPP platform
- `@sockethub/platform-feeds` - RSS/Atom platform
- `@sockethub/irc2as` - IRC-to-ActivityStreams translator
