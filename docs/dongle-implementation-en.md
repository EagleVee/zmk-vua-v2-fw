# Chutchit 2.4 GHz Dongle — Detailed Implementation Guide

---

## 1. Design Overview

The dongle is an nRF52840 board running the PH Design ZMK fork with
`CONFIG_ZMK_2G4_DONGLE=y`. It is a **pure USB HID relay**:

```
Chutchit Left Half                        Dongle PCA10059
┌──────────────────────────────┐          ┌─────────────────────────────┐
│                              │          │                             │
│  Scan matrix ──► Keymap eng. │  ESB     │  ESB PRX (radio receiver)  │
│  Generate HID report         │  2.4GHz  │  AES-128-CCM decrypt       │
│  AES encrypt ──► ESB PTX ────┼─────────►│  zmk_usb_hid_send_*()      │──► PC
│                              │          │                             │
└──────────────────────────────┘          └─────────────────────────────┘
        ▲ BLE split link
┌───────┴──────────────────────┐
│  Right half + Num half       │
└──────────────────────────────┘
```

The dongle does not contain a keymap. It only receives pre-built HID bytes
from the left half and forwards them over USB.

---

## 2. Bill of Materials (BOM)

### Option A — Use Nordic PCA10059 (fastest, no soldering)

| # | Part | Spec | Qty | Notes |
|---|---|---|---|---|
| 1 | **Nordic nRF52840 Dongle (PCA10059)** | USB-A male, nRF52840, 1 MB flash | 1 | Ships with USB bootloader, no JLink needed |

Buy at: Mouser (P/N 768-PCA10059), Digikey (1490-PCA10059-ND), or Nordic
webshop. ~$10 USD.

---

### Option B — Custom PCB (smaller, USB-A or USB-C direct plug)

Complete parts list for a self-designed dongle:

| # | Part | Spec | Qty | Footprint | Notes |
|---|---|---|---|---|---|
| 1 | **nRF52840-QIAA** | ARM Cortex-M4F, 1 MB flash, 256 KB RAM, 2.4 GHz, USB FS | 1 | QFN73 (7×7 mm) or QFN48 (6×6 mm) | Main SoC, mandatory |
| 2 | **32 MHz crystal** | 32 MHz, ±10 ppm, CL=8 pF | 1 | 2016 (2×1.6 mm) | Main clock for USB & radio |
| 3 | **32.768 kHz crystal** | 32.768 kHz, CL=7 pF | 1 | 3215 (3.2×1.5 mm) | RTC / sleep clock |
| 4 | **USB D+/D- filter caps** | 1 nF, 50 V, C0G/NP0 | 2 | 0402 | Common-mode EMI filter |
| 5 | **VDD bypass caps** | 100 nF, 10 V, X5R | 4 | 0402 | One near each VDD pin |
| 6 | **VDD bulk cap** | 10 µF, 10 V, X5R | 1 | 0805 | Energy reservoir |
| 7 | **VDDUSB caps** | 4.7 µF + 100 nF | 2 | 0402/0603 | USB 3.3 V rail filter |
| 8 | **D+ pull-up resistor** | 1.5 kΩ, 1% | 1 | 0402 | Only if not using chip's internal pull-up |
| 9 | **2.4 GHz chip antenna** | e.g. Johanson 0433AT62A0100E | 1 | 0402 / SMD | Or PCB trace antenna (free) |
| 10 | **Antenna matching caps** | Per antenna datasheet (typically 1–5 pF) | 2–3 | 0402 | Matching network |
| 11 | **DC-DC inductor** | 10 µH for internal nRF DC-DC | 1 | 0402 | Required if using DC-DC mode |
| 12 | **Reset button** | SPST, 50 mA, SMD | 1 | 3×4 mm | Enter DFU bootloader |
| 13 | **LED** | RGB LED or single | 1 | 0603 | Optional, status indicator |
| 14 | **LED resistors** | 33–100 Ω | 1–3 | 0402 | Current limiting |
| 15 | **USB-A male connector** | USB 2.0 Type-A, horizontal PCB mount | 1 | PCB horizontal | Plugs directly into PC |
| _(alt)_ | **USB-C male connector** | USB 2.0 Type-C, 24-pin plug | 1 | PCB mount | If USB-C preferred |
| 16 | **VDDIO decoupling** | 100 nF per rail | 2–4 | 0402 | GPIO power rails |

