# Chutchit 2.4 GHz Dongle Implementation Guide

## What is the dongle?

The dongle is a small nRF52840 device that you plug directly into your computer's
USB port. It receives keypresses from the Chutchit keyboard over a 2.4 GHz radio
link and presents itself to the computer as a standard USB HID keyboard/mouse. No
driver installation is needed.

---

## Hardware: Nordic nRF52840 USB Dongle (PCA10059)

This is the recommended hardware. It has a **USB-A male plug** that inserts
directly into your computer — no cable needed.

```
┌─────────────────────────┐
│  nRF52840 chip          │ ██ USB-A ──► into your PC
│  PCB antenna            │
│  Reset button           │
│  RGB LED                │
└─────────────────────────┘
     ~35 × 13 mm
```

- **Where to buy:** Nordic Semiconductor official store, Mouser, Digikey, or
  Amazon (~$10 USD)
- **Product page:** https://www.nordicsemi.com/Products/Development-hardware/nRF52840-Dongle
- **Zephyr board name:** `nrf52840dongle_nrf52840`

### USB-C computers

The PCA10059 has USB-A. If your computer only has USB-C ports, use a **USB-A to
USB-C passive adapter** (not a cable hub). The dongle still plugs in directly —
the adapter just changes the physical connector shape.

Alternatively, you can design a custom PCB with a **USB-C male plug** using the
same nRF52840 chip — but the Nordic dongle is the fastest path to get started.

---

## How it works

```
Chutchit Left Half                  Dongle (PCA10059)
┌─────────────────────┐             ┌───────────────────┐
│ Scans key matrix    │  2.4 GHz   │ ESB PRX mode      │
│ Runs keymap/layers  │ ─────────► │ Decrypts payload  │
│ Produces HID report │  ESB radio │ Forwards over USB  │──► PC
│ Encrypts & sends    │             └───────────────────┘
└─────────────────────┘
        ▲ BLE split
┌───────┴─────────────┐
│ Right half + Num    │
└─────────────────────┘
```

The dongle does **not** contain your keymap. It only receives pre-processed HID
reports from the left half and relays them to the PC via USB. Your keymap lives
only in the left half's flash — same as USB and BLE modes.

---

## Shield files to create

Create the folder `boards/shields/Chutchit_dongle/` in your `zmk-vua-v2-fw` repo
with the following files:

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
# Dongle receives over 2.4 GHz and outputs over USB
CONFIG_ZMK_2G4_DONGLE=y
CONFIG_ZMK_USB=y
CONFIG_ZMK_BLE=n

# MUST match the keyboard's config exactly
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""

CONFIG_ZMK_ESB=y
CONFIG_ZMK_POINTING=y
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
// The dongle has no key matrix. All input arrives via ESB radio.
```

---

## Keyboard config changes

### `config/west.yml` — point to the fork

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

### `config/Chutchit.conf` — enable 2.4 GHz on the left half

Add these lines:

```conf
CONFIG_ZMK_2G4=y
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""
CONFIG_ZMK_ESB=y
```

### `config/Chutchit.keymap` — add transport-switch keys

Add to a layer (e.g. Layer 1 / LOWER):

```c
&out OUT_USB    // force USB
&out OUT_BLE    // force BLE
&out OUT_2G4    // force 2.4 GHz dongle
&out OUT_TOG    // toggle BLE ↔ 2.4 GHz
```

### `build.yaml` — add dongle build target

```yaml
  - board: nrf52840dongle_nrf52840
    shield: Chutchit_dongle
```

---

## Building the firmware

### GitHub Actions (recommended)

Push to your repo. GitHub Actions will build all targets including the dongle.
Download `Chutchit_dongle-nrf52840dongle_nrf52840-zmk.uf2` from the Artifacts.

### Local build

```bash
cd app
west build -b nrf52840dongle_nrf52840 -- -DSHIELD=Chutchit_dongle
```

---

## Flashing the dongle

The PCA10059 ships with Nordic's USB bootloader. You **do not need a debugger**.

### Step 1 — Install nrfutil
```bash
pip install nrfutil
```

### Step 2 — Enter DFU mode
With the dongle plugged into USB, **press and hold the reset button** for 1–2
seconds. The red LED will begin pulsing slowly. The dongle is now in DFU mode
and will appear as a USB device.

### Step 3 — Flash

Convert the ZMK `.hex` to a DFU package and flash it:

```bash
nrfutil dfu genpkg --dev-type 0x0052 --application zmk.hex dfu_package.zip
nrfutil dfu usb-serial -pkg dfu_package.zip -p /dev/ttyACM0
```

Replace `/dev/ttyACM0` with the actual port. On macOS it will be
`/dev/cu.usbmodem*`. On Windows it will be `COMx`.

> **Tip:** If your ZMK build outputs a `.uf2` file for this board, you can also
> drag-and-drop it onto the `MDBT50Q` drive that appears when the dongle is in
> DFU mode — no command line needed.

---

## Flashing the keyboard halves

Flash the settings_reset firmware first on the **left half only** to wipe old
BLE/transport settings, then reflash with the new left firmware:

```
1. Flash settings_reset → left half
2. Flash Chutchit_left  → left half
3. Flash Chutchit_right → right half  (no settings_reset needed)
4. Flash Chutchit_num   → num half    (no settings_reset needed)
5. Flash Chutchit_dongle → PCA10059 dongle
```

---

## Radio settings — must match on both sides

These Kconfig values must be **identical** in `Chutchit.conf` (keyboard) and
`Chutchit_dongle.conf` (dongle). If they differ, the radio link will not work.

| Setting | Default | Notes |
|---|---|---|
| `CONFIG_ZMK_2G4_RF_CHANNEL` | `76` | 2400 + 76 = 2476 MHz |
| `CONFIG_ZMK_2G4_ADDR_BASE_0..3` | `0x50 0x48 0x44 0x53` | "PHDS" |
| `CONFIG_ZMK_2G4_ADDR_PREFIX` | `0x4E` | "N" → pipe 0 = "PHDSN" |
| `CONFIG_ZMK_2G4_AES_KEY` | `""` | Set a 32 hex-char key before shipping |

---

## AES encryption (optional but recommended)

During development, leave `CONFIG_ZMK_2G4_AES_KEY=""` (no encryption) for easy
debugging.

Before daily use, set a 16-byte key expressed as 32 hex characters — **same
value on keyboard and dongle**:

```conf
# Example key — generate your own random value
CONFIG_ZMK_2G4_AES_KEY="a1b2c3d4e5f60718293a4b5c6d7e8f90"
```

Generate a random key:
```bash
openssl rand -hex 16
```

---

## Transport switching behaviour

| Situation | Active transport |
|---|---|
| USB cable plugged into keyboard | USB (automatic, no key press needed) |
| USB unplugged | Last saved wireless preference |
| First boot | BLE (default) |
| Press `OUT_2G4` | 2.4 GHz via dongle, saved to flash |
| Press `OUT_BLE` | BLE, saved to flash |
| Press `OUT_TOG` | Toggles BLE ↔ 2.4 GHz |

Switching between BLE and 2.4 GHz takes ~50 ms (radio settle time). USB
override is instant. Your last wireless preference survives power cycles.

---

## Keepalive & timeout

- Keyboard sends a keepalive packet every **250 ms** when idle
- Dongle releases all keys after **500 ms** of silence (safety net for dropped packets)
- Link considered lost after **40 consecutive TX failures** (~10 seconds)
