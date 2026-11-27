# Garage Door Controller (ESPHome)

ESP32-based garage door controller for Home Assistant. It uses two reed switches for position feedback, a relay to trigger the opener, a buzzer relay for audible warnings, and an auto-close timer you can adjust and turn on or off from Home Assistant.

- **Device name:** `garagedoorcontroller`
- **Board:** ESP32 Relay x2, a 2-channel relay board built on an ESP32-WROOM-32E (`esp32dev`, ESP-IDF framework). See [Hardware](#hardware)
- **Config file:** `garagedoorcontroller.yaml`
- **Validated against:** ESPHome 2026.11.0

---

## Features

- Template **cover** entity, so Home Assistant shows a native garage door with open, close and stop controls
- Open and closed state comes from two reed switches
- Warning beeps before every open or close, including manual commands from HA
- **Auto-close timer** that starts when the door reaches fully open
  - Turned on or off with a switch in HA. **Always comes back on after a reboot or power loss**
  - Timeout adjustable from 1 to 30 minutes with a slider in HA (survives reboots)
  - Shows a live countdown, both in seconds and as `M:SS` text
  - Can be cancelled at any point, including during the warning beeps
- Built-in web interface on port 80, **protected by a username and password** (digest auth)
- **Encrypted OTA updates.** Firmware upload from the web UI is disabled
- Fallback hotspot (AP) with captive portal
- WiFi signal and uptime diagnostics

---

## Hardware

### Board: ESP32 Relay x2 (ESP32-WROOM-32E, 2-channel)

- **Amazon listing:** https://a.co/d/0dR3Fgg7 (ASIN `B0B8J9SNB5`)
- **ESPHome device page:** https://devices.esphome.io/devices/ESP32-Relay-x2/
- **Also sold as:** a generic "ESP32 2-channel relay module" on eBay and AliExpress, from many sellers

> The specs below come from the ESPHome device database entry for this board. The relay and LED pins in this config (GPIO16, GPIO17, GPIO23) match it exactly. Clone boards vary, so check the silkscreen and relay labels on your own unit.

| Spec | Value |
|---|---|
| MCU module | ESP32-WROOM-32E (`board: esp32dev`) |
| Relays | 2 relays, each with **COM / NO / NC** screw terminals |
| Relay rating | 10 A max load per relay (check the rating printed on the relay) |
| Power input | **7–30 VDC** or **5 VDC**, on separate terminals |
| USB-serial | None on board. Program through the 6-pin header with an external adapter (the ESPHome page names a CP2102N) |
| Programming header | 6-pin. **Not** an FTDI-compatible pinout, so check the silkscreen before connecting |
| GPIO breakout | Headers for every ESP32 GPIO, labeled on the back of the board |
| Onboard button | GPIO0 (boot / programming) |
| Onboard LED | GPIO23 |

#### Board pin functions

| GPIO | Board function | Used in this config as |
|---|---|---|
| GPIO16 | Relay 1 (outermost) | **Door opener relay** |
| GPIO17 | Relay 2 | **Buzzer relay** |
| GPIO23 | Onboard LED | `status_led` |
| GPIO0 | Onboard button | Not used. Keep it free for flashing |
| GPIO4 | Free header pin | Open reed switch |
| GPIO5 | Free header pin (strapping pin) | Closed reed switch |

#### Relay wiring for this project

- **Relay 1 (opener):** wire **COM** and **NO** in parallel with the opener's wall-button terminals. Pulsing the relay is then the same as pressing the button. Leave NC unconnected.
- **Relay 2 (buzzer):** wire the buzzer's supply through **COM** and **NO**, so the buzzer only sounds while the relay is pulsed.
- **Reed switches:** connect each one between its GPIO header pin and a **GND** header pin. No external resistors are needed, because the config enables the internal pullups.

#### Power

Use one input only: either 7–30 VDC on the wide-range terminal, or a regulated 5 VDC on the 5 V terminal. Don't connect both.

#### Flashing (first install)

1. Connect a 3.3 V USB-serial adapter to the 6-pin header: TX→RX, RX→TX, GND→GND. Match the silkscreen, because the pin order isn't FTDI-standard.
2. Hold the **GPIO0** button while the board powers up or resets, to enter the bootloader.
3. Flash with `esphome run garagedoorcontroller.yaml --device /dev/ttyUSB0`, or with ESPHome Web.
4. After that, update over encrypted OTA (see Build & Flash).

### Pin assignments

| Function | Substitution | GPIO | Notes |
|---|---|---|---|
| Door opener relay | `door_relay_pin` | GPIO16 | 500ms momentary pulse |
| Buzzer relay | `buzzer_relay_pin` | GPIO17 | 300ms momentary pulse |
| Door **open** reed switch | `door_open_sensor_pin` | GPIO4 | Contact closes with the magnet present. Internal pullup, `inverted: true` |
| Door **closed** reed switch | `door_closed_sensor_pin` | GPIO5 | Contact closes with the magnet present. Internal pullup, `inverted: true` (strapping pin, see Known Issues) |
| Status LED | `status_led_pin` | GPIO23 | Optional |

### Wiring notes

- Each reed switch connects between its GPIO and **GND**. The internal pullup holds the pin high, and the magnet closes the contact, pulling the pin low. `inverted: true` makes "magnet present" read as **ON**.
- **Fail-safe:** a broken wire or a jumper that has come off leaves the pin floating high, which reads **OFF**. The door then shows as "not at that position" rather than giving a false "closed".
- **If you swap a reed switch, check its type.** If the contact closes with the magnet, use `inverted: true`. If it opens with the magnet, use `inverted: false`.
- **Troubleshooting:** if a sensor never changes state, check its jumper or header connection at the ESP first. The closed-sensor problem during setup was a GPIO5 jumper that had come off the pin.
- The opener relay contacts go in parallel with the wall button on your opener. They should be dry contacts, so no voltage is fed into the opener.
- Both relays use `restore_mode: ALWAYS_OFF`, so neither pulses on boot or reboot.

---

## Configuration

### Substitutions

| Key | Default | Description |
|---|---|---|
| `door_relay_pin` | `GPIO16` | Opener relay output |
| `buzzer_relay_pin` | `GPIO17` | Buzzer relay output |
| `door_open_sensor_pin` | `GPIO4` | Fully-open reed switch |
| `door_closed_sensor_pin` | `GPIO5` | Fully-closed reed switch |
| `status_led_pin` | `GPIO23` | Status LED |
| `auto_close_timer_minutes` | `10` | Starting value for the timeout slider (**first boot only**) |
| `auto_close_enabled_value` | `true` | Auto-close state applied at **every boot** (not saved) |

> `auto_close_timer_minutes` is only a first-boot default. After that, the slider value saved in flash takes priority.
> `auto_close_enabled_value` is applied on **every** boot: auto-close isn't saved, so it always comes back on after a reboot or power loss. To have it remember your last setting instead, set `restore_value: true` on the `auto_close_enabled` global.

### Required secrets (`secrets.yaml`)

```yaml
api_key1: "<32-byte base64 API encryption key>"
wifi_ssid: "<primary SSID>"
wifi_password: "<primary password>"
2_wifi_ssid: "<secondary SSID>"
2_wifi_password: "<secondary password>"
wifi_local_ssid: "<fallback AP SSID>"
wifi_local_password: "<fallback AP password, min 8 chars>"
web_username: "<web UI username>"
web_password: "<web UI password>"
```

Generate an API key with:

```bash
openssl rand -base64 32
```

| Secret | Used for |
|---|---|
| `api_key1` | Home Assistant API encryption **and** OTA encryption (`ota: encryption: {}` inherits it) |
| `wifi_ssid` / `wifi_password` | Primary Wi-Fi network |
| `2_wifi_ssid` / `2_wifi_password` | Secondary Wi-Fi network |
| `wifi_local_ssid` / `wifi_local_password` | Fallback hotspot the device creates if it can't connect to Wi-Fi |
| `web_username` / `web_password` | Login for the local web interface |

> `ota_pass1` is no longer used and can be deleted from `secrets.yaml`.

---

## Web Interface & Security

### Logging in

Browse to `http://garagedoorcontroller.local/`, or to the device's IP address. The browser asks for a username and password:

- **Username:** the `web_username` value in `secrets.yaml`
- **Password:** the `web_password` value in `secrets.yaml`

The credentials are compiled into the firmware. **They are not stored in this README**, so keep them in `secrets.yaml` or a password manager.

### Configuration

```yaml
web_server:
  port: 80
  ota: false            # no firmware upload from the web UI
  auth:
    type: digest        # password is never sent in cleartext
    username: !secret web_username
    password: !secret web_password
```

### Changing the username or password

1. Edit `web_username` and `web_password` in `secrets.yaml`.
2. Rebuild and flash over OTA: `esphome run garagedoorcontroller.yaml`.
3. Close every browser window that had the old login open. Browsers cache digest credentials until they're closed, and there's no logout button.

### What the login protects

| Access path | Protection |
|---|---|
| Web UI and its REST/event endpoints (port 80) | **Username and password** (digest auth) |
| Web UI firmware upload (`/update`) | **Disabled** (`ota: false`) |
| Home Assistant native API | **Encrypted** with `api_key1`. Doesn't use the web login |
| OTA updates (`esphome run`) | **Encrypted** with `api_key1` |
| Fallback hotspot / captive portal | Only the AP's Wi-Fi password (`wifi_local_password`) |

### Limitations

- **Plain HTTP:** digest auth protects the password, but the pages and commands themselves aren't encrypted. Someone sniffing the same network can see the pages, door states and commands, but they can't recover the password from that traffic.
- **No lockout or rate limiting:** the ESP doesn't block repeated failed logins. Use a strong, unique password.
- **Captive portal:** if the device can't reach Wi-Fi, it starts its fallback AP. Anyone who knows `wifi_local_password` can join that AP and use the captive portal to change the Wi-Fi settings or upload firmware. Make `wifi_local_password` strong too.
- **Best practice:** put the device on an isolated IoT VLAN, and only allow Home Assistant (and your admin host) to reach ports 80 and 6053.

---

## Home Assistant Entities

| Entity | Type | Description |
|---|---|---|
| Garage Door | `cover` (garage) | Open, close and stop, with state from the reed switches |
| Garage Door Open Sensor | `binary_sensor` (garage_door) | `on` (shown as "Open") when the door is fully open. Can be used in automations |
| Garage Door Closed Sensor | `binary_sensor` (no device class) | `on` when the door is fully closed, `off` as soon as it leaves the closed position. Can be used in automations. It has no `garage_door` class, because that class would display ON as "Open" |
| Garage Door Relay | `switch` | Raw opener pulse (500ms). Doesn't beep or check state |
| Garage Door Buzzer | `switch` | Raw buzzer pulse (300ms) |
| Garage Door Auto-Close | `switch` | Turns auto-close on or off. Resets to **on** at every boot |
| Garage Door Auto-Close Minutes | `number` (slider, config) | Auto-close timeout, 1–30 min. Saved across reboots |
| Garage Door Auto-Close Countdown | `sensor` (s) | Seconds remaining. Empty when idle |
| Garage Door State | `text_sensor` | `Open`, `Closed`, `Opening`, `Closing`, `Stopped`, or `Sensor fault`. Updates when the cover's state changes |
| Garage Door Countdown Display | `text_sensor` | `M:SS` remaining, or `--:--` when idle |
| Garage Door WiFi Signal | `sensor` (dBm) | Updates every 60s |
| Garage Door Uptime | `sensor` (s) | Device uptime |

> **Tip:** Use the **Garage Door** cover for normal operation. The raw **Relay** switch skips the warning beeps and the state checks.

### Using the reed sensors in automations

Both reed sensors are regular (non-diagnostic) entities, so they show up in the automation editor's entity pickers and on auto-generated dashboards. They report changes immediately, after a 100ms debounce.

| Sensor | `on` | `off` |
|---|---|---|
| Garage Door Open Sensor | Door fully open | Door not fully open |
| Garage Door Closed Sensor | Door fully closed | Door not fully closed (opening, stopped partway, or open) |

**Find the entity IDs:** in **Settings → Devices & services → ESPHome → GarageDoorController**, each sensor's entity ID is listed. They're usually `binary_sensor.garagedoorcontroller_garage_door_open_sensor` and `binary_sensor.garagedoorcontroller_garage_door_closed_sensor`, but check yours.

#### Example: turn on the garage lights when the door starts to open

```yaml
alias: Garage lights on when door opens
description: Turn on garage lights as soon as the door leaves the closed position
triggers:
  - trigger: state
    entity_id: binary_sensor.garagedoorcontroller_garage_door_closed_sensor
    from: "on"      # only a real closed -> open transition
    to: "off"
actions:
  - action: light.turn_on
    target:
      entity_id: light.garage_lights   # your light entity
mode: single
```

> **Why `from: "on"`:** when the ESP reboots or reconnects, the sensor goes `unavailable` → `off` (if the door is open). Without `from: "on"`, that reconnect would also turn the lights on.

#### Optional: turn the lights off a few minutes after the door closes

```yaml
alias: Garage lights off after door closes
triggers:
  - trigger: state
    entity_id: binary_sensor.garagedoorcontroller_garage_door_closed_sensor
    from: "off"
    to: "on"
    for:
      minutes: 5
actions:
  - action: light.turn_off
    target:
      entity_id: light.garage_lights
mode: single
```

The `for: minutes: 5` only fires if the door stays closed for the full 5 minutes. If it reopens, the timer resets.

---

## Behavior

### Manual open (cover → Open)

Runs unless the door already reads fully open: the open sensor is OFF, **or** the closed sensor is ON. Each request is logged with both sensor states.

1. 5 beeps, 600ms apart
2. Opener relay pulses

### Manual close (cover → Close)

Runs unless the door already reads fully closed: the closed sensor is OFF, **or** the open sensor is ON. Each request is logged with both sensor states.

1. Any pending auto-close timer is cancelled
2. The cover shows **Closing** right away
3. 7 beeps, a 1.5s pause, then 3 beeps
4. Opener relay pulses
5. If the door never leaves the open sensor, the watchdog resets the state after 30s

Pressing **Stop** or **Open** during the beeps cancels the close, and the state goes back to **Open**.

### Stop (cover → Stop)

Sets the operation to IDLE and cancels a close that's still in its warning beeps. The relay is pulsed **only if the door is between the switches**, because a pulse at fully open or fully closed would start the door moving.

### Door State text

**Garage Door State** shows the full door status, including movement. The ESP web UI shows the cover itself only as `OPEN` or `CLOSED`, so this sensor is the one to watch there.

| Event | State |
|---|---|
| Close pressed (or the auto-close warning starts) | `Closing` |
| Closed sensor releases | `Opening` |
| Open sensor releases | `Closing` |
| Open sensor triggers | `Open` |
| Closed sensor triggers | `Closed` |
| Stop pressed, or watchdog fires, between sensors | `Stopped` |
| Both sensors read ON | `Sensor fault` |

In Home Assistant, the **Garage Door** cover entity already shows `Opening` and `Closing`, because those come from the cover's `current_operation`.

### Travel watchdog

If no reed switch changes within **30s** of a sensor releasing or a relay pulse, so the door stopped partway or never moved, (because of a wall-button stop or a safety reverse, say), the operation resets to IDLE and a warning is logged. Without this, HA would keep showing Opening/Closing and grey out that button.

### Auto-close

```
Door reaches fully open ──► auto-close enabled? ──no──► nothing
                                   │ yes
                                   ▼
                     countdown = slider minutes × 60
                                   │
                         wait slider minutes
                                   │
             still open AND still enabled? ──no──► clear countdown
                                   │ yes
                                   ▼
      state → Closing; 7 beeps ─ 1.5s pause ─ 3 beeps
                                   │
             still open AND still enabled? ──no──► state → Open, clear countdown
                                   │ yes
                                   ▼
                        pulse opener relay
```

**What cancels the timer:**

- Turning **Garage Door Auto-Close** off. This takes effect immediately, even during the beeps.
- The door leaving the fully-open position (open sensor releases)
- A manual **Close** command from the cover

**What starts or restarts the timer:**

- The door reaching fully open, if auto-close is enabled
- Turning **Garage Door Auto-Close** on while the door is open
- Booting with the door open (auto-close is always on after boot)

> Changing the **Minutes** slider doesn't affect a countdown that's already running. The new value applies the next time the timer starts.

### Cover state logic

| Open sensor | Closed sensor | Reported state |
|---|---|---|
| ON | – | Open |
| OFF | ON | Closed |
| OFF | OFF | Unchanged (last published state and operation) |

Moving between the sensors publishes `OPENING` (closed sensor released) or `CLOSING` (open sensor released) as the current operation.

Both reed switches have a **100ms debounce** (`delayed_on_off`), so a bouncing contact can't fire repeated triggers.

---

## Build & Flash

```bash
# Validate config
esphome config garagedoorcontroller.yaml

# First flash over USB
esphome run garagedoorcontroller.yaml --device /dev/ttyUSB0

# Later updates over OTA
esphome run garagedoorcontroller.yaml --device garagedoorcontroller.local

# Logs
esphome logs garagedoorcontroller.yaml
```

The device also serves a local web UI at `http://garagedoorcontroller.local/`. It asks for the `web_username` and `web_password` from `secrets.yaml`.

---

## Test Checklist

1. **Open:** cover → Open. You should hear 5 beeps, then the relay pulses and the countdown starts once the open sensor triggers.
2. **Disable mid-countdown:** turn Auto-Close off. The switch should **stay off** and the countdown should show `--:--`.
3. **Disabled stays disabled:** close and reopen the door. No timer should start.
4. **Re-enable:** turn Auto-Close on with the door open. The timer should restart.
5. **Abort during beeps:** set Minutes to 1 and let the timer run out. Turn Auto-Close off during the beeps. The relay should **not** fire.
6. **Full auto-close:** with Minutes at 1, let it run all the way. You should hear the beeps, then the door closes and the countdown clears.
7. **Manual close:** cover → Close while a countdown is running. The countdown should clear right away, then the close beeps play and the relay pulses.
8. **Reboot behavior:** change the Minutes slider and turn Auto-Close **off**, then power-cycle the device. Minutes should keep its value and Auto-Close should come back **on**. If the door is open, the countdown should start on boot.
9. **Web login:** open `http://garagedoorcontroller.local/` in a private browser window. It should ask for a login. A wrong password should be rejected, and `web_username` / `web_password` should get you in.
10. **Web OTA disabled:** browse to `http://garagedoorcontroller.local/update`. It shouldn't offer a firmware upload.

---

## Known Issues / Notes

- **GPIO5 is an ESP32 strapping pin.** A pullup is its normal boot state, but if the door is closed (pin pulled low) while the ESP32 boots, startup can act strangely. If you see boot problems, move the closed sensor to GPIO18, 19, 21 or 22.
- **Web UI security:** the web UI requires a login, but it runs over plain HTTP. See [Web Interface & Security](#web-interface--security).
- **OTA encryption:** OTA uploads are encrypted with the API key (`ota: encryption: {}` inherits `api_key1`). There's no OTA password anymore. If an OTA upload ever fails with *"device did not offer encryption"*, the running firmware is too old to encrypt. Flash once over USB to fix it.
- **Flash wear:** the Minutes slider is saved to flash. That's normal for occasional changes, but avoid automating them on a tight loop.
- **Single trigger relay:** the opener relay works like the wall button. If an obstruction or a missed pulse puts the opener out of step, the reed switches will still report the real door state.

---

## Changelog

### Rev 13

- **Changed:** removed `entity_category: diagnostic` from both reed sensors, so they're first-class entities in Home Assistant (they show in entity pickers and default dashboards)
- **Docs:** added "Using the reed sensors in automations", with example automations that turn the garage lights on when the door opens and off after it closes

### Rev 12

- **Docs:** added a **Web Interface & Security** section covering the login, how to change the credentials, what each access path is protected by, and the limitations. Also added a table explaining each secret, and web-login tests to the checklist

### Rev 11

- **Docs:** added the board's details (ESP32 Relay x2): specs, pin functions, relay wiring, power and first-flash steps

### Rev 10

- **Added:** web UI login (`web_server: auth:` with `type: digest`), using the `web_username` and `web_password` secrets

### Rev 9

- **Fixed:** Auto-Close came up **OFF** after every boot. Template switches default to `restore_mode: ALWAYS_OFF`, which runs `turn_off_action` during setup and overwrote the global with `false`, so neither `initial_value` nor `restore_value` ever took effect. The switch is now `restore_mode: DISABLED`, which leaves the boot state to the `auto_close_enabled` global (on). The Rev 7 change only works with this fix

### Rev 8

- **Security:** set `web_server: ota: false`. The web UI's `/update` endpoint accepted unauthenticated, plaintext firmware uploads from anyone on the LAN, which bypassed OTA encryption entirely
- **Changed:** replaced the OTA `password:` with `encryption: {}`, which inherits the API encryption key. This clears the "password wastes significant flash and RAM" warning and encrypts OTA uploads. `ota_pass1` is no longer needed in `secrets.yaml`

### Rev 7

- **Changed:** auto-close is no longer saved (`restore_value: false`), so it always comes back **on** after a reboot or power loss. The door is never left open without a timer. If the door is open at boot, the countdown starts automatically

### Rev 6

- **Added:** the cover shows **Closing** as soon as Close is pressed, and when the auto-close warning starts, not only once the door leaves the open sensor
- **Added:** "Closing" is cleared if the close is cancelled: by Stop, by Open, by turning off auto-close during the warning, by the last-moment abort check, or if the door never moves (watchdog)
- **Changed:** the watchdog now resets any non-IDLE operation after 30s with no reed switch change, including a door that never moved
- **Fixed:** Stop pulsed the relay even with the door at fully open or closed, which would **start** the door. It now pulses only when the door is between the switches

### Rev 5

- **Added:** **Garage Door State** text sensor (Open, Closed, Opening, Closing, Stopped, Sensor fault), updated by the cover's `on_opening`, `on_closing`, `on_idle`, `on_opened` and `on_closed` triggers
- **Added:** "Cover is Opening" and "Cover is Closing" log lines

### Rev 4

- **Fixed:** the closed sensor never changed state because its GPIO5 jumper had come off the pin. With it reconnected, the closed switch behaves like the open switch (contact closes with the magnet), so it's back to `inverted: true`. A broken wire now reads OFF instead of a false "closed"
- **Removed:** the Rev 2 "NC switch" note, which came from the disconnected wire

### Rev 3

- **Fixed:** a faulty closed sensor stuck ON blocked **Close** with no log message. Close now runs if the closed sensor is OFF **or** the open sensor is ON. Open works the same way the other direction
- **Added:** log lines for every reed switch change, every Open/Close/Stop request (with both sensor states), and any ignored command
- **Added:** the cover lambda detects both sensors reading ON and logs a warning once, without reporting a state
- **Added:** `travel_watchdog` script. If the door stops between the sensors for 30s, it resets the operation to IDLE so HA doesn't grey out the buttons
- **Changed:** Stop now publishes IDLE and cancels the watchdog

### Rev 2

- **Fixed:** the Closed Sensor had `device_class: garage_door`, so HA showed it as **"Open"** while the door was closed. The device class is removed, so it now shows plain ON/OFF
- **Changed:** both reed sensors moved to the diagnostic category (the **Garage Door** cover is the main entity). Reverted in Rev 13
- **Added:** a boot log line showing both reed switch states and the cover position
- **Fixed:** replaced the deprecated `on_open` on the cover with `on_opened`
- ~~Fixed: the closed-position reed switch is NC (`inverted: false`)~~ This was wrong. See Rev 4

### Rev 1

- **Fixed:** turning the Auto-Close switch off set the flag back to `true` (it used `$auto_close_enabled_value`), so auto-close couldn't be disabled
- **Fixed:** the auto-close warning and relay pulse ran inside the cover's `close_action`, where `script.stop` couldn't cancel them. They now run inside the countdown script
- **Fixed:** an invalid `delay` with `seconds: -1`
- **Fixed:** the cover lambda returned `COVER_OPERATION_OPENING` (which equals `1.0`, the same as `COVER_OPEN`) when mid-travel. It now returns `{}`
- **Fixed:** buzzer pulses collided and dropped beeps. The pulse is now 300ms and the beeps are spaced 600ms apart
- **Added:** "Auto-Close Minutes" slider (1–30 min, saved across reboots)
- **Added:** 100ms reed switch debounce
- **Added:** `restore_mode: ALWAYS_OFF` on both relays
- **Added:** manual close cancels any pending auto-close
- **Changed:** `sprintf` → `snprintf` with a larger buffer
- **Changed:** the repeated buzzer actions are now `repeat:` blocks