**Estimated unit cost (custom, small volume):** ~$3–5 USD.  
**Achievable PCB size:** ~18×8 mm (USB-A) or ~16×8 mm (USB-C).

---

## 3. Hardware Block Diagrams (Pseudo Schematic)

### 3.1 Power Supply

```
USB VBUS (5 V)
    │
    ├──[Bulk cap 10 µF]──GND
    │
    └──► VDDUSB pin (pin 6 of nRF52840)
              │
              └──► Internal USB LDO ──► 3.3 V internal rail
                                                │
                                   ┌────────────┤
                                   │            │
                              [100nF ×4]  [Inductor 10 µH]
                                   │            │
                                  GND      VDDMAIN (3.3V)
                                           (via internal DC-DC)
```

### 3.2 USB Interface

```
PC USB Host
    │
    ├── D+  ──[1 nF]──GND          ─────────── P0.13 (USB D+) ─┐
    │                                                            │ nRF52840
    └── D-  ──[1 nF]──GND          ─────────── P0.15 (USB D-) ─┘

    GND ──────────────────────────────────────── GND
    VBUS ─────────────────────────────────────── VDDUSB
```

> **PCB rule:** D+ and D- must be routed as a differential pair — equal length,
> parallel, same layer, avoid vias. The 1 nF caps are common-mode EMI filters.

### 3.3 32 MHz Crystal (Main clock)

```
                    ┌─────────────┐
nRF52840            │             │
  XC1 (P0.00) ──── │  XTAL 32MHz │
  XC2 (P0.01) ──── │             │
                    └─────────────┘
       │                               │
  [CL/2 ≈ 15 pF]               [CL/2 ≈ 15 pF]
       │                               │
      GND                             GND

  CL = crystal load capacitance (typically 8–12 pF per spec)
  Cap value = 2 × CL because two caps are in series to GND
```

### 3.4 32.768 kHz Crystal (RTC / Sleep)

```
                    ┌──────────────────┐
nRF52840            │                  │
  P0.00 ─────────── │  XTAL 32.768kHz  │
  P0.01 ─────────── │                  │
                    └──────────────────┘
       │                                    │
  [6–9 pF]                            [6–9 pF]
       │                                    │
      GND                                  GND
```

### 3.5 2.4 GHz Antenna

**Option 1: Chip antenna (simpler)**
```
nRF52840                      Chip Antenna
  ANT1 ──[Cmatch1]──┬──[Cmatch2]──► ANT
                    │
                [Lmatch]
                    │
                   GND

  Example: Johanson 0433AT62A0100E
  Typical matching: C=1.8 pF, L=1.5 nH, C=1.8 pF (per antenna datasheet)
```

**Option 2: PCB trace antenna (free)**
```
nRF52840
  ANT1 ──────────────── [~31.2 mm meander trace]

  No matching network required if you follow Nordic's reference layout
  from the nRF52840 Product Specification (PS v1.7 section 7).
  Works well on 2-layer FR4 with ground plane kept away from trace area.
```

### 3.6 Reset Button (for DFU Mode)

```
                    VDD (3.3 V)
                         │
                      [10 kΩ pull-up]
                         │
nRF52840 P0.18 ──────────┤
(RESET pin)              │
                      [Reset button]  ◄── push towards USB end of board
                         │
                        GND
```

> P0.18 is the dedicated reset/GPIO pin. Hold it to enter Nordic DFU bootloader.
> On PCA10059 this button is on the side edge of the PCB.

### 3.7 Status LED (optional — matches PCA10059 pinout)

```
VDD (3.3 V)
    │
    ├──[33 Ω]── P0.06   (Green LED  — LED0)
    │
    ├──[33 Ω]── P0.08   (Red LED    — LED1 Red,   fades during DFU)
    ├──[33 Ω]── P1.09   (Green LED  — LED1 Green)
    └──[33 Ω]── P0.12   (Blue LED   — LED1 Blue)

All LEDs are ACTIVE_LOW (pull pin to GND to turn on).
```

### 3.8 PCB Layout Overview (top view, not to scale)

