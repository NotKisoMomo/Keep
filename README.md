# Keep

A thin, opinionated wrapper around [ProfileStore](https://github.com/MadStudioRoblox/ProfileStore) that adds promise-based session management, multi-schema support, dot-path mutations, per-key signals, and auto-save — all behind a single module.

---

## Table of Contents

- [Setup](#setup)
- [Keep.Configure](#keepconfigure)
- [Keep.SetSchema](#keepsetschema)
- [Session Lifecycle](#session-lifecycle)
  - [Keep.StartPlayerSession](#keepstartplayersession)
  - [Keep.AwaitSession](#keepawaitsession)
  - [Keep.WaitForSession](#keepwaitforsession)
  - [Keep.EndPlayerSession](#keependplayersession)
  - [Keep.EndAllSessions](#keependallsessions)
  - [Keep.IsSessionActive](#keepissessionactive)
  - [Owner Object](#owner-object)
- [Events](#events)
- [Handle — Reading](#handle--reading)
- [Handle — Writing](#handle--writing)
- [Handle — Observing](#handle--observing)
- [Handle — Metadata](#handle--metadata)
- [Keep.Debug](#keepdebug)

---

## Setup

```lua
local Keep = require(ReplicatedStorage.Keep)

-- 1. Configure (optional — all keys have defaults)
Keep.Configure({
    AutoSave         = true,
    AutoSaveInterval = 60,
    Reconcile        = true,
    LogLevel         = "Warn",
})

-- 2. Register schemas
Keep.SetSchema("Account", {
    Level    = 1,
    XP       = 0,
    Currency = 0,
})
Keep.SetSchema("Inventory", {
    Items = {},
})

-- 3. Start sessions on player join
Players.PlayerAdded:Connect(function(player)
    Keep.StartPlayerSession(player)
        :andThen(function(stores, owner)
            local account   = stores.Account
            local inventory = stores.Inventory
            -- ready to use
        end)
end)
```

> Sessions are automatically cleaned up on `PlayerRemoving` and `game:BindToClose` — no manual teardown needed for normal flows.

---

## Keep.Configure

Override default module settings. Call before any sessions are started.

```lua
Keep.Configure(config: table)
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `StorageType` | `string` | `"Single"` | ProfileStore storage type. |
| `StudioSeparate` | `boolean` | `false` | Prefixes store names with `Studio_` in Studio to avoid polluting production data. |
| `AutoSave` | `boolean` | `true` | Periodically saves all active profiles. |
| `AutoSaveInterval` | `number` | `60` | Seconds between automatic saves. |
| `SessionLockRetries` | `number` | `5` | Passed to ProfileStore session lock logic. |
| `SessionLockInterval` | `number` | `15` | Seconds between session lock retries. |
| `LogLevel` | `string` | `"Warn"` | `"None"` \| `"Warn"` \| `"Verbose"` |
| `Reconcile` | `boolean` | `true` | Fills missing keys from schema defaults on load. |
| `Migration` | `function?` | `nil` | Called with `(data, oldVersion)` when the stored version is older than `Version`. Must return the migrated data table. |
| `Version` | `number` | `1` | Current schema version. Increment to trigger migrations. |

---

## Keep.SetSchema

Register a data schema. Each schema becomes a separate ProfileStore and its own channel accessor on the `Keep` module.

```lua
-- Named schema — exposes Keep[name] as a channel accessor
Keep.SetSchema(name: string, schema: table)

-- Default schema — exposes Keep.Account
Keep.SetSchema(schema: table)
```

After registration, `Keep[name]` becomes a function that accepts a player or userId and returns their handle:

```lua
Keep.SetSchema("Account", { Level = 1, XP = 0 })

local handle = Keep.Account(player)
```

> **Note:** Channel accessors return `nil` if no session is active for that player. Always start a session with `StartPlayerSession` first, or use `AwaitSession` to wait for it.

---

## Session Lifecycle

### Keep.StartPlayerSession

```lua
Keep.StartPlayerSession(plrOrId: Player | number, overrides?: table) -> Promise<stores, owner>
```

Loads all registered schemas for a player, registers the session, and starts auto-save. Resolves with `(stores: { [schemaName]: Handle }, owner: Owner)`.

If a session is already active for this player, resolves immediately with the existing handles.

| Param | Type | Description |
|-------|------|-------------|
| `plrOrId` | `Player \| number` | The player instance or their numeric userId. |
| `overrides` | `table?` | Deep-merged into schema defaults before the session starts. |

```lua
Keep.StartPlayerSession(player)
    :andThen(function(stores, owner)
        print(owner.SessionId, owner.StartTime)
        stores.Account:Increment("XP", 50)
    end)
    :catch(function(err)
        warn("Failed to load:", err)
    end)
```

---

### Keep.AwaitSession

```lua
Keep.AwaitSession(plrOrId: Player | number) -> Promise<stores, owner>
```

Queues until the player's session is active, then resolves with `(stores, owner)`. Useful in systems that initialize independently of `PlayerAdded`.

---

### Keep.WaitForSession

```lua
Keep.WaitForSession(plrOrId: Player | number) -> (stores, owner) | (nil, nil)
```

Synchronous yield — polls until the session is active. Returns `(stores, owner)` or `nil, nil` if the userId cannot be resolved. Prefer `AwaitSession` in production code.

---

### Keep.EndPlayerSession

```lua
Keep.EndPlayerSession(plrOrId: Player | number) -> Promise<owner>
```

Ends the session, saves and releases all profiles, stops auto-save, clears the handle cache, and fires `OnSessionEnd`. Resolves with the owner object.

---

### Keep.EndAllSessions

```lua
Keep.EndAllSessions() -> Promise
```

Calls `EndPlayerSession` for every active userId. Used internally in `BindToClose`.

---

### Keep.IsSessionActive

```lua
Keep.IsSessionActive(plrOrId: Player | number) -> boolean
```

Returns whether a session is currently active for the given player or userId.

---

### Owner Object

The owner table is resolved from `StartPlayerSession` and passed to event callbacks.

| Field | Type | Description |
|-------|------|-------------|
| `Player` | `Player?` | The Player instance, or `nil` if started by userId only. |
| `UserId` | `number` | Numeric userId. |
| `SessionId` | `string` | Unique identifier for this session. |
| `StartTime` | `number` | Unix timestamp of session start. |
| `StoreNames` | `string[]` | List of schema names active in this session. |

---

## Events

All events return a connection object with a `:Disconnect()` method.

```lua
Keep.OnSessionStart(fn: (stores: { [string]: Handle }, owner: Owner) -> ())  -> Connection
Keep.OnSessionEnd(fn: (owner: Owner) -> ())                                  -> Connection
Keep.OnSaveComplete(fn: (owner: Owner) -> ())                                -> Connection
```

| Event | Fires when... |
|-------|--------------|
| `OnSessionStart` | A session finishes loading all schemas. |
| `OnSessionEnd` | A session has been fully released. |
| `OnSaveComplete` | Any save completes — auto or manual. |

```lua
Keep.OnSessionStart(function(stores, owner)
    print("Loaded:", owner.UserId)
end)

Keep.OnSaveComplete(function(owner)
    print("Auto-saved:", owner.UserId)
end)
```

---

## Handle — Reading

All read methods are available on the handle returned from a session. Indexing an unknown key falls through to `profile.Data` via the proxy.

### Proxy read

```lua
local xp = handle.XP  -- reads profile.Data.XP directly
```

---

### `handle:Get(path)`

```lua
handle:Get(path: string) -> any
```

Reads a value by dot-path. Safely traverses nested tables.

```lua
local xp = handle:Get("Stats.XP")
```

---

### `handle:GetAll()`

```lua
handle:GetAll() -> table
```

Returns a deep copy of the entire `profile.Data` table.

---

### `handle:Snapshot()`

```lua
handle:Snapshot() -> table
```

Alias for `GetAll`. Returns a deep copy of the current data state — useful for diffing or undo logic.

---

## Handle — Writing

### Proxy write

```lua
handle.XP = 100  -- sets profile.Data.XP directly (does not fire signals)
```

---

### `handle:Set(path, value)`

```lua
handle:Set(path: string, value: any)
```

Sets a value by dot-path. Fires the signal for the top-level key and the wildcard `"*"` signal.

```lua
handle:Set("Stats.XP", 1500)
```

---

### `handle:Patch(tbl)`

```lua
handle:Patch(tbl: table)
```

Shallow-merges a table into `profile.Data`. Fires per-key signals for each changed key and the wildcard signal.

```lua
handle:Patch({ Level = 10, Currency = 250 })
```

---

### `handle:DeepPatch(tbl)`

```lua
handle:DeepPatch(tbl: table)
```

Recursively merges a table into `profile.Data`. Fires the wildcard signal only.

---

### `handle:Increment(path, amount?)`

```lua
handle:Increment(path: string, amount?: number)
```

Increments a numeric value at the given dot-path by `amount` (default `1`). Errors if the target is not a number.

```lua
handle:Increment("XP", 100)
handle:Increment("Deaths")  -- +1
```

---

### `handle:Append(path, value)`

```lua
handle:Append(path: string, value: any)
```

Inserts a value at the end of the array at the given dot-path. Errors if the target is not a table.

---

### `handle:Remove(path, index?)`

```lua
handle:Remove(path: string, index?: number)
```

Removes an entry from an array at the given dot-path. If `index` is omitted, removes the last element.

---

### `handle:Save()`

```lua
handle:Save() -> Promise
```

Forces an immediate save of this profile. Resolves on success, rejects with the error string on failure.

---

### `handle:Reconcile()`

```lua
handle:Reconcile()
```

Fills any missing keys in `profile.Data` from the registered schema defaults. Runs automatically on session start when `Reconcile = true`.

---

### `handle:Release()`

```lua
handle:Release() -> Promise<owner>
```

Ends this profile's session individually. Prefer `Keep.EndPlayerSession` to release all schemas at once.

---

## Handle — Observing

### `handle:Observe(path, fn)`

```lua
handle:Observe(path: string, fn: (newValue: any, oldValue: any) -> ()) -> Connection
```

Subscribes to changes on a specific top-level key. Fires from `Set` and `Patch`. Returns a connection; call `:Disconnect()` to unsubscribe.

> **Note:** `Observe` tracks the top-level key derived from the dot-path. Observing `"Stats.XP"` registers a listener on `"Stats"`. Use `ObserveAll` and diff manually for sub-key granularity.

```lua
local conn = handle:Observe("XP", function(new, old)
    print("XP changed:", old, "→", new)
end)

-- later:
conn:Disconnect()
```

---

### `handle:ObserveAll(fn)`

```lua
handle:ObserveAll(fn: (data: table) -> ()) -> Connection
```

Subscribes to any data change. Fires with a deep copy of the full data table after every mutation. Higher overhead than `Observe` — use for replication or debugging.

```lua
handle:ObserveAll(function(data)
    ReplicateToClient(player, data)
end)
```

---

## Handle — Metadata

| Method | Returns | Description |
|--------|---------|-------------|
| `handle:IsActive()` | `boolean` | Whether the underlying ProfileStore profile is still active. |
| `handle:GetOwner()` | `Owner` | The owner object for this session. |
| `handle:GetVersion()` | `number` | Schema version stored in MetaTags, or configured `Version` as fallback. |
| `handle:GetSaveCount()` | `number` | How many times this profile has been saved in the current session. |
| `handle:GetLastSave()` | `number` | Unix timestamp of the most recent save. |

---

## Keep.Debug

Utilities for inspection and testing. Do not use in production game logic.

| Method | Returns | Description |
|--------|---------|-------------|
| `Keep.Debug.GetAllSessions()` | `table` | Raw session table from Live for all active users. |
| `Keep.Debug.GetSession(plrOrId)` | `table` | All session data for a specific user. |
| `Keep.Debug.GetLiveCount()` | `number` | Number of currently active sessions. |
| `Keep.Debug.PrintSession(plrOrId)` | — | Prints a formatted session summary to output. |
| `Keep.Debug.DumpLive()` | — | Calls `PrintSession` for every active userId. |
| `Keep.Debug.SimulateRelease(plrOrId)` | — | Calls `EndPlayerSession` for the given user. Simulates a player leaving mid-session. |
| `Keep.Debug.ForceExpire(plrOrId)` | — | Force-invalidates all profiles and wipes Live state without going through the normal session-end flow. Tests stale-handle behavior. |
| `Keep.Debug.GetConfig()` | `table` | Deep copy of the current config table. |
| `Keep.Debug.GetSchemas()` | `table` | Deep copy of all registered schemas. |
| `Keep.Debug.GetStoreNames()` | `string[]` | List of all ProfileStore names that have been created. |
