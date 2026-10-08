# HomeMate IR Blaster → ESPHome

ESPHome firmware for the HomeMate Wi-Fi IR blaster (Tuya, **T1-3S module / Beken BK7238**), for local Home Assistant control of a Panasonic AC.

## Hardware

| Item | Value |
|---|---|
| Module | T1-3S (sticker) |
| Chip | Beken **BK7238**, 2 MiB flash |
| ESPHome board | `bk72xx` → `board: t1-3s` |

### Pinout (traced with a multimeter on this unit)

| Function | Pin | Notes |
|---|---|---|
| IR transmit | **P6** | 1 kΩ to the base of all 4 S8050 (`J3Y`) LED drivers |
| IR receive | **P26** | Receiver OUT, active-low; receiver powered from 3.3 V via 100 Ω |
| Status LED | **P24** | Polarity unconfirmed; set `inverted: true` if it looks backwards |
| Button | **P9** | Active-low (pulls to GND via 1 kΩ) |
| UART (flashing) | RX = P10, TX = P11 | Header pins **RX** / **TX** (module-side labels) |

Programming header: `RX, TX, Trs, GND, 3.3V`. `Trs` is not needed and was left unconnected.

> Other HomeMate/Tuya IR blasters use different modules and pins. Verify yours before reusing this pinout.

## What you need

- A 3.3 V USB-UART adapter. An ESP32 dev board with a CP2102 works if **EN is jumpered to GND** (this holds the ESP32 in reset).
- A **USB-A → USB-C** cable to power the blaster. With USB-C → USB-C, the blaster gets no power.
- [uv](https://docs.astral.sh/uv/), then:

```sh
uv tool install --python 3.12 ltchiptool   # flash backup / first flash
uv tool install esphome                     # build + OTA
```

Tested with ltchiptool 4.14.4, ESPHome 2026.9.1 and LibreTiny 1.13.0.

## Wiring (first flash only)

| Adapter (ESP32 dev board) | Blaster header |
|---|---|
| GND | GND |
| RX0 / RXD (GPIO3) | RX |
| TX0 / TXD (GPIO1) | TX |
| 3V3 / 5V | **not connected** |

- On an ESP32 dev board, use the TX0/RX0 pins, **not TX2/RX2**.
- If you use a standalone USB-UART adapter, cross the lines instead: adapter TX → RX, adapter RX → TX.
- Power the blaster from its own USB. To enter the bootloader, **unplug and replug the blaster** while ltchiptool is waiting.

## 1. Back up the stock firmware

Do this first, and do it twice.

```sh
mkdir -p backup && cd backup
ltchiptool flash read bk7238 -d /dev/cu.usbserial-0001 -b 115200 -t 300 stock_1.bin   # power-cycle the blaster
ltchiptool flash read bk7238 -d /dev/cu.usbserial-0001 -b 115200 -t 300 stock_2.bin   # power-cycle again
cmp stock_1.bin stock_2.bin && echo IDENTICAL    # both files should be 2097152 bytes
```

If a read stops partway, re-run it. Keep the boards and cables still while it runs.

## 2. Configure and build

```sh
cp secrets.example.yaml secrets.yaml   # add your Wi-Fi details (2.4 GHz only)
esphome compile homemate-ir.yaml
```

## 3. First flash (serial)

```sh
ltchiptool flash write -d /dev/cu.usbserial-0001 -b 115200 -t 300 \
  .esphome/build/homemate-ir/.pioenvs/homemate-ir/firmware.uf2
```

Power-cycle the blaster once when prompted. Then disconnect the wires and power-cycle it again.

If Wi-Fi fails, the blaster opens a **`HomeMate-IR Fallback`** hotspot. Connect to it and enter your Wi-Fi details at `http://192.168.4.1`.

## 4. Updates (over Wi-Fi)

```sh
esphome run homemate-ir.yaml --device homemate-ir.local
```

## Home Assistant

The blaster is auto-discovered under **Settings → Devices & services → ESPHome**. The API is unencrypted, which is intended for a trusted LAN.

It provides these entities: **Panasonic AC** (climate), **Button**, **WiFi Signal**, **Uptime**, **IP Address** and **Restart**.

The AC is controlled with `heatpumpir`, `protocol: panasonic_dke`. For a different AC brand, swap the `climate:` block for a suitable [IR climate platform](https://esphome.io/components/climate/climate_ir.html).

## Status

- ✅ Backup, serial flash, Wi-Fi, API and OTA are working.
- ⏳ IR receive, IR transmit to the AC, and the button are not yet tested on this unit.

## Restore stock firmware

```sh
ltchiptool flash write -d /dev/cu.usbserial-0001 -b 115200 -t 300 -f bk7238 -s 0 backup/stock_1.bin
```
