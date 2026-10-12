# remoteStorage.js API Reference

Method signatures for the `RemoteStorage` instance and the `BaseClient` returned
from `rs.scope()`. For usage patterns, examples, and the event system see
[SKILL.md](../SKILL.md).

## Constructor options

`cache` (default: `true`), `logging` (`false`), `changeEvents` (object with
`local`, `window`, `remote`, `conflict` booleans; defaults `true, false, true, true` —
passing it replaces the whole object, so set all four), `discovery`
(`{ allowPrivateAddresses: true, timeout: 5000 }`; set `allowPrivateAddresses: false`
outside browsers), `cordovaRedirectUri`, `modules`.

## RemoteStorage methods

```javascript
rs.connect(userAddress)          // WebFinger + OAuth flow
rs.connect(userAddress, token)   // Connect with pre-acquired token (Node.js)
rs.disconnect()                  // Disconnect and clear cache
rs.reconnect()                   // Re-authorize (e.g. after 401)
rs.reauthorize()                 // Alias of reconnect() (for scope-change-required)
rs.setApiKeys({ dropbox, googledrive })  // Enable third-party backends
rs.startSync()                           // → Promise; no-op with a warning if cache:false
rs.stopSync()                            // Pause the periodic sync cycle
rs.setSyncInterval(10000)                // Foreground interval (ms)
rs.setBackgroundSyncInterval(60000)      // Background interval (ms)
```

Both interval setters throw unless the value is between 2000 and 3600000 ms.

## RemoteStorage events

`ready`, `connecting` (before WebFinger lookup), `authing` (before OAuth
redirect), `connected`, `disconnected`, `not-connected`, `error`,
`scope-change-required` (`{ authorizedScope, requestedScope, reauthorize }`),
`sync-started`, `sync-req-done` (per request, reports remaining queue items),
`sync-done` (`{ completed: boolean }`), `sync-interval-change`
(`{ oldValue, newValue }`), `wire-busy` / `wire-done` (a network request
starts / completes), `network-offline` / `network-online`.

> **Note:** Local writes (`storeObject`, `storeFile`, `remove`) are automatically
> queued for sync. Do not call `startSync()` after writing — it is only needed to
> manually request remote changes (e.g. on `visibilitychange` or a "sync now" button).

## BaseClient — data operations

Paths are relative to the client's scope. Paths ending in `/` denote folders.
`maxAge` (ms) controls cache freshness — pass `false` to skip network check.

```javascript
// Read
client.getListing(path?)               // → { 'file': true, 'dir/': true }
client.getObject(path, maxAge?)        // → object | null
client.getFile(path, maxAge?)          // → { data, contentType, revision }
client.getAll(path?, maxAge?)          // → { 'key': object, ... }

// Write — both store calls resolve with the new revision string
client.storeObject(typeAlias, path, obj)  // JSON with schema validation; type must
                                          // be declared; max once per second per path
client.storeFile(contentType, path, body) // body: string | ArrayBuffer | ArrayBufferView
client.remove(path)                       // Delete

// Utility
client.scope(path)                     // New client scoped to subpath
client.declareType(alias, schema)      // Register JSON Schema type (URI auto-generated)
client.declareType(alias, uri, schema) // ...with an explicit JSON-LD @context URI
client.validate(object)                // Validate via object['@context']; throws
                                       // SchemaNotFound if no schema is registered
client.getItemURL(path)                // → Promise<string | undefined>; public URL for
                                       // /public/ items; undefined when not connected.
                                       // remoteStorage + Dropbox only (not Google Drive)
```

## Change event payload

`client.on('change', ...)` events carry `path` (absolute), `relativePath`,
`origin` (`local` | `window` | `remote` | `conflict`), `oldValue` / `newValue`
(undefined on creation / deletion), and `oldContentType` / `newContentType`.
Conflict events additionally carry `lastCommonValue` and `lastCommonContentType`
(the state when local and remote last agreed); `oldValue` is the local version
and `newValue` the remote one.
