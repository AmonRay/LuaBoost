# ⚡️ LuaBoost v1.3.0 (WotLK 3.3.5a)

**Lua runtime optimizer + SmartGC + SpeedyLoad + UI Thrashing Protection for World of Warcraft 3.3.5a (build 12340)**  
Author: **Suprematist**

LuaBoost improves addon performance by eliminating GC stutter with per-frame incremental garbage collection, speeding up loading screens by suppressing noisy events, and preventing redundant UI widget updates across all addons.

Designed for **Warmane** and other 3.3.5a servers.

---

## 🆕 What's New in v1.3.0

| Feature | Description |
|---------|-------------|
| **UI Thrashing Protection** | Automatically caches widget values and skips redundant engine calls. Speeds up all addons that update UI every frame (unitframes, nameplates, status bars). Zero configuration needed — works globally on all addons. |
| **6 Hooked Methods** | `SetText`, `SetFormattedText`, `SetTextColor`, `SetValue`, `SetMinMaxValues`, `SetStatusBarColor` — safe methods only, no taint, no visual bugs. |
| **Runtime Toggle** | `/lb tg toggle` to enable/disable without reload. `/lb tg` for live stats including skip rate. |
| **Cache Stats** | Real-time hit rate monitoring in GUI and via slash commands. Track how many redundant calls are being eliminated. |

---

## 📋 Version History

| Version | Key Changes |
|---------|-------------|
| **v1.3.0** | UI Thrashing Protection (ThrashGuard) — global widget call deduplication |
| **v1.2.2** | 100% Taint-Free — removed `math.*` and `table.insert` global overrides |
| **v1.2.1** | UI/UX polish, GC preset names based on memory usage |
| **v1.2.0** | Localization support (English, Korean) |
| **v1.1.0** | SpeedyLoad, DLL integration, protection hooks |
| **v1.0.0** | Initial release — SmartGC, table pool, throttle API |

---

## 🌍 Localization / Credits

