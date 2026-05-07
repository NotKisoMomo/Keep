# Keep: A Roblox Data Persistence Library

[![Static Badge](https://img.shields.io/badge/build-v1.0.0-black)](https://github.com/TheRealKr3ative)
![Static Badge](https://img.shields.io/badge/stability-stable-green)

Keep is a straightforward, opinionated wrapper for [ProfileStore](https://github.com/MadStudioRoblox/ProfileStore). It provides a clean API for managing player sessions, multi-schema stores, dot-path mutations, and automatic save cycles, all packed into one single module.

---

## Table of Contents

* [Features](#features)
* [Installation](#installation)
* [Quick Start](#quick-start)
* [Core Concepts](#core-concepts)
* [API Reference](#api-reference)
* [Owner Object](#owner-object)
* [Handle API](#handle-api)
* [Events](#events)
* [Auto-Save](#auto-save)
* [Migration](#migration)
* [Advanced Usage](#advanced-usage)
* [Examples](#examples)
* [Keep.Debug](#keepdebug)
* [Exported Types](#exported-types)
* [Contact](#contact)

---

## Features

* **Multi-Schema Support:** Register multiple independent stores per player, such as Account, Inventory, and Settings.
* **Promise-Based Sessions:** Manage session loading and releasing easily using a Promise API.
* **Dot-Path Mutations:** Read and write deeply nested data using simple string paths like "Stats.XP".
* **Per-Key Signals:** Subscribe to changes for individual keys or watch everything at once with fine-grained subscriptions.
* **Auto-Save:** Set up background saving at specific intervals with zero extra boilerplate.
* **Schema Reconciliation:** Automatically fills in any missing keys from your defaults every time data loads.
* **Data Migration:** Use version-aware callbacks to safely update your schema as your game grows.
* **Handle Proxy:** Accessing a key directly on a handle will automatically fall through to profile.Data.
* **Debug Utilities:** Includes built-in tools for inspecting sessions and simulating data scenarios during testing.

---

## Installation

To get started, place the Keep module inside ServerScriptService or another shared location, then require it on the server:

```lua
local Keep = require(ServerScriptService.Keep)
```

Keep is server-only. You should never require it on the client. Instead, replicate data handles to your players manually using RemoteEvents or your preferred networking layer.

---

## Quick Start

```lua
local Players = game:GetService("Players")
local Keep    = require(ServerScriptService.Keep)

-- 1. Configure your settings (optional)
Keep.Configure({
    AutoSave         = true,
    AutoSaveInterval = 60,
    Reconcile        = true,
    LogLevel         = "Warn",
})

-- 2. Register your data schemas
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

-- 3. Start sessions when players join
Players.PlayerAdded:Connect(function(player)
    Keep.StartPlayerSession(player)
        .next(function(stores, owner)
            local account   = stores.Account
            local inventory = stores.Inventory

            print("Loaded session for:", owner.UserId)
            print("Current Level:", account:Get("Level"))

            account:Increment("XP", 100)
            inventory:Append("Items", "Sword")
        end)
        .catch(function(err)
            warn("The session failed to load:", err)
        end)
end)
```

> **Note:** Sessions are automatically released when a player leaves or when the server shuts down. You usually won't need to call EndPlayerSession yourself.

---

## Core Concepts

### Schemas
A schema is just a plain Lua table that defines the structure and default values for a player's data. When you call Keep.SetSchema, you are registering a new named store. When a session starts, all these registered schemas load at once and are handed back to you in the stores table.

Schemas also handle reconciliation. If you add a new key to your schema later, Keep will see it is missing from a returning player's data and fill it in with the default value automatically.

### Sessions
A session covers the entire life of a player's data while they are in your game. Keep.StartPlayerSession opens a ProfileStore session for every schema you've registered, wraps them in handles, and tracks them under one "owner" object.

Keep is smart about tracking these internally. If you call StartPlayerSession for someone who already has an active session, it will just return the existing handles. It also handles the cleanup for PlayerRemoving and BindToClose so you can focus on the game logic.

### Handles
A handle is your primary tool for interacting with a player's data. It uses a proxy, which means if you try to read a key that isn't a built-in method, it looks inside profile.Data for you. 

```lua
-- Simple access
local level = handle.Level       -- reads profile.Data.Level
handle.Level = 10                -- writes directly (no signals fired)

-- Explicit access (this triggers signals/observers)
handle:Set("Level", 10)
handle:Increment("XP", 500)
```

---

## API Reference

### Keep.Configure
```lua
Keep.Configure(config: table)
```
Use this to override default settings. It is best to call this before you register schemas or start sessions.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| StorageType | string | "Single" | The ProfileStore storage type. |
| StudioSeparate | boolean | false | Prefixes store names with Studio_ while in the editor. |
| AutoSave | boolean | true | Enables background saving for active sessions. |
| AutoSaveInterval | number | 60 | Seconds between each automatic save. |
| LogLevel | string | "Warn" | One of "None", "Warn", or "Verbose". |
| Version | number | 1 | Increment this to trigger the Migration callback. |

---

### Keep.SetSchema
```lua
Keep.SetSchema(name: string, schema: table)
```
This registers a schema and creates the associated ProfileStore. Once registered, you can use Keep[name] as a shortcut to get a player's handle.

---

### Keep.StartPlayerSession
```lua
Keep.StartPlayerSession(plrOrId: Player | number, overrides?: table) -> Promise<stores, owner>
```
This is the main entry point. It loads all schemas, starts the auto-save loop, and resolves with the handles and owner information. If you provide overrides, they will be merged into the defaults for brand-new players.

---

### Keep.AwaitSession
```lua
Keep.AwaitSession(plrOrId: Player | number) -> Promise<stores, owner>
```
This returns a promise that resolves as soon as the session is ready. It is great for systems that start up independently and need to wait for data to become available.

---

### Keep.WaitForSession
```lua
Keep.WaitForSession(plrOrId: Player | number) -> (stores, owner) | (nil, nil)
```
A synchronous version of AwaitSession. It will yield the current thread until the session is active.

---

## Handle API

### Reading Data

* **handle:Get(path)**: Reads a value using a dot-path (like "Stats.Wins"). It won't error if a middle part of the path is missing.
* **handle:GetAll()**: Returns a deep copy of all the player's data. You can safely modify this copy without changing the saved data.
* **handle:Snapshot()**: An alias for GetAll, perfect for checking data state before and after changes.

### Writing Data

* **handle:Set(path, value)**: Sets a value at a dot-path and notifies any observers.
* **handle:Patch(tbl)**: Merges a table into the top level of the data and fires signals for every changed key.
* **handle:DeepPatch(tbl)**: Recursively merges a table. This is useful for updating one nested value without wiping out the rest of the table.
* **handle:Increment(path, amount)**: Adds to a number at the given path. If you don't provide an amount, it defaults to 1.
* **handle:Append(path, value)**: Adds a value to the end of an array.
* **handle:Remove(path, index)**: Removes an item from an array at a specific index.

---

## Observing Data

### handle:Observe(path, callback)
Subscribe to changes on a specific key. The callback gives you the new value and the old value. 

```lua
local connection = handle:Observe("XP", function(new, old)
    print("XP changed from", old, "to", new)
end)
```

### handle:ObserveAll(callback)
This fires whenever any part of the data changes. Use this sparingly, as it returns a full deep copy of the data every time.

---

## Events

You can listen for these globally on the Keep module:

* **OnSessionStart**: Fires when a player's data is fully loaded.
* **OnSessionEnd**: Fires after a session has been safely closed and saved.
* **OnSaveComplete**: Fires every time a save cycle finishes successfully.

---

## Migration

If you need to change your data structure (like renaming "Coins" to "Gold"), use the Migration setting in Keep.Configure.

```lua
Keep.Configure({
    Version   = 2,
    Migration = function(data, oldVersion)
        if oldVersion < 2 then
            data.Gold = data.Coins or 0
            data.Coins = nil
        end
        return data
    end,
})
```

The migration runs before your code ever sees the data, ensuring everything is in the correct format.

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