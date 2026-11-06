# Growniverse Lua API

Scripts are Lua 5.4 files in `Android/media/<package>/Scripts/` (e.g. `/sdcard/Android/media/com.growniverse.rhyuniverse/Scripts/`). Manage them from the **Scripts** tab: the left list shows your `.lua` files (with **+ New script** at the top); the right side is a code editor with a toolbar (Run, Stop, Delete), quick keys and a status bar. Edits are saved automatically. Nothing runs on its own; a script only runs after you tap Run. Over adb you can also push files straight into that folder.

All functions are flat globals (`getLocal()`, not `growtopia.getLocal()`). There is no script-facing UI API.

## 1. Runtime and sandbox

| Topic | Behaviour |
|---|---|
| Libraries | `base`, `string`, `table`, `math`, `utf8` only. `io`, `os`, `package`, `debug`, `coroutine`, `load`, `loadfile`, `dofile`, `require` and `print` do not exist. `collectgarbage()` only returns the memory in use. |
| Threads | Each script has its own thread and its own Lua state. `sleep` is the only way to wait. |
| Memory | 32 MB per script. Going over stops the script with a memory error. |
| Stop | The Stop button aborts the script, even inside a loop, at the next Lua instruction or the next `sleep` / `warpTo`. One long library call (for example a backtracking `string.find`) is not interrupted. |
| Errors | A Lua error stops only that script. The message is printed in the game console and logcat. |
| Output | `log()` and `logToConsole()` print to the **game console** (`OnConsoleMessage`, only you see it) and to logcat tag `GrowniverseLua`. Script errors are printed there in red. |
| Pointers | Scripts never see game pointers. Every table a function returns is a copy made at call time, so it can go stale but can never crash the game. |

A script ends when its main chunk returns. **Exception:** a script that registered a hook with `addHook` stays loaded until you tap Stop.

## 2. Coordinates

- A **tile** is an integer pair `(x, y)`, `y` grows downward.
- A **position** (`posX`, `posY`) is in pixels. The tile of a player is `floor((posX + 10) / 32)`, `floor((posY + 15) / 32)`; the API gives you `tileX` / `tileY` already.
- A **drop** (floating item) uses `floor((pos + 8) / 32)`.

## 3. Console and utilities

| Function | Returns | Notes |
|---|---|---|
| `log(...)` | - | Prints all arguments separated by a tab, in the game console. Any type. |
| `logToConsole(...)` | - | Same as `log`. |
| `error(...)` | - | Stops the script. The arguments (joined by a tab) are printed in red in the game console as the error message. |
| `sleep(ms)` | - | Waits. Wakes early on Stop. |
| `randomSleep(min, max)` | - | Waits a random time in the range. |
| `getTime()` | number | Unix time in milliseconds. |
| `split(str, sep)` | table | Splits on a plain-text separator (not a Lua pattern). 1-based. An empty `sep` returns `{str}`. |

## 4. Player

