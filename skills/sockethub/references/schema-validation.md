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

There is no global object vocabulary: each platform's `messages` schema
enumerates the `object` shapes it accepts from clients.

| Platform | Accepted `object.type` values (client to platform) |
|----------|---------------------------------------------------|
| `irc` | `message`, `topic`, `address`, `attendance`, `channels` (`/me` and `/notice` are `message` content prefixes; `me` is only emitted inbound) |
| `xmpp` | `message`, `presence`, `attendance`, `room-info` |
| `feeds` | No `type`; `object` is optional `{ since, limit }` |
| `metadata` | No `object` accepted |

Credentials use `object.type: 'credentials'` on the `credentials` event, not
`message`. Platform responses use their own shapes (e.g. feeds `collection`
with `post` items, metadata `page`).

`ObjectTypesList` in `@sockethub/schemas` is the actor/target vocabulary
(`feed`, `person`, `room`, `service`, `platform`, `website`, `address`,
`calendar`, `addressBook`), not the message object vocabulary.

Using an invalid type (e.g., `'Note'`) causes validation failure -- the server
returns the error via the ack callback (if one was provided), but the error
message may not clearly indicate that the object type was the problem.

## Exported Schemas

`@sockethub/schemas` (AJV-based) exports:

```typescript
// Context constants and helpers
AS2_BASE_CONTEXT_URL, SOCKETHUB_BASE_CONTEXT_URL, PLATFORM_CONTEXT_PREFIX
buildCanonicalContext(platformIdOrUrl)      // -> three-element @context array
platformIdFromContext(context)              // platform id from a @context array, or undefined
resolvePlatformId(msg)                      // platform id from an activity, or null

// Platform schema registry
addPlatformSchema(schema, platformId)
addPlatformContext(platformId, contextUrl)
getPlatformSchema(platformId)               // compiled AJV validator, or undefined

// Validators: all return an error string, or "" when valid
validatePlatformSchema(schema)
validateCredentials(activity)
validateActivityStream(activity)
validateActivityStreamResponse(activity)    // "" when the platform has no responses schema
validateServiceDescriptor(descriptor)
validateSockethubConfig(config)
setValidationErrorOptions(options)          // tune AJV error text

// Normalization
normalizeActivityStream(activity)           // expand string actor/target refs

// Schema objects and type lists
PlatformSchema, ServiceDescriptorSchema, ActivityStreamSchema, SockethubConfigSchema
ObjectTypesList, ObjectTypesSchema          // actor/target object vocabulary
InternalObjectTypesList
```

Example:

```javascript
import { validateActivityStream } from '@sockethub/schemas';

const err = validateActivityStream(activity);
if (err) throw new Error(err);   // "" means valid
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
The server encrypts credentials before storing them in Redis, keyed by session ID,
platform, and actor ID. They are not logged or broadcast to other sockets; the
`credentials` ack callback does receive the normalized credentials activity back.

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
