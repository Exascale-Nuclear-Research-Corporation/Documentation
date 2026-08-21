# ENRCo. Salto XS Series - API Documentation

## Version: 1.2.0

---

## Overview

The ENRCo. Salto XS Series is a card reader system for Roblox providing access control with encrypted keys, animations, statistics, and access logging. Other scripts integrate with it through a single `Network` `BindableFunction` exposed as an action dispatcher — not through direct calls into the reader script.

> **Breaking change from earlier docs:** `Network:Invoke` no longer takes `(player, encryptedKey, handle)` and returns a plain boolean. It now takes an **action name** as its first argument and returns `(ok: boolean, result: any)`. Update any integrations written against the old signature.

---

## Table of Contents
1. [BindableFunction API](#bindablefunction-api)
2. [RemoteEvent API](#remoteevent-api)
3. [Action Reference](#action-reference)
4. [Data Structures](#data-structures)
5. [Examples](#examples)
6. [Error Handling](#error-handling)
7. [Best Practices](#best-practices)

---

## BindableFunction API

### `Network`

**Location:** `<ReaderModel>.<ScriptName>.Network` — the `BindableFunction` is created as a **child of the reader script itself**, not of the reader model. Substitute your script's actual instance name (e.g. `SaltoMain`).

```lua
local network = reader_model.SaltoMain.Network
```

#### Signature
```lua
function Network:Invoke(action: string, ...: any) -> (ok: boolean, result: any)
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|--------------|
| `action` | `string` | ✅ Yes | Name of the action to run (see [Action Reference](#action-reference)) |
| `...` | `any` | Depends | Arguments forwarded to the handler for that action |

#### Returns

| Position | Type | Description |
|----------|------|--------------|
| `ok` | `boolean` | `true` if a matching handler ran without erroring, `false` otherwise |
| `result` | `any` | The handler's **first** return value on success; an error code string on failure (`"UnknownAction"` or `"HandlerError"`) |

> **Note:** If a handler itself returns multiple values, only the **first** is passed through `Network:Invoke`. See [`HasValidKey`](#hasvalidkey) for a case where this matters.

Every call is internally wrapped in `pcall`, so a bad action name or an erroring handler will never throw across the boundary — it just returns `false` plus a reason string.

---

## RemoteEvent API

### `__saltotag` RemoteEvent

For client-side card tools. Handled automatically by the reader; fire-and-forget, no return value.

**Location:** `CardReader.__saltotag`

#### Signature
```lua
RemoteEvent:FireServer(encryptedKey: string, handle: BasePart)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|--------------|
| `encryptedKey` | `string` | ✅ Yes | XOR-encrypted key data from the card |
| `handle` | `BasePart` | ✅ Yes | The card tool's `Handle`, used as the decryption key source |

#### Example (LocalScript inside the card tool)
```lua
local cardReader = workspace.Door.Reader.CardReader
local saltotag = cardReader:FindFirstChild("__saltotag")

if saltotag then
	local encryptedKey = "4A6F8B2C"
	local handle = script.Parent:FindFirstChild("Handle")
	saltotag:FireServer(encryptedKey, handle)
end
```

This triggers the same internal `perform_card_read` flow as a physical swipe/tap: LED flash, sound, grant/deny, logging, and statistics — there's no separate result to read back from this event; watch the LED or subscribe to the access log via `Network` instead.

---

## Action Reference

### `GetStatistics`
Returns cumulative grant/deny counts.

```lua
local ok, stats = network:Invoke("GetStatistics")
-- stats = { Granted = 42, Denied = 7 }
```

### `GetAccessLog`
Returns up to `CONFIG.MaxLogEntries` recent access attempts, oldest first.

```lua
local ok, logs = network:Invoke("GetAccessLog")
for _, entry in ipairs(logs) do
	print(entry.PlayerName, entry.Result, os.date("%c", entry.Timestamp))
end
```

Returns `{}` if `CONFIG.EnableAccessLogging` is off or no attempts have been logged yet.

### `GetVersion`
Returns the reader's version string.

```lua
local ok, version = network:Invoke("GetVersion") -- "1.2.0"
```

### `IsProcessing`
Returns whether the reader is currently mid-read (LED animating, door timer running, etc.). Useful for gating UI so you don't fire a redundant swipe.

```lua
local ok, busy = network:Invoke("IsProcessing")
```

### `IsDisabled`
Returns whether the reader has auto-disabled itself due to a major-version mismatch against the remote version manifest.

```lua
local ok, disabled = network:Invoke("IsDisabled")
```

### `GetAllowedKeys`
Returns a **copy** of the currently loaded allowed-key list (safe to mutate; won't affect the live reader).

```lua
local ok, keys = network:Invoke("GetAllowedKeys")
```

### `ReloadDoorConfiguration`
Re-reads `SystemSettings` from the door model and repopulates the allowed-key list. Returns `true` on success, `false` if `SystemSettings` is missing or malformed.

```lua
local ok, reloaded = network:Invoke("ReloadDoorConfiguration")
```

### `HasValidKey`
Checks whether a player is currently carrying a valid key, without triggering a read (no LED, no sound, no logging).

```lua
local ok, hasKey = network:Invoke("HasValidKey", somePlayer)
```

> **Truncation caveat:** internally this handler returns `(hasKey, toolInstance)`, but `Network:Invoke` only forwards the first value — you get `hasKey` back, not the matching `Tool`. If you need the tool instance too, call the reader's `get_player_card_tool`-equivalent logic yourself, or ask for a dedicated action to be added.

---

## Data Structures

### Access Log Entry
```lua
{
	PlayerName = "Player123",
	UserId = 123456789,
	Result = "Granted", -- or "Denied"
	Timestamp = 1234567890, -- os.time()
}
```

### Statistics
```lua
{
	Granted = 42,
	Denied = 7,
}
```

---

## Examples

### 1. External Script — Log Denial Spikes
```lua
local network = workspace.Door.Reader.SaltoMain.Network

task.spawn(function()
	while true do
		task.wait(60)
		local ok, stats = network:Invoke("GetStatistics")
		if ok and stats.Denied > 10 then
			warn("High denial rate detected on this reader!")
		end
	end
end)
```

### 2. Admin Panel Integration
```lua
local network = script.Parent.SaltoMain.Network

local function checkPlayerAccess(player)
	local ok, hasKey = network:Invoke("HasValidKey", player)
	return ok and hasKey
end

local function getAccessLogs()
	local ok, logs = network:Invoke("GetAccessLog")
	return ok and logs or {}
end
```

### 3. Hot-Reload Keys After an Admin Edits SystemSettings
```lua
local network = script.Parent.SaltoMain.Network

local function pushKeyUpdate()
	local ok, reloaded = network:Invoke("ReloadDoorConfiguration")
	if ok and reloaded then
		print("Reader picked up new key list")
	else
		warn("Reload failed — check SystemSettings on the door")
	end
end
```

---

## Error Handling

### Common Failure Cases

| `ok` | `result` | Cause | Fix |
|------|----------|-------|-----|
| `false` | `"UnknownAction"` | Action name typo or unsupported action | Check spelling against the [Action Reference](#action-reference) |
| `false` | `"HandlerError"` | The handler itself errored (bad argument type, etc.) | Check server output for the warned error message; verify argument types |
| `true` | `nil`/`false` | Handler ran but the semantic answer was negative (e.g. `HasValidKey` → `false`) | This is a normal result, not a failure — check `result`, not just `ok` |

### Safe Invoke Wrapper
```lua
local function safeInvoke(network, action, ...)
	local ok, result = pcall(function()
		return network:Invoke(action, ...)
	end)
	if not ok then
		warn("Network dispatch threw: " .. tostring(result))
		return false, "DispatchError"
	end
	return result -- (handlerOk, handlerResult) from the reader
end
```

Note: `Network.OnInvoke` already pcalls the handler internally, so this outer wrapper is only defending against the (rare) case of the `BindableFunction` itself being missing or destroyed mid-call.

---

## Best Practices

1. **Always check `ok` before trusting `result`.** A `false` result value and a `false` `ok` mean different things.
2. **Cache the `Network` reference**, don't re-index `script.Parent...Network` on every call.
3. **Don't poll `IsProcessing` in a tight loop** — check it once before you'd otherwise fire a duplicate swipe, not continuously.
4. **Use `GetAccessLog`/`GetStatistics` for monitoring, not access decisions** — they reflect history, not real-time authorization state.
5. **After calling `ReloadDoorConfiguration`, re-fetch `GetAllowedKeys`** if you're displaying the key list somewhere, since the old copy is now stale.
