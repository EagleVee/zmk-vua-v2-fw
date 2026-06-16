# Hướng Dẫn Chi Tiết Triển Khai Dongle 2.4 GHz — Chutchit

---

## 1. Tổng quan thiết kế

Dongle là một board nRF52840 chạy ZMK fork của PH Design với
`CONFIG_ZMK_2G4_DONGLE=y`. Nó hoạt động như một **USB HID relay thuần túy**:

```
Chutchit Left Half                        Dongle PCA10059
┌──────────────────────────────┐          ┌─────────────────────────────┐
│                              │          │                             │
│  Quét phím ──► Keymap engine │  ESB     │  ESB PRX (radio receiver)  │
│  Tạo HID report              │  2.4GHz  │  AES-128-CCM decrypt       │
│  AES encrypt ──► ESB PTX ────┼─────────►│  zmk_usb_hid_send_*()      │──► PC USB
│                              │          │                             │
└──────────────────────────────┘          └─────────────────────────────┘
        ▲ BLE split link
┌───────┴──────────────────────┐
│  Right half + Num half       │
└──────────────────────────────┘
```

Dongle không chứa keymap. Nó chỉ nhận HID bytes đã được xử lý từ nửa trái
và forward thẳng qua USB.

---

## 2. Danh sách linh kiện (BOM)

### Option A — Dùng Nordic PCA10059 (nhanh nhất, không cần hàn)

| # | Linh kiện | Thông số | Số lượng | Ghi chú |
|---|---|---|---|---|
| 1 | **Nordic nRF52840 Dongle (PCA10059)** | USB-A male, nRF52840, 1MB flash | 1 | Có sẵn bootloader USB, không cần JLink |

