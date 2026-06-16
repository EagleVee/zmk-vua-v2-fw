# Chutchit Keyboard — 3-Mode Dongle Implementation Summary

> **Purpose:** This document summarises both repositories so they can be combined
> to add 2.4 GHz dongle support (3-mode: USB + BLE + 2.4 GHz) to the Chutchit keyboard.
>
> **Branch read:** `zmk-vua-v2-fw @ vua-v2` + `zmks @ 2g4-impl`

---

## 1. Repository Overview

| Repository | Path | Branch | Purpose |
|---|---|---|---|
| **zmk-vua-v2-fw** | `/Users/nnavu/OtherProjects/zmk-vua-v2-fw` | `vua-v2` | Chutchit keyboard firmware (config-only repo) |
| **zmks** | `/Users/nnavu/OtherProjects/zmks` | `2g4-impl` | ZMK fork by PH Design — adds 2.4 GHz ESB dongle transport |

---

## 2. Chutchit Hardware (`zmk-vua-v2-fw @ vua-v2`)

### MCU / Board
- **Processor:** nRF52840 (ARM Cortex-M4)
- **Board:** `nice_nano` (Nice! Nano v2) — three separate MCUs, one per half
- **Split topology:** **3-way split** — `Chutchit_left` (central), `Chutchit_right` (peripheral), `Chutchit_num` (peripheral)
- **`CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2`** — central maintains BLE links to both right and num halves

### Key Matrix
- **5 rows × 19 columns** (columns distributed across the three halves)
- **Diode direction:** `col2row`

**Row GPIOs** (shared across all halves, active HIGH with pull-down):
| Row | GPIO |
|---|---|
| 0 | GPIO0.31 |
| 1 | GPIO0.29 |
| 2 | GPIO0.2 |
| 3 | GPIO1.15 |
| 4 | GPIO1.13 |

**Column GPIOs per half:**

| Half | Col offset | Active columns (GPIOs) | Count |
|---|---|---|---|
| **Left** (central) | 0 | GPIO0.17, GPIO0.20, GPIO0.22, GPIO0.24, GPIO1.0, GPIO0.11 | 6 |
| **Right** (peripheral) | 6 | GPIO0.9, GPIO1.6, GPIO1.4, GPIO0.11, GPIO1.0, GPIO0.24, GPIO0.22, GPIO0.20, GPIO0.17 | 9 |
| **Num** (peripheral) | 15 | GPIO0.24, GPIO0.22, GPIO0.20, GPIO0.17 | 4 |
| **Total** | — | — | 19 |

> Left overlay has 2 additional column slots commented out (`GPIO1.4`, `GPIO1.6`), currently unused.

### Encoders
- **Left encoder:** Pin A = GPIO1.2, Pin B = GPIO1.7 — active LOW, pull-up, 24 steps (disabled by default)
- **Right encoder:** Same GPIO1.2 / GPIO1.7 on its own MCU — enabled by default on right overlay
- Triggers per rotation: 12

### ZMK Version
```yaml
# config/west.yml
manifest:
  defaults:
    revision: v0.3          # ← pinned to ZMK v0.3
  projects:
    - name: zmk
      remote: zmkfirmware   # github.com/zmkfirmware/zmk
      import: app/west.yml
```

### Build Matrix (`build.yaml`)
```yaml
include:
  - board: nice_nano
    shield: Chutchit_left
    snippet: studio-rpc-usb-uart
    cmake-args: -DCONFIG_ZMK_STUDIO=y -DCONFIG_ZMK_STUDIO_LOCKING=n -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=y

  - board: nice_nano
    shield: Chutchit_right
    cmake-args: -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n

  - board: nice_nano
    shield: Chutchit_num
    cmake-args: -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n

  - board: nice_nano
    shield: settings_reset
```

