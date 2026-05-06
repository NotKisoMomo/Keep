# Keep — Roblox Data Persistence Library

[![Static Badge](https://img.shields.io/badge/build-v1.0.0-black)](https://github.com/TheRealKr3ative)
![Static Badge](https://img.shields.io/badge/stability-stable-green)

A lightweight, opinionated wrapper around [ProfileStore](https://github.com/MadStudioRoblox/ProfileStore) for Roblox. Clean API for managing player data sessions, multi-schema stores, dot-path mutations, per-key observation, and automatic save cycles — all behind a single module.

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
- [API Reference](#api-reference)
  - [Keep.Configure](#keepconfigure)
  - [Keep.SetSchema](#keepsetschema)
  - [Keep.StartPlayerSession](#keepstartplayersession)
  - [Keep.AwaitSession](#keepawaitsession)
  - [Keep.WaitForSession](#keepwaitforsession)
  - [Keep.EndPlayerSession](#keependplayersession)
  - [Keep.EndAllSessions](#keependallsessions)
  - [Keep.IsSessionActive](#keepissessionactive)
  - [Keep.GetOwner](#keepgetowner)
  - [Keep.GetAllOwners](#keepgetallowners)
  - [Keep.ResolveUserId](#keepresolveuserid)
- [Owner Object](#owner-object)
- [Handle API](#handle-api)
  - [Reading Data](#reading-data)
  - [Writing Data](#writing-data)
  - [Observing Data](#observing-data)
  - [Metadata](#metadata)
- [Events](#events)
- [Auto-Save](#auto-save)
- [Migration](#migration)
- [Advanced Usage](#advanced-usage)
- [Examples](#examples)
- [Keep.Debug](#keepdebug)
- [Exported Types](#exported-types)
- [Contact](#contact)

---

## Features

- **Multi-Schema Support** -- register multiple independent stores per player (Account, Inventory, Settings, etc.)
- **Promise-Based Sessions** -- async session loading and releasing via a Promise API
- **Dot-Path Mutations** -- read and write deeply nested data with string paths like `"Stats.XP"`
- **Per-Key Signals** -- observe individual keys or all changes with fine-grained subscriptions
- **Auto-Save** -- configurable interval-based background saving with zero boilerplate
- **Schema Reconciliation** -- automatically fills missing keys from defaults on every load
- **Data Migration** -- version-aware migration callbacks for safe schema evolution
- **Handle Proxy** -- direct key access on the handle falls through to `profile.Data`
- **Debug Utilities** -- built-in inspection and simulation tools for testing

---

## Installation

Place the Keep module inside `ServerScriptService` or a shared location and require it on the server:

```lua
local Keep = require(ServerScriptService.Keep)
```

Keep is a **server-only** module. Never require it on the client — data handles should be replicated manually to clients via `RemoteEvents` or a networking layer.

---

## Quick Start

```lua
local Players = game:GetService("Players")
local Keep    = require(ServerScriptService.Keep)

-- 1. Configure (optional)
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
    Stats    = {
        Kills  = 0,
        Deaths = 0,
        Wins   = 0,
    },
})

Keep.SetSchema("Inventory", {
    Items    = {},
    Equipped = {},
})

-- 3. Start sessions on join
Players.PlayerAdded:Connect(function(player)
    Keep.StartPlayerSession(player)
        .next(function(stores, owner)
            local account   = stores.Account
            local inventory = stores.Inventory

            print("Loaded session for", owner.UserId)
            print("Level:", account:Get("Level"))

            account:Increment("XP", 100)
            inventory:Append("Items", "Sword")
        end)
        .catch(function(err)
            warn("Session failed:", err)
        end)
end)
```

> Sessions are automatically released on `PlayerRemoving` and `game:BindToClose`. You do not need to call `EndPlayerSession` in standard flows — it is handled internally.

---

## Core Concepts

### Schemas

A schema is a plain Lua table that defines the structure and default values for a player's data store. Each call to `Keep.SetSchema` registers a new named store. At session start, all registered schemas are loaded simultaneously and returned together in the `stores` table.

```lua
Keep.SetSchema("Account", {
    Level = 1,
    XP    = 0,
})
Keep.SetSchema("Inventory", {
    Items = {},
})
```

Schemas drive reconciliation — any key present in the schema but missing from a player's saved data is filled in with the default value on load.

---

### Sessions

A session represents the full lifecycle of a player's data — from load to release. Calling `Keep.StartPlayerSession` opens a ProfileStore session for every registered schema, wraps each in a handle, and registers them under a single owner object.

Sessions are tracked internally. If a session is already active when `StartPlayerSession` is called again, the existing handles are returned immediately.

Keep automatically wires `PlayerRemoving` and `game:BindToClose` internally — you do not need to call `EndPlayerSession` yourself in standard flows. `PlayerRemoving` triggers a graceful release for that player, and `BindToClose` ends all remaining sessions before the server shuts down. Only call `EndPlayerSession` manually when you need to force an early release outside of those events.

---

### Handles

A handle is the primary interface for reading and writing a player's data within one schema. It wraps `profile.Data` with a proxy — indexing an unknown key reads directly from the data table, and assigning to one writes to it. Named methods (`Get`, `Set`, `Patch`, `Increment`, etc.) are explicit operations that also fire observation signals.

```lua
-- Proxy access
local level = handle.Level       -- reads profile.Data.Level
handle.Level = 10                -- writes profile.Data.Level directly (no signals)

-- Explicit access (fires signals)
handle:Set("Level", 10)
handle:Increment("XP", 500)
```

---

### Channel Accessors

After `SetSchema` is called, `Keep[schemaName]` becomes a channel accessor — a function that accepts a player or userId and returns their active handle for that schema. This is a convenience shorthand for cases where you need a handle outside of the `StartPlayerSession` promise chain.

```lua
Keep.SetSchema("Account", { Level = 1 })

-- Somewhere else in your codebase:
local handle = Keep.Account(player)
if handle then
    handle:Increment("Level")
end
```

> Channel accessors return `nil` if no session is active for that player. Always ensure a session has been started with `StartPlayerSession` before using them.

---

## API Reference

### Keep.Configure

```lua
Keep.Configure(config: table)
```

Overrides default module settings. Call this before registering any schemas or starting any sessions.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `StorageType` | `string` | `"Single"` | ProfileStore storage type. |
| `StudioSeparate` | `boolean` | `false` | Prefixes store names with `Studio_` in Studio to avoid polluting production data. |
| `AutoSave` | `boolean` | `true` | Enables periodic background saving for all active sessions. |
| `AutoSaveInterval` | `number` | `60` | Seconds between each automatic save cycle. |
| `SessionLockRetries` | `number` | `5` | Number of session lock acquisition retries before giving up. |
| `SessionLockInterval` | `number` | `15` | Seconds between session lock retry attempts. |
| `LogLevel` | `string` | `"Warn"` | Controls log verbosity. One of `"None"`, `"Warn"`, `"Verbose"`. |
| `Reconcile` | `boolean` | `true` | Automatically fills missing keys from schema defaults on session start. |
| `Migration` | `function?` | `nil` | Migration callback. Receives `(data, oldVersion)` and must return the updated data table. Only called when the stored version is less than `Version`. |
| `Version` | `number` | `1` | Current schema version. Incrementing this triggers the `Migration` callback on next load. |

---

### Keep.SetSchema

```lua
-- Named schema
Keep.SetSchema(name: string, schema: table)

-- Default schema (exposes Keep.Account)
Keep.SetSchema(schema: table)
```

Registers a data schema and creates a ProfileStore for it. Named schemas expose `Keep[name]` as a channel accessor after registration.

```lua
Keep.SetSchema("Account", {
    Level    = 1,
    XP       = 0,
    Currency = 0,
})

-- Keep.Account(player) is now available
```

Both server and client should define the same schemas if you intend to replicate data. Keep itself is server-only, but the shape can be shared.

---

### Keep.StartPlayerSession

```lua
Keep.StartPlayerSession(plrOrId: Player | number, overrides?: table) -> Promise<stores, owner>
```

Loads all registered schemas for a player, wraps each in a handle, registers the session, and starts auto-save. Resolves with `(stores: { [schemaName]: Handle }, owner: Owner)`.

If a session is already active for this player, resolves immediately with the existing handles without reloading.

| Param | Type | Description |
|-------|------|-------------|
| `plrOrId` | `Player \| number` | The player instance or their numeric userId. |
| `overrides` | `table?` | Optional table deep-merged into schema defaults before the ProfileStore is created. Only meaningful for first-time players — if the player already has saved data, ProfileStore loads that existing data and the overrides have no effect. |

```lua
Keep.StartPlayerSession(player)
    .next(function(stores, owner)
        print("Session started:", owner.SessionId)

        stores.Account:Increment("Currency", 500)
        stores.Inventory:Append("Items", "Shield")
    end)
    .catch(function(err)
        warn("Could not load session:", err)
    end)
```

---

### Keep.AwaitSession

```lua
Keep.AwaitSession(plrOrId: Player | number) -> Promise<stores, owner>
```

Returns a Promise that resolves once the player's session is active. If the session is already active, resolves immediately. If not, queues internally and resolves when `StartPlayerSession` completes.

Useful for systems that initialise independently of `PlayerAdded` and need to wait for data to be available.

```lua
-- In a separate system module
Keep.AwaitSession(player).next(function(stores, owner)
    local level = stores.Account:Get("Level")
    assignSpawnPoint(player, level)
end)
```

---

### Keep.WaitForSession

```lua
Keep.WaitForSession(plrOrId: Player | number) -> (stores, owner) | (nil, nil)
```

Synchronous blocking yield. Polls until the session is active and returns `(stores, owner)`. Returns `nil, nil` if the userId cannot be resolved.

Prefer `AwaitSession` in production code — this exists for simple scripts where promise chaining is inconvenient.

```lua
local stores, owner = Keep.WaitForSession(player)
if stores then
    print("XP:", stores.Account:Get("XP"))
end
```

---

### Keep.EndPlayerSession

```lua
Keep.EndPlayerSession(plrOrId: Player | number) -> Promise<owner>
```

Gracefully ends a player's session. Saves and releases all profiles, stops auto-save, clears the handle cache, and fires `OnSessionEnd`. Resolves with the owner object.

This is called automatically on `PlayerRemoving`. Only call it manually if you need to force an early session end.

```lua
Keep.EndPlayerSession(player)
    .next(function(owner)
        print("Session ended cleanly for", owner.UserId)
    end)
    .catch(function(err)
        warn("Release error:", err)
    end)
```

---

### Keep.EndAllSessions

```lua
Keep.EndAllSessions() -> Promise
```

Calls `EndPlayerSession` for every currently active userId and resolves once all sessions have concluded. Called automatically in `game:BindToClose`. Useful for manual server shutdown sequences.

---

### Keep.IsSessionActive

```lua
Keep.IsSessionActive(plrOrId: Player | number) -> boolean
```

Returns `true` if a session is currently active for the given player or userId.

---

### Keep.GetOwner

```lua
Keep.GetOwner(plrOrId: Player | number) -> Owner?
```

Returns the owner object for the given player's active session, or `nil` if no session exists.

---

### Keep.GetAllOwners

```lua
Keep.GetAllOwners() -> { [number]: Owner }
```

Returns a table of all active owner objects keyed by userId.

---

### Keep.ResolveUserId

```lua
Keep.ResolveUserId(plrOrId: Player | number) -> number?
```

Resolves a Player instance or raw userId to a numeric userId. Returns `nil` if resolution fails. Useful when working with player identifiers from external sources.

---

## Owner Object

The owner table is created at session start and passed to event callbacks and some handle methods.

| Field | Type | Description |
|-------|------|-------------|
| `Player` | `Player?` | The Player instance. `nil` if the session was started with a raw userId. |
| `UserId` | `number` | Numeric userId. |
| `SessionId` | `string` | Unique identifier for this session. |
| `StartTime` | `number` | Unix timestamp of when the session started. |
| `StoreNames` | `string[]` | List of all schema names loaded in this session. |

---

## Handle API

A handle is the interface for all data operations within a single schema session. Every schema loaded in a session returns its own handle via `stores[schemaName]`.

### Reading Data

#### Proxy read

```lua
local value = handle[key]
```

Falls through to `profile.Data[key]` for any key not defined as a method on the handle itself. Does not support dot-path traversal — use `Get` for nested paths.

---

#### `handle:Get(path)`

```lua
handle:Get(path: string) -> any
```

Reads a value by dot-path. Safely traverses nested tables without erroring on missing intermediate keys.

```lua
local xp    = handle:Get("XP")
local kills = handle:Get("Stats.Kills")
local first = handle:Get("Items.1")    -- array indexing
```

---

#### `handle:GetAll()`

```lua
handle:GetAll() -> table
```

Returns a deep copy of the entire `profile.Data` table. Safe to modify without affecting the live data.

---

#### `handle:Snapshot()`

```lua
handle:Snapshot() -> table
```

Alias for `GetAll`. Returns a deep copy of the current data state — useful for diffing before and after a mutation, implementing undo, or serialising for replication.

```lua
local before = handle:Snapshot()
handle:Increment("XP", 500)
local after  = handle:Snapshot()
```

---

### Writing Data

#### Proxy write

```lua
handle[key] = value
```

Writes directly to `profile.Data[key]`. Does not fire per-key or wildcard signals. Use `Set` or `Patch` if observers should be notified.

---

#### `handle:Set(path, value)`

```lua
handle:Set(path: string, value: any)
```

Sets a value by dot-path. Fires the signal for the top-level key derived from the path, as well as the wildcard `"*"` signal. Use this over the proxy write whenever you want observers to react.

```lua
handle:Set("Level", 25)
handle:Set("Stats.Kills", 0)
```

---

#### `handle:Patch(tbl)`

```lua
handle:Patch(tbl: table)
```

Shallow-merges a table into `profile.Data`. Fires per-key signals for each changed key and the wildcard `"*"` signal. Equivalent to calling `Set` on each key individually.

```lua
handle:Patch({
    Level    = 10,
    Currency = 500,
})
```

---

#### `handle:DeepPatch(tbl)`

```lua
handle:DeepPatch(tbl: table)
```

Recursively merges a table into `profile.Data`, preserving untouched nested keys. Fires only the wildcard `"*"` signal. Use when you need to update nested structures without overwriting sibling keys.

```lua
-- Only updates Stats.Wins; Stats.Kills and Stats.Deaths remain unchanged
handle:DeepPatch({
    Stats = { Wins = 5 }
})
```

---

#### `handle:Increment(path, amount?)`

```lua
handle:Increment(path: string, amount?: number)
```

Increments a numeric value at the given dot-path by `amount` (default `1`). Errors if the resolved value is not a number. Fires `Set` signals.

```lua
handle:Increment("XP", 250)
handle:Increment("Stats.Deaths")     -- +1
handle:Increment("Currency", -50)    -- subtract
```

---

#### `handle:Append(path, value)`

```lua
handle:Append(path: string, value: any)
```

Inserts a value at the end of the array at the given dot-path using `table.insert`. Errors if the resolved value is not a table. Fires the wildcard `"*"` signal.

```lua
handle:Append("Items", "FireSword")
handle:Append("Items", { id = "shield_gold", level = 3 })
```

---

#### `handle:Remove(path, index?)`

```lua
handle:Remove(path: string, index?: number)
```

Removes an entry from the array at the given dot-path using `table.remove`. If `index` is omitted, removes the last element. Errors if the resolved value is not a table. Fires the wildcard `"*"` signal.

```lua
handle:Remove("Items", 2)    -- removes index 2
handle:Remove("Items")       -- removes last entry
```

---

#### `handle:Save()`

```lua
handle:Save() -> Promise
```

Forces an immediate save of this profile outside of the auto-save cycle. Resolves on success, rejects with the error string on failure. Updates the save count and last save timestamp internally.

```lua
handle:Save()
    .next(function()
        print("Saved successfully")
    end)
    .catch(function(err)
        warn("Save failed:", err)
    end)
```

---

#### `handle:Reconcile()`

```lua
handle:Reconcile()
```

Fills any keys missing from `profile.Data` using the registered schema defaults. Called automatically on session start when `Reconcile = true` in config. Call manually if you modify the schema at runtime and want to re-reconcile a live session.

---

#### `handle:Release()`

```lua
handle:Release() -> Promise<owner>
```

Ends this individual profile's session. Resolves with the owner object. Prefer `Keep.EndPlayerSession` in most cases — it ends all schemas for a player atomically and cleans up shared state (auto-save, owner, handle cache).

---

### Observing Data

#### `handle:Observe(path, fn)`

```lua
handle:Observe(path: string, fn: (newValue: any, oldValue: any) -> ()) -> Connection
```

Subscribes to changes on a specific top-level key. The callback receives `(newValue, oldValue)`. Fires from `Set` and `Patch`. Returns a connection — call `:Disconnect()` to unsubscribe.

> Observe resolves the signal key from the top-level segment of the dot-path. Observing `"Stats.Kills"` registers on `"Stats"` and fires on any change to the `Stats` table.

```lua
local conn = handle:Observe("XP", function(new, old)
    print(string.format("XP: %d → %d (+%d)", old, new, new - old))
end)

-- Later, when cleaning up:
conn:Disconnect()
```

---

#### `handle:ObserveAll(fn)`

```lua
handle:ObserveAll(fn: (data: table) -> ()) -> Connection
```

Subscribes to any data change. Fires after every mutation with a deep copy of the full `profile.Data` table. Higher overhead than `Observe` — use for replication, logging, or debugging rather than game logic.

```lua
handle:ObserveAll(function(data)
    -- Replicate full data snapshot to the client
    DataReplicator.Send(player, data)
end)
```

---

### Metadata

| Method | Returns | Description |
|--------|---------|-------------|
| `handle:IsActive()` | `boolean` | Whether the underlying ProfileStore profile is still active. |
| `handle:GetOwner()` | `Owner` | The owner object for this session. |
| `handle:GetVersion()` | `number` | Schema version stored in the profile's MetaTags. Falls back to the configured `Version` if not present. |
| `handle:GetSaveCount()` | `number` | How many times this profile has been saved in the current session (includes auto-saves). |
| `handle:GetLastSave()` | `number` | Unix timestamp of the most recent save for this profile. `0` if never saved this session. |

---

## Events

All event listeners return a connection object. Call `:Disconnect()` to stop listening.

```lua
Keep.OnSessionStart(fn: (stores: { [string]: Handle }, owner: Owner) -> ())  -> Connection
Keep.OnSessionEnd(fn: (owner: Owner) -> ())                                   -> Connection
Keep.OnSaveComplete(fn: (owner: Owner) -> ())                                 -> Connection
```

| Event | Fires when... | Callback receives |
|-------|--------------|-------------------|
| `OnSessionStart` | A session finishes loading all schemas. | `stores, owner` |
| `OnSessionEnd` | A session has been fully released. | `owner` |
| `OnSaveComplete` | Any save completes, auto or manual. | `owner` |

```lua
Keep.OnSessionStart(function(stores, owner)
    print("Session opened:", owner.UserId, "—", owner.SessionId)
end)

Keep.OnSessionEnd(function(owner)
    print("Session closed:", owner.UserId)
end)

Keep.OnSaveComplete(function(owner)
    -- e.g. log save metrics or notify a dashboard
end)
```

---

## Auto-Save

When `AutoSave = true` (default), Keep starts a repeating save loop for each player at session start. Every `AutoSaveInterval` seconds, all active profiles in that session are saved, the save count is incremented, and `OnSaveComplete` fires.

```lua
Keep.Configure({
    AutoSave         = true,
    AutoSaveInterval = 120,    -- save every 2 minutes
})
```

Auto-save is stopped automatically when a session ends. You can still trigger manual saves at any time via `handle:Save()`.

---

## Migration

Keep supports version-aware data migration. Set a `Version` number and provide a `Migration` function. When a player's stored version is lower than `Version`, the migration callback runs before the session is handed to your code.

```lua
Keep.Configure({
    Version   = 2,
    Migration = function(data, oldVersion)
        if oldVersion < 2 then
            -- Rename old field, set new defaults
            data.Currency = data.Coins or 0
            data.Coins    = nil
            data.Stats    = data.Stats or { Kills = 0, Deaths = 0, Wins = 0 }
        end
        return data
    end,
})
```

The migration callback receives `(data: table, oldVersion: number)` and **must return the updated data table**. The stored version is updated to `Version` after migration completes.

> Always increment `Version` when making breaking changes to your schema — do not rely on reconciliation alone for structural changes.

---

## Advanced Usage

### Accessing handles outside the promise chain

Use channel accessors when you need a handle from a system that did not receive it directly from `StartPlayerSession`:

```lua
-- In any server Script
local handle = Keep.Account(player)

if handle and handle:IsActive() then
    handle:Increment("Currency", 100)
end
```

---

### Observing for client replication

```lua
Keep.StartPlayerSession(player).next(function(stores)
    -- Replicate initial state
    DataReplicator.Send(player, stores.Account:GetAll())

    -- Replicate all future changes
    stores.Account:ObserveAll(function(data)
        DataReplicator.Send(player, data)
    end)
end)
```

---

### Filtering observations by key

```lua
stores.Account:Observe("XP", function(new, old)
    local level = stores.Account:Get("Level")
    if new >= XP_THRESHOLDS[level + 1] then
        stores.Account:Increment("Level")
        stores.Account:Set("XP", 0)
    end
end)
```

---

### Deferred session access with AwaitSession

```lua
-- LeaderboardModule -- initialises separately from PlayerAdded
local function initLeaderboard(player)
    Keep.AwaitSession(player).next(function(stores)
        local level = stores.Account:Get("Level")
        Leaderboard.SetEntry(player, level)

        stores.Account:Observe("Level", function(new)
            Leaderboard.SetEntry(player, new)
        end)
    end)
end
```

---

### Using overrides for first-time setup

```lua
Keep.StartPlayerSession(player, {
    Level    = 1,
    Currency = 250,    -- starter bonus
})
```

Overrides are deep-merged into schema defaults before the ProfileStore is created. They only apply when the player has no existing saved data — if ProfileStore loads a saved profile, that data takes precedence and the overrides are ignored entirely.

---

## Examples

### Tracking Kills and Deaths

```lua
Keep.StartPlayerSession(player).next(function(stores)
    local account = stores.Account

    KillSignal:Connect(function(killer, victim)
        if killer == player then
            account:Increment("Stats.Kills")
            account:Increment("Currency", 50)
        end
        if victim == player then
            account:Increment("Stats.Deaths")
        end
    end)
end)
```

---

### Inventory Management

```lua
Keep.StartPlayerSession(player).next(function(stores)
    local inv = stores.Inventory

    local function grantItem(itemId)
        inv:Append("Items", itemId)
    end

    local function removeItem(itemId)
        local items = inv:Get("Items")
        for i, v in ipairs(items) do
            if v == itemId then
                inv:Remove("Items", i)
                break
            end
        end
    end

    local function equipItem(itemId)
        inv:Set("Equipped", itemId)
    end
end)
```

---

### Level-Up System

```lua
local XP_PER_LEVEL = 1000

Keep.StartPlayerSession(player).next(function(stores)
    local account = stores.Account

    local function awardXP(amount)
        account:Increment("XP", amount)

        local currentXP    = account:Get("XP")
        local currentLevel = account:Get("Level")

        if currentXP >= XP_PER_LEVEL then
            account:Set("XP", currentXP - XP_PER_LEVEL)
            account:Increment("Level")
            print(player.Name .. " levelled up to", account:Get("Level"))
        end
    end

    account:Observe("Level", function(new)
        player.leaderstats.Level.Value = new
    end)

    return awardXP
end)
```

---

### Full Session Wiring

```lua
local Players = game:GetService("Players")
local Keep    = require(ServerScriptService.Keep)

Keep.Configure({
    AutoSave         = true,
    AutoSaveInterval = 90,
    Reconcile        = true,
    StudioSeparate   = true,
    LogLevel         = "Warn",
    Version          = 3,
    Migration        = function(data, v)
        if v < 2 then data.Stats    = { Kills = 0, Deaths = 0, Wins = 0 } end
        if v < 3 then data.Prestige = 0 end
        return data
    end,
})

Keep.SetSchema("Account", {
    Level    = 1,
    XP       = 0,
    Currency = 0,
    Prestige = 0,
    Stats    = { Kills = 0, Deaths = 0, Wins = 0 },
})

Keep.SetSchema("Inventory", {
    Items    = {},
    Equipped = nil,
})

Keep.OnSessionStart(function(stores, owner)
    print("+ Session:", owner.UserId, "#" .. owner.SessionId)
end)

Keep.OnSessionEnd(function(owner)
    print("- Session:", owner.UserId)
end)

Players.PlayerAdded:Connect(function(player)
    Keep.StartPlayerSession(player)
        .next(function(stores, owner)
            local account = stores.Account

            local ls      = Instance.new("Folder", player)
            ls.Name       = "leaderstats"
            local lvl     = Instance.new("IntValue", ls)
            lvl.Name      = "Level"
            lvl.Value     = account:Get("Level")

            account:Observe("Level", function(new)
                lvl.Value = new
            end)
        end)
        .catch(warn)
end)
```

---

## Keep.Debug

Utilities for inspection, simulation, and testing. Do not use in production game logic.

| Method | Returns | Description |
|--------|---------|-------------|
| `Keep.Debug.GetAllSessions()` | `table` | Raw session table from Live for all active users. |
| `Keep.Debug.GetSession(plrOrId)` | `table` | All session data for a specific user. |
| `Keep.Debug.GetLiveCount()` | `number` | Total number of currently active sessions. |
| `Keep.Debug.PrintSession(plrOrId)` | — | Prints a formatted session summary to output — SessionId, store names, save counts, last save timestamps, and raw Data tables. |
| `Keep.Debug.DumpLive()` | — | Calls `PrintSession` for every active userId. Useful for a full server-state snapshot. |
| `Keep.Debug.SimulateRelease(plrOrId)` | — | Calls `EndPlayerSession` for the given user. Simulates a player leaving mid-session without them actually disconnecting. |
| `Keep.Debug.ForceExpire(plrOrId)` | — | Force-invalidates all profiles and wipes Live state without going through the normal session-end flow. Used to test stale-handle behaviour. |
| `Keep.Debug.GetConfig()` | `table` | Deep copy of the current config table. |
| `Keep.Debug.GetSchemas()` | `table` | Deep copy of all registered schemas. |
| `Keep.Debug.GetStoreNames()` | `string[]` | List of all ProfileStore names that have been initialised. |

```lua
-- Print a full session snapshot mid-game
Keep.Debug.PrintSession(player)

-- Simulate a player dropping mid-session for testing
Keep.Debug.SimulateRelease(player)

-- Dump all active sessions
Keep.Debug.DumpLive()
```

---

## Exported Types

```lua
export type Owner = {
    Player     : Player?,
    UserId     : number,
    SessionId  : string,
    StartTime  : number,
    StoreNames : { string },
}

export type Handle = {
    -- Reading
    Get      : (self: Handle, path: string) -> any,
    GetAll   : (self: Handle) -> { [string]: any },
    Snapshot : (self: Handle) -> { [string]: any },

    -- Writing
    Set       : (self: Handle, path: string, value: any) -> (),
    Patch     : (self: Handle, tbl: { [string]: any }) -> (),
    DeepPatch : (self: Handle, tbl: { [string]: any }) -> (),
    Increment : (self: Handle, path: string, amount: number?) -> (),
    Append    : (self: Handle, path: string, value: any) -> (),
    Remove    : (self: Handle, path: string, index: number?) -> (),
    Save      : (self: Handle) -> Promise,
    Reconcile : (self: Handle) -> (),
    Release   : (self: Handle) -> Promise,

    -- Observing
    Observe    : (self: Handle, path: string, fn: (new: any, old: any) -> ()) -> Connection,
    ObserveAll : (self: Handle, fn: (data: { [string]: any }) -> ()) -> Connection,

    -- Metadata
    IsActive     : (self: Handle) -> boolean,
    GetOwner     : (self: Handle) -> Owner,
    GetVersion   : (self: Handle) -> number,
    GetSaveCount : (self: Handle) -> number,
    GetLastSave  : (self: Handle) -> number,
}

export type Config = {
    StorageType         : string,
    StudioSeparate      : boolean,
    AutoSave            : boolean,
    AutoSaveInterval    : number,
    SessionLockRetries  : number,
    SessionLockInterval : number,
    LogLevel            : "None" | "Warn" | "Verbose",
    Reconcile           : boolean,
    Migration           : ((data: { [string]: any }, oldVersion: number) -> { [string]: any })?,
    Version             : number,
}

export type Connection = {
    Disconnect : () -> (),
}

export type Promise = {
    next      : (self: Promise, fn: (...any) -> ()) -> Promise,
    catch     : (self: Promise, fn: (err: string) -> ()) -> Promise,
    concluded : (self: Promise, fn: () -> ()) -> Promise,
}
```

---

## Contact

| Platform | Handle |
|---|---|
| Roblox | [Kr3ativeKrayon](https://www.roblox.com/users/1911367519/profile) |
| YouTube | [TotallyKr3ative](https://www.youtube.com/channel/UCpNZQoKVclQ74Pk5GmzdQDA) |
| X (Twitter) | [TotallyNotKr3ative](https://x.com/TheRealKr3ative) |
| Email | [TheRealKr3ative@gmail.com](mailto:TheRealKr3ative@gmail.com) |

---

*Last Updated: May 6, 2026*

---