| Function | Returns | Notes |
|---|---|---|
| `getLocal()` | [Player](#player-table) or `nil` | `nil` when not in a world. |
| `getGems()` | number | Gem count as the game holds it (0 before the first update). |
| `getPing()` | number or `nil` | Round trip in ms. `nil` when unknown. The read offset is a guess until checked on a device. |
| `getInventory()` | array of [InventoryItem](#inventoryitem-table) | Read from the game's own item list; falls back to the last inventory packet if that cannot be read. |
| `checkInventory(id)` | boolean | `true` if you have at least one. |
| `checkInventoryCount(id)` | number | Amount of that item. |

## 5. Items

| Function | Returns | Notes |
|---|---|---|
| `findItemID(name)` | number or `nil` | Case-insensitive **exact** name match. |
| `getItemName(id)` | string or `nil` | |
| `getItemInfoByID(id)` | [ItemInfo](#iteminfo-table) or `nil` | |
| `getItemInfoByName(name)` | [ItemInfo](#iteminfo-table) or `nil` | Same matching as `findItemID`. |

`findItemID` scans the whole item database, so cache the result if you call it in a loop.

## 6. World

| Function | Returns | Notes |
|---|---|---|
| `getWorldName()` | string | Empty when not in a world. |
| `isInside()` | boolean | `true` when a world is loaded. |
| `getTile(x, y)` | [Tile](#tile-table) or `nil` | `nil` outside the map. |
| `getTiles()` | array of [Tile](#tile-table) | Every tile that has a foreground or background. Creates thousands of tables: call it sparingly. |
| `getPlayerList()` | array of [Player](#player-table) | Everyone in the world, you included. |
| `getPlayerByNetID(netID)` | [Player](#player-table) or `nil` | |
| `getObjectList()` | array of [Object](#object-table) | Floating items. |

## 7. Actions

All sends go through the same path the mod itself uses. Packet types 24 and 25 are still dropped by AntiBan when it is on. **Limit: 200 sends per second per script**; going over stops the script with an error.

| Function | Returns | Notes |
|---|---|---|
| `sendPacket(type, text)` | - | Raw text packet to the server, e.g. `sendPacket(2, "action|input\n|text|hi")`. Text up to 16 KB. |
| `sendPacketRaw(pkt)` | - | Tank packet to the server. `pkt` is a [Packet](#packet-table) table; missing fields are 0. |
| `sendVariant(var, toClient?)` | - | Sends a variant. `var` is `{v1="OnConsoleMessage", v2="hi", netID=-1}` (up to 16 values; strings, integers and floats). With `toClient = true` the game receives it as if the server sent it (not counted in the send limit). |
| `FindPath(x, y, checkOnly?)` | boolean | Walks to the tile. With `checkOnly = true` it only answers whether a path exists. |
| `warpTo(world)` | boolean | Joins the world and **waits** (up to 25 s) until you are loaded in. Returns `false` on timeout. Hooks keep working while it waits. |
| `enter(world)` | - | Sends the join request and returns at once. Name up to 64 characters. |
| `enterPos()` | - | Activates the tile under you (walks through a door). |
| `isOnPos(x, y)` | boolean | `true` if your tile is `(x, y)`. |
| `sendChat(text, toClient?)` | - | Chat to the server. With `toClient = true` it prints a console line only you see. Up to 120 characters. |
| `dropItem(id)` | - | Asks the server to drop the item. The game's amount dialog is shown to you. |
| `confirmDropItem(id, amount)` | - | Answers that dialog with an amount. |
| `tileChange(x, y, value)` | - | `value = 18` punches the tile, any other item id places it. Only tiles within 2 tiles of you. |
| `setFg(x, y, id)` / `setBg(x, y, id)` | - | Changes the tile **on your screen only**; the server keeps the real one. |
| `sendDialog(text)` | - | Shows a dialog (Growtopia dialog syntax) as if the server sent it. Up to 16 KB. |
| `setWeather(id)` | - | Weather on your screen only. |
| `notify(text)` | - | Game banner text. |

## 8. Hooks

```lua
addHook(event, fn)
removeHook(event, fn)   -- the same function value you added
```

`event` is one of the four names below; anything else is an error. A handler that **returns `true` consumes the packet**: the game (and the rest of the mod) never sees it. Any other return value lets it through.

| Event | Handler | Fires on |
|---|---|---|
| `onVariant` | `fn(var)` | Incoming variant (function-call packet). |
| `onSendPacket` | `fn(type, text)` | Outgoing text packet the **game** sends. |
| `onSendPacketRaw` | `fn(pkt)` | Outgoing tank packet the game sends. |
| `onTankPacket` | `fn(pkt)` | Any incoming tank packet. It runs before `onVariant`, and consuming it there skips `onVariant`. |

Rules:

- Handlers run on the **game thread** with a **20 ms budget**; a handler that runs longer is aborted and an error is logged. Keep them short and do not call `sleep` or `warpTo` in them.
- If the script is busy (its main code is running without `sleep`), the event is skipped after about 3 ms instead of waiting, so the game never freezes.
- Packets that **scripts and the mod send themselves** (`sendPacket`, `sendPacketRaw`, `sendVariant`) do not trigger `onSendPacket*`. This also means a script cannot loop on its own sends.
- Only registered events cost anything; with no hooks the packet paths are untouched.
- A handler error is logged and the next handler still runs.

```lua
-- hide one console line, log everything you type
addHook("onVariant", function(v)
  if v.v1 == "OnConsoleMessage" and v.v2 and v.v2:find("secret") then return true end
end)
addHook("onSendPacket", function(type, text) log("sent", type, text) end)
```

## 9. Tables

### Player table
`getLocal`, `getPlayerList`, `getPlayerByNetID`.

| Field | Type | |
|---|---|---|
| `name` | string | Display name as the game holds it. |
| `netID` | number | |
| `userID` | number | |
| `posX`, `posY` | number | Pixels. |
| `tileX`, `tileY` | number | |
| `facingLeft` | boolean | |

### InventoryItem table
| Field | Type | |
|---|---|---|
| `id` | number | |
| `amount` | number | Capped at 250 by the game's own format. |
| `flags` | number | |

### ItemInfo table
| Field | Type | |
|---|---|---|
| `id`, `name` | number, string | |
| `itemType` | number | The "action type" from `items.dat` (20 = clothes). |
| `collisionType` | number | 0 none, 1 solid, 2 platform, ... |
| `breakHits` | number | Hits to break. |
| `rarity`, `maxAmount` | number | |
| `growTime` | number | Seconds a seed takes to grow (not verified on a device). |
| `restoreTime` | number | Seconds until a broken block returns. |
| `clothingType` | number | Slot, for clothes. |

### Tile table
| Field | Type | |
|---|---|---|
| `x`, `y` | number | |
| `fg`, `bg` | number | Item id; 0 is empty. |
| `flags` | number | |
| `readyharvest` | boolean | A tree that is ready to harvest. |

### Object table
`getObjectList` (floating items).

| Field | Type | |
|---|---|---|
| `id` | number | Item id. |
| `amount` | number | |
| `uid` | number | Unique id of this drop. |
| `posX`, `posY` | number | Pixels. |
| `tileX`, `tileY` | number | |

### Packet table
Given to `onTankPacket` / `onSendPacketRaw`, and accepted by `sendPacketRaw`.

| Field | Type |
|---|---|
| `type` | number |
| `netID`, `targetNetID` | number |
| `flags` | number |
| `floatVar`, `intVar` | number |
| `posX`, `posY` | number |
| `xSpeed`, `ySpeed` | number |
| `secondaryVar` | number |
| `tileX`, `tileY` | number |

The extra data of a packet is not exposed, only the 56-byte header fields. Example, equip the item in hand slot (type 10):

```lua
sendPacketRaw({type = 10, intVar = 242})
```

### Variant table
Given to `onVariant`. Argument *n* of the call is `v<n>`, so `v1` is the function name.

| Field | |
|---|---|
| `v1`, `v2`, ... | Strings stay strings; int, uint and float arguments become numbers. Vector arguments are left out (the field is `nil`). |
| `netID` | Net id of the packet. |

## 10. Not available

Not available: `runThread`, `getNPCList`, `fetch`, `writeToLocal`, `createPlayer`, `Preferences`, `UserInterface` and every other UI function, ImGui, coroutines, shortcuts.

## 11. Example scripts

Three samples are written to the Scripts folder the first time the tab opens:

- `example.lua`: `log`, `sleep`.
- `whoami.lua`: prints your player, the tile below you, and counts of players and drops.
- `hooks.lua`: logs your outgoing text packets and hides console lines containing `secret`. Stays loaded until Stop.

```lua
-- walk to a tile, then punch the block to its right
local me = getLocal()
if me and FindPath(me.tileX + 3, me.tileY) then
  while not isOnPos(me.tileX + 3, me.tileY) do sleep(100) end
  tileChange(me.tileX + 4, me.tileY, 18)
end
```

## 12. Notes for developers

- Adding a function: register it in `features/lua/api*.cpp`, and add its name to `LuaHighlighter.API` so the editor colors it. Do not raise a Lua error while a `std::string` or `std::vector` is alive: Lua is built as C and errors `longjmp`.
- Host test of the sandbox: `tests/lua_engine_test.cpp` (build line in its header).
- Not yet checked on a device: packet table field names, `tileChange`, `enterPos`, `dropItem`, `getPing`, and the layout of the game's item list behind `getInventory`.