Mua tại: Mouser (Part# 768-PCA10059), Digikey (1490-PCA10059-ND), hoặc Nordic
webshop. Giá ~$10 USD.

---

### Option B — Thiết kế PCB custom nhỏ hơn (USB-A hoặc USB-C cắm thẳng)

Đây là toàn bộ linh kiện cần thiết cho một dongle tự thiết kế:

| # | Linh kiện | Thông số kỹ thuật | SL | Footprint | Ghi chú |
|---|---|---|---|---|---|
| 1 | **nRF52840-QIAA** | ARM Cortex-M4F, 1MB flash, 256KB RAM, 2.4GHz, USB FS | 1 | QFN73 (7×7mm) hoặc QFN48 (6×6mm) | IC chính, bắt buộc |
| 2 | **Crystal 32 MHz** | 32MHz, ±10ppm, CL=8pF | 1 | 2016 (2×1.6mm) | Main clock cho USB & radio |
| 3 | **Crystal 32.768 kHz** | 32.768kHz, CL=7pF | 1 | 3215 (3.2×1.5mm) | RTC / sleep clock |
| 4 | **Tụ lọc USB D+/D-** | 1nF, 50V, C0G/NP0 | 2 | 0402 | Lọc EMI trên cặp USB differential |
| 5 | **Tụ bypass VDD** | 100nF, 10V, X5R | 4 | 0402 | Đặt gần mỗi chân VDD của chip |
| 6 | **Tụ bulk VDD** | 10µF, 10V, X5R | 1 | 0805 | Dự trữ năng lượng |
| 7 | **Tụ VDDUSB** | 4.7µF + 100nF | 2 | 0402/0603 | Lọc nguồn USB 3.3V |
| 8 | **Điện trở pull-up D+** | 1.5kΩ, 1% | 1 | 0402 | Chỉ cần nếu không dùng nội bộ chip |
| 9 | **Anten 2.4GHz** | Chip antenna 2.4GHz, ví dụ Johanson 0433AT62A0100E | 1 | 0402 / SMD | Hoặc anten trace PCB (miễn phí) |
| 10 | **Tụ matching anten** | Theo datasheet anten (thường 1–5pF) | 2–3 | 0402 | Matching network |
| 11 | **DC-DC inductor** | 10µH cho DCDC converter nội bộ nRF | 1 | 0402 | Bắt buộc nếu dùng DC-DC mode |
| 12 | **Nút reset** | SPST, 50mA, SMD | 1 | 3×4mm | Để vào bootloader DFU |
| 13 | **LED** | RGB LED hoặc 1 LED đơn | 1 | 0603 | Tùy chọn, hiển thị trạng thái |
| 14 | **Điện trở LED** | 33–100Ω | 1–3 | 0402 | Current limiting |
| 15 | **Đầu cắm USB-A male** | USB 2.0 Type-A, PCB mount thẳng | 1 | Horizontal PCB | Cắm thẳng vào máy tính |
| _(alt)_ | **Đầu cắm USB-C male** | USB 2.0 Type-C, 24-pin, GX01 series | 1 | PCB mount | Nếu muốn USB-C |
| 16 | **Decoupling VDDIO** | 100nF mỗi rail | 2–4 | 0402 | Các rail GPIO |

**Tổng chi phí linh kiện custom (ước tính):** ~$3–5 USD/đơn vị ở số lượng nhỏ.  
**Kích thước PCB có thể đạt được:** ~18×8mm (USB-A) hoặc ~16×8mm (USB-C).

---

## 3. Sơ đồ khối phần cứng (Pseudo Schematic)

### 3.1 Tổng quan nguồn điện

```
USB VBUS (5V)
    │
    ├──[Bulk cap 10µF]──GND
    │
    └──► VDDUSB pin (chân 6 nRF52840)
         │
         └──► Internal USB LDO của nRF52840 ──► 3.3V internal rail
                                                       │
                                    ┌──────────────────┤
                                    │                  │
                               [100nF×4]         [Inductor 10µH]
                                    │                  │
                                   GND           VDDMAIN (3.3V)
                                               (qua DC-DC nội bộ)
```

### 3.2 USB Interface

```
PC USB Host
    │
    ├── D+  ──[1nF to GND]── ──────────────── P0.13 (USB D+)  ─┐
    │                                                            │ nRF52840
    └── D-  ──[1nF to GND]── ──────────────── P0.15 (USB D-)  ─┘
         
    GND ──────────────────────────────────────── GND

    VBUS ─────────────────────────────────────── VDDUSB
```

> **Lưu ý:** USB D+/D- là differential pair — phải đi song song, cùng độ dài,
> trên cùng layer, không đi qua via nếu có thể. Tụ 1nF là common-mode filter.

### 3.3 Crystal 32 MHz (Main clock)

```
                    ┌─────────────┐
nRF52840            │             │
  XC1 (P0.00) ──── │  XTAL 32MHz │
  XC2 (P0.01) ──── │             │
                    └─────────────┘
       │                               │
   [CL/2 cap                       [CL/2 cap
    ~ 15pF]                         ~ 15pF]
       │                               │
      GND                             GND

  CL = crystal load capacitance (thường 8–12pF)
  Giá trị cap = CL × 2 (vì 2 cap nối series với nhau)
```

### 3.4 Crystal 32.768 kHz (RTC / Sleep)

```
                    ┌──────────────────┐
nRF52840            │                  │
  P0.00 ─────────── │  XTAL 32.768kHz  │
  P0.01 ─────────── │                  │
                    └──────────────────┘
       │                                    │
  [6–9pF]                              [6–9pF]
       │                                    │
      GND                                  GND
```

### 3.5 Anten 2.4 GHz

**Option 1: Chip antenna (đơn giản hơn)**
```
nRF52840                   Chip Antenna
  ANT1 ──[Cmatch1]──┬──[Cmatch2]──► ANT
                    │
                [Lmatch]
                    │
                   GND

  Ví dụ: Johanson 0433AT62A0100E
  Matching: C=1.8pF, L=1.5nH, C=1.8pF (theo datasheet anten)
```

**Option 2: PCB trace antenna (không tốn tiền)**
```
nRF52840
  ANT1 ──────────────────── [Trace anten ~31.2mm dạng meander]

  Không cần matching network nếu board <2 layers và follow
  Nordic reference layout từ nRF52840 Product Specification.
```

### 3.6 Nút Reset (DFU Mode)

```
                    VDD (3.3V)
                         │
                      [10kΩ pull-up]
                         │
nRF52840 P0.18 ──────────┤
(RESET pin)              │
                      [Reset button]
                         │
                        GND
```

> P0.18 là dedicated reset/GPIO. Nhấn giữ để vào Nordic DFU bootloader.

### 3.7 LED trạng thái (tùy chọn — theo PCA10059)

```
VDD (3.3V)
    │
    ├──[33Ω]── P0.06  (LED xanh lá — LED0)
    │
    ├──[33Ω]── P0.08  (LED đỏ    — LED1 Red,   nhấp nháy khi DFU mode)
    ├──[33Ω]── P1.09  (LED xanh  — LED1 Green)
    └──[33Ω]── P0.12  (LED xanh dương — LED1 Blue)

Tất cả đều ACTIVE_LOW (kéo xuống GND để sáng).
```

### 3.8 Sơ đồ layout PCB tổng quát (top view, không đúng tỉ lệ)

```
                    ← 18mm →
        ┌──────────────────────────┐
        │  ┌──────────┐   🔴LED   │  ↑
        │  │ nRF52840 │  [Reset]  │  8mm
        │  │  QFN48   │   XTAL    │  ↓
        │  └──────────┘  [Ant]    │
        └────────────┐            │
        USB-A male   ████████████ │  (cắm thẳng sang phải vào PC)
        ──────────────────────────┘
                 ↑
           Đầu cắm USB-A
           nhô ra ngoài PCB
```

---

## 4. Các pin nRF52840 được sử dụng trong firmware ZMK dongle

| Pin | Chức năng | Ghi chú |
|---|---|---|
| P0.13 | USB D+ | Fixed, không thay đổi được |
| P0.15 | USB D- | Fixed, không thay đổi được |
| P0.18 | RESET | Fixed, dùng cho DFU bootloader |
| P0.00 / P0.01 | XC1 / XC2 | Crystal 32 MHz |
| ANT1 | RF antenna | Internal, đi ra qua matching network |
| VDDUSB | Nguồn USB | 5V từ VBUS |
| VDD | Nguồn digital | 3.3V |
| P0.06 | LED xanh lá | Tùy chọn, status indicator |
| P0.08 | LED đỏ | Tùy chọn, nhấp nháy khi DFU |
| P1.09 | LED xanh | Tùy chọn |
| P0.12 | LED xanh dương | Tùy chọn |
| P1.06 | SW0 (button) | Tùy chọn, vào DFU MCUboot |

Tất cả các chân còn lại (GPIO, SPI, I2C, UART) **không dùng đến** trong
firmware dongle — nRF52840 chỉ cần USB và radio để làm dongle.

---

## 5. Firmware files cần tạo trong `zmk-vua-v2-fw`

### Cấu trúc thư mục

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
# Dongle nhận qua 2.4 GHz và xuất ra USB
CONFIG_ZMK_2G4_DONGLE=y
CONFIG_ZMK_USB=y
CONFIG_ZMK_BLE=n

# PHẢI khớp hoàn toàn với Chutchit.conf của bàn phím
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""          # đặt key thật trước khi dùng

CONFIG_ZMK_ESB=y
CONFIG_ZMK_POINTING=y              # cần thiết cho mouse report
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
// Dongle không có ma trận phím.
// Toàn bộ input đến qua ESB radio từ nửa trái bàn phím.
```

---

## 6. Thay đổi config bàn phím

### `config/west.yml` — trỏ sang fork PH Design

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

### `config/Chutchit.conf` — thêm 2.4 GHz cho nửa trái

```conf
# 2.4 GHz transport
CONFIG_ZMK_2G4=y
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""
CONFIG_ZMK_ESB=y
```

### `config/Chutchit.keymap` — thêm phím chuyển kết nối

Thêm vào Layer 1 (LOWER) hoặc ADJUST:
```c
&out OUT_USB    // ép dùng USB
&out OUT_BLE    // ép dùng BLE
&out OUT_2G4    // ép dùng dongle 2.4 GHz
&out OUT_TOG    // toggle BLE ↔ 2.4 GHz
```

### `build.yaml` — thêm dongle build

```yaml
  - board: nrf52840dongle_nrf52840
    shield: Chutchit_dongle
```

---

## 7. Nạp firmware (flashing)

### Thứ tự nạp firmware

```
Bước 1: settings_reset ──────► Nửa trái   (xóa flash settings cũ)
Bước 2: Chutchit_left ────────► Nửa trái
Bước 3: Chutchit_right ───────► Nửa phải  (không cần settings_reset)
Bước 4: Chutchit_num ─────────► Nửa num   (không cần settings_reset)
Bước 5: Chutchit_dongle ──────► PCA10059
```

### Nạp firmware vào PCA10059

**Cài nrfutil (một lần):**
```bash
pip install nrfutil
nrfutil install nrf5sdk-tools
```

**Vào DFU mode:**
- Cắm dongle vào USB
- Nhấn giữ nút **RESET** (ở cạnh bên, đẩy vào phía trong hướng đầu USB)
- Đèn đỏ sẽ fade in/out chậm → đang ở DFU mode

**Flash bằng nrfutil:**
```bash
# Tạo gói DFU từ file .hex
nrfutil nrf5sdk-tools pkg generate \
    --hw-version 52 \
    --sd-req=0x00 \
    --application Chutchit_dongle-nrf52840dongle_nrf52840-zmk.hex \
    --application-version 1 \
    chutchit_dongle.zip

# Flash qua USB serial
# macOS:
nrfutil nrf5sdk-tools dfu usb-serial -pkg chutchit_dongle.zip -p /dev/cu.usbmodem*
# Linux:
nrfutil nrf5sdk-tools dfu usb-serial -pkg chutchit_dongle.zip -p /dev/ttyACM0
# Windows:
nrfutil nrf5sdk-tools dfu usb-serial -pkg chutchit_dongle.zip -p COM3
```

> Nếu build xuất ra file `.uf2`: kéo-thả thẳng vào ổ `MDBT50Q` xuất hiện khi
> dongle ở DFU mode — không cần dòng lệnh.

---

## 8. Cài đặt radio — bắt buộc khớp nhau

Các giá trị này **phải giống hệt nhau** trong `Chutchit.conf` và
`Chutchit_dongle.conf`. Chỉ cần lệch 1 thứ là mất kết nối hoàn toàn.

| Kconfig | Mặc định | Ý nghĩa |
|---|---|---|
| `ZMK_2G4_RF_CHANNEL` | `76` | Tần số = 2400 + 76 = **2476 MHz** |
| `ZMK_2G4_ADDR_BASE_0..3` | `0x50 0x48 0x44 0x53` | 4 bytes base address = "PHDS" |
| `ZMK_2G4_ADDR_PREFIX` | `0x4E` | Prefix byte = "N" → full pipe 0 address = "PHDSN" |
| `ZMK_2G4_AES_KEY` | `""` | Key AES-128, 32 ký tự hex = 16 bytes |

**Tạo key mã hóa ngẫu nhiên:**
```bash
openssl rand -hex 16
# Ví dụ output: a1b2c3d4e5f60718293a4b5c6d7e8f90
```

Đặt vào **cả hai** config:
```conf
CONFIG_ZMK_2G4_AES_KEY="a1b2c3d4e5f60718293a4b5c6d7e8f90"
```

---

## 9. Cơ chế chuyển đổi kết nối

| Tình huống | Kết nối hoạt động | Cách kích hoạt |
|---|---|---|
| Cắm cáp USB vào bàn phím | USB | Tự động, ưu tiên tuyệt đối |
| Rút cáp USB | Wireless preference cuối | Tự động |
| Boot lần đầu / chưa có preference | BLE | Mặc định trong code |
| Nhấn `OUT_2G4` | 2.4 GHz (dongle) | Lưu vào flash, giữ qua reset |
| Nhấn `OUT_BLE` | BLE | Lưu vào flash |
| Nhấn `OUT_TOG` | Toggle BLE ↔ 2.4 GHz | Lưu vào flash |

**Chi tiết kỹ thuật:**
- Chuyển BLE ↔ 2.4 GHz: radio phải tắt hẳn rồi mới khởi động lại → mất ~50ms
- Có watchdog 1 giây: nếu quá trình chuyển radio bị treo → tự động reboot
- Keepalive mỗi 250ms → dongle timeout sau 500ms im lặng → nhả tất cả phím
- Kết nối bị coi là mất sau 40 lần gửi thất bại liên tiếp (~10 giây)

---

## 10. Debug & kiểm tra

**Kiểm tra kết nối radio:**
Bật `CONFIG_ZMK_USB_LOGGING=y` trên bàn phím (left half), kết nối USB, mở
serial terminal ở 115200 baud. Log sẽ hiển thị:

```
[INF] 2.4G transport started
[INF] 2.4G BOOT sent: reason=0x00000000 session=0xABCD1234 ret=0
[INF] 2.4G keyboard connected (rf_len=12, total=1)
```

**Kiểm tra dongle nhận được:**
Bật `CONFIG_ZMK_USB_LOGGING=y` trên dongle firmware, kiểm tra log từ dongle:
```
[INF] 2.4G dongle receiver started (ch=76)
[INF] Keyboard BOOT: reset_reason=0x00000000 session=0xABCD1234
[INF] 2.4G keyboard connected (rf_len=12, total=1)
```
