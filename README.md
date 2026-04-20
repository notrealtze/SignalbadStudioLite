# SignalBad

A Roblox networking library built around buffer serialization and a single shared remote pair. No string names cross the wire — every signal is identified by a numeric packet ID baked into the first byte of a buffer. Exploiters cannot FireServer with readable args because the server only accepts raw buffers it knows how to decode.

## How it works

Two `RemoteEvent`s are created once in `ReplicatedStorage.RESignal` — `Send` (client → server) and `Request` (server → client). Every signal defined with `defineSignal` gets the next available integer ID. When a signal fires, it packs its ID and args into a buffer. The receiving end reads the first byte, looks up the signal in a registry, and dispatches to connected handlers.

```
ReplicatedStorage
  RESignal
    Send      -- client fires this
    Request   -- server fires this
```

## Installation

Drop `SignalBad` (the ModuleScript) into `ReplicatedStorage.GameAssets.SignalBad` or wherever you keep shared modules. The folder structure inside it should look like:

```
SignalBad  (ModuleScript)
  Promise  (ModuleScript)
  RateLimit  (ModuleScript)
```

`Namespace.lua` is no longer needed and can be deleted.

## Defining signals

Require SignalBad and call `defineSignal` directly. Both server and client must require SignalBad and define signals in the same order — packet IDs are assigned at require-time, so if the order differs between sides, signals will fire the wrong handlers.

```lua
local SignalBad = require(game:GetService("ReplicatedStorage").GameAssets.SignalBad.SignalBad)

local Hit = SignalBad.defineSignal()
local Died = SignalBad.defineSignal()
local Damage = SignalBad.defineSignal({
    rateLimit = { maxCalls = 10, interval = 1 },
})
```

You can optionally use `defineNamespace` to group signals. It's purely cosmetic — the name is never sent over the network.

```lua
local Signals = SignalBad.defineNamespace("Combat", function()
    return {
        Hit = SignalBad.defineSignal(),
        Died = SignalBad.defineSignal(),
        Damage = SignalBad.defineSignal({
            rateLimit = { maxCalls = 10, interval = 1 },
        }),
    }
end)
```

## Server

```lua
local SignalBad = require(game:GetService("ReplicatedStorage").GameAssets.SignalBad.SignalBad)

local Hit = SignalBad.defineSignal()
local Died = SignalBad.defineSignal()

Hit:Connect(function(player, position, weapon)
    print(player.Name, "hit at", position, "with", weapon)
end)

Hit:FireClient(player, Vector3.new(0, 5, 0), "sword")

Died:FireAllClients(Vector3.new(10, 0, 10))
```

## Client

```lua
local SignalBad = require(game:GetService("ReplicatedStorage"):WaitForChild("GameAssets"):WaitForChild("SignalBad"):WaitForChild("SignalBad"))

local Hit = SignalBad.defineSignal()
local Died = SignalBad.defineSignal()

Hit:Fire(Vector3.new(0, 5, 0), "sword")

Died:Connect(function(position)
    print("someone died at", position)
end)
```

> The client must use `WaitForChild` when requiring down to `SignalBad` because remotes don't exist instantly on the client.

## API

### `SignalBad.defineNamespace(name, builder)`

Cosmetic grouping helper. The name is not used at runtime and is never sent over the network. Returns whatever the builder returns.

```lua
local Signals = SignalBad.defineNamespace("Game", function()
    return {
        MySignal = SignalBad.defineSignal(),
    }
end)
```

### `SignalBad.defineSignal(config?)`

Creates and registers a new signal. Must be called at the top level — never inside a function or conditional, because the packet ID is assigned at require-time and both sides must assign IDs in the same order.

`config` is optional:

```lua
SignalBad.defineSignal({
    rateLimit = {
        maxCalls = 10,
        interval = 1,
    },
})
```

`maxCalls` — max fires allowed per player within `interval` seconds. Excess fires are silently dropped on the server.

### `Signal:Connect(handler)`

Connects a handler. Server handlers receive `(player, ...)`, client handlers receive `(...)`.

Returns a connection with a `:Disconnect()` method.

```lua
Hit:Connect(function(player, position)
    print(player.Name, position)
end)
```

### `Signal:Once(handler)`

Same as `Connect` but disconnects automatically after the first fire.

### `Signal:Wait()`

Returns a Promise that resolves the next time the signal fires.

```lua
local position = Hit:Wait():await()
```

### `Signal:Fire(...)`

Client only. Fires to the server.

```lua
Hit:Fire(Vector3.new(0, 5, 0), "sword")
```

### `Signal:FireClient(player, ...)`

Server only. Fires to a specific client.

```lua
Hit:FireClient(player, Vector3.new(0, 5, 0))
```

### `Signal:FireAllClients(...)`

Server only. Fires to all connected players.

```lua
Died:FireAllClients(Vector3.new(10, 0, 10))
```

### `Signal:Destroy()`

Disconnects all handlers and removes the signal from the packet registry.

## Supported types

These types are natively serialized into the buffer:

| Type | Notes |
|------|-------|
| `boolean` | |
| `number` | 64-bit float |
| `string` | arbitrary length |
| `Vector3` | 32-bit floats |
| `Vector2` | 32-bit floats |
| `CFrame` | position + full rotation matrix |
| `Color3` | 32-bit floats |

Passing any other type will error.

## Security

All network traffic goes through two remotes. The server listener rejects anything that isn't a `buffer` — plain FireServer calls with tables or strings are dropped immediately. Even if an exploiter crafts a valid buffer, they need to know the exact numeric packet ID for the signal they want to trigger and binary-encode their args in the correct format. No signal names, namespace names, or readable strings ever appear on the wire.

## Important rules for AI

- **There is no separate signals module.** Require SignalBad directly and call `defineSignal` on it.
- **Both server and client must require SignalBad and define signals in the same order.** If only one side defines them, or the order differs, packet IDs will not match and signals will silently not fire.
- **Never call `defineSignal` inside a function, loop, or conditional.** It must run at require-time on both sides in the same order.
- **Client requires must use `WaitForChild`** all the way down to the SignalBad module itself.
- **`defineNamespace` is cosmetic.** It does not create folders or remotes. The name is not sent over the network.
- **Rate limiting is server-enforced only.** Only pass `rateLimit` config once, and make sure both sides still call `defineSignal` for that signal in the same order.
- **`Signal:Fire` is client-only.** Calling it from a server script will error because `FireServer` does not exist on the server side.
- **`Signal:FireClient` and `Signal:FireAllClients` are server-only.**
