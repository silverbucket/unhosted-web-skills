# Sockethub Schema and Validation Reference

## Schema Registry

The schema registry is loaded when the client calls `await sc.ready()`. It contains
validation rules for all enabled platforms, including supported action types and
object schemas. The registry is provided by `@sockethub/schemas`.

## @context Structure

Every ActivityStreams message uses a three-element `@context` array. Use
`sc.contextFor(platform)` to build it automatically from server metadata:

```javascript
sc.contextFor('irc')
// Returns:
[
  'https://www.w3.org/ns/activitystreams',                              // 1. AS2 base
  'https://sockethub.org/ns/context/v1.jsonld',                         // 2. Sockethub base
  'https://sockethub.org/ns/context/platform/irc/v1.jsonld'             // 3. Platform-specific
]
```

**Context URL constants:**
- `AS2_BASE_CONTEXT_URL`: `https://www.w3.org/ns/activitystreams`
- `SOCKETHUB_BASE_CONTEXT_URL`: `https://sockethub.org/ns/context/v1.jsonld`
- `PLATFORM_CONTEXT_PREFIX`: `https://sockethub.org/ns/context/platform/`

The `contextFor()` method is preferred over manually constructing the array because
it derives context URLs from the server's schema registry, staying correct across
versions.

## Valid Object Types

Outbound messages (client to platform) must use one of these object types.
Server responses may also include AS2 core types like `collection` and `create`
(e.g., feed fetch results).

| Type | Description | Usage |
|------|-------------|-------|
| `message` | Chat message or content | Sending/receiving text messages |
| `me` | Action message (IRC /me) | Emoting or action descriptions |
| `credentials` | Authentication data | Setting platform credentials |
| `attendance` | User list for a room | Channel/room user listings |
| `presence` | Online/away status | Presence updates |
| `topic` | Room topic/subject | Channel topic changes |
| `address` | Identity/nick change | Nick changes, identity updates |

Using an invalid type (e.g., `'Note'`) causes validation failure -- the server
returns the error via the ack callback (if one was provided), but the error
message may not clearly indicate that the object type was the problem.

## Exported Schemas

`@sockethub/schemas` exports:

```typescript
// Schema objects
PlatformSchema          // Platform configuration structure
ActivityObjectSchema    // Individual activity objects
ActivityStreamSchema    // Full ActivityStreams messages
SockethubConfigSchema   // Server configuration

// Validation functions
validateSchema(schema, data)      // Validate data against a schema
validateCredentials(creds)        // Validate credential objects
validateActivity(activity)        // Validate ActivityStreams messages
validateObject(obj)               // Validate activity objects

// Platform utilities
getPlatformId(context)            // Extract platform ID from @context
resolvePlatform(id)               // Resolve platform module
registerPlatform(id, schema)      // Register a platform schema

// Type lists
ObjectTypesList                   // Public object types
InternalObjectTypesList           // Internal-only types
```

## Message Validation Flow

1. Client sends message via Socket.IO
2. Server extracts platform ID from `@context` array
3. Server validates message structure against `ActivityStreamSchema`
4. Server validates object type against platform's allowed types
5. Server validates action type (e.g., `send`, `join`) against platform's supported actions
6. If validation passes, message is encrypted and enqueued
7. If validation fails, the error is returned via the ack callback

## Credential Validation

Credentials follow a platform-specific schema but share common structure:

```javascript
{
  '@context': sc.contextFor('irc'),   // Required: platform context
  type: 'credentials',               // Required: must be 'credentials'
  actor: { id: 'user@host' },        // Required: who the credentials belong to
  object: {
    type: 'credentials',             // Required: must be 'credentials'
    // ... platform-specific fields (see platforms.md)
  }
}
```

Credentials are sent via the dedicated `credentials` event, not the `message` event.
The server encrypts credentials before storing them in Redis, keyed by session ID
and actor ID. Credentials never leave the server in plain text.

### IRC Credential Validation

The IRC credential schema enforces mutual exclusivity between `password` and
`token` via a JSON Schema `not` constraint. Additional `allOf`/`anyOf` rules
ensure that:

- When `saslMechanism` is `'PLAIN'`, either `password` or `token` is present
- When `saslMechanism` is `'OAUTHBEARER'`, `token` is present

### XMPP Credential Validation

XMPP requires `userAddress`, `password`, and `resource`. The `password` field
is passed directly to `xmpp.js` for SASL negotiation. Some deployments accept
a token string in the password slot as a compatibility mode, but this is
server-dependent — dedicated token SASL mechanisms are not implemented.

## Acknowledgement Callbacks

Message sends support an optional acknowledgement callback as the third argument
to `emit()`. The callback receives an ack object:

```javascript
sc.socket.emit('message', activity, (ack) => {
  if (ack?.error) {
    console.error('Server rejected message:', ack.error);
  } else {
    console.log('Message accepted:', ack);
  }
});
```

The ack callback fires once the platform worker finishes processing the job
(e.g., after the IRC server confirms delivery). BullMQ `completed`/`failed`
events are consumed internally by the server — they are not forwarded to
the client as separate Socket.IO events. Asynchronous platform messages
(incoming chat, presence updates) arrive on the `message` event.
