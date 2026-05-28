# Watchy Menu Structure

How the on-watch menu system is laid out, how the four buttons behave in
every state, and where in the source each piece lives. Pairs with the
top-level [README](README.md).

## At a glance

```
WATCHFACE  ──MENU──►  MAIN MENU (5 items)
                          │
                          │ (selecting "Settings")
                          ▼
                      SETTINGS MENU (3 items)
                          │
                          │ (selecting an editor)
                          ▼
                      LEAF EDITOR (Set Time / Night Mode / Sync NTP)
                          │
                          │ (commit + return)
                          ▼
                      back to SETTINGS MENU
```

BACK always moves one level toward the watchface. Two minutes of inactivity
in any menu state automatically returns to the watchface.

## Button conventions

The four buttons keep the same role across every menu state:

| Button | In a menu (main / settings) | In an editor (Set Time / Night Mode) |
|---|---|---|
| **MENU**  | select the highlighted item | advance to the next field; from the last field, commit + exit |
| **BACK**  | go up one level (settings → main → watchface) | retreat to the previous field (no exit) |
| **UP**    | move highlight up (wraps) | increment the currently blinking value |
| **DOWN**  | move highlight down (wraps) | decrement the currently blinking value |

There is no separate "save" action — exiting the last field of an editor
is the commit signal.

## State machine

| State constant | Value | Reached from | Display |
|---|---|---|---|
| `WATCHFACE_STATE`     | -1 | boot / BACK from main menu / 2-min auto-close | the watchface |
| `MAIN_MENU_STATE`     |  0 | MENU from watchface / BACK from settings | 5-item list |
| `APP_STATE`           |  1 | inside a leaf function (editor / accel viewer / about / buzz / setupWifi) | per-leaf |
| `FW_UPDATE_STATE`     |  2 | (BLE OTA — disabled by default, see [BLE.cpp](src/BLE.cpp)) | OTA progress |
| `SETTINGS_MENU_STATE` |  3 | selecting "Settings" from the main menu | 3-item list |

Two `RTC_DATA_ATTR` variables persist between deep-sleep wakes inside the
menu system:

- `menuIndex` — currently-highlighted row in the main menu (0–4)
- `settingsMenuIndex` — currently-highlighted row in the settings submenu (0–2)
- `parentMenuState` — where a leaf function should return to. Set by the
  dispatch site before calling a leaf used from multiple menus (currently
  `setTime`, `showSyncNTP`, `showSetNightMode`). Anything other than
  `SETTINGS_MENU_STATE` means "return to the main menu".

## Main menu

Five entries, ordered top to bottom on the display:

