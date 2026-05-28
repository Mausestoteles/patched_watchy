# Patched Watchy

A fork of the official [SQFMI Watchy](https://watchy.sqfmi.com) Arduino library with
security, privacy, reliability and UX patches applied to the upstream `src/`
tree and example watchfaces.

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
- **BLE OTA gated behind `WATCHY_ENABLE_BLE_OTA`.** The unauthenticated /
  unsigned OTA path no longer compiles into the binary by default. The flag
  is documented in [src/config.h](src/config.h) and the security warning
  block at the top of [src/BLE.cpp](src/BLE.cpp) lists the prerequisites
  (BLE pairing with MITM protection, ESP32 Secure Boot v2 + signed images,
  explicit end-of-stream marker) that MUST be added before re-enabling it.
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

### UX

- **QR-code on the setup AP screen.** `_configModeCallback` renders a
  scannable `WIFI:T:WPA;S:Watchy AP;P:<pw>;;` URI as a 33×33-module QR
  (version 4, ECC_LOW) centered on the e-paper. Point an iOS or Android
  camera at the watch — the phone joins the AP automatically and opens the
  captive portal. SSID and password are still printed underneath as a
  fallback. Requires the [QRCode](https://github.com/ricmoo/QRCode)
  library; if it isn't on the include path the code falls back to a
  text-only screen via `__has_include`, so the build never breaks because
  of a missing QR dependency.

## Setup

1. **Install dependencies.** The library lists everything it needs in
   [library.properties](library.properties) — Arduino IDE picks them up via
   *Library Manager* automatically when you add the library, but if you need
   to install them by hand:
   ```
   Adafruit GFX Library, Arduino_JSON, DS3232RTC, NTPClient,
   Rtc_Pcf8563, GxEPD2, WiFiManager, QRCode
   ```
   Or via `arduino-cli`:
   ```sh
   arduino-cli lib install "Adafruit GFX Library" Arduino_JSON DS3232RTC \
                           NTPClient Rtc_Pcf8563 GxEPD2 WiFiManager QRCode
   ```
2. Open one of the example sketches under
   [examples/WatchFaces](examples/WatchFaces).
3. **Copy `settings.example.h` to `settings.h`** in the same folder.
4. Put your own OpenWeatherMap API key in the `OPENWEATHERMAP_APIKEY` line
   (free key at <https://openweathermap.org/api>) and adjust `CITY_ID` /
   `GMT_OFFSET_SEC` for your location.
5. *(Optional)* Override the AP password by defining `WIFI_AP_PASSWORD` in
   your build config before `#include "config.h"`, or edit
   [src/config.h](src/config.h) directly. The default is `watchysetup`.
6. Build & flash from the Arduino IDE with Watchy V1/V1.5/V2/V3 selected as
   the target board.

`settings.h` is `.gitignore`d on purpose — your API key, home location and
GMT offset stay out of git history. The committed `settings.example.h`
templates only carry placeholders.

## Using the watch

### First-time WiFi setup

1. From the watchface, press *Menu* → *Setup WiFi*.
2. The screen shows a QR code plus the SSID/password underneath.
3. Scan the QR with your phone camera (iOS Camera, Google Lens, or any QR
   scanner). The phone joins the `Watchy AP` access point and opens the
   captive portal automatically. If you can't scan, connect manually with
   the printed password.
4. In the portal, pick your home WiFi and enter the password — the watch
   stores it and reboots into the watchface.

Subsequent taps on *Setup WiFi* re-open the portal **without** wiping the
stored credentials — if you cancel or it times out, your previous WiFi is
still there.

### Weather

Weather refresh happens every `WEATHER_UPDATE_INTERVAL` minutes
(default 30) over HTTPS. If the request fails or the response is
malformed, the previous reading stays put; if it succeeds, the watch also
syncs NTP using the timezone reported by OpenWeatherMap.

## Advanced

### Re-enabling BLE OTA (NOT RECOMMENDED)

The BLE-based firmware-update path in [src/BLE.cpp](src/BLE.cpp) is
disabled at compile time. **Do not re-enable it as-is** — it accepts
unsigned firmware images from any BLE peer in range. If you've added BLE
pairing with MITM protection, signed-image verification via Secure Boot v2,
and an explicit end-of-stream marker, you can enable the build path with:

```sh
arduino-cli compile --build-property "build.extra_flags=-DWATCHY_ENABLE_BLE_OTA" ...
```

You'll also need to wire `updateFWBegin()` back into the menu state machine
in [src/Watchy.cpp](src/Watchy.cpp) (currently behind a block comment).

### Updating the embedded TLS root cert

`WATCHY_ROOT_CA_ISRG_X1` in [src/Watchy.cpp](src/Watchy.cpp) is valid until
**2035-06-04**. If OpenWeatherMap migrates to a different CA before then,
the weather request will fall back to the internal temperature sensor.
Replace the cert string with the new root CA in PEM format to restore
HTTPS weather updates.

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
