# Patched Watchy

A fork of the official [SQFMI Watchy](https://watchy.sqfmi.com) Arduino library with
security, privacy and reliability patches applied to the upstream `src/` tree
and example watchfaces.

![Watchy](https://watchy.sqfmi.com/img/watchy_render.png)

## What's patched

### Security / privacy
- **TLS-validated weather requests.** `_getWeatherData` switches to
  `https://api.openweathermap.org` and validates the cert against an embedded
  ISRG Root X1 (Let's Encrypt) — see `WATCHY_ROOT_CA_ISRG_X1` in
  [src/Watchy.cpp](src/Watchy.cpp). API key and location no longer leak in
  cleartext, and on-path tampering of the `timezone` field can't shift the
  clock anymore.
- **WPA2-protected setup AP.** The `Watchy AP` config portal now requires
  the password defined by `WIFI_AP_PASSWORD` in
  [src/config.h](src/config.h) (default `watchysetup`). The setup screen
  shows SSID + password instead of the watch's MAC address.
- **"Setup WiFi" doesn't wipe saved credentials.** The menu entry now opens
  `startConfigPortal()` directly instead of calling `resetSettings()` first —
  an accidental tap or portal timeout no longer leaves the watch
  unconfigured.
- **BLE OTA flagged unsafe.** The unauthenticated, unsigned OTA path remains
  disabled (as upstream); [src/BLE.cpp](src/BLE.cpp) now carries a prominent
  warning block listing what needs to be added before it can be re-enabled
  safely (BLE pairing with MITM protection, ESP32 Secure Boot v2 + signed
  images, explicit end-of-stream marker).
- **No more shared/leaked OWM API key in tracked files.** Each watchface
  ships a `settings.example.h` template with a `YOUR_OWM_API_KEY`
  placeholder; the real `settings.h` is `.gitignore`d.

### Reliability
- **JSON response validation.** Weather updates only commit to RTC RAM when
  the response parses and contains every field we read (`main`, `weather`,
  `sys`, `timezone`). HTML error pages and partial responses no longer poison
  the displayed temperature, sunrise/sunset or GMT offset.
- **Task watchdog.** A 30 s WDT (`WDT_TIMEOUT_SECONDS` in
  [src/config.h](src/config.h)) covers each wake cycle and reboots the
  board on hangs. Deep sleep pauses the WDT automatically; `setupWifi`
  detaches the task around the portal call so user-paced input doesn't
  trigger it. Works on both ESP-IDF 4.x and 5.x (Arduino Core 2.x / 3.x).
- **Lazy display init.** `display.epd2.initWatchy()` is skipped on wake
  cycles that never touch the panel (menu auto-close ticks; USB plug/unplug
  while not on the watchface). `deepSleep()` only hibernates a panel that
  was actually brought up.

## Setup

1. Open one of the example sketches under [examples/WatchFaces](examples/WatchFaces).
2. Copy `settings.example.h` to `settings.h` in the same folder.
3. Put your own OpenWeatherMap API key in the `OPENWEATHERMAP_APIKEY` line
   (free key at <https://openweathermap.org/api>) and adjust `CITY_ID` /
   `GMT_OFFSET_SEC` for your location.
4. (Optional) Override the AP password by defining `WIFI_AP_PASSWORD` in
   your build config before `#include "config.h"`, or edit
   [src/config.h](src/config.h) directly.
5. Build & flash from the Arduino IDE with Watchy V1/V1.5/V2/V3 selected as
   the target board.

`settings.h` is `.gitignore`d on purpose — your API key, home location and
GMT offset stay out of git history. The committed `settings.example.h`
templates only carry placeholders.

## Hardware

Buy Watchy from
[Mouser](https://www.mouser.com/ProductDetail/SQFMI/SQFMI-WATCHY-10?qs=DRkmTr78QARN9VSJRzqRxw%3D%3D),
[The Pi Hut](https://thepihut.com/collections/sqfmi) or
[Crowd Supply](https://www.crowdsupply.com/sqfmi/watchy).
Cases & accessories: [shop.sqfmi.com](https://shop.sqfmi.com).

## Upstream

- Original project: <https://watchy.sqfmi.com>
- Getting-started docs: <https://watchy.sqfmi.com/docs/getting-started>
- Community Discord: <https://discord.gg/ZXDegGV8E7>