```
                ← 18 mm →
    ┌──────────────────────────┐
    │  ┌──────────┐  🔴 LED   │  ↑
    │  │ nRF52840 │  [RESET]  │  8 mm
    │  │  QFN48   │  XTAL     │  ↓
    │  └──────────┘  [Ant]    │
    └────────────┐            │
    USB-A male   ████████████ │  (slides directly into PC port →)
    ──────────────────────────┘
             ↑
       USB-A plug extends
       beyond PCB edge
```

---

## 4. nRF52840 Pins Used by ZMK Dongle Firmware

| Pin | Function | Notes |
|---|---|---|
| P0.13 | USB D+ | Fixed by hardware, cannot change |
| P0.15 | USB D- | Fixed by hardware, cannot change |
| P0.18 | RESET | Fixed, used by DFU bootloader |
| P0.00 / P0.01 | XC1 / XC2 | 32 MHz crystal |
| ANT1 | RF antenna | Internal, routed through matching network |
| VDDUSB | USB power in | 5 V from VBUS |
| VDD | Digital power | 3.3 V |
| P0.06 | Green LED | Optional |
| P0.08 | Red LED | Optional, blinks in DFU mode |
| P1.09 | Green LED | Optional |
| P0.12 | Blue LED | Optional |
| P1.06 | SW0 (button) | Optional, enters MCUboot recovery |

All other GPIO, SPI, I2C, UART pins are **unused** in dongle firmware.
The nRF52840 only needs its USB and radio peripherals to function as a dongle.

---

## 5. Firmware Files to Create in `zmk-vua-v2-fw`

### Folder structure

```
boards/shields/Chutchit_dongle/
├── Kconfig.shield
├── Kconfig.defconfig
├── Chutchit_dongle.conf
├── Chutchit_dongle.zmk.yml
└── Chutchit_dongle.overlay
```

### `Kconfig.shield`
```kconfig
config SHIELD_CHUTCHIT_DONGLE
    def_bool $(shields_list_contains,Chutchit_dongle)
```

### `Kconfig.defconfig`
```kconfig
if SHIELD_CHUTCHIT_DONGLE

config ZMK_KEYBOARD_NAME
    default "Chutchit Dongle"

endif
```

### `Chutchit_dongle.conf`
```conf
# Dongle: receives via 2.4 GHz, outputs via USB
CONFIG_ZMK_2G4_DONGLE=y
CONFIG_ZMK_USB=y
CONFIG_ZMK_BLE=n

# MUST be identical to keyboard's Chutchit.conf
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""          # set a real key before daily use

CONFIG_ZMK_ESB=y
CONFIG_ZMK_POINTING=y              # required for mouse report passthrough
```

### `Chutchit_dongle.zmk.yml`
```yaml
file_format: "1"
id: Chutchit_dongle
name: Chutchit Dongle
type: shield
arch: arm
output:
  - usb
features:
  - keys
```

### `Chutchit_dongle.overlay`
```dts
// Dongle has no key matrix.
// All input arrives via ESB radio from the keyboard's left half.
```

---

## 6. Keyboard Config Changes

### `config/west.yml` — point to PH Design fork

```yaml
manifest:
  defaults:
    revision: 2g4-impl
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: ph-design
      url-base: https://github.com/ph-design
  projects:
    - name: zmk
      remote: ph-design
      repo-path: zmks
      revision: 2g4-impl
      import: app/west.yml
  self:
    path: config
```

### `config/Chutchit.conf` — enable 2.4 GHz on left half

```conf
CONFIG_ZMK_2G4=y
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""
CONFIG_ZMK_ESB=y
```

### `config/Chutchit.keymap` — add transport-select keys

Add to Layer 1 (LOWER) or ADJUST:
```c
&out OUT_USB    // force USB
&out OUT_BLE    // force BLE
&out OUT_2G4    // force 2.4 GHz dongle
&out OUT_TOG    // toggle BLE ↔ 2.4 GHz
```

### `build.yaml` — add dongle target

```yaml
  - board: nrf52840dongle_nrf52840
    shield: Chutchit_dongle
```

---

## 7. Flashing

### Flash order

```
Step 1: settings_reset ──────► Left half    (wipe old BLE/transport settings)
Step 2: Chutchit_left ────────► Left half
Step 3: Chutchit_right ───────► Right half  (no settings_reset needed)
Step 4: Chutchit_num ─────────► Num half    (no settings_reset needed)
Step 5: Chutchit_dongle ──────► PCA10059
```

