# ENRCo. Salto XS Series - API Documentation

## Version: 1.2.0

---

## Overview

The ENRCo. Salto XS Series is a robust card reader system for Roblox that provides secure access control with support for encrypted keys, animations, statistics, and logging. It features a modular architecture with a BindableFunction interface for easy integration with other scripts.

---

## Table of Contents
1. [BindableFunction API](#bindablefunction-api)
2. [RemoteEvent API](#remoteevent-api)
3. [Public Functions](#public-functions)
4. [Events](#events)
5. [Data Structures](#data-structures)
6. [Examples](#examples)
7. [Error Handling](#error-handling)
8. [Best Practices](#best-practices)

---

## BindableFunction API

### `Network`

The primary interface for triggering card reads from other scripts. Located at `script.Network`.

> **Note:** This BindableFunction is pre-created in the script. Do not instantiate it yourself.

#### Signature
```lua
function Network:Invoke(player: Player, encryptedKey: string?, handle: BasePart?) -> boolean
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `player` | `Player` | ✅ Yes | The player attempting access |
| `encryptedKey` | `string` | ❌ No | Encrypted key data from the card (if using encryption) |
| `handle` | `BasePart` | ❌ No | The handle part of the card tool (used for size vector decryption) |

#### Returns
- `boolean` - `true` if access was granted, `false` otherwise

#### Throws
- Error if `player` is nil
- Error if `Network` BindableFunction is not found

#### Usage Examples

**Basic card check (no encryption):**
```lua
local Network = script.Parent.Network
local granted = Network:Invoke(player)
if granted then
    print("Access granted!")
else
    print("Access denied!")
end
```

**With encrypted key:**
```lua
local Network = script.Parent.Network
local encryptedKey = "4A6F8B2C" -- Example encrypted key
local handle = tool:FindFirstChild("Handle")
local granted = Network:Invoke(player, encryptedKey, handle)
```

---

## RemoteEvent API

### `__saltotag` RemoteEvent

For backward compatibility with existing card tools. This RemoteEvent is automatically handled by the system.

**Location:** `CardReader.__saltotag`

#### Signature
```lua
RemoteEvent:FireServer(encryptedKey: string, handle: BasePart)
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `encryptedKey` | `string` | ✅ Yes | Encrypted key data from the card |
| `handle` | `BasePart` | ✅ Yes | The handle part of the card tool |

#### Example

**Client-side (LocalScript inside tool):**
```lua
local cardReader = workspace:FindFirstChild("Door").Reader.CardReader
local saltotag = cardReader:FindFirstChild("__saltotag")

if saltotag then
    local encryptedKey = "4A6F8B2C" -- Your encrypted key
    local handle = script.Parent:FindFirstChild("Handle")
    saltotag:FireServer(encryptedKey, handle)
end
```

---

## Public Functions

### `debug_print(message: string)`

Prints debug messages to the console if `DebugEnabled` is true in configuration.

**Parameters:**
- `message: string` - The debug message to print

**Example:**
```lua
debug_print("Processing card read for " .. player.Name)
-- Output: [ENRCo. Salto]: Processing card read for Player123
```

---

### `get_access_log() -> table`

Returns the access log containing all access attempts stored in memory.

**Returns:** Table of log entries (max `MaxLogEntries`)

**Structure:**
```lua
{
    {
        PlayerName = "Player123",
        UserId = 123456789,
        Result = "Granted", -- or "Denied"
        Timestamp = 1234567890
    },
    -- More entries...
}
```

**Example:**
```lua
local logs = get_access_log()
for _, entry in ipairs(logs) do
    print(string.format("%s was %s at %s", 
        entry.PlayerName, 
        entry.Result, 
        os.date("%Y-%m-%d %H:%M:%S", entry.Timestamp)
    ))
end
```

---

### `get_statistics() -> table`

Returns usage statistics for the reader.

**Returns:**
```lua
{
    Granted = 42,  -- Number of granted accesses
    Denied = 7     -- Number of denied accesses
}
```

**Example:**
```lua
local stats = get_statistics()
local total = stats.Granted + stats.Denied
print("Total attempts: " .. total)
print("Success rate: " .. string.format("%.1f%%", (stats.Granted / total) * 100))
```

---

### `player_has_valid_key(player: Player) -> boolean, Tool`

Checks if a player has a valid key in their inventory.

**Parameters:**
- `player: Player` - The player to check

**Returns:**
- `boolean` - `true` if the player has a valid key
- `Tool` - The tool containing the valid key (if found)

**Example:**
```lua
local hasKey, tool = player_has_valid_key(player)
if hasKey then
    print("Player has a valid key: " .. tool.Name)
end
```

---

### `get_player_card_tool(player: Player) -> Tool`

Gets the card tool from a player's inventory.

**Parameters:**
- `player: Player` - The player to check

**Returns:**
- `Tool` - The card tool if found, `nil` otherwise

**Example:**
```lua
local cardTool = get_player_card_tool(player)
if cardTool then
    print("Found card: " .. cardTool.Name)
end
```

---

## Events

### ProximityPrompt Events

When `EnableProximityPrompt = true`:

| Event | Description |
|-------|-------------|
| `PromptButtonHoldBegan` | Fired when player starts holding the prompt |
| `PromptButtonHoldEnded` | Fired when player releases the prompt (cancels swipe) |

### Touch Events

When `EnableProximityPrompt = false`:

| Event | Description |
|-------|-------------|
| `Touched` | Fired when something touches the CardReader part |

---

## Data Structures

### Access Log Entry
```lua
{
    PlayerName = "Player123",   -- Player's display name
    UserId = 123456789,         -- Player's UserId
    Result = "Granted",         -- "Granted" or "Denied"
    Timestamp = 1234567890      -- Unix timestamp
}
```

### Statistics Entry
```lua
{
    Granted = 42,  -- Total grants
    Denied = 7     -- Total denials
}
```

---

## Examples

### 1. External Script - Basic Access Check
```lua
-- Place this in any server script
local readerScript = workspace.Door.Reader.Network
local Network = readerScript:FindFirstChild("Network")

if Network then
    game.Players.PlayerAdded:Connect(function(player)
        -- Check access on join
        local hasAccess = Network:Invoke(player)
        if hasAccess then
            print(player.Name .. " has access to this area!")
        end
    end)
end
```

### 2. Admin Panel Integration
```lua
-- Admin panel script
local Network = workspace.Door.Reader.Network:FindFirstChild("Network")

-- Grant access to specific player
function grantPlayerAccess(player)
    -- You would need to add the key to SystemSettings here
    -- Then trigger a card read
    local result = Network:Invoke(player)
    return result
end

-- Get access logs for display
function getAccessLogs()
    local logs = get_access_log()
    return logs
end
```

### 3. Custom Key Validation
```lua
-- Extend the reader with custom validation
local Network = script.Parent.Network

-- Store original function
local originalInvoke = Network.OnInvoke

-- Override with custom logic
Network.OnInvoke = function(player, encryptedKey, handle)
    -- Custom pre-validation
    if not player:IsInGroup(123456) then
        warn(player.Name .. " is not in the required group")
        return false
    end
    
    -- Call original logic
    return originalInvoke(player, encryptedKey, handle)
end
```

### 4. Remote Access Control
```lua
-- Remote admin script
local Network = workspace.Door.Reader.Network:FindFirstChild("Network")

-- Remote function to check access
game.ReplicatedStorage:WaitForChild("CheckAccess").OnServerEvent:Connect(function(player, targetPlayer)
    local result = Network:Invoke(targetPlayer)
    player:SendNotification("Access for " .. targetPlayer.Name .. ": " .. (result and "Granted" or "Denied"))
end)
```

### 5. Access Logger
```lua
-- Custom logging script
local Network = workspace.Door.Reader.Network:FindFirstChild("Network")

-- Monitor access attempts
game.Players.PlayerAdded:Connect(function(player)
    local hasAccess = Network:Invoke(player)
    
    -- Log to external service
    if hasAccess then
        print("[ACCESS] " .. player.Name .. " granted access")
    else
        warn("[ACCESS] " .. player.Name .. " denied access")
    end
end)
```

---

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Network BindableFunction not found` | The `Network` BindableFunction is missing | Ensure the BindableFunction exists in the script |
| `player is nil` | Called Invoke without a player | Always pass a valid Player object |
| `is_processing` lock | Another card read is in progress | Wait for the current read to complete |
| `reader_disabled` | Version is outdated | Update to the latest version |

### Pcall Wrapper Example
```lua
local Network = script.Parent.Network

local success, result = pcall(function()
    return Network:Invoke(player, encryptedKey, handle)
end)

if not success then
    warn("Card read failed: " .. result)
    return false
end

return result
```

---

## Best Practices

### 1. Always Pcall Your Invokes
```lua
local success, result = pcall(function()
    return Network:Invoke(player)
end)
```

### 2. Check for Nil Values
```lua
if not Network then
    warn("Network BindableFunction not found")
    return
end
```

### 3. Handle Asynchronous Calls Properly
```lua
task.spawn(function()
    local result = Network:Invoke(player)
    -- Handle result here
end)
```

### 4. Use Debug Mode During Development
```lua
-- Set this in your Configuration
DebugEnabled = true
```

### 5. Monitor Statistics
```lua
local stats = get_statistics()
if stats.Denied > 10 then
    warn("High denial rate detected!")
end
```