| # | Item | Function | What it does |
|---|---|---|---|
| 0 | About Watchy        | `showAbout`         | Board info: hardware revision, firmware version, battery voltage, free heap |
| 1 | Vibrate Motor       | `showBuzz`          | Triggers the vibration motor (test that the haptic works) |
| 2 | Show Accelerometer  | `showAccelerometer` | Live X/Y/Z readings + step counter from the BMA423 |
| 3 | Setup WiFi          | `setupWifi`         | Opens the WPA2 setup AP (see [README §First-time WiFi setup](README.md#first-time-wifi-setup)) |
| 4 | Settings            | `showSettings`      | Opens the [Settings submenu](#settings-submenu) |

About, Vibrate Motor, Accelerometer and Setup WiFi return directly to the
main menu when they finish (their final `showMenu(menuIndex, false)` call).

## Settings submenu

Three entries:

| # | Item | Function | What it does |
|---|---|---|---|
| 0 | Set Time   | `setTime`           | Manual time entry with the blink-edit pattern (Hour → Minute → Year → Month → Day) |
| 1 | Sync NTP   | `showSyncNTP`       | Connects to WiFi, runs one NTP `forceUpdate()`, writes the result to the RTC |
| 2 | Night Mode | `showSetNightMode`  | Opens the [Night Mode editor](#night-mode-editor) |

All three return to the settings submenu via `_returnToPreviousMenu()` —
which consults `parentMenuState` (set to `SETTINGS_MENU_STATE` by the
dispatch site before the call) and picks `showSettings(...)` over
`showMenu(...)`.

## Night Mode editor

Three editable fields. The currently-selected field blinks; MENU advances,
BACK retreats, UP/DOWN change the value.

| Field | Range | Notes |
|---|---|---|
| **Interval** | 0–30 minutes | `0` disables night mode entirely (the editor displays `OFF`) |
| **Start**    | 0–23 (hour)  | first hour of the quiet window, inclusive |
| **End**      | 0–23 (hour)  | first hour after the quiet window, exclusive |

If `Start > End` the window wraps over midnight — the default `Start=22`,
`End=5` therefore covers 22:00 through 04:59.

**Save semantics**: the values are committed to RTC RAM **and** to NVS the
moment you press MENU on the last (End) field. There is no separate "save"
or "cancel" — if you want to bail without changing anything, just power-cycle
the watch before pressing MENU on the last field.

**Hourly vibration interaction**: `vibrateOClock` (still a compile-time
setting in `settings.h`) fires at minute=0 regardless of night-mode state,
because minute=0 is always a refresh minute for any interval that divides
60. Set `vibrateOClock = false` in `settings.h` for a silent night.

## Set Time editor

Five-field editor inherited from upstream — Hour → Minute → Year → Month →
Day. Same button conventions as Night Mode (MENU advances, BACK retreats,
UP/DOWN change). After the Day field, the values are written to the
hardware RTC and control returns to the settings submenu.

## Sync NTP

Connects to WiFi (using whatever credentials Setup WiFi stored), runs one
`forceUpdate()` against `settings.ntpServer` with `settings.gmtOffset`, and
prints the resulting time on the display for 3 s before returning to the
settings submenu. If WiFi connection fails it prints `WiFi Not Configured`
and bails after the same 3 s.

## Auto-close & lazy display init

Two efficiency mechanisms layer onto the menu system:

- **Auto-close**: in both `MAIN_MENU_STATE` and `SETTINGS_MENU_STATE` the
  next RTC tick (1 minute later) flips `alreadyInMenu` from `false` to
  `true`; the tick after that returns to the watchface. So a menu that
  isn't actively used closes itself in ~2 minutes.
- **Lazy display init**: `display.epd2.initWatchy()` is skipped when the
  watch wakes from the auto-close timer in either menu state, and
  `deepSleep()` only hibernates a panel that was actually brought up. This
  saves the ~50 ms panel boot for ticks that wouldn't redraw anything.

Both mechanisms also cover the night-mode "in-between" minutes — see the
README's [Night mode](README.md#whats-patched) entry.

## Persistence model

Three layers, fastest to slowest:

| Layer | Survives | Used for |
|---|---|---|
| `RTC_DATA_ATTR` | deep-sleep wakes (lost on power-off / hard reset) | `menuIndex`, `settingsMenuIndex`, `parentMenuState`, `guiState`, `currentWeather`, `lastSSID`, `lastIPAddress` |
| NVS (Preferences) | deep-sleep **and** power-off | edited `nightModeMinutes` / `nightModeStart` / `nightModeEnd` (under namespace `"watchy"`) |
| Compile-time `#define` in `settings.h` | flashed firmware | initial defaults for all three night-mode values when the NVS namespace doesn't exist yet |

On boot, `_loadPersistedSettings()` runs right after `RTC.init()` and
overrides the settings struct with whatever NVS holds. NVS wins over the
`settings.h` defaults — to reset to compile defaults you have to erase the
NVS partition (`nvs_flash_erase()` or `esptool.py erase_flash`).

## Implementation pointers

| Concept | Constants / functions / file |
|---|---|
| State values | [`src/config.h`](src/config.h) — `WATCHFACE_STATE`, `MAIN_MENU_STATE`, `SETTINGS_MENU_STATE`, `APP_STATE`, `FW_UPDATE_STATE` |
| Menu lengths | [`src/config.h`](src/config.h) — `MENU_LENGTH` (5), `SETTINGS_MENU_LENGTH` (3) |
| Night-mode field IDs | [`src/config.h`](src/config.h) — `NIGHT_FIELD_INTERVAL`, `NIGHT_FIELD_START`, `NIGHT_FIELD_END`, `NIGHT_FIELD_COUNT` |
| Button dispatch | [`src/Watchy.cpp`](src/Watchy.cpp) — `handleButtonPress`, `_dispatchMainMenu`, `_dispatchSettingsMenu` |
| Main menu rendering | [`src/Watchy.cpp`](src/Watchy.cpp) — `showMenu`, `showFastMenu` |
| Settings rendering | [`src/Watchy.cpp`](src/Watchy.cpp) — `showSettings`, `showFastSettings` |
| Night-mode editor | [`src/Watchy.cpp`](src/Watchy.cpp) — `showSetNightMode` |
| Return helper | [`src/Watchy.cpp`](src/Watchy.cpp) — `_returnToPreviousMenu` |
| NVS load/save | [`src/Watchy.cpp`](src/Watchy.cpp) — `_loadPersistedSettings`, `_savePersistedSettings` |
| Persisted-settings struct fields | [`src/Watchy.h`](src/Watchy.h) — `nightModeMinutes`, `nightModeStart`, `nightModeEnd` on `watchySettings` |

## Extending the menus

### Add a top-level menu item

1. Bump `MENU_LENGTH` in `src/config.h`.
2. Add the label to the `menuItems[]` array in **both** `showMenu` and
   `showFastMenu` (the `Edit` tool's `replace_all` on the literal works
   because both copies are identical).
3. Add a case to the `switch` in `_dispatchMainMenu`. Either call a
   blocking leaf function that returns to the main menu, or open another
   submenu.

### Add a Settings submenu item

1. Bump `SETTINGS_MENU_LENGTH` in `src/config.h`.
2. Add the label to the `items[]` array in both `showSettings` and
   `showFastSettings`.
3. Add a case to the `switch` in `_dispatchSettingsMenu`. The dispatch
   helper already sets `parentMenuState = SETTINGS_MENU_STATE` before
   calling, so a leaf that uses `_returnToPreviousMenu()` will naturally
   come back to the settings submenu.

### Add an NVS-persisted setting

1. Add the field to the `watchySettings` struct in `src/Watchy.h`. Place
   it at the end so existing initializer lists in `settings.h` continue to
   compile (missing fields zero-init).
2. Provide a compile default via `#define` in each `settings.example.h` /
   `settings.h` and reference it in the struct initializer.
3. Read/write the field in `_loadPersistedSettings` / `_savePersistedSettings`
   (use a short, unique key — NVS keys are limited to 15 characters).

### Add a new editor screen

Use `setTime` or `showSetNightMode` as a template. The pattern is:

1. Set `guiState = APP_STATE` at the top.
2. Copy current values into locals.
3. `pinMode(...INPUT)` each of the four buttons.
4. `while (1)` loop polling the buttons: MENU advances `setIndex`,
   BACK retreats, UP/DOWN change the value of the field at `setIndex`.
5. When `setIndex` overflows past the last field, `break` out of the loop.
6. Commit the locals back to the settings struct + NVS, then call
   `_returnToPreviousMenu()`.

That keeps the new editor consistent with the existing ones and inherits
the auto-close + lazy-display-init behaviour for free.
