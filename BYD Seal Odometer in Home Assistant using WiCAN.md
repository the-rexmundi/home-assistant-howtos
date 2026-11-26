# BYD Seal and Sealion 6 odometer in Home Assistant using MeatPi WiCAN OBD

This guide reads the odometer from a passive CAN broadcast using the **MeatPi WiCAN OBD MQTT CAN filter** and displays it in Home Assistant.

The odometer filter does **not** send a diagnostic request to the car. It can run alongside the WiCAN's AutoPID vehicle profile, which I use for battery state of charge.

## Tested vehicles

This method has been tested on:

- **2024 BYD Seal**, Australia
- **2024 BYD Sealion 6**, Australia
- Original **MeatPi WiCAN OBD**, firmware **4.21**

On both vehicles, the decoded reading matched the dashboard. The Seal reading was also confirmed to increment during a drive. Other model years, markets or vehicle software versions may differ, so compare the result with your own dashboard.

## What you need

- A MeatPi WiCAN OBD connected to the car
- WiCAN connected to your network in **AP+Station** mode
- An MQTT broker accessible to both WiCAN and Home Assistant
- The MQTT integration configured in Home Assistant
- Optionally, MQTT Explorer to inspect messages
- Your car's dashboard odometer reading for comparison

I use WiCAN's **AutoPID** mode with the vehicle-specific profile **BYD: Seal**, which already provides SoC. The odometer uses the separate MQTT CAN filter described below.

> **Before you start:** Note your existing WiCAN settings. On some vehicles, active AutoPID polling while the car is locked can trigger the alarm. The odometer filter itself listens to a broadcast and does not send a request.

## 1. Find the odometer frame

With the car awake, open the WiCAN web interface and go to **Monitor**. Look for CAN ID **`3D9h`**, which is **985 in decimal**.

A sample from the Sealion 6:

```text
ID    Data
3D9h  05 EC 7C 03 70 00 47 6E
```

The odometer is in **bytes 1–3**, counting the first byte as byte 0:

| Byte | Sample | Meaning |
|---|---|---|
| 0 | `05` | Observed as `05` so far; purpose unknown |
| 1 | `EC` | Odometer low byte |
| 2 | `7C` | Odometer middle byte |
| 3 | `03` | Odometer high byte |
| 4–6 | `70 00 47` | Not used for this calculation |
| 7 | `6E` | Changes between frames; purpose unknown |

The three odometer bytes are **little-endian** and represent tenths of a kilometre. In this example, `EC 7C 03` decodes to **22,858.8 km**.

If you don't see `3D9h`, make sure the car is awake and the monitor is running at the correct bus speed; the tested setup used **500K**.

## 2. Configure MQTT on WiCAN

In the WiCAN web interface:

1. Under **Settings → CAN**, enable **MQTT**.
2. In the MQTT card, enter your broker address, port and credentials.
3. Make sure the **RX topic** is enabled.
4. Check that WiCAN is connected to your network in **AP+Station** mode.

The WiCAN documentation says Bluetooth LE and MQTT cannot run together, so disable BLE if necessary.

See the [MeatPi MQTT documentation](https://docs.meatpi.com/wican-obd/send-data/mqtt/) for the current settings and limitations.

## 3. Add one MQTT CAN filter

In the **MQTT CAN filter** table, add this single row:

| Field | Value |
|---|---|
| CAN ID (dec) | `985` |
| Name | `ODO_RAW` |
| PID | `-1` |
| Index | `2` |
| Start Bit | `8` |
| Bit Length | `24` |
| Expression | `V` |
| Cycle ms | `10000` |

Then click **Add ID**, **Store**, and **Submit Changes**. WiCAN restarts to apply the filter.

> **Note:** Once a CAN filter exists, WiCAN publishes filter values instead of unfiltered raw frames on the RX topic. This may affect any other consumers that rely on raw frames. AutoPID remained in use alongside the filter in the tested setup.

## 4. Find your MQTT topic

Open MQTT Explorer and look for the WiCAN topic tree. The default RX topic has this form:

```text
wican/<YOUR_WICAN_ID>/can/rx
```

For example, if your device ID were `example123`, the topic would be:

```text
wican/example123/can/rx
```

Find your actual device ID in the WiCAN web interface or its Home Assistant device information. You can also find the exact topic directly in MQTT Explorer under:

```text
wican → <YOUR_WICAN_ID> → can → rx
```

**Use the topic shown by your own device**, because topics can be renamed.

You should see messages such as:

```json
{"ODO_RAW": 15498243.000000}
```

For the sample `EC 7C 03`, `15498243` is the three-byte field as WiCAN extracted it. Home Assistant will rearrange the bytes to obtain the odometer.

## 5. Add the raw MQTT sensor to Home Assistant

Edit `/config/configuration.yaml` and add the following. Replace `<YOUR_WICAN_ID>` with your actual device ID, or replace the entire `state_topic` with the topic you found in MQTT Explorer.

```yaml
mqtt:
  sensor:
    - name: "BYD Odometer Raw"
      unique_id: byd_odometer_raw
      state_topic: "wican/<YOUR_WICAN_ID>/can/rx"
      value_template: >-
        {% if value_json.ODO_RAW is defined %}
          {{ value_json.ODO_RAW }}
        {% else %}
          {{ this.state }}
        {% endif %}
```

If `configuration.yaml` already contains an `mqtt:` section, add the sensor to its existing `sensor:` list rather than creating a second `mqtt:` key. Alternatively, put the configuration in a Home Assistant package if you use packages.

The template retains the sensor's previous value when another message on the same topic does not contain `ODO_RAW`.

Check the configuration in **Developer Tools → YAML**, then reload MQTT entities if available or restart Home Assistant.

In **Developer Tools → States**, check the entity ID Home Assistant created. The template below assumes:

```text
sensor.byd_odometer_raw
```

If HA assigned a different ID, use that ID in the template.

## 6. Create the odometer template sensor

Create a template sensor in the Home Assistant UI:

**Settings → Devices & services → Helpers → Create helper → Template → Template a sensor**

Paste this into the **State** field:

```jinja
{% set raw = states('sensor.byd_odometer_raw') | float(none) %}
{% if raw is none %}
  {{ none }}
{% else %}
  {% set v = raw | int %}
  {% set b1 = (v // 65536) % 256 %}
  {% set b2 = (v // 256) % 256 %}
  {% set b3 = v % 256 %}
  {{ ((b3 * 65536 + b2 * 256 + b1) / 10) | round(1) }}
{% endif %}
```

Set the remaining fields to:

- **Name:** `Odometer` or `BYD Odometer`
- **Device class:** `Distance`
- **Unit of measurement:** `km`
- **Device:** select your WiCAN/car device

Leave the **state class** unset while testing. Once you have confirmed the value is stable and increases correctly, you can select **Total increasing**.

The preview should show a value close to the dashboard odometer. Submit the helper to create it.

## 7. Verify the result

1. Compare the Home Assistant reading with the dashboard while parked.
2. Drive a known distance and check that the sensor increases accordingly.
3. Check it again after the car sleeps and wakes.

The tested car's app/dashboard appeared to **truncate** the decimal part rather than round it. For example, a reading of `22,858.8 km` was displayed as `22,858 km`.

If you prefer whole kilometres in Home Assistant, replace the final calculation in the template with:

```jinja
{{ ((b3 * 65536 + b2 * 256 + b1) / 10) | int }}
```

The decimal version retains the extra resolution.

## Troubleshooting

| Problem | What to check |
|---|---|
| No `3D9h` frame | Wake the car, check the monitor and bus speed |
| No MQTT messages | Check broker connection, RX topic, filter settings, Store and Submit |
| `ODO_RAW` remains unknown | Check the exact MQTT topic and that the message contains `ODO_RAW` |
| Odometer template remains unknown | Check the raw sensor's actual entity ID |
| Reading is wrong | Check Start Bit `8`, Bit Length `24`, expression `V`, and the template byte order |
| Reading doesn't update | Check the car is broadcasting and that the filter cycle was accepted |
| SoC or other data stops updating | Check AutoPID and existing MQTT consumers after adding the filter |

### Stale readings

The sensor holds its last value when the car is asleep or out of Wi-Fi range. A displayed number therefore doesn't necessarily mean the car is currently reporting. Check WiCAN connectivity or a separate last-update indicator if freshness matters.

## What didn't work

Before using the passive CAN frame, I tried the suggested diagnostic odometer requests, including:

- `22DD01` on `7E4`
- `220026` on `743`
- `222143` and `22210A` on `7E0`

None produced the correct odometer on the tested cars. Car Scanner profiles also did not show the Seal odometer. The `3D9h` broadcast worked without needing an odometer DID.
