# Scene Switch

A 6-button ESP32 scene controller with Kailh Choc switches and per-button WS2812B LED feedback, built for Home Assistant.

## Features

- 6 physical buttons, each with single/double/long-click detection
- Per-button addressable LED with color feedback (partitioned from a single LED strip)
- Soft white "click" pulse on every press for tactile feedback, independent of whatever effect/color is currently running
- Custom LED effects: Rainbow Flow, Meteor Flow, Knight Rider, Police Strobe, Color Wipe, Button Colors
- Boot LED sweep test on startup
- WiFi with primary + fallback SSID, plus local AP fallback if both fail
- Remembers and restores the last color/effect after a WiFi disconnect (shows an amber "breathing" indicator while disconnected)
- OTA updates with safe-mode rollback protection

## Setup

1. Copy `secrets_example.yaml` to `secrets.yaml` and fill in your own WiFi credentials, OTA password, and AP fallback password.
2. Generate a 32-byte base64 API encryption key using one of these terminal commands and add it to your `secrets.yaml`:
   - **Python**: `python -c "import secrets; import base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"`
   - **PowerShell**: `$bytes = New-Object Byte[] 32; [Security.Cryptography.RNGCryptoServiceProvider]::Create().GetBytes($bytes); [Convert]::ToBase64String($bytes)`
3. Adjust GPIO pin assignments in `sceneswitch.yaml` if your wiring differs.
4. Flash via USB for the first install:
   ```bash
   esphome run sceneswitch.yaml
   ```
5. Subsequent updates can be done over OTA once the device is on your network.

## Hardware

- ESP32 (any variant with enough GPIOs for 6 buttons + 1 LED data pin)
- 6x Kailh Choc V1 switches
- 6x WS2812B addressable LEDs (one per button)
- Custom 3D-printed enclosure (FDM) — CAD files in [`cad/`](./cad).

## CAD Files

STEP files for the enclosure are in `cad/`:
- `Cover.step` — back cover
- `Faceplate.step` — front faceplate

This is v1 — fit and finish aren't perfect yet, see Known Issues below.

## Known Issues / v2 TODO

- Enclosure fit tolerances are a bit tight/loose in places, needs dialing in.
- Back cover was originally too close to the ESP32 (1-2mm gap), caused thermal buildup — fixed in software by switching WiFi `power_save_mode` to `light`, but v2 needs proper ventilation holes in the CAD design.
- **Stop Hand-Soldering**: WS2812B LEDs had to be cut apart individually and hand-soldered with very fine wire. DIN of LED1 ended up wired to what's physically Button 6, requiring software mapping fixes. **v2 MUST use a custom printed PCB (e.g., from JLCPCB).** It's significantly cheaper in time/effort, makes the build 10x more rigid, and ensures correct LED data routing without software hacks.

## Firmware Gotchas / Lessons Learned

- **Custom LED effects going completely dark**: effects that called `current_values_as_rgb()` on the light from inside their own effect update (Meteor Flow, Knight Rider, Scanner, Solder Chase) would render nothing at all. Fixed by reading the last-known color from stored `prev_red`/`prev_green`/`prev_blue` globals instead of querying the light's own live state mid-effect.
- **LED flash scripts doing nothing visible**: using a raw C++ `delay()` inside a single lambda (between two `schedule_show()` calls) never actually pushes pixel data to the strip — `schedule_show()` just flags "dirty," and the real hardware write only happens when control returns to ESPHome's scheduler. A blocking `delay()` never yields that control back, so the flash gets computed and immediately overwritten before it's ever shown. Fixed by using real ESPHome `delay:` script steps (which do yield) instead of blocking C++ calls.
- **WiFi-reconnect state getting stuck on the disconnect indicator (amber breathing)**: two separate causes — (1) the "previous state" globals were saved with `restore_value: true`, so a corrupted value from early testing got permanently written to flash and kept surviving reflashes; and (2) the light's generic `on_state` handler saved "previous state" on *every* state change, including the disconnect indicator's own color changes, overwriting the real saved state moments after it was captured. Fixed by setting `restore_value: false` on those globals and adding a guard so state is never saved while the disconnect indicator itself is active.
- **WiFi fallback SSID never triggering**: pinning a `bssid:` on the primary network entry prevents fallback to a second SSID entirely if that specific access point becomes unreachable, since the device won't roam off a pinned BSSID. Removed the pin so both SSIDs can be tried freely, with `priority:` used to prefer one over the other.

## Notes

- `power_save_mode: light` is used to reduce WiFi radio heat/idle temperature — set to `none` only if you need lower API latency and don't mind the extra heat.
- The LED-to-button physical mapping may need adjusting (`from:`/`to:` under the `partition` platform lights) depending on your specific wiring order.