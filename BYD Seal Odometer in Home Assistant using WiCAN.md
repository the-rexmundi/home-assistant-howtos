# BYD Seal odometer in Home Assistant via MeatPi WiCAN OBD (passive CAN filter)

This guide reads the odometer from a broadcast CAN frame on a BYD Seal, using the **MeatPi WiCAN OBD** and its **MQTT CAN filter**, and shows it in Home Assistant. It doesn't send any requests to the car, so it runs alongside AutoPID polling (SoC, charge times).

**Status: working and confirmed.** The reading matched the dashboard when parked, and it incremented correctly on a drive.

**Tested on:**
- BYD Seal, 2024 model, Australia
- MeatPi WiCAN OBD, firmware **4.21**
- Home Assistant with an MQTT broker (Mosquitto), checked with MQTT Explorer

Other Seal years, markets, or software versions may use a different frame ID or byte layout. Check the Monitor tab on yours before you copy the settings.

---

## What you need

- A MeatPi WiCAN OBD, connected to the car's OBD port
- WiCAN on your network (AP+Station mode) with MQTT enabled
- An MQTT broker that Home Assistant also uses
- MQTT Explorer (or `mosquitto_sub`) to check the topics
- Your dashboard odometer reading, to compare against

For SoC, I use the built-in vehicle-specific profile. In the WiCAN web UI, set **Protocol** to **AutoPID**, then on the **Automate** tab enable **Vehicle Specific** and choose **BYD: Seal**. That profile already includes SoC, so you don't need to add it yourself.

---

## Step 1: Find the odometer frame

In the WiCAN web UI, open the **Monitor** tab with the car awake. Look for frame **`3D9h`** (decimal **985**). A sample looked like this:

```
ID    Data
3D9h  05 1F D9 03 F8 FF 47 2E
```

The odometer is in **bytes 1 to 3**, as a 24-bit little-endian value in tenths of a km:

| Byte | Example | Role |
|---|---|---|
| 0 | `05` | Status/sequence (not used) |
| 1 | `1F` | Odometer, low byte |
| 2 | `D9` | Odometer, middle byte |
| 3 | `03` | Odometer, high byte |
| 4–6 | `F8 FF 47` | Not used here |
| 7 | `2E` | Rolling counter or checksum (not used) |

Decoding: `0x03D91F` = 252,191, so 252,191 / 10 = **25,219.1 km**.

> If you don't see `3D9h`, check the car is awake (ignition on, or charging), and that the Monitor is running at 500 kbit/s.

---

## Step 2: Add three rows to the MQTT CAN filter

Under the **MQTT** card on the WiCAN, enable MQTT and fill in the broker address and port. Then add three rows to the **CAN filter** table, one per byte:

| Name | CAN ID (dec) | PID | Index | Start Bit | Bit Length | Expression | Cycle ms |
|---|---|---|---|---|---|---|---|
| `ODO_B1` | 985 | -1 | 2 | 8 | 8 | `V` | 1000 |
| `ODO_B2` | 985 | -1 | 2 | 16 | 8 | `V` | 1000 |
| `ODO_B3` | 985 | -1 | 2 | 24 | 8 | `V` | 1000 |

- **CAN ID 985** is `0x3D9` in decimal.
- **PID -1** marks it as a non-OBD frame. The Index value is ignored.
- **Start Bit** counts from the first data byte, so byte 1 starts at bit 8, byte 2 at 16, byte 3 at 24.

Click **Add ID** for each row, then **Store**, then **Submit Changes**. WiCAN restarts.

This runs alongside AutoPID, so SoC and charge times keep working.

---

## Step 3: Check the topic in MQTT Explorer

After the restart, each rule publishes to the RX topic:

```
wican/dc067519eb01/can/rx
```

The payload is one key per message:

```json
{"ODO_B1": 31}
{"ODO_B2": 217}
{"ODO_B3": 3}
```

Replace `dc067519eb01` with your own device ID, which appears in the topic. The device ID matched the serial number shown in the WiCAN device info.

---

## Step 4: MQTT sensors in Home Assistant

Each message carries one key, so each sensor has to keep its last value when its key is missing. Otherwise the other two sensors go to *unknown* on every message, and the odometer never calculates.

Add this to `configuration.yaml`, or to a package file if you use packages:

```yaml
mqtt:
  sensor:
    - name: "ODO_B1"
      state_topic: "wican/dc067519eb01/can/rx"
      value_template: >-
        {% if value_json.ODO_B1 is defined %}{{ value_json.ODO_B1 }}{% else %}{{ states('sensor.odo_b1') }}{% endif %}
    - name: "ODO_B2"
      state_topic: "wican/dc067519eb01/can/rx"
      value_template: >-
        {% if value_json.ODO_B2 is defined %}{{ value_json.ODO_B2 }}{% else %}{{ states('sensor.odo_b2') }}{% endif %}
    - name: "ODO_B3"
      state_topic: "wican/dc067519eb01/can/rx"
      value_template: >-
        {% if value_json.ODO_B3 is defined %}{{ value_json.ODO_B3 }}{% else %}{{ states('sensor.odo_b3') }}{% endif %}
```

Check the config under **Developer Tools → YAML**, then restart Home Assistant. Check the entity IDs under **Developer Tools → States**. If the names are different, update the `states()` calls to match.

---

## Step 5: Combine the bytes into the odometer