### Flashing the PCA10059 dongle

**Install nrfutil (one time):**
```bash
pip install nrfutil
nrfutil install nrf5sdk-tools
```

**Enter DFU mode:**
- Plug dongle into USB
- Press and hold the **RESET button** (on the side edge, push inward toward USB plug)
- Red LED will fade in/out slowly → DFU mode active

**Flash:**
```bash
# Create DFU package from .hex
nrfutil nrf5sdk-tools pkg generate \
    --hw-version 52 \
    --sd-req=0x00 \
    --application Chutchit_dongle-nrf52840dongle_nrf52840-zmk.hex \
    --application-version 1 \
    chutchit_dongle.zip

# Flash over USB serial
# macOS:
nrfutil nrf5sdk-tools dfu usb-serial -pkg chutchit_dongle.zip -p /dev/cu.usbmodem*
# Linux:
nrfutil nrf5sdk-tools dfu usb-serial -pkg chutchit_dongle.zip -p /dev/ttyACM0
# Windows:
nrfutil nrf5sdk-tools dfu usb-serial -pkg chutchit_dongle.zip -p COM3
```

> If the build outputs a `.uf2` file: drag-and-drop it onto the `MDBT50Q` drive
> that appears when the dongle is in DFU mode — no command line required.

---

## 8. Radio Settings — Must Match on Both Sides

These values must be **identical** in `Chutchit.conf` (keyboard) and
`Chutchit_dongle.conf` (dongle). A single mismatch breaks the link entirely.

| Kconfig | Default | Meaning |
|---|---|---|
| `ZMK_2G4_RF_CHANNEL` | `76` | Frequency = 2400 + 76 = **2476 MHz** |
| `ZMK_2G4_ADDR_BASE_0..3` | `0x50 0x48 0x44 0x53` | 4-byte base address = "PHDS" |
| `ZMK_2G4_ADDR_PREFIX` | `0x4E` | Prefix = "N" → pipe 0 full address = "PHDSN" |
| `ZMK_2G4_AES_KEY` | `""` | AES-128 key as 32 hex chars = 16 bytes |

**Generate a random encryption key:**
```bash
openssl rand -hex 16
# Example output: a1b2c3d4e5f60718293a4b5c6d7e8f90
```

Set the **same value** in both configs:
```conf
CONFIG_ZMK_2G4_AES_KEY="a1b2c3d4e5f60718293a4b5c6d7e8f90"
```

---

## 9. Transport Switching Behaviour

| Situation | Active transport | How |
|---|---|---|
| USB cable plugged into keyboard | USB | Automatic, always overrides |
| USB unplugged | Last saved wireless preference | Automatic |
| First boot / no saved preference | BLE | Hardcoded default |
| Press `OUT_2G4` | 2.4 GHz (dongle) | Saved to flash, survives reboot |
| Press `OUT_BLE` | BLE | Saved to flash |
| Press `OUT_TOG` | Toggle BLE ↔ 2.4 GHz | Saved to flash |

**Technical details:**
- BLE ↔ 2.4 GHz switch: radio must fully stop then restart → ~50 ms latency
- 1-second watchdog on radio switch: hangs trigger a cold reboot
- Keyboard sends keepalive every 250 ms; dongle releases all keys after 500 ms silence
- Link declared lost after 40 consecutive TX failures (~10 seconds)

---

## 10. Debug & Verification

**Verify radio link on keyboard side:**
Enable `CONFIG_ZMK_USB_LOGGING=y` in `Chutchit.conf`, connect USB, open serial
terminal at 115200 baud. Expect to see:

```
[INF] 2.4G transport started
[INF] 2.4G BOOT sent: reason=0x00000000 session=0xABCD1234 ret=0
[INF] 2.4G keyboard connected (rf_len=12, total=1)
```

**Verify dongle receives packets:**
Enable `CONFIG_ZMK_USB_LOGGING=y` in `Chutchit_dongle.conf`, check dongle log:
```
[INF] 2.4G dongle receiver started (ch=76)
[INF] Keyboard BOOT: reset_reason=0x00000000 session=0xABCD1234
[INF] 2.4G keyboard connected (rf_len=12, total=1)
```
