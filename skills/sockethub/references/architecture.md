# Sockethub Architecture

## Multi-Process Model

Sockethub runs as multiple cooperating processes:

```
┌─────────────┐     ┌─────────────────────────────┐     ┌──────────────────┐
│ Web Clients │◄───►│ Sockethub Server             │◄───►│ Redis (BullMQ)   │
│ (Socket.IO) │     │ Express + Socket.IO          │     │ Job Queue        │
└─────────────┘     │ Validation, Encryption,      │     │ Credential Store │
                    │ Routing, Rate Limiting       │     └────────┬─────────┘
                    └─────────────────────────────┘              │
                              ▲ IPC                              │
                              │                                  ▼
                    ┌─────────┴──────────────────────────────────────┐
                    │ Platform Workers (isolated processes)           │
                    │ ┌─────┐ ┌──────┐ ┌───────┐ ┌──────────┐      │
                    │ │ IRC │ │ XMPP │ │ Feeds │ │ Metadata │ ... │
                    │ └─────┘ └──────┘ └───────┘ └──────────┘      │
                    └────────────────────────────────────────────────┘
```

## Message Flow

### Client to Platform (outbound)

1. Client sends ActivityStreams message via Socket.IO
2. Server validates message against platform schema
3. Server encrypts message with session encryption key
4. Encrypted job is enqueued to Redis via BullMQ
5. Platform worker picks up job from its queue
6. Worker decrypts and processes the message
7. Worker executes protocol-specific action (e.g., IRC PRIVMSG)

### Platform to Client (inbound)

1. Platform worker receives protocol event (e.g., incoming IRC message)
2. Worker converts event to ActivityStreams format
3. Worker sends message to server via IPC (bypasses Redis for low latency)
4. Server routes message to the correct client session via Socket.IO

## Session Lifecycle

1. **Connect**: Client establishes Socket.IO connection, server assigns unique session ID
2. **Initialize**: Client calls `await sc.ready()`, server sends schema registry
3. **Authenticate**: Client sends credentials via `credentials` event, server encrypts
   and stores in Redis keyed by session ID + credential ID
4. **Operate**: Client sends/receives ActivityStreams messages
5. **Disconnect**: Session cleanup, credentials removed from Redis, platform connections
   closed if no other sessions reference them

## Credential Encryption

All credentials are encrypted using `@sockethub/crypto`:

- Each session gets a unique encryption key generated at connection time
- Credentials are encrypted before storage in Redis
- Platform workers decrypt credentials when processing jobs
- Keys are never stored persistently -- lost on server restart
- Redis key pattern: `sockethub:{parentId}:data-layer:credentials-store:{sessionId}:{credentialId}`

## Job Queue

BullMQ manages the message queue in Redis:

- Queue naming: `sockethub:{parentId}:data-layer:queue:{platformId}`
- Each platform has its own dedicated queue
- Jobs are encrypted before enqueuing
- Failed jobs trigger `failed` events back to the client
- Completed jobs trigger `completed` events

## Platform Workers

### Lifecycle

1. **Spawn**: Server spawns worker process for each enabled platform
2. **Register**: Worker registers its supported actions with the server
3. **Heartbeat**: Worker sends periodic heartbeats (configurable via
   `SOCKETHUB_PLATFORM_HEARTBEAT_INTERVAL_MS`)
4. **Process**: Worker polls its BullMQ queue for jobs
5. **Timeout**: If heartbeat is missed beyond `SOCKETHUB_PLATFORM_TIMEOUT_MS`,
   server marks worker as unhealthy and respawns

### Persistent vs Stateless

**Persistent platforms** (IRC, XMPP):
- Maintain long-lived protocol connections
- `isInitialized()` returns `true` only when connected
- Track connection state per actor
- Auto-reconnect on temporary failures
- Connections shared across sessions using the same actor credentials

**Stateless platforms** (Feeds, Metadata):
- No persistent connections
- `isInitialized()` always returns `true`
- Each job is independent
- No connection management needed

## Rate Limiting

- Configurable per-client rate limit (default: 100 events/second)
- Applied at the Socket.IO event level before validation
- Excess messages are dropped with an error event sent to the client

## Scaling

- Multiple Sockethub server instances can share a Redis cluster
- Platform workers scale independently per platform
- Redis handles job distribution across workers
- Socket.IO adapter (e.g., Redis adapter) enables multi-instance client routing
