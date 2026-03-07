# ⚡ LuaBoost v1.2.2 (WotLK 3.3.5a)

**Lua runtime optimizer + SmartGC + SpeedyLoad for World of Warcraft 3.3.5a (build 12340)**  
Author: **Suprematist**

LuaBoost improves addon performance by eliminating GC stutter with per-frame incremental garbage collection and speeding up loading screens by suppressing noisy events. 

Designed for **Warmane** and other 3.3.5a servers.

---

## 🆕 What's New in v1.2.2

| Feature | Description |
|---------|-------------|
| **100% Taint-Free** | Completely removed `math.*` and `table.insert` global overrides. This permanently fixes "Action Blocked" (Taint) errors with the Blizzard UI (secure frames) and resolves crashes in addons like Skada that pass string values to math functions. |
| **UI/UX Polish** | GC presets are now clearly named based on memory usage: `Light (< 150MB)`, `Standard (150-300MB)`, and `Heavy (> 300MB)`. |

*(For older changes like Localization and Teleport Guard, see previous releases).*

---

## 🌍 Localization / Credits

- **English (`enUS`)**: Default
- **Korean (`koKR`)**: Translated and optimized by [**nadugi**](https://github.com/nadugi)

*If you want to translate LuaBoost to your language, feel free to submit a Pull Request! Just copy `enUS.lua`, rename it to your locale (e.g., `ruRU.lua`), translate the strings, and add it to `!LuaBoost.toc`.*

---

## ✅ Features

### Safe Runtime Optimizations (automatic, always active)
- `GetTimeCached()` — cached `GetTime()` value updated once per frame
- `LuaBoost_Throttle(id, interval)` — shared throttle helper for any addon
- Table pool: `LuaBoost_AcquireTable()` / `LuaBoost_ReleaseTable(t)` / `LuaBoost_GetPoolStats()`
- `GetDateCached(fmt)` — opt-in cached date helper (does **not** replace global `date()`)

### Smart GC Manager (configurable)
- Stops Lua auto-GC and performs **incremental GC steps every frame**
- **4-tier stepping**: loading → combat → idle → normal
  - **Loading**: aggressive cleanup (no rendering happening anyway)
  - **Combat**: minimal GC to protect frametime
  - **Idle**: aggressive cleanup while AFK
  - **Normal**: balanced stepping
- Emergency full GC when Lua memory exceeds threshold (outside combat)
- GC burst on heavy events (boss kill, LFG popup, achievement, loot spam)
- **3 presets**: Light / Standard / Heavy — pick based on your addon load
- GUI: `ESC → Interface → AddOns → LuaBoost`

### SpeedyLoad — Fast Loading Screens
- Temporarily suppresses noisy events during loading screens
- Restores all events after loading completes (with event replay)
- **Two modes**:
  - **Safe** (11 events) — cosmetic events only, no risk
  - **Aggressive** (23 events) — includes actionbar/aura/inventory events, faster but slightly more aggressive

### Protection Hooks (optional, OFF by default)
- **Intercept `collectgarbage()`** — blocks other addons from forcing full GC spikes
- **Block `UpdateAddOnMemoryUsage()`** — prevents CPU spikes from frequent memory scans

### DLL Integration (optional)
Works with **[wow_optimize.dll](https://github.com/suprepupre/wow-optimize)**:
- DLL does GC stepping from C (via Sleep hook) — zero Lua overhead
- DLL replaces Lua allocator with mimalloc — faster allocation, less fragmentation
- DLL reads `LUABOOST_ADDON_COMBAT`, `LUABOOST_ADDON_IDLE`, `LUABOOST_ADDON_LOADING` globals
- DLL adjusts step size based on 4-tier state
- Stats available via `/lb gc`

---

## ⚙️ Settings Reference

Open settings: `ESC → Interface → AddOns → LuaBoost → GC Settings`

### Step Sizes — KB collected per frame

These control how much garbage the GC collects **every single frame**. Higher = more cleanup per frame (uses slightly more CPU) but keeps memory lower. Lower = less CPU but memory accumulates faster.

| Setting | What it does | Increase if... | Decrease if... |
|---------|-------------|----------------|----------------|
| **Normal Step** | GC work per frame during regular gameplay | Memory climbs over time; you see gradual FPS degradation | You want minimal CPU overhead; you have very few addons |
| **Combat Step** | GC work per frame **during combat**. | You get a **big freeze after boss kills** (garbage accumulated during combat) | Combat FPS is unstable; you're CPU-limited |
| **Idle Step** | GC work per frame while AFK/idle | Memory stays high even while AFK | Not important — idle mode uses zero rendering resources |
| **Loading Step** | GC work per frame during loading screens | Loading screens feel slow | Not important — no rendering during loading |

### Thresholds

| Setting | What it does | Increase if... | Decrease if... |
|---------|-------------|----------------|----------------|
| **Emergency Full GC (MB)** | When Lua memory exceeds this value (outside combat), LuaBoost forces a **full garbage collection**. Auto-raises if the collection takes >50ms (capped at 1000 MB). | **You get freezes after boss kills** or dungeon queue pops. Set to 500+ for heavy addon setups. | You want memory to stay low and don't mind occasional brief pauses |
| **Idle Timeout (sec)** | Seconds without player activity before switching to idle GC mode | You want faster cleanup during brief AFK moments | You don't want aggressive GC to kick in while reading chat/AH |

### Presets Comparison

| Setting | Light (<150MB) | Standard (150-300MB) | Heavy (>300MB) |
|---------|------|-----|--------|
| Normal Step (KB/f) | 20 | 50 | 100 |
| Combat Step (KB/f) | 5 | 15 | 30 |
| Idle Step (KB/f) | 80 | 150 | 300 |
| Loading Step (KB/f) | 150 | 300 | 500 |
| Emergency GC (MB) | 150 | 300 | 500 |
| Idle Timeout (sec) | 15 | 15 | 20 |

**How to choose:**
- If you have very few addons → **Light**
- Start with **Standard** (default)
- If you use DBM + WA + Skada + ElvUI and get post-boss freezes → switch to **Heavy**

---

## 🔧 Fixing Freezes After Boss Kills / Dungeon Queue Pops

**Symptom:** A 5–10 second freeze right after killing a boss, leaving combat, or when a dungeon queue pops. The game becomes completely unresponsive.

**Cause:** When you fight a boss (especially in 25-man raids), Lua memory spikes rapidly. When combat ends, the Emergency Full GC triggers because memory exceeded the threshold, causing a **full garbage collection** that blocks the entire game.

### Quick Fix
Open settings (`/lb settings`) and click the **Heavy (> 300MB)** preset.

### Manual Fix
1. Set **Emergency Full GC** to **500 MB** (or higher, up to 1000). This prevents the full GC from triggering right after combat.
2. Set **Combat Step** to **30** KB/f (or higher, up to 50). This collects more garbage *during* combat so less accumulates.

---

## ⚠️ Conflicts

### SmartGC
**Do NOT use SmartGC together with LuaBoost.**
SmartGC has been integrated into LuaBoost. Using two GC managers simultaneously will conflict.

### KPack SpeedyLoad
**Disable KPack's SpeedyLoad module if you enable LuaBoost's SpeedyLoad.**
Both addons suppress the same events during loading screens. Running both will cause double-suppression.

---

## 📦 Installation

Copy the addon folder into your WoW directory:

```text
Interface/AddOns/!LuaBoost/
├── !LuaBoost.toc
├── LuaBoost.lua
├── enUS.lua
└── koKR.lua
```

*(Make sure the folder is named `!LuaBoost` so it loads early).*

---

## 🧰 Commands

| Command | Description |
|---------|-------------|
| `/lb` or `/luaboost` | Status overview |
| `/lb gc` | GC stats + DLL stats (if present) |
| `/lb pool` | Table pool stats |
| `/lb toggle` | Enable/disable GC manager |
| `/lb force` | Force full GC now |
| `/lb sl` | Toggle SpeedyLoad on/off |
| `/lb sl safe` | Enable SpeedyLoad in safe mode (11 events) |
| `/lb sl agg` | Enable SpeedyLoad in aggressive mode (23 events) |
| `/lb settings` | Open GC settings panel |
| `/lb help` | Show all commands |

---

## 📜 License

MIT License — do whatever you want with it.