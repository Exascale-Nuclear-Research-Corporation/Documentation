# ENRCo. HID Proximity & multiClass with Signo - API Documentation

## Version: 3.0.2

---

## Overview

The HID Proximity & multiClass reader is a lighter-weight sibling of the Salto XS Series: encrypted key validation, card-swipe animation, and grant/deny LED flashing, without statistics tracking or access logging. Other scripts integrate through a single `Network` `BindableFunction` action dispatcher.

---

## Table of Contents
1. [BindableFunction API](#bindablefunction-api)
2. [RemoteEvent API](#remoteevent-api)
3. [Action Reference](#action-reference)
4. [Examples](#examples)
5. [Error Handling](#error-handling)
6. [Best Practices](#best-practices)
7. [Differences from Salto XS](#differences-from-salto-xs)

---

## BindableFunction API

### `Network`

**Location:** `<ReaderModel>.<ScriptName>.Network` — a child of the reader script itself. Substitute your script's actual instance name (e.g. `HIDMain`).

```lua
local network = reader_model.HIDMain.Network
```

#### Signature
```lua
function Network:Invoke(action: string, ...: any) -> (ok: boolean, result: any)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|--------------|
| `action` | `string` | ✅ Yes | Name of the action to run (see [Action Reference](#action-reference)) |
| `...` | `any` | Depends | Arguments forwarded to the handler for that action |

#### Returns

| Position | Type | Description |
|----------|------|--------------|
| `ok` | `boolean` | `true` if a matching handler ran without erroring |
| `result` | `any` | Handler's first return value on success, or an error string (`"UnknownAction"` / `"HandlerError"`) on failure |

All calls are internally `pcall`-wrapped, so bad input never throws across the boundary.

---

## RemoteEvent API

### `__hidtag` RemoteEvent

For client-side card tools. Fire-and-forget, no return value.

**Location:** `CardReader.__hidtag`

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
local hidtag = cardReader:FindFirstChild("__hidtag")

if hidtag then
	local encryptedKey = "4A6F8B2C"
	local handle = script.Parent:FindFirstChild("Handle")
	hidtag:FireServer(encryptedKey, handle)
end
```

This runs the same grant/deny + animation flow as a physical tap. There's nothing to read back from the event itself — watch the LED, or check `IsProcessing`/`HasValidKey` via `Network` if you need programmatic status.

---

## Action Reference

### `GetVersion`
```lua
local ok, version = network:Invoke("GetVersion") -- "3.0.1"
```

### `IsProcessing`
Returns whether the reader is mid-grant or mid-deny-flash. Note that unlike Salto, this reader does **not** lock during the card-swipe *animation* itself — `is_processing` only covers the grant/deny LED sequence.
```lua
local ok, busy = network:Invoke("IsProcessing")
```

### `IsDisabled`
Returns whether the reader auto-disabled itself due to a major-version mismatch against the remote version manifest.
```lua
local ok, disabled = network:Invoke("IsDisabled")
```

### `GetAllowedKeys`
Returns a copy of the current allowed-key list.
```lua
local ok, keys = network:Invoke("GetAllowedKeys")
```

### `ReloadDoorConfiguration`
Re-reads `SystemSettings` from the door model.

> **Important — self-destructing config:** this reader destroys the `SystemSettings` `ModuleScript` 6–8 seconds after first reading it (anti-tamper behavior). If you call `ReloadDoorConfiguration` after that window and `SystemSettings` hasn't been re-placed under the door, this returns `false`.

```lua
local ok, reloaded = network:Invoke("ReloadDoorConfiguration")
```

### `HasValidKey`
Checks whether a player is carrying a valid key without triggering a read.
```lua
local ok, hasKey = network:Invoke("HasValidKey", somePlayer)
```
> Same truncation caveat as Salto: the internal handler returns `(hasKey, toolInstance)` but only `hasKey` is forwarded through `Invoke`.

---

## Examples

### 1. Gate a Custom UI on Key Possession
```lua
local network = script.Parent.HIDMain.Network

local function showAccessPrompt(player)
	local ok, hasKey = network:Invoke("HasValidKey", player)
	if ok and hasKey then
		-- show "tap to enter" UI
	end
end
```

### 2. Refresh Keys After Re-Placing SystemSettings
```lua
local network = script.Parent.HIDMain.Network

local function refreshAfterEdit(newSettingsModule)
	newSettingsModule.Parent = doorModel
	task.wait() -- let it settle in the hierarchy
	local ok, reloaded = network:Invoke("ReloadDoorConfiguration")
	print(ok and reloaded and "Keys reloaded" or "Reload failed")
end
```

---

## Error Handling

| `ok` | `result` | Cause | Fix |
|------|----------|-------|-----|
| `false` | `"UnknownAction"` | Action name typo or unsupported action | Check spelling against the [Action Reference](#action-reference) |
| `false` | `"HandlerError"` | Handler errored (e.g. bad argument) | Check server output; verify argument types |
| `true` | `false` | Handler ran, semantic answer is negative | Normal — not a failure, check `result` itself |

```lua
local ok, result = network:Invoke("HasValidKey", player)
if not ok then
	warn("Network call failed: " .. tostring(result))
	return
end
-- result is the actual hasKey boolean here
```

---

## Best Practices

1. **Check `ok` before trusting `result`** — same rule as Salto.
2. **Cache the `Network` reference** rather than re-indexing it per call.
3. **Don't rely on `ReloadDoorConfiguration` working long after startup** — remember the self-destruct timer on `SystemSettings`.
4. **This reader has no logging/statistics actions** — if you need audit history, use a Salto XS reader or build logging into your own integration script listening for grants via a custom hook.

---

## Differences from Salto XS

| Feature | Salto XS | HID Proximity |
|---|---|---|
| Access logging | ✅ `GetAccessLog` | ❌ Not implemented |
| Statistics | ✅ `GetStatistics` | ❌ Not implemented |
| Card-read flash while waiting | ✅ Blocking flash during read | ❌ Animation runs in parallel, doesn't gate the decision |
| Hold-based swipe (cancelable) | ✅ `PromptButtonHoldBegan`/`HoldEnded` | ❌ Uses `Triggered` (instant) |
| SystemSettings lifecycle | Persists | Self-destructs 6–8s after first load |
| Config field for read timing | `CardReadDuration` | `AnimationDuration` |