### Active Features (`config/Chutchit.conf`)
```conf
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_TIMEOUT=30000          # 30 s idle
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=600000   # 10 min deep sleep
CONFIG_ZMK_BATTERY_REPORT_INTERVAL=1200
CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2
CONFIG_ZMK_EXT_POWER=y
CONFIG_ZMK_POINTING=y
CONFIG_ZMK_STUDIO_LOCKING=n            # Studio keymap unlocked
CONFIG_ZMK_KEYBOARD_NAME="Chutchit"
CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
CONFIG_EC11=y
CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y
CONFIG_ZMK_POINTING_SMOOTH_SCROLLING=y
CONFIG_SENSOR_LOG_LEVEL_DBG=y
CONFIG_BT_GATT_ENFORCE_SUBSCRIPTION=y
# CONFIG_ZMK_USB_LOGGING=y            # disabled (battery intensive)
```

### Shield Kconfig (`Kconfig.shield` / `Kconfig.defconfig`)
```
SHIELD_CHUTCHIT_LEFT   — central, ZMK_KEYBOARD_NAME="Chutchit", SPLIT_ROLE_CENTRAL=y
SHIELD_CHUTCHIT_RIGHT  — peripheral
SHIELD_CHUTCHIT_NUM    — peripheral (number row section)
CONFIG_ZMK_SOFT_OFF=y  — 3-second hold to power off
```

### Keymap Summary (5 layers)
- **Layer 0 (DEFAULT):** QWERTY + mod-taps; `&lt 1 SPACE` / `&lt 1 ENTER`; mouse buttons on thumb cluster; `&mt CAPS ESC`
- **Layer 1 (LOWER):** Navigation, media control, mouse movement
- **Layers 2–3:** Empty (transparent)
- **Layer 4 (FUNCTION):** Unused in current keymap
- **Behaviors:** Tap-dance `Td_01` (`=`→`*`→`/`→`PAUSE_BREAK`), mod-taps, layer-taps
- **Combos:** BT clear-all (positions 6+7 on layer 1), left/right parentheses
- **Mouse:** `&mmv MOVE_*` (speed 1200), `&msc SCRL_*` (speed 900), `&mkp MB1/MB2/MB3`
- **Encoder:** Scroll up/down + left/right on rotary encoder

### GitHub Actions (`Build Firmware.yml`)
```yaml
on: [push, pull_request, workflow_dispatch]
jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
```

---

## 3. PH Design ZMK Fork (`zmks @ 2g4-impl`)

### Fork Details
- **Remote:** `git@github.com:ph-design/zmks.git`
- **Branch:** `2g4-impl`
- **Base:** ZMK v0.3 + Zephyr v3.5.0+zmk-fixes
- **Version compatibility with Chutchit:** ✅ Both repos target ZMK v0.3

### What the Fork Adds

| Feature | Source file | Kconfig symbol |
|---|---|---|
| 2.4 GHz keyboard transport (PTX) | `app/src/2g4.c` | `CONFIG_ZMK_2G4` |
| 2.4 GHz dongle receiver (PRX) | `app/src/2g4_dongle.c` | `CONFIG_ZMK_2G4_DONGLE` |
| AES-128-CCM encryption | `app/src/2g4_crypto.c` | `CONFIG_ZMK_2G4_AES_KEY` |
| Nordic ESB radio driver | `app/module/drivers/esb/esb.c` | `CONFIG_ZMK_ESB` |
| `OUT_2G4` keymap behavior | `app/src/behaviors/behavior_outputs.c` | `CONFIG_ZMK_2G4` |
| Third endpoint type | `app/include/zmk/endpoints_types.h` | (auto) |
| BLE ↔ 2.4 GHz radio coexistence | `app/src/main.c` + `endpoints.c` | (BLE + 2G4 both enabled) |

### Transport Architecture
```
USB  ──┐
BLE  ──┼── zmk_endpoints ──► keymap behaviors ──► host PC
2G4  ──┘
        (OUT_USB / OUT_BLE / OUT_2G4 / OUT_TOG)
```

