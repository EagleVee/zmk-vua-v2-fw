# Hướng Dẫn Triển Khai Dongle 2.4 GHz cho Chutchit

## Dongle là gì?

Dongle là một thiết bị nRF52840 nhỏ cắm trực tiếp vào cổng USB của máy tính.
Nó nhận tín hiệu phím từ bàn phím Chutchit qua sóng radio 2.4 GHz và xuất ra
cho máy tính như một bàn phím/chuột USB HID thông thường. Không cần cài driver.

---

## Phần cứng: Nordic nRF52840 USB Dongle (PCA10059)

Đây là phần cứng được khuyến nghị. Nó có **đầu cắm USB-A male** cắm thẳng vào
máy tính — không cần cáp.

```
┌─────────────────────────┐
│  Chip nRF52840          │ ██ USB-A ──► cắm thẳng vào PC
│  Anten PCB              │
│  Nút Reset              │
│  Đèn LED RGB            │
└─────────────────────────┘
     ~35 × 13 mm
```

- **Mua ở đâu:** Cửa hàng chính thức Nordic Semiconductor, Mouser, Digikey,
  hoặc Amazon (~10 USD)
- **Trang sản phẩm:** https://www.nordicsemi.com/Products/Development-hardware/nRF52840-Dongle
- **Tên board trong Zephyr:** `nrf52840dongle_nrf52840`

### Máy tính chỉ có cổng USB-C

PCA10059 có đầu USB-A. Nếu máy tính chỉ có cổng USB-C, dùng **adapter USB-A
sang USB-C thụ động** (không phải hub). Dongle vẫn cắm thẳng vào — adapter chỉ
thay đổi hình dạng đầu cắm.

Ngoài ra, bạn có thể thiết kế PCB tùy chỉnh với đầu cắm **USB-C male** dùng
chip nRF52840 — nhưng dùng dongle Nordic là cách nhanh nhất để bắt đầu.

---

## Cách hoạt động

```
Nửa trái Chutchit                   Dongle (PCA10059)
┌─────────────────────┐             ┌───────────────────┐
│ Quét ma trận phím   │  2.4 GHz   │ Chế độ ESB PRX    │
│ Xử lý keymap/layer  │ ─────────► │ Giải mã payload   │
│ Tạo HID report      │ sóng radio │ Chuyển tiếp USB   │──► PC
│ Mã hóa & gửi        │             └───────────────────┘
└─────────────────────┘
        ▲ BLE split
┌───────┴─────────────┐
│ Nửa phải + Num      │
└─────────────────────┘
```

Dongle **không chứa keymap của bạn**. Nó chỉ nhận HID report đã được xử lý sẵn
từ nửa trái và chuyển tiếp qua USB đến PC. Keymap chỉ nằm trong flash của nửa
trái — giống như chế độ USB và BLE.

---

## Các file shield cần tạo

Tạo thư mục `boards/shields/Chutchit_dongle/` trong repo `zmk-vua-v2-fw` với
các file sau:

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

# PHẢI khớp hoàn toàn với config của bàn phím
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
// Dongle không có ma trận phím. Toàn bộ input đến qua radio ESB.
```

---

## Thay đổi config bàn phím

### `config/west.yml` — trỏ sang fork

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

### `config/Chutchit.conf` — bật 2.4 GHz trên nửa trái

Thêm các dòng sau:

```conf
CONFIG_ZMK_2G4=y
CONFIG_ZMK_2G4_RF_CHANNEL=76
CONFIG_ZMK_2G4_TX_POWER=4
CONFIG_ZMK_2G4_AES_KEY=""
CONFIG_ZMK_ESB=y
```

### `config/Chutchit.keymap` — thêm phím chuyển kết nối

Thêm vào một layer (ví dụ Layer 1 / LOWER):

```c
&out OUT_USB    // chuyển sang USB
&out OUT_BLE    // chuyển sang BLE
&out OUT_2G4    // chuyển sang dongle 2.4 GHz
&out OUT_TOG    // chuyển đổi qua lại BLE ↔ 2.4 GHz
```

### `build.yaml` — thêm target build cho dongle

```yaml
  - board: nrf52840dongle_nrf52840
    shield: Chutchit_dongle
