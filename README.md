# ⚡️ LuaBoost v1.6.0 (WotLK 3.3.5a)

**Lua runtime optimizer + SmartGC + SpeedyLoad + UI Thrashing Protection for World of Warcraft 3.3.5a (build 12340)**
Author: **Suprematist**

LuaBoost improves addon performance by eliminating GC stutter with per-frame incremental garbage collection, speeding up loading screens by suppressing noisy events, and preventing redundant UI widget updates across all addons.

Designed for **Warmane** and other 3.3.5a servers.

---

## ⭐ Reviews

See what other players say: [**Reviews & Testimonials**](https://github.com/suprepupre/wow-optimize/discussions/10)

---

## 🆕 What's New in v1.6.0

| Feature | Description |
|---------|-------------|
| **GC Step Sync to DLL** | Addon GUI now controls DLL GC step sizes. Slider changes propagate to wow_optimize.dll within ~250ms via Lua globals. |
| **UI Cache Stats** | `/lb` and `/lb gc` show DLL UI cache skip rate when wow_optimize.dll is active. |
| **Smart ThrashGuard** | Auto-disables Lua-level StatusBar hooks when DLL is detected (DLL handles same methods at C level). |
| **Tooltip Throttle Fix** | Same-target tooltip calls pass through immediately — fixes flickering in RaidRoll and similar addons. |
| **Emergency GC Guard** | Emergency full GC skipped if current frame is already slow (>33ms) — prevents compounding lag spikes. |
| **Table Pool Safety** | Tables with metatables are rejected from the pool — prevents __gc/__index issues on reuse. |

### Previous Highlights (v1.5.x)

| Feature | Description |
|---------|-------------|
| **Event Profiler** | `/lb events` — profiles all events for 10 seconds, shows top 15. |
| **FPS Monitor** | `/lb fps` — 10-second capture with min/max/avg/1% low/stutter detection. |
| **OnUpdate Dispatcher** | `LuaBoost_RegisterUpdate(id, interval, callback)` — shared throttled callbacks. |
| **UI Thrashing Protection** | StatusBar metatable hooks — Lua-side fallback when DLL not present. |
| **SpeedyLoad** | Event suppression during loading screens (safe/aggressive modes). |

---

## 🌍 Localization

- **English (`enUS`)**: Default
- **Korean (`koKR`)**: Translated by [**nadugi**](https://github.com/nadugi)
- **German (`deDE`)**: Translated by [**Raz0r1337**](https://github.com/Raz0r1337)

*Want to translate? Copy `enUS.lua`, rename to your locale, translate, add to TOC, submit PR!*

---

## ✅ Features

### 🔄 GC Step Sync (NEW in v1.6.0)

When wow_optimize.dll is installed, the addon writes GC step sizes to Lua globals on every settings change. The DLL reads them and applies automatically — no restart needed.

```
Addon GUI slider → LUABOOST_ADDON_STEP_NORMAL → DLL reads → Config.normalStepKB
```

This means:
- **One place to configure** — the addon GUI
- **Zero Lua overhead** — DLL does the actual GC stepping from C
- **Instant feedback** — changes apply within ~250ms

### 📊 UI Cache Stats (NEW in v1.6.0)

When the DLL is active, `/lb` and `/lb gc` display:

```
  UI Cache: 72% skip (14523 skipped, 5621 passed)
```

### 🛡️ UI Thrashing Protection

Hooks widget metatable methods globally and caches the last value. If the new value is identical, the engine call is skipped.

**Auto-disabled when DLL is detected** — DLL hooks the same methods at C level (faster, taint-free). Shows "TG:DLL" in login message.

**Hooked methods (when no DLL, 100% Taint-Free):**
- `StatusBar:SetValue`
- `StatusBar:SetMinMaxValues`
- `StatusBar:SetStatusBarColor`

### 🎯 Tooltip Throttle

Throttles `GameTooltip:SetUnit`, `SetSpell`, `SetHyperlink` to max 10 updates/sec **per target**. Same-target repeat calls always pass through — prevents flickering in RaidRoll and similar addons.

### Safe Runtime Optimizations (automatic, always active)

- `GetTimeCached()` — cached `GetTime()` value updated once per frame
- `LuaBoost_Throttle(id, interval)` — shared throttle helper
- Table pool: `LuaBoost_AcquireTable()` / `LuaBoost_ReleaseTable(t)`
- `GetDateCached(fmt)` — opt-in cached date helper

### Smart GC Manager (configurable)

- Per-frame incremental GC with **4-tier stepping**: loading → combat → idle → normal
- Emergency full GC when memory exceeds threshold (guarded by frame time)
- GC burst on heavy events (boss kill, LFG popup, achievement)
- **3 presets**: Light / Standard / Heavy
- GUI: `ESC → Interface → AddOns → LuaBoost`

### SpeedyLoad — Fast Loading Screens

- Suppresses noisy events during loading screens
- **Safe** (11 events) or **Aggressive** (23 events) mode

### 📊 Event Profiler

Type `/lb events` for 10-second event capture. Shows top 15 by frequency:
- 🟡 Yellow: < 20/sec (normal)
- 🟠 Orange: 20-50/sec (elevated)
- 🔴 Red: > 50/sec (excessive)

### 📈 FPS Monitor

Type `/lb fps` for 10-second frametime capture with:
- Average, median, min, max FPS
- 1% low FPS
- Stutter detection (frames > 3× average)

### 🔄 OnUpdate Dispatcher API

```lua
LuaBoost_RegisterUpdate("MyAddon_Update", 0.1, function(now, elapsed)
    -- runs every 0.1 seconds
end)

LuaBoost_UnregisterUpdate("MyAddon_Update")
LuaBoost_GetUpdateCount()
```

---

## 🔧 Recommended Optimization Ecosystem

| Layer | Tool | What It Does |
|-------|------|--------------|
| **C / Engine** | [wow_optimize.dll](https://github.com/suprepupre/wow-optimize) | Faster memory, I/O, network, timers, Lua GC from C, combat log fix, UI widget cache (10 hooks) |
| **Lua / Runtime** | **!LuaBoost** | GC step sync to DLL, SpeedyLoad, diagnostics, table pool, GUI |

> 💡 **With wow_optimize.dll v1.8.0+**: DLL handles all UI widget caching at C level and reads step sizes from LuaBoost GUI. ThrashGuard auto-disables. The addon provides settings, diagnostics, and non-UI optimizations.

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

When wow_optimize.dll is installed, these values are synced to the DLL automatically.

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
| `/lb` or `/luaboost` | Status overview + UI cache stats |
| `/lb gc` | GC stats + DLL stats + UI cache stats |
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
| `/lb fps` | FPS monitor for 10 seconds |
| `/lb updates` | Show registered update callbacks |
| `/lb settings` | Open GC settings panel |
| `/lb help` | Show all commands |

---

## 📜 License

MIT License — do whatever you want with it.