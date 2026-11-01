# Fisher & Paykel fridge → Home Assistant (ESPHome, listen only)

A Seeed XIAO ESP32-C3 listens to the two serial links on the main board of a **Fisher & Paykel RF505BRNUX1**
(Series 5, 515 L, bottom freezer, ice & water) and reads two of its temperature sensors. It never transmits.

What Home Assistant gets:

| Entity | Source |
|---|---|
| Fridge door, freezer drawer | CN11 |
| Fridge setting, freezer setting (°C) | CN11 |
| Fast Freeze, Bottle Chill, ice maker, Ice Boost, door alarm muted, water filter due | CN11 |
| Compressor, fan, defrost, defrost stage | BUS |
| Water dispenser locked, water valve | BUS |
| Freezer temperature, evaporator temperature (°C, the fridge's own sensors) | BUS |
| Fridge temperature (°C), plus the R-S and F-S sensor voltages | thermistor taps |

The main board is a Haier platform board (F&P is Haier-owned), so other F&P and Haier fridges built on the same
board may speak the same protocol. Check yours with a logic analyser before trusting the byte map.

## Safety

- The main board has **mains on it**. Unplug the fridge at the wall before you take the controller cover off.
- Only the low-voltage connectors are touched: CN11, the BUS line and two thermistor pins.
- Power the ESP32 from an **isolated** 5 V supply, such as a USB phone charger. Never from the fridge's own 5 V.
- Flash the ESP32 on the bench first. After it is wired to the fridge, update it over Wi-Fi only.

## Where things are

The main controller (`0061801364`) and the ice & water board (`0061801177`) sit on top of the cabinet under
the controller cover.

**Main board, CN11** (6 pins): `RG-S`, `+5V`, `D-TX`, `D-RX`, `GND`, `+12V`.
`D-TX` is the main board talking to the control panel inside the fridge. `D-RX` stays silent.

**BUS**: on the main board's top connector (`+12V`, `GND`, `+5V`, `BUS`, `JUMP`). It runs to `CN8` on the ice & water
board (`12V`, `GND`, `5V`, `BUS`, `W_SW`).

**Thermistors**: each one is paired with a `GND` pin on the main board's sensor connectors. `R-S` = fridge,
`F-S` = freezer. Also present: `FD-S` (freezer defrost/evaporator), `RT-S`, `RS-S` and `RH-S`.

## The two links

Both run at **9600 baud, 8N1**.

| Link | Polarity | Logic level | Divider into the ESP32 |
|---|---|---|---|
| CN11 D-TX | **inverted** (idles at 0 V) | 5 V | 100k top, 220k bottom → 3.4 V |
| BUS | normal | idles at **12 V** (1.8k pull-up to +12 V) | 100k top, 33k bottom → 3.0 V |

Every message has the same shape:

```text
A5  dst  len  src  cmd  payload ...  crcL crcH  5A
```

- `len` is the length of the whole frame, `A5` to `5A`.
- Addresses: `FF` = main board, `FC` = inside control panel (on CN11), `04` = ice & water board, `08` = door
  display and dispenser.
- The 23-, 28- and 34-byte frames carry a **CRC-16/MODBUS** (poly `0xA001` reflected, init `0xFFFF`) over bytes
  `1 … n-4` (`dst` to the end of the payload), stored low byte first. The firmware drops frames that fail it.
  Shorter frames use a check that isn't decoded, so nothing in Home Assistant comes from them.

## Byte map

Byte numbers count from 0 at the `A5`.

### CN11, main board → panel, 28 bytes (`A5 FC 1C FF 03 …`)

| Byte | Meaning |
|---|---|
| 6 | Fridge setting, °C |
| 7 | Freezer setting + 24 (`06` = −18 °C) |
| 9 | `0x01` Fast Freeze, `0x08` door alarm muted |
| 10 | `0x01` ice maker on, `0x02` Ice Boost, `0x08` Bottle Chill, `0x20` water filter due |
| 18 | `0x04` fridge door open, `0x08` freezer drawer open |

### BUS, main board → ice & water board, 23 bytes (`A5 04 17 FF 03 …`)

| Byte | Meaning |
|---|---|
| 7 | `0x01` fridge door / light, `0x08` fan, `0x10` compressor, `0x20` water dispenser locked |
| 8 | `0x02` Fast Freeze, `0x04` ice maker on |
| 9 | `0x01` Ice Boost |
| 13–14 | Freezer sensor, 16-bit (see below) |
| 19 | Defrost stage: `0` none, `1`–`2` before the heater, `4` heater, `5` drip, `6` fan delay, `8`, then `0` |

### BUS, ice & water board → main board, 34 bytes (`A5 FF 22 04 03 …`)

| Byte | Meaning |
|---|---|
| 6–7 | Evaporator (defrost) sensor, 16-bit (see below) |
| 8–9 | Freezer sensor again |
| 10–17 | Four constants: 7200, 1500, 8400, 4800 (seconds) |
| 18–23 | Three 16-bit counters in seconds; they reset when the ice maker is turned on and stop at 7201 |
| 27 | `0x01` ice maker on, `0x08` water valve open |

### BUS, main board → door display (`A5 08 … FF 03 …`)

| Byte | Meaning |
|---|---|
| 5 | `0x01` water filter due, `0x02` Bottle Chill, `0x04` fridge door open |

### Temperatures

The main board sends its sensor readings as **(°C + 40) × 10**, so `°C = value / 10 − 40`.

| Reading | Value | °C |
|---|---|---|
| Freezer, normal compressor cycle | 182–211 | −21.8 to −18.9 (setting −18) |
| Evaporator, compressor running | down to 44 | −35.6 |
| Evaporator, end of defrost | 475 | +7.5 |

The fridge compartment sensor (`R-S`) is not on either link, so it is read from the thermistor through a
1M/1M divider (see below) and converted with the usual NTC divider formula:

```text
°C = 1 / (1/298.15 − ln((5.236/V − 1) / k) / 3591) − 273.15        k = 2.275 on mine
```

`5.236` and `3591` come from fitting the F-S voltage against the main board's own freezer reading (correlation
0.998, 12 mV rms). `k` was fitted against a thermometer in the middle of the fridge overnight (0.2 °C rms on
30-minute averages). On mine, 2.80 V = 3.0 °C and 2.70 V = 4.6 °C. Check `k` on your own fridge.

## Wiring

```text
CN11 D-TX ──[100k]──┬── D10 (GPIO10)        BUS ──[100k]──┬── D5 (GPIO7)
                  [220k]                                [33k]
GND ────────────────┴── GND                 GND ──────────┴── GND

R-S ──[1M]──┬── D1 (GPIO3)                  F-S ──[1M]──┬── D2 (GPIO4)
          [1M] ‖ 100 nF                               [1M] ‖ 100 nF
GND ────────┴── GND                         GND ────────┴── GND

Fridge GND (CN11 pin 5) ── XIAO GND
Optional DS18B20 module:  S ── D3 (GPIO5),  V ── 3V3,  G ── GND
Power: 5 V from an isolated supply into the XIAO (USB-C, or its 5V pin)
```

- The 1M/1M dividers are high impedance on purpose, so the fridge's own reading of the thermistor doesn't shift.
  The 100 nF capacitor gives the ESP32's ADC a stiff source to sample from.
- The XIAO's D-labels are not its GPIO numbers. Wire by the D-labels above.
- My build is on a Coreva 101S-UCC perfboard with the fridge wires spliced and heatshrunk:
  [docs/board-layout.md](docs/board-layout.md).

## Firmware

[`fridge-bridge.yaml`](fridge-bridge.yaml) (ESPHome 2026.10). Put `wifi_ssid`, `wifi_password` and
`ap_password` in your `secrets.yaml`. The UARTs have only `rx_pin`s, so the ESP32 cannot send anything to the fridge.

## Licence

MIT
