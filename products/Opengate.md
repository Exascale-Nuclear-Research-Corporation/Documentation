# ENRCo. CEIA OPENGATE Weapons Detection System

![Preview](https://payhip.com/cdn-cgi/image/format=auto,width=1500/https://pe56d.s3.amazonaws.com/o_1jpds9cu5r2pgr61q6pkm4fb15.png)

![Structure Example](https://r2.lukedevelopment.com/Screenshot%202026-06-02%20211556.png)

## API Documentation

All events are fired through the `DC` BindableEvent located inside the OpenGate model.  
Place your script inside the OpenGate model and connect to `script.Parent.Parent.DC`.

---

## Setup

```lua
local DC = script.Parent.Parent.DC

DC.Event:Connect(function(Event, ...)
    local Args = {...}
    -- handle events here
end)
```

---

## Events

### `PlayerProhibitedItems`
Fired when a player is detected carrying prohibited items.

| Argument | Type | Description |
|---|---|---|
| `Args[1]` | Player | The player who was detected |
| `Args[2]` | table | List of prohibited item names |

```lua
if Event == "PlayerProhibitedItems" then
    local Player = Args[1]
    local ProhibitedItems = Args[2]
    print("ALERT: " .. Player.Name .. " was detected with: " .. table.concat(ProhibitedItems, ", "))
end
```

---

### `PlayerPassed`
Fired when a player passes through the detector with no prohibited items.

| Argument | Type | Description |
|---|---|---|
| `Args[1]` | Player | The player who passed |

```lua
elseif Event == "PlayerPassed" then
    local Player = Args[1]
    print("CLEAR: " .. Player.Name .. " passed scan with no prohibited items.")
end
```

---

### `PlayerLeft`
Fired when a player leaves the detector zone.

| Argument | Type | Description |
|---|---|---|
| `Args[1]` | Player | The player who left |

```lua
elseif Event == "PlayerLeft" then
    local Player = Args[1]
    print("LEFT: " .. Player.Name .. " left the detector zone.")
end
```

---

### `DetectorEnabled`
Fired when the detector is enabled or disabled via chat command.

| Argument | Type | Description |
|---|---|---|
| `Args[1]` | boolean | `true` if enabled, `false` if disabled |

```lua
elseif Event == "DetectorEnabled" then
    local State = Args[1]
    if State then
        print("Detector has been enabled.")
    else
        print("Detector has been disabled.")
    end
end
```

---

## Full Example

```lua
local Gate = script.Parent
local DC = Gate.DC

DC.Event:Connect(function(Event, ...)
    local Args = {...}

    if Event == "PlayerProhibitedItems" then
        local Player = Args[1]
        local ProhibitedItems = Args[2]
        print("ALERT: " .. Player.Name .. " was detected with: " .. table.concat(ProhibitedItems, ", "))

    elseif Event == "PlayerPassed" then
        local Player = Args[1]
        print("CLEAR: " .. Player.Name .. " passed scan with no prohibited items.")

    elseif Event == "PlayerLeft" then
        local Player = Args[1]
        print("LEFT: " .. Player.Name .. " left the detector zone.")

    elseif Event == "DetectorEnabled" then
        local State = Args[1]
        if State then
            print("Detector has been enabled.")
        else
            print("Detector has been disabled.")
        end
    end
end)
```
