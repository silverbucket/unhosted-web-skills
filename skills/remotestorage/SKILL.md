---
name: remotestorage
description: Build offline-first, unhosted web apps with remoteStorage.js — user-owned
  data storage with automatic sync, zero backend infrastructure, and cross-app data
  interoperability. Use when creating local-first applications, implementing per-user
  cloud storage without a server, building data modules for the remoteStorage ecosystem,
  working with the remotestoragejs npm package, connecting to Dropbox or Google Drive
  as storage backends, or contributing to the remoteStorage.js library itself.
license: MIT
metadata:
  author: remotestorage
---

# remoteStorage.js

A JavaScript/TypeScript library for building unhosted web applications where users
own their data. Apps run entirely in the browser with no backend — data lives in the
user's own storage (a remoteStorage server, Dropbox, or Google Drive) and syncs
automatically across devices. Apps work offline by default.

## When to Use

Invoke when you need to:

- Build a web app that stores user data without running a backend server
- Add offline-first capability with automatic cloud sync to any browser app
- Let users bring their own storage (remoteStorage, Dropbox, Google Drive)
- Create data modules that multiple apps can share (contacts, bookmarks, etc.)
- Store and validate structured data using JSON Schema types
- Share public data via URLs without authentication
- Build progressive web apps where anonymous local use upgrades to synced storage
- Contribute to or debug the remoteStorage.js library itself

## Quick Start

### Install

```bash
npm install remotestoragejs
# Widget (optional, provides connect UI)
npm install remotestorage-widget
```

### Basic Setup

```javascript
import RemoteStorage from 'remotestoragejs';
import Widget from 'remotestorage-widget';

// 1. Create instance
const rs = new RemoteStorage({
  logging: false,
  changeEvents: {
    local:    true,
    window:   true,
    remote:   true,
    conflict: true
  }
});

// 2. Claim access to a category (creates /myfavoritedrinks/ namespace)
rs.access.claim('myfavoritedrinks', 'rw');

// 3. Enable caching for offline use
rs.caching.enable('/myfavoritedrinks/');

// 4. Attach the connect widget to your page
const widget = new Widget(rs);
widget.attach();  // or widget.attach('my-element-id')

// 5. Get a client scoped to your namespace
const client = rs.scope('/myfavoritedrinks/');

// 6. Store and retrieve data
await client.storeObject('drink', 'coffee', {
  name: 'Coffee',
  caffeine: true
});

const drink = await client.getObject('coffee');
console.log(drink.name); // 'Coffee'
```

The app works immediately in anonymous mode (local storage only). When the user
connects via the widget, data syncs to their remote storage automatically — no
code changes needed.

## Examples

### Example 1: Offline-first todo app with sync

```javascript
import RemoteStorage from 'remotestoragejs';
import Widget from 'remotestorage-widget';

const rs = new RemoteStorage({
  changeEvents: { local: true, window: true, remote: true, conflict: true }
});

rs.access.claim('todos', 'rw');
rs.caching.enable('/todos/');

const client = rs.scope('/todos/');

// Declare a type with JSON Schema for validation
client.declareType('todo-item', {
  type: 'object',
  properties: {
    id:        { type: 'string' },
    title:     { type: 'string' },
    completed: { type: 'boolean' },
    createdAt: { type: 'string', format: 'date-time' }
  },
  required: ['id', 'title', 'completed']
});

// Create
async function addTodo(title) {
  const id = crypto.randomUUID();
  await client.storeObject('todo-item', `items/${id}`, {
    id,
    title,
    completed: false,
    createdAt: new Date().toISOString()
  });
  return id;
}

// Read all
async function getAllTodos() {
  const items = await client.getAll('items/');
  return Object.values(items || {});
}

// Update
async function toggleTodo(id) {
  const todo = await client.getObject(`items/${id}`);
  if (todo) {
    todo.completed = !todo.completed;
    await client.storeObject('todo-item', `items/${id}`, todo);
  }
}

// Delete
async function removeTodo(id) {
  await client.remove(`items/${id}`);
}

// React to changes from any source (sync, other tabs, local writes)
client.on('change', (event) => {
  // event.origin: 'local' | 'remote' | 'window' | 'conflict'
  // event.relativePath: path within the scoped client
  // event.newValue / event.oldValue: the data
  if (event.origin !== 'window') {
    renderTodoList(); // re-render when data changes externally
  }
});

// Attach widget so user can connect their storage
const widget = new Widget(rs);
widget.attach();
```

### Example 2: Reusable data module with public sharing

Modules encapsulate data logic so multiple apps can share the same data format.
Each module receives a `privateClient` (scoped to `/<module>/`) and a
`publicClient` (scoped to `/public/<module>/`).