### ESB Radio Driver Config
```kconfig
CONFIG_ZMK_ESB=y
# Requires: SOC_SERIES_NRF52X, NRFX_PPI, DYNAMIC_INTERRUPTS (nRF52840 satisfies all)
CONFIG_ZMK_ESB_MAX_PAYLOAD_LENGTH=32   # bytes, 1-252
CONFIG_ZMK_ESB_TX_FIFO_SIZE=8
CONFIG_ZMK_ESB_RX_FIFO_SIZE=8
CONFIG_ZMK_ESB_PIPE_COUNT=8
```

### 2.4 GHz Kconfig Options
```kconfig
# ── Shared (keyboard + dongle) ──────────────────────────────────────
CONFIG_ZMK_2G4_TX_POWER=4             # dBm, range -40 to +8
CONFIG_ZMK_2G4_RF_CHANNEL=76          # 2400 + 76 = 2476 MHz (range 0-100)

# ESB pipe 0 address = bytes BASE_0–3 + PREFIX → "PHDSN" by default
CONFIG_ZMK_2G4_ADDR_BASE_0=0x50       # 'P'
CONFIG_ZMK_2G4_ADDR_BASE_1=0x48       # 'H'
CONFIG_ZMK_2G4_ADDR_BASE_2=0x44       # 'D'
CONFIG_ZMK_2G4_ADDR_BASE_3=0x53       # 'S'
CONFIG_ZMK_2G4_ADDR_PREFIX=0x4E       # 'N'

CONFIG_ZMK_2G4_AES_KEY=""             # 32 hex chars = 16-byte AES-128 key
                                       # empty string = encryption disabled

# ── Keyboard PTX only ────────────────────────────────────────────────
CONFIG_ZMK_2G4=y                      # mutually exclusive with ZMK_2G4_DONGLE
CONFIG_ZMK_2G4_INIT_PRIORITY=50
CONFIG_ZMK_2G4_RETRANSMIT_COUNT=5     # 0-15
CONFIG_ZMK_2G4_RETRANSMIT_DELAY=500   # µs, 500-1000
CONFIG_ZMK_2G4_MAX_CONSEC_FAILURES=40 # link declared down after 40 failures ≈ 10 s

# ── Dongle PRX only ──────────────────────────────────────────────────
CONFIG_ZMK_2G4_DONGLE=y               # mutually exclusive with ZMK_2G4
CONFIG_ZMK_2G4_DONGLE_INIT_PRIORITY=51
```

### Over-the-Air Protocol Messages

**Keyboard → Dongle:**
| ID | Name | Notes |
|---|---|---|
| `0x01` | `ZMK_2G4_MSG_KEYBOARD_REPORT` | HID keyboard report |
| `0x02` | `ZMK_2G4_MSG_CONSUMER_REPORT` | Media/consumer keys |
| `0x03` | `ZMK_2G4_MSG_MOUSE_REPORT` | Mouse (if `CONFIG_ZMK_POINTING`) |
| `0x04` | `ZMK_2G4_MSG_SYSTEM_REPORT` | System power keys |
| `0x05` | `ZMK_2G4_MSG_BOOT` | Boot + reset cause + session ID |
| `0x10` | `ZMK_2G4_MSG_STUDIO_RPC_TX` | ZMK Studio RPC passthrough |
| `0xFE` | `ZMK_2G4_MSG_KEEP_ALIVE` | Sent every 250 ms; prevents dongle timeout |

**Dongle → Keyboard (ACK payload):**
| ID | Name | Notes |
|---|---|---|
| `0x01` | `ZMK_2G4_ACK_LED_INDICATORS` | LED state from host |
| `0x02` | `ZMK_2G4_ACK_RESOLUTION_MULTIPLIER` | Mouse resolution |
| `0x03` | `ZMK_2G4_ACK_HOST_CONNECTED` | USB host connected status |
| `0x10` | `ZMK_2G4_ACK_STUDIO_RPC_RX` | Studio RPC response |

### Multi-Transport Switching Behaviour
- USB is always prioritised when the cable is connected
- 50 ms radio settle time when switching BLE ↔ 2.4 GHz
- Preferred wireless transport persisted to ZMK settings (`endpoints/preferred`)
- Manual selection: `&out OUT_USB`, `&out OUT_BLE`, `&out OUT_2G4`, `&out OUT_TOG`
- Link-down detection: 40 consecutive TX failures ≈ 10 s → marks 2.4 GHz not ready
- Dongle releases all keys after **500 ms** of silence; keyboard keepalive is every **250 ms**

### Fork's West Manifest (`app/west.yml`)
```yaml
remotes:
  - name: zmkfirmware
    url-base: https://github.com/zmkfirmware
  - name: ph-design
    url-base: https://github.com/ph-design
projects:
  - name: zephyr
    remote: zmkfirmware
    revision: v3.5.0+zmk-fixes
  - name: zmk-studio-messages
    revision: v0.3_lts
    path: modules/msgs/zmk-studio-messages
    remote: ph-design
```

---

## 4. What Needs to Change in `zmk-vua-v2-fw`

### 4.1 `config/west.yml` — Point to the fork

Replace the current `zmk` project entry:

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
      repo-path: zmks          # github.com/ph-design/zmks
      revision: 2g4-impl
      import: app/west.yml
  self:
    path: config
```

> The fork's `app/west.yml` already pulls in the correct Zephyr (`v3.5.0+zmk-fixes`), nanopb, and zmk-studio-messages dependencies. No extra manifest entries needed.

### 4.2 `config/Chutchit.conf` — Enable 2.4 GHz on the keyboard (central half)

Add these lines:

```conf
# ── 2.4 GHz transport (keyboard/PTX side) ──────────────────────
CONFIG_ZMK_2G4=y

# ── Radio settings — must match dongle exactly ──────────────────
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""          # set a 32-hex-char key before shipping

# ── ESB driver ──────────────────────────────────────────────────
CONFIG_ZMK_ESB=y
```

> These apply to the **left (central)** half only. The `right` and `num` peripherals communicate with the central over the existing BLE split link — no changes needed in their configs.

### 4.3 `config/Chutchit.keymap` — Add output-switch keys

Add `&out OUT_2G4` to a convenient layer. The `outputs.h` include is already present:

```c
// Example placement in Layer 1 (LOWER) alongside existing &bt bindings:
&out OUT_USB   // Switch to USB
&out OUT_BLE   // Switch to BLE
&out OUT_2G4   // Switch to 2.4 GHz dongle  ← ADD THIS
&out OUT_TOG   // Cycle through all three
```

### 4.4 `build.yaml` — Add dongle build target

```yaml
  # ── NEW: 2.4 GHz dongle receiver ───────────────────────────────
  # Replace <dongle_board> with the actual target board name, e.g.:
  #   nrf52840dongle_nrf52840    (Nordic USB dongle)
  #   seeeduino_xiao_ble         (Seeed XIAO BLE)
  - board: <dongle_board>
    shield: Chutchit_dongle
```

### 4.5 New dongle shield to create

Create `boards/shields/Chutchit_dongle/` with these files:

#### `Kconfig.shield`
```kconfig
config SHIELD_CHUTCHIT_DONGLE
    def_bool $(shields_list_contains,Chutchit_dongle)
```

#### `Kconfig.defconfig`
```kconfig
if SHIELD_CHUTCHIT_DONGLE

config ZMK_KEYBOARD_NAME
    default "Chutchit Dongle"

endif
```

#### `Chutchit_dongle.conf`
```conf
# Dongle: receives via 2.4 GHz, outputs via USB
CONFIG_ZMK_2G4_DONGLE=y
CONFIG_ZMK_USB=y
CONFIG_ZMK_BLE=n

# Must match keyboard config exactly
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""         # must equal keyboard's AES key

