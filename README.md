# ESPHome PN532 Component (Enhanced)

[![ESPHome Compile](../../actions/workflows/compile.yml/badge.svg)](../../actions/workflows/compile.yml)

An enhanced external ESPHome component for the PN532 NFC/RFID reader. Drop-in compatible with the ESPHome native `pn532_spi` / `pn532_i2c` / `pn532` API, with critical bug fixes and new reliability features.

## Improvements Over the Native Component

### Bug Fixes

| Issue | Fix |
|---|---|
| **Flapping tag detection** (#9875) — card on reader causes alternating `on_tag`/`on_tag_removed` | Same UID detected → stay present without triggering RF cycle |
| **I2C freeze after hours** (#3281, #4745) — device stops responding, requires power cycle | Health check with configurable auto-reset |
| **Blocking operation warning** — `Component pn532 took a long time` floods logs | Exponential backoff throttle after bus failures |
| **Setup retry** — single firmware version check failure marks component failed | 2-attempt retry on `GetFirmwareVersion` at boot |
| **WiFi interference** — PN532 RF field causes WiFi disconnects on nearby ESPs | RF field disabled between polls by default |

### New Features

- **Health check** with auto-reset: periodically re-validates communication and reinitialises if needed
- **Exponential backoff** on bus errors: 5s → 10s → 60s intervals to prevent log flooding
- **`rf_field_enabled`** option: explicitly control whether the RF field stays on between polls
- Binary sensor UID accepts both `74-10-37-94` (native format) and `74:10:37:94` formats
- Both SPI and I2C variants share a common base, making fixes apply to both

---

## Installation

Add to your ESPHome YAML:

```yaml
external_components:
  - source: github://JohnMcLear/esphome_pn532
    components: [pn532, pn532_spi, pn532_i2c]
    refresh: 1d
```

---

## Over SPI

```yaml
spi:
  clk_pin: GPIO18
  miso_pin: GPIO19
  mosi_pin: GPIO23

pn532_spi:
  cs_pin: GPIO5
  irq_pin: GPIO4    # optional; replaces bus readiness polling with GPIO read
  reset_pin: GPIO7  # optional; hardware reset on boot and re-init
  update_interval: 1s
  health_check_enabled: true
  health_check_interval: 60s
  max_failed_checks: 3
  auto_reset_on_failure: true
  rf_field_enabled: false
  on_tag:
    then:
      - logger.log:
          format: "Tag: %s"
          args: ['x.c_str()']
  on_tag_removed:
    then:
      - logger.log:
          format: "Tag removed: %s"
          args: ['x.c_str()']

binary_sensor:
  - platform: pn532
    name: "My NFC Tag"
    uid: "74-10-37-94"
```

### SPI Configuration Variables

- **`cs_pin`** (**Required**): Chip select pin.
- **`update_interval`** (*Optional*, default `1s`): How often to scan for tags.
- **`on_tag`** (*Optional*): Automation triggered when a tag is detected. Variable `x` is the UID string e.g. `74-10-37-94`.
- **`on_tag_removed`** (*Optional*): Automation triggered when a tag is removed. Variable `x` is the last UID.
- **`health_check_enabled`** (*Optional*, default `true`): Enable periodic health checks.
- **`health_check_interval`** (*Optional*, default `60s`): How often to run the health check.
- **`max_failed_checks`** (*Optional*, default `3`): Failures before marking component as unhealthy and attempting reset.
- **`auto_reset_on_failure`** (*Optional*, default `true`): Attempt re-initialisation when unhealthy.
- **`rf_field_enabled`** (*Optional*, default `false`): Keep the RF field on between polls. Leaving this `false` (default) reduces WiFi interference.
- **`irq_pin`** (*Optional*): GPIO input wired to the PN532 IRQ pin (active-low). When configured, readiness checking in `loop()` becomes a GPIO read instead of an SPI/I2C bus transaction — near-zero overhead between scans. Common on Elechouse v3/v4 boards.
- **`reset_pin`** (*Optional*): GPIO output wired to the PN532 RSTPD_N pin. When configured, a hardware reset is performed on boot and on every re-initialisation attempt. Common on Elechouse v3/v4 boards.
- **`spi_id`** (*Optional*): Manually specify the SPI bus ID.
- **`id`** (*Optional*): Manually specify the component ID.

---

## Over I²C

```yaml
i2c:
  sda: GPIO21
  scl: GPIO22

pn532_i2c:
  irq_pin: GPIO4    # optional; replaces bus readiness polling with GPIO read
  reset_pin: GPIO7  # optional; hardware reset on boot and re-init
  update_interval: 1s
  health_check_enabled: true
  health_check_interval: 60s
  max_failed_checks: 3
  auto_reset_on_failure: true
  rf_field_enabled: false
  on_tag:
    then:
      - homeassistant.tag_scanned: !lambda 'return x;'

binary_sensor:
  - platform: pn532
    name: "My NFC Tag"
    uid: "74-10-37-94"
```

### I²C Configuration Variables

All the same options as SPI above (including `irq_pin` and `reset_pin`), plus:

- **`address`** (*Optional*, default `0x24`): I2C address of the PN532.
- **`i2c_id`** (*Optional*): Manually specify the I2C bus ID.

---

## `pn532` Binary Sensor

```yaml
binary_sensor:
  - platform: pn532
    pn532_id: pn532_board   # required if multiple PN532 instances
    name: "My Tag"
    uid: "74-10-37-94"      # hyphen or colon separated hex
```

### Binary Sensor Configuration Variables

- **`uid`** (**Required**): The UID to match. Hyphen-separated hex: `74-10-37-94`. Colon-separated is also accepted: `74:10:37:94`.
- **`pn532_id`** (*Optional*): The ID of the `pn532_spi` or `pn532_i2c` hub. Required when you have more than one.
- All other options from [Binary Sensor](https://esphome.io/components/binary_sensor/).

---

## Setting Up Tags

To discover a tag's UID, configure without any binary sensors first:

```yaml
pn532_spi:
  cs_pin: GPIO5
  update_interval: 1s
  on_tag:
    then:
      - logger.log:
          format: "Found tag: %s"
          args: ['x.c_str()']
```

Flash this, open the logs, and scan your tag. You'll see:

```
Found new tag '74-10-37-94'
```

Copy that UID into a `binary_sensor` entry.

---

## Health Check

The health check periodically re-issues the `GetFirmwareVersion` command to verify the PN532 is still responding. If it fails `max_failed_checks` times consecutively:

1. The component is marked as errored in ESPHome.
2. If `auto_reset_on_failure: true`, a re-initialisation is attempted automatically.
3. On successful recovery, normal operation resumes.

This resolves the known long-running freeze issue on I2C (#3281).

---

## Known Issues (Upstream) Addressed Here

| ESPHome Issue | Description | Fix in this component |
|---|---|---|
| [#9875](https://github.com/esphome/esphome/issues/9875) | Tag causes alternating on/off when left on reader | Same UID early-return without RF cycle |
| [#3281](https://github.com/esphome/issues/issues/3281) | I2C freezes after a few hours | Health check + auto-reset |
| [#4745](https://github.com/esphome/issues/issues/4745) | Blocking reads cause `took a long time` warnings | Backoff throttle + non-blocking read ready |
| [esphome-core#201](https://github.com/esphome/esphome-core/issues/201) | Some tags never fire `on_tag` | Proper vector UID comparison |
| [PR#1046](https://github.com/esphome/esphome/pull/1046) | RF field causes WiFi interference | `rf_field_enabled: false` by default |

---

## Compatibility

- ESPHome 2024.x and later
- ESP32 (Arduino & ESP-IDF frameworks)
- ESP8266 (Arduino framework)
- PN532 modules over SPI or I2C (4-wire or 2-wire mode)

---

## ⚠️ WARNING: COUNTERFEIT PN532 MODULES

Many PN532 modules sold on common marketplaces are counterfeit or clones. These modules often:
- Pass the initial version check (reporting Firmware v1.6).
- **Fail** during actual tag polling, leading to I2C/SPI timeouts and "took a long time" warnings in ESPHome.
- Exhibit highly unstable behavior compared to legitimate NXP chips.

**We are currently investigating ways to automatically detect and handle these counterfeit devices.** If you are experiencing persistent "Timed out waiting for readiness" errors, you likely have a counterfeit module. Please join the [ESPHome Discord](https://esphome.io/guides/faq.html#where-can-i-get-help) or community forums to discuss how best to handle these clones.

---

## Supported Card Types


This component currently supports **ISO14443A** cards (106 kbit/s), which includes:
- Mifare Classic (1k, 4k)
- Mifare Ultralight / Ultralight C
- NTAG series (203, 213, 215, 216)

Support for ISO14443B, FeliCa, and Jewel cards is physically possible with the PN532 but is not yet implemented in this component's polling logic.

---

## TODO

- [x] Ensure esphome code conventions are referenced and used throughout component
- [x] Fix CI build
- [x] **SPI Hardware Support:** Confirmed stable initialization and reliable tag reading over SPI.
- [x] **Multi-Tag Support:** Successfully implemented simultaneous detection and parsing of up to 2 tags per polling cycle.
- [x] **Tag Persistence:** Implemented robust removal persistence (3-cycle threshold) to eliminate LED/trigger flapping.
- [ ] **NTAG216 Write Stability:** Resolve the issue where NTAG216 modules time out during NDEF write operations (even with a 3000ms readiness timeout and inter-page delays).
- [ ] **Mifare Authentication Fix:** Investigate and resolve persistent authentication failures on some Mifare Classic tags, ensuring fallback keys and sector-specific authentication are handled correctly.
- [ ] **Multi-Type Polling:** Extend `InListPassiveTarget` logic to optionally poll for ISO14443B and FeliCa tags.
- [ ] **Robust Counterfeit Detection:** Research and implement automated detection of counterfeit/clone modules using deeper silicon-level checks (e.g., `Diagnose` 0x00 command RAM/ROM tests and CIU register bitmask validation).
