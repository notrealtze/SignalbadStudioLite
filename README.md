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

Create a shared ModuleScript that both server and client require. Both sides must require it so their packet registries are built in the same order with the same IDs.

```lua
local SignalBad = require(game:GetService("ReplicatedStorage").GameAssets.SignalBad.SignalBad)

return SignalBad.defineNamespace("Combat", function()
    return {
        Hit = SignalBad.defineSignal(),
        Died = SignalBad.defineSignal(),
        Damage = SignalBad.defineSignal({
            rateLimit = { maxCalls = 10, interval = 1 },
        }),
    }
end)
```

`defineNamespace` is a grouping helper — it just calls the builder and returns the table. The namespace name is not sent over the network.

## Server

```lua
local Signals = require(path.to.SharedSignals)

Signals.Hit:Connect(function(player, position, weapon)
    print(player.Name, "hit at", position, "with", weapon)
end)

Signals.Hit:FireClient(player, Vector3.new(0, 5, 0), "sword")

Signals.Died:FireAllClients(Vector3.new(10, 0, 10))
```

## Client

```lua
local SignalBad = require(game:GetService("ReplicatedStorage"):WaitForChild("GameAssets"):WaitForChild("SignalBad"):WaitForChild("SignalBad"))
local Signals = require(path.to.SharedSignals)

Signals.Hit:Fire(Vector3.new(0, 5, 0), "sword")

Signals.Died:Connect(function(position)
    print("someone died at", position)
end)
```

> The client must use `WaitForChild` when requiring down to `SignalBad` because remotes don't exist instantly on the client.

## API

### `SignalBad.defineNamespace(name, builder)`

Groups signals together. The name is not used at runtime — it's purely for your own organization. Returns whatever the builder returns.

```lua
local Signals = SignalBad.defineNamespace("Game", function()
    return {
        MySignal = SignalBad.defineSignal(),
    }
end)
```

### `SignalBad.defineSignal(config?)`

Creates and registers a new signal. Must be called at the top level of a shared module — never inside a function or conditional, because the packet ID is assigned at require-time and both sides must assign IDs in the same order.

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
Signals.Hit:Connect(function(player, position)
    print(player.Name, position)
end)
```

### `Signal:Once(handler)`

Same as `Connect` but disconnects automatically after the first fire.

### `Signal:Wait()`

Returns a Promise that resolves the next time the signal fires.

```lua
local position = Signals.Hit:Wait():await()
```

### `Signal:Fire(...)`

Client only. Fires to the server.

```lua
Signals.Hit:Fire(Vector3.new(0, 5, 0), "sword")
```

### `Signal:FireClient(player, ...)`

Server only. Fires to a specific client.

```lua
Signals.Hit:FireClient(player, Vector3.new(0, 5, 0))
```

### `Signal:FireAllClients(...)`

Server only. Fires to all connected players.

```lua
Signals.Died:FireAllClients(Vector3.new(10, 0, 10))
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

- **Always require the shared signals module on both server and client.** If only one side requires it, the packet IDs will not match and signals will silently not fire.
- **Never call `defineSignal` inside a function, loop, or conditional.** It must run at require-time on both sides in the same order. If the order differs, every signal will be wired to the wrong handler.
- **Client requires must use `WaitForChild`** all the way down to the SignalBad module itself, not just to the signals module.
- **`defineNamespace` is cosmetic.** It does not create folders or remotes. The name is not sent over the network.
- **Rate limiting is server-enforced only.** Do not pass `rateLimit` config on the client side — only define it once in the shared module.
- **`Signal:Fire` is client-only.** Calling it from a server script will error because `FireServer` does not exist on the server side.
- **`Signal:FireClient` and `Signal:FireAllClients` are server-only.**