CONFIG_ZMK_ESB=y
CONFIG_ZMK_POINTING=y             # enable if you want mouse passthrough
```

#### `Chutchit_dongle.zmk.yml`
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

#### `Chutchit_dongle.overlay`
```dts
// Minimal overlay: dongle has no key matrix, just USB output
/ {
    chosen {
        zmk,kscan = &kscan0;
    };
};
```

> The dongle MCU does not scan any keys — all inputs arrive over ESB from the keyboard and are forwarded to the USB host.

---

## 5. Critical Constraints and Notes

### Split Architecture
- **Only the left (central) half** communicates with the dongle over 2.4 GHz.
- The **right and num halves** communicate with the left via their existing ZMK BLE split links — these links are unaffected.
- On the left half: USB + BLE split (to right & num) + 2.4 GHz (to dongle) all operate simultaneously.

### Version Compatibility
- Chutchit is pinned to `zmkfirmware/zmk@v0.3`; the fork (`ph-design/zmks@2g4-impl`) is also built on ZMK v0.3. ✅ Compatible.

### Hardware for the Dongle
- Any nRF52840 USB device works: Nordic USB Dongle (`nrf52840dongle_nrf52840`), second Nice! Nano, Seeed XIAO BLE, etc.
- No hardware changes needed to the keyboard halves themselves.

### Encryption
- AES-128-CCM is hardware accelerated on nRF52840.
- Set the **same** 32 hex character key (`CONFIG_ZMK_2G4_AES_KEY`) in both `Chutchit.conf` and `Chutchit_dongle.conf`.
- Leave empty during development to avoid decryption mismatches.

### ZMK Studio
- The fork forwards Studio RPC over 2.4 GHz (`ZMK_2G4_MSG_STUDIO_RPC_TX`), so Studio works wirelessly through the dongle without a USB cable.
- The `snippet: studio-rpc-usb-uart` on the left half build keeps Studio working over USB too.

### GitHub Actions
- The existing workflow uses `build-user-config.yml@v0.3` from `zmkfirmware/zmk`.
- After switching west.yml to the fork, the workflow will still build correctly because the fork exposes the same reusable workflow — **or** you may need to pin the workflow reference to the fork's copy. Verify after the first build.

---

## 6. Step-by-Step Implementation Checklist

- [ ] **1. Update `config/west.yml`** to point to `ph-design/zmks@2g4-impl` (§4.1)
- [ ] **2. Add 2.4 GHz Kconfig** to `config/Chutchit.conf` (§4.2)
- [ ] **3. Add `&out OUT_2G4`** to the keymap in `config/Chutchit.keymap` (§4.3)
- [ ] **4. Update `build.yaml`** with the dongle build target (§4.4)
- [ ] **5. Create dongle shield** under `boards/shields/Chutchit_dongle/` — 5 files (§4.5)
- [ ] **6. Choose dongle hardware** — buy/use an nRF52840 USB dongle
- [ ] **7. Set matching AES keys** in both `Chutchit.conf` and `Chutchit_dongle.conf`
- [ ] **8. Build and flash** all three keyboard halves + dongle
- [ ] **9. Verify all three modes**: USB cable, BLE host, 2.4 GHz dongle

---

## 7. File Change Summary

| File | Action | Key change |
|---|---|---|
| `config/west.yml` | **Modify** | Point `zmk` project to `ph-design/zmks@2g4-impl` |
| `config/Chutchit.conf` | **Modify** | Add `CONFIG_ZMK_2G4=y` + channel/key/ESB options |
| `config/Chutchit.keymap` | **Modify** | Add `&out OUT_2G4` binding to a layer |
| `build.yaml` | **Modify** | Add dongle board entry |
| `boards/shields/Chutchit_dongle/Kconfig.shield` | **Create** | Define `SHIELD_CHUTCHIT_DONGLE` |
| `boards/shields/Chutchit_dongle/Kconfig.defconfig` | **Create** | Default keyboard name |
| `boards/shields/Chutchit_dongle/Chutchit_dongle.conf` | **Create** | `CONFIG_ZMK_2G4_DONGLE=y` + radio params |
| `boards/shields/Chutchit_dongle/Chutchit_dongle.zmk.yml` | **Create** | Shield metadata |
| `boards/shields/Chutchit_dongle/Chutchit_dongle.overlay` | **Create** | Minimal DTS overlay |