```javascript
// bookmarks-module.js
const Bookmarks = {
  name: 'bookmarks',
  builder: function (privateClient, publicClient) {

    privateClient.declareType('bookmark', {
      type: 'object',
      properties: {
        id:    { type: 'string' },
        title: { type: 'string' },
        url:   { type: 'string', format: 'uri' },
        tags:  { type: 'array', items: { type: 'string' } },
        notes: { type: 'string' }
      },
      required: ['id', 'title', 'url']
    });

    return {
      exports: {
        // Save a bookmark (private by default)
        async add(bookmark) {
          const id = bookmark.id || crypto.randomUUID();
          const record = { ...bookmark, id };
          await privateClient.storeObject('bookmark', id, record);
          return id;
        },

        // List all bookmarks
        async list() {
          const all = await privateClient.getAll('/');
          return Object.values(all || {});
        },

        // Get one bookmark
        async get(id) {
          return privateClient.getObject(id);
        },

        // Delete a bookmark
        async remove(id) {
          await privateClient.remove(id);
        },

        // Share a bookmark publicly (returns a URL anyone can access)
        async share(id) {
          const bookmark = await privateClient.getObject(id);
          if (!bookmark) throw new Error(`Bookmark ${id} not found`);
          await publicClient.storeObject('bookmark', id, bookmark);
          return publicClient.getItemURL(id);
        },

        // Unshare
        async unshare(id) {
          await publicClient.remove(id);
        }
      }
    };
  }
};

// --- Using the module ---
import RemoteStorage from 'remotestoragejs';

const rs = new RemoteStorage();
rs.addModule(Bookmarks);
rs.access.claim('bookmarks', 'rw');
rs.caching.enable('/bookmarks/');

// Module methods are available on rs.<moduleName>
await rs.bookmarks.add({
  title: 'remoteStorage spec',
  url: 'https://remotestorage.io/spec',
  tags: ['decentralization', 'storage']
});

const all = await rs.bookmarks.list();
const shareUrl = await rs.bookmarks.share(all[0].id);
console.log('Public URL:', shareUrl);
```

### Example 3: Dropbox / Google Drive as backend

Users without a remoteStorage server can connect Dropbox or Google Drive instead.
Register your API keys and the widget handles the rest.

```javascript
import RemoteStorage from 'remotestoragejs';
import Widget from 'remotestorage-widget';

const rs = new RemoteStorage();

// Register third-party backend API keys
rs.setApiKeys({
  dropbox: 'YOUR_DROPBOX_APP_KEY',
  googledrive: 'YOUR_GOOGLE_CLIENT_ID'
});

rs.access.claim('notes', 'rw');
rs.caching.enable('/notes/');

// Widget now shows remoteStorage, Dropbox, and Google Drive options
const widget = new Widget(rs);
widget.attach();

// The storage API is identical regardless of backend
const client = rs.scope('/notes/');
await client.storeFile('text/plain', 'hello.txt', 'Hello from any backend!');

// Detect which backend connected
rs.on('connected', () => {
  console.log('Backend:', rs.backend);
  // 'remotestorage' | 'dropbox' | 'googledrive'
});
```

**Dropbox**: Register a scoped app at Dropbox App Console with `account_info.read`,
`files.metadata.*`, `files.content.*` permissions.
**Google Drive**: Enable Drive API in Google Cloud Console, create OAuth 2.0
browser credentials.

## Data Modules

Modules encapsulate data logic so multiple apps can share the same data. See
Example 2 above for a complete module implementation. The key pattern:

- `name` determines the storage path (`/<name>/`) — changing it orphans data
- `builder(privateClient, publicClient)` receives scoped clients
- `privateClient` writes to `/<name>/`, `publicClient` to `/public/<name>/`
- Return an `exports` object — its methods become available on `rs.<name>`
- Register with `rs.addModule(module)`, then `rs.access.claim('<name>', 'rw')`

Keep module names lowercase and hyphen-separated.

## Data Type System

Declare types using JSON Schema. `storeObject()` validates automatically — it
rejects if data doesn't match the schema. The type alias is stored as `@context`
on the object for schema-aware retrieval.

```javascript
client.declareType('contact', {
  type: 'object',
  properties: {
    name:  { type: 'string' },
    email: { type: 'string', format: 'email' },
    tags:  { type: 'array', items: { type: 'string' } }
  },
  required: ['name']
});

// Validates against schema — rejects on failure
await client.storeObject('contact', 'alice', { name: 'Alice', email: 'alice@example.com' });

// Manual validation
const result = client.validate({ name: 123 }); // result.valid === false
```

## Caching Strategies

Caching determines what gets stored locally and how sync behaves. There are three
strategies — choose based on your app's needs:

| Strategy | Behavior | Best for |
|----------|----------|----------|
| **ALL** | Proactively fetches and caches everything in a path. Checks ETags on sync. | Main app data you need offline |
| **SEEN** | Caches only items the app has read or written. Default behavior. | Large datasets where you only need a subset |
| **FLUSH** | Caches outgoing writes only; clears after successful remote sync. | Write-heavy paths (logs, analytics) |

```javascript
rs.caching.enable('/tasks/');           // ALL strategy
rs.caching.set('/logs/', 'FLUSH');      // Write-only caching
rs.caching.disable('/archive/');        // Explicitly FLUSH

// Check current strategy for a path
rs.caching.checkPath('/tasks/');        // 'ALL'
```