- **English (`enUS`)**: Default
- **Korean (`koKR`)**: Translated and optimized by [**nadugi**](https://github.com/nadugi)

*If you want to translate LuaBoost to your language, feel free to submit a Pull Request! Just copy `enUS.lua`, rename it to your locale (e.g., `ruRU.lua`), translate the strings, and add it to `!LuaBoost.toc`.*

---

## ✅ Features

### 🛡️ UI Thrashing Protection (NEW in v1.3.0)

Poorly written addons call UI methods every single frame even when the value hasn't changed:
```lua
-- This happens 60 times per second in many unitframe addons:
HealthText:SetText("100%")  -- same value, but engine recalculates font geometry every time
HealthBar:SetValue(45000)   -- same value, but engine recomputes bar fill every time
```

LuaBoost hooks widget metatable methods globally and caches the last value. If the new value is identical, the engine call is **completely skipped** — saving font geometry recalculation, texture coordinate updates, and color pipeline flushes.

**Hooked methods (6):**

| Widget Type | Method | What It Saves |
|-------------|--------|---------------|
| FontString | `SetText` | Font geometry, glyph layout |
| FontString | `SetFormattedText` | Format + font geometry |
| FontString | `SetTextColor` | Color pipeline flush |
| StatusBar | `SetValue` | Bar fill recomputation |
| StatusBar | `SetMinMaxValues` | Range recalculation |
| StatusBar | `SetStatusBarColor` | Color pipeline flush |

**NOT hooked (unsafe — values can change via C++ animations):**
- `Texture:SetTexture` — atlas system, dynamic textures
- `Texture:SetTexCoord` — scrolling texture animations
- `Texture:SetVertexColor` — color animations
- `Region:SetAlpha` — fade in/out animations

**Safety:**
- ✅ 100% Taint-Free — these are non-secure rendering methods
- ✅ GC-safe — cache uses weak keys (`__mode = "k"`), dead widgets collected automatically
- ✅ Error-safe — each hook wrapped in `pcall`, failure doesn't break other hooks
- ✅ Compatible with ElvUI, oUF, Pitbull, TellMeWhen, WeakAuras, Skada, DBM
- ✅ Enabled by default, can be toggled at runtime

**Typical performance impact:**

| Scenario | Skip Rate | Engine Calls Saved |
|----------|-----------|-------------------|
| 25-man raid, unitframes updating HP text | 85-95% | ~1400/sec |
| Dalaran with 50+ nameplates | 80-90% | ~2500/sec |
| Idle in city, addons ticking | 90-98% | ~450/sec |
| Boss fight with status bars | 70-90% | ~800/sec |

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

Open settings: `ESC → Interface → AddOns → LuaBoost`

### UI Thrashing Protection

| Setting | What It Does |
|---------|-------------|
| **Enable UI Thrashing Protection** | Master toggle. Hooks widget methods to skip redundant calls. Enabled by default. |

**Runtime commands:**
- `/lb tg` — show stats (skip rate, cached widgets, hook count)
- `/lb tg toggle` — enable/disable at runtime (no reload needed)
- `/lb tg reset` — reset skip/pass counters

**API for addon developers:**
```lua
-- Get current stats
local skipped, passed, hooks, active, widgets = LuaBoost_GetThrashStats()

-- Force a widget to refresh on next call (invalidate cache)
LuaBoost_InvalidateWidget(myFontString)
```

### Step Sizes — KB collected per frame

| Setting | What it does | Increase if... | Decrease if... |
|---------|-------------|----------------|----------------|
| **Normal Step** | GC work per frame during regular gameplay | Memory climbs over time | You want minimal CPU overhead |
| **Combat Step** | GC work per frame **during combat** | Big freeze after boss kills | Combat FPS is unstable |
| **Idle Step** | GC work per frame while AFK/idle | Memory stays high while AFK | Not important |
| **Loading Step** | GC work per frame during loading screens | Loading screens feel slow | Not important |

### Thresholds

| Setting | What it does | Increase if... | Decrease if... |
|---------|-------------|----------------|----------------|
| **Emergency Full GC (MB)** | Force full GC outside combat when memory exceeds this | Freezes after boss kills (set 500+) | Want memory to stay low |
| **Idle Timeout (sec)** | Seconds without activity before idle GC mode | Want faster AFK cleanup | Don't want aggressive GC while reading chat |

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

**Symptom:** A 5–10 second freeze right after killing a boss, leaving combat, or when a dungeon queue pops.

**Cause:** Lua memory spikes during combat. When combat ends, Emergency Full GC triggers and blocks the entire game.

### Quick Fix
Open settings (`/lb settings`) and click the **Heavy (> 300MB)** preset.

### Manual Fix
1. Set **Emergency Full GC** to **500 MB** (or higher, up to 1000).
2. Set **Combat Step** to **30** KB/f (or higher, up to 50).

---

## ⚠️ Conflicts

### SmartGC
**Do NOT use SmartGC together with LuaBoost.**
SmartGC has been integrated into LuaBoost. Using two GC managers simultaneously will conflict.

### KPack SpeedyLoad
**Disable KPack's SpeedyLoad module if you enable LuaBoost's SpeedyLoad.**
Both addons suppress the same events during loading screens.

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
| `/lb tg` | UI Thrashing Protection stats |
| `/lb tg toggle` | Enable/disable ThrashGuard at runtime |
| `/lb tg reset` | Reset ThrashGuard counters |
| `/lb settings` | Open GC settings panel |
| `/lb help` | Show all commands |

---

## 🔧 Recommended Combo

For maximum optimization, use this addon together with **[wow_optimize.dll](https://github.com/suprepupre/wow-optimize)**:

| Layer | Tool | What It Does |
|-------|------|--------------|
| **C / Engine** | wow_optimize.dll | Faster memory, I/O, network, timers, Lua allocator + GC from C, combat log fix |
| **Lua / Addons** | !LuaBoost addon | Incremental GC, SpeedyLoad, ThrashGuard, table pool, throttle API, GUI |

When both are installed, the DLL handles Lua allocator replacement, GC stepping from C (zero Lua overhead), and combat log buffering. The addon provides the GUI, combat awareness, idle detection, SpeedyLoad, UI Thrashing Protection, and runtime function optimizations.

---

## 🧠 Technical Details: UI Thrashing Protection

### How It Works

```
Without ThrashGuard:
  OnUpdate → addon calls SetText("100%") → engine recalculates font geometry
  OnUpdate → addon calls SetText("100%") → engine recalculates font geometry (WASTED)
  OnUpdate → addon calls SetText("100%") → engine recalculates font geometry (WASTED)
  ... 60 times per second, identical work

With ThrashGuard:
  OnUpdate → addon calls SetText("100%") → hook checks cache → MISS → engine call → cache "100%"
  OnUpdate → addon calls SetText("100%") → hook checks cache → HIT → return (SKIPPED)
  OnUpdate → addon calls SetText("100%") → hook checks cache → HIT → return (SKIPPED)
  OnUpdate → addon calls SetText("99%")  → hook checks cache → MISS → engine call → cache "99%"
```

### Implementation Details

- **Metatable hooking**: Replaces methods on widget type metatables (`FontString.__index.SetText`, etc.)
- **Weak cache**: `setmetatable({}, { __mode = "k" })` — widgets are keys, cache tables are values. When a widget is garbage collected, its cache entry disappears automatically.
- **String interning**: Lua 5.1 interns all strings, so `"100%" == "100%"` is a pointer comparison — essentially free.
- **SetFormattedText optimization**: Formats the string in Lua via `string.format`, then checks cache before calling the original `SetText`. Avoids double formatting.
- **SetMinMaxValues invalidation**: When min/max range changes, the cached `SetValue` is invalidated since the bar needs to recalculate fill even if the numeric value is the same.
- **Error isolation**: Each hook is wrapped in `pcall`. If one hook fails to install, all others continue working.

### Why These 6 Methods Are Safe

The key requirement is that the engine value can **only** change through the Lua API call we're hooking. If the engine can change the value through a different path (C++ animations, internal state changes), our cache becomes stale and we'd skip legitimate updates.

| Method | Why Safe |
|--------|----------|
| `SetText` | Text content only changes via `SetText` or `SetFormattedText` Lua calls |
| `SetFormattedText` | Delegates to `SetText` — shares the same cache |
| `SetTextColor` | Font color is not affected by AnimationGroups in 3.3.5a |
| `SetValue` | StatusBar value only changes via `SetValue` Lua call |
| `SetMinMaxValues` | Range only changes via `SetMinMaxValues` Lua call |
| `SetStatusBarColor` | Bar color is not animated by the engine in 3.3.5a |

### Why These Methods Are NOT Hooked

| Method | Why Unsafe |
|--------|-----------|
| `SetTexture` | Atlas system, dynamic textures, multi-argument solid color form |
| `SetTexCoord` | Scrolling texture animations modify coords via C++ |
| `SetVertexColor` | Color animations modify vertex colors via C++ |
| `SetAlpha` | Fade animations modify alpha via C++, desync causes invisible widgets |

---

## 📜 License

MIT License — do whatever you want with it.