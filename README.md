# ⚡️ LuaBoost v1.5.0 (WotLK 3.3.5a)

**Lua runtime optimizer + SmartGC + SpeedyLoad + UI Thrashing Protection for World of Warcraft 3.3.5a (build 12340)**  
Author: **Suprematist**

LuaBoost improves addon performance by eliminating GC stutter with per-frame incremental garbage collection, speeding up loading screens by suppressing noisy events, and preventing redundant UI widget updates across all addons.

Designed for **Warmane** and other 3.3.5a servers.

---

## ⭐ Reviews

See what other players say: [**Reviews & Testimonials**](https://github.com/suprepupre/wow-optimize/discussions/10)

---

## 🆕 What's New in v1.5.0

| Feature | Description |
|---------|-------------|
| **Event Profiler** | `/lb events` — profiles all WoW events for 10 seconds and shows top 15 by frequency. Color-coded: yellow (<20/sec), orange (20-50/sec), red (>50/sec). Helps identify event spam from addons. |
| **OnUpdate Dispatcher API** | `LuaBoost_RegisterUpdate(id, interval, callback)` — addons can register throttled callbacks without creating their own Frame objects. Built-in throttling and error handling. |
| **Frame Merge** | `timeFrame` + `gcFrame` merged into single `coreFrame`. One fewer C++ Frame object. |
| **Frame Consolidation** | 5 event frames merged into single master dispatcher. All 32 events routed through one C++ → Lua boundary. |
| **GC Memory Check Throttle** | `collectgarbage("count")` every 60 frames instead of every frame. |
| **ThrashGuard Pool Integration** | Widget cache tables use shared table pool. Less GC pressure. |

### Previous Highlights (v1.3.0-v1.4.0)
| Feature | Description |
|---------|-------------|
| **UI Thrashing Protection** | Caches StatusBar widget values and skips redundant engine calls. |
| **Safe StatusBar Hooks** | `SetValue`, `SetMinMaxValues`, `SetStatusBarColor`. 100% taint-free. |
| **Runtime Toggle** | `/lb tg toggle` to enable/disable without reload. |

---

## 🌍 Localization / Credits

- **English (`enUS`)**: Default
- **Korean (`koKR`)**: Translated by [**nadugi**](https://github.com/nadugi)
- **German (`deDE`)**: Translated by [**Raz0r1337**](https://github.com/Raz0r1337)

*Want to translate? Copy `enUS.lua`, rename to your locale, translate, add to TOC, submit PR!*

---

## ✅ Features

### 📊 Event Profiler (NEW in v1.5.0)
Type `/lb events` to start a 10-second event capture. Shows the top 15 most frequent events with color-coded frequency:
- 🟡 Yellow: < 20 events/sec (normal)
- 🟠 Orange: 20-50 events/sec (elevated)
- 🔴 Red: > 50 events/sec (excessive — likely an addon problem)

Helps you find which events are causing CPU load and which addons might need optimization.

### 🔄 OnUpdate Dispatcher API (NEW in v1.5.0)
Addons can register throttled update callbacks without creating Frame objects:
```lua
LuaBoost_RegisterUpdate("MyAddon_Update", 0.1, function(now, elapsed)
    -- runs every 0.1 seconds
    -- 'now' = cachedTime, 'elapsed' = frame elapsed
end)

LuaBoost_UnregisterUpdate("MyAddon_Update")
LuaBoost_GetUpdateCount() -- number of registered callbacks
```

### 🛡️ UI Thrashing Protection
Hooks widget metatable methods globally and caches the last value. If the new value is identical, the engine call is skipped.

**Hooked methods (100% Taint-Free):**
- `StatusBar:SetValue`
- `StatusBar:SetMinMaxValues`
- `StatusBar:SetStatusBarColor`

### Safe Runtime Optimizations (automatic, always active)
- `GetTimeCached()` — cached `GetTime()` value updated once per frame
- `LuaBoost_Throttle(id, interval)` — shared throttle helper
- Table pool: `LuaBoost_AcquireTable()` / `LuaBoost_ReleaseTable(t)`
- `GetDateCached(fmt)` — opt-in cached date helper

### Smart GC Manager (configurable)
- Per-frame incremental GC with **4-tier stepping**: loading → combat → idle → normal
- Emergency full GC when memory exceeds threshold (checked every 60 frames)
- GC burst on heavy events (boss kill, LFG popup, achievement)
- **3 presets**: Light / Standard / Heavy
- GUI: `ESC → Interface → AddOns → LuaBoost`

### SpeedyLoad — Fast Loading Screens
- Suppresses noisy events during loading screens
- **Safe** (11 events) or **Aggressive** (23 events) mode

### 🏗️ Optimized Architecture
- Single master event dispatcher for all 32 events
- `coreFrame` handles both time cache and GC stepping
- Minimal C++ Frame object count

---

## 🔧 Recommended Optimization Ecosystem

| Layer | Tool | What It Does |
|-------|------|--------------|
| **C / Engine** | [wow_optimize.dll](https://github.com/suprepupre/wow-optimize) | Faster memory allocator, full network latency stack, precision timers, Lua GC from C, combat log fix |
| **Lua / Runtime** | **!LuaBoost** | Smart GC, SpeedyLoad, UI Thrashing Protection, Event Profiler, OnUpdate API, table pool, GUI |

---

## ⚙️ Settings Reference

Open settings: `ESC → Interface → AddOns → LuaBoost → GC Settings`

### Presets Comparison

| Setting | Light (<150MB) | Standard (150-300MB) | Heavy (>300MB) |
|---------|------|-----|--------|
| Normal Step (KB/f) | 20 | 50 | 100 |
| Combat Step (KB/f) | 5 | 15 | 30 |
| Idle Step (KB/f) | 80 | 150 | 300 |
| Loading Step (KB/f) | 150 | 300 | 500 |
| Emergency GC (MB) | 150 | 300 | 500 |
| Idle Timeout (sec) | 15 | 15 | 20 |

---

## 🔧 Fixing Freezes After Boss Kills

**Symptom:** 5-10 second freeze after boss kill or dungeon queue pop.

**Quick Fix:** `/lb settings` → click **Heavy (> 300MB)** preset.

**Manual Fix:** Set Emergency Full GC to 500+ MB, Combat Step to 30+ KB/f.

---

## ⚠️ Conflicts

- **SmartGC** — Do NOT use together. SmartGC is integrated into LuaBoost.
- **KPack SpeedyLoad** — Disable KPack's SpeedyLoad if using LuaBoost's.

---

## 📦 Installation

```text
Interface/AddOns/!LuaBoost/
├── !LuaBoost.toc
├── LuaBoost.lua
├── enUS.lua
├── koKR.lua
└── deDE.lua
```

---

## 🧰 Commands

| Command | Description |
|---------|-------------|
| `/lb` or `/luaboost` | Status overview |
| `/lb gc` | GC stats + DLL stats |
| `/lb pool` | Table pool stats |
| `/lb toggle` | Enable/disable GC manager |
| `/lb force` | Force full GC now |
| `/lb sl` | Toggle SpeedyLoad |
| `/lb sl safe` | SpeedyLoad safe mode |
| `/lb sl agg` | SpeedyLoad aggressive mode |
| `/lb tg` | UI Thrashing Protection stats |
| `/lb tg toggle` | Enable/disable ThrashGuard |
| `/lb tg reset` | Reset ThrashGuard counters |
| `/lb events` | Profile events for 10 seconds |
| `/lb updates` | Show registered update callbacks |
| `/lb settings` | Open GC settings panel |
| `/lb help` | Show all commands |

---

## 📜 License

MIT License — do whatever you want with it.
