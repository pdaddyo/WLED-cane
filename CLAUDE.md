# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read this first

`AGENTS.md` (repo root) is the authoritative reference for build commands, environments,
C++/web code style, naming conventions, the usermod pattern, memory/PSRAM rules, hot-path
rules, security hardening, and PR etiquette. **Follow it.** The detailed coding-convention
docs it links live in `docs/` (`cpp.instructions.md`, `web.instructions.md`,
`hardening.instructions.md`, `securecode.instructions.md`, etc.). This file does not repeat
that content — it covers the runtime architecture that AGENTS.md does not.

Quick reminders that bite if forgotten:
- Run `npm ci && npm run build` **before** `pio run` — the web UI build generates the
  `wled00/html_*.h` / `wled00/js_*.h` C headers that firmware compilation needs.
- **Never edit or commit** `wled00/html_*.h` / `wled00/js_*.h` — they are generated from
  `wled00/data/`. Edit the source in `data/`, then rebuild.
- C++ files use **2-space** indent; web files (`wled00/data/`) use **tabs**.
- Changes to `platformio.ini` require maintainer approval.

## What this is

WLED is single-binary C++ firmware for ESP32/ESP8266 that drives addressable LED strips and
serves a web UI + JSON/HTTP API + realtime protocols. There is no OS threading model to lean
on (ESP8266 is single-core, cooperative); almost everything is driven from one cooperative
super-loop. Understanding that loop and the strip/segment/bus data model is the key to being
productive here.

## Runtime architecture

### The super-loop (`wled00/wled.cpp`)
`WLED::setup()` initializes filesystem, config, buses, network, and usermods; `WLED::loop()`
runs forever. Each iteration, in order: handle time/IR/connection/serial, run notifications
and transitions, call `userLoop()` + `UsermodManager::loop()`, service IO, then — unless a
realtime override is active — render via `strip.service()`. `yield()` is sprinkled throughout
to feed the WiFi stack and watchdog; **never block** in the loop, in usermod `loop()`, or in
effects. The loop also contains a heap-watchdog that progressively purges segments → resets
segments → re-creates the whole `WS2812FX` strip object if contiguous free heap stays below
`MIN_HEAP_SIZE` for 15/30/45 seconds. Low-RAM degradation is a designed-in behavior, not a bug.

### Strip → Segment → Bus model (the heart of the system)
Defined in `wled00/FX.h`, implemented across `FX_fcn.cpp`, `FX_2Dfcn.cpp`, `FX.cpp`:
- **`WS2812FX strip`** — the global singleton (`extern WS2812FX strip`). Owns segments and
  the global render pipeline. `strip.service()` is called once per loop; it iterates segments,
  and for each due segment calls its effect function, applies brightness/transition/blending,
  and pushes the result to the buses.
- **`Segment`** — a contiguous (or 2D matrix) span of the LED space with its own effect mode,
  palette, colors, speed/intensity, and per-segment scratch state. Effects read/write pixels
  *within segment-local coordinates*; the segment maps them to physical bus pixels. Segment
  data (effect working memory) is heap-allocated and is the first thing freed under memory
  pressure.
- **Effects** live in `FX.cpp` as `mode_*()` functions registered in a mode table; 2D effects
  use the helpers in `FX_2Dfcn.cpp`. `FXparticleSystem.cpp` is a shared particle engine used
  by several effects. These are the hottest code paths — see AGENTS.md "Hot-Path Code" rules.
- **Buses** (`bus_manager.h/.cpp`, `bus_wrapper.h`) abstract the physical output hardware
  (digital WS2812-style, analog/PWM, network/virtual). `BusManager` holds the configured
  buses; the strip writes finished pixel data to them. Pin allocation goes through
  `pin_manager` (`PinOwner` enum), which prevents two features from claiming the same GPIO.

### Configuration & persistence
Config lives on the LittleFS filesystem as JSON. `cfg.cpp` (de)serializes the full config
(`cfg.json`), `set.cpp` handles HTML settings-form POSTs, and `wled00/data/` settings pages
feed both. Presets and playlists (`presets.cpp`, `playlist.cpp`) are JSON documents that
capture/replay state and can chain API calls. ArduinoJSON (bundled under `wled00/src/`) is
used throughout; buffer sizes are tight — respect existing size constants.

### Network & control surfaces (all converge on the same global state)
- **HTTP/web** — `wled_server.cpp` (AsyncWebServer routes), `xml.cpp` (legacy XML/HTML state).
- **JSON API** — `json.cpp` is the canonical state API (`/json/state`, `/json/info`, etc.).
  Most external control and the modern UI go through here. New state fields generally need
  handling in both serialize and deserialize paths here.
- **Realtime / sync** — `udp.cpp` (WLED UDP notifier + sync), `e131.cpp` (E1.31/Art-Net/DDP),
  `mqtt` , Alexa (`alexa.cpp`), Hue sync, `ir.cpp` (infrared), `button.cpp`. Realtime modes
  can take over the strip and bypass effect rendering (see the `realtimeMode` checks in the
  loop).
- **Usermods** — `um_manager.cpp` dispatches lifecycle callbacks (`setup/loop/addToConfig/
  readFromConfig/addToJsonInfo/...`) to every registered usermod. Usermods are the supported
  extension point; see AGENTS.md "Usermod Pattern" and `usermods/EXAMPLE/`.

### Compile-time feature model
Behavior is configured heavily by preprocessor flags (`WLED_DISABLE_*` / `WLED_ENABLE_*`,
platform macros), resolved in `wled.h` and `const.h`. Flag spelling matters — a misspelled
flag is silently ignored. `wled.h` holds global `extern` declarations and the build version;
`const.h` holds enums, IDs (including usermod IDs), and limits. Disabling features is how WLED
fits on tiny ESP8266 flash/RAM.

## Where things are

| Area | Files |
|---|---|
| Main loop / setup / globals | `wled.cpp`, `wled.h`, `const.h`, `fcn_declare.h` |
| Effects + strip/segment engine | `FX.cpp`, `FX.h`, `FX_fcn.cpp`, `FX_2Dfcn.cpp`, `FXparticleSystem.cpp` |
| Output hardware | `bus_manager.cpp/.h`, `bus_wrapper.h`, `pin_manager.cpp` |
| State API & config | `json.cpp`, `cfg.cpp`, `set.cpp`, `xml.cpp`, `presets.cpp`, `playlist.cpp` |
| Network / realtime / control | `wled_server.cpp`, `udp.cpp`, `e131.cpp`, `mqtt.cpp`, `alexa.cpp`, `ir.cpp`, `button.cpp` |
| Color / palettes | `colors.cpp`, `palettes.cpp` |
| Web UI source (edit here, then rebuild) | `wled00/data/` |
| Build tooling / tests | `tools/cdata.js`, `tools/cdata-test.js`, `pio-scripts/` |
| Community extensions | `usermods/` (base new ones on `usermods/EXAMPLE/`) |

## Testing

There are **no C++ unit tests** — firmware is validated by successful compilation across
target environments (`pio run -e esp32dev` is the common smoke check). The only JS test suite
covers the web-UI build tool: `npm test` (or `node --test tools/cdata-test.js`). CI compiles
~22 firmware targets plus the web build on every PR (`.github/workflows/wled-ci.yml`).
