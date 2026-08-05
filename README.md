# ESP32 Stock & Crypto Ticker

A live stock, cryptocurrency, and foreign exchange ticker for the 7" touchscreen display. Built with LVGL 9, LovyanGFX, FreeRTOS, and a suite of free-tier data APIs.

![Version](https://img.shields.io/badge/version-v9.0.7-blue)
![Platform](https://img.shields.io/badge/platform-ESP32--S3-green)
![Framework](https://img.shields.io/badge/framework-Arduino%20%2B%20PlatformIO-orange)

---

## Features

### Current release — v9.0.7

- **UI animation polish** — eased page transitions, staggered card entrance animation, price-change flash, card press feedback, animated status bar icons and popups, softer splash fade-out
- **Price alerts** — per-symbol above/below price thresholds with a status bar bell icon, amber card notification, and tap-to-acknowledge popup; optional piezo buzzer support is wired but inert pending a confirmed spare GPIO
- **OTA firmware updates** — check for and install new firmware directly from Settings > About, sourced from a GitHub Releases manifest
- **Background daily update check** — a status bar icon appears automatically when a newer version is available; installing is always a separate, manual step
- **Bootloader rollback protection** — a failed update automatically reverts to the previous working firmware, with an on-screen notice if this happens
- **Boot splash screen** — full-color 800×480 image shown for ~3.5s at startup, stored in LittleFS (not compiled into the firmware image)
- **Multi-AP Wi-Fi** — up to 5 saved networks, tried in order of scan signal strength before falling back to the provisioning screen
- **Wi-Fi signal-strength indicator** — 4-bar status bar icon, RSSI-tiered, distinct from the Wi-Fi association and internet-reachability text
- **Live prices** — crypto via CoinGecko, equities via Twelve Data or Yahoo Finance
- **8-cell watchlist grid** — up to 8 symbols per page, unlimited pages with auto-rotate and swipe navigation
- **Three chart styles** — sparkline, area fill (gradient), and candlestick — global default with per-symbol override
- **FX screen** — dedicated foreign exchange rate table with 32 supported currencies via Frankfurter API
- **Local display currency** — convert all prices to your preferred currency, shown alongside the native price on every card
- **LSE equity support** — London Stock Exchange stocks via Yahoo Finance supplementary data source (per-symbol selectable)
- **Enhanced detail screen** — tap any card for a full candlestick chart with:
  - Timeframe selector (1D / 1W / 1M / 3M)
  - SMA overlay (configurable period)
  - Toggleable RSI panel (configurable period)
  - Prev-close and today's-open change % shown side by side (equities)
- **On-device Wi-Fi provisioning** — scan and connect directly on the touchscreen; no phone or captive portal required
- **On-device timezone configuration** — ~35 curated zones with a DST toggle, no reflash needed to relocate a unit
- **Stale / offline data states** — clear visual distinction between fresh, stale, and failed price data per card
- **Internet-reachability indicator** — status bar shows Wi-Fi association and true internet reachability as two separate signals
- **API rate-limit management** — auto-derived Twelve Data poll interval from your daily budget and symbol count, plus a shared per-minute call gate across every Twelve Data consumer (live poll, seed backfill, on-demand OHLC)
- **CoinGecko Demo API key support** — resolves multi-unit shared-IP rate limiting
- **Tabbed settings screen** — left-hand sidebar (Watchlist / FX Pairs / Display / API & Rate Limits / Wi-Fi / Alerts / About), each independently scrollable
- **No compile-time secrets** — Wi-Fi credentials and the Twelve Data API key are entered on-device and stored in NVS only; the same firmware binary can be flashed to every unit

---

## Hardware

| Component | Detail |
|-----------|--------|
| Board | Sunton ESP32-8048S070C |
| MCU | ESP32-S3 |
| Flash | 16MB |
| PSRAM | 8MB octal |
| Display | 7" 800×480 RGB-parallel (EK9716 driver) |
| Touch | GT911 capacitive, I2C |

### Required hardware modification

The GT911 touch controller's INT pin is not connected to the ESP32 by default, forcing I2C polling every LVGL tick and driving CPU usage above 80%.

**The R17 mod fixes this:**
1. Bridge resistor **R17** with a 0Ω resistor → routes GT911 INT to GPIO18
2. Remove resistor **R5** (pull-up — the GT911 INT pin cannot sink enough current through it)
3. Remove the unpopulated **XPT2046** footprint (U1) if present

Full documentation with photos: [ESP3D-TFT Hardware Mod for Sunton 7" GT911](https://esp3d.io/esp3d-tft/version_1x/hardware/esp32-s3/sunton-70-8048/)

Without this mod the firmware still works — set `TOUCH_INT_PIN` to `GPIO_NUM_NC` in `display_setup.h`.

---

## Data Sources

| Asset class | API | Key required | Notes |
|-------------|-----|-------------|-------|
| Crypto prices | [CoinGecko](https://www.coingecko.com/en/api) | No (optional Demo key) | Free keyless tier; Demo key raises rate limits |
| Equity prices | [Twelve Data](https://twelvedata.com) | Yes (free tier) | 800 calls/day free; US/major exchanges only |
| LSE equities | [Yahoo Finance](https://finance.yahoo.com) | No | Unofficial API; used as supplementary source for exchanges not on Twelve Data free tier |
| FX rates | [Frankfurter](https://www.frankfurter.dev) | No | 32 currencies, ECB reference rates, updated daily |
| Crypto OHLC | CoinGecko `/coins/{id}/ohlc` | No | On-demand, detail screen only |
| Equity OHLC | Twelve Data `/time_series` | Yes | On-demand, detail screen only |
| LSE OHLC | Yahoo Finance v8 chart | No | On-demand, detail screen only |

---

## Getting Started

### Prerequisites

- Wi-Fi network (2.4GHz — the ESP32-S3 does not support 5GHz)
- Free API keys: [Twelve Data](https://twelvedata.com/pricing) and optionally [CoinGecko Demo](https://www.coingecko.com/en/api)

On first boot, the device will prompt for Wi-Fi credentials on-screen. Scan for your network, tap it, and enter the password using the on-screen keyboard. Set your timezone afterward in Settings > Wi-Fi (defaults to UTC until then).

### First-time Wi-Fi setup

The device uses **on-device provisioning** as its primary Wi-Fi setup method:
- On first boot (or when saved credentials fail), a network scan screen appears
- Tap your network SSID from the list
- Enter your password using the touchscreen keyboard (monospace font to help distinguish similar characters)
- The device connects, saves credentials to NVS flash, and reboots into the main grid

If on-device provisioning fails repeatedly, the device falls back to a **WiFiManager captive portal** — connect your phone to the `Ticker-XXXX` access point and follow the portal instructions.

---

## Usage

### Main grid screen

The main screen shows up to 8 symbol cards per page. Each card displays:
- Symbol name and asset type (crypto / equity / forex)
- Current price in native currency
- Converted price in your chosen local currency (if different)
- 24h change percentage (green = up, red = down)
- Mini chart (sparkline, area fill, or candlestick — per your settings)

**Navigation:**
- Pages auto-rotate on a configurable timer (default 30s)
- Swipe left/right to manually change page
- Tap a card to open the detail screen
- Long-press a card to override the chart style for that symbol
- Tap the **gear icon** (status bar, right) to open Settings
- Tap the **currency icon** (status bar, left of gear) to open the FX screen

### FX screen

Shows a scrollable rate table of your tracked currency pairs. Each row shows:
- Currency pair (e.g. EUR/USD)
- Current rate
- 24h change %
- 5-day rate range (note: Frankfurter provides daily ECB rates, not intraday — "5d range" reflects daily movements)

Tap **+** to add a new pair. Long-press a row to pin/unpin it to the main watchlist grid.

The **Display currency selector** at the top of the FX screen sets which currency is used for price conversion across the whole app.

### Detail screen

Tap any card to see a full-screen candlestick chart for that symbol. Features:
- **Timeframe tabs:** 1D / 1W / 1M / 3M — tap to switch
- **SMA overlay:** moving average line on the chart (period configurable in Settings)
- **RSI panel:** tap the RSI button to show/hide the RSI indicator below the chart

### Settings screen

Reach via the gear icon. A left-hand tab sidebar with five independently
scrollable sections:

| Section | What you can configure |
|---------|----------------------|
| Watchlist | Add/remove symbols, select data source (Twelve Data or Yahoo Finance) per equity |
| FX Pairs | Remove pairs, toggle pin-to-watchlist per pair |
| Display | Brightness, chart style, page rotate interval, MA period, RSI period, RSI default visibility |
| API & Rate Limits | Twelve Data key, daily budget, per-minute limit, CoinGecko Demo key |
| Wi-Fi | Saved networks list (up to 5, add/forget), timezone (~35 curated zones, DST toggle) |

---

## Configuration persistence

| Data | Storage |
|------|---------|
| Wi-Fi credentials | NVS (Preferences library) |
| API keys | NVS |
| Display preferences | NVS |
| Symbol watchlist | LittleFS (`/config.json`) |
| FX pairs | LittleFS (`/config.json`) |
| Local display currency | LittleFS (`/config.json`) |
| Per-symbol settings | LittleFS (`/config.json`) |

All storage is **power-loss tolerant** — NVS and LittleFS are both designed to survive interrupted writes without corruption. No SD card is used or required.

---

## Version history

| Version | Key features |
|---------|-------------|
| **v9.0.7** | UI/animation polish (card entrance/press/flash animations, eased page transitions, animated popups), price alerts (per-symbol above/below threshold, status bar bell, card notification, tap-to-acknowledge), OTA install heap/progress diagnostics |
| **v8.0.0** | OTA firmware updates (GitHub Releases manifest + dual-slot bootloader rollback), background daily update check with status bar indicator, two-step manual check/install flow, rollback failure notice |
| **v7.0.0** | Multi-AP Wi-Fi (up to 5 saved networks, scan-strength-ordered reconnect), Wi-Fi signal-strength status bar indicator, boot splash screen (LittleFS-stored) |
| **v6.1.0** | On-device timezone configuration, removed hardcoded Wi-Fi/API credentials (no compile-time secrets left), tabbed settings screen, internet-reachability status indicator, Twelve Data per-minute call gate, detail-screen prev-close/today's-open stats |
| **v6.0.0** | FX screen (Frankfurter), local currency conversion, Yahoo Finance for LSE equities, CoinGecko Demo key, timeframe selector, SMA + RSI on detail screen |
| **v4.0.0** | Paging (auto-rotate + swipe), display settings, area fill gradient chart, Twelve Data rate-limit auto-derivation, per-symbol chart style override |
| **v2.0.0** | Live prices (CoinGecko + Twelve Data), on-device Wi-Fi provisioning, four-state stale/offline data model |
| **v1.0.0** | Display and touch driver bring-up, NTP sync, 8-cell grid screen, custom candlestick widget |

---

## Roadmap

v9.0.7 is the current final version.  Furture releases will include Portfolio Management, Currency Converter, Headline News, Earnings Calendar, Fear & Greed (crypto).

---

## Technical notes

### Colour palette

All up/down colouring uses **Material Design A400** values, chosen after testing on the TN LCD panel which hue-shifts desaturated colours significantly at off-axis viewing angles:

| Semantic use | Hex |
|-------------|-----|
| Up / positive | `#00E676` (Green A400) |
| Down / negative | `#FF1744` (Red A400) |
| Stale (amber accent) | `#BA7517` |
| Error (red accent) | `#E24B4A` |
| Card background | `#0D1219` |
| Screen background | `#0A0E14` |

### Networking architecture

All HTTPS calls run on a **dedicated FreeRTOS task pinned to core 1**. Results are written to a mutex-protected shared struct that the LVGL UI thread (core 0) reads on its own refresh timer. The UI never calls the network directly — this prevents HTTP blocking calls from stalling the display.

### Multi-unit deployment

This firmware is designed to run on multiple physical units at different locations. Key considerations:
- Each unit provisions its own Wi-Fi credentials on-screen — no hardcoded network config
- A **CoinGecko Demo API key** is strongly recommended for multi-unit deployments — keyless CoinGecko calls share a rate limit bucket per IP, so multiple units on the same network will hit 429 errors regardless of poll interval
- The Twelve Data **daily call budget** and **per-minute rate limit** fields in settings allow the firmware to auto-derive a safe poll interval — adjust these if you upgrade to a paid Twelve Data plan

---

## Acknowledgements

- Pin mapping sourced from [HarryVienna's Makerfabs-Sunton-ESP32-S3-7 reference config](https://github.com/HarryVienna/Makerfabs-Sunton-ESP32-S3-7-Display-with-LovyanGFX-and-LVGL)
- Hardware modification documented by [ESP3D-TFT project](https://esp3d.io/esp3d-tft/version_1x/hardware/esp32-s3/sunton-70-8048/)
- Built with [LVGL 9](https://lvgl.io), [LovyanGFX](https://github.com/lovyan03/LovyanGFX), [ArduinoJson](https://arduinojson.org), [WiFiManager](https://github.com/tzapu/WiFiManager)

---

## License

MIT License — see [LICENSE](LICENSE) for details.