**Caching strategies are not persisted** — they reset on page reload. Always set
them during app initialization, before the first sync runs.

## Event System

### RemoteStorage instance events

| Event | Fires when |
|-------|-----------|
| `connected` | User connected storage |
| `disconnected` | User disconnected |
| `not-connected` | Running in anonymous/local mode |
| `ready` | All features loaded |
| `sync-started` / `sync-done` | Sync cycle begins / completes |
| `network-offline` / `network-online` | Network status changes |
| `error` | See Error Handling below |

### BaseClient change events

The `change` event is the primary way to keep UI in sync with data. Each event
carries `relativePath`, `newValue`, `oldValue`, and `origin`:

| Origin | Fires when |
|--------|-----------|
| `local` | Page load, for each cached item |
| `remote` | Sync pulls in server changes |
| `window` | Current browser context writes data |
| `conflict` | Local and remote versions diverge (remote wins by default) |

Enable only the origins you need via `changeEvents` in the constructor.

**Recommended loading pattern** — use `getAll()` for initial render, then
`change` events for incremental updates:

```javascript
const all = await client.getAll('items/');
renderAll(Object.entries(all || {}));

client.on('change', (event) => {
  if (event.origin === 'conflict') {
    console.warn('Conflict on', event.relativePath);
  }
  event.newValue ? upsertInUI(event.relativePath, event.newValue)
                 : removeFromUI(event.relativePath);
});
```

## API Reference

### Constructor options

`cache` (default: `true`), `logging` (`false`), `changeEvents` (object with
`local`, `window`, `remote`, `conflict` booleans), `cordovaRedirectUri`, `modules`.

### RemoteStorage methods

```javascript
rs.connect(userAddress)          // WebFinger + OAuth flow
rs.connect(userAddress, token)   // Connect with pre-acquired token (Node.js)
rs.disconnect()                  // Disconnect and clear cache
rs.reconnect()                   // Re-authorize (e.g. after 401)
rs.setApiKeys({ dropbox, googledrive })  // Enable third-party backends
rs.startSync() / rs.stopSync()           // Control sync cycle
rs.setSyncInterval(10000)                // Foreground interval (ms)
rs.setBackgroundSyncInterval(60000)      // Background interval (ms)
```

### BaseClient — data operations

Paths are relative to the client's scope. Paths ending in `/` denote folders.
`maxAge` (ms) controls cache freshness — pass `false` to skip network check.

```javascript
// Read
client.getListing(path?)               // → { 'file': true, 'dir/': true }
client.getObject(path, maxAge?)        // → object | null
client.getFile(path, maxAge?)          // → { data, contentType, revision }
client.getAll(path?, maxAge?)          // → { 'key': object, ... }

// Write
client.storeObject(typeAlias, path, obj)  // JSON with schema validation
client.storeFile(contentType, path, body) // Raw data (string, ArrayBuffer)
client.remove(path)                       // Delete

// Utility
client.scope(path)                     // New client scoped to subpath
client.declareType(alias, schema)      // Register JSON Schema type
client.validate(object)               // Validate against declared schemas
client.getItemURL(path)                // Public URL (if in /public/)
```

## Third-Party Backends

```javascript
rs.setApiKeys({ dropbox: 'APP_KEY', googledrive: 'CLIENT_ID' });
```

**Dropbox**: Register a scoped app. Needs `account_info.read`, `files.metadata.*`,
`files.content.*`. Limitations: 150 MB max file, no `getItemURL()`, case-insensitive paths.

**Google Drive**: Enable Drive API, create OAuth 2.0 browser credentials.
Limitations: no `getItemURL()`, requires Google verification for production.

## Error Handling

```javascript
rs.on('error', (error) => {
  switch (error.name) {
    case 'Unauthorized':
      // 401/403 — token expired or revoked
      rs.reconnect();  // re-trigger OAuth
      break;
    case 'DiscoveryError':
      // WebFinger lookup failed — bad user address or server down
      showMessage('Could not find storage server. Check the address.');
      break;
    case 'SyncError':
      // Sync failed — usually transient network issue
      console.warn('Sync error, will retry:', error.message);
      break;
    default:
      console.error('remoteStorage error:', error);
  }
});

// storeObject rejects if data fails schema validation
try {
  await client.storeObject('contact', 'bob', { name: 42 });
} catch (validationError) {
  console.error('Invalid data:', validationError.message);
}
```

## Node.js Usage

Works in Node 18+ with two key differences: no IndexedDB/localStorage (uses
**in-memory storage** — data lost on exit; use Deno for persistence), and no
browser OAuth (pass a pre-acquired token to `rs.connect(address, token)`).

## Requirements

- Browser (any modern browser with IndexedDB) or Node.js 18+
- npm package: `remotestoragejs`
- Optional: `remotestorage-widget` for the connect UI

## Related Packages

- `remotestoragejs` — Core library (this skill)
- `remotestorage-widget` — Drop-in connect/sync UI widget
- `rs.js` community modules — Shared data modules (contacts, bookmarks, etc.)