You can create this as a **template sensor** in the UI, which lets you link it to the Seal device. Go to **Settings → Devices & services → Helpers → Create helper → Template → Template a sensor**, then enter:

**State template**

```jinja
{% set b1 = states('sensor.odo_b1') | float(none) %}
{% set b2 = states('sensor.odo_b2') | float(none) %}
{% set b3 = states('sensor.odo_b3') | float(none) %}
{% if b1 is none or b2 is none or b3 is none %}
  {{ none }}
{% else %}
  {{ (((b3 * 65536) + (b2 * 256) + b1) / 10) | round(1) }}
{% endif %}
```

**Settings**

- Device class: **Distance**
- State class: **Total increasing**
- Unit of measurement: **km**
- Device: your WiCAN device (the "Seal" device in my case)

The preview should show about **25,219.1 km**. Then submit, and rename the entity if you want.

If you'd rather use YAML, put this under `template:` in the same file:

```yaml
template:
  - sensor:
      - name: "Seal Odometer"
        unit_of_measurement: "km"
        device_class: distance
        state_class: total_increasing
        state: >-
          {% set b1 = states('sensor.odo_b1') | float(none) %}
          {% set b2 = states('sensor.odo_b2') | float(none) %}
          {% set b3 = states('sensor.odo_b3') | float(none) %}
          {% if b1 is none or b2 is none or b3 is none %}
            {{ none }}
          {% else %}
            {{ (((b3 * 65536) + (b2 * 256) + b1) / 10) | round(1) }}
          {% endif %}
```

The guard returns unknown instead of a wrong number when any byte is missing. A low number would break `total_increasing` statistics.

---

## Step 6: Verify

1. With the car parked, check the odometer sensor shows about **25,219.1 km**.
2. Check the three byte sensors update in the Activity log, and the odometer doesn't go to *unknown* between updates.
3. Drive and check the reading rises as you go. This was confirmed on a drive test.

### Truncated vs rounded values

The car's app and dashboard **truncate** to whole km, so 25,219.9 km shows as 25,219. The HA sensor rounds to one decimal place, so it shows 25,219.9. Both values come from the same reading. The difference is always less than 1 km.

If you want the HA value to match the car's display, you can change the template's final line to:

```jinja
{{ (((b3 * 65536) + (b2 * 256) + b1) / 10) | int }}
```

Keep the decimal version if you want to see tenths of a km while you drive.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Byte sensors show *unknown* | `value_json` key missing from the message | Use the sensor templates above, which keep the last value |
| Odometer shows *unknown* | One byte sensor hasn't received a value yet | Wait a few seconds after restart. Check the Activity log |
| Nothing in MQTT Explorer | RX checkbox off, car asleep, or the filter wasn't stored | Check the RX option, wake the car, and re-check **Store** / **Submit Changes** |
| Value is off by a multiple of 65,536 | Bytes read in the wrong order | Check the expression is `((B3*65536)+(B2*256)+B1)/10` |
| Value is 10× too large or small | Missing `/10`, or the expression is in the wrong units | Check the value is about 25,219 km, not 252,191 |
| Entity IDs don't match | HA created different names | Check the entity IDs in Developer Tools → States and update the `states()` calls |

---

## Caveats

- **One car tested.** This is a 2024 Seal in Australia on firmware 4.21 of the WiCAN OBD. Other years, markets, or software versions may use a different frame.
- **Byte 0 and byte 7 are not used.** They change between samples, so they're likely a status byte and a rolling counter.
- **Check after car updates.** BYD software updates could change the frame, so check the reading after an update.

---

## What didn't work

For anyone searching later, I tried the odometer combinations suggested in the forum posts and OBD PID lists, and **none of them returned the correct odometer** on my Seal:

- **`22DD01` on `7E4`** (the request from the BYD post). Read as 2-byte or 3-byte, big- or little-endian, with and without `/10` and the miles conversion. The closest result was 9,776, and the 3-byte value was 58,038,048. Neither matched 25,219.
- **`220026` on `743`** (OBDb `BYD_ODO`, 24-bit little-endian in tenths of a km). No usable response with the header tested.
- **`222143` and `22210A` on `7E0`** (OBDb instrument-panel and engine total mileage). No usable response.
- **Car Scanner.** No BYD Seal profile shows the odometer.

The passive broadcast frame was the only route that worked for me.

---

## Bonus: Charge times (AutoPID custom PID)

The vehicle-specific `BYD: Seal` profile covers SoC but not charge times, so I added charge times as a custom PID. It's the BYD charge counter, based on the BYD Atto 3 data in the OBDb project. It shows up as **ChargeTimes** on the WiCAN device, and on my Seal it reads 1,402.

1. In the WiCAN web UI, go to the **Automate** tab and scroll to **Custom PIDs**.
2. Click **New** and enter these values:

| Field | Value |
|---|---|
| Name | `ChargeTimes` |
| Init | `ATSH7E7;` (keep the trailing `;`) |
| PID | `22000B1` |
| Expression | `(B5*256)+B4` |
| Min Value | `0` |
| Max Value | `65535` |
| Period (ms) | `60000` |

3. Click **Store**, then **Submit Changes**.

The `1` at the end of the PID requests a single-frame reply. The expression reads the two bytes little-endian. Charge times only changes when you charge, so a 60-second period is plenty.