```

---

## Build firmware

### GitHub Actions (khuyến nghị)

Push lên repo. GitHub Actions sẽ tự build tất cả các target kể cả dongle.
Tải file `Chutchit_dongle-nrf52840dongle_nrf52840-zmk.uf2` từ Artifacts.

### Build local

```bash
cd app
west build -b nrf52840dongle_nrf52840 -- -DSHIELD=Chutchit_dongle
```

---

## Nạp firmware vào dongle

PCA10059 đã có sẵn bootloader USB của Nordic. **Không cần thiết bị debug (JLink)**.

### Bước 1 — Cài nrfutil
```bash
pip install nrfutil
```

### Bước 2 — Vào chế độ DFU
Cắm dongle vào USB, **nhấn và giữ nút reset** khoảng 1–2 giây. Đèn đỏ sẽ nhấp
nháy chậm. Dongle đã ở chế độ DFU và xuất hiện như một USB device.

### Bước 3 — Flash

Tạo gói DFU từ file `.hex` và flash:

```bash
nrfutil dfu genpkg --dev-type 0x0052 --application zmk.hex dfu_package.zip
nrfutil dfu usb-serial -pkg dfu_package.zip -p /dev/ttyACM0
```

Thay `/dev/ttyACM0` bằng port thực tế. Trên macOS sẽ là `/dev/cu.usbmodem*`.
Trên Windows sẽ là `COMx`.

> **Mẹo:** Nếu ZMK build xuất ra file `.uf2`, bạn có thể kéo-thả thẳng vào ổ
> `MDBT50Q` xuất hiện khi dongle ở chế độ DFU — không cần dòng lệnh.

---

## Thứ tự nạp firmware cho toàn bộ thiết bị

Nạp `settings_reset` trước cho **nửa trái** để xóa các cài đặt BLE/transport
cũ, sau đó nạp firmware mới:

```
1. Nạp settings_reset  → nửa trái  (xóa cài đặt cũ)
2. Nạp Chutchit_left   → nửa trái
3. Nạp Chutchit_right  → nửa phải  (không cần settings_reset)
4. Nạp Chutchit_num    → nửa num   (không cần settings_reset)
5. Nạp Chutchit_dongle → dongle PCA10059
```

---

## Cài đặt radio — phải khớp nhau trên cả hai bên

Các giá trị Kconfig này phải **giống hệt nhau** trong `Chutchit.conf` (bàn phím)
và `Chutchit_dongle.conf` (dongle). Nếu khác nhau, kết nối radio sẽ không hoạt
động.

| Cài đặt | Mặc định | Ghi chú |
|---|---|---|
| `CONFIG_ZMK_2G4_RF_CHANNEL` | `76` | 2400 + 76 = 2476 MHz |
| `CONFIG_ZMK_2G4_ADDR_BASE_0..3` | `0x50 0x48 0x44 0x53` | "PHDS" |
| `CONFIG_ZMK_2G4_ADDR_PREFIX` | `0x4E` | "N" → pipe 0 = "PHDSN" |
| `CONFIG_ZMK_2G4_AES_KEY` | `""` | Đặt key 32 ký tự hex trước khi dùng thật |

---

## Mã hóa AES (tùy chọn nhưng nên dùng)

Trong quá trình phát triển, để `CONFIG_ZMK_2G4_AES_KEY=""` (không mã hóa) để
dễ debug.

Trước khi dùng hàng ngày, đặt key 16 byte biểu diễn dưới dạng 32 ký tự hex —
**giá trị giống nhau trên bàn phím và dongle**:

```conf
# Ví dụ — hãy tạo giá trị ngẫu nhiên của riêng bạn
CONFIG_ZMK_2G4_AES_KEY="a1b2c3d4e5f60718293a4b5c6d7e8f90"
```

Tạo key ngẫu nhiên:
```bash
openssl rand -hex 16
```

---

## Cơ chế chuyển đổi kết nối

| Tình huống | Kết nối được dùng |
|---|---|
| Cắm cáp USB vào bàn phím | USB (tự động, không cần nhấn phím) |
| Rút cáp USB | Wireless preference được lưu lần cuối |
| Khởi động lần đầu | BLE (mặc định) |
| Nhấn `OUT_2G4` | 2.4 GHz qua dongle, lưu vào flash |
| Nhấn `OUT_BLE` | BLE, lưu vào flash |
| Nhấn `OUT_TOG` | Chuyển đổi qua lại BLE ↔ 2.4 GHz |

Chuyển đổi giữa BLE và 2.4 GHz mất ~50 ms (thời gian ổn định radio). USB
override là tức thời. Wireless preference được giữ lại sau khi tắt nguồn.

---

## Keepalive & timeout

- Bàn phím gửi gói keepalive mỗi **250 ms** khi không có phím nào được nhấn
- Dongle nhả tất cả phím sau **500 ms** im lặng (bảo vệ khi mất gói tin)
- Kết nối được coi là mất sau **40 lần gửi thất bại liên tiếp** (~10 giây)
