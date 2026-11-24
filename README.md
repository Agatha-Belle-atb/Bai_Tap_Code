# Bai_Tap_Code
Nới chứa lời giải Bài Tập đã code
# Đây là sách về Thư Viện VL53

# HƯỚNG DẪN SỬ DỤNG
## Thư viện VL53L0X đa cảm biến cho STM32 HAL (`tof_multi` + API ST v1.0.4)

> **Dành cho ai?** Người muốn **tải về và dùng ngay**: nối dây, thêm file, gọi vài hàm là có khoảng cách.
> Mỗi hàm có: **công dụng → cú pháp → tham số → kết quả → ví dụ ngắn → lưu ý**.
> Muốn hiểu sâu (lý thuyết, bài tập, lời giải) thì đọc bộ sách trong [`docs/`](docs/README.md).
>
> ⚠️ Thư viện đã được kiểm thử trên **bộ giả lập** (biên dịch GCC, chạy với nhiều cảm biến ảo trên cùng bus I2C). **Chưa chạy trên mạch thật và Keil thật.** Nếu thấy kết quả khác hướng dẫn, xem [mục 12](#12-sự-cố-thường-gặp).

---

## Mục lục

1. [Thư viện làm được gì](#1-thư-viện-làm-được-gì)
2. [Bắt đầu nhanh trong 5 bước](#2-bắt-đầu-nhanh-trong-5-bước)
3. [Các file trong gói](#3-các-file-trong-gói)
4. [Khai báo cảm biến: `TOF_Sensor` và `TOF_Bus`](#4-khai-báo-cảm-biến-tof_sensor-và-tof_bus)
5. [Bảng tóm tắt tất cả các hàm](#5-bảng-tóm-tắt-tất-cả-các-hàm)
6. [Các hàm chính của `tof_multi`](#6-các-hàm-chính-của-tof_multi)
7. [Đọc dữ liệu trong `TOF_Sensor`](#7-đọc-dữ-liệu-trong-tof_sensor)
8. [Chương trình chạy thử có sẵn](#8-chương-trình-chạy-thử-có-sẵn)
9. [Các hàm nâng cao của ST](#9-các-hàm-nâng-cao-của-st)
10. [Các hàm cấp thấp (platform)](#10-các-hàm-cấp-thấp-platform)
11. [Tuỳ chỉnh bằng macro](#11-tuỳ-chỉnh-bằng-macro)
12. [Sự cố thường gặp](#12-sự-cố-thường-gặp)
13. [Bảng tra mã lỗi và trạng thái](#13-bảng-tra-mã-lỗi-và-trạng-thái)
14. [Nên và không nên](#14-nên-và-không-nên)

---

## 1. Thư viện làm được gì

| Tính năng | Chi tiết |
|---|---|
| Nhiều cảm biến trên **một** bus I2C | 1–32 cảm biến, mỗi con một chân XSHUT. Thư viện tự gán địa chỉ |
| Đo **không chặn CPU** | Gọi `TOF_Poll()` trong `while(1)`; mỗi lần gọi chặn tối đa ~3 ms |
| Hai chế độ đo | **Song song** (~30 lần/s mỗi con) hoặc **tuần tự** (chống nhiễu giữa các cảm biến đặt sát nhau) |
| Tự phát hiện lỗi | Cảm biến mất dây, treo, không có dữ liệu → trạng thái `ERROR` kèm mã lỗi |
| Tự phục hồi | `TOF_Restart()` một cảm biến; `TOF_I2C_Recover()` gỡ bus bị treo |
| Hiệu chuẩn offset | `TOF_CalibrateOffset()`; lưu kết quả để dùng lại lần sau |
| Dùng được mọi hàm của ST | Mỗi cảm biến có sẵn `tof[i].dev` để gọi API gốc của ST |

**Yêu cầu:**
- STM32 dùng HAL. Hướng dẫn viết cho STM32F103C8T6; dòng khác chỉ cần đổi một macro.
- Module VL53L0X (GY-530 / GY-VL53L0XV2).
- Keil MDK hoặc STM32CubeIDE; trình biên dịch ở chế độ **C99**.

---

## 2. Bắt đầu nhanh trong 5 bước

### Bước 1 — Nối dây (2 cảm biến)

| Module VL53L0X | TOF1 | TOF2 | STM32F103 (Blue Pill) |
|---|---|---|---|
| VIN | ● | ● | 3.3V |
| GND | ● | ● | GND |
| SCL | ● | ● | PB6 (nối chung) |
| SDA | ● | ● | PB7 (nối chung) |
| XSHUT | ● | | PA0 |
| XSHUT | | ● | PA1 |
| GPIO1 (không bắt buộc) | ● | | PB0 |
| GPIO1 (không bắt buộc) | | ● | PB1 |

**Bóc miếng dán bảo vệ** màu vàng/cam trên mắt cảm biến.

### Bước 2 — STM32CubeMX

| Mục | Thiết lập |
|---|---|
| SYS | Debug = **Serial Wire** |
| RCC / Clock | HSE Crystal → PLL ×9 → **72 MHz** |
| I2C1 | **Standard mode 100 kHz** |
| PA0, PA1 | GPIO_Output, **Open Drain**, mức **Low**, label `XSHUT1`, `XSHUT2` |
| PB0, PB1 | GPIO_Input, **Pull-up**, label `TOF1_INT`, `TOF2_INT` |
| Project Manager | Toolchain **MDK-ARM**, Minimum Stack **0x800** |

### Bước 3 — Thêm thư viện vào Keil

1. Chép thư mục `vl53l0x_stm32` vào thư mục project, đổi tên thành `VL53L0X`.
2. Tạo group `VL53L0X`, thêm các file:
   - `st_core/vl53l0x_api.c`, `vl53l0x_api_core.c`, `vl53l0x_api_calibration.c`, `vl53l0x_api_strings.c`, `vl53l0x_api_ranging.c`
   - `platform/vl53l0x_platform.c`
   - `tof/tof_multi.c`
3. *Options for Target → C/C++*:
   - **Include Paths:** `../VL53L0X/st_core;../VL53L0X/platform;../VL53L0X/tof;../VL53L0X/app`
   - **Define:** `USE_HAL_DRIVER,STM32F103xB,USE_I2C_2V8`
   - Bật **C99** (AC5: tick *C99 Mode*; AC6: Language C = *c99*)

### Bước 4 — Viết code

Tạo file `my_tof.c`:
```c
#include "main.h"
#include "tof_multi.h"

extern I2C_HandleTypeDef hi2c1;

TOF_Sensor tof[2] = {
    { .xshut_port = XSHUT1_GPIO_Port, .xshut_pin = XSHUT1_Pin,
      .gpio1_port = TOF1_INT_GPIO_Port, .gpio1_pin = TOF1_INT_Pin, .addr = 0x54 },
    { .xshut_port = XSHUT2_GPIO_Port, .xshut_pin = XSHUT2_Pin,
      .gpio1_port = TOF2_INT_GPIO_Port, .gpio1_pin = TOF2_INT_Pin, .addr = 0x56 },
};
TOF_Bus bus = { .hi2c = &hi2c1, .sensor = tof, .count = 2, .mode = TOF_MODE_PARALLEL };

volatile uint16_t distance_mm[2];    /* xem trong Watch cua Keil */

void MyTOF_Init(void)
{
    TOF_Init(&bus);
}

void MyTOF_Loop(void)
{
    uint16_t mm;
    uint8_t  i;

    TOF_Poll(&bus);
    for (i = 0; i < 2; i++)
        if (TOF_GetRange(&bus, i, 100, &mm))
            distance_mm[i] = mm;
}
```

Trong `main.c`:
```c
/* USER CODE BEGIN 0 */
void MyTOF_Init(void);
void MyTOF_Loop(void);
/* USER CODE END 0 */

  /* USER CODE BEGIN 2 */
  MyTOF_Init();
  /* USER CODE END 2 */

  while (1)
  {
    /* USER CODE END WHILE */
    /* USER CODE BEGIN 3 */
    MyTOF_Loop();
  }
```

### Bước 5 — Chạy và xem kết quả

1. F7 (build) → Ctrl+F5 (debug) → F5 (run).
2. Mở *View → Watch 1*, gõ `distance_mm`, bật *View → Periodic Window Update*.
3. Đưa tay trước cảm biến: giá trị thay đổi theo khoảng cách.

> 💡 Không thấy gì? Dùng chương trình chạy thử có sẵn (mục 8) để biết lỗi ở đâu.

---

## 3. Các file trong gói

```
vl53l0x_stm32/
├── st_core/    Lõi đo của ST: THÊM toàn bộ file .c
├── platform/   Nối với HAL: THÊM vl53l0x_platform.c
├── tof/        tof_multi.c: THÊM.   tof_example.c: KHÔNG thêm (ví dụ tham khảo)
├── app/        Chương trình chạy thử: thêm MỘT trong hai file (mục 8)
├── docs/       Bộ sách học
├── README.md
└── HUONG_DAN_SU_DUNG.md   ← file này
```

| Không được thêm vào project | Lý do |
|---|---|
| `vl53l0x_i2c_platform.c`, `vl53l0x_i2c_win_serial_comms.c`, `vl53l0x_platform_log.c` (bản gốc ST) | Code cho PC Windows |
| Cả hai file `app_tof_test.c` và `app_tof_dbg.c` cùng lúc | Trùng tên biến; `app_tof_test.c` cần UART |
| `#include "vl53l0x_tuning.h"` trong file của bạn | Gây lỗi link "multiple definition" |

---

## 4. Khai báo cảm biến: `TOF_Sensor` và `TOF_Bus`

### 4.1. `TOF_Sensor`: một cảm biến

**Bạn điền:**

| Trường | Bắt buộc | Ý nghĩa | Ví dụ |
|---|---|---|---|
| `xshut_port`, `xshut_pin` | ✔ | Chân XSHUT | `GPIOA`, `GPIO_PIN_0` |
| `gpio1_port`, `gpio1_pin` | | Chân GPIO1. Bỏ trống (= `NULL`) thì thư viện hỏi qua I2C | `GPIOB`, `GPIO_PIN_0` |
| `addr` | ✔ | Địa chỉ mới, **8-bit**: chẵn, 0x10–0xEE, ≠ 0x52, không trùng | `0x54` |
| `calib` | | Dữ liệu hiệu chuẩn đã lưu (mục 7.6) | |

**Ví dụ: cảm biến không nối GPIO1**
```c
TOF_Sensor tof_front = { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_2, .addr = 0x58 };
```

> 💡 Địa chỉ gợi ý: 0x54, 0x56, 0x58, 0x5A… Tránh **0x4E** và **0x7E** nếu có LCD PCF8574 trên cùng bus.

### 4.2. `TOF_Bus`: một nhóm cảm biến trên một bus I2C

| Trường | Mặc định khi để 0 | Ý nghĩa |
|---|---|---|
| `hi2c` | — (bắt buộc) | `&hi2c1`, `&hi2c2`… |
| `sensor` | — (bắt buộc) | Tên mảng `TOF_Sensor` |
| `count` | — (bắt buộc) | Số cảm biến (1–32) |
| `mode` | `TOF_MODE_PARALLEL` | `TOF_MODE_PARALLEL` hoặc `TOF_MODE_SEQUENTIAL` |
| `budget_us` | 33000 | Thời gian một lần đo (µs): 20000 nhanh, 200000 chính xác |
| `poll_ms` | 10 | Chu kỳ hỏi qua I2C khi không có GPIO1 |
| `sigma_check` | 0 (tắt) | 1 = bật lọc sigma của ST (chậm hơn ~4 lần) |

**Ví dụ: đầy đủ tuỳ chọn**
```c
TOF_Bus bus_full = {
    .hi2c        = &hi2c1,
    .sensor      = tof,
    .count       = 2,
    .mode        = TOF_MODE_SEQUENTIAL,  /* 2 cam bien dat sat nhau, cung huong */
    .budget_us   = 50000,                /* 50 ms/lan do: on dinh hon */
    .poll_ms     = 10,
    .sigma_check = 0,
};
```

**Chọn chế độ nào?**

| Tình huống | Chế độ |
|---|---|
| Các cảm biến nhìn các hướng khác nhau (lệch nhau > 25°) | `TOF_MODE_PARALLEL` (nhanh nhất) |
| Các cảm biến đặt sát nhau, nhìn cùng hướng | `TOF_MODE_SEQUENTIAL` (không nhiễu, chậm hơn N lần) |
| Có cả hai nhóm | Hai `TOF_Bus` trên cùng `hi2c1`: một song song, một tuần tự. Địa chỉ giữa hai nhóm phải khác nhau |

**Ví dụ: hai nhóm trên cùng một I2C**
```c
TOF_Sensor front[2] = { { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_0, .addr = 0x54 },
                        { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_1, .addr = 0x56 } };
TOF_Sensor side[2]  = { { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_2, .addr = 0x58 },
                        { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_3, .addr = 0x5A } };
TOF_Bus bus_front = { .hi2c = &hi2c1, .sensor = front, .count = 2, .mode = TOF_MODE_SEQUENTIAL };
TOF_Bus bus_side  = { .hi2c = &hi2c1, .sensor = side,  .count = 2, .mode = TOF_MODE_PARALLEL };
/* TOF_Init(&bus_front); TOF_Init(&bus_side);  -- trong while(1): TOF_Poll ca hai */
```

---

## 5. Bảng tóm tắt tất cả các hàm

### Hàm của thư viện (`tof_multi.h`): đủ cho 90% trường hợp

| Hàm | Làm gì | Gọi khi nào | Chặn CPU |
|---|---|---|---|
| `TOF_Init(&bus)` | Gán địa chỉ, khởi tạo, bắt đầu đo | 1 lần lúc khởi động | ~0,2 s/cảm biến |
| `TOF_Poll(&bus)` | Đọc số đo mới; trả bitmask | **Mỗi vòng `while(1)`** | ≤ ~3 ms |
| `TOF_GetRange(&bus, i, age, &mm)` | Lấy khoảng cách hợp lệ của cảm biến i | Bất cứ lúc nào | Không |
| `TOF_Restart(&bus, i)` | Khởi động lại riêng cảm biến i | Khi `state == ERROR` | ~0,2 s |
| `TOF_CalibrateOffset(&bus, i, mm)` | Hiệu chuẩn offset | Khi lắp đặt / bấm nút | ~2 s |
| `TOF_I2C_Recover(&hi2c, port, scl, sda)` | Gỡ bus I2C bị treo | Khi tất cả cảm biến cùng lỗi | < 1 ms |

### Chương trình chạy thử (`app/`)

| Hàm | Làm gì |
|---|---|
| `App_TOF_Test_Init()` / `App_TOF_Test_Loop()` | Chạy thử 2 cảm biến, in kết quả qua UART |
| `App_TOF_Dbg_Init()` / `App_TOF_Dbg_Loop()` | Chạy thử 2 cảm biến, xem bằng Watch (biến `g_tof`) |
| `App_TOF_Dbg_OnError(idx)` | Đặt breakpoint ở đây để dừng khi cảm biến lỗi |

### Hàm nâng cao của ST (gọi trên `&tof[i].dev`)

| Nhóm | Hàm |
|---|---|
| Thời gian đo | `VL53L0X_SetMeasurementTimingBudgetMicroSeconds`, `Get…` |
| Tầm đo | `VL53L0X_SetVcselPulsePeriod`, `GetVcselPulsePeriod` |
| Ngưỡng chất lượng | `VL53L0X_SetLimitCheckValue`, `GetLimitCheckValue`, `SetLimitCheckEnable`, `GetLimitCheckEnable`, `GetLimitCheckCurrent` |
| Dữ liệu chi tiết | `VL53L0X_GetRangingMeasurementData` |
| Offset / crosstalk | `VL53L0X_Get/SetOffsetCalibrationDataMicroMeter`, `PerformXTalkCalibration`, `Set/GetXTalkCompensationRateMegaCps`, `Set/GetXTalkCompensationEnable` |
| Điều khiển đo | `VL53L0X_StopMeasurement`, `StartMeasurement`, `ClearInterruptMask`, `GetDeviceMode` |
| Thông tin | `VL53L0X_GetVersion`, `GetDeviceInfo`, `GetProductRevision`, `GetReferenceSpads`, `GetRefCalibration`, `GetPalState` |
| Chuỗi mô tả | `VL53L0X_GetPalErrorString`, `GetRangeStatusString` |
| Ngắt theo ngưỡng | `VL53L0X_SetGpioConfig`, `SetInterruptThresholds` (có cảnh báo) |

### Hàm cấp thấp (platform)

`VL53L0X_RdByte`, `RdWord`, `RdDWord`, `WrByte`, `WrWord`, `WrDWord`, `ReadMulti`, `WriteMulti`, `UpdateByte`, `PollingDelay`.

---

## 6. Các hàm chính của `tof_multi`

### 6.1. `TOF_Init` — khởi tạo tất cả cảm biến

```c
uint8_t TOF_Init(TOF_Bus *bus);
```

| | |
|---|---|
| **Công dụng** | Tắt hết → bật từng con → đổi địa chỉ → khởi tạo theo chuẩn ST → bắt đầu đo |
| **Tham số** | `bus`: đã điền `hi2c`, `sensor`, `count`, `mode` |
| **Trả về** | Số cảm biến chạy được (0 … `count`) |
| **Gọi khi** | **Một lần**, sau `MX_GPIO_Init()` và `MX_I2C1_Init()` |
| **Chặn CPU** | ~0,2–0,3 s mỗi cảm biến |

**Ví dụ:**
```c
uint8_t n = TOF_Init(&bus);          /* goi 1 lan, sau MX_I2C1_Init() */
if (n != bus.count) {
    /* co cam bien loi: xem tof[i].state va tof[i].last_error */
}
```

**Lưu ý:**
- Gọi lại `TOF_Init` được, ví dụ khi đổi chế độ đo; mọi cảm biến bị khởi tạo lại. Dữ liệu `calib` được giữ.
- Cảm biến lỗi khi khởi tạo bị giữ tắt (XSHUT thấp) để không làm hỏng các con khác.

---

### 6.2. `TOF_Poll` — đọc số đo mới (gọi liên tục)

```c
uint32_t TOF_Poll(TOF_Bus *bus);
```

| | |
|---|---|
| **Công dụng** | Kiểm tra cảm biến nào đã đo xong và đọc kết quả; phát hiện lỗi |
| **Trả về** | **Bitmask**: bit i = 1 nghĩa là cảm biến i vừa có số đo mới |
| **Gọi khi** | **Mỗi vòng `while(1)`, đúng một lần** |
| **Chặn CPU** | Thường gần 0; tối đa ~3 ms (song song) / ~6 ms (tuần tự) |

**Ví dụ:**
```c
uint32_t upd = TOF_Poll(&bus);       /* goi MOT lan moi vong while(1) */
if (upd & (1UL << 0)) { /* cam bien 0 co so do moi */ }
if (upd & (1UL << 1)) { /* cam bien 1 co so do moi */ }
```

**Lưu ý:**
- Bit bằng 1 chỉ có nghĩa là **có số đo mới**, kể cả số đo **không hợp lệ** (không có vật). Dùng `TOF_GetRange`, hoặc kiểm tra `tof[i].range_status == 0`.
- Bitmask: 0x01 = cảm biến 0, 0x02 = cảm biến 1, 0x03 = cả hai. **Không** viết `if (upd == 1)` để kiểm tra cảm biến 1.
- **Không** gọi trong hàm ngắt.

---

### 6.3. `TOF_GetRange` — lấy khoảng cách tin cậy

```c
uint8_t TOF_GetRange(const TOF_Bus *bus, uint8_t i, uint32_t max_age_ms, uint16_t *mm);
```

| | |
|---|---|
| **Công dụng** | Trả về khoảng cách của cảm biến `i` **chỉ khi** số đo hợp lệ và đủ mới |
| **Tham số** | `i`: chỉ số cảm biến (0, 1, …); `max_age_ms`: tuổi tối đa (0 = không kiểm tra); `mm`: nơi nhận kết quả |
| **Trả về** | 1 = có số đo tốt (đã ghi `*mm`); 0 = không có (`*mm` giữ nguyên) |
| **Chặn CPU** | Không (không dùng I2C) |

**Ví dụ:**
```c
uint16_t mm;
if (TOF_GetRange(&bus, 0, 100, &mm)) {
    /* mm hop le, moi hon 100 ms */
} else {
    /* khong co vat, cam bien loi, hoac so do cu */
}
```

**Chọn `max_age_ms`:**

| Chế độ | Gợi ý |
|---|---|
| Song song, budget 33 ms | 100 |
| Tuần tự với N cảm biến | 100 × N |
| Budget 200 ms (chính xác cao) | 500 |

---

### 6.4. `TOF_Restart` — khởi động lại một cảm biến

```c
VL53L0X_Error TOF_Restart(TOF_Bus *bus, uint8_t i);
```

| | |
|---|---|
| **Công dụng** | Reset cảm biến `i` bằng XSHUT, gán lại địa chỉ, khởi tạo lại. **Các cảm biến khác vẫn chạy** |
| **Trả về** | `VL53L0X_ERROR_NONE` (0) nếu thành công; mã lỗi nếu không |
| **Chặn CPU** | ~0,2 s nếu cảm biến có mặt; vài ms nếu không trả lời |

**Ví dụ: tự phục hồi mỗi giây**
```c
static uint32_t t_check = 0;
if (HAL_GetTick() - t_check >= 1000U) {
    t_check = HAL_GetTick();
    if (tof[1].state == TOF_STATE_ERROR) {
        VL53L0X_Error e = TOF_Restart(&bus, 1);   /* chan CPU ~0,2 s */
        if (e != VL53L0X_ERROR_NONE) { /* van loi: kiem tra day, nguon */ }
    }
}
```

**Lưu ý:**
- Mọi cấu hình bạn tự đặt bằng hàm ST (mục 9) **bị mất** sau restart: phải đặt lại.
- Đừng gọi trong mỗi vòng lặp; mỗi giây một lần là đủ.

---

### 6.5. `TOF_CalibrateOffset` — hiệu chuẩn sai số cố định

```c
VL53L0X_Error TOF_CalibrateOffset(TOF_Bus *bus, uint8_t i, uint16_t target_mm);
```

| | |
|---|---|
| **Công dụng** | Đo 50 lần một vật ở khoảng cách biết trước, tính offset, áp dụng ngay |
| **Tham số** | `target_mm`: khoảng cách thật từ mặt cảm biến tới tấm trắng (nên dùng 100) |
| **Trả về** | 0 = thành công. −6 = không đo được (không có vật). −30 = cảm biến chưa chạy. −4 = tham số sai |
| **Kết quả** | `tof[i].calib.offset_um` (µm) và `tof[i].calib.valid = 1` |
| **Chặn CPU** | **~2 s** |

**Ví dụ:**
```c
/* Dat tam giay trang cach TOF1 dung 100 mm roi goi: */
if (TOF_CalibrateOffset(&bus, 0, 100) == VL53L0X_ERROR_NONE) {
    int32_t off = tof[0].calib.offset_um;     /* vd -7000 = chip doc xa hon 7 mm */
    (void)off;                                /* luu tof[0].calib vao flash de dung lan sau */
}
```

**Lưu ý:**
- Tấm trắng (giấy A4) đặt vuông góc, không có vật khác trong vùng nhìn, tránh nắng.
- Kết quả chỉ nằm trong RAM. Muốn giữ sau khi tắt nguồn, lưu `tof[i].calib` vào flash (ví dụ đầy đủ trong [docs Q3 VD8](docs/03_ViDu_GiaiThich.md)).

---

### 6.6. `TOF_I2C_Recover` — gỡ bus I2C bị treo

```c
void TOF_I2C_Recover(I2C_HandleTypeDef *hi2c, GPIO_TypeDef *port, uint16_t scl_pin, uint16_t sda_pin);
```

| | |
|---|---|
| **Công dụng** | Khi một thiết bị giữ SDA thấp (thường gặp sau reset MCU giữa chừng): tạo 9 xung SCL + STOP, rồi khởi tạo lại I2C |
| **Tham số** | Bus và chân SCL, SDA (cùng port). Blue Pill I2C1: `GPIOB`, `GPIO_PIN_6`, `GPIO_PIN_7` |
| **Chặn CPU** | < 1 ms |

**Ví dụ:**
```c
TOF_I2C_Recover(&hi2c1, GPIOB, GPIO_PIN_6, GPIO_PIN_7);   /* SCL = PB6, SDA = PB7 */
(void)TOF_Restart(&bus, 0);
(void)TOF_Restart(&bus, 1);
```

**Lưu ý:** hàm ảnh hưởng **mọi thiết bị** trên bus (LCD…). Chỉ gọi khi tất cả cảm biến cùng lỗi.

---

## 7. Đọc dữ liệu trong `TOF_Sensor`

Ngoài các hàm, bạn đọc trực tiếp các trường của `tof[i]`. **Chỉ đọc, không ghi** (trừ `calib` trước `TOF_Init`).

### 7.1. `state` — cảm biến có đang chạy không

| Giá trị | Ý nghĩa |
|---|---|
| `TOF_STATE_OFF` (0) | Chưa khởi tạo / đang restart |
| `TOF_STATE_RUNNING` (1) | Đang đo bình thường |
| `TOF_STATE_ERROR` (2) | Lỗi, xem `last_error` |

```c
if (tof[0].state != TOF_STATE_RUNNING) {
    /* bao loi cam bien 1 (bat LED, hien len LCD...) */
}
```

### 7.2. `last_error` — mã lỗi gần nhất

```c
switch (tof[1].last_error) {
case VL53L0X_ERROR_NONE:              /* OK */                                   break;
case VL53L0X_ERROR_CONTROL_INTERFACE: /* -20: khong tra loi I2C: day, nguon */    break;
case VL53L0X_ERROR_TIME_OUT:          /* -7: khong co so do: day GPIO1? */         break;
case VL53L0X_ERROR_INVALID_PARAMS:    /* -4: dia chi cau hinh sai */               break;
default:                                                                         break;
}
```

### 7.3. `range_mm` và `range_status` — khoảng cách và chất lượng

| `range_status` | Ý nghĩa | Dùng `range_mm`? |
|---|---|---|
| 0 | Hợp lệ | ✔ |
| 2 | Tín hiệu yếu / không có vật | ✘ |
| 3 | Vật quá gần | ✘ |
| 4 | Ngoài tầm | ✘ |
| 1, 5 | Không chắc chắn / lỗi phần cứng | ✘ |
| 255 | Chưa có số đo | ✘ |

```c
uint32_t upd = TOF_Poll(&bus);
if ((upd & 1UL) && tof[0].range_status == 0) {
    uint16_t d = tof[0].range_mm;            /* khoang cach (mm) */
    (void)d;
} else if ((upd & 1UL) && (tof[0].range_status == 2 || tof[0].range_status == 4)) {
    /* duong thong, khong co vat trong tam */
}
```

### 7.4. `timestamp_ms` — số đo được lấy lúc nào

```c
uint32_t age_ms = HAL_GetTick() - tof[0].timestamp_ms;   /* so do nay cu bao nhieu ms */
(void)age_ms;
```

### 7.5. `err_count` — số lỗi liên tiếp

Đến 5 thì cảm biến chuyển sang `ERROR`. Đọc thành công một lần là về 0.
```c
uint8_t warn = (tof[0].err_count >= 2);      /* canh bao som truoc khi ERROR */
(void)warn;
```

### 7.6. `calib` — dữ liệu hiệu chuẩn (lưu / nạp)

| Trường | Ý nghĩa |
|---|---|
| `refSpadCount`, `isApertureSpads` | Hiệu chuẩn SPAD (thư viện tự đo lần đầu) |
| `offset_um` | Offset (µm) |
| `valid` | 1 = dùng các giá trị này khi `TOF_Init` / `TOF_Restart` |

**Nạp giá trị đã lưu trước khi khởi tạo** (giúp khởi động nhanh hơn):
```c
tof[0].calib.refSpadCount    = 5;      /* gia tri da luu truoc do */
tof[0].calib.isApertureSpads = 0;
tof[0].calib.offset_um       = -7000;
tof[0].calib.valid           = 1;      /* -> TOF_Init dung gia tri nay */
TOF_Init(&bus);
```

### 7.7. `dev` — dùng hàm gốc của ST

```c
VL53L0X_DEV d = &tof[0].dev;           /* truyen d vao moi ham VL53L0X_... (muc 9) */
(void)d;
```

---

## 8. Chương trình chạy thử có sẵn

Dùng khi lắp mạch lần đầu, để kiểm tra dây, địa chỉ và tốc độ đo. **Chỉ thêm một trong hai file** vào project.

### 8.1. `app_tof_dbg.c` — xem bằng Watch của Keil (không cần UART)

```c
void App_TOF_Dbg_Init(void);         /* USER CODE 2 */
void App_TOF_Dbg_Loop(void);         /* USER CODE 3, trong while(1) */
void App_TOF_Dbg_OnError(uint8_t idx);   /* dat breakpoint o day */
```

**Cách dùng:**
```c
#include "app_tof_dbg.h"
/* USER CODE BEGIN 2 */
App_TOF_Dbg_Init();
/* USER CODE BEGIN 3 (trong while) */
App_TOF_Dbg_Loop();
```

Trong Keil: Debug (Ctrl+F5) → Run (F5) → Watch 1, gõ **`g_tof`** → bật *Periodic Window Update*.

| Biến | Giá trị đúng |
|---|---|
| `g_tof.init_ok` | 2 |
| `g_tof.scan_before_n` | 0 (trước init không thấy thiết bị nào) |
| `g_tof.scan_after[0..1]` | 0x54, 0x56 (84, 86 ở dạng thập phân) |
| `g_tof.s[i].state` | 1 |
| `g_tof.s[i].range_mm`, `range_status` | khoảng cách, 0 |
| `g_tof.s[i].rate_hz` | ~30 |
| `g_tof.poll_us_max` | ≤ ~3000 |
| `g_tof.loop_count` | tăng liên tục |

**Ra lệnh từ debugger:** sửa giá trị trong Watch, ghi `cmd_arg` trước rồi mới ghi `cmd`:
```c
g_tof.cmd_arg = 1;     /* chon TOF2 */
g_tof.cmd     = 1;     /* 1 = restart; 2 = hieu chuan offset 100 mm; 3 = doi song song/tuan tu; 9 = xoa poll_us_max */
```
Kết quả ở `g_tof.cmd_result` (0 = OK).

### 8.2. `app_tof_test.c` — in qua UART

Cần USART1 (PA9, 115200) và tick *Use MicroLIB*.
```c
#include "app_tof_test.h"
/* USER CODE BEGIN 2 */
App_TOF_Test_Init();
/* USER CODE BEGIN 3 (trong while) */
App_TOF_Test_Loop();
```
Kết quả in 5 lần mỗi giây:
```text
TOF1:  123 mm OK     30 Hz   TOF2:  356 mm OK     30 Hz
```

---

## 9. Các hàm nâng cao của ST

Gọi trên `&tof[i].dev`. Mọi hàm trả `VL53L0X_Error` (0 = OK).

### ⚠️ Ba quy tắc khi dùng hàm ST cùng `tof_multi`

1. **Hàm `Set…` phải đặt trong mẫu Dừng → Đổi → Chạy lại** (chế độ song song):
```c
VL53L0X_DEV d = &tof[0].dev;
VL53L0X_StopMeasurement(d);
HAL_Delay(40);                          /* cho lan do dang chay ket thuc */
/* ... goi cac ham VL53L0X_Set...(d, ...) o day ... */
VL53L0X_ClearInterruptMask(d, 0);
VL53L0X_StartMeasurement(d);
```
   Ở chế độ tuần tự: chỉ đổi cấu hình, **không** gọi `StartMeasurement`, vì thư viện tự bật đo từng con.
2. Sau `TOF_Restart`, cấu hình tự đặt bị mất: **đặt lại**.
3. **Không gọi** các hàm sau khi dùng `tof_multi` (thư viện đã lo): `DataInit`, `StaticInit`, `SetDeviceAddress`, `PerformRefSpadManagement`, `PerformRefCalibration`, `PerformSingleRangingMeasurement`, `GetMeasurementDataReady`.

---

### 9.1. `VL53L0X_SetMeasurementTimingBudgetMicroSeconds` / `Get…` — thời gian một lần đo

```c
VL53L0X_Error VL53L0X_SetMeasurementTimingBudgetMicroSeconds(VL53L0X_DEV Dev, uint32_t us);
VL53L0X_Error VL53L0X_GetMeasurementTimingBudgetMicroSeconds(VL53L0X_DEV Dev, uint32_t *us);
```
Budget lớn thì chính xác hơn, chậm hơn: ×4 budget làm dao động giảm 2 lần. Khoảng 20000 → 200000 µs.

**Ví dụ:**
```c
uint32_t b;
VL53L0X_GetMeasurementTimingBudgetMicroSeconds(&tof[0].dev, &b);   /* b ~ 33000 (ST lam tron, vd 32908) */
/* Doi (trong mau Stop/Start): */
VL53L0X_SetMeasurementTimingBudgetMicroSeconds(&tof[0].dev, 200000);
bus.budget_us = 200000;    /* BAT BUOC: bao cho thu vien biet, neu khong se bao TIME_OUT */
```
💡 Muốn đổi budget cho **mọi** cảm biến, cách đơn giản nhất là sửa `bus.budget_us` rồi gọi lại `TOF_Init(&bus)`.

---

### 9.2. `VL53L0X_SetVcselPulsePeriod` / `Get…` — tăng tầm đo

```c
VL53L0X_Error VL53L0X_SetVcselPulsePeriod(VL53L0X_DEV Dev, VL53L0X_VcselPeriod type, uint8_t pclks);
VL53L0X_Error VL53L0X_GetVcselPulsePeriod(VL53L0X_DEV Dev, VL53L0X_VcselPeriod type, uint8_t *pclks);
```

| `type` | Giá trị hợp lệ | Mặc định | Tầm xa |
|---|---|---|---|
| `VL53L0X_VCSEL_PERIOD_PRE_RANGE` | 12, 14, 16, 18 | 14 | 18 |
| `VL53L0X_VCSEL_PERIOD_FINAL_RANGE` | 8, 10, 12, 14 | 10 | 14 |

**Ví dụ: chế độ tầm xa** (trong mẫu Stop/Start)
```c
VL53L0X_SetVcselPulsePeriod(&tof[0].dev, VL53L0X_VCSEL_PERIOD_PRE_RANGE, 18);
VL53L0X_SetVcselPulsePeriod(&tof[0].dev, VL53L0X_VCSEL_PERIOD_FINAL_RANGE, 14);
VL53L0X_SetLimitCheckValue(&tof[0].dev, VL53L0X_CHECKENABLE_SIGNAL_RATE_FINAL_RANGE,
                           (FixPoint1616_t)(0.10 * 65536));      /* ha nguong tin hieu */
```

---

### 9.3. `VL53L0X_SetLimitCheckValue` / `GetLimitCheckValue` — ngưỡng chất lượng

```c
VL53L0X_Error VL53L0X_SetLimitCheckValue(VL53L0X_DEV Dev, uint16_t id, FixPoint1616_t value);
VL53L0X_Error VL53L0X_GetLimitCheckValue(VL53L0X_DEV Dev, uint16_t id, FixPoint1616_t *value);
```

| `id` | Đơn vị | Mặc định | Ý nghĩa |
|---|---|---|---|
| `VL53L0X_CHECKENABLE_SIGNAL_RATE_FINAL_RANGE` | MCPS | 0,25 | Tín hiệu nhỏ hơn mức này thì status 2 |
| `VL53L0X_CHECKENABLE_SIGMA_FINAL_RANGE` | mm | 18 | Chỉ có tác dụng khi `sigma_check = 1` |

Giá trị dạng **16.16**: số thực × 65536.

**Ví dụ:**
```c
FixPoint1616_t v;
VL53L0X_GetLimitCheckValue(&tof[0].dev, VL53L0X_CHECKENABLE_SIGNAL_RATE_FINAL_RANGE, &v);
/* v = 16384 -> 16384 / 65536 = 0,25 MCPS */
```

---

### 9.4. `VL53L0X_SetLimitCheckEnable` / `GetLimitCheckEnable` — bật/tắt một kiểm tra

```c
VL53L0X_Error VL53L0X_SetLimitCheckEnable(VL53L0X_DEV Dev, uint16_t id, uint8_t enable);
VL53L0X_Error VL53L0X_GetLimitCheckEnable(VL53L0X_DEV Dev, uint16_t id, uint8_t *enable);
```
**Ví dụ: kiểm tra sigma đang bật hay tắt**
```c
uint8_t en;
VL53L0X_GetLimitCheckEnable(&tof[0].dev, VL53L0X_CHECKENABLE_SIGMA_FINAL_RANGE, &en);
/* en = 0 khi bus.sigma_check = 0 (mac dinh) */
```
💡 Muốn bật sigma cho cả bus thì đặt `bus.sigma_check = 1` trước `TOF_Init`, không cần gọi hàm này.

---

### 9.5. `VL53L0X_GetLimitCheckCurrent` — tín hiệu của lần đo vừa rồi (không tốn I2C)

```c
VL53L0X_Error VL53L0X_GetLimitCheckCurrent(VL53L0X_DEV Dev, uint16_t id, FixPoint1616_t *value);
```
Đọc giá trị **đã lưu** sau lần đọc gần nhất của thư viện: không truy cập I2C, gọi thoải mái.

**Ví dụ: cường độ phản xạ** (vật sáng màu, ở gần thì lớn)
```c
FixPoint1616_t sig;
VL53L0X_GetLimitCheckCurrent(&tof[0].dev, VL53L0X_CHECKENABLE_SIGNAL_RATE_FINAL_RANGE, &sig);
uint32_t sig_kcps = (uint32_t)(((uint64_t)sig * 1000U) >> 16);   /* MCPS (16.16) -> kcps */
(void)sig_kcps;
```

---

### 9.6. `VL53L0X_GetRangingMeasurementData` — dữ liệu chi tiết của lần đo

```c
VL53L0X_Error VL53L0X_GetRangingMeasurementData(VL53L0X_DEV Dev, VL53L0X_RangingMeasurementData_t *m);
```
Đọc lại 12 byte kết quả gần nhất từ chip (1 giao dịch I2C, ~1,4 ms). Gọi **sau** khi `TOF_Poll` báo có số đo mới.

| Trường | Đổi ra số thực |
|---|---|
| `RangeMilliMeter` | mm |
| `RangeStatus` | xem mục 13 |
| `SignalRateRtnMegaCps` | `/ 65536` → MCPS |
| `AmbientRateRtnMegaCps` | `/ 65536` → MCPS (ánh sáng môi trường) |
| `EffectiveSpadRtnCount` | `/ 256` → số SPAD |

**Ví dụ: đo ánh sáng môi trường** (nắng mạnh làm giảm tầm đo)
```c
VL53L0X_RangingMeasurementData_t m;
if (VL53L0X_GetRangingMeasurementData(&tof[0].dev, &m) == VL53L0X_ERROR_NONE) {
    uint32_t amb_kcps = (uint32_t)(((uint64_t)m.AmbientRateRtnMegaCps * 1000U) >> 16);
    (void)amb_kcps;
}
```

---

### 9.7. `VL53L0X_GetOffsetCalibrationDataMicroMeter` / `Set…` — đọc / đặt offset

```c
VL53L0X_Error VL53L0X_GetOffsetCalibrationDataMicroMeter(VL53L0X_DEV Dev, int32_t *um);
VL53L0X_Error VL53L0X_SetOffsetCalibrationDataMicroMeter(VL53L0X_DEV Dev, int32_t um);
```
Khoảng cho phép: −512000 … +511000 µm, bước 250 µm.

**Ví dụ: chỉnh tay** (chip đọc dư 5 mm thì offset = −5000)
```c
int32_t off;
VL53L0X_GetOffsetCalibrationDataMicroMeter(&tof[0].dev, &off);
VL53L0X_SetOffsetCalibrationDataMicroMeter(&tof[0].dev, off - 5000);   /* trong mau Stop/Start */
tof[0].calib.offset_um = off - 5000;                                    /* de TOF_Restart dung lai */
tof[0].calib.valid     = 1;
```

---

### 9.8. Hiệu chuẩn crosstalk — chỉ khi có kính/mica che phía trước

```c
VL53L0X_Error VL53L0X_PerformXTalkCalibration(VL53L0X_DEV Dev, FixPoint1616_t dist_mm, FixPoint1616_t *rate);
VL53L0X_Error VL53L0X_SetXTalkCompensationRateMegaCps(VL53L0X_DEV Dev, FixPoint1616_t rate);
VL53L0X_Error VL53L0X_GetXTalkCompensationRateMegaCps(VL53L0X_DEV Dev, FixPoint1616_t *rate);
VL53L0X_Error VL53L0X_SetXTalkCompensationEnable(VL53L0X_DEV Dev, uint8_t enable);
VL53L0X_Error VL53L0X_GetXTalkCompensationEnable(VL53L0X_DEV Dev, uint8_t *enable);
```
**Ví dụ:** đặt tấm **xám** cách 600 mm, phía sau tấm kính. Hàm chặn CPU ~2 s.
```c
FixPoint1616_t xtalk;
VL53L0X_StopMeasurement(&tof[0].dev);
HAL_Delay(40);
if (VL53L0X_PerformXTalkCalibration(&tof[0].dev, (FixPoint1616_t)600 << 16, &xtalk) == VL53L0X_ERROR_NONE) {
    /* luu xtalk; lan sau: SetXTalkCompensationRateMegaCps(d, xtalk) + SetXTalkCompensationEnable(d, 1) */
}
VL53L0X_SetDeviceMode(&tof[0].dev, VL53L0X_DEVICEMODE_CONTINUOUS_RANGING);   /* ham tren doi sang SINGLE */
VL53L0X_ClearInterruptMask(&tof[0].dev, 0);
VL53L0X_StartMeasurement(&tof[0].dev);
```

---

### 9.9. `VL53L0X_StopMeasurement` / `StartMeasurement` / `ClearInterruptMask` — điều khiển đo

```c
VL53L0X_Error VL53L0X_StopMeasurement(VL53L0X_DEV Dev);
VL53L0X_Error VL53L0X_StartMeasurement(VL53L0X_DEV Dev);
VL53L0X_Error VL53L0X_ClearInterruptMask(VL53L0X_DEV Dev, uint32_t mask);   /* mask = 0 */
```
**Ví dụ: tạm dừng một cảm biến để tiết kiệm điện** (chế độ song song)
```c
VL53L0X_StopMeasurement(&tof[1].dev);        /* TOF2 ngung phat laser */
/* ... sau nay chay lai: */
VL53L0X_ClearInterruptMask(&tof[1].dev, 0);
VL53L0X_StartMeasurement(&tof[1].dev);
```
⚠️ Dừng lâu hơn ~0,7 s thì thư viện coi cảm biến bị lỗi (TIME_OUT) và chuyển sang ERROR. Muốn tắt hẳn một cảm biến, tốt nhất **khởi tạo `bus` không có nó**.

---

### 9.10. `VL53L0X_GetDeviceMode` — chế độ đo hiện tại

```c
VL53L0X_Error VL53L0X_GetDeviceMode(VL53L0X_DEV Dev, VL53L0X_DeviceModes *mode);
```
```c
VL53L0X_DeviceModes mode;
VL53L0X_GetDeviceMode(&tof[0].dev, &mode);
/* VL53L0X_DEVICEMODE_CONTINUOUS_RANGING (1) o che do song song, SINGLE_RANGING (0) o tuan tu */
```

---

### 9.11. `VL53L0X_GetVersion` — phiên bản API của ST

```c
VL53L0X_Error VL53L0X_GetVersion(VL53L0X_Version_t *v);
```
```c
VL53L0X_Version_t v;
VL53L0X_GetVersion(&v);      /* v.major = 1, v.minor = 0, v.build = 4, v.revision = 4960 */
```

---

### 9.12. `VL53L0X_GetDeviceInfo` / `GetProductRevision` — thông tin con chip

```c
VL53L0X_Error VL53L0X_GetDeviceInfo(VL53L0X_DEV Dev, VL53L0X_DeviceInfo_t *info);
VL53L0X_Error VL53L0X_GetProductRevision(VL53L0X_DEV Dev, uint8_t *major, uint8_t *minor);
```
Đọc từ bộ nhớ NVM của chip, nên gọi khi **đã dừng đo**.
```c
VL53L0X_DeviceInfo_t info;
VL53L0X_StopMeasurement(&tof[0].dev);
HAL_Delay(40);
if (VL53L0X_GetDeviceInfo(&tof[0].dev, &info) == VL53L0X_ERROR_NONE) {
    /* info.Name (vd "VL53L0X ES1 or later"), info.Type = "VL53L0X",
       info.ProductRevisionMajor, info.ProductRevisionMinor */
}
VL53L0X_ClearInterruptMask(&tof[0].dev, 0);
VL53L0X_StartMeasurement(&tof[0].dev);
```

---

### 9.13. `VL53L0X_GetReferenceSpads` / `GetRefCalibration` — thông số hiệu chuẩn bên trong

```c
VL53L0X_Error VL53L0X_GetReferenceSpads(VL53L0X_DEV Dev, uint32_t *count, uint8_t *isAperture);
VL53L0X_Error VL53L0X_GetRefCalibration(VL53L0X_DEV Dev, uint8_t *vhv, uint8_t *phase);
```
```c
uint32_t spad;
uint8_t  aperture;
VL53L0X_GetReferenceSpads(&tof[0].dev, &spad, &aperture);   /* = tof[0].calib.refSpadCount */
```
`GetRefCalibration` có ghi thanh ghi, nên **chỉ gọi khi đã dừng đo**.

---

### 9.14. `VL53L0X_GetPalErrorString` / `GetRangeStatusString` — chuỗi mô tả (tiếng Anh)

```c
VL53L0X_Error VL53L0X_GetPalErrorString(VL53L0X_Error err, char *buf);       /* buf >= 32 byte */
VL53L0X_Error VL53L0X_GetRangeStatusString(uint8_t status, char *buf);       /* buf >= 32 byte */
```
```c
char txt[VL53L0X_MAX_STRING_LENGTH];                 /* 32 byte */
VL53L0X_GetPalErrorString(tof[0].last_error, txt);   /* vd "Control Interface Error" */
VL53L0X_GetRangeStatusString(tof[0].range_status, txt);   /* vd "Range Valid" */
```
💡 Define `USE_EMPTY_STRING` để bỏ các chuỗi này, tiết kiệm ~1–2 KB flash; khi đó chuỗi trả về là rỗng.

---

### 9.15. `VL53L0X_GetPalState` — trạng thái bên trong API

```c
VL53L0X_Error VL53L0X_GetPalState(VL53L0X_DEV Dev, VL53L0X_State *st);
```
```c
VL53L0X_State st;
VL53L0X_GetPalState(&tof[0].dev, &st);     /* VL53L0X_STATE_RUNNING (4) hoac IDLE (3) */
```

---

### 9.16. Ngắt theo ngưỡng khoảng cách — `SetGpioConfig` + `SetInterruptThresholds` ⚠️ nâng cao

```c
VL53L0X_Error VL53L0X_SetGpioConfig(VL53L0X_DEV Dev, uint8_t pin, VL53L0X_DeviceModes mode,
                                    VL53L0X_GpioFunctionality func, VL53L0X_InterruptPolarity pol);
VL53L0X_Error VL53L0X_SetInterruptThresholds(VL53L0X_DEV Dev, VL53L0X_DeviceModes mode,
                                             FixPoint1616_t low_mm, FixPoint1616_t high_mm);
```
Chân GPIO1 chỉ báo khi khoảng cách **< low** (hoặc **> high**, hoặc ra ngoài khoảng, tuỳ `func`).
```c
/* Truoc tien: tof[0].gpio1_port = NULL (thu vien khong dung GPIO1 de doc nua) */
VL53L0X_SetGpioConfig(&tof[0].dev, 0, VL53L0X_DEVICEMODE_CONTINUOUS_RANGING,
                      VL53L0X_GPIOFUNCTIONALITY_THRESHOLD_CROSSED_LOW,
                      VL53L0X_INTERRUPTPOLARITY_LOW);
VL53L0X_SetInterruptThresholds(&tof[0].dev, VL53L0X_DEVICEMODE_CONTINUOUS_RANGING,
                               (FixPoint1616_t)300 << 16, (FixPoint1616_t)400 << 16);
```
⚠️ **Chưa kiểm thử cùng `tof_multi`.**
- Ở chế độ này GPIO1 **không còn** nghĩa "có số đo mới". Phải đặt `gpio1_port = NULL` cho cảm biến đó.
- Gọi trong mẫu Stop/Start.
- Mất sau `TOF_Restart`.

---

## 10. Các hàm cấp thấp (platform)

Thường **không cần** dùng; lõi ST gọi chúng. Hữu ích khi cần đọc nhanh một thanh ghi.

| Hàm | Công dụng |
|---|---|
| `VL53L0X_RdByte(Dev, reg, *u8)` | Đọc 1 byte |
| `VL53L0X_RdWord(Dev, reg, *u16)` | Đọc 2 byte (byte cao trước) |
| `VL53L0X_RdDWord(Dev, reg, *u32)` | Đọc 4 byte |
| `VL53L0X_WrByte(Dev, reg, u8)` | Ghi 1 byte |
| `VL53L0X_WrWord(Dev, reg, u16)` | Ghi 2 byte |
| `VL53L0X_WrDWord(Dev, reg, u32)` | Ghi 4 byte |
| `VL53L0X_ReadMulti(Dev, reg, buf, n)` | Đọc n byte liên tiếp |
| `VL53L0X_WriteMulti(Dev, reg, buf, n)` | Ghi n byte liên tiếp |
| `VL53L0X_UpdateByte(Dev, reg, and, or)` | Đọc → `(x & and) \| or` → ghi |
| `VL53L0X_PollingDelay(Dev)` | Chờ 1 ms (dùng trong vòng chờ của ST) |

Lỗi I2C (sau 1 lần thử lại) → trả về **−20**.

**Ví dụ: kiểm tra đúng là VL53L0X**
```c
uint8_t id = 0;
if (VL53L0X_RdByte(&tof[0].dev, VL53L0X_REG_IDENTIFICATION_MODEL_ID, &id) == VL53L0X_ERROR_NONE
    && id == 0xEE) {
    /* dung la VL53L0X, dang tra loi o dia chi tof[0].addr */
}
```
**Ví dụ: đọc khoảng cách thô** (không kiểm tra chất lượng, chỉ để thử nghiệm)
```c
uint16_t raw;
VL53L0X_RdWord(&tof[0].dev, VL53L0X_REG_RESULT_RANGE_STATUS + 10, &raw);   /* thanh ghi 0x1E */
```
⚠️ **Không ghi** thanh ghi tuỳ ý khi không chắc chắn: có thể làm cảm biến chạy sai cho tới khi restart.

---

## 11. Tuỳ chỉnh bằng macro

Thêm vào *Options for Target → C/C++ → Define*, phân cách bằng dấu phẩy:
```text
USE_HAL_DRIVER,STM32F103xB,USE_I2C_2V8,USE_EMPTY_STRING,TOF_MAX_ERRORS=10
```

| Macro | Mặc định | Công dụng |
|---|---|---|
| `USE_I2C_2V8` | tắt | **Nên bật** với module GY-530 (I/O 2,8 V) |
| `USE_EMPTY_STRING` | tắt | Bỏ chuỗi mô tả lỗi, tiết kiệm ~1–2 KB flash |
| `VL53L0X_HAL_HEADER` | `"stm32f1xx_hal.h"` | Dùng chip khác, ví dụ `VL53L0X_HAL_HEADER="stm32f4xx_hal.h"` |
| `VL53L0X_I2C_TIMEOUT_MS` | 10 | Timeout mỗi lệnh I2C |
| `VL53L0X_I2C_RETRIES` | 1 | Số lần thử lại khi I2C lỗi |
| `TOF_BOOT_MS` | 2 | Chờ sau khi bật XSHUT |
| `TOF_MAX_ERRORS` | 5 | Số lỗi liên tiếp trước khi chuyển sang ERROR |
| `TOF_MAX_READS_PER_POLL` | 1 | Số cảm biến đọc tối đa mỗi lần `TOF_Poll` |
| `TOF_STALE_FACTOR` | 4 | Quá `budget × hệ số + 10 ms` không có số đo thì tính 1 lỗi |

---

## 12. Sự cố thường gặp

| Hiện tượng | Nguyên nhân hay gặp | Cách xử lý |
|---|---|---|
| `TOF_Init` trả 0, lỗi −20 ở mọi cảm biến | SDA/SCL lắp ngược; thiếu nguồn VIN/GND; chưa bật I2C1; gọi `TOF_Init` trước `MX_I2C1_Init` | Kiểm tra PB6 = SCL, PB7 = SDA, nguồn 3,3 V, thứ tự gọi hàm |
| Chỉ một cảm biến lỗi −20 | Dây XSHUT của nó, hoặc module hỏng | Đổi chỗ hai module để khoanh vùng |
| Quét I2C trước init đã thấy 0x52 | Một chân XSHUT chưa nối hoặc sai chân: hai chip cùng địa chỉ | Kiểm tra dây và label XSHUT trong CubeMX |
| Có vật mà `range_status` luôn 2, `range_mm` ≈ 8190 | **Chưa bóc miếng dán bảo vệ**; kính che bẩn | Bóc miếng dán, lau mắt cảm biến |
| Lỗi −7 lặp lại, I2C vẫn tốt | Dây GPIO1 đứt hoặc sai chân | Thử đặt `gpio1_port = NULL` |
| Hai cảm biến đặt sát nhau, số đo nhảy | Nhiễu laser lẫn nhau | `bus.mode = TOF_MODE_SEQUENTIAL` |
| Số đo lệch cố định vài mm đến vài cm | Offset | `TOF_CalibrateOffset` |
| Sau khi đổi budget lên 200 ms, cảm biến báo −7 | Chưa sửa `bus.budget_us` | `bus.budget_us = 200000` |
| Chương trình treo sau khi reset nóng | Bus I2C bị kẹt | Tắt nguồn cả mạch, hoặc `TOF_I2C_Recover` |
| Keil báo lỗi ở `LOG_FUNCTION_START` | Chưa bật C99 | Bật C99 |
| Keil: `L6218E Undefined symbol VL53L0X_...` | Thiếu file `.c` | Xem bước 3 |
| Keil: `L6050U code size exceeds` | Keil bản Lite giới hạn 32 KB | Tối ưu `-Oz`, `USE_EMPTY_STRING`, hoặc MDK Community |
| Watch đứng yên | Chưa bật *Periodic Window Update* | Bật mục đó |
| HardFault | Thiếu stack | Minimum Stack = 0x800 |

---

## 13. Bảng tra mã lỗi và trạng thái

### Mã lỗi (`last_error`, giá trị trả về của các hàm)

| Mã | Tên macro | Ý nghĩa |
|---|---|---|
| 0 | `VL53L0X_ERROR_NONE` | Thành công |
| −4 | `VL53L0X_ERROR_INVALID_PARAMS` | Tham số / địa chỉ sai |
| −6 | `VL53L0X_ERROR_RANGE_ERROR` | Không có số đo hợp lệ (khi hiệu chuẩn) |
| −7 | `VL53L0X_ERROR_TIME_OUT` | Quá lâu không có số đo |
| −12 | `VL53L0X_ERROR_INTERRUPT_NOT_CLEARED` | Không xoá được cờ ngắt |
| −20 | `VL53L0X_ERROR_CONTROL_INTERFACE` | **Không trả lời I2C** |
| −30 | `VL53L0X_ERROR_INVALID_COMMAND` | Gọi sai lúc (ví dụ cảm biến chưa chạy) |
| −50 | `VL53L0X_ERROR_REF_SPAD_INIT` | Hiệu chuẩn SPAD lỗi (bị che?) |

### Trạng thái số đo (`range_status`)

| Giá trị | Ý nghĩa |
|---|---|
| 0 | Hợp lệ |
| 1 | Không chắc chắn (sigma) |
| 2 | Tín hiệu yếu / không có vật |
| 3 | Vật quá gần |
| 4 | Ngoài tầm |
| 5 | Lỗi phần cứng |
| 255 | Chưa có số đo |

### Thông số nhanh

| Thông số | Giá trị |
|---|---|
| Địa chỉ mặc định | 0x52 (8-bit) |
| Tầm đo mặc định / tầm xa | ~1,2 m / ~2 m |
| Góc nhìn | ~25° |
| Tốc độ đo (budget 33 ms) | ~30 Hz mỗi cảm biến (song song) |
| RAM mỗi cảm biến | ~470 byte |
| Flash (lõi ST) | ước tính ~25–30 KB |

---

## 14. Nên và không nên

| ✔ Nên | ✘ Không nên |
|---|---|
| Gọi `TOF_Poll` **một lần** mỗi vòng `while(1)` | Gọi `TOF_Poll` trong hàm ngắt |
| Dùng `TOF_GetRange` hoặc kiểm tra `range_status == 0` | Dùng `range_mm` mà không kiểm tra |
| Khai báo `tof[]`, `bus` là biến toàn cục | Khai báo trong hàm (tràn stack, ~470 byte mỗi cảm biến) |
| Cấu hình XSHUT: Open-Drain, mức Low | Để XSHUT trống hoặc mức High ban đầu |
| Đặt `bus.budget_us` đúng budget đang dùng | Đổi budget bằng hàm ST mà quên báo thư viện |
| Đặt lại cấu hình riêng sau `TOF_Restart` | Cho rằng cấu hình tự đặt vẫn còn sau restart |
| Bus I2C 100 kHz nếu có LCD PCF8574 | Chạy 400 kHz khi có PCF8574 trên bus |
| Bật C99, define `USE_I2C_2V8` | Thêm file Windows của ST vào project |
| Lưu `calib` vào flash sau khi hiệu chuẩn | Hiệu chuẩn lại mỗi lần khởi động |
| `HAL_GetTick() - t >= khoảng` khi so sánh thời gian | `HAL_GetTick() > t + khoảng` (sai khi tick tràn) |

---

> 📚 Tài liệu chi tiết hơn: [Sách giáo khoa](docs/01_SachGiaoKhoa.md) · [Tham khảo API](docs/02_SachThamKhao_API.md) · [12 ví dụ](docs/03_ViDu_GiaiThich.md) · [Bài tập](docs/04_BaiTap.md) · [Lời giải](docs/05_LoiGiai.md) · [Ôn tập](docs/06_OnThi.md) · [Nâng cao](docs/07_ChuyenDe_NangCao.md)


# Bộ sách: Thư viện VL53L0X đa cảm biến cho STM32 HAL

> Áp dụng cho thư viện trong thư mục `vl53l0x_stm32/` (lõi ST STSW-IMG005 v1.0.4 + lớp `platform` + lớp `tof_multi`), vi điều khiển STM32F103C8T6 (Blue Pill), STM32CubeMX và Keil MDK.

## Các quyển trong bộ sách

| # | Quyển | Nội dung | Dành cho |
|---|---|---|---|
| 1 | [Sách giáo khoa](01_SachGiaoKhoa.md) | Lý thuyết nền: nguyên lý ToF, I2C, kiến trúc thư viện, nhiều cảm biến, khởi tạo, đo không chặn CPU, định dạng số, hiệu chuẩn, lỗi | Người mới, đọc theo thứ tự |
| 2 | [Sách tham khảo API](02_SachThamKhao_API.md) | Tra cứu từng macro, kiểu dữ liệu và hàm: tham số, giá trị trả về, thời gian chặn CPU, lưu ý; bảng mã lỗi, thanh ghi, thông số | Tra cứu khi viết code |
| 3 | [Ví dụ và giải thích](03_ViDu_GiaiThich.md) | 12 ví dụ từ dễ đến khó, giải thích từng khối code, kết quả mong đợi, lỗi hay gặp | Học bằng thực hành |
| 4 | [Bài tập](04_BaiTap.md) | Trắc nghiệm, tính toán, đọc code tìm lỗi, chẩn đoán qua Watch, lập trình, dự án nhỏ | Tự luyện |
| 5 | [Lời giải chi tiết](05_LoiGiai.md) | Lời giải đầy đủ cho quyển 4 | Đối chiếu sau khi làm |
| 6 | [Ôn tập](06_OnThi.md) | Tóm tắt trọng tâm, công thức, bảng tra, các bẫy, 2 đề kiểm tra có đáp án và thang điểm | Ôn nhanh, kiểm tra |
| 7 | [Chuyên đề nâng cao](07_ChuyenDe_NangCao.md) | Phân tích sâu, chứng minh, tối ưu, mở rộng thư viện, bài toán thử thách có lời giải | Muốn hiểu tận gốc |

## Lộ trình học gợi ý

```
Người mới:      Q1 (chương 1→6) → Q3 (VD1→VD5) → Q4 phần A, B → Q5 → nạp thử mạch thật
Đã quen STM32:  Q1 (chương 3, 4, 6) → Q2 → Q3 (VD2, VD6→VD12) → Q4 phần C, D, E
Ôn kiểm tra:    Q6 → làm 2 đề → đối chiếu đáp án
Nâng cao:       Q7 → các bài thử thách → tự mở rộng thư viện
```

## Quy ước trong sách

| Ký hiệu | Ý nghĩa |
|---|---|
| `TOF_Init()` | Tên hàm, biến, macro trong code |
| **0x52** | Địa chỉ I2C dạng **8-bit** (dạng mà HAL dùng), trừ khi ghi rõ "7-bit" |
| ⚠️ | Bẫy hay gặp, dễ gây lỗi |
| 💡 | Mẹo, kinh nghiệm |
| 🔬 | Kết quả đo trên **bộ giả lập** (chưa đo trên phần cứng thật) |
| [Q2 §C.1] | Tham chiếu: quyển 2, mục C.1 |

## Về các con số trong sách

- Các thông số của cảm biến (tầm đo, góc nhìn, thời gian khởi động…) lấy theo tài liệu của ST (datasheet, UM2039, API v1.0.4). Đây là giá trị **điển hình**; con chip cụ thể có thể khác.
- Các con số đánh dấu 🔬 lấy từ bộ giả lập dùng để kiểm thử thư viện. Bộ giả lập mô phỏng nhiều VL53L0X trên một bus I2C 100 kHz; mỗi lần đo của nó kéo dài 30 ms. **Mạch thật sẽ khác một chút**: mỗi lần đo theo timing budget là 33 ms, nên tốc độ thật khoảng 30 Hz thay vì 33 Hz.
- Mọi đoạn code trong quyển 3 đều đã được biên dịch (GCC `-std=c99 -Wall -Wextra -pedantic`) và chạy trên bộ giả lập. **Chưa đoạn nào được chạy trên Keil hay mạch thật.**

## Cấu trúc thư viện (nhắc lại)

```
vl53l0x_stm32/
├── st_core/    Lõi của ST: 12 file nguyên bản + 2 file vá lỗi (đánh dấu PATCH(tof-lib))
├── platform/   Nối API của ST với HAL: vl53l0x_platform.c/.h, vl53l0x_types.h, vl53l0x_platform_log.h
├── tof/        tof_multi.c/.h: quản lý nhiều cảm biến. tof_example.c: ví dụ 4 cảm biến
├── app/        app_tof_test.c (in qua UART), app_tof_dbg.c (xem bằng Watch của Keil)
├── docs/       Bộ sách này
└── README.md   Hướng dẫn tích hợp nhanh
```

# Quyển 1 — SÁCH GIÁO KHOA
## Thư viện VL53L0X đa cảm biến cho STM32 HAL

> **Mục tiêu của quyển này:** sau khi học xong, bạn hiểu *vì sao* thư viện được thiết kế như vậy, chứ không chỉ biết gọi hàm. Mỗi chương có phần **Tóm tắt** và **Câu hỏi nhanh** (bấm để xem đáp án).
>
> **Kiến thức cần có:** C cơ bản (struct, con trỏ, mảng, phép bit), biết tạo project STM32CubeMX và nạp code bằng Keil.

---

## Mục lục

1. [Cảm biến VL53L0X và nguyên lý đo thời gian bay (ToF)](#chương-1--cảm-biến-vl53l0x-và-nguyên-lý-tof)
2. [Giao tiếp I2C trên STM32 — những điều phải biết](#chương-2--giao-tiếp-i2c-trên-stm32)
3. [Kiến trúc thư viện](#chương-3--kiến-trúc-thư-viện)
4. [Nhiều cảm biến trên một bus I2C](#chương-4--nhiều-cảm-biến-trên-một-bus-i2c)
5. [Khởi tạo cảm biến theo chuẩn của ST](#chương-5--khởi-tạo-cảm-biến-theo-chuẩn-của-st)
6. [Đo không chặn CPU](#chương-6--đo-không-chặn-cpu)
7. [Định dạng số và đơn vị](#chương-7--định-dạng-số-và-đơn-vị)
8. [Chất lượng số đo — RangeStatus](#chương-8--chất-lượng-số-đo--rangestatus)
9. [Hiệu chuẩn](#chương-9--hiệu-chuẩn)
10. [Lỗi, phục hồi và độ tin cậy](#chương-10--lỗi-phục-hồi-và-độ-tin-cậy)
11. [Phần cứng, công cụ và tài nguyên](#chương-11--phần-cứng-công-cụ-và-tài-nguyên)

---

## Chương 1 — Cảm biến VL53L0X và nguyên lý ToF

### 1.1. Đo khoảng cách bằng thời gian bay của ánh sáng

**ToF** (*Time of Flight*, thời gian bay): cảm biến phát một xung ánh sáng, xung đập vào vật rồi phản xạ về. Cảm biến đo thời gian `t` từ lúc phát đến lúc nhận. Ánh sáng đi **cả lượt đi lẫn lượt về**, nên:

$$d = \frac{c \cdot t}{2}, \qquad c \approx 3 \times 10^{8}\ \text{m/s}$$

| Khoảng cách `d` | Thời gian đi + về `t = 2d/c` |
|---|---|
| 1 mm | 6,67 ps (pico giây = 10⁻¹² s) |
| 10 mm | 66,7 ps |
| 100 mm | 667 ps |
| 1 m | 6,67 ns |
| 2 m | 13,3 ns |

**Nhận xét:** muốn phân giải 1 mm, cảm biến phải phân biệt được chênh lệch cỡ vài pico giây. Không bộ đếm thông thường nào làm được việc này. VL53L0X giải quyết bằng hai cách:
- Dùng **SPAD** (*Single Photon Avalanche Diode*), loại diode nhạy tới mức phát hiện được **từng photon**.
- **Lặp lại rất nhiều xung** trong một lần đo, rồi xử lý thống kê để ra kết quả.

### 1.2. Cấu tạo bên trong

```
            ┌──────────────── VL53L0X ────────────────┐
  Ánh sáng ◄┤ VCSEL 940 nm (laser hồng ngoại, class 1) │
  phản xạ  ►┤ Mảng SPAD (đếm photon trở về)            │
            │ Vi xử lý bên trong + bộ nhớ NVM (hiệu    │
            │ chuẩn xuất xưởng: SPAD, offset...)       │
            │ Giao tiếp I2C (tối đa 400 kHz)           │◄──► SDA, SCL
            │ XSHUT (vào): mức thấp = tắt hẳn chip     │◄─── XSHUT
            │ GPIO1 (ra): báo "có số đo mới" (mức thấp)│───► GPIO1
            └──────────────────────────────────────────┘
```

| Bộ phận | Vai trò |
|---|---|
| **VCSEL** (*Vertical-Cavity Surface-Emitting Laser*) | Phát xung hồng ngoại 940 nm. Mắt người không thấy; công suất ở mức an toàn class 1 |
| **Mảng SPAD** | Đếm photon phản xạ về. Có SPAD **tham chiếu** (đo ánh sáng đi thẳng bên trong chip) và SPAD **thu** (đo ánh sáng từ vật) |
| **NVM** | Lưu dữ liệu hiệu chuẩn của nhà máy: số SPAD tham chiếu, bản đồ SPAD tốt, offset |
| **XSHUT** | Chân tắt chip, tích cực mức thấp. Kéo thấp thì chip tắt hẳn và **quên mọi cấu hình, kể cả địa chỉ I2C** |
| **GPIO1** | Chân ngắt, open-drain. Sau khi thư viện khởi tạo, chân này xuống thấp khi có số đo mới |

### 1.3. Các thông số chính (giá trị điển hình theo ST)

| Thông số | Giá trị |
|---|---|
| Địa chỉ I2C mặc định | **0x52** dạng 8-bit (= 0x29 dạng 7-bit) |
| Tốc độ I2C | Đến 400 kHz. Thư viện này chạy **100 kHz** vì dùng chung bus với LCD PCF8574 |
| Tầm đo | ~1,2 m ở cấu hình mặc định; ~2 m ở cấu hình tầm xa (trong nhà, vật sáng màu) |
| Góc nhìn (FoV) | ~25° |
| Timing budget (thời gian dành cho 1 lần đo) | 20 ms → 200 ms; mặc định ~33 ms |
| Thời gian khởi động sau khi XSHUT lên cao (tBOOT) | Tối đa 1,2 ms |
| Đầu ra khi không có vật | Khoảng 8190–8191 mm, kèm trạng thái "không hợp lệ" |

### 1.4. Một lần đo diễn ra thế nào

Mỗi lần đo, chip chạy một **chuỗi bước** (*sequence*). Thanh ghi `SYSTEM_SEQUENCE_CONFIG` (0x01) quy định bước nào được bật:

| Bước | Tên đầy đủ | Sau khi thư viện khởi tạo |
|---|---|---|
| TCC | Target Centre Check | Tắt |
| MSRC | Minimum Signal Rate Check | Tắt |
| DSS | Dynamic SPAD Selection: tự chọn số SPAD theo độ sáng | **Bật** |
| Pre-range | Đo sơ bộ | **Bật** |
| Final-range | Đo chính xác, chiếm phần lớn thời gian | **Bật** |

**Timing budget** là tổng thời gian cho cả chuỗi. API của ST trừ đi thời gian cố định (khởi động, kết thúc) và thời gian các bước phụ; **phần còn lại dành cho bước final-range**. Vì vậy budget càng lớn thì final-range càng dài, càng nhiều photon được đếm, kết quả càng ổn định.

> 💡 **Quy luật thống kê:** tăng timing budget lên **N lần** thì độ lệch chuẩn của kết quả giảm khoảng **√N lần**. Ví dụ: budget từ 33 ms lên 132 ms (×4) thì số đo dao động ít đi khoảng 2 lần, đổi lại tốc độ đo giảm 4 lần.

### 1.5. Các đại lượng cảm biến trả về

| Đại lượng | Ý nghĩa | Dùng để |
|---|---|---|
| `RangeMilliMeter` | Khoảng cách (mm) | Kết quả chính |
| `RangeStatus` | Chất lượng số đo: 0 = hợp lệ | **Luôn kiểm tra trước khi dùng khoảng cách** |
| `SignalRateRtnMegaCps` | Cường độ tín hiệu phản xạ, triệu photon mỗi giây (MCPS) | Vật sáng màu, ở gần thì giá trị lớn |
| `AmbientRateRtnMegaCps` | Cường độ ánh sáng môi trường | Lớn (ví dụ dưới nắng) thì tầm đo giảm |
| `EffectiveSpadRtnCount` | Số SPAD hiệu dụng | Phục vụ ước lượng sigma |
| Sigma (bên trong ST) | Ước lượng độ lệch chuẩn của số đo (mm) | Lọc số đo kém tin cậy |

### 1.6. Vùng nhìn và nhiễu giữa các cảm biến

Góc nhìn 25° nghĩa là chùm tia là **một hình nón** có nửa góc 12,5°. Ở khoảng cách `d`, đường kính vùng nhìn là:

$$D = 2 \cdot d \cdot \tan(12{,}5^\circ) \approx 0{,}443 \cdot d$$

| `d` | Đường kính vùng nhìn `D` |
|---|---|
| 100 mm | ~44 mm |
| 500 mm | ~222 mm |
| 1000 mm | ~443 mm |

⚠️ **Nhiễu quang:** khi 2 cảm biến nhìn cùng hướng và vùng nhìn chồng lên nhau, xung laser của con A có thể phản xạ vào con B. Con B nhận nhầm photon không phải của mình, nên số đo nhảy hoặc sai. Thư viện có **chế độ tuần tự** (`TOF_MODE_SEQUENTIAL`, chương 6) để tại mỗi thời điểm chỉ một cảm biến phát tia.

### Tóm tắt chương 1
- ToF: `d = c·t/2`. 1 mm tương ứng ~6,67 ps, nên chip cần SPAD và phép thống kê nhiều xung.
- XSHUT thấp: chip tắt hẳn và mất cấu hình, kể cả địa chỉ. GPIO1 thấp: có số đo mới.
- Timing budget lớn thì chính xác hơn nhưng chậm hơn (độ lệch chuẩn giảm theo √N).
- Vùng nhìn rộng ~0,443·d. Vùng nhìn chồng nhau thì có nguy cơ nhiễu.

### Câu hỏi nhanh
<details><summary>1. Vật cách 300 mm thì thời gian đi và về của ánh sáng là bao nhiêu?</summary>

`t = 2d/c = 2 × 0,3 / 3×10⁸ = 2×10⁻⁹ s = 2 ns`.
</details>

<details><summary>2. Vì sao kéo XSHUT xuống thấp rồi lên lại thì phải cấu hình lại toàn bộ?</summary>

Vì chip tắt hẳn. Mọi thanh ghi, kể cả địa chỉ I2C, trở về mặc định. Chỉ dữ liệu trong NVM là còn.
</details>

<details><summary>3. Muốn số đo dao động ít đi 3 lần thì timing budget phải tăng bao nhiêu lần?</summary>

√N = 3, nên N = 9. Ví dụ từ 22 ms lên ~200 ms.
</details>

---

## Chương 2 — Giao tiếp I2C trên STM32

### 2.1. Khung truyền I2C

I2C dùng 2 dây là **SDA** (dữ liệu) và **SCL** (xung clock). Cả hai đều là **open-drain**: thiết bị chỉ kéo được xuống thấp, còn mức cao do **điện trở kéo lên** tạo ra.

**Ghi một thanh ghi** (hàm `HAL_I2C_Mem_Write`):
```
S | ĐịaChỉ+W | A | Thanh ghi | A | Dữ liệu 1 | A | ... | Dữ liệu n | A | P
```
**Đọc một thanh ghi** (hàm `HAL_I2C_Mem_Read`, dùng *repeated START*):
```
S | ĐịaChỉ+W | A | Thanh ghi | A | Sr | ĐịaChỉ+R | A | Dữ liệu 1 | A | ... | Dữ liệu n | NA | P
```
S = START, Sr = START lặp lại, P = STOP, A = ACK (thiết bị nhận kéo SDA xuống thấp), NA = NACK.

⚠️ Nếu không thiết bị nào trả ACK cho byte địa chỉ (NACK), HAL trả về `HAL_ERROR` ngay. Đây là cách thư viện phát hiện "cảm biến không trả lời".

### 2.2. Địa chỉ 7-bit và 8-bit

Byte địa chỉ gồm **7 bit địa chỉ** và **1 bit R/W** ở vị trí thấp nhất. Có hai cách viết cùng một địa chỉ:

| Thiết bị | 7-bit (Arduino, datasheet) | **8-bit** (HAL STM32, thư viện này) |
|---|---|---|
| VL53L0X mặc định | 0x29 | **0x52** |
| PCF8574 (LCD) | 0x27 | 0x4E |
| PCF8574A (LCD) | 0x3F | 0x7E |
| Ví dụ TOF1 sau khi đổi | 0x2A | **0x54** |

Công thức: `8-bit = 7-bit << 1`. **HAL STM32 nhận địa chỉ 8-bit.**

⚠️ Hàm `VL53L0X_SetDeviceAddress()` của ST cũng nhận địa chỉ **8-bit**. Bên trong nó ghi `địa_chỉ / 2` vào thanh ghi 0x8A.

### 2.3. Thời gian của một giao dịch

Mỗi byte trên bus tốn **9 bit** (8 bit dữ liệu + 1 bit ACK). Ở 100 kHz, mỗi bit mất 10 µs, nên **1 byte mất 90 µs**.

$$t \approx \frac{9 \times n_\text{byte} + 3}{f_\text{SCL}}$$

| Giao dịch | Số byte trên bus | Thời gian @100 kHz | @400 kHz |
|---|---|---|---|
| Ghi 1 thanh ghi (`WrByte`) | 3 (địa chỉ, thanh ghi, dữ liệu) | ~0,3 ms | ~0,08 ms |
| Đọc 1 thanh ghi (`RdByte`) | 4 (địa chỉ W, thanh ghi, địa chỉ R, dữ liệu) | ~0,4 ms | ~0,1 ms |
| Đọc 12 byte kết quả (`ReadMulti`) | 15 | ~1,35 ms | ~0,35 ms |

> 💡 Các con số trên là thời gian **CPU bị chặn**, vì HAL ở chế độ polling: CPU đứng chờ cho tới khi giao dịch xong.

### 2.4. Thứ tự byte: big-endian

VL53L0X gửi và nhận số nhiều byte theo thứ tự **byte cao trước** (*big-endian*). STM32 lưu trong RAM theo thứ tự **byte thấp trước** (*little-endian*). Vì vậy **không được `memcpy` thẳng** từ bộ đệm I2C vào biến `uint16_t`.

Ví dụ: khoảng cách 500 mm = 0x01F4. Chip gửi byte `0x01` trước, rồi `0xF4`. Ghép đúng:
```c
uint16_t mm = ((uint16_t)buf[0] << 8) | buf[1];   /* = 0x01F4 = 500 */
```
Nếu `memcpy`, bạn nhận được 0xF401 = 62465, sai hoàn toàn. Lớp platform (`VL53L0X_RdWord`, `RdDWord`, `WrWord`, `WrDWord`) đã xử lý việc này.

### 2.5. Trang thanh ghi

Địa chỉ thanh ghi chỉ có 8 bit (0x00–0xFF), nhưng VL53L0X có nhiều thanh ghi hơn thế. Chip chia thành **các trang**: ghi giá trị vào thanh ghi **0xFF** để chọn trang. Ví dụ ST ghi chú `0x0030 /* 0x130 */` nghĩa là thanh ghi 0x30 **ở trang 1**: phải ghi `0xFF = 0x01` trước khi truy cập, xong ghi `0xFF = 0x00` để quay về.

> 💡 Vì vậy một số thanh ghi trong file `vl53l0x_device.h` trùng số (ví dụ hai thanh ghi cùng 0x30). Chúng nằm ở hai trang khác nhau. API của ST tự chuyển trang; bạn không cần làm gì.

### 2.6. Điện áp, điện trở kéo lên, tốc độ

- Module GY-530 có sẵn điện trở kéo lên cho SDA/SCL. Hai module song song vẫn ổn; từ 5 module trở lên nên tháo bớt để tổng trở kéo lên không dưới ~1,5 kΩ.
- Chân PB6/PB7 (I2C1) của STM32F103 chịu được 5 V, nhưng **VL53L0X thì không**. Hãy cấp VIN 3,3 V cho module.
- Bus dùng chung với LCD PCF8574 thì **phải để 100 kHz**: PCF8574 chỉ hỗ trợ chế độ chuẩn 100 kHz.

### 2.7. HAL ở chế độ polling: thời gian và bộ đếm tick

| Hàm HAL | Đặc điểm cần nhớ |
|---|---|
| `HAL_I2C_Mem_Read/Write(..., Timeout)` | **Chặn CPU** cho tới khi xong hoặc hết `Timeout` ms. Thư viện dùng 10 ms, không dùng `HAL_MAX_DELAY` |
| `HAL_GetTick()` | Đếm mili giây từ lúc khởi động, kiểu `uint32_t`. Quay vòng về 0 sau 2³² ms ≈ **49,7 ngày** |
| `HAL_Delay(n)` | Chờ **từ n đến n+1 ms**, vì HAL cộng thêm 1 tick để đảm bảo tối thiểu n ms |

⚠️ **So sánh thời gian đúng cách khi tick quay vòng:**
```c
if (HAL_GetTick() - t_start >= 1000U) { ... }          /* ĐÚNG: phép trừ không dấu */
if ((int32_t)(now - deadline) >= 0) { ... }            /* ĐÚNG: kiểm tra "đã tới hạn chưa" */
if (HAL_GetTick() >= t_start + 1000U) { ... }          /* SAI khi t_start + 1000 bị tràn */
```

### 2.8. Bus I2C bị treo

Nếu MCU reset đúng lúc cảm biến đang gửi bit 0, cảm biến **tiếp tục giữ SDA ở mức thấp**, chờ xung clock không bao giờ tới. Khi đó khối I2C của STM32F1 thấy bus "bận" (cờ BUSY), và mọi lệnh HAL đều trả lỗi.

**Cách phục hồi** (hàm `TOF_I2C_Recover`):
1. Reset mềm khối I2C.
2. Chuyển chân SCL/SDA thành GPIO open-drain.
3. Tạo tối đa **9 xung SCL**, cho tới khi slave nhả SDA.
4. Tạo điều kiện STOP.
5. Khởi tạo lại I2C bằng HAL.

### Tóm tắt chương 2
- Địa chỉ dùng với HAL là **8-bit**: VL53L0X 0x52, LCD 0x4E.
- 1 byte @100 kHz mất 90 µs. Đọc 1 thanh ghi ~0,4 ms; đọc 12 byte ~1,35 ms.
- VL53L0X là big-endian: ghép byte cao trước, không `memcpy`.
- So sánh thời gian bằng phép trừ không dấu; `HAL_Delay(n)` chờ n..n+1 ms.

### Câu hỏi nhanh
<details><summary>1. Địa chỉ 7-bit 0x2B ứng với địa chỉ 8-bit bao nhiêu?</summary>

0x2B << 1 = **0x56**.
</details>

<details><summary>2. Chip gửi 2 byte 0x03, 0x20. Khoảng cách là bao nhiêu mm?</summary>

0x0320 = 3×256 + 32 = **800 mm**.
</details>

<details><summary>3. Ở 100 kHz, đọc 6 byte liên tiếp bằng Mem_Read mất khoảng bao lâu?</summary>

Số byte trên bus = 3 (địa chỉ W, thanh ghi, địa chỉ R) + 6 = 9 byte → 9 × 90 µs ≈ **0,81 ms**, cộng thêm vài chục µs cho START/STOP.
</details>

---

## Chương 3 — Kiến trúc thư viện

### 3.1. Ba lớp

```
 ┌──────────────────────────────────────────────────────────┐
 │ Ứng dụng: main.c, app_tof_test.c, app_tof_dbg.c          │
 └───────────────┬──────────────────────────────────────────┘
                 │ TOF_Init, TOF_Poll, TOF_GetRange, TOF_Restart...
 ┌───────────────▼──────────────────────────────────────────┐
 │ Lớp 3 — tof/tof_multi.c                                  │  ← viết mới
 │  Quản lý N cảm biến: XSHUT, gán địa chỉ, lịch đo,        │
 │  phát hiện lỗi, phục hồi                                 │
 └───────────────┬──────────────────────────────────────────┘
                 │ VL53L0X_DataInit, StaticInit, StartMeasurement...
 ┌───────────────▼──────────────────────────────────────────┐
 │ Lớp 2 — st_core/ (API chính thức của ST v1.0.4)          │  ← chép nguyên, vá 2 lỗi
 │  Thuật toán, trình tự khởi tạo, hiệu chuẩn, tính sigma   │
 └───────────────┬──────────────────────────────────────────┘
                 │ VL53L0X_WrByte, RdWord, ReadMulti, PollingDelay...
 ┌───────────────▼──────────────────────────────────────────┐
 │ Lớp 1 — platform/vl53l0x_platform.c                      │  ← viết lại cho HAL
 │  12 hàm nối API ST với HAL_I2C_Mem_Write/Read            │
 └───────────────┬──────────────────────────────────────────┘
                 ▼
          HAL I2C (phần cứng, polling)
```

**Nguyên tắc thiết kế:**
1. **Không sửa thuật toán của ST.** Lõi đo được chép nguyên vẹn, chỉ vá 2 lỗi đã được chứng minh. Mọi chỗ vá đánh dấu `PATCH(tof-lib)`.
2. **Mỗi lớp chỉ gọi lớp ngay dưới nó.** `tof_multi` chỉ dùng các hàm công khai trong `vl53l0x_api.h`, cộng `VL53L0X_RdByte` để đọc ID.
3. **Không có biến toàn cục giữ trạng thái**, nên một lõi chạy được cho nhiều cảm biến.

### 3.2. Các file và nguồn gốc

| Thư mục | File | Nguồn | Ghi chú |
|---|---|---|---|
| `st_core/` | `vl53l0x_api.c/.h` | ST | Các hàm công khai (94 hàm) |
| | `vl53l0x_api_core.c/.h` | ST | Tính timeout, sigma, Dmax, giải mã trạng thái |
| | `vl53l0x_api_calibration.c/.h` | ST, **có vá** | Hiệu chuẩn SPAD, VHV/phase, offset, crosstalk |
| | `vl53l0x_api_strings.c/.h` | ST, **.h có vá** | Chuỗi mô tả mã lỗi |
| | `vl53l0x_api_ranging.c/.h` | ST | File rỗng (ST đã bỏ tính năng histogram) |
| | `vl53l0x_def.h`, `vl53l0x_device.h` | ST | Kiểu dữ liệu, mã lỗi, bảng địa chỉ thanh ghi |
| | `vl53l0x_tuning.h`, `vl53l0x_interrupt_threshold_settings.h` | ST | Bảng cấu hình nạp vào chip. **Chỉ `vl53l0x_api.c` được include hai file này** |
| `platform/` | `vl53l0x_platform.c/.h` | Viết lại theo khung của ST | Dùng HAL |
| | `vl53l0x_types.h`, `vl53l0x_platform_log.h` | ST, rút gọn | |
| `tof/` | `tof_multi.c/.h` | Mới | Quản lý nhiều cảm biến |
| `app/` | `app_tof_test.c`, `app_tof_dbg.c` | Mới | Chương trình chạy thử |

### 3.3. Hai bản vá trong lõi ST

| File | Lỗi gốc | Hậu quả | Cách vá |
|---|---|---|---|
| `vl53l0x_api_calibration.c` | Tổng 50 lần đo lưu trong `uint16_t sum_ranging`, rồi tính `sum_ranging << 16` | Trung bình > 655 mm thì `<< 16` tràn số `int`; > 1310 mm thì chính phép cộng tràn. Hiệu chuẩn crosstalk ở khoảng cách lớn ra kết quả sai | Đổi sang `uint32_t`, tính trung bình 16.16 bằng macro `VL53L0X_MEAN_1616` (không tràn) |
| `vl53l0x_api_strings.h` | Nhánh `USE_EMPTY_STRING` thiếu 3 macro | Define `USE_EMPTY_STRING` để tiết kiệm flash thì không biên dịch được | Thêm 3 macro chuỗi rỗng |

### 3.4. Đường đi của một lần đọc khoảng cách

```
TOF_Poll(&bus)                                    tof_multi.c
 └─ tof_service(b, s, now)
     ├─ tof_data_ready() → đọc chân GPIO1 (không tốn I2C)
     └─ tof_read(s, now)
         ├─ VL53L0X_GetRangingMeasurementData()   vl53l0x_api.c (ST)
         │   └─ VL53L0X_ReadMulti(Dev, 0x14, buf, 12)
         │       └─ i2c_read()                    vl53l0x_platform.c
         │           └─ HAL_I2C_Mem_Read(Dev->I2cHandle, Dev->I2cDevAddr, 0x14, ...)
         └─ VL53L0X_ClearInterruptMask()           ghi 0x0B, đọc lại 0x13
```

### 3.5. Struct thiết bị của ST: `VL53L0X_Dev_t`

```c
typedef struct {
    VL53L0X_DevData_t   Data;            /* toàn bộ trạng thái nội bộ của ST (~390 byte) */
    uint8_t             I2cDevAddr;      /* địa chỉ 8-bit của CHÍNH cảm biến này */
    uint8_t             comms_type;      /* giữ để tương thích, không dùng */
    uint16_t            comms_speed_khz; /* giữ để tương thích, không dùng */
    I2C_HandleTypeDef  *I2cHandle;       /* thêm mới: bus I2C mà cảm biến nằm trên */
} VL53L0X_Dev_t;
```

**Ý tưởng then chốt:** mọi hàm của ST nhận tham số đầu tiên là `VL53L0X_DEV Dev` (con trỏ tới struct này). Mọi trạng thái nằm trong `Dev->Data`, không nằm trong biến toàn cục. Mình đã kiểm tra: các biến toàn cục duy nhất của lõi ST là 3 bảng hằng chỉ đọc. Do đó:

> **N cảm biến = N struct `VL53L0X_Dev_t`, dùng chung một bộ code.**

### 3.6. Dữ liệu của `tof_multi`

```
TOF_Bus  bus ───────────────┐
 ├─ hi2c      → &hi2c1      │   người dùng điền
 ├─ sensor    → tof[ ]  ────┼──► TOF_Sensor tof[0]: xshut, gpio1, addr, calib   (người dùng điền)
 ├─ count, mode, budget_us, │                       dev, state, range_mm, ...   (thư viện ghi)
 │  poll_ms, sigma_check    │    TOF_Sensor tof[1]: ...
 └─ active (nội bộ)         │
```

### Tóm tắt chương 3
- 3 lớp: `tof_multi` (quản lý), lõi ST (thuật toán), `platform` (HAL).
- Lõi ST chép nguyên, chỉ vá 2 lỗi có đánh dấu.
- Mỗi cảm biến có một `VL53L0X_Dev_t` riêng, nên chạy được nhiều cảm biến.

### Câu hỏi nhanh
<details><summary>1. Vì sao không được include vl53l0x_tuning.h trong một file .c thứ hai?</summary>

Vì file đó **định nghĩa** (không chỉ khai báo) mảng `DefaultTuningSettings[]`. Include ở 2 file thì có 2 định nghĩa, và linker báo lỗi "multiple definition".
</details>

<details><summary>2. Muốn chạy thư viện trên STM32F4 thì sửa ở lớp nào?</summary>

Chỉ lớp platform: define `VL53L0X_HAL_HEADER="stm32f4xx_hal.h"`. Hai lớp còn lại không phụ thuộc dòng chip.
</details>

---

## Chương 4 — Nhiều cảm biến trên một bus I2C

### 4.1. Vấn đề

Sau khi cấp nguồn, **mọi VL53L0X đều ở địa chỉ 0x52**. Hai con cùng bật thì cả hai cùng trả lời mọi lệnh, gây xung đột trên bus và cho dữ liệu sai.

**Giải pháp:** đổi địa chỉ cho từng con. Nhưng lệnh đổi địa chỉ cũng gửi tới 0x52, nên **khi gửi lệnh, chỉ được có đúng một con ở 0x52**. Phải có cách tắt các con còn lại: đó là chân **XSHUT**.

### 4.2. Chân XSHUT

- Tích cực mức thấp: thấp = tắt, cao = chạy.
- Module GY-530 có điện trở kéo lên sẵn trên XSHUT. Vì vậy cấu hình chân STM32 là **Output Open-Drain**: kéo xuống để tắt, thả ra để module tự kéo lên.
- Trong CubeMX phải đặt **mức ban đầu Low**. Như vậy ngay sau reset MCU, mọi cảm biến đều tắt.

### 4.3. Thuật toán gán địa chỉ

```
Bước 0:  XSHUT[0..N-1] = THẤP, chờ 2 ms          → không ai ở 0x52
Với i = 0 .. N-1:
  Bước 1: XSHUT[i] = CAO, chờ 2 ms (tBOOT ≤ 1,2 ms)
  Bước 2: đọc thanh ghi 0xC0 ở địa chỉ 0x52, phải = 0xEE (mã của VL53L0X)
  Bước 3: VL53L0X_SetDeviceAddress(dev, addr[i])   → chip i chuyển sang addr[i]
  Bước 4: dev.I2cDevAddr = addr[i]                 → ST KHÔNG tự cập nhật trường này!
  Bước 5: đọc lại 0xC0 ở addr[i], phải = 0xEE      → xác nhận
  Bước 6: khởi tạo đầy đủ chip i (chương 5)
  Nếu bước 2–5 lỗi: XSHUT[i] = THẤP                → tắt lại để không chiếm 0x52
```

**Diễn biến với 2 cảm biến:**

| Thời điểm | XSHUT1 | XSHUT2 | Ai đang ở 0x52 | Ai ở 0x54 | Ai ở 0x56 |
|---|---|---|---|---|---|
| Sau reset | thấp | thấp | — | — | — |
| Bật TOF1 | **cao** | thấp | TOF1 | — | — |
| Đổi địa chỉ TOF1 | cao | thấp | — | TOF1 | — |
| Bật TOF2 | cao | **cao** | TOF2 | TOF1 | — |
| Đổi địa chỉ TOF2 | cao | cao | — | TOF1 | TOF2 |

### 4.4. Ba quy tắc bắt buộc

| # | Quy tắc | Vi phạm thì sao |
|---|---|---|
| 1 | Địa chỉ mới **mất khi mất điện hoặc kéo XSHUT** → phải gán lại mỗi lần khởi động | Sau reset, mọi con lại về 0x52 và xung đột |
| 2 | **Không con nào được giữ 0x52** khi đang chạy | Khi restart một con khác (nó sẽ lên lại ở 0x52), hai con trùng địa chỉ |
| 3 | Con nào lỗi khi gán địa chỉ **phải giữ XSHUT thấp** | Nó tiếp tục chiếm 0x52, làm hỏng việc gán địa chỉ cho con sau |

> 💡 **Bất biến của thuật toán:** *tại mọi thời điểm, có nhiều nhất một thiết bị ở 0x52.* Ba quy tắc trên chính là điều kiện để bất biến này luôn đúng. Phần chứng minh ở [Q7 chuyên đề 2].

### 4.5. Chọn địa chỉ

Hàm `tof_addr_valid()` chỉ chấp nhận địa chỉ thoả **cả 4 điều kiện**:
1. **Chẵn** (dạng 8-bit có bit 0 = 0).
2. Trong khoảng **0x10–0xEE** (tránh các vùng địa chỉ dành riêng của I2C).
3. **Khác 0x52** (quy tắc 2).
4. **Không trùng** với cảm biến khác trong cùng `TOF_Bus`.

Ngoài ra **bạn phải tự tránh** địa chỉ của các thiết bị khác trên bus: **0x4E** (PCF8574) và **0x7E** (PCF8574A). Gợi ý dùng: 0x54, 0x56, 0x58, 0x5A…

### Tóm tắt chương 4
- Mỗi cảm biến cần **một chân XSHUT riêng** (Output Open-Drain, mức ban đầu Low).
- Bật từng con → đổi địa chỉ → cập nhật `I2cDevAddr` → xác nhận → khởi tạo → bật con tiếp theo.
- 3 quy tắc: gán lại mỗi lần khởi động; không ai giữ 0x52; con lỗi phải giữ XSHUT thấp.

### Câu hỏi nhanh
<details><summary>1. Quên nối XSHUT của TOF2 (chân bỏ trống, module tự kéo lên). Chuyện gì xảy ra?</summary>

TOF2 luôn chạy ở 0x52 ngay từ đầu. Khi bật TOF1, cả hai cùng ở 0x52 và cùng nhận lệnh đổi địa chỉ, nên **cả hai cùng sang 0x54**. Thư viện tưởng TOF1 thành công, còn TOF2 báo lỗi −20 vì 0x52 không còn ai. Từ đó số đo của "TOF1" là dữ liệu trộn của 2 chip. Dấu hiệu nhận biết: lần quét I2C **trước** init đã thấy 0x52. Xem [Q4 bài D1].
</details>

<details><summary>2. Vì sao địa chỉ 0x55 bị từ chối?</summary>

Vì là số lẻ. Địa chỉ 8-bit phải có bit 0 = 0, vì bit đó dành cho R/W.
</details>

---

## Chương 5 — Khởi tạo cảm biến theo chuẩn của ST

### 5.1. Máy trạng thái của API ST

```
POWERDOWN ──DataInit──► WAIT_STATICINIT ──StaticInit──► IDLE ──StartMeasurement──► RUNNING
                                                          ▲                          │
                                                          └──────StopMeasurement─────┘
```
Gọi sai thứ tự (ví dụ `StaticInit` khi chưa `DataInit`) không phải lúc nào cũng báo lỗi, nhưng kết quả sẽ sai.

### 5.2. Từng bước trong `tof_setup()` và lý do

| # | Hàm ST | Làm gì | Vì sao |
|---|---|---|---|
| 1 | `VL53L0X_DataInit` | Đọc `StopVariable`; bật I/O 2V8 (nếu define `USE_I2C_2V8`); bật sẵn kiểm tra sigma (18 mm) và signal rate (0,25 MCPS) | **Chỉ gọi 1 lần sau mỗi lần reset chip.** XSHUT bảo đảm điều này |
| 2 | `VL53L0X_StaticInit` | Đọc NVM, nạp 80 lệnh cấu hình tuning, đặt GPIO1 = "có mẫu mới", tích cực mức thấp | Đưa chip về cấu hình chuẩn của ST |
| 3 | `PerformRefSpadManagement` **hoặc** `SetReferenceSpads` | Chọn số SPAD tham chiếu phù hợp | Có dữ liệu đã lưu (`calib.valid = 1`) thì nạp lại, nhanh hơn |
| 4 | `VL53L0X_PerformRefCalibration` | Hiệu chuẩn VHV và phase | **Phụ thuộc nhiệt độ** nên chạy lại mỗi lần khởi động |
| 5 | `SetOffsetCalibrationDataMicroMeter` **hoặc** `Get...` | Nạp offset đã lưu, hoặc đọc offset của nhà máy | |
| 6 | `VL53L0X_SetDeviceMode` | CONTINUOUS (song song) hoặc SINGLE (tuần tự) | ⚠️ **Phải gọi sau bước 3**, vì `PerformRefSpadManagement` tự đổi chế độ về SINGLE |
| 7 | `SetMeasurementTimingBudgetMicroSeconds` | Đặt timing budget | |
| 8 | `SetLimitCheckEnable(SIGMA, 0)` | Tắt kiểm tra sigma (khi `sigma_check = 0`) | Tiết kiệm ~19 giao dịch I2C mỗi lần đọc (chương 6.6) |
| 9 | `VL53L0X_StartMeasurement` | Bắt đầu đo liên tục (chỉ ở chế độ song song) | Chế độ tuần tự do `TOF_Poll` tự bật từng con |

Thứ tự SPAD → VHV/phase → offset theo tài liệu UM2039 của ST.

### 5.3. Thời gian khởi tạo

🔬 Giả lập: ~0,22 s mỗi cảm biến (2 cảm biến: 449 ms; 4 cảm biến: 894 ms). Khi đã có `calib.valid = 1`: 2 cảm biến mất 343 ms. Phần lớn thời gian dành cho các lần đo nội bộ khi hiệu chuẩn SPAD và VHV/phase. Mạch thật có thể khác.

### Tóm tắt chương 5
- Thứ tự: DataInit → StaticInit → SPAD → VHV/phase → offset → SetDeviceMode → budget → (tắt sigma) → Start.
- `SetDeviceMode` phải sau `PerformRefSpadManagement`.
- `DataInit` chỉ 1 lần sau mỗi lần reset chip.

### Câu hỏi nhanh
<details><summary>1. Vì sao lần nào khởi động thư viện cũng chạy lại PerformRefCalibration, kể cả khi đã có calib?</summary>

Vì VHV và phase phụ thuộc nhiệt độ. Giá trị đo lúc 25 °C có thể không đúng lúc 45 °C. Hàm này chạy nhanh nên chạy lại là an toàn nhất.
</details>

---

## Chương 6 — Đo không chặn CPU

### 6.1. Vì sao phải không chặn

Hàm đo một lần của ST (`PerformSingleRangingMeasurement`) **đứng chờ ~35 ms** mỗi lần gọi. Trong lúc đó CPU không làm được gì khác: LCD không cập nhật, HX711 không đọc, nút nhấn không phản hồi.

Thư viện tách việc đo thành 2 nửa:
- **Cảm biến tự đo** (chế độ continuous), CPU không phải chờ.
- **CPU chỉ đọc khi đã có số đo mới**, trong hàm `TOF_Poll()` gọi liên tục từ vòng lặp chính.

### 6.2. Cách biết "có số đo mới"

| Cách | Chi phí | Khi nào dùng |
|---|---|---|
| **GPIO1** xuống thấp | Đọc 1 chân GPIO, ~0 µs, **không tốn I2C** | Có nối GPIO1 (`gpio1_port != NULL`) |
| `VL53L0X_GetMeasurementDataReady` | 1 lần đọc I2C, ~0,4 ms | Không nối GPIO1 |

### 6.3. Lịch hỏi qua I2C (khi không có GPIO1)

Hỏi liên tục thì tốn bus vô ích, vì số đo chưa thể xong sớm. Thư viện tính lịch riêng cho từng cảm biến:

```
           bắt đầu đo / vừa đọc xong
                 │
                 │◄──── first_check = 3/4 × budget ────►│ hỏi │ hỏi │ ... (mỗi poll_ms)
                 ▼                                      ▼     ▼
  thời gian ─────┴──────────────────────────────────────┴─────┴──────►
```
- `first_check_ms = budget_us × 3 / 4000`. Với budget 33 ms → **24 ms**.
- Sau đó hỏi mỗi `poll_ms` (mặc định 10 ms) cho tới khi có dữ liệu.

### 6.4. Giới hạn thời gian của một lần `TOF_Poll`

Đọc một cảm biến tốn ~2,5–3 ms ở 100 kHz. Nếu một lần `TOF_Poll` đọc cả 4 cảm biến thì CPU bị giữ tới ~11 ms. Vì vậy thư viện:
- Chỉ đọc **tối đa `TOF_MAX_READS_PER_POLL` (= 1) cảm biến** mỗi lần gọi.
- **Xoay vòng**: lần sau bắt đầu từ cảm biến kế tiếp, để không con nào bị bỏ đói.

🔬 Với 4 cảm biến song song: một lần `TOF_Poll` giữ CPU **tối đa 3,1 ms** (hỏi qua I2C) hoặc **2,6 ms** (có GPIO1).

### 6.5. Song song và tuần tự

| | `TOF_MODE_PARALLEL` | `TOF_MODE_SEQUENTIAL` |
|---|---|---|
| Cách chạy | Mọi cảm biến đo liên tục cùng lúc | Lần lượt: con 0 đo xong → đọc → bật con 1 → ... |
| Tốc độ mỗi con | ≈ 1 / budget ≈ **30 Hz** | ≈ 1 / (N × chu kỳ) |
| Nhiễu quang | Có thể có nếu vùng nhìn chồng nhau | **Không**: mỗi lúc chỉ một con phát tia |
| Khi nào dùng | Cảm biến nhìn các hướng khác nhau | Cảm biến đặt sát nhau, cùng hướng |

**Chu kỳ một lượt ở chế độ tuần tự** ≈ budget + thời gian đọc (~2,6 ms) + thời gian bật đo (~3,5 ms) + độ trễ phát hiện. Với budget 33 ms, chu kỳ ≈ 39–40 ms, nên tổng ~25 lần đo/giây chia cho N cảm biến. 🔬 Giả lập (đo 30 ms/lần): 2 cảm biến ~14–15 Hz mỗi con; 4 cảm biến ~7 Hz mỗi con.

### 6.6. Chi phí I2C của một lần đọc và tuỳ chọn `sigma_check`

Khi đếm từng giao dịch I2C, hàm `VL53L0X_GetRangingMeasurementData` của ST tốn:

| `sigma_check` | Giao dịch I2C mỗi lần đọc (kể cả xoá cờ ngắt) | Thời gian @100 kHz |
|---|---|---|
| **0** (mặc định thư viện) | **4**: đọc 12 byte kết quả, 2 lần ghi xoá cờ, 1 lần đọc kiểm tra | **~2,5–3 ms** |
| 1 (mặc định của ST) | **23**: thêm ~19 lần đọc thanh ghi để ước lượng sigma | ~10–12 ms |

Lý do: khi bật sigma, hàm ước lượng sigma của ST **đọc lại từ chip** timing budget, chu kỳ VCSEL và các giới hạn **mỗi lần đo**. Tắt sigma thì mất trạng thái "Sigma fail" (RangeStatus = 1), các trạng thái khác vẫn còn.

### 6.7. Phát hiện cảm biến ngừng trả dữ liệu

Mỗi cảm biến lưu `start_ms`, là lúc bắt đầu đo hoặc lúc có số đo gần nhất. Nếu quá `stale_ms` mà chưa có số đo mới thì tính là **1 lỗi** (`VL53L0X_ERROR_TIME_OUT`, −7):

$$\text{stale\_ms} = \frac{\text{budget\_us}}{1000} \times \text{TOF\_STALE\_FACTOR} + 10 = 33 \times 4 + 10 = 142\ \text{ms}$$

Sau **`TOF_MAX_ERRORS` = 5 lỗi liên tiếp**, cảm biến chuyển sang `TOF_STATE_ERROR`. Chỉ cần đọc thành công một lần là bộ đếm lỗi về 0.

### Tóm tắt chương 6
- Cảm biến tự đo; CPU chỉ đọc khi có dữ liệu mới (`TOF_Poll`).
- GPIO1 miễn phí; hỏi qua I2C thì chờ 3/4 budget rồi hỏi mỗi `poll_ms`.
- Mỗi lần `TOF_Poll` đọc tối đa 1 cảm biến, xoay vòng giữa các cảm biến.
- Song song ~30 Hz mỗi con; tuần tự chống nhiễu nhưng tốc độ chia cho N.
- `sigma_check = 0`: 4 giao dịch I2C; `sigma_check = 1`: 23 giao dịch.

### Câu hỏi nhanh
<details><summary>1. Budget 20 ms thì first_check và stale_ms là bao nhiêu?</summary>

first_check = 20000 × 3 / 4000 = **15 ms**; stale = 20 × 4 + 10 = **90 ms**.
</details>

<details><summary>2. Vì sao chân GPIO1 bị đứt dây lại làm cảm biến báo lỗi −7 chứ không phải −20?</summary>

Dây đứt thì điện trở kéo lên giữ chân GPIO1 ở mức cao, nên thư viện tưởng "chưa có dữ liệu". I2C vẫn chạy tốt (không có −20), nhưng không bao giờ đọc được số đo. Quá `stale_ms` thì tính lỗi TIME_OUT (−7); sau 5 lần thì cảm biến chuyển sang ERROR.
</details>

---

## Chương 7 — Định dạng số và đơn vị

Các vi điều khiển không có bộ xử lý số thực (FPU), như STM32F1 (Cortex-M3), tính `float` rất chậm. Vì vậy ST dùng **số dấu phẩy cố định** (*fixed-point*): lưu số nguyên, và quy ước một số bit thấp nhất là phần thập phân.

| Định dạng | Số bit thập phân | Giá trị thực | Dùng cho |
|---|---|---|---|
| **16.16** (`FixPoint1616_t`) | 16 | `x / 65536` | Signal rate, giới hạn sigma, khoảng cách hiệu chuẩn, ngưỡng ngắt |
| **9.7** | 7 | `x / 128` | Thanh ghi signal rate (0x44, 0x1A…) |
| **8.8** | 8 | `x / 256` | `EffectiveSpadRtnCount` |
| **4.12** | 12 | `x / 4096` | Tần số dao động nội |
| **10.2** (12 bit có dấu) | 2 | `x / 4` mm | Thanh ghi offset 0x28 (bước 0,25 mm) |

### Ví dụ chuyển đổi

| Bài toán | Phép tính | Kết quả |
|---|---|---|
| 0,25 MCPS → 16.16 | 0,25 × 65536 | 16384 = 0x4000 |
| 0,25 MCPS → 9.7 | 0,25 × 128 | 32 = 0x0020 |
| Sigma 18 mm → 16.16 | 18 × 65536 | 1 179 648 |
| 100 mm → 16.16 (tham số hiệu chuẩn) | `100 << 16` | 6 553 600 |
| Thanh ghi signal 0x0280 (9.7) → MCPS | 640 / 128 | 5,0 MCPS |
| SPAD 0x0A80 (8.8) → số SPAD | 2688 / 256 | 10,5 |

### Thanh ghi offset (10.2, 12 bit có dấu)

ST đổi offset (µm) sang giá trị thanh ghi như sau:
- Offset ≥ 0: `giá_trị = offset_um / 250`.
- Offset < 0: `giá_trị = 4096 + offset_um / 250` (bù hai trên 12 bit).

| Offset | Giá trị thanh ghi |
|---|---|
| +2,5 mm (2500 µm) | 10 = 0x00A |
| −7 mm (−7000 µm) | 4096 − 28 = 4068 = **0xFE4** (🔬 khớp với giá trị đọc trong kiểm thử) |
| Giới hạn | −512 mm … +511 mm |

### Tóm tắt chương 7
- 16.16 chia 65536; 9.7 chia 128; 8.8 chia 256; offset 10.2 có bước 0,25 mm.
- Khoảng cách hiệu chuẩn truyền vào ST phải là 16.16: `mm << 16`.

### Câu hỏi nhanh
<details><summary>Thanh ghi offset đọc được 0xFF6. Offset là bao nhiêu?</summary>

0xFF6 = 4086 > 2047 nên là số âm: 4086 − 4096 = −10 → −10 × 250 = **−2500 µm = −2,5 mm**.
</details>

---

## Chương 8 — Chất lượng số đo: RangeStatus

Bit 3–6 của thanh ghi 0x14 là **mã trạng thái thô** của chip. API của ST đổi mã thô sang `RangeStatus` chuẩn (hàm `VL53L0X_get_pal_range_status`):

| `RangeStatus` | Ý nghĩa | Mã thô của chip `(reg & 0x78) >> 3` | Xử lý gợi ý |
|---|---|---|---|
| **0** | Hợp lệ | 11 | Dùng `range_mm` |
| 1 | Sigma fail: nhiễu cao | 11 + sigma > giới hạn | Không chắc chắn; chỉ xuất hiện khi `sigma_check = 1` |
| 2 | Signal fail: tín hiệu yếu | 4 | Thường là **không có vật** hoặc vật quá tối, quá xa |
| 3 | Min range: vật quá gần | 8, 10 | Vật sát cảm biến |
| 4 | Phase fail | 6, 9 | Thường là **ngoài tầm** |
| 5 | Lỗi phần cứng | 1, 2, 3 | Kiểm tra cảm biến |
| 255 | Chưa có số đo | 0, 5, 7, 12–15 | Chờ |

⚠️ **Không bao giờ dùng `range_mm` khi `range_status != 0`.** Khi không có vật, chip vẫn trả về một con số (thường 8190 hoặc 8191). Con số đó không phải khoảng cách.

Hàm `TOF_GetRange()` đã kiểm tra sẵn 3 điều: cảm biến đang RUNNING, `range_status == 0`, và số đo chưa quá cũ.

### Câu hỏi nhanh
<details><summary>Đưa tay ra xa dần, range_status đổi từ 0 sang 2. Có phải cảm biến hỏng?</summary>

Không. Tay ra khỏi tầm đo nên tín hiệu phản xạ quá yếu (Signal fail). Đây là hành vi bình thường.
</details>

---

## Chương 9 — Hiệu chuẩn

| Loại | Thực hiện | Khi nào | Trong `tof_multi` |
|---|---|---|---|
| **SPAD tham chiếu** | `PerformRefSpadManagement` (tự động, không cần vật mẫu) | Lần đầu; khi có kính che | Tự làm trong `TOF_Init`; lưu vào `calib.refSpadCount`, `calib.isApertureSpads` |
| **VHV / phase** | `PerformRefCalibration` (tự động) | Mỗi lần khởi động; khi nhiệt độ thay đổi > ~8 °C | Tự làm mỗi lần `TOF_Init` / `TOF_Restart` |
| **Offset** | `PerformOffsetCalibration` với tấm trắng ở khoảng cách biết trước (ST khuyên 100 mm) | Một lần khi sản xuất; khi số đo lệch cố định | `TOF_CalibrateOffset(&bus, i, 100)` → `calib.offset_um` |
| **Crosstalk** | `PerformXTalkCalibration` | **Chỉ khi có kính/mica che phía trước** | Không có sẵn; gọi API ST trên `&tof[i].dev` |

### Quy trình hiệu chuẩn offset

1. Gắn cảm biến cố định. Đặt **tấm trắng** (giấy trắng, độ phản xạ ~88%) vuông góc, cách mặt cảm biến **đúng 100 mm**.
2. Tránh nắng trực tiếp; trong vùng nhìn không được có vật khác.
3. Gọi `TOF_CalibrateOffset(&bus, i, 100)`. Hàm **chặn CPU ~2 s** (50 lần đo).
4. Kết quả: `tof[i].calib.offset_um` = 1000 × (khoảng cách thật − trung bình đo được). `calib.valid = 1`.
5. **Tự lưu** `tof[i].calib` vào flash (ví dụ VD8, quyển 3).
6. Lần khởi động sau: nạp `calib` từ flash **trước** `TOF_Init`. Thư viện bỏ qua bước đo SPAD và nạp offset.

🔬 Giả lập: đặt vật đúng 100 mm, chip đọc 107 mm → `offset_um = −7000`; thanh ghi 0x28 = 0xFE4.

### Tóm tắt chương 9
- SPAD và VHV/phase: tự động. Offset: cần tấm trắng ở 100 mm. Crosstalk: chỉ khi có kính che.
- Hiệu chuẩn nằm trong RAM; muốn giữ phải tự lưu vào flash.

---

## Chương 10 — Lỗi, phục hồi và độ tin cậy

### 10.1. Mã lỗi (`VL53L0X_Error`, kiểu `int8_t`)

| Mã | Tên | Nguyên nhân thường gặp |
|---|---|---|
| 0 | `VL53L0X_ERROR_NONE` | Thành công |
| −4 | `INVALID_PARAMS` | Địa chỉ cấu hình sai (lẻ, trùng, là 0x52…), chỉ số cảm biến sai |
| −7 | `TIME_OUT` | Quá lâu không có số đo (chân GPIO1 đứt, cảm biến treo) |
| −12 | `INTERRUPT_NOT_CLEARED` | Xoá cờ ngắt 3 lần không được |
| −20 | `CONTROL_INTERFACE` | **Không trả lời I2C**: đứt dây, mất nguồn, XSHUT thấp, sai địa chỉ, ID ≠ 0xEE |
| −30 | `INVALID_COMMAND` | Gọi sai trạng thái (ví dụ hiệu chuẩn khi cảm biến không RUNNING) |
| −50 | `REF_SPAD_INIT` | Hiệu chuẩn SPAD thất bại (có vật che, miếng dán bảo vệ…) |

### 10.2. Trạng thái của một cảm biến trong `tof_multi`

```
           TOF_Init / TOF_Restart thành công
  OFF ─────────────────────────────────────────► RUNNING
   ▲                                            │   ▲
   │ TOF_Restart (bắt đầu)                      │   │ đọc OK → err_count = 0
   │                                            ▼   │
  ERROR ◄──── 5 lỗi liên tiếp (I2C, TIME_OUT) ── RUNNING
   │
   └── gán địa chỉ / khởi tạo lỗi (khi đó XSHUT bị giữ thấp)
```

### 10.3. Các lớp bảo vệ

| Lớp | Cơ chế | Giá trị |
|---|---|---|
| Platform | Timeout mỗi lệnh HAL | 10 ms (`VL53L0X_I2C_TIMEOUT_MS`) |
| Platform | Thử lại khi lỗi | 1 lần (`VL53L0X_I2C_RETRIES`) |
| Lõi ST | Số vòng chờ tối đa | 2000 (`VL53L0X_DEFAULT_MAX_LOOP`) × (1 lần đọc + `PollingDelay` 1 ms) ≈ 2–5 s |
| `tof_multi` | Phát hiện mất dữ liệu | `stale_ms` = 142 ms (budget 33 ms) |
| `tof_multi` | Đánh dấu hỏng | 5 lỗi liên tiếp |
| Ứng dụng | Tự khởi động lại | `TOF_Restart` mỗi giây nếu ERROR |
| Ứng dụng | Phục hồi bus | `TOF_I2C_Recover` khi **tất cả** cảm biến cùng lỗi |

### 10.4. Quy tắc an toàn khi dùng

1. **Không gọi hàm của thư viện trong ngắt hoặc callback.** Các hàm có `HAL_Delay` và dùng chung struct.
2. `TOF_Restart` và `TOF_CalibrateOffset` **chặn CPU** (0,2 s và ~2 s). Chỉ gọi khi chấp nhận được.
3. Sau `TOF_Restart`, mọi cấu hình bạn tự đặt bằng API ST (profile, VCSEL…) **bị mất**, vì thư viện khởi tạo lại từ đầu. Phải đặt lại (VD10, quyển 3).

### Tóm tắt chương 10
- −20 = không trả lời I2C; −7 = không có số đo; −4 = cấu hình sai.
- OFF → RUNNING → ERROR (5 lỗi liên tiếp) → `TOF_Restart`.
- Không gọi thư viện trong ngắt; Restart làm mất cấu hình riêng.

---

## Chương 11 — Phần cứng, công cụ và tài nguyên

### 11.1. Sơ đồ nối 2 cảm biến (Blue Pill)

| Chân module | TOF1 | TOF2 | Blue Pill |
|---|---|---|---|
| VIN / GND | ● | ● | 3.3 / G |
| SCL / SDA | ● | ● | PB6 / PB7 (chung) |
| XSHUT | PA0 | PA1 | Mỗi con một chân |
| GPIO1 | PB0 | PB1 | Không bắt buộc |

ST-Link: 3.3 / G / PA13 (SWDIO) / PA14 (SWCLK). Muốn in qua UART thì thêm USB-TTL vào PA9.

### 11.2. Cấu hình CubeMX (tóm tắt)

| Mục | Thiết lập |
|---|---|
| SYS | Debug = **Serial Wire** |
| RCC + Clock | HSE crystal 8 MHz → PLL ×9 → **72 MHz**, APB1 /2 |
| I2C1 | Standard mode, **100 kHz** |
| GPIO | XSHUT: Output **Open-Drain**, mức **Low**. GPIO1: Input **Pull-up** |
| Project | MDK-ARM, Stack **0x800** |

### 11.3. Cấu hình Keil (tóm tắt)

| Mục | Thiết lập |
|---|---|
| Group `VL53L0X` | 5 file `st_core/*.c` + `platform/vl53l0x_platform.c` + `tof/tof_multi.c` + **một** trong hai file `app/` |
| Include paths | `st_core`, `platform`, `tof`, `app` |
| Define | `USE_HAL_DRIVER,STM32F103xB,USE_I2C_2V8` |
| Ngôn ngữ | **C99**, bắt buộc vì lõi ST dùng macro variadic |
| MicroLIB | Chỉ cần với bản in qua UART |
| Debug | ST-Link, SW, tick Debug Information |

### 11.4. Tài nguyên

| Tài nguyên | Mức dùng |
|---|---|
| RAM | ~470 byte mỗi `TOF_Sensor` + ~700 byte 2 bảng cấu hình của ST |
| Flash | Lõi ST ước tính ~25–30 KB. Xem số chính xác ở dòng `Program Size` của Keil |
| ⚠️ Keil bản Lite | Giới hạn **32 KB code** → dùng `-Oz`/`USE_EMPTY_STRING`, hoặc MDK Community |
| Bus I2C (4 cảm biến, 30 Hz, `sigma_check = 0`) | 🔬 ~35–44% thời gian CPU nằm trong I2C. Giảm bằng GPIO1, budget lớn hơn, hoặc đưa cảm biến sang I2C2 ở 400 kHz |

---

**Hết quyển 1.** Tiếp theo: [Quyển 2 — Sách tham khảo API](02_SachThamKhao_API.md).

# Quyển 2 — SÁCH THAM KHẢO API

> Tra cứu nhanh từng macro, kiểu dữ liệu và hàm. Mỗi hàm được trình bày theo mẫu **"thẻ hàm"**:
> khai báo → chức năng → tham số → giá trị trả về → thời gian chặn CPU → điều kiện trước/sau → hoạt động bên trong → ví dụ → lưu ý.
>
> Thời gian ghi theo bus I2C 100 kHz. 🔬 = số đo trên bộ giả lập.

## Mục lục

- [A. Macro cấu hình](#a-macro-cấu-hình)
- [B. Kiểu dữ liệu của tof_multi](#b-kiểu-dữ-liệu-của-tof_multi)
- [C. Hàm của tof_multi](#c-hàm-của-tof_multi)
- [D. Lớp platform](#d-lớp-platform)
- [E. Các hàm hay dùng của API ST](#e-các-hàm-hay-dùng-của-api-st)
- [F. Chương trình chạy thử (app/)](#f-chương-trình-chạy-thử-app)
- [G. Bảng tra](#g-bảng-tra)

---

## A. Macro cấu hình

Các macro bên dưới có thể định nghĩa lại trong *Options for Target → C/C++ → Define*, hoặc định nghĩa trước khi include header.

### A.1. Của `tof_multi.h`

| Macro | Mặc định | Ý nghĩa | Khi nào đổi |
|---|---|---|---|
| `TOF_DEFAULT_ADDR` | `0x52U` | Địa chỉ 8-bit của VL53L0X sau reset | Không đổi |
| `TOF_BOOT_MS` | `2U` | Chờ sau khi kéo XSHUT lên (datasheet: tBOOT ≤ 1,2 ms) | Tăng lên 5 nếu dây dài, nguồn yếu |
| `TOF_MAX_ERRORS` | `5U` | Số lỗi **liên tiếp** trước khi chuyển sang ERROR | Giảm để phát hiện nhanh hơn; tăng để chịu nhiễu tốt hơn |
| `TOF_MAX_READS_PER_POLL` | `1U` | Số cảm biến đọc tối đa trong một lần `TOF_Poll` | Tăng nếu chấp nhận một lần Poll giữ CPU lâu hơn |
| `TOF_STALE_FACTOR` | `4U` | Quá `budget × hệ số + 10 ms` không có số đo thì tính 1 lỗi | Tăng khi nguồn hoặc bus hay chập chờn |

### A.2. Của lớp platform

| Macro | Mặc định | Ý nghĩa |
|---|---|---|
| `VL53L0X_HAL_HEADER` | `"stm32f1xx_hal.h"` | Header HAL của dòng chip. Ví dụ F4: `VL53L0X_HAL_HEADER="stm32f4xx_hal.h"` |
| `VL53L0X_I2C_TIMEOUT_MS` | `10U` | Timeout mỗi lệnh `HAL_I2C_Mem_Read/Write` |
| `VL53L0X_I2C_RETRIES` | `1U` | Số lần thử lại khi lệnh I2C lỗi. 0 = không thử lại |

### A.3. Của lõi ST

| Macro | Mặc định | Ý nghĩa |
|---|---|---|
| `USE_I2C_2V8` | (không define) | **Nên define** với module có ổn áp 2,8 V. `DataInit` chuyển I/O sang chế độ 2V8 |
| `USE_EMPTY_STRING` | (không define) | Thay các chuỗi mô tả lỗi bằng chuỗi rỗng, tiết kiệm ~1–2 KB flash. Đã vá để biên dịch được |
| `VL53L0X_DEFAULT_MAX_LOOP` | `2000` (`vl53l0x_def.h`) | Số vòng chờ tối đa trong các vòng lặp chờ của ST |
| `VL53L0X_LOG_ENABLE` | (không define) | **Không bật** trên STM32 (cần `sys/time.h` của PC) |

---

## B. Kiểu dữ liệu của tof_multi

### B.1. `TOF_Mode`

| Giá trị | Ý nghĩa |
|---|---|
| `TOF_MODE_PARALLEL` (0) | Mọi cảm biến đo liên tục cùng lúc, ~30 Hz mỗi con |
| `TOF_MODE_SEQUENTIAL` (1) | Đo lần lượt, mỗi lúc một con. Chống nhiễu quang khi vùng nhìn chồng nhau |

### B.2. `TOF_State`

| Giá trị | Ý nghĩa |
|---|---|
| `TOF_STATE_OFF` (0) | Chưa khởi tạo, hoặc đang trong quá trình restart |
| `TOF_STATE_RUNNING` (1) | Đang đo bình thường |
| `TOF_STATE_ERROR` (2) | Lỗi. Xem `last_error`; gọi `TOF_Restart` để thử lại |

### B.3. `TOF_Calib`: dữ liệu hiệu chuẩn, lưu vào flash

| Trường | Kiểu | Ý nghĩa |
|---|---|---|
| `refSpadCount` | `uint32_t` | Số SPAD tham chiếu |
| `isApertureSpads` | `uint8_t` | 1 = loại aperture, 0 = non-aperture |
| `offset_um` | `int32_t` | Offset (micromét) |
| `valid` | `uint8_t` | **1 = dùng dữ liệu này** khi khởi tạo, bỏ qua bước đo SPAD và nạp offset |

`sizeof(TOF_Calib)` = 16 byte, có byte đệm (*padding*).

### B.4. `TOF_Sensor`: một cảm biến

| Trường | Ai ghi | Ý nghĩa |
|---|---|---|
| `xshut_port`, `xshut_pin` | **Người dùng** | Chân XSHUT (bắt buộc) |
| `gpio1_port`, `gpio1_pin` | **Người dùng** | Chân GPIO1; `NULL` nếu không nối |
| `addr` | **Người dùng** | Địa chỉ 8-bit mong muốn (chẵn, 0x10–0xEE, ≠ 0x52, không trùng) |
| `calib` | Người dùng / thư viện | Đặt `calib.valid = 1` để nạp; thư viện điền sau khi khởi tạo hoặc hiệu chuẩn |
| `dev` | Thư viện | `VL53L0X_Dev_t` của ST. Có thể gọi API ST trên `&dev` (mục E) |
| `state` | Thư viện | `TOF_State` |
| `last_error` | Thư viện | Mã lỗi gần nhất (`VL53L0X_Error`) |
| `err_count` | Thư viện | Số lỗi liên tiếp |
| `range_mm` | Thư viện | Khoảng cách gần nhất (mm) |
| `range_status` | Thư viện | 0 = hợp lệ; 255 = chưa có số đo (bảng G.2) |
| `timestamp_ms` | Thư viện | `HAL_GetTick()` lúc có số đo gần nhất |
| `start_ms`, `next_ms` | Thư viện (nội bộ) | Mốc bắt đầu đo / lần hỏi I2C tiếp theo |

🔬 Kích thước ~470 byte trên ARM, phần lớn là `dev`. **Luôn khai báo là biến toàn cục hoặc `static`.**

### B.5. `TOF_Bus`: một nhóm cảm biến trên một bus

| Trường | Ai ghi | Mặc định khi = 0 | Ý nghĩa |
|---|---|---|---|
| `hi2c` | **Người dùng** | — | `&hi2c1`... |
| `sensor` | **Người dùng** | — | Con trỏ tới mảng `TOF_Sensor` |
| `count` | **Người dùng** | — | Số cảm biến, 1–32 |
| `mode` | **Người dùng** | `TOF_MODE_PARALLEL` | Chế độ đo |
| `budget_us` | Người dùng | **33000** | Timing budget (20000–200000) |
| `poll_ms` | Người dùng | **10** | Chu kỳ hỏi qua I2C khi không có GPIO1 |
| `sigma_check` | Người dùng | **0** (tắt) | 1 = giữ kiểm tra sigma của ST (chậm hơn ~4 lần) |
| `active` | Thư viện | — | Nội bộ: cảm biến đang đo (tuần tự) / vừa đọc (song song) |

---

## C. Hàm của tof_multi

### C.1. `TOF_Init`

```c
uint8_t TOF_Init(TOF_Bus *bus);
```

| Mục | Nội dung |
|---|---|
| **Chức năng** | Gán địa chỉ và khởi tạo toàn bộ cảm biến trong `bus` |
| **Tham số** | `bus`: đã điền `hi2c`, `sensor`, `count` (1–32), `mode`; mỗi `sensor[i]` đã điền xshut, gpio1, addr |
| **Trả về** | Số cảm biến chạy được (0…count). Trả 0 nếu `bus` sai (NULL, count = 0 hoặc > 32) |
| **Chặn CPU** | ~0,2–0,3 s mỗi cảm biến (🔬 2 con: 449 ms) |
| **Điều kiện trước** | Đã gọi `MX_GPIO_Init()` và `MX_I2C1_Init()` |
| **Điều kiện sau** | Mỗi `sensor[i].state` là `RUNNING` hoặc `ERROR` (kèm `last_error`). Cảm biến lỗi bị giữ XSHUT thấp |

**Hoạt động bên trong:**
1. Kéo mọi XSHUT xuống thấp, chờ `TOF_BOOT_MS`.
2. Với từng cảm biến `i`:
   - Kiểm tra địa chỉ bằng `tof_addr_valid`. Sai thì đánh dấu ERROR, lỗi −4.
   - Gọi `tof_bringup`:
     - `tof_wake_and_address`: bật XSHUT, đọc ID, `SetDeviceAddress`, cập nhật `I2cDevAddr`, đọc lại ID.
     - `tof_setup`: trình tự khởi tạo ST (Q1 §5.2).
     - `StartMeasurement` nếu ở chế độ song song.
3. Chế độ tuần tự: bật đo cho cảm biến đầu tiên chạy được.

**Ví dụ:**
```c
uint8_t n = TOF_Init(&bus);
if (n < bus.count) {
    for (uint8_t i = 0; i < bus.count; i++)
        if (tof[i].state != TOF_STATE_RUNNING) { /* báo lỗi tof[i].last_error */ }
}
```
⚠️ **Lưu ý:** gọi lại `TOF_Init` được (ví dụ khi đổi chế độ đo); hàm reset toàn bộ cảm biến. `calib` được giữ nguyên.

---

### C.2. `TOF_Poll`

```c
uint32_t TOF_Poll(TOF_Bus *bus);
```

| Mục | Nội dung |
|---|---|
| **Chức năng** | Kiểm tra và đọc số đo mới. **Gọi liên tục trong `while(1)`** |
| **Trả về** | Bitmask: **bit i = 1** nếu cảm biến `i` vừa có số đo mới (đọc trong `sensor[i].range_mm`, `range_status`) |
| **Chặn CPU** | 🔬 Song song: tối đa ~3 ms; tuần tự: tối đa ~6 ms (đọc + bật lượt kế). Phần lớn các lần gọi gần như không tốn thời gian |
| **Điều kiện** | Đã gọi `TOF_Init`. **Không gọi trong ngắt** |

**Hoạt động bên trong:**
- **Song song:**
  - Duyệt xoay vòng, bắt đầu từ cảm biến sau con vừa đọc.
  - Bỏ qua cảm biến không RUNNING, hoặc chưa tới lịch hỏi (khi không có GPIO1).
  - Gọi `tof_service`; dừng sau `TOF_MAX_READS_PER_POLL` lần có kết quả.
- **Tuần tự:**
  - Chỉ xem cảm biến đang tới lượt.
  - Có số đo, hoặc lỗi/quá hạn, thì chuyển lượt (`tof_seq_next`: `StartMeasurement` con kế tiếp).
- **`tof_service`** (cả hai chế độ):
  - Kiểm tra dữ liệu sẵn sàng (GPIO1 hoặc I2C).
  - Có dữ liệu thì `GetRangingMeasurementData` rồi `ClearInterruptMask`.
  - Không có thì kiểm tra quá hạn (`stale_ms`).

**Ví dụ:**
```c
uint32_t upd = TOF_Poll(&bus);
if (upd & (1UL << 0)) { /* cảm biến 0 có số đo mới */ }
```
⚠️ Bitmask chỉ báo **có số đo mới**, kể cả số đo **không hợp lệ**. Vẫn phải kiểm tra `range_status == 0`, hoặc dùng `TOF_GetRange`.

---

### C.3. `TOF_GetRange`

```c
uint8_t TOF_GetRange(const TOF_Bus *bus, uint8_t i, uint32_t max_age_ms, uint16_t *mm);
```

| Mục | Nội dung |
|---|---|
| **Chức năng** | Lấy khoảng cách **hợp lệ và còn mới** của cảm biến `i` |
| **Tham số** | `i`: chỉ số cảm biến; `max_age_ms`: tuổi tối đa của số đo (0 = không kiểm tra tuổi); `mm`: nơi ghi kết quả |
| **Trả về** | 1 nếu thoả **cả ba**: RUNNING, `range_status == 0`, tuổi ≤ `max_age_ms`. Ngược lại trả 0 và **không ghi** `*mm` |
| **Chặn CPU** | Không (không truy cập I2C) |

**Ví dụ:**
```c
uint16_t mm;
if (TOF_GetRange(&bus, 1, 100, &mm)) { /* dùng mm */ }
else                                 { /* không có số đo tin cậy trong 100 ms gần nhất */ }
```
💡 Chọn `max_age_ms` ≈ 3 × budget: 100 ms cho budget 33 ms ở chế độ song song. Ở chế độ tuần tự, nhân thêm số cảm biến.

---

### C.4. `TOF_Restart`

```c
VL53L0X_Error TOF_Restart(TOF_Bus *bus, uint8_t i);
```

| Mục | Nội dung |
|---|---|
| **Chức năng** | Reset riêng cảm biến `i` bằng XSHUT, gán lại địa chỉ, khởi tạo lại. Các cảm biến khác vẫn chạy |
| **Trả về** | `VL53L0X_ERROR_NONE` hoặc mã lỗi; −4 nếu `i` hoặc địa chỉ sai |
| **Chặn CPU** | ~0,2 s nếu cảm biến có mặt; vài ms nếu cảm biến không trả lời |
| **Điều kiện** | Không cảm biến nào khác đang ở 0x52. Điều này luôn đúng khi dùng thư viện đúng cách |

**Hoạt động:** XSHUT thấp → chờ → `tof_bringup` (như `TOF_Init` nhưng chỉ cho một con). Ở chế độ tuần tự, nếu đúng lượt của con này thì chuyển lượt.

⚠️ **Mất cấu hình riêng:** mọi thiết lập bạn tự gọi qua API ST trên `&tof[i].dev` (VCSEL, signal rate, ngắt theo ngưỡng…) bị xoá. Đặt lại sau khi restart.

---

### C.5. `TOF_CalibrateOffset`

```c
VL53L0X_Error TOF_CalibrateOffset(TOF_Bus *bus, uint8_t i, uint16_t target_mm);
```

| Mục | Nội dung |
|---|---|
| **Chức năng** | Hiệu chuẩn offset cho cảm biến `i` với vật ở đúng `target_mm` |
| **Tham số** | `target_mm`: khoảng cách thật tới tấm trắng (ST khuyên 100). Phải > 0 |
| **Trả về** | 0 nếu thành công; −4 nếu tham số sai; −30 nếu cảm biến không RUNNING; −6 (`RANGE_ERROR`) nếu cả 50 lần đo đều không hợp lệ |
| **Chặn CPU** | **~2 s** (50 lần đo × budget) |
| **Điều kiện sau** | `sensor[i].calib.offset_um` mới, `calib.valid = 1`. Cảm biến tiếp tục đo. **Tự lưu `calib` vào flash** |

**Hoạt động:**
1. Dừng đo (song song) và chờ lần đo đang chạy kết thúc.
2. Xoá cờ ngắt.
3. Gọi `VL53L0X_PerformOffsetCalibration(target_mm << 16)`.
4. Đặt lại chế độ đo và chạy tiếp.

---

### C.6. `TOF_I2C_Recover`

```c
void TOF_I2C_Recover(I2C_HandleTypeDef *hi2c, GPIO_TypeDef *port, uint16_t scl_pin, uint16_t sda_pin);
```

| Mục | Nội dung |
|---|---|
| **Chức năng** | Gỡ bus I2C bị treo (slave giữ SDA thấp, cờ BUSY bị kẹt) |
| **Tham số** | Ví dụ `(&hi2c1, GPIOB, GPIO_PIN_6, GPIO_PIN_7)`; SCL và SDA phải cùng port |
| **Chặn CPU** | < 1 ms |
| **Ảnh hưởng** | **Mọi thiết bị trên bus** (LCD…). Sau khi gọi, nên `TOF_Restart` các cảm biến lỗi |

**Hoạt động:**
1. Reset mềm khối I2C (`SWRST`, trên dòng F1/F4).
2. `HAL_I2C_DeInit`.
3. Chuyển SCL/SDA thành GPIO open-drain.
4. Tạo tối đa 9 xung SCL cho tới khi SDA được nhả.
5. Tạo điều kiện STOP.
6. `HAL_I2C_Init`, nên `HAL_I2C_MspInit` đưa chân về chế độ I2C.

---

## D. Lớp platform

Đây là 12 hàm mà lõi ST gọi. Bạn **hiếm khi** gọi trực tiếp, trừ `VL53L0X_RdByte` khi cần đọc nhanh một thanh ghi.

| Hàm | Làm gì | Ghi chú |
|---|---|---|
| `VL53L0X_WrByte(Dev, idx, data)` | Ghi 1 byte | `HAL_I2C_Mem_Write` |
| `VL53L0X_WrWord(Dev, idx, data)` | Ghi 2 byte | **Byte cao trước** |
| `VL53L0X_WrDWord(Dev, idx, data)` | Ghi 4 byte | Byte cao trước |
| `VL53L0X_RdByte(Dev, idx, *data)` | Đọc 1 byte | |
| `VL53L0X_RdWord(Dev, idx, *data)` | Đọc 2 byte | Ghép byte cao trước |
| `VL53L0X_RdDWord(Dev, idx, *data)` | Đọc 4 byte | Ghép byte cao trước |
| `VL53L0X_WriteMulti(Dev, idx, p, n)` | Ghi n byte liên tiếp | `n > 0xFFFF` → −4 |
| `VL53L0X_ReadMulti(Dev, idx, p, n)` | Đọc n byte liên tiếp | |
| `VL53L0X_UpdateByte(Dev, idx, and, or)` | Đọc → `(x & and) \| or` → ghi | Hai giao dịch, **không nguyên tử** |
| `VL53L0X_PollingDelay(Dev)` | `HAL_Delay(1)` | Gọi trong các vòng chờ của ST |
| `VL53L0X_LockSequenceAccess` / `Unlock...` | Rỗng | ST không gọi tới |

**Quy ước lỗi:** HAL trả khác `HAL_OK` (sau `1 + VL53L0X_I2C_RETRIES` lần thử) thì hàm trả **−20** (`VL53L0X_ERROR_CONTROL_INTERFACE`).

**Đọc nhanh một thanh ghi** (ví dụ ID):
```c
uint8_t id;
if (VL53L0X_RdByte(&tof[0].dev, VL53L0X_REG_IDENTIFICATION_MODEL_ID, &id) == 0 && id == 0xEE) { /* đúng là VL53L0X */ }
```

---

## E. Các hàm hay dùng của API ST

Gọi trên `&tof[i].dev` (khi dùng `tof_multi`) hoặc trên một `VL53L0X_Dev_t` tự quản lý. Mọi hàm trả `VL53L0X_Error`.

### E.1. Khởi tạo (`tof_multi` đã gọi sẵn)

| Khai báo | Ghi chú |
|---|---|
| `VL53L0X_DataInit(Dev)` | 1 lần sau mỗi lần reset chip |
| `VL53L0X_StaticInit(Dev)` | Nạp tuning, cấu hình GPIO1 |
| `VL53L0X_PerformRefSpadManagement(Dev, uint32_t *refSpadCount, uint8_t *isApertureSpads)` | ⚠️ Đổi chế độ đo về SINGLE |
| `VL53L0X_SetReferenceSpads(Dev, uint32_t refSpadCount, uint8_t isApertureSpads)` | Nạp SPAD đã lưu |
| `VL53L0X_PerformRefCalibration(Dev, uint8_t *pVhvSettings, uint8_t *pPhaseCal)` | VHV/phase, phụ thuộc nhiệt độ |
| `VL53L0X_SetRefCalibration(Dev, uint8_t Vhv, uint8_t Phase)` / `Get...` | Nạp/đọc VHV/phase |
| `VL53L0X_SetDeviceAddress(Dev, uint8_t DeviceAddress)` | ⚠️ Địa chỉ **8-bit**; **không** cập nhật `Dev->I2cDevAddr` |

### E.2. Cấu hình đo

| Khai báo | Ghi chú |
|---|---|
| `VL53L0X_SetDeviceMode(Dev, VL53L0X_DeviceModes)` | `VL53L0X_DEVICEMODE_SINGLE_RANGING` (0), `_CONTINUOUS_RANGING` (1), `_CONTINUOUS_TIMED_RANGING` (3) |
| `VL53L0X_SetMeasurementTimingBudgetMicroSeconds(Dev, uint32_t)` / `Get...` | 20000–200000 µs |
| `VL53L0X_SetVcselPulsePeriod(Dev, VL53L0X_VcselPeriod, uint8_t)` | PRE_RANGE: 12/14/16/18; FINAL_RANGE: 8/10/12/14. Chu kỳ dài hơn thì tầm xa hơn |
| `VL53L0X_SetLimitCheckEnable(Dev, uint16_t Id, uint8_t Enable)` | Id: `VL53L0X_CHECKENABLE_SIGMA_FINAL_RANGE` (0), `_SIGNAL_RATE_FINAL_RANGE` (1)… |
| `VL53L0X_SetLimitCheckValue(Dev, uint16_t Id, FixPoint1616_t)` / `Get...` | Giá trị dạng 16.16 |
| `VL53L0X_SetInterMeasurementPeriodMilliSeconds(Dev, uint32_t)` | Chỉ dùng cho chế độ CONTINUOUS_TIMED |

### E.3. Đo

| Khai báo | Ghi chú |
|---|---|
| `VL53L0X_StartMeasurement(Dev)` / `StopMeasurement(Dev)` | |
| `VL53L0X_GetMeasurementDataReady(Dev, uint8_t *ready)` | **Không chặn** |
| `VL53L0X_GetRangingMeasurementData(Dev, VL53L0X_RangingMeasurementData_t *)` | 4 hoặc 23 giao dịch I2C, tuỳ sigma |
| `VL53L0X_ClearInterruptMask(Dev, 0)` | **Phải gọi sau mỗi lần đọc** |
| `VL53L0X_PerformSingleRangingMeasurement(Dev, VL53L0X_RangingMeasurementData_t *)` | ⚠️ **Chặn ~35 ms**, tự xoá cờ ngắt. Không dùng chung với `GetMeasurementDataReady` |

### E.4. Hiệu chuẩn

| Khai báo | Ghi chú |
|---|---|
| `VL53L0X_PerformOffsetCalibration(Dev, FixPoint1616_t CalDistanceMilliMeter, int32_t *pOffsetMicroMeter)` | Khoảng cách dạng 16.16 (`mm << 16`). Chặn ~2 s |
| `VL53L0X_SetOffsetCalibrationDataMicroMeter(Dev, int32_t)` / `Get...` | −512000 … +511000 µm |
| `VL53L0X_PerformXTalkCalibration(Dev, FixPoint1616_t XTalkCalDistance, FixPoint1616_t *pRate)` | Chỉ khi có kính che. Đã vá lỗi tràn số |
| `VL53L0X_SetXTalkCompensationRateMegaCps(Dev, FixPoint1616_t)` + `SetXTalkCompensationEnable(Dev, 1)` | Nạp kết quả crosstalk đã lưu |

### E.5. Ngắt GPIO1 và tiện ích

| Khai báo | Ghi chú |
|---|---|
| `VL53L0X_SetGpioConfig(Dev, uint8_t Pin, VL53L0X_DeviceModes, VL53L0X_GpioFunctionality, VL53L0X_InterruptPolarity)` | `Pin` chỉ nhận 0. ⚠️ Đổi sang chế độ ngắt theo ngưỡng thì GPIO1 không còn nghĩa "có mẫu mới" mà `tof_multi` đang giả định |
| `VL53L0X_SetInterruptThresholds(Dev, Mode, FixPoint1616_t Low, FixPoint1616_t High)` | Ngưỡng theo mm, dạng 16.16 |
| `VL53L0X_GetPalErrorString(VL53L0X_Error, char *buf)` | `buf` ≥ 32 byte (`VL53L0X_MAX_STRING_LENGTH`) |
| `VL53L0X_GetRangeStatusString(uint8_t, char *buf)` | Như trên |
| `VL53L0X_GetVersion(VL53L0X_Version_t *)` | 1.0.4 build 4960 |

### E.6. Cấu trúc kết quả `VL53L0X_RangingMeasurementData_t`

| Trường | Kiểu | Đổi ra số thực |
|---|---|---|
| `RangeMilliMeter` | `uint16_t` | mm |
| `RangeStatus` | `uint8_t` | Bảng G.2 |
| `SignalRateRtnMegaCps` | `FixPoint1616_t` | `/65536` → MCPS |
| `AmbientRateRtnMegaCps` | `FixPoint1616_t` | `/65536` → MCPS |
| `EffectiveSpadRtnCount` | `uint16_t` | `/256` |
| `RangeDMaxMilliMeter` | `uint16_t` | Tầm tối đa ước lượng; chỉ có khi bật sigma |
| `TimeStamp`, `MeasurementTimeUsec` | `uint32_t` | Luôn 0 (ST không cài đặt) |

---

## F. Chương trình chạy thử (`app/`)

### F.1. `app_tof_test.c`: in kết quả qua UART

| Hàm | Gọi ở |
|---|---|
| `void App_TOF_Test_Init(void)` | USER CODE 2 |
| `void App_TOF_Test_Loop(void)` | USER CODE 3 (trong `while(1)`) |

Cần USART1 (115200), tick *Use MicroLIB*. In 5 dòng mỗi giây: khoảng cách, trạng thái, tần số đo.

### F.2. `app_tof_dbg.c`: xem bằng Watch của Keil

| Hàm / biến | Ý nghĩa |
|---|---|
| `void App_TOF_Dbg_Init(void)` | USER CODE 2 |
| `void App_TOF_Dbg_Loop(void)` | USER CODE 3 |
| `void App_TOF_Dbg_OnError(uint8_t idx)` | Đặt breakpoint ở đây để dừng khi một cảm biến chuyển sang ERROR |
| `volatile TOF_DbgView g_tof` | Gõ `g_tof` vào Watch, bật *Periodic Window Update* |

**Các trường của `g_tof`:**

| Nhóm | Trường | Giá trị đúng |
|---|---|---|
| Hệ thống | `sysclk_hz`, `uptime_ms`, `loop_count` | 72000000; tăng; tăng liên tục |
| Khởi tạo | `init_done`, `init_ok`, `init_ms` | 1, 2, vài trăm ms |
| | `scan_before_n`, `scan_before[4]` | 0 |
| | `scan_after_n`, `scan_after[4]` | 2, {0x54, 0x56} |
| | `mode` | 0 song song / 1 tuần tự |
| Thời gian | `poll_us_last`, `poll_us_max` | ≤ ~3000 µs (song song) |
| Mỗi cảm biến `s[i]` | `addr`, `state`, `last_error`, `err_count` | 0x54/0x56, 1, 0, 0 |
| | `range_mm`, `range_status`, `valid`, `age_ms` | Khoảng cách, 0, 1, < 100 |
| | `rate_hz`, `restarts` | ~30, 0 |
| | `spad_count`, `spad_aperture`, `offset_um` | Theo chip |
| Lệnh | `cmd`, `cmd_arg`, `cmd_dist_mm`, `cmd_result`, `cmd_count` | Bảng dưới |

**Lệnh từ debugger:** ghi `cmd_arg` trước, rồi ghi `cmd`.

| `cmd` | Tác dụng |
|---|---|
| 1 | `TOF_Restart(cmd_arg)`; `restarts++` |
| 2 | `TOF_CalibrateOffset(cmd_arg, cmd_dist_mm` (0 = 100)`)` |
| 3 | Đổi song song ↔ tuần tự, khởi tạo lại |
| 9 | Xoá `poll_us_max` |
| Khác | `cmd_result = −4` |

---

## G. Bảng tra

### G.1. Mã lỗi `VL53L0X_Error` (kiểu `int8_t`)

| Mã | Tên | Ý nghĩa |
|---|---|---|
| 0 | `VL53L0X_ERROR_NONE` | Thành công |
| −1 | `CALIBRATION_WARNING` | Dữ liệu hiệu chuẩn có thể sai (gọi DataInit nhiều lần mà không reset chip) |
| −2 | `MIN_CLIPPED` | Tham số bị cắt về giá trị nhỏ nhất |
| −3 | `UNDEFINED` | Không xác định (hay gặp khi các mã lỗi bị OR với nhau) |
| −4 | `INVALID_PARAMS` | Tham số sai |
| −5 | `NOT_SUPPORTED` | Không hỗ trợ |
| −6 | `RANGE_ERROR` | Không có số đo hợp lệ (ví dụ khi hiệu chuẩn) |
| −7 | `TIME_OUT` | Hết thời gian chờ |
| −8 | `MODE_NOT_SUPPORTED` | Chế độ đo không hỗ trợ |
| −10 / −11 | `GPIO_NOT_EXISTING` / `GPIO_FUNCTIONALITY_NOT_SUPPORTED` | Cấu hình GPIO sai |
| −12 | `INTERRUPT_NOT_CLEARED` | Không xoá được cờ ngắt |
| −20 | `CONTROL_INTERFACE` | **Lỗi I2C** |
| −30 | `INVALID_COMMAND` | Sai trạng thái |
| −40 | `DIVISION_BY_ZERO` | Chia cho 0 |
| −50 | `REF_SPAD_INIT` | Hiệu chuẩn SPAD lỗi |
| −99 | `NOT_IMPLEMENTED` | Chưa cài đặt |

### G.2. `RangeStatus`

| Giá trị | Ý nghĩa | Mã thô của chip |
|---|---|---|
| 0 | Hợp lệ | 11 |
| 1 | Sigma fail | 11 + sigma > giới hạn |
| 2 | Signal fail (tín hiệu yếu / không có vật) | 4 |
| 3 | Min range (quá gần) | 8, 10 |
| 4 | Phase fail (ngoài tầm) | 6, 9 |
| 5 | Lỗi phần cứng | 1, 2, 3 |
| 255 | Chưa có / không cập nhật | 0, 5, 7, 12–15 |

### G.3. Thanh ghi hay gặp (trang 0)

| Địa chỉ | Tên | Ghi chú |
|---|---|---|
| 0x00 | `SYSRANGE_START` | 0x01 đo một lần; 0x02 đo liên tục; 0x04 đo liên tục có chu kỳ; 0x00 dừng |
| 0x01 | `SYSTEM_SEQUENCE_CONFIG` | Các bước bật trong chuỗi đo |
| 0x0B | `SYSTEM_INTERRUPT_CLEAR` | Ghi 0x01 để xoá cờ ngắt |
| 0x13 | `RESULT_INTERRUPT_STATUS` | `& 0x07` ≠ 0 → có số đo |
| 0x14 | `RESULT_RANGE_STATUS` | Đầu khối 12 byte kết quả; khoảng cách ở offset +10 (0x1E–0x1F) |
| 0x28 | `ALGO_PART_TO_PART_RANGE_OFFSET_MM` | Offset dạng 10.2 |
| 0x44 | `FINAL_RANGE_CONFIG_MIN_COUNT_RATE_RTN_LIMIT` | Giới hạn signal rate, dạng 9.7 |
| 0x50 / 0x70 | `PRE_RANGE` / `FINAL_RANGE_CONFIG_VCSEL_PERIOD` | Mã hoá: `(pclk / 2) − 1` |
| 0x8A | `I2C_SLAVE_DEVICE_ADDRESS` | Địa chỉ **7-bit** |
| 0xC0 | `IDENTIFICATION_MODEL_ID` | **0xEE** |
| 0xFF | (chọn trang) | |

### G.4. Thông số thời gian

| Thao tác | Thời gian |
|---|---|
| 1 byte trên bus @100 kHz | 90 µs |
| Đọc 1 thanh ghi / ghi 1 thanh ghi | ~0,4 ms / ~0,3 ms |
| Đọc 1 số đo (`sigma_check = 0` / `1`) | 🔬 ~2,6 ms / ~10–12 ms |
| `TOF_Poll` tối đa (song song / tuần tự) | 🔬 ~3 ms / ~6 ms |
| `TOF_Init` mỗi cảm biến | 🔬 ~0,2 s |
| `TOF_CalibrateOffset` | 🔬 ~1,8–2 s |
| `PerformSingleRangingMeasurement` (đo chặn) | ~35–45 ms |
| `first_check` / `stale` (budget 33 ms) | 24 ms / 142 ms |

---

**Hết quyển 2.** Tiếp theo: [Quyển 3 — Ví dụ và giải thích](03_ViDu_GiaiThich.md).

# Quyển 3 — VÍ DỤ VÀ GIẢI THÍCH CHI TIẾT

> 12 ví dụ đi từ dễ đến khó. Mỗi ví dụ gồm: **Mục tiêu → Kiến thức dùng → Code → Giải thích → Kết quả mong đợi → Sai lầm thường gặp.**
>
> ✅ Mọi đoạn code trong quyển này đã được biên dịch (GCC `-std=c99 -Wall -Wextra -pedantic`) và chạy trên bộ giả lập. Kết quả 🔬 là đầu ra thật của bộ giả lập. **Chưa chạy trên Keil hay mạch thật.**
>
> Chú thích trong code viết **không dấu**, giống phong cách của thư viện, để Keil mở đúng với mọi bảng mã.

## Chuẩn bị chung

- Phần cứng và CubeMX như [Q1 chương 11]. Các label CubeMX: `XSHUT1` (PA0), `XSHUT2` (PA1), `TOF1_INT` (PB0), `TOF2_INT` (PB1).
- Trong Keil, thêm các file thư viện: `st_core/*.c`, `platform/vl53l0x_platform.c`, `tof/tof_multi.c`.
- Mỗi ví dụ viết thành một file `.c` riêng (ví dụ `vd2.c`). Trong `main.c`:
  - Gọi `VDx_Init()` ở USER CODE 2.
  - Gọi `VDx_Loop()` trong `while(1)` ở USER CODE 3.
- Xem biến bằng cửa sổ Watch của Keil (bật *Periodic Window Update*), hoặc in qua UART.

| VD | Nội dung | Độ khó |
|---|---|---|
| 1 | Một cảm biến, tối giản | ★☆☆ |
| 2 | Hai cảm biến song song, đọc theo bitmask | ★☆☆ |
| 3 | Không nối GPIO1 | ★☆☆ |
| 4 | Chế độ đo tuần tự | ★☆☆ |
| 5 | Đọc an toàn, phân loại kết quả | ★★☆ |
| 6 | Phát hiện vật cản có ngưỡng trễ và lọc trung bình | ★★☆ |
| 7 | Hiển thị lên LCD dùng chung bus | ★★☆ |
| 8 | Hiệu chuẩn offset và lưu vào flash | ★★★ |
| 9 | Tự giám sát, tự phục hồi | ★★☆ |
| 10 | Đổi cấu hình đo (tầm xa / chính xác / nhanh) | ★★★ |
| 11 | Dùng thẳng API của ST (không qua `tof_multi`) | ★★☆ |
| 12 | Đo thời gian thực thi bằng DWT | ★★☆ |

---

## VD1 — Một cảm biến, tối giản ★☆☆

**Mục tiêu:** đo khoảng cách bằng 1 cảm biến, với ít code nhất.
**Kiến thức:** `TOF_Sensor`, `TOF_Bus`, `TOF_Init`, `TOF_Poll`, bitmask, `range_status` [Q1 §3.6, §6, §8].

```c
#include "main.h"
#include "tof_multi.h"

extern I2C_HandleTypeDef hi2c1;

static TOF_Sensor tof1[1] = {
    { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_0, .addr = 0x54 }
};
static TOF_Bus bus1 = { .hi2c = &hi2c1, .sensor = tof1, .count = 1,
                        .mode = TOF_MODE_PARALLEL };

volatile uint16_t vd1_mm;      /* xem trong Watch */
volatile uint8_t  vd1_ok;

void VD1_Init(void)
{
    vd1_ok = TOF_Init(&bus1);              /* 1 = OK, 0 = loi */
}

void VD1_Loop(void)
{
    if (TOF_Poll(&bus1) & 1UL) {           /* bit 0 = cam bien 0 vua co so do moi */
        if (tof1[0].range_status == 0)     /* 0 = hop le */
            vd1_mm = tof1[0].range_mm;
    }
}
```

**Giải thích từng khối:**
- **Khai báo `tof1[1]`**:
  - Dùng *designated initializer* (cú pháp `.ten = giá trị`, chuẩn C99) để chỉ điền 3 trường người dùng cần: chân XSHUT và địa chỉ.
  - Không có `.gpio1_port`, nên trường này mặc định là `NULL`: thư viện sẽ hỏi qua I2C.
  - Các trường còn lại tự bằng 0, vì biến `static` luôn được khởi tạo 0.
- **Khai báo `bus1`**:
  - `.budget_us`, `.poll_ms`, `.sigma_check` không ghi, nên đều bằng 0. Thư viện hiểu 0 là "dùng mặc định": 33 ms, 10 ms, tắt sigma.
- **`vd1_mm`, `vd1_ok` là `volatile`**: trình biên dịch không được tối ưu bỏ biến, nên Watch luôn đọc được giá trị thật.
- **`TOF_Init`**:
  1. Kéo XSHUT xuống thấp.
  2. Bật cảm biến, đổi địa chỉ 0x52 → 0x54.
  3. Khởi tạo đầy đủ.
  4. Bắt đầu đo liên tục.
  5. Trả về 1 nếu thành công.
- **`TOF_Poll(&bus1) & 1UL`**: hàm trả bitmask; bit 0 tương ứng cảm biến 0. Dùng `1UL` (số `unsigned long`) để phép AND đúng kiểu `uint32_t`.
- **Kiểm tra `range_status == 0`** trước khi dùng `range_mm` [Q1 §8].

**Kết quả mong đợi** 🔬: vật ở 250 mm → `vd1_ok = 1`, `vd1_mm = 250`.

**Sai lầm thường gặp:**
- Gọi `TOF_Init` trước `MX_I2C1_Init()`: bus chưa sẵn sàng, `vd1_ok = 0`.
- Dùng `range_mm` mà không kiểm tra `range_status`: khi không có vật, bạn nhận giá trị ~8190.

---

## VD2 — Hai cảm biến song song ★☆☆

**Mục tiêu:** đo 2 cảm biến cùng lúc, mỗi con ~30 Hz.
**Kiến thức:** gán địa chỉ [Q1 §4], đọc bitmask.

```c
#include "main.h"
#include "tof_multi.h"

extern I2C_HandleTypeDef hi2c1;

TOF_Sensor tof[2] = {
    { .xshut_port = XSHUT1_GPIO_Port, .xshut_pin = XSHUT1_Pin,
      .gpio1_port = TOF1_INT_GPIO_Port, .gpio1_pin = TOF1_INT_Pin, .addr = 0x54 },
    { .xshut_port = XSHUT2_GPIO_Port, .xshut_pin = XSHUT2_Pin,
      .gpio1_port = TOF2_INT_GPIO_Port, .gpio1_pin = TOF2_INT_Pin, .addr = 0x56 },
};

TOF_Bus bus = {
    .hi2c        = &hi2c1,
    .sensor      = tof,
    .count       = 2,
    .mode        = TOF_MODE_PARALLEL,
    .budget_us   = 33000,
    .poll_ms     = 10,
    .sigma_check = 0,
};

volatile uint16_t dist_mm[2];
volatile uint8_t  n_ok;

void VD2_Init(void)
{
    n_ok = TOF_Init(&bus);
    /* n_ok < 2 -> xem tof[i].state va tof[i].last_error */
}

void VD2_Loop(void)
{
    uint32_t upd = TOF_Poll(&bus);
    uint8_t  i;

    for (i = 0; i < 2; i++)
        if ((upd & (1UL << i)) && tof[i].range_status == 0)
            dist_mm[i] = tof[i].range_mm;
}
```

**Giải thích:**
- **Dùng label của CubeMX** (`XSHUT1_GPIO_Port`, `XSHUT1_Pin`…) thay vì viết cứng `GPIOA`, `GPIO_PIN_0`. Đổi chân trong CubeMX thì code tự đúng theo.
- **`tof` và `bus` không có `static`**: các ví dụ VD5–VD12 dùng lại hai biến này (khai báo `extern` ở file khác).
- **Ghi rõ mọi tuỳ chọn** của `bus` (budget, poll, sigma) để người đọc thấy giá trị đang dùng, dù chúng trùng với mặc định.
- **Vòng `for`**: với từng cảm biến, `upd & (1UL << i)` kiểm tra bit i của bitmask. Chỉ cập nhật khi có số đo mới **và** hợp lệ.

**Kết quả** 🔬: hai vật ở 180 mm và 420 mm → `n_ok = 2`, `dist_mm = {180, 420}`. Mỗi cảm biến ~33 Hz trên giả lập; mạch thật ~30 Hz.

**Sai lầm thường gặp:**
- Đặt hai cảm biến cùng `.addr`: cảm biến thứ hai bị từ chối, `last_error = -4`.
- Quên đặt mức Low cho XSHUT trong CubeMX: trước khi `TOF_Init` chạy, cả hai đã bật ở 0x52 một lúc. Thư viện vẫn kéo thấp lại nên vẫn chạy, nhưng tốt nhất đặt Low ngay từ đầu.

---

## VD3 — Không nối GPIO1 ★☆☆

**Mục tiêu:** tiết kiệm dây và chân MCU, chỉ dùng I2C và XSHUT.

So với VD2, chỉ bỏ hai trường GPIO1:
```c
TOF_Sensor tof[2] = {
    { .xshut_port = XSHUT1_GPIO_Port, .xshut_pin = XSHUT1_Pin, .addr = 0x54 },
    { .xshut_port = XSHUT2_GPIO_Port, .xshut_pin = XSHUT2_Pin, .addr = 0x56 },
};
/* .gpio1_port = NULL (mac dinh) -> thu vien hoi qua I2C */
```

**Giải thích:**
- Không có GPIO1, thư viện hỏi cảm biến qua I2C bằng `VL53L0X_GetMeasurementDataReady` (1 lần đọc, ~0,4 ms).
- Lịch hỏi: sau khi đọc xong, chờ `first_check` = 3/4 × budget = 24 ms rồi mới hỏi; sau đó hỏi mỗi `poll_ms` = 10 ms [Q1 §6.3].
- Trung bình mỗi số đo tốn thêm ~1 lần hỏi, tức ~0,4 ms I2C.

**Kết quả** 🔬 (4 cảm biến): tốc độ vẫn ~33 Hz mỗi con. Một lần `TOF_Poll` giữ CPU tối đa 3,1 ms (so với 2,6 ms khi có GPIO1).

💡 **Khi nào nên nối GPIO1:** bus dùng chung với nhiều thiết bị, hoặc muốn tiết kiệm từng phần trăm thời gian CPU. Có thể trộn: con có GPIO1, con không.

---

## VD4 — Chế độ đo tuần tự ★☆☆

**Mục tiêu:** hai cảm biến đặt sát nhau, nhìn cùng hướng, nhưng không nhiễu lẫn nhau.

So với VD2, chỉ đổi một dòng:
```c
    .mode = TOF_MODE_SEQUENTIAL,
```

**Giải thích:**
- Mỗi lúc chỉ một cảm biến phát tia laser: con 0 đo xong → đọc → bật con 1 đo → đọc → bật con 0…
- Thư viện tự đặt `SINGLE_RANGING` cho mọi cảm biến; `TOF_Poll` tự chuyển lượt.

**Khi nào cần?** Hai cảm biến cách nhau 10 cm, nhìn song song cùng hướng. Vùng nhìn chồng nhau khi `2 × d × tan(12,5°) > 100 mm`, tức `d > ~226 mm` [Q4 bài B6]. Vật xa hơn 23 cm sẽ được cả hai con "thấy", nên có nguy cơ nhiễu.

**Kết quả** 🔬: mỗi cảm biến ~14–15 Hz (2 con); giả lập xác nhận **không bao giờ có 2 con đo cùng lúc**. Mạch thật (đo 33 ms/lần) khoảng 12–13 Hz mỗi con.

---

## VD5 — Đọc an toàn, phân loại kết quả ★★☆

**Mục tiêu:** viết một hàm đọc trả về **lý do** khi không có số đo, để ứng dụng xử lý đúng từng trường hợp.

```c
typedef enum {
    DIST_OK = 0,        /* co so do hop le, moi          */
    DIST_NO_TARGET,     /* tin hieu yeu / ngoai tam      */
    DIST_TOO_CLOSE,     /* vat qua gan                   */
    DIST_UNRELIABLE,    /* sigma fail: so do khong chac  */
    DIST_STALE,         /* chua co / qua cu              */
    DIST_SENSOR_FAIL    /* cam bien loi / loi phan cung  */
} DistResult;

DistResult tof_read_safe(uint8_t i, uint16_t *mm)
{
    const TOF_Sensor *s = &tof[i];

    if (s->state != TOF_STATE_RUNNING)
        return DIST_SENSOR_FAIL;
    if (TOF_GetRange(&bus, i, 100, mm))
        return DIST_OK;

    switch (s->range_status) {
    case 1:          return DIST_UNRELIABLE;
    case 2: case 4:  return DIST_NO_TARGET;
    case 3:          return DIST_TOO_CLOSE;
    case 0:                                  /* hop le nhung qua 100 ms */
    case 255:        return DIST_STALE;      /* chua co so do nao */
    default:         return DIST_SENSOR_FAIL;
    }
}
```

**Giải thích:**
- **Bước 1**: cảm biến không RUNNING thì báo hỏng ngay.
- **Bước 2**: `TOF_GetRange` trả 1 thì có số đo tốt, kết thúc.
- **Bước 3**: `TOF_GetRange` trả 0 thì tìm lý do qua `range_status`:
  - `case 0` rơi xuống dòng `case 255` (*fall-through*): `range_status = 0` mà `TOF_GetRange` vẫn trả 0, thì lý do duy nhất còn lại là số đo **quá cũ**.
  - `case 2` và `case 4` gộp chung: với người dùng, cả hai đều nghĩa là "không thấy vật trong tầm".

**Cách dùng:**
```c
uint16_t mm;
switch (tof_read_safe(0, &mm)) {
case DIST_OK:         /* dung mm */                break;
case DIST_NO_TARGET:  /* duong thong thoang */     break;
case DIST_TOO_CLOSE:  /* dung khan cap */          break;
default:              /* giu gia tri cu / bao loi */ break;
}
```

**Kết quả** 🔬: vật ở 180 mm → trả `DIST_OK`, `mm = 180`.

---

## VD6 — Phát hiện vật cản có ngưỡng trễ và lọc trung bình ★★☆

**Mục tiêu:** bật cờ `obstacle` khi vật vào gần hơn 300 mm, tắt khi ra xa hơn 350 mm, không bị "nháy" khi vật đứng đúng ở ngưỡng.
**Kiến thức:** trung bình trượt (*moving average*), ngưỡng trễ (*hysteresis*).

```c
#define OBS_ON_MM   300U     /* gan hon 300 mm -> bat canh bao  */
#define OBS_OFF_MM  350U     /* xa hon 350 mm  -> tat canh bao  */
#define AVG_N       4U

static uint16_t avg_buf[AVG_N];
static uint8_t  avg_cnt, avg_pos;
volatile uint8_t  obstacle;
volatile uint16_t avg_mm;

static uint16_t avg_push(uint16_t v)
{
    uint32_t sum = 0;
    uint8_t  k;

    avg_buf[avg_pos] = v;
    avg_pos = (uint8_t)((avg_pos + 1U) % AVG_N);
    if (avg_cnt < AVG_N)
        avg_cnt++;
    for (k = 0; k < avg_cnt; k++)
        sum += avg_buf[k];
    return (uint16_t)(sum / avg_cnt);
}

void VD6_Loop(void)
{
    uint32_t upd = TOF_Poll(&bus);
    uint16_t mm;

    if (!(upd & 1UL))
        return;                                   /* cam bien 0 chua co so do moi */

    if (TOF_GetRange(&bus, 0, 100, &mm)) {
        avg_mm = avg_push(mm);
        if (!obstacle && avg_mm < OBS_ON_MM)
            obstacle = 1;
        else if (obstacle && avg_mm > OBS_OFF_MM)
            obstacle = 0;
    } else if (tof[0].range_status == 2 || tof[0].range_status == 4) {
        avg_cnt = 0;                              /* khong co vat: xoa bo loc */
        avg_pos = 0;
        obstacle = 0;
    }
}
```

**Giải thích:**
- **Vì sao cần ngưỡng trễ?** Chỉ có một ngưỡng 300 mm thì khi vật đứng ở ~300 mm, nhiễu ±3 mm làm cờ bật tắt liên tục. Hai ngưỡng (300 để bật, 350 để tắt) tạo **vùng trễ 50 mm**: trong vùng này cờ **giữ nguyên trạng thái cũ**.
- **`avg_push`**: bộ đệm vòng 4 phần tử.
  - `avg_cnt` đếm số phần tử đã có (tối đa 4).
  - Nhờ vậy những lần đo đầu tiên **không bị kéo về 0** bởi các ô chưa có dữ liệu. Đây là lỗi hay gặp nếu luôn chia cho 4.
  - Tổng tính bằng `uint32_t` để không tràn số: 4 × 65535 > 65535.
- **Không có vật** (status 2/4): xoá bộ lọc để lần sau bắt đầu lại từ đầu, và tắt cảnh báo.

**Kết quả** 🔬 (vật di chuyển 500 → 320 → 280 → 330 → 400 mm):

| Vật ở | `avg_mm` | `obstacle` | Giải thích |
|---|---|---|---|
| 500 | 500 | 0 | Xa |
| 320 | 320 | 0 | Trong vùng trễ, đang tắt nên giữ tắt |
| 280 | 280 | **1** | < 300, bật |
| 330 | 330 | **1** | Trong vùng trễ, đang bật nên giữ bật |
| 400 | 400 | 0 | > 350, tắt |

**Mở rộng:** độ trễ của bộ lọc = AVG_N × chu kỳ đo ≈ 4 × 33 = 132 ms. Bộ lọc càng dài thì càng mượt nhưng phản ứng càng chậm.

---

## VD7 — Hiển thị lên LCD dùng chung bus ★★☆

**Mục tiêu:** hiện khoảng cách 2 cảm biến lên LCD 16x2 (PCF8574, địa chỉ 0x4E), cùng bus I2C1 với cảm biến.

```c
#include "myLCD_I2C.h"
#include <stdio.h>

void VD7_Loop(void)
{
    static uint32_t t_lcd = 0;
    char     line[20];                            /* 16 ky tu + du phong */
    uint16_t mm;
    uint8_t  i;

    (void)TOF_Poll(&bus);                         /* luon goi, khong phu thuoc LCD */

    if (HAL_GetTick() - t_lcd < 250U)
        return;                                   /* LCD chi cap nhat 4 lan/giay */
    t_lcd = HAL_GetTick();

    for (i = 0; i < 2; i++) {
        if (TOF_GetRange(&bus, i, 200, &mm))
            snprintf(line, sizeof line, "TOF%u: %4u mm   ", (unsigned)(i + 1U), (unsigned)mm);
        else
            snprintf(line, sizeof line, "TOF%u:  ----     ", (unsigned)(i + 1U));
        LCD_set_pos(0, i);
        LCD_write_str(line);                      /* ghi de du 16 ky tu, khong LCD_clear() */
    }
}
```

**Giải thích:**
- **Tách tần số**: `TOF_Poll` gọi mỗi vòng lặp, còn LCD chỉ cập nhật 4 lần/giây. Mắt người không cần nhanh hơn, và ghi LCD tốn thời gian bus.
- **Mỗi dòng đúng 16 ký tự**, nhờ khoảng trắng ở cuối chuỗi định dạng. Cách này ghi đè lên chữ cũ, **không cần `LCD_clear()`**: `LCD_clear` gây nháy màn hình và tốn ~2 ms chờ.
- **`max_age_ms = 200`**: LCD chậm hơn nên chấp nhận số đo cũ hơn một chút.
- **Ép kiểu `(unsigned)`** cho `%u`: `uint8_t`/`uint16_t` được nâng thành `int` khi truyền vào `snprintf`, nên ép để khớp chuẩn.
- **Bộ đệm 20 byte** dù chỉ dùng 16: trình biên dịch không biết `mm` ≤ 9999, nên cảnh báo có thể tràn. Bộ đệm dư giúp hết cảnh báo một cách an toàn.

**Kết quả** 🔬:
```
[TOF1:  400 mm   ]
[TOF2:  420 mm   ]
```

⚠️ Dùng **thư viện LCD đã sửa**, không còn chờ 5 ms mỗi ký tự. Bản LCD cũ mất ~200 ms để vẽ 2 dòng, làm `TOF_Poll` bị trễ theo. Địa chỉ LCD là 0x4E, không trùng 0x54/0x56.

---

## VD8 — Hiệu chuẩn offset và lưu vào flash ★★★

**Mục tiêu:** hiệu chuẩn một lần, lưu vào flash, các lần khởi động sau tự nạp lại.
**Kiến thức:** `TOF_Calib`, `TOF_CalibrateOffset`, HAL_FLASH [Q1 §9].

```c
#include "main.h"
#include "tof_multi.h"
#include <string.h>

#define CALIB_FLASH_ADDR  0x0800FC00UL   /* trang cuoi cung cua 64 KB flash */
#define CALIB_MAGIC       0x544F4631UL   /* "TOF1" */
#define CALIB_N           2U

typedef struct {
    uint32_t  magic;
    TOF_Calib c[CALIB_N];
    uint32_t  checksum;                  /* tong cac word phia truoc */
} CalibBlock;                            /* 40 byte = 10 word */

static uint32_t calib_sum(const uint32_t *w, uint32_t nwords)
{
    uint32_t s = 0;
    while (nwords--)
        s += *w++;
    return s;
}

HAL_StatusTypeDef Calib_Save(const TOF_Sensor *t)
{
    CalibBlock             blk;
    FLASH_EraseInitTypeDef er;
    const uint32_t        *w = (const uint32_t *)&blk;
    uint32_t               page_err, i;

    memset(&blk, 0, sizeof blk);                     /* xoa ca byte dem -> checksum on dinh */
    blk.magic = CALIB_MAGIC;
    for (i = 0; i < CALIB_N; i++)
        blk.c[i] = t[i].calib;
    blk.checksum = calib_sum(w, sizeof(blk) / 4U - 1U);

    HAL_FLASH_Unlock();
    er.TypeErase   = FLASH_TYPEERASE_PAGES;
    er.Banks       = FLASH_BANK_1;
    er.PageAddress = CALIB_FLASH_ADDR;
    er.NbPages     = 1;
    if (HAL_FLASHEx_Erase(&er, &page_err) != HAL_OK) {
        HAL_FLASH_Lock();
        return HAL_ERROR;
    }
    for (i = 0; i < sizeof(blk) / 4U; i++) {
        if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, CALIB_FLASH_ADDR + 4U * i, w[i]) != HAL_OK) {
            HAL_FLASH_Lock();
            return HAL_ERROR;
        }
    }
    HAL_FLASH_Lock();
    return HAL_OK;
}

uint8_t Calib_Load(TOF_Sensor *t)
{
    const CalibBlock *blk = (const CalibBlock *)CALIB_FLASH_ADDR;
    uint32_t i;

    if (blk->magic != CALIB_MAGIC)
        return 0;                                    /* chua tung luu */
    if (calib_sum((const uint32_t *)blk, sizeof(CalibBlock) / 4U - 1U) != blk->checksum)
        return 0;                                    /* du lieu hong */
    for (i = 0; i < CALIB_N; i++)
        t[i].calib = blk->c[i];                      /* valid = 1 da duoc luu cung */
    return 1;
}
```

**Cách dùng:**
```c
/* Khoi dong */
Calib_Load(tof);                 /* neu co du lieu hop le -> calib.valid = 1 */
TOF_Init(&bus);                  /* valid = 1 -> bo qua RefSpadManagement, nap offset */

/* Khi nguoi dung bam nut hieu chuan (tam trang o dung 100 mm truoc TOF1) */
if (TOF_CalibrateOffset(&bus, 0, 100) == VL53L0X_ERROR_NONE)
    Calib_Save(tof);
```

**Giải thích:**
- **Vì sao trang cuối (0x0800FC00)?**
  - STM32F103C8 có 64 KB flash, chia thành các trang 1 KB. Trang cuối bắt đầu tại 0x08000000 + 63 × 1024 = 0x0800FC00.
  - Chương trình nằm ở đầu flash. Chỉ cần code < 63 KB là không đè lên trang này; kiểm tra trong file `.map` hoặc dòng *Program Size* của Keil.
- **Flash phải xoá rồi mới ghi.** Xoá theo **cả trang**, khi đó mọi byte thành 0xFF. Ghi theo từng **word** 32 bit.
- **`magic`** để phân biệt "chưa từng lưu", vì flash đã xoá thì đọc ra 0xFFFFFFFF. **`checksum`** phát hiện dữ liệu hỏng, ví dụ mất điện giữa lúc đang ghi.
- **`memset(&blk, 0, ...)`**: `TOF_Calib` có byte đệm. Không xoá thì byte đệm chứa rác, làm checksum thay đổi ngẫu nhiên.
- **`sizeof(blk) / 4U - 1U`**: số word **trừ word checksum cuối cùng**. Struct dài 40 byte: magic 4 + 2 × 16 + checksum 4.
- **Đọc flash** chỉ cần ép con trỏ, vì flash được ánh xạ vào bộ nhớ như ROM.

⚠️ **Lưu ý:**
- Xoá hoặc ghi flash làm CPU dừng vài chục ms. Không làm việc này khi đang điều khiển động cơ.
- Mỗi trang chịu được ~10 000 lần xoá: chỉ lưu khi người dùng thật sự hiệu chuẩn.
- Code > 63 KB sẽ đè lên vùng hiệu chuẩn. Khi đó dời `CALIB_FLASH_ADDR`, hoặc dùng chip 128 KB.

---

## VD9 — Tự giám sát, tự phục hồi ★★☆

**Mục tiêu:** hệ thống chạy lâu dài không cần người can thiệp: cảm biến lỗi thì tự khởi động lại; bus treo thì tự gỡ.

```c
void VD9_Supervisor(void)
{
    static uint32_t t = 0;
    uint8_t i, n_err = 0;

    if (HAL_GetTick() - t < 1000U)
        return;
    t = HAL_GetTick();

    for (i = 0; i < 2; i++)
        if (tof[i].state == TOF_STATE_ERROR)
            n_err++;
    if (n_err == 0)
        return;

    if (n_err == 2)                               /* ca 2 cung loi -> nghi bus ket */
        TOF_I2C_Recover(&hi2c1, GPIOB, GPIO_PIN_6, GPIO_PIN_7);

    for (i = 0; i < 2; i++)
        if (tof[i].state == TOF_STATE_ERROR)
            (void)TOF_Restart(&bus, i);
}
/* main loop: VD2_Loop(); VD9_Supervisor(); */
```

**Giải thích:**
- **Chạy mỗi giây một lần**, không phải mỗi vòng lặp: `TOF_Restart` chặn CPU ~0,2 s nếu cảm biến có mặt.
- **Phân biệt hai loại sự cố:**
  - Một con lỗi: chỉ con đó có vấn đề (dây, nguồn của nó), chỉ cần restart con đó.
  - **Tất cả cùng lỗi**: khả năng cao là **bus** có vấn đề, nên gỡ bus trước rồi mới restart.
- Cảm biến bị rút ra hẳn: mỗi giây `TOF_Restart` thử lại một lần, tốn vài ms (đọc ID nhận NACK ngay). Cắm lại thì tự chạy.

**Kết quả** 🔬: rút TOF2 → `tof[1].state = 2` (ERROR), TOF1 vẫn chạy. Cắm lại → `state = 1`, `dist_mm[1] = 420`, **0 xung đột** trên bus.

---

## VD10 — Đổi cấu hình đo: tầm xa / chính xác cao / tốc độ cao ★★★

**Mục tiêu:** dùng API của ST trên `&tof[i].dev` để chỉnh cấu hình đo mà `tof_multi` không có sẵn.

| Profile | Signal limit | VCSEL (pre/final) | Budget | Đặc điểm |
|---|---|---|---|---|
| Mặc định | 0,25 MCPS | 14 / 10 | 33 ms | Cân bằng, ~1,2 m |
| Tầm xa | **0,10 MCPS** | **18 / 14** | 33 ms | ~2 m trong nhà; dễ nhiễu hơn |
| Chính xác cao | 0,25 | 14 / 10 | **200 ms** | Mượt, chậm (5 Hz) |
| Tốc độ cao | 0,25 | 14 / 10 | **20 ms** | ~50 Hz, kém mượt hơn |

```c
typedef enum { PROF_DEFAULT, PROF_LONG_RANGE, PROF_HIGH_ACCURACY, PROF_HIGH_SPEED } TOF_Profile;

/* Chi dung o che do TOF_MODE_PARALLEL. Truoc khi goi: bus.budget_us >= budget cua profile. */
VL53L0X_Error tof_apply_profile(TOF_Sensor *s, TOF_Profile p)
{
    VL53L0X_DEV    d      = &s->dev;
    FixPoint1616_t sig    = (FixPoint1616_t)(0.25 * 65536);   /* 0,25 MCPS */
    uint32_t       budget = 33000;
    uint8_t        pre    = 14, fin = 10;                     /* chu ky VCSEL mac dinh */
    VL53L0X_Error  st;

    switch (p) {
    case PROF_LONG_RANGE:    sig = (FixPoint1616_t)(0.10 * 65536); pre = 18; fin = 14; break;
    case PROF_HIGH_ACCURACY: budget = 200000; break;
    case PROF_HIGH_SPEED:    budget = 20000;  break;
    default: break;
    }

    st = VL53L0X_StopMeasurement(d);
    HAL_Delay(40);                                 /* cho lan do dang chay (<= 33 ms) ket thuc */
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_SetLimitCheckValue(d, VL53L0X_CHECKENABLE_SIGNAL_RATE_FINAL_RANGE, sig);
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_SetVcselPulsePeriod(d, VL53L0X_VCSEL_PERIOD_PRE_RANGE, pre);
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_SetVcselPulsePeriod(d, VL53L0X_VCSEL_PERIOD_FINAL_RANGE, fin);
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_SetMeasurementTimingBudgetMicroSeconds(d, budget);
    (void)VL53L0X_ClearInterruptMask(d, 0);
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_StartMeasurement(d);
    return st;
}
```

**Cách dùng:**
```c
bus.budget_us = 200000;                         /* (1) bao cho thu vien biet budget lon nhat */
tof_apply_profile(&tof[0], PROF_HIGH_ACCURACY); /* (2) */
```

**Giải thích:**
- **Dừng → chỉnh → chạy lại**: ST yêu cầu thay đổi cấu hình khi cảm biến không đo. `HAL_Delay(40)` chờ lần đo đang dở kết thúc.
- **`(FixPoint1616_t)(0.10 * 65536)`**: trình biên dịch tính sẵn thành hằng số 6553, nên **không** sinh phép tính `float` lúc chạy.
- **`SetVcselPulsePeriod`**: ST tự chạy lại hiệu chuẩn phase bên trong hàm, vì đổi VCSEL làm phase thay đổi.
- **Đặt budget sau VCSEL**: đổi VCSEL làm thay đổi cách chia thời gian giữa các bước, nên đặt budget cuối cùng.

⚠️ **Ba bẫy quan trọng:**
1. **Phải tăng `bus.budget_us`** khi dùng profile 200 ms. Thư viện dùng `bus.budget_us` để tính `stale_ms` = 33×4+10 = 142 ms < 200 ms. Không tăng thì cảm biến bị báo TIME_OUT liên tục, rồi chuyển sang ERROR.
2. **Mất cấu hình sau `TOF_Restart`**: restart khởi tạo lại từ đầu, chỉ áp dụng `bus.budget_us`, còn VCSEL và signal limit về mặc định. Phải gọi lại `tof_apply_profile` sau mỗi lần restart [Q4 bài E7].
3. **Chế độ tuần tự**: không gọi `StartMeasurement` (thư viện tự bật từng con). Hàm trên chỉ dành cho chế độ song song.

**Kết quả** 🔬: profile tầm xa ghi thanh ghi VCSEL đúng 18 / 14; cảm biến vẫn RUNNING, không lỗi.

---

## VD11 — Dùng thẳng API của ST (không qua tof_multi) ★★☆

**Mục tiêu:** hiểu thư viện làm gì bên trong, bằng cách tự gọi API của ST cho **một** cảm biến, đo kiểu chặn CPU.

```c
#include "main.h"
#include "vl53l0x_api.h"

extern I2C_HandleTypeDef hi2c1;
static VL53L0X_Dev_t dev0;                 /* bien toan cuc -> da duoc xoa 0 */

VL53L0X_Error VD11_Init(void)
{
    uint32_t spad;
    uint8_t  aperture, vhv, phase;
    VL53L0X_Error st;

    HAL_GPIO_WritePin(XSHUT1_GPIO_Port, XSHUT1_Pin, GPIO_PIN_SET);   /* bat cam bien */
    HAL_Delay(2);                                                     /* tBOOT <= 1,2 ms */

    dev0.I2cHandle  = &hi2c1;
    dev0.I2cDevAddr = 0x52;                                           /* dia chi mac dinh */

    st = VL53L0X_DataInit(&dev0);
    if (st == VL53L0X_ERROR_NONE) st = VL53L0X_StaticInit(&dev0);
    if (st == VL53L0X_ERROR_NONE) st = VL53L0X_PerformRefSpadManagement(&dev0, &spad, &aperture);
    if (st == VL53L0X_ERROR_NONE) st = VL53L0X_PerformRefCalibration(&dev0, &vhv, &phase);
    if (st == VL53L0X_ERROR_NONE) st = VL53L0X_SetDeviceMode(&dev0, VL53L0X_DEVICEMODE_SINGLE_RANGING);
    if (st == VL53L0X_ERROR_NONE) st = VL53L0X_SetMeasurementTimingBudgetMicroSeconds(&dev0, 33000);
    return st;
}

/* CHAN CPU ~35 ms. Tra 0xFFFF neu loi hoac so do khong hop le. */
uint16_t VD11_MeasureOnce(void)
{
    VL53L0X_RangingMeasurementData_t m;

    if (VL53L0X_PerformSingleRangingMeasurement(&dev0, &m) != VL53L0X_ERROR_NONE)
        return 0xFFFF;
    return (m.RangeStatus == 0) ? m.RangeMilliMeter : 0xFFFFU;
}
```

**Giải thích:**
- Đây chính là trình tự trong `tof_setup()` [Q1 §5.2], viết tay cho một cảm biến ở địa chỉ mặc định.
- Mẫu `if (st == NONE) st = ...` **dừng ngay ở lỗi đầu tiên** và giữ đúng mã lỗi đó. Cách này tốt hơn `st |= ...`, vì OR các mã âm với nhau ra mã vô nghĩa [Q7 chuyên đề 4].
- `PerformSingleRangingMeasurement` = bật đo → **đứng chờ** → đọc → xoá cờ. Đơn giản nhưng chặn CPU.

**So sánh với `tof_multi`:**

| | VD11 (API ST trực tiếp) | `tof_multi` |
|---|---|---|
| Số cảm biến | 1 (ở 0x52) | 1–32 |
| Chặn CPU mỗi lần đo | ~35–45 ms | ≤ ~3 ms |
| Phát hiện lỗi, tự phục hồi | Tự viết | Có sẵn |
| Dễ hiểu | ✔ | Cần đọc Q1 |

**Kết quả** 🔬: vật ở 777 mm → `VD11_MeasureOnce()` trả 777, mất ~45 ms (giả lập).

---

## VD12 — Đo thời gian thực thi bằng DWT ★★☆

**Mục tiêu:** chứng minh bằng số liệu rằng `TOF_Poll` không chặn CPU lâu, dùng bộ đếm chu kỳ của lõi Cortex-M3.

```c
static uint32_t cyc_per_us = 72;
volatile uint32_t t_poll_us, t_poll_max_us;

void dwt_init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;   /* bat khoi trace/DWT */
    DWT->CYCCNT = 0;
    DWT->CTRL  |= DWT_CTRL_CYCCNTENA_Msk;             /* bat bo dem chu ky */
    cyc_per_us  = SystemCoreClock / 1000000U;         /* 72 */
}

void VD12_Loop(void)
{
    uint32_t c0 = DWT->CYCCNT;
    (void)TOF_Poll(&bus);
    t_poll_us = (DWT->CYCCNT - c0) / cyc_per_us;
    if (t_poll_us > t_poll_max_us)
        t_poll_max_us = t_poll_us;
}
```

**Giải thích:**
- **`DWT->CYCCNT`** tăng 1 mỗi chu kỳ CPU (72 triệu lần mỗi giây ở 72 MHz). Độ phân giải 1/72 µs, tốt hơn nhiều so với `HAL_GetTick` (1 ms).
- Phải bật `TRCENA` trong `CoreDebug->DEMCR` thì khối DWT mới chạy.
- **Phép trừ không dấu `CYCCNT - c0`** vẫn đúng khi bộ đếm quay vòng (sau ~59,6 s ở 72 MHz).
- `SystemCoreClock` là biến của HAL, chứa tần số HCLK hiện tại.

**Kết quả** 🔬: `t_poll_max_us = 2610` (2 cảm biến, có GPIO1), nằm dưới mức 3 ms mà thiết kế đặt ra. Bạn có thể dùng mẫu này để đo bất kỳ đoạn code nào khác: vẽ LCD, đọc HX711…

---

**Hết quyển 3.** Tiếp theo: [Quyển 4 — Bài tập](04_BaiTap.md).

# Quyển 4 — BÀI TẬP

> Làm theo thứ tự từ phần A đến phần F. Mức độ: **NB** nhận biết, **TH** thông hiểu, **VD** vận dụng, **VDC** vận dụng cao.
> Lời giải chi tiết ở [Quyển 5](05_LoiGiai.md). **Tự làm trước khi xem lời giải.**
>
> Giả thiết chung (trừ khi đề ghi khác): STM32F103 72 MHz, I2C 100 kHz, `budget_us = 33000`, `poll_ms = 10`, `sigma_check = 0`, `TOF_STALE_FACTOR = 4`, `TOF_MAX_ERRORS = 5`.

---

## Phần A — Trắc nghiệm (NB, TH)

Chọn **một** đáp án đúng.

**A1.** Địa chỉ I2C mặc định của VL53L0X, viết theo kiểu mà HAL STM32 dùng, là:
a) 0x29  b) 0x52  c) 0x54  d) 0xEE

**A2.** Kéo chân XSHUT xuống mức thấp thì:
a) chip báo có số đo mới  b) chip tắt hẳn và mất toàn bộ cấu hình, kể cả địa chỉ  c) bus I2C bị reset  d) chip chuyển sang trang thanh ghi 1

**A3.** Hàm nào phải được gọi liên tục trong `while(1)`?
a) `TOF_Init`  b) `TOF_Poll`  c) `TOF_Restart`  d) `TOF_CalibrateOffset`

**A4.** `TOF_Poll()` trả về `0x02`. Ý nghĩa là:
a) có 2 cảm biến có số đo mới  b) cảm biến chỉ số 1 vừa có số đo mới  c) lỗi −2  d) cảm biến chỉ số 2 vừa có số đo mới

**A5.** `range_status = 2` thường có nghĩa là:
a) số đo hợp lệ  b) vật quá gần  c) tín hiệu yếu / không có vật trong tầm  d) lỗi phần cứng

**A6.** Địa chỉ nào được hàm `tof_addr_valid()` chấp nhận?
a) 0x52  b) 0x55  c) 0x56  d) 0x08

**A7.** Vì sao chân XSHUT nên cấu hình **Output Open-Drain**?
a) để chuyển mức nhanh hơn
b) module đã có điện trở kéo lên; open-drain chỉ kéo xuống, không ép 3,3 V vào miền điện áp 2,8 V của chip
c) để đọc được trạng thái chân
d) vì HAL bắt buộc

**A8.** Với `sigma_check = 0`, một lần đọc số đo (gồm cả xoá cờ ngắt) tốn bao nhiêu giao dịch I2C?
a) 1  b) 4  c) 12  d) 23

**A9.** Hàm nào của ST tự đổi chế độ đo về `SINGLE_RANGING`, nên `SetDeviceMode` phải gọi sau nó?
a) `DataInit`  b) `StaticInit`  c) `PerformRefSpadManagement`  d) `SetMeasurementTimingBudgetMicroSeconds`

**A10.** `VL53L0X_SetDeviceAddress(dev, addr)`:
a) nhận địa chỉ 7-bit và tự cập nhật `dev->I2cDevAddr`
b) nhận địa chỉ 8-bit và tự cập nhật `dev->I2cDevAddr`
c) nhận địa chỉ 8-bit và **không** cập nhật `dev->I2cDevAddr`
d) nhận địa chỉ 7-bit và không cập nhật

**A11.** Tăng timing budget lên 4 lần thì độ lệch chuẩn của số đo:
a) giảm 4 lần  b) giảm khoảng 2 lần  c) giảm 16 lần  d) không đổi

**A12.** Chế độ `TOF_MODE_SEQUENTIAL` nên dùng khi:
a) muốn tốc độ đo cao nhất  b) vùng nhìn của các cảm biến chồng lên nhau  c) không nối GPIO1  d) chỉ có 1 cảm biến

**A13.** `last_error = -20` có nghĩa:
a) tham số sai  b) hết thời gian chờ số đo  c) cảm biến không trả lời I2C  d) lỗi hiệu chuẩn SPAD

**A14.** Keil MDK bản Lite (không license) giới hạn kích thước code là:
a) 16 KB  b) 32 KB  c) 64 KB  d) không giới hạn

**A15.** Cảm biến báo có số đo qua chân GPIO1 và bạn dùng ngắt EXTI. Trong hàm callback của ngắt nên làm gì?
a) gọi `TOF_Poll`  b) gọi `TOF_Restart`  c) gọi `TOF_CalibrateOffset`  d) không gọi hàm nào của thư viện; chỉ đặt một biến cờ

**A16.** Sau khi thư viện khởi tạo, GPIO1 báo "có số đo mới" bằng:
a) mức cao  b) mức thấp  c) một xung 1 µs  d) không báo gì

**A17.** Trong cửa sổ Watch, biến nào cho biết chương trình **đang chạy** (không treo)?
a) `g_tof.init_ok`  b) `g_tof.sysclk_hz`  c) `g_tof.loop_count`  d) `g_tof.mode`

**A18.** Trên STM32F103C8, địa chỉ 0x0800FC00 là:
a) đầu flash  b) đầu trang flash cuối cùng của 64 KB  c) đầu RAM  d) một thanh ghi ngoại vi

**A19.** Sau `TOF_Restart(&bus, i)`, chu kỳ VCSEL mà bạn tự đặt trước đó bằng API ST sẽ:
a) giữ nguyên  b) mất, trở về mặc định  c) được lưu vào flash  d) gây lỗi −4

**A20.** Vì sao project phải bật chế độ C99 trong Keil?
a) vì các ví dụ dùng *designated initializer* (`.addr = 0x54`)
b) vì lõi ST dùng macro có tham số biến đổi (`...`)
c) cả a và b
d) không cần thiết

---

## Phần B — Tính toán (TH, VD)

**B1.** Vật cách cảm biến 1,5 m. Tính thời gian ánh sáng đi và về. Muốn phân biệt chênh lệch 2 mm thì cảm biến phải phân biệt được chênh lệch thời gian bao nhiêu?

**B2.** Tính thời gian một lần `VL53L0X_RdByte` (đọc 1 thanh ghi) ở 100 kHz và ở 400 kHz. Bỏ qua thời gian START/STOP.

**B3.** Một lần `tof_read()` khi `sigma_check = 0` gồm 4 giao dịch: đọc 12 byte từ 0x14; ghi 0x0B = 0x01; ghi 0x0B = 0x00; đọc 1 byte 0x13. Tính tổng thời gian trên bus ở 100 kHz.

**B4.** Chuyển đổi định dạng số:
a) Giới hạn signal 0,15 MCPS sang 16.16 và sang 9.7.
b) Giá trị 9.7 đọc được là 0x01C0. Bằng bao nhiêu MCPS?
c) `EffectiveSpadRtnCount` = 0x0C40 (8.8). Bằng bao nhiêu SPAD?
d) Muốn hiệu chuẩn ở 150 mm thì tham số `CalDistanceMilliMeter` (16.16) bằng bao nhiêu?

**B5.** Thanh ghi offset 0x28 (12 bit, bước 0,25 mm):
a) Mã hoá offset +4 mm.
b) Mã hoá offset −12,5 mm.
c) Thanh ghi đọc ra 0xFD8. Offset bằng bao nhiêu mm?

**B6.** Hai cảm biến đặt song song, cách nhau 10 cm, cùng nhìn về phía trước, góc nhìn 25°.
a) Từ khoảng cách nào trở đi thì vùng nhìn của hai cảm biến bắt đầu chồng nhau?
b) Ở 1 m, đường kính vùng nhìn của một cảm biến là bao nhiêu?

**B7.** Tính `first_check_ms`, `stale_ms` và thời gian tối thiểu để một cảm biến **ngừng trả dữ liệu** chuyển sang ERROR, với:
a) budget 33 ms  b) budget 50 ms

**B8.** Bốn cảm biến chạy song song, mỗi con ~30 Hz; mỗi lần đọc tốn 2,6 ms bus.
a) Tính phần trăm thời gian bus bận do việc đọc số đo.
b) Muốn tải bus không quá 50% thì được tối đa bao nhiêu cảm biến?
c) Trên bus còn có LCD, cứ 250 ms vẽ lại mất 17 ms. Làm lại câu b.

**B9.** Ba cảm biến chạy chế độ tuần tự, budget 33 ms. Mỗi lượt tốn thêm: đọc 2,6 ms + bật đo 3,5 ms + trễ phát hiện ~1 ms. Ước lượng tần số đo của mỗi cảm biến.

**B10.**
a) Địa chỉ 7-bit 0x2C ứng với địa chỉ 8-bit nào?
b) Khi gọi `VL53L0X_SetDeviceAddress(dev, 0x58)`, giá trị nào được ghi vào thanh ghi 0x8A?

**B11.** `now = 0x00000010`, `next_ms = 0xFFFFFFF8` (tick vừa quay vòng). Tính `(int32_t)(now - next_ms)`. Hàm `tof_due()` cho kết quả "đã tới hạn" hay chưa?

**B12.** Ước lượng RAM cho 6 cảm biến: mỗi `TOF_Sensor` ~470 byte, `TOF_Bus` ~40 byte, hai bảng cấu hình của ST ~700 byte. Chiếm bao nhiêu phần trăm RAM 20 KB của F103C8?

**B13.**
a) `CalibBlock` cho **4** cảm biến (`TOF_Calib` 16 byte) có kích thước bao nhiêu byte, và bao nhiêu word?
b) Trang flash số 60 của STM32F103C8 (trang 1 KB, đánh số từ 0) bắt đầu ở địa chỉ nào?

**B14.** `TOF_CalibrateOffset` đo 50 lần. Ước lượng thời gian chặn CPU với budget 33 ms và với budget 50 ms (bỏ qua chi phí phụ).

**B15.** Đo bằng DWT ở 72 MHz: `CYCCNT` tăng thêm 216 000. Bao nhiêu µs? Sau bao lâu `CYCCNT` quay vòng?

---

## Phần C — Đọc code, tìm lỗi (VD)

Mỗi đoạn có **ít nhất một lỗi**. Chỉ ra lỗi, giải thích hậu quả và sửa lại.

**C1.** Tự viết đoạn gán địa chỉ cho 2 cảm biến:
```c
HAL_GPIO_WritePin(XSHUT1_GPIO_Port, XSHUT1_Pin, GPIO_PIN_SET);
HAL_Delay(2);
devA.I2cHandle = &hi2c1;  devA.I2cDevAddr = 0x52;
VL53L0X_SetDeviceAddress(&devA, 0x54);
VL53L0X_DataInit(&devA);
```

**C2.**
```c
VL53L0X_PerformSingleRangingMeasurement(&dev0, &m);
uint8_t ready = 0;
while (!ready) VL53L0X_GetMeasurementDataReady(&dev0, &ready);   /* cho co du lieu */
```

**C3.** Muốn kiểm tra cảm biến chỉ số 1 có số đo mới:
```c
if (TOF_Poll(&bus) == 1) { dist1 = tof[1].range_mm; }
```

**C4.**
```c
uint16_t mm;
TOF_Poll(&bus);
mm = tof[0].range_mm;
if (mm < 200) motor_stop();
```

**C5.**
```c
TOF_Sensor tof[3] = {
    { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_0, .addr = 0x54 },
    { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_1, .addr = 0x53 },
    { .xshut_port = GPIOA, .xshut_pin = GPIO_PIN_2, .addr = 0x54 },
};
```

**C6.**
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == TOF1_INT_Pin) TOF_Poll(&bus);
}
```

**C7.** Tự đọc thanh ghi 16 bit chứa khoảng cách:
```c
uint8_t buf[2];
uint16_t mm;
HAL_I2C_Mem_Read(&hi2c1, 0x54, 0x1E, I2C_MEMADD_SIZE_8BIT, buf, 2, 10);
memcpy(&mm, buf, 2);
```

**C8.**
```c
bus.budget_us = 33000;
TOF_Init(&bus);
tof_apply_profile(&tof[0], PROF_HIGH_ACCURACY);   /* budget 200 ms */
```

**C9.**
```c
static uint32_t t_next = 0;
if (HAL_GetTick() > t_next) {      /* moi 1 giay */
    t_next = HAL_GetTick() + 1000;
    toggle_led();
}
```

**C10.** Bộ lọc trung bình 4 mẫu:
```c
static uint16_t buf[4]; static uint8_t pos;
uint16_t avg(uint16_t v) {
    buf[pos++ & 3] = v;
    return (buf[0] + buf[1] + buf[2] + buf[3]) / 4;
}
```

---

## Phần D — Chẩn đoán qua cửa sổ Watch (VD, VDC)

Dùng chương trình `app_tof_dbg.c` với 2 cảm biến. Với mỗi tình huống, cho biết **nguyên nhân có khả năng nhất** và **cách kiểm tra / khắc phục**.

**D1.** `scan_before_n = 1`, `scan_before[0] = 0x52`; `init_ok = 1`; `s[0].state = 1`; `s[1].state = 2`, `s[1].last_error = -20`; `scan_after_n = 1`, `scan_after[0] = 0x54`.

**D2.** `init_ok = 2`; `rate_hz` ≈ 30 cả hai; nhưng `range_status` luôn bằng 2 và `range_mm` ≈ 8190, kể cả khi đưa tay sát cảm biến.

**D3.** Mọi giá trị trong Watch đều **đứng yên**, kể cả `loop_count`. LED PC13 vẫn nháy đều.

**D4.** Mọi thứ bình thường, nhưng `poll_us_max` ≈ 12 000 µs.

**D5.** `s[0].restarts` tăng khoảng mỗi 1–2 giây; `s[0].last_error = -7`; giữa các lần đó cảm biến 0 vẫn có lúc `state = 1` nhưng `rate_hz = 0`. Cảm biến 1 bình thường. Quét I2C vẫn thấy 0x54.

**D6.** `sysclk_hz = 8000000`.

**D7.** `mode = 1`, `rate_hz` ≈ 13–15 mỗi cảm biến, trong khi bạn muốn ~30 Hz.

---

## Phần E — Lập trình (VD, VDC)

Dùng `tof[]` và `bus` như VD2 (quyển 3) khi đề không ghi khác.

**E1.** Viết hàm `uint32_t tof_obstacle_mask(uint16_t thr_mm)` trả về bitmask các cảm biến có số đo **hợp lệ, mới (≤ 100 ms)** và **nhỏ hơn** `thr_mm`.

**E2.** Viết bộ lọc trung bình trượt **8 mẫu riêng cho từng cảm biến** (2 cảm biến). Chỉ đưa mẫu vào bộ lọc khi `TOF_Poll` báo cảm biến đó có số đo mới **và** hợp lệ.

**E3.** Viết hàm ước lượng **tốc độ tiến lại gần** (mm/s, dương khi vật tiến lại) của cảm biến 0, từ hai số đo hợp lệ liên tiếp, dùng `timestamp_ms`.

**E4.** Thêm cảm biến thứ ba: XSHUT ở PA2, **không nối GPIO1**. Liệt kê mọi chỗ phải thay đổi (CubeMX và code).

**E5.** Viết hàm `const char *tof_err_str(VL53L0X_Error e)` trả chuỗi tiếng Việt không dấu cho các mã lỗi 0, −4, −6, −7, −20, −30, −50; mã khác trả "loi khac". Không dùng các hàm chuỗi của ST.

**E6.** Viết hàm `uint8_t tof_switch_mode(TOF_Mode m)` đổi chế độ đo khi chương trình đang chạy, trả về số cảm biến chạy được.

**E7.** Viết hàm `VL53L0X_Error tof_restart_keep_profile(uint8_t i, TOF_Profile p)`: restart cảm biến i rồi áp dụng lại profile p, dùng `tof_apply_profile` của VD10.

**E8.** Viết đoạn code đếm **tần số đo thực tế** (Hz) của từng cảm biến, cập nhật mỗi giây vào mảng `volatile uint16_t rate_hz[2]`.

---

## Phần F — Dự án nhỏ (VDC)

**F1. Robot tránh vật cản 3 hướng.** Ba cảm biến gắn ở trước-trái (lệch −45°), giữa (0°), trước-phải (+45°). Thiết kế:
a) Chọn chế độ đo song song hay tuần tự. Giải thích.
b) Viết hàm `Action decide(void)` trả về `GO`, `TURN_LEFT`, `TURN_RIGHT` hoặc `BACK`, với ngưỡng an toàn 250 mm phía trước và 150 mm hai bên.
c) Xử lý trường hợp một cảm biến hỏng.

**F2. Trạm đo chiều cao và cân nặng.** Một VL53L0X gắn trên trần ở độ cao `H = 2000 mm`, nhìn thẳng xuống. Bàn cân HX711 bên dưới. LCD 16x2 trên cùng bus I2C với cảm biến.
a) Viết công thức chiều cao. Vì sao nên dùng profile *chính xác cao*?
b) Vẽ (bằng chữ) kiến trúc vòng lặp chính, sao cho LCD, HX711 và VL53L0X không làm trễ lẫn nhau.
c) Viết hàm `uint16_t height_mm(void)` có lọc trung bình và kiểm tra hợp lệ.
d) Khi nào hiển thị kết quả "ổn định"?

---

**Hết quyển 4.** Lời giải: [Quyển 5](05_LoiGiai.md).

# Quyển 5 — LỜI GIẢI CHI TIẾT

> Lời giải cho [Quyển 4](04_BaiTap.md). Mỗi bài trình bày theo thứ tự: **đáp án → lập luận → kiến thức liên quan.**
> ✅ Code ở phần E và F đã được biên dịch (`-std=c99 -Wall -Wextra -pedantic`, không cảnh báo) và chạy trên bộ giả lập. Các lỗi mẫu C7, C9, C10 đã được tái hiện bằng code.

---

## Phần A — Trắc nghiệm

| Câu | Đáp án | Giải thích ngắn |
|---|---|---|
| A1 | **b** | 0x29 (7-bit) << 1 = 0x52 (8-bit, dạng HAL dùng). 0xEE là mã ID chứ không phải địa chỉ [Q1 §2.2] |
| A2 | **b** | XSHUT thấp: chip tắt hẳn, mất cả địa chỉ đã đổi [Q1 §1.2] |
| A3 | **b** | `TOF_Poll` là hàm không chặn, gọi mỗi vòng lặp. Các hàm còn lại chặn CPU lâu |
| A4 | **b** | Bitmask: 0x02 = bit 1 bật, tức cảm biến **chỉ số 1** (cảm biến thứ hai) |
| A5 | **c** | Signal fail: phản xạ quá yếu, thường là không có vật [Q1 §8] |
| A6 | **c** | 0x52 bị cấm (địa chỉ mặc định); 0x55 lẻ; 0x08 < 0x10 (vùng dành riêng) |
| A7 | **b** | Push-pull ép 3,3 V vào chân kéo lên 2,8 V; open-drain chỉ kéo xuống [Q1 §4.2] |
| A8 | **b** | Đọc 12 byte + ghi 0x0B hai lần + đọc 0x13 [Q1 §6.6] |
| A9 | **c** | `PerformRefSpadManagement` đổi chế độ về SINGLE nên `SetDeviceMode` phải gọi sau nó [Q1 §5.2] |
| A10 | **c** | Nhận địa chỉ 8-bit, ghi `addr/2` vào 0x8A; **không** đụng tới `Dev->I2cDevAddr` |
| A11 | **b** | Độ lệch chuẩn giảm theo √N: √4 = 2 |
| A12 | **b** | Tuần tự để chống nhiễu quang giữa các cảm biến |
| A13 | **c** | −20 = `CONTROL_INTERFACE`: lỗi I2C |
| A14 | **b** | 32 KB. Lõi ST + HAL có thể vượt mức này |
| A15 | **d** | Hàm thư viện có `HAL_Delay`, I2C chặn CPU, dùng chung struct. Trong ngắt chỉ đặt cờ |
| A16 | **b** | `StaticInit` cấu hình GPIO1: "có mẫu mới", tích cực mức thấp |
| A17 | **c** | `loop_count` tăng mỗi vòng lặp; các biến khác đứng yên khi chạy bình thường |
| A18 | **b** | 0x08000000 + 63 × 1024 = 0x0800FC00 |
| A19 | **b** | Restart khởi tạo lại từ đầu, chỉ áp dụng các tham số trong `TOF_Bus` [Q2 §C.4] |
| A20 | **c** | Lõi ST bắt buộc C99 (macro variadic); các ví dụ cũng dùng designated initializer |

---

## Phần B — Tính toán

### B1
$$t = \frac{2d}{c} = \frac{2 \times 1{,}5}{3\times10^8} = 10^{-8}\ \text{s} = \mathbf{10\ ns}$$
Chênh lệch 2 mm: $\Delta t = \dfrac{2 \times 0{,}002}{3\times10^8} \approx \mathbf{13{,}3\ ps}$.

**Nhận xét:** cảm biến phải phân biệt được chênh lệch cỡ chục pico giây. Đó là lý do cần SPAD và phép thống kê trên rất nhiều xung.

### B2
`RdByte` = Mem_Read 1 byte: địa chỉ W + thanh ghi + địa chỉ R + dữ liệu = **4 byte** = 36 bit.
- 100 kHz (10 µs/bit): 36 × 10 = **360 µs**
- 400 kHz (2,5 µs/bit): 36 × 2,5 = **90 µs**

### B3

| Giao dịch | Byte trên bus | Bit | Thời gian |
|---|---|---|---|
| Đọc 12 byte từ 0x14 | 3 + 12 = 15 | 135 | 1 350 µs |
| Ghi 0x0B = 0x01 | 3 | 27 | 270 µs |
| Ghi 0x0B = 0x00 | 3 | 27 | 270 µs |
| Đọc 0x13 | 4 | 36 | 360 µs |
| **Tổng** | | | **2 250 µs ≈ 2,25 ms** |

Cộng thêm START/STOP và thời gian HAL xử lý (~0,3 ms), ra ≈ 2,6 ms, khớp với 🔬 giả lập.

### B4
a) 16.16: 0,15 × 65536 = 9830,4 → **9830** (0x2666). 9.7: 0,15 × 128 = 19,2 → **19** (0x13); làm tròn còn 19/128 ≈ 0,148 MCPS.
b) 0x01C0 = 448 → 448 / 128 = **3,5 MCPS**.
c) 0x0C40 = 3136 → 3136 / 256 = **12,25 SPAD**.
d) `150 << 16` = 150 × 65536 = **9 830 400** (0x00960000).

### B5
a) +4 mm = 4000 µm → 4000 / 250 = **16 = 0x010**.
b) −12,5 mm = −12500 µm → −12500 / 250 = −50 → 4096 − 50 = **4046 = 0xFCE**.
c) 0xFD8 = 4056 > 2047, nên là số âm: 4056 − 4096 = −40 → −40 × 0,25 = **−10 mm**.

### B6
a) Nửa độ rộng vùng nhìn của mỗi cảm biến tại khoảng cách d: $r = d \cdot \tan 12{,}5^\circ \approx 0{,}2217\,d$.
Hai vùng nhìn bắt đầu chồng nhau khi tổng hai nửa độ rộng vượt khoảng cách giữa hai cảm biến:
$$2 \times 0{,}2217\, d > 100\ \text{mm} \;\Rightarrow\; d > \mathbf{225{,}5\ mm}$$
b) $D = 2 \times 1000 \times \tan 12{,}5^\circ \approx \mathbf{443\ mm}$.

**Kết luận:** mọi vật xa hơn ~23 cm đều có thể bị cả hai cảm biến nhìn thấy, nên nên dùng chế độ tuần tự.

### B7
Công thức (phép chia nguyên của C):
- `first_check = budget_us × 3 / 4000`
- `stale = budget_us / 1000 × 4 + 10`

| Budget | first_check | stale | Thời gian đến ERROR (5 lần quá hạn liên tiếp) |
|---|---|---|---|
| a) 33 ms | 99000 / 4000 = **24 ms** | 33 × 4 + 10 = **142 ms** | ≈ 5 × 142 = **~710 ms** |
| b) 50 ms | 150000 / 4000 = **37 ms** | 50 × 4 + 10 = **210 ms** | ≈ 5 × 210 = **~1050 ms** |

Vì sao nhân 5: mỗi lần quá hạn, `tof_service` gọi `tof_fail` và đặt lại `start_ms = now`, nên lần quá hạn tiếp theo cách thêm `stale` ms nữa. Khi không có GPIO1, mỗi lần phát hiện có thể trễ thêm tối đa `poll_ms`.

### B8
a) 4 × 30 × 2,6 ms = 312 ms mỗi giây = **31,2 %**.
b) N × 30 × 2,6 ms ≤ 500 ms → N ≤ 6,41 → **N = 6**.
c) LCD chiếm 17 / 250 = 6,8 %. Phần còn lại là 43,2 % → N ≤ 43,2 / 7,8 = 5,54 → **N = 5**.

### B9
Chu kỳ một lượt = 33 + 2,6 + 3,5 + 1 = 40,1 ms. Một vòng 3 cảm biến = 3 × 40,1 = 120,3 ms.
Tần số mỗi cảm biến ≈ 1 / 0,1203 ≈ **8,3 Hz**. Ở chế độ song song mỗi con được ~30 Hz.

### B10
a) 0x2C << 1 = **0x58**.
b) ST ghi `0x58 / 2` = **0x2C** vào thanh ghi 0x8A (thanh ghi này lưu địa chỉ dạng 7-bit).

### B11
`now - next_ms` = 0x00000010 − 0xFFFFFFF8. Trên 32 bit không dấu, kết quả = 0x10 + 0x08 = **0x18 = 24**.
`(int32_t)24` = 24 ≥ 0, nên **đã tới hạn** (và đã quá hạn 24 ms). Phép trừ không dấu rồi ép về có dấu vẫn đúng khi tick quay vòng, miễn là khoảng cách thật giữa hai mốc nhỏ hơn 2³¹ ms (~24,8 ngày).

### B12
6 × 470 + 40 + 700 = **3 560 byte ≈ 3,5 KB** → 3560 / 20480 ≈ **17,4 %** RAM.

### B13
a) 4 (magic) + 4 × 16 + 4 (checksum) = **72 byte = 18 word**.
b) 0x08000000 + 60 × 1024 = 0x08000000 + 0xF000 = **0x0800F000**.

### B14
- Budget 33 ms: 50 × 33 = **1,65 s**.
- Budget 50 ms: 50 × 50 = **2,5 s**.

Cộng thêm ~1 budget chờ lần đo cũ kết thúc và chi phí I2C. 🔬 Giả lập (30 ms/lần đo) mất 1,84 s.

### B15
- 216 000 / 72 = **3 000 µs = 3 ms**.
- Quay vòng sau 2³² / 72×10⁶ ≈ **59,65 s**. Vì vậy chỉ dùng DWT để đo các khoảng thời gian ngắn hơn ~1 phút; phép trừ không dấu vẫn đúng qua một lần quay vòng.

---

## Phần C — Đọc code, tìm lỗi

### C1
**Lỗi 1:** sau `SetDeviceAddress`, quên cập nhật `devA.I2cDevAddr = 0x54`. ST không tự cập nhật trường này. `DataInit` vẫn gửi tới 0x52, nơi không còn ai, nên nhận NACK và lỗi −20.
**Lỗi 2:** không kiểm tra mã trả về của từng lệnh, và không đọc lại ID để xác nhận địa chỉ mới.
**Lỗi 3** (ngầm): đoạn code không bảo đảm cảm biến kia đang tắt (XSHUT2 thấp) trước khi bật cảm biến A.

```c
HAL_GPIO_WritePin(XSHUT2_GPIO_Port, XSHUT2_Pin, GPIO_PIN_RESET);  /* con kia phai tat */
HAL_GPIO_WritePin(XSHUT1_GPIO_Port, XSHUT1_Pin, GPIO_PIN_SET);
HAL_Delay(2);
devA.I2cHandle = &hi2c1;  devA.I2cDevAddr = 0x52;
st = VL53L0X_SetDeviceAddress(&devA, 0x54);
if (st == VL53L0X_ERROR_NONE) { devA.I2cDevAddr = 0x54; st = VL53L0X_DataInit(&devA); }
```
💡 `tof_wake_and_address()` làm đúng các bước này, kèm đọc ID trước và sau khi đổi địa chỉ.

### C2
**Lỗi:** `PerformSingleRangingMeasurement` đã tự chờ, tự đọc **và tự xoá cờ ngắt**. Sau đó `GetMeasurementDataReady` không bao giờ trả `ready = 1`, nên vòng `while` **lặp vô hạn**. Tài liệu của ST cũng ghi rõ hai hàm này không dùng chung được.
**Sửa:** dùng luôn `m` sau `PerformSingleRangingMeasurement`. Hoặc dùng luồng không chặn: `StartMeasurement` → `GetMeasurementDataReady` → `GetRangingMeasurementData` → `ClearInterruptMask`. `tof_multi` đi theo luồng này.

### C3
**Lỗi 1:** `TOF_Poll` trả **bitmask**. Cảm biến chỉ số 1 ứng với bit 1, tức giá trị 0x02. Điều kiện `== 1` thực ra kiểm tra cảm biến 0, và còn sai khi nhiều bit cùng bật.
**Lỗi 2:** không kiểm tra `range_status`.
**Lỗi 3** (hay gặp): gọi `TOF_Poll` nhiều lần trong một vòng lặp. Bit của lần gọi trước bị mất. Hãy lưu kết quả vào biến.
```c
uint32_t upd = TOF_Poll(&bus);
if ((upd & (1UL << 1)) && tof[1].range_status == 0) dist1 = tof[1].range_mm;
```

### C4
**Lỗi:** dùng `range_mm` mà **không kiểm tra** trạng thái, độ hợp lệ hay tuổi của số đo. Hậu quả:
- Cảm biến lỗi thì `range_mm` giữ **giá trị cũ**. Nếu giá trị cũ là 500 trong khi vật đang lao tới, xe **không dừng**. Rất nguy hiểm.
- Ngay sau `TOF_Restart`, `range_mm = 0` (< 200), nên xe dừng vô cớ.
- Không có vật thì `range_mm ≈ 8190`. Vô tình vẫn đúng, nhưng chỉ là may.

**Sửa:** với điều khiển an toàn, coi "không có số đo tin cậy" là nguy hiểm (*fail-safe*):
```c
uint16_t mm;
if (TOF_GetRange(&bus, 0, 100, &mm)) {
    if (mm < 200) motor_stop();
} else if (!(tof[0].state == TOF_STATE_RUNNING &&
             (tof[0].range_status == 2 || tof[0].range_status == 4))) {
    motor_stop();      /* loi / qua cu / qua gan -> dung cho an toan */
}
```

### C5
- 0x53 là **số lẻ**, bị `tof_addr_valid` từ chối (−4).
- Cảm biến thứ ba **trùng** 0x54 với cảm biến đầu, cũng bị từ chối (−4).

Chỉ cảm biến đầu chạy. **Sửa:** 0x54, 0x56, 0x58.

### C6
**Lỗi:** gọi `TOF_Poll` **trong hàm ngắt**.
- I2C ở chế độ polling chặn CPU tới vài ms **bên trong ngắt**, làm trễ mọi ngắt khác.
- Nếu `TOF_Poll` (hoặc `TOF_Restart`, có `HAL_Delay`) chạy trong ngắt có mức ưu tiên ≥ SysTick, `HAL_GetTick` không tăng, nên chương trình **treo**.
- Vòng lặp chính cũng đang gọi `TOF_Poll`. Hai lời gọi lồng nhau tranh cùng handle I2C và cùng struct: `HAL_BUSY`, dữ liệu hỏng.

**Sửa:** trong ngắt chỉ đặt một cờ `volatile`. Thực ra thư viện đã tự đọc mức chân GPIO1 trong `TOF_Poll`, nên **không cần EXTI**.

### C7
**Lỗi:** `memcpy` vào `uint16_t` trên STM32 (little-endian) làm **đảo byte**. Ví dụ chip gửi 0x01, 0xF4 (500 mm), `memcpy` ra **62465**.
🔬 Đã tái hiện: `memcpy` → 62465, ghép đúng → 500.
**Sửa:** `mm = ((uint16_t)buf[0] << 8) | buf[1];`, kiểm tra giá trị trả về của HAL, và kiểm tra `RangeStatus`. Đơn giản nhất là dùng thư viện.

### C8
**Lỗi:** profile 200 ms nhưng `bus.budget_us` vẫn là 33 000. Thư viện tính `stale` = 142 ms < 200 ms, nên cứ mỗi số đo lại bị coi là quá hạn: TIME_OUT, rồi ERROR. Nếu có chương trình tự restart, restart lại xoá profile, tạo vòng lặp lỗi.
**Sửa:** `bus.budget_us = 200000;` **trước** khi áp profile.
⚠️ **Hệ quả cần biết** (mức VDC): `bus.budget_us` **dùng chung** cho mọi cảm biến trên bus. Nếu chỉ một con dùng 200 ms, các con 33 ms còn lại (không có GPIO1) chỉ được hỏi sau `first_check` = 150 ms, nên tốc độ tụt xuống ~6 Hz. Có hai cách:
- Nối GPIO1 cho mọi cảm biến: `tof_due` luôn đúng, không phụ thuộc `first_check`.
- Hoặc dùng cùng một profile cho cả bus.

### C9
**Lỗi:** `t_next = HAL_GetTick() + 1000` có thể **tràn số** khi tick gần 2³². Sau khi tràn, `t_next` nhỏ, nên `HAL_GetTick() > t_next` **luôn đúng**: đoạn code chạy **mỗi lần lặp** cho tới khi chính tick quay vòng. Lỗi chỉ xuất hiện sau ~49,7 ngày chạy liên tục, rất khó phát hiện khi thử nghiệm.
🔬 Đã tái hiện: trong 2 s quanh điểm quay vòng, cách sai kích hoạt **23 lần**, cách đúng 2 lần.
**Sửa:**
```c
static uint32_t t_last = 0;
if (HAL_GetTick() - t_last >= 1000U) { t_last = HAL_GetTick(); toggle_led(); }
```

### C10
**Lỗi:** luôn chia cho 4, kể cả khi bộ đệm chưa đủ 4 mẫu. Mẫu đầu 400 mm ra trung bình **100 mm**: báo "có vật cản" giả.
🔬 Đã tái hiện: mẫu đầu 400 → trung bình 100.
**Sửa:** đếm số mẫu đã có, như `avg_push` ở VD6, hoặc như lời giải E2.

---

## Phần D — Chẩn đoán qua Watch

### D1 — XSHUT của TOF2 không có tác dụng
**Dấu hiệu then chốt:** `scan_before_n = 1` với 0x52, khi lẽ ra cả hai XSHUT đang thấp. Vậy có một cảm biến **đang bật dù XSHUT "thấp"**.

**Diễn biến:**
1. TOF2 luôn bật ở 0x52.
2. Thư viện bật TOF1: cả hai cùng ở 0x52, **cùng nhận** lệnh đổi địa chỉ, nên **cả hai sang 0x54**.
3. Đọc lại ID ở 0x54 vẫn ra 0xEE (hai chip gửi cùng một giá trị), nên TOF1 "OK".
4. Tới lượt TOF2: 0x52 không còn ai, nên −20.

`scan_after` chỉ thấy 0x54 vì hai chip cùng chiếm địa chỉ này.

**Hậu quả:** số đo của "TOF1" là dữ liệu trộn của hai chip, **không tin được**.
**Kiểm tra:** dây PA1 tới XSHUT của TOF2; label `XSHUT2` có đúng PA1 không; đo điện áp chân XSHUT2 khi đang giữ thấp (phải ~0 V).

### D2 — Có thấy cảm biến nhưng "không thấy vật"
I2C, địa chỉ và tốc độ đo đều bình thường. Cảm biến **đo được**, nhưng tín hiệu phản xạ luôn quá yếu (status 2).
**Nguyên nhân khả dĩ nhất:** **miếng dán bảo vệ** màu vàng/cam trên mắt cảm biến chưa bóc. Ngoài ra có thể do kính che bẩn, hoặc băng dính dán lên mắt.
**Khắc phục:** bóc miếng dán, lau mắt cảm biến, rồi thử lại với tay ở 10 cm.

### D3 — Watch đứng yên nhưng LED vẫn nháy
LED nháy chứng tỏ **chương trình vẫn chạy**: LED do `App_TOF_Dbg_Loop` đảo trạng thái mỗi giây. Vậy chỉ là **Watch không cập nhật**.
**Nguyên nhân:** chưa bật *View → Periodic Window Update*, nên Watch chỉ cập nhật khi CPU dừng. Cũng có thể debugger đã mất kết nối (xem thanh trạng thái).
**Khắc phục:** bật Periodic Window Update.

### D4 — `poll_us_max` ≈ 12 ms
Mức bình thường là ≤ 3 ms. 12 ms khớp với chi phí đọc khi **bật sigma** (23 giao dịch, ~10–12 ms) [Q1 §6.6].
**Nguyên nhân khả dĩ:**
1. `sigma_check = 1`.
2. Ít gặp hơn: ai đó tăng `TOF_MAX_READS_PER_POLL`, hoặc I2C bị cấu hình chậm hơn 100 kHz (kiểm tra CubeMX).

`poll_us_max` chỉ đo `TOF_Poll`, không tính `TOF_Restart` hay hiệu chuẩn, nên không phải do hai việc đó.
**Khắc phục:** `sigma_check = 0`, hoặc chấp nhận 12 ms nếu cần lọc sigma.

### D5 — Lỗi −7 lặp lại, I2C vẫn tốt
- `last_error = -7` (TIME_OUT) chứ không phải −20: **I2C vẫn trả lời** (quét vẫn thấy 0x54), nhưng **không bao giờ đọc được số đo**.
- `restarts` tăng mỗi 1–2 s: khớp với ~0,71 s để chuyển sang ERROR [B7] cộng tối đa 1 s chờ lần kiểm tra của supervisor.

**Nguyên nhân khả dĩ nhất:** **dây GPIO1 của TOF1 bị đứt hoặc nối sai chân**. Điện trở kéo lên giữ chân ở mức cao, nên thư viện tưởng "chưa có dữ liệu" mãi.
**Kiểm tra:** đặt `tof[0].gpio1_port = NULL` (hỏi qua I2C). Nếu cảm biến chạy lại bình thường thì đúng là do dây GPIO1. Kiểm tra cả label `TOF1_INT` trong CubeMX.

### D6 — `sysclk_hz = 8 000 000`
Chip đang chạy bằng **HSI 8 MHz**, không phải 72 MHz.
**Nguyên nhân:** chưa chọn HSE Crystal hoặc chưa cấu hình PLL ×9 trong CubeMX; hoặc thạch anh trên board (hàng nhái) không dao động.
**Ảnh hưởng:** mọi thứ chậm hơn ~9 lần. I2C vẫn đúng tốc độ, vì HAL tính theo clock thực tế lúc khởi tạo. Nếu dùng UART với cấu hình 72 MHz thì baud sẽ sai.
**Khắc phục:** cấu hình lại RCC/Clock; kiểm tra mối hàn thạch anh 8 MHz.

### D7 — Đang ở chế độ tuần tự
`mode = 1`, tức tuần tự, nên tốc độ chia đều cho 2 cảm biến (~13–15 Hz mỗi con).
**Khắc phục:**
- Vùng nhìn **không** chồng nhau: chuyển sang song song (`cmd = 3`, hoặc `bus.mode = TOF_MODE_PARALLEL`).
- Vùng nhìn chồng nhau mà vẫn cần nhanh: giảm budget xuống 20 ms (~20 Hz mỗi con), hoặc xoay các cảm biến lệch nhau > 25°.

---

## Phần E — Lập trình

### E1 — Bitmask vật cản
```c
uint32_t tof_obstacle_mask(uint16_t thr_mm)
{
    uint32_t mask = 0;
    uint16_t mm;
    uint8_t  i;

    for (i = 0; i < bus.count; i++)
        if (TOF_GetRange(&bus, i, 100, &mm) && mm < thr_mm)
            mask |= 1UL << i;
    return mask;
}
```
**Giải thích:** `TOF_GetRange` đã kiểm tra cả ba điều kiện: RUNNING, hợp lệ, tuổi ≤ 100 ms. Toán tử `&&` **ngắn mạch**: nếu `TOF_GetRange` trả 0 thì không xét `mm` (lúc đó `mm` chưa được ghi).
🔬 Vật ở 150 mm và 600 mm, ngưỡng 300 mm → `0x1`.

### E2 — Trung bình trượt 8 mẫu cho từng cảm biến
```c
#define FLT_N 8U
typedef struct {
    uint16_t buf[FLT_N];
    uint8_t  pos, cnt;
    uint32_t sum;                     /* tong dang giu -> moi lan chi cong/tru 1 mau */
} MovAvg;

static MovAvg flt[2];
volatile uint16_t dist_avg[2];

static uint16_t movavg_push(MovAvg *f, uint16_t v)
{
    if (f->cnt == FLT_N)
        f->sum -= f->buf[f->pos];     /* bo mau cu nhat (dung o vi tri sap ghi) */
    else
        f->cnt++;
    f->buf[f->pos] = v;
    f->sum += v;
    f->pos = (uint8_t)((f->pos + 1U) % FLT_N);
    return (uint16_t)(f->sum / f->cnt);
}

void E2_Loop(void)
{
    uint32_t upd = TOF_Poll(&bus);
    uint8_t  i;

    for (i = 0; i < 2; i++)
        if ((upd & (1UL << i)) && tof[i].range_status == 0)
            dist_avg[i] = movavg_push(&flt[i], tof[i].range_mm);
}
```
**Giải thích:**
- **Giữ tổng chạy** (`sum`): mỗi mẫu mới chỉ cộng 1 và trừ 1, độ phức tạp O(1), thay vì cộng lại cả 8 phần tử.
- Khi bộ đệm đầy, `pos` luôn trỏ đúng vào **mẫu cũ nhất**: con trỏ vòng đi hết một vòng thì quay về chỗ ghi sớm nhất. Vì vậy trừ `buf[pos]` trước khi ghi đè.
- `cnt` giúp tránh lỗi C10.
- Mỗi cảm biến có một `MovAvg` riêng, nên các bộ lọc không lẫn nhau.

🔬 Hai vật ở 150 mm và 600 mm → `dist_avg = {150, 600}`.

### E3 — Tốc độ tiến lại gần
```c
uint8_t approach_speed(int32_t *mm_per_s)
{
    static uint16_t last_mm;
    static uint32_t last_t;
    static uint8_t  have_last = 0;
    uint16_t mm;
    uint32_t t;

    if (!TOF_GetRange(&bus, 0, 100, &mm)) {
        have_last = 0;                            /* mat so do -> lam lai tu dau */
        return 0;
    }
    t = tof[0].timestamp_ms;
    if (have_last && t != last_t) {
        uint32_t dt = t - last_t;                 /* ms, > 0 */
        *mm_per_s = ((int32_t)last_mm - (int32_t)mm) * 1000 / (int32_t)dt;
        last_mm = mm;
        last_t  = t;
        return 1;
    }
    last_mm   = mm;
    last_t    = t;
    have_last = 1;
    return 0;
}
/* Goi khi TOF_Poll bao cam bien 0 co so do moi */
```
**Giải thích:**
- **Dùng `timestamp_ms` của số đo**, không dùng thời điểm gọi hàm: đo `dt` đúng giữa hai lần đo thật.
- **Ép sang `int32_t` trước khi trừ**: vật ra xa thì kết quả âm. Trừ hai `uint16_t` cho kết quả `int` (đúng dấu), nhưng ép rõ ràng thì dễ đọc hơn.
- Tràn số? Độ chênh tối đa ±8191 mm, nhân 1000 = 8,2×10⁶, nằm trong `int32_t`.
- Mất số đo thì **làm lại từ đầu**, không ghép hai số đo cách xa nhau.

⚠️ **Nhiễu:** sai số ±3 mm mỗi lần đo, chia cho 33 ms, cho nhiễu tốc độ tới ±90 mm/s. Nên lọc khoảng cách trước (E2) hoặc tính trên khoảng thời gian dài hơn.
🔬 Vật tiến lại đều 300 mm/s → ước lượng **300 mm/s**.

### E4 — Thêm cảm biến thứ ba (PA2, không GPIO1)
1. **CubeMX:** PA2 → GPIO_Output, **Open-Drain, mức Low**, label `XSHUT3`.
2. **Phần cứng:** XSHUT của module 3 nối vào PA2. VIN, GND, SDA, SCL nối chung. GPIO1 để trống.
3. **Code:**
```c
TOF_Sensor tof[3] = {
    /* ... 2 cam bien cu ... */
    { .xshut_port = XSHUT3_GPIO_Port, .xshut_pin = XSHUT3_Pin, .addr = 0x58 },  /* gpio1 = NULL */
};
/* bus.count = 3; moi vong for 0..1 -> 0..2; mang dist_mm[2] -> [3] ... */
```
4. **Các chỗ kéo theo:**
   - Mảng kết quả, bộ lọc E2: kích thước 3.
   - `CALIB_N = 3` (VD8); khối flash thành 4 + 48 + 4 = 56 byte.
   - Nếu dùng `app_tof_dbg`: sửa `APP_TOF_N` và `tof_start()`.
   - Supervisor VD9: điều kiện "tất cả cùng lỗi" thành `n_err == 3`.
5. **Tài nguyên:** thêm ~470 byte RAM. Tải bus tăng ~8% (30 Hz × 2,6 ms + vài lần hỏi I2C).

### E5 — Chuỗi mô tả lỗi
```c
const char *tof_err_str(VL53L0X_Error e)
{
    switch (e) {
    case VL53L0X_ERROR_NONE:              return "OK";
    case VL53L0X_ERROR_INVALID_PARAMS:    return "tham so sai";
    case VL53L0X_ERROR_RANGE_ERROR:       return "khong co so do hop le";
    case VL53L0X_ERROR_TIME_OUT:          return "het thoi gian";
    case VL53L0X_ERROR_CONTROL_INTERFACE: return "khong tra loi I2C";
    case VL53L0X_ERROR_INVALID_COMMAND:   return "sai trang thai";
    case VL53L0X_ERROR_REF_SPAD_INIT:     return "loi hieu chuan SPAD";
    default:                              return "loi khac";
    }
}
```
**Giải thích:** trả con trỏ tới **chuỗi hằng** (nằm trong flash), không cần bộ đệm. An toàn hơn `VL53L0X_GetPalErrorString`, vốn cần bộ đệm ≥ 32 byte và dùng `strcpy`. Dùng tên macro thay vì số (−20…) để code dễ đọc.

### E6 — Đổi chế độ khi đang chạy
```c
uint8_t tof_switch_mode(TOF_Mode m)
{
    uint8_t i, n = 0;

    if (bus.mode != m) {
        bus.mode = m;
        return TOF_Init(&bus);        /* chan CPU ~0,2 s moi cam bien; calib duoc giu */
    }
    for (i = 0; i < bus.count; i++)   /* da dung che do -> chi dem so cam bien dang chay */
        if (tof[i].state == TOF_STATE_RUNNING)
            n++;
    return n;
}
```
**Giải thích:**
- Đổi chế độ cần đặt lại `SetDeviceMode` cho mọi cảm biến, và đổi cách bật đo. Cách an toàn nhất là khởi tạo lại bằng `TOF_Init`; hàm này giữ nguyên `calib`.
- Nếu đã đúng chế độ thì không làm gì, tránh chặn CPU vô ích.

🔬 Sang tuần tự → 2 cảm biến; gọi lại → vẫn 2, không khởi tạo lại.

### E7 — Restart giữ profile
```c
VL53L0X_Error tof_restart_keep_profile(uint8_t i, TOF_Profile p)
{
    VL53L0X_Error st = TOF_Restart(&bus, i);

    if (st == VL53L0X_ERROR_NONE && p != PROF_DEFAULT)
        st = tof_apply_profile(&tof[i], p);   /* chi o che do song song (xem VD10) */
    return st;
}
```
**Giải thích:**
- `TOF_Restart` khởi tạo lại theo cấu hình mặc định cộng `bus.budget_us`, nên phải áp lại profile.
- Nếu restart lỗi thì không áp profile (cảm biến không chạy).
- Nếu áp profile lỗi, cảm biến vẫn chạy với cấu hình mặc định, và hàm trả mã lỗi để ứng dụng biết.

💡 Nên thay `TOF_Restart` bằng hàm này trong supervisor (VD9) khi dùng profile.
🔬 Restart rồi áp profile tầm xa → VCSEL 18 / 14, trả 0.

### E8 — Đếm tần số đo
```c
volatile uint16_t rate_hz[2];

void E8_Loop(void)
{
    static uint16_t cnt[2];
    static uint32_t t0 = 0;
    uint32_t upd = TOF_Poll(&bus);
    uint8_t  i;

    for (i = 0; i < 2; i++)
        if (upd & (1UL << i))
            cnt[i]++;

    if (HAL_GetTick() - t0 >= 1000U) {
        t0 = HAL_GetTick();
        for (i = 0; i < 2; i++) {
            rate_hz[i] = cnt[i];
            cnt[i] = 0;
        }
    }
}
```
**Giải thích:** đếm số bit bật trong mỗi cửa sổ 1 s. Dùng `t0 = HAL_GetTick()` thay vì `t0 += 1000`: nếu vòng lặp bị chặn lâu (ví dụ đang hiệu chuẩn 2 s), cửa sổ không bị "đuổi" liên tục với các giá trị 0. Đổi lại, cửa sổ có thể dài hơn 1 s một chút.
🔬 `rate_hz = {34, 33}` (giả lập; mạch thật ~30).

---

## Phần F — Dự án nhỏ

### F1 — Robot tránh vật cản 3 hướng

**a) Chọn chế độ: song song.** Ba cảm biến lệch nhau 45°, mỗi cảm biến nhìn ±12,5°. Cảm biến giữa nhìn từ −12,5° đến +12,5°; cảm biến trái từ −57,5° đến −32,5°. Giữa hai vùng còn **khe 20°**, nên vùng nhìn không chồng nhau, và không cần tuần tự. Song song cho tốc độ cao nhất (~30 Hz mỗi con), rất quan trọng với robot đang chạy. Nếu thử thực tế thấy nhiễu do phản xạ từ tường, mới chuyển sang tuần tự.

**b) Hàm quyết định:**
```c
typedef enum { GO, TURN_LEFT, TURN_RIGHT, BACK } Action;

#define SAFE_FRONT_MM  250U
#define SAFE_SIDE_MM   150U
#define FAR_MM         2000U      /* "khong co vat" coi nhu rat xa */

/* Tra khoang cach; neu khong tin duoc thi tang *n_fail va tra 0 (= coi nhu co vat sat) */
static uint16_t dist_or_far(uint8_t i, uint8_t *n_fail)
{
    uint16_t mm;

    if (TOF_GetRange(&bus3, i, 100, &mm))
        return mm;
    if (tof3[i].state == TOF_STATE_RUNNING &&
        (tof3[i].range_status == 2 || tof3[i].range_status == 4))
        return FAR_MM;                            /* duong trong */
    (*n_fail)++;                                  /* loi / qua gan / qua cu */
    return 0;
}

Action decide(void)
{
    uint8_t  n_fail = 0;
    uint16_t L = dist_or_far(0, &n_fail);
    uint16_t C = dist_or_far(1, &n_fail);
    uint16_t R = dist_or_far(2, &n_fail);

    if (n_fail >= 2)                              /* khong du thong tin -> an toan la lui */
        return BACK;
    if (L < SAFE_SIDE_MM && R < SAFE_SIDE_MM)     /* bi kep hai ben */
        return BACK;
    if (C < SAFE_FRONT_MM)                        /* vuong phia truoc -> re ve phia thoang hon */
        return (L > R) ? TURN_LEFT : TURN_RIGHT;
    if (L < SAFE_SIDE_MM)
        return TURN_RIGHT;
    if (R < SAFE_SIDE_MM)
        return TURN_LEFT;
    return GO;
}
```
**Thứ tự ưu tiên:** mất thông tin → bị kẹt hai bên → vướng phía trước → vướng một bên → đi thẳng. Trường hợp nguy hiểm nhất được xét trước.

🔬 Kết quả trên giả lập:

| L | C | R | Quyết định |
|---|---|---|---|
| 800 | 900 | 800 | GO |
| 800 | 200 | 500 | TURN_LEFT (bên trái thoáng hơn) |
| 300 | 200 | 900 | TURN_RIGHT |
| 100 | 900 | 800 | TURN_RIGHT |
| 800 | 900 | 100 | TURN_LEFT |
| 120 | 900 | 120 | BACK (bị kẹt) |
| 2 cảm biến hỏng | | | BACK |

**c) Khi cảm biến hỏng:**
- `dist_or_far` trả 0, tức "có vật sát". Robot **tránh hướng đó**, là phản ứng an toàn (*fail-safe*). Trạng thái "quá gần" (3) cũng trả 0, đúng vì đó là nguy hiểm thật.
- Hỏng từ 2 cảm biến trở lên thì lùi lại.
- Kết hợp supervisor (VD9) để tự restart; bật LED báo lỗi để người dùng biết.

### F2 — Trạm đo chiều cao và cân nặng

**a) Công thức:** `chiều_cao = H − d`, với `d` là khoảng cách từ cảm biến (trên trần) tới đỉnh đầu.

Vì sao nên dùng profile *chính xác cao* (budget 200 ms):
- Người đứng yên nên không cần tốc độ; 5 lần đo/giây là đủ.
- Độ lệch chuẩn giảm √(200/33) ≈ **2,5 lần** so với mặc định.

Giới hạn cần biết:
- Ở cấu hình mặc định, tầm đo ~1,2 m, nên chỉ đo được người **cao trên ~0,8 m**.
- Vùng nhìn ở 1 m rộng ~44 cm, có thể bao cả vai. Cảm biến khi đó trả về một khoảng cách "pha trộn", có thể lớn hơn khoảng cách tới đỉnh đầu.

**b) Kiến trúc vòng lặp chính** (không hàm nào được chặn CPU lâu):
```
while (1) {
    upd = TOF_Poll(&bus);                       ≤ ~3 ms, chỉ khi có số đo mới
    height_update(upd);                         vài µs
    HX711_Read_Filtered_NonBlocking(&hx,&f,&w); ~50 µs khi có mẫu mới (10 Hz)
    mỗi 250 ms: vẽ LCD (thư viện LCD đã sửa)     ~17 ms
    supervisor mỗi 1 s (VD9)
}
Không HAL_Delay trong vòng lặp; không LCD_clear; không gọi hx711_weight() (chặn ~1 s).
```

**c) Hàm chiều cao:**
```c
#define H_CEIL_MM  2000U
#define H_N        5U

static uint16_t hbuf[H_N];
static uint8_t  hcnt, hpos;
volatile uint16_t g_height_mm;

uint16_t height_update(uint32_t upd)
{
    uint16_t d;
    uint32_t sum = 0;
    uint8_t  k;

    if (!(upd & 1UL))
        return g_height_mm;                       /* chua co so do moi -> giu ket qua cu */

    if (!TOF_GetRange(&bus, 0, 500, &d) || d >= H_CEIL_MM) {
        hcnt = 0;                                 /* khong co nguoi duoi cam bien */
        hpos = 0;
        g_height_mm = 0;
        return 0;
    }
    hbuf[hpos] = (uint16_t)(H_CEIL_MM - d);       /* chieu cao = H - d */
    hpos = (uint8_t)((hpos + 1U) % H_N);
    if (hcnt < H_N)
        hcnt++;
    for (k = 0; k < hcnt; k++)
        sum += hbuf[k];
    g_height_mm = (uint16_t)(sum / hcnt);
    return g_height_mm;
}

uint16_t height_mm(void) { return g_height_mm; }
```
**Vì sao tách `height_update(upd)` và `height_mm()`?** Hàm cần biết **có số đo mới hay không**, tức kết quả của `TOF_Poll`. `TOF_Poll` chỉ nên gọi **một lần** mỗi vòng lặp [C3], nên truyền `upd` vào. `height_mm()` chỉ trả kết quả đã tính, dùng ở chỗ khác (LCD…).
- `max_age_ms = 500`: profile 200 ms nên số đo "còn mới" cho phép lâu hơn.
- `d >= H` hoặc không có số đo thì hiểu là không có người, và xoá bộ lọc.

**d) Khi nào kết quả "ổn định":**
```c
uint8_t height_stable(void)
{
    uint16_t mn = 0xFFFF, mx = 0;
    uint8_t  k;

    if (hcnt < H_N)
        return 0;                                 /* chua du 5 mau */
    for (k = 0; k < H_N; k++) {
        if (hbuf[k] < mn) mn = hbuf[k];
        if (hbuf[k] > mx) mx = hbuf[k];
    }
    return (uint8_t)((mx - mn) <= 10);            /* dao dong <= 10 mm */
}
```
Hiển thị "ỔN ĐỊNH" khi **đồng thời**:
- đủ 5 mẫu;
- chênh lệch lớn nhất − nhỏ nhất ≤ 10 mm (người đã đứng yên);
- bộ lọc cân HX711 cũng ổn định.

Khi đó có thể "chốt" kết quả lên LCD.
🔬 `d = 330 mm` → chiều cao **1670 mm**, ổn định; không có người → 0.

---

**Hết quyển 5.** Tiếp theo: [Quyển 6 — Ôn tập](06_OnThi.md).

# Quyển 6 — ÔN TẬP

> Dùng để ôn nhanh trước khi làm dự án hoặc kiểm tra. Gồm: sơ đồ tư duy, 30 điều phải nhớ, công thức, bảng tra, 20 bẫy hay gặp, 2 đề kiểm tra có đáp án và thang điểm.

---

## 1. Sơ đồ tư duy

```
                                 THƯ VIỆN VL53L0X ĐA CẢM BIẾN
                                            │
   ┌──────────────┬──────────────┬──────────┴───────┬──────────────┬──────────────┐
CẢM BIẾN        I2C          KIẾN TRÚC         NHIỀU CẢM BIẾN     ĐO KHÔNG CHẶN    LỖI & PHỤC HỒI
   │              │              │                  │                 │                │
 ToF d=ct/2    8-bit 0x52    3 lớp:             XSHUT riêng       TOF_Poll         −20 I2C
 SPAD,VCSEL    90µs/byte     tof_multi          bật từng con      bitmask          −7 timeout
 FoV 25°       big-endian    lõi ST (vá 2)      đổi địa chỉ       GPIO1 / I2C      −4 tham số
 budget √N     trang 0xFF    platform (HAL)     cập nhật          3/4 budget       5 lỗi → ERROR
 RangeStatus   100 kHz       1 Dev_t/cảm biến   I2cDevAddr        1 lần đọc/Poll   TOF_Restart
 0=OK          errata BUSY                      3 quy tắc         song song/tuần tự TOF_I2C_Recover
                                                                  sigma: 4 vs 23   mất profile
```

---

## 2. Ba mươi điều phải nhớ

**Cảm biến**
1. `d = c·t/2`; 1 mm ứng với ~6,67 ps thời gian đi và về.
2. Địa chỉ mặc định **0x52** (8-bit); ID ở thanh ghi 0xC0 = **0xEE**.
3. XSHUT thấp: chip tắt hẳn, **mất địa chỉ**. GPIO1 thấp: có số đo mới.
4. Góc nhìn 25°; vùng nhìn rộng D ≈ 0,443·d.
5. Budget 20–200 ms, mặc định 33 ms. Budget ×N thì độ lệch chuẩn giảm √N.
6. **Chỉ dùng `range_mm` khi `range_status == 0`.** Không có vật: ~8190 kèm status 2/4.

**I2C**
7. HAL dùng địa chỉ **8-bit** (= 7-bit << 1).
8. 1 byte @100 kHz = 90 µs. Đọc 1 thanh ghi ~0,4 ms; đọc 12 byte ~1,35 ms.
9. VL53L0X là big-endian: ghép byte cao trước, **không `memcpy`**.
10. Bus chung với PCF8574 thì **100 kHz**.
11. Bus treo (slave giữ SDA): 9 xung SCL + STOP + Init lại (`TOF_I2C_Recover`).

**Kiến trúc**
12. 3 lớp: `tof_multi` → lõi ST → `platform` → HAL.
13. Lõi ST chép nguyên, chỉ vá 2 lỗi (tràn số khi hiệu chuẩn; `USE_EMPTY_STRING`).
14. Mỗi cảm biến có **một `VL53L0X_Dev_t`** riêng; lõi ST không giữ trạng thái toàn cục.
15. Chỉ `vl53l0x_api.c` được include `vl53l0x_tuning.h` và `vl53l0x_interrupt_threshold_settings.h`.

**Nhiều cảm biến**
16. Mỗi con một chân XSHUT: **Open-Drain, mức Low** ban đầu.
17. Bật con i → đọc ID ở 0x52 → `SetDeviceAddress` (**8-bit**) → **tự gán `I2cDevAddr`** → đọc lại ID.
18. Ba quy tắc: gán lại mỗi lần khởi động; không ai giữ 0x52; con lỗi phải giữ XSHUT thấp.
19. Địa chỉ hợp lệ: chẵn, 0x10–0xEE, ≠ 0x52, không trùng. Tự tránh 0x4E / 0x7E (LCD).

**Khởi tạo**
20. DataInit → StaticInit → SPAD → VHV/phase → offset → **SetDeviceMode (sau SPAD)** → budget → (tắt sigma) → Start.
21. `DataInit` chỉ một lần sau mỗi lần reset chip.

**Đo không chặn**
22. `TOF_Poll` gọi mỗi vòng lặp, trả **bitmask**; lưu kết quả vào biến, chỉ gọi **một lần** mỗi vòng.
23. Không GPIO1: hỏi sau 3/4 budget, rồi mỗi `poll_ms`.
24. Mỗi lần Poll đọc **tối đa 1** cảm biến (~3 ms), xoay vòng.
25. Song song ~30 Hz mỗi con; tuần tự chống nhiễu, tốc độ chia cho N.
26. `sigma_check = 0`: 4 giao dịch / lần đọc; `= 1`: 23 giao dịch.

**Lỗi và phục hồi**
27. −20 = lỗi I2C; −7 = không có số đo; −4 = tham số / địa chỉ sai.
28. `stale` = budget×4 + 10 ms; **5 lỗi liên tiếp** thì ERROR.
29. `TOF_Restart` khởi tạo lại một con, các con khác không ảnh hưởng, nhưng **mất cấu hình riêng** (profile).
30. **Không gọi hàm thư viện trong ngắt.**

---

## 3. Công thức

| Đại lượng | Công thức | Ví dụ |
|---|---|---|
| Khoảng cách ToF | $d = c\,t/2$ | t = 2 ns → d = 300 mm |
| Đường kính vùng nhìn | $D = 2d\tan 12{,}5^\circ \approx 0{,}443\,d$ | d = 500 → D ≈ 222 mm |
| Hai cảm biến cách nhau `s`, vùng nhìn bắt đầu chồng | $d > s / (2\tan 12{,}5^\circ) \approx 2{,}26\,s$ | s = 100 → d > 226 mm |
| Thời gian I2C | $t \approx 9\,n_\text{byte} / f_\text{SCL}$ | 15 byte @100k = 1,35 ms |
| Độ lệch chuẩn theo budget | $\sigma \propto 1/\sqrt{\text{budget}}$ | ×4 budget → σ/2 |
| first_check (ms) | `budget_us × 3 / 4000` | 33000 → 24 |
| stale (ms) | `budget_us/1000 × 4 + 10` | 33000 → 142 |
| Thời gian đến ERROR khi mất dữ liệu | ≈ 5 × stale | ≈ 710 ms |
| Tải bus | $N \times f \times T_\text{đọc}$ | 4 × 30 × 2,6 ms = 31 % |
| Tốc độ tuần tự mỗi con | $1 / (N \times (\text{budget} + T_\text{phụ}))$ | 3 con, 40 ms → 8,3 Hz |
| 16.16 / 9.7 / 8.8 | x / 65536, x / 128, x / 256 | |
| Offset → thanh ghi | `v = um/250`; âm thì `4096 + um/250` | −7 mm → 0xFE4 |
| Thanh ghi → offset | v > 2047 thì (v − 4096) × 0,25 mm | 0xFF6 → −2,5 mm |
| Thời gian từ DWT | `Δcycles / (SystemCoreClock/10⁶)` µs | 216000 → 3000 µs |

---

## 4. Bảng tra nhanh

| Tra gì | Ở đâu |
|---|---|
| Khai báo, tham số các hàm `TOF_*` | [Q2 §C](02_SachThamKhao_API.md#c-hàm-của-tof_multi) |
| Hàm API của ST | [Q2 §E](02_SachThamKhao_API.md#e-các-hàm-hay-dùng-của-api-st) |
| Mã lỗi | [Q2 §G.1](02_SachThamKhao_API.md#g1-mã-lỗi-vl53l0x_error-kiểu-int8_t) |
| RangeStatus | [Q2 §G.2](02_SachThamKhao_API.md#g2-rangestatus) |
| Thanh ghi | [Q2 §G.3](02_SachThamKhao_API.md#g3-thanh-ghi-hay-gặp-trang-0) |
| Code mẫu | [Q3](03_ViDu_GiaiThich.md) |
| Biến `g_tof` khi debug | [Q2 §F.2](02_SachThamKhao_API.md#f2-app_tof_dbgc-xem-bằng-watch-của-keil) |

---

## 5. Hai mươi bẫy hay gặp

| # | Bẫy | Đúng là |
|---|---|---|
| 1 | Dùng địa chỉ 7-bit 0x29 với HAL | 0x52 (8-bit) |
| 2 | Sau `SetDeviceAddress` quên gán `I2cDevAddr` | Gán ngay, rồi đọc lại ID |
| 3 | XSHUT để mức High ban đầu, hoặc push-pull | Open-Drain, Low |
| 4 | Quên nối XSHUT một con | Hai chip cùng địa chỉ: dữ liệu trộn (Q4 D1) |
| 5 | `memcpy` thanh ghi 16 bit | Ghép byte cao trước |
| 6 | Dùng `range_mm` không xét `range_status` | `TOF_GetRange` hoặc kiểm tra `== 0` |
| 7 | `TOF_Poll(&bus) == 1` để kiểm tra cảm biến 1 | `& (1UL << 1)` |
| 8 | Gọi `TOF_Poll` nhiều lần mỗi vòng lặp | Gọi 1 lần, lưu bitmask |
| 9 | Gọi thư viện trong ngắt EXTI | Đặt cờ; thư viện tự đọc GPIO1 |
| 10 | `PerformSingleRangingMeasurement` rồi chờ `GetMeasurementDataReady` | Treo: hàm đầu đã xoá cờ |
| 11 | `SetDeviceMode` trước `PerformRefSpadManagement` | Gọi sau |
| 12 | Profile 200 ms nhưng `bus.budget_us` = 33 ms | Tăng `bus.budget_us` |
| 13 | Profile mất sau `TOF_Restart` | Áp lại (Q4 E7) |
| 14 | `t + 1000 > tick` khi tick gần tràn | `tick − t >= 1000` |
| 15 | Bộ lọc chia cố định N khi chưa đủ mẫu | Đếm số mẫu |
| 16 | Không bóc miếng dán bảo vệ | Status luôn 2 |
| 17 | Quên bật C99 trong Keil | Lỗi ở macro `LOG_FUNCTION_*` |
| 18 | Thêm cả `app_tof_test.c` và `app_tof_dbg.c` | Chỉ thêm một |
| 19 | Keil Lite (32 KB) không đủ chỗ | `-Oz`, `USE_EMPTY_STRING`, hoặc MDK Community |
| 20 | Watch đứng yên tưởng chương trình treo | Bật Periodic Window Update; nhìn `loop_count` / LED |

---

## 6. ĐỀ KIỂM TRA SỐ 1 (45 phút, 10 điểm)

### Phần I — Trắc nghiệm (4 điểm, mỗi câu 0,4)

1. Địa chỉ 7-bit 0x29 viết theo dạng HAL là: a) 0x29 b) 0x52 c) 0x58 d) 0x92
2. Đọc thanh ghi 0xC0 của VL53L0X phải được: a) 0x52 b) 0xC0 c) 0xEE d) 0x00
3. Vì sao bus I2C trong dự án chạy 100 kHz? a) VL53L0X chỉ hỗ trợ 100 kHz b) LCD PCF8574 chỉ hỗ trợ 100 kHz c) HAL không hỗ trợ 400 kHz d) để tiết kiệm điện
4. Hàm nào trả về bitmask? a) `TOF_Init` b) `TOF_Poll` c) `TOF_GetRange` d) `TOF_Restart`
5. `range_status = 2` thì `TOF_GetRange` trả về: a) 1 b) 0 c) 2 d) −2
6. Budget 20 ms thì `stale` bằng: a) 80 ms b) 90 ms c) 100 ms d) 142 ms
7. Hai cảm biến đặt sát nhau cùng hướng nên dùng: a) song song b) tuần tự c) tắt GPIO1 d) tăng `poll_ms`
8. Chế độ đo ST mà `tof_multi` đặt ở chế độ song song: a) SINGLE_RANGING b) CONTINUOUS_RANGING c) CONTINUOUS_TIMED d) SINGLE_HISTOGRAM
9. Đặt `calib.valid = 1` trước `TOF_Init` thì: a) bỏ qua mọi hiệu chuẩn b) bỏ qua đo SPAD, nạp SPAD và offset đã lưu; VHV/phase vẫn chạy lại c) chỉ nạp VHV d) không có tác dụng
10. Một cảm biến ngừng trả dữ liệu thì chuyển sang ERROR sau khoảng (budget 33 ms): a) 142 ms b) ~710 ms c) 5 s d) không bao giờ

### Phần II — Tự luận (6 điểm)

**Câu 1 (2 điểm).** Trình bày thuật toán gán địa chỉ cho 3 cảm biến (0x54, 0x56, 0x58). Lập bảng cho biết sau mỗi bước, cảm biến nào đang ở địa chỉ nào.

**Câu 2 (2 điểm).**
a) Thời gian đọc 12 byte bằng `Mem_Read` ở 400 kHz?
b) Mã hoá offset −3 mm vào thanh ghi 0x28.
c) Hai cảm biến chạy tuần tự, budget 20 ms, mỗi lượt tốn thêm 7 ms. Tần số đo mỗi cảm biến?

**Câu 3 (2 điểm).** Tìm và sửa các lỗi:
```c
void HAL_GPIO_EXTI_Callback(uint16_t pin)
{
    uint32_t u = TOF_Poll(&bus);
    if (u == 2) d = tof[2].range_mm;     /* muon lay so do cua cam bien thu hai */
}
```

### Đáp án và thang điểm Đề 1

**Phần I:** 1b · 2c · 3b · 4b · 5b · 6b (20×4 + 10) · 7b · 8b · 9b · 10b (5 × 142).

**Câu 1 (2 điểm):**
- (1,0 đ) Thuật toán:
  1. Mọi XSHUT thấp, chờ 2 ms.
  2. Với i = 0..2: XSHUT[i] cao, chờ 2 ms → đọc ID ở 0x52 → `SetDeviceAddress(addr[i])` → **gán `I2cDevAddr`** → đọc lại ID → khởi tạo.
  3. Con nào lỗi thì giữ XSHUT thấp.
- (1,0 đ) Bảng:

| Bước | XSHUT 1/2/3 | 0x52 | 0x54 | 0x56 | 0x58 |
|---|---|---|---|---|---|
| Đầu | T/T/T | — | — | — | — |
| Bật 1 | C/T/T | S1 | — | — | — |
| Đổi 1 | C/T/T | — | S1 | — | — |
| Bật 2 | C/C/T | S2 | S1 | — | — |
| Đổi 2 | C/C/T | — | S1 | S2 | — |
| Bật 3 | C/C/C | S3 | S1 | S2 | — |
| Đổi 3 | C/C/C | — | S1 | S2 | S3 |

(T = thấp, C = cao.) Mất 0,5 đ nếu thiếu bước gán `I2cDevAddr`.

**Câu 2 (2 điểm):**
- a) (0,7 đ) 3 + 12 = 15 byte × 9 = 135 bit × 2,5 µs = **337,5 µs**.
- b) (0,6 đ) −3000 / 250 = −12 → 4096 − 12 = **4084 = 0xFF4**.
- c) (0,7 đ) Một lượt = 20 + 7 = 27 ms; 2 con = 54 ms → **≈ 18,5 Hz** mỗi con.

**Câu 3 (2 điểm), mỗi lỗi 0,5 đ:**
1. Gọi `TOF_Poll` **trong ngắt**: I2C chặn CPU trong ngắt; tranh chấp với vòng lặp chính; có thể treo nếu dùng `HAL_Delay`.
2. `u == 2` kiểm tra sai: bitmask 0x02 là cảm biến **chỉ số 1**; phải dùng `u & (1UL << 1)`.
3. `tof[2]` sai chỉ số: cảm biến thứ hai là `tof[1]`; `tof[2]` còn có thể nằm ngoài mảng.
4. Không kiểm tra `range_status`.

Sửa:
```c
volatile uint8_t tof_irq;                                   /* (neu that su can EXTI) */
void HAL_GPIO_EXTI_Callback(uint16_t pin) { if (pin == TOF2_INT_Pin) tof_irq = 1; }
/* trong while(1): */
uint32_t u = TOF_Poll(&bus);
if ((u & (1UL << 1)) && tof[1].range_status == 0) d = tof[1].range_mm;
```

---

## 7. ĐỀ KIỂM TRA SỐ 2 (60 phút, 10 điểm)

### Phần I — Trắc nghiệm (3 điểm, mỗi câu 0,3)

1. Bật `sigma_check = 1` thì mỗi lần đọc tốn khoảng: a) 4 b) 12 c) 23 d) 50 giao dịch I2C
2. `HAL_Delay(2)` chờ: a) đúng 2 ms b) 1–2 ms c) 2–3 ms d) 0–2 ms
3. `now = 0x05`, `next = 0xFFFFFFFB`. `(int32_t)(now − next)` bằng: a) −10 b) 10 c) 0 d) 0xFFFFFFF6
4. Thanh ghi offset đọc 0xFF6, offset là: a) +2,5 mm b) −2,5 mm c) −10 mm d) +10 mm
5. Gọi `PerformSingleRangingMeasurement`, rồi `while(!ready) GetMeasurementDataReady(...)`: a) chạy bình thường b) treo c) lỗi −4 d) nhanh hơn
6. Profile 200 ms, `bus.budget_us` để 33 ms: a) bình thường b) báo TIME_OUT rồi ERROR c) đo nhanh hơn d) lỗi biên dịch
7. Watch đứng yên, LED vẫn nháy: a) chương trình treo b) chưa bật Periodic Window Update c) hết RAM d) I2C lỗi
8. Keil báo lỗi ở `LOG_FUNCTION_START`: a) thiếu include path b) chưa bật C99 c) thiếu MicroLIB d) sai địa chỉ
9. Trang flash cuối của F103C8: a) 0x08000000 b) 0x0800F000 c) 0x0800FC00 d) 0x20004C00
10. 4 cảm biến tốn khoảng bao nhiêu RAM cho `TOF_Sensor`? a) 470 B b) ~1,9 KB c) ~4 KB d) ~19 KB

### Phần II — Tự luận (7 điểm)

**Câu 1 (1,5 điểm).** Vì sao `TOF_Poll` chỉ đọc tối đa một cảm biến mỗi lần gọi và xoay vòng? So sánh thời gian chặn tối đa nếu đọc cả 4 cảm biến trong một lần.

**Câu 2 (1,5 điểm).** Watch cho thấy: `init_ok = 0`, `scan_before_n = 0`, `scan_after_n = 0`, cả hai `last_error = −20`. Nêu các nguyên nhân có thể và trình tự kiểm tra.

**Câu 3 (2 điểm).** Viết hàm `uint16_t tof_min_distance(uint8_t *idx)` trả khoảng cách nhỏ nhất trong các cảm biến có số đo hợp lệ và mới (≤ 100 ms), ghi chỉ số vào `*idx`; nếu không có thì trả 0xFFFF.

**Câu 4 (2 điểm).** Robot có 4 cảm biến: 2 cái phía trước đặt song song cách nhau 6 cm, 1 cái nhìn sang trái (90°), 1 cái nhìn sang phải. Đề xuất cách tổ chức chế độ đo để vừa không nhiễu, vừa giữ tốc độ cao nhất có thể. Ước lượng tần số đo của từng cảm biến.

### Đáp án và thang điểm Đề 2

**Phần I:** 1c · 2c · 3b (0x05 + 5 = 10) · 4b · 5b · 6b · 7b · 8b · 9c · 10b (4 × 470 ≈ 1,9 KB).

**Câu 1 (1,5 điểm):**
- (0,5 đ) Mỗi lần đọc tốn ~2,6 ms I2C. Đọc cả 4 trong một lần thì chặn **~10,4 ms**; một cảm biến thì chỉ **~2,6–3 ms**.
- (0,5 đ) Giới hạn thời gian chặn giúp các tác vụ khác (LCD, HX711, nút nhấn) không bị trễ lâu.
- (0,5 đ) Xoay vòng (bắt đầu từ cảm biến sau con vừa đọc) để công bằng: không con nào bị "bỏ đói" khi nhiều con cùng sẵn sàng. Tần số trung bình không giảm, vì vòng lặp gọi `TOF_Poll` hàng nghìn lần mỗi giây.

**Câu 2 (1,5 điểm):**
- (0,5 đ) Nhận định: cả hai đều không trả lời I2C, nên lỗi **chung** (bus/nguồn), không phải XSHUT của riêng một con.
- (1,0 đ) Nguyên nhân và trình tự kiểm tra:
  1. Nguồn VIN/GND của module (đo 3,3 V).
  2. SDA/SCL bị lắp ngược (PB7 = SDA, PB6 = SCL).
  3. I2C1 chưa bật trong CubeMX, hoặc `MX_I2C1_Init` gọi sau `App_..._Init`.
  4. Thiếu điện trở kéo lên (module lạ), hoặc bus đang treo: thử `TOF_I2C_Recover`, tắt nguồn cả mạch.
  5. Label XSHUT sai chân, nên cả hai luôn tắt: đo điện áp chân XSHUT khi đang được thả lên (phải ~2,8 V).

**Câu 3 (2 điểm):**
```c
uint16_t tof_min_distance(uint8_t *idx)
{
    uint16_t best = 0xFFFF, mm;
    uint8_t  i;

    for (i = 0; i < bus.count; i++) {
        if (TOF_GetRange(&bus, i, 100, &mm) && mm < best) {
            best = mm;
            if (idx) *idx = i;
        }
    }
    return best;
}
```
Thang điểm:
- (0,5 đ) Dùng `TOF_GetRange` với `max_age`.
- (0,5 đ) Tìm min đúng.
- (0,5 đ) Ghi chỉ số, kiểm tra `idx != NULL`.
- (0,5 đ) Trả 0xFFFF khi không có số đo nào.

**Câu 4 (2 điểm):**
- (0,5 đ) **Phân tích:** hai cảm biến trước cách 6 cm, nên vùng nhìn chồng từ d > 60 / (2 × 0,2217) ≈ **135 mm**: có nhiễu, cần đo tuần tự **giữa hai con này**. Hai cảm biến bên lệch 90° so với phía trước (> 25°), nên không chồng với ai.
- (0,5 đ) **Giải pháp tối ưu:** dùng **hai `TOF_Bus` trên cùng `hi2c1`**:
  - Nhóm A = 2 cảm biến trước, `TOF_MODE_SEQUENTIAL`.
  - Nhóm B = 2 cảm biến bên, `TOF_MODE_PARALLEL`.
  - Gọi `TOF_Init(&busA)` rồi `TOF_Init(&busB)`; mỗi vòng lặp gọi `TOF_Poll(&busA)` và `TOF_Poll(&busB)`.
- (0,5 đ) **Điều kiện an toàn:**
  - Địa chỉ phải khác nhau **giữa hai nhóm** (ví dụ 0x54, 0x56 | 0x58, 0x5A). `tof_addr_valid` chỉ kiểm tra trong cùng một nhóm.
  - Không ai giữ 0x52, nên restart trong nhóm này không ảnh hưởng nhóm kia.
  - Mỗi nhóm có XSHUT riêng.
- (0,5 đ) **Tốc độ ước lượng:** nhóm A ~11–13 Hz mỗi con (tuần tự 2 con, ~40 ms/lượt); nhóm B ~30 Hz mỗi con.
  Phương án "cả 4 tuần tự" chỉ được ~6 Hz mỗi con. Phương án "cả 4 song song" bị nhiễu ở phía trước.

🔬 Đã kiểm chứng trên giả lập (4 cảm biến, không GPIO1):
- Địa chỉ gán đúng 0x54/0x56/0x58/0x5A.
- Nhóm A 11–12 Hz mỗi con, **không bao giờ có 2 con nhóm A đo cùng lúc**; nhóm B 33 Hz mỗi con.
- Restart một con nhóm A trong khi nhóm B đang chạy: 0 xung đột.

---

**Hết quyển 6.** Tiếp theo: [Quyển 7 — Chuyên đề nâng cao](07_ChuyenDe_NangCao.md).

# Quyển 7 — CHUYÊN ĐỀ NÂNG CAO

> Dành cho người muốn hiểu tận gốc và tự mở rộng thư viện. Mỗi chuyên đề có: **vấn đề → phân tích / chứng minh → kết luận thiết kế → bài toán (có lời giải).**
> Cần nắm vững quyển 1 và quyển 2 trước khi đọc.

## Mục lục
1. [Giải phẫu chi phí I2C của một lần đọc](#chuyên-đề-1--giải-phẫu-chi-phí-i2c-của-một-lần-đọc)
2. [Tính đúng đắn của thuật toán gán địa chỉ](#chuyên-đề-2--tính-đúng-đắn-của-thuật-toán-gán-địa-chỉ)
3. [Lý thuyết lịch hỏi và giới hạn số cảm biến](#chuyên-đề-3--lý-thuyết-lịch-hỏi-và-giới-hạn-số-cảm-biến)
4. [Số học máy: tràn số, mã lỗi, thời gian quay vòng](#chuyên-đề-4--số-học-máy-tràn-số-mã-lỗi-thời-gian-quay-vòng)
5. [Nhiễu quang và lập lịch theo đồ thị xung đột](#chuyên-đề-5--nhiễu-quang-và-lập-lịch-theo-đồ-thị-xung-đột)
6. [Kiểm thử bằng bộ giả lập](#chuyên-đề-6--kiểm-thử-bằng-bộ-giả-lập)
7. [Mở rộng thư viện](#chuyên-đề-7--mở-rộng-thư-viện)
8. [Bài toán thử thách](#chuyên-đề-8--bài-toán-thử-thách)

---

## Chuyên đề 1 — Giải phẫu chi phí I2C của một lần đọc

### Vấn đề
Trong quá trình phát triển, 4 cảm biến song song chỉ đạt **20 Hz mỗi con** thay vì ~30 Hz, và một lần `TOF_Poll` giữ CPU tới ~11 ms. Câu hỏi: thời gian bị tiêu ở đâu?

### Phương pháp
Ghi lại **từng giao dịch I2C** trong một lần `VL53L0X_GetRangingMeasurementData` + `ClearInterruptMask` (khi kiểm tra sigma đang bật, mặc định của ST):

| Thanh ghi | Tên | Số lần | Kiểu |
|---|---|---|---|
| 0x14 (12 byte) | RESULT_RANGE_STATUS… | 1 | **Kết quả: cần thiết** |
| 0x50 | PRE_RANGE_CONFIG_VCSEL_PERIOD | 4 | Đọc lại cấu hình |
| 0x01 | SYSTEM_SEQUENCE_CONFIG | 3 | Đọc lại cấu hình |
| 0x64 | PRE_RANGE_MIN_COUNT_RATE_RTN_LIMIT | 2 | Đọc lại cấu hình |
| 0x51 | PRE_RANGE_CONFIG_TIMEOUT_MACROP_HI | 2 | Đọc lại cấu hình |
| 0x70, 0x71 | FINAL_RANGE VCSEL / TIMEOUT | 2 | Đọc lại cấu hình |
| 0x46 | MSRC_CONFIG_TIMEOUT_MACROP | 1 | Đọc lại cấu hình |
| 0x44 | FINAL_RANGE…RTN_LIMIT | 1 | Đọc lại cấu hình |
| 0x28 | Offset | 1 | Đọc lại cấu hình |
| 0x20 | Crosstalk compensation | 1 | Đọc lại cấu hình |
| 0x04, 0xF8 | Inter-measurement period, OSC_CALIBRATE | 2 | Đọc lại cấu hình |
| 0x0B (×2), 0x13 | Xoá cờ ngắt + kiểm tra | 3 | **Cần thiết** |
| **Tổng** | | **23** | Chỉ 4 giao dịch thật sự cần |

### Phân tích
Để ước lượng sigma, ST cần **timing budget hiện tại**, **chu kỳ VCSEL** và **các giới hạn**. Thay vì lưu sẵn trong `Dev->Data`, các hàm `Get…` của ST **đọc lại từ chip mỗi lần**. Những giá trị này **chỉ đổi khi bạn đổi cấu hình**, nên 19/23 giao dịch là lặp lại không cần thiết.

Chi phí ở 100 kHz:
- Bật sigma: ~23 giao dịch ≈ 10–12 ms mỗi lần đọc. 4 cảm biến × 30 Hz ≈ 120 lần đọc mỗi giây ≈ **1,3 s bus mỗi giây**: không thể đạt.
- Tắt sigma (`SetLimitCheckEnable(SIGMA, 0)`): 4 giao dịch ≈ 2,6 ms.

### Kết luận thiết kế
`tof_multi` mặc định `sigma_check = 0`. Đây là một **đánh đổi có ý thức**:
- Mất trạng thái "Sigma fail" (RangeStatus = 1) và `RangeDMaxMilliMeter`.
- Giữ đầy đủ các trạng thái signal/phase/min range.
- Nhanh hơn ~4 lần.

### Bài toán 1.1
Đề xuất cách **giữ kiểm tra sigma mà vẫn chỉ tốn ~4 giao dịch**. Phân tích cái giá phải trả.

<details><summary>Lời giải</summary>

**Ý tưởng:** lưu các giá trị cấu hình (VCSEL, timeout, giới hạn, offset, xtalk…) vào `Dev->Data` một lần sau khi khởi tạo hoặc mỗi khi đổi cấu hình. Sửa các hàm `Get…` được dùng trong ước lượng sigma để đọc bản lưu thay vì đọc chip.

**Cái giá:**
1. **Phải sửa logic của ST**, trái nguyên tắc "chép nguyên lõi ST". Mỗi lần cập nhật phiên bản ST phải vá lại.
2. **Rủi ro lệch dữ liệu:** nếu ai đó ghi cấu hình trực tiếp bằng `VL53L0X_WrByte` mà không cập nhật bản lưu, sigma sẽ tính sai mà không báo gì.
3. Tốn thêm RAM (vài chục byte mỗi cảm biến).

**Phương án thay thế** không sửa ST: tự tính chỉ tiêu chất lượng từ dữ liệu đã có trong 12 byte kết quả, ví dụ loại số đo có `SignalRateRtnMegaCps` quá thấp so với `AmbientRateRtnMegaCps`.
</details>

---

## Chuyên đề 2 — Tính đúng đắn của thuật toán gán địa chỉ

### Mô hình
- Tập cảm biến S = {s₀, …, s₍ₙ₋₁₎}. Mỗi cảm biến có trạng thái **tắt** (XSHUT thấp) hoặc **bật**; khi bật có địa chỉ `a(s)`. Bật lên từ trạng thái tắt thì `a(s) = 0x52`.
- **Bất biến I:** *tại mọi thời điểm, số cảm biến đang bật có `a(s) = 0x52` là ≤ 1.*
- **Bất biến J:** *mọi cảm biến đang bật và đã khởi tạo xong có địa chỉ khác 0x52, và đôi một khác nhau.*

### Định lý
Nếu ba quy tắc ở [Q1 §4.4] được tuân thủ và mọi địa chỉ cấu hình thoả `tof_addr_valid`, thì I và J đúng tại mọi thời điểm. Do đó **mọi lệnh I2C tới một địa chỉ đều được đúng một cảm biến (hoặc không cảm biến nào) nhận**.

### Chứng minh (quy nạp theo các bước của thuật toán)
- **Cơ sở:** `TOF_Init` bước 1 kéo mọi XSHUT thấp. Không ai bật, nên I và J đúng.
- **Bước bật sᵢ (XSHUT cao):** theo giả thiết quy nạp, mọi cảm biến đang bật khác đều có địa chỉ ≠ 0x52 (J). Vậy sau bước này, chỉ sᵢ ở 0x52: I đúng.
- **Bước đổi địa chỉ thành công:** `a(sᵢ)` = `addr[i]` ≠ 0x52 (vì hợp lệ), và khác mọi địa chỉ đang dùng (vì không trùng trong cùng bus). J đúng; không còn ai ở 0x52, nên I đúng.
- **Bước thất bại:** quy tắc 3 kéo XSHUT của sᵢ xuống thấp, sᵢ tắt, nên I và J được khôi phục.
- **`TOF_Restart(i)`:** XSHUT của sᵢ thấp → sᵢ tắt → quay lại "bước bật sᵢ". Các cảm biến khác vẫn thoả J, nên lập luận trên vẫn áp dụng.
- **Mất điện / reset MCU:** khi mất điện, mọi cảm biến về 0x52. Khi chỉ reset MCU, cảm biến có thể vẫn giữ địa chỉ cũ, nhưng `TOF_Init` kéo mọi XSHUT xuống thấp nên chúng cũng bị reset. Quy tắc 1 (gán lại mỗi lần khởi động) đưa hệ thống về bước cơ sở. ∎

### Vi phạm từng quy tắc dẫn tới gì (phản ví dụ)

| Bỏ quy tắc | Kịch bản | Hậu quả |
|---|---|---|
| 1 (gán lại mỗi lần khởi động) | MCU reset, cảm biến vẫn có điện nên giữ 0x54; thư viện không gán lại. Hoặc ngược lại: cảm biến mất điện, về 0x52 | Thư viện nói chuyện với địa chỉ không còn ai, hoặc hai con cùng 0x52 |
| 2 (không ai giữ 0x52) | s₀ được để ở 0x52. Restart s₁ thì s₁ bật ở 0x52 | **Hai con ở 0x52**, vi phạm I |
| 3 (con lỗi giữ XSHUT thấp) | s₁ bật nhưng đọc ID lỗi do nhiễu; vẫn để bật ở 0x52; sau đó bật s₂ | s₁ và s₂ cùng 0x52 → cùng nhận `SetDeviceAddress` → cùng sang `addr[2]`. Đúng hiện tượng [Q4 D1] |

### Mở rộng: nhiều `TOF_Bus` trên cùng một I2C
`tof_addr_valid` chỉ kiểm tra trùng **trong một bus**. Với nhiều nhóm trên cùng một I2C (Q6 Đề 2 câu 4), J phải đúng **trên toàn bus vật lý**, nên **người dùng phải tự bảo đảm** địa chỉ khác nhau giữa các nhóm. Bất biến I vẫn đúng: mỗi nhóm chỉ bật cảm biến của mình khi mọi cảm biến khác (cả nhóm khác) đã ≠ 0x52.
🔬 Đã kiểm chứng: 2 nhóm (tuần tự + song song) trên một I2C, restart chéo nhóm, 0 xung đột.

### Bài toán 2.1
Vì sao `tof_addr_valid(b, i)` chỉ so sánh với các cảm biến `j < i` mà vẫn phát hiện được mọi địa chỉ trùng?

<details><summary>Lời giải</summary>

Giả sử `addr[j] = addr[k]` với j < k. Khi kiểm tra k, vòng lặp duyệt mọi chỉ số nhỏ hơn k, trong đó có j, nên k bị từ chối. Vậy với mọi cặp trùng, **phần tử đứng sau** luôn bị từ chối; phần tử đầu tiên được giữ. Không cần so sánh với `j > i`, vì cặp đó sẽ được phát hiện khi tới lượt phần tử đứng sau. Độ phức tạp O(n²), không đáng kể với n ≤ 32.
</details>

---

## Chuyên đề 3 — Lý thuyết lịch hỏi và giới hạn số cảm biến

### 3.1. Mô hình
- Chế độ continuous back-to-back: chip **tự đo liên tục**, mỗi số đo xong sau một chu kỳ ≈ B (budget). Số đo mới **ghi đè** số đo cũ nếu chưa được đọc.
- Sau khi đọc (thời điểm t₀), lần hỏi đầu tiên ở t₀ + αB (α = 3/4), sau đó mỗi p ms.

### 3.2. Vì sao α = 3/4?
- **α quá nhỏ** (ví dụ 0,1): nhiều lần hỏi vô ích, mỗi lần ~0,4 ms bus.
- **α ≥ 1**: nguy cơ hỏi **sau** khi số đo kế tiếp đã ghi đè: mất mẫu, và tăng độ trễ.
- Thời gian đo thật dao động theo bộ dao động nội của chip và theo thời điểm đọc. α = 3/4 chừa **25% biên an toàn**: lần hỏi đầu gần như chắc chắn rơi vào trước khi số đo hoàn tất, và cách thời điểm hoàn tất không quá xa.

Số lần hỏi "chưa sẵn sàng" mỗi số đo: $k \approx \lceil (1-\alpha)B / p \rceil$. Với B = 33, p = 10: k ≈ ⌈8,25/10⌉ = 1. Chi phí phụ ~0,4 ms mỗi số đo. Độ trễ tối đa từ lúc có số đo đến lúc đọc ≈ p = 10 ms.

### 3.3. Định lý: số cảm biến tối đa mà không mất mẫu
**Giả thiết:**
- Chế độ song song, `TOF_MAX_READS_PER_POLL = 1`.
- Mỗi lần đọc tốn T_r.
- Giữa hai lần gọi `TOF_Poll`, vòng lặp chính tốn tối đa L.

**Mệnh đề:** nếu $N \cdot (T_r + L) < B$ thì không cảm biến nào bị mất mẫu, kể cả khi cả N cảm biến cùng sẵn sàng một lúc.

**Chứng minh:** trường hợp xấu nhất là N cảm biến cùng sẵn sàng tại t. Mỗi lần Poll đọc một con, xoay vòng. Con cuối được đọc chậm nhất tại $t + N(T_r + L)$. Số đo kế tiếp của nó chỉ hoàn tất ở $t + B$. Nếu $N(T_r + L) < B$ thì nó được đọc trước khi bị ghi đè. Xoay vòng bảo đảm không con nào phải chờ quá N lượt. ∎

**Áp dụng:** B = 33 ms, T_r = 2,6 ms, L = 1 ms → N < 9,2, nên **tối đa 9 cảm biến**. Nếu vòng lặp có vẽ LCD (L = 17 ms) thì N < 1,7: chỉ còn 1 cảm biến được bảo đảm **trong lượt có vẽ LCD**. Đây là lý do phải giữ L nhỏ (dùng LCD đã sửa, vẽ thưa).

> 💡 Mất một mẫu thỉnh thoảng thường chấp nhận được (tốc độ giảm nhẹ). Định lý trên là **điều kiện đủ** cho trường hợp xấu nhất.

### 3.4. Giới hạn tải bus (nhắc lại)
$U = N \cdot f \cdot T_r + U_\text{khác}$. Để hệ thống còn "thở", giữ U ≤ 50%.

### Bài toán 3.1
Với p = 5 ms và p = 20 ms, tính k và độ trễ tối đa (B = 33). Chọn p nào cho robot tốc độ cao? Cho trạm đo chiều cao?

<details><summary>Lời giải</summary>

(1 − α)B = 8,25 ms.
- p = 5: k = ⌈8,25/5⌉ = 2 lần hỏi vô ích (~0,8 ms), độ trễ ≤ 5 ms.
- p = 20: k = 1 (~0,4 ms), nhưng độ trễ có thể tới 20 ms.

**Robot tốc độ cao:** cần số đo "tươi", chọn p = 5, hoặc tốt nhất là **dùng GPIO1** (độ trễ ≈ thời gian một vòng lặp, không tốn bus).
**Trạm đo chiều cao:** không cần nhanh, chọn p = 20 để tiết kiệm bus.
</details>

---

## Chuyên đề 4 — Số học máy: tràn số, mã lỗi, thời gian quay vòng

### 4.1. Lỗi tràn số trong hiệu chuẩn của ST, và chứng minh bản vá
**Code gốc:**
```c
uint16_t sum_ranging;                                   /* cong 50 so do */
StoredMeanRange = (FixPoint1616_t)((uint32_t)(sum_ranging << 16) / total_count);
```
- **Tràn khi cộng:** tổng lớn nhất 50 × 8191 = 409 550 > 65 535. Trung bình trên 1310 mm thì `uint16_t` quay vòng.
- **Tràn khi dịch:** `sum_ranging` được nâng lên `int` trước khi `<< 16`. Khi sum ≥ 32 768 (trung bình > 655 mm), `sum << 16` ≥ 2³¹: **tràn số có dấu, hành vi không xác định** (UB). Ép kiểu `(uint32_t)(...)` đặt **sau** phép dịch nên không cứu được.

**Bản vá:** `uint32_t sum_ranging` và
```c
#define VL53L0X_MEAN_1616(sum, n) \
    ((FixPoint1616_t)((((uint32_t)(sum) / (n)) << 16) + ((((uint32_t)(sum) % (n)) << 16) / (n))))
```
**Chứng minh đúng và không tràn:** đặt sum = q·n + r với 0 ≤ r < n.
$$\left\lfloor \frac{\text{sum}\cdot 2^{16}}{n} \right\rfloor = \left\lfloor q\cdot 2^{16} + \frac{r\cdot 2^{16}}{n} \right\rfloor = q\cdot 2^{16} + \left\lfloor \frac{r\cdot 2^{16}}{n} \right\rfloor$$
(tách được vì q·2¹⁶ là số nguyên). Đó chính là công thức của macro, nên kết quả **chính xác**.
- Không tràn: q ≤ 8191 nên q·2¹⁶ ≤ 536 805 376 < 2³². r < n ≤ 50 nên r·2¹⁶ < 3 276 800. Tổng < 2³².

### 4.2. Gộp mã lỗi bằng `|=`
ST có nhiều chỗ viết `Status |= VL53L0X_WrByte(...)`. Mã lỗi là `int8_t` âm, dạng bù hai:

| Mã | Nhị phân (8 bit) |
|---|---|
| −7 (TIME_OUT) | 1111 1001 |
| −20 (CONTROL_INTERFACE) | 1110 1100 |
| OR | 1111 1101 = **−3 (UNDEFINED)** |

Kết quả là một mã **không liên quan** tới cả hai lỗi thật. **Bài học:** dùng mẫu `if (st == OK) st = f();` để giữ lỗi đầu tiên. `tof_multi` và các ví dụ đều viết theo mẫu này.

### 4.3. So sánh thời gian khi tick quay vòng
**Mệnh đề:** với a, b là `uint32_t` và khoảng cách thật Δ = a − b (số nguyên) thoả −2³¹ < Δ < 2³¹, ta có `(int32_t)(a - b)` = Δ.
**Chứng minh:** phép trừ không dấu cho (a − b) mod 2³². Ép sang `int32_t` (bù hai) ánh xạ [2³¹, 2³²) về [−2³¹, 0) và giữ nguyên [0, 2³¹). Nếu Δ ∈ (−2³¹, 2³¹) thì giá trị sau ép đúng bằng Δ. ∎
**Hệ quả:**
- `tof_due()` (`(int32_t)(now - next_ms) >= 0`) đúng khi hai mốc cách nhau < 24,8 ngày.
- `now - t >= 1000U` (không dấu) đúng khi khoảng chờ < 49,7 ngày.
- Còn `now > t + 1000` sai ngay khi `t + 1000` tràn (Q4 C9).

### 4.4. Ca kinh điển: lỗi timeout 16 bit của một bản port VL53L0X phổ biến
```c
#define checkTimeoutExpired() (g_ioTimeout > 0 && ((uint16_t)HAL_GetTick() - g_timeoutStartMs) > g_ioTimeout)
```
`(uint16_t)HAL_GetTick()` và `g_timeoutStartMs` (`uint16_t`) đều được **nâng lên `int`** trước khi trừ, nên hiệu có thể **âm** khi tick 16 bit quay vòng (mỗi 65,5 s). Khi đó so sánh `> timeout` luôn sai: **timeout 100 ms có thể kéo dài tới ~65 s**. Sửa: ép kiểu **cả hiệu** `(uint16_t)(now16 - start16)`. Bài học trùng với 4.3: muốn kết quả "modulo", phép trừ phải thực hiện ở đúng độ rộng không dấu.

### Bài toán 4.1
Chứng minh `(uint16_t)((uint16_t)now - start) > timeout` đúng khi khoảng chờ thật < 65 536 ms.

<details><summary>Lời giải</summary>

Hiệu thật Δ ∈ [0, 65536). `(uint16_t)now - start` sau khi nâng lên `int` bằng Δ hoặc Δ − 65536 (khi tick 16 bit quay vòng). Ép về `uint16_t` lấy theo modulo 65536, cho đúng Δ trong cả hai trường hợp. Vậy phép so sánh với `timeout` (cũng ≤ 65535) là đúng.
</details>

---

## Chuyên đề 5 — Nhiễu quang và lập lịch theo đồ thị xung đột

### 5.1. Khi nào hai cảm biến xung đột?
- **Đặt song song, cùng hướng, cách nhau s:** vùng nhìn chồng khi d > s / (2 tan 12,5°) ≈ 2,26·s.
- **Lệch góc Δθ ≥ 25°** (và không nhìn chéo vào nhau): hai hình nón tách rời nhau. Ở xa không chồng; gần cảm biến cũng chỉ chồng nếu đặt rất sát.
- **Phản xạ gián tiếp** (tường, sàn bóng) vẫn có thể gây nhiễu. Cách duy nhất để biết là thử thực tế.

### 5.2. Đồ thị xung đột
Coi mỗi cảm biến là một **đỉnh**; nối **cạnh** giữa hai cảm biến có thể nhiễu nhau. Lập lịch không nhiễu tương đương **tô màu đồ thị**: hai đỉnh kề nhau không được đo cùng lúc. Mỗi màu là một "khung thời gian"; các cảm biến cùng màu đo song song.

Với χ là **số màu tối thiểu**, tốc độ mỗi cảm biến ≈ 1 / (χ × chu kỳ đo).

### 5.3. `tof_multi` hỗ trợ đến đâu?
- Một `TOF_Bus` tuần tự = một **clique** (mọi cặp đều xung đột) đo lần lượt.
- Một `TOF_Bus` song song = các đỉnh **không xung đột** với ai.
- **Nhiều `TOF_Bus` trên cùng I2C chạy đồng thời** (chuyên đề 2), nên `tof_multi` xử lý tối ưu khi **đồ thị là hợp rời của các clique**.

| Bố trí | Đồ thị | Tổ chức | Tốc độ mỗi con |
|---|---|---|---|
| 2 trước sát nhau + trái + phải | {A,B} là một cạnh; C, D cô lập | Bus 1 tuần tự {A,B}; Bus 2 song song {C,D} | A,B ~12 Hz; C,D ~30 Hz |
| 2 trước + 2 sau (mỗi cặp sát nhau) | 2 cạnh rời nhau | 2 bus tuần tự | Cả 4 ~12 Hz |
| Vòng 6 cảm biến, mỗi con chỉ chồng với 2 con kề | Chu trình C₆ (không phải hợp các clique) | Cần lịch tự viết (bài 5.1) | Tối ưu: χ = 2 → ~15 Hz |

### Bài toán 5.1
Sáu cảm biến xếp vòng tròn, mỗi con chỉ xung đột với hai con kề (đồ thị C₆). Đề xuất lịch đo tốt nhất và giải thích vì sao `tof_multi` hiện tại chưa làm được tối ưu.

<details><summary>Lời giải</summary>

- C₆ là chu trình chẵn nên **2 màu là đủ**: nhóm chẵn {0, 2, 4} và nhóm lẻ {1, 3, 5}. Các con cùng nhóm không kề nhau, nên đo song song được.
- **Lịch:** khung 1 nhóm chẵn đo; khung 2 nhóm lẻ đo; lặp lại. Mỗi con ~1/(2 × ~40 ms) ≈ **12–15 Hz**.
- `tof_multi` chỉ có hai kiểu: "tuần tự trong một bus" (mỗi lúc một con) và "song song trong một bus". Đặt cả 6 vào một bus tuần tự thì mỗi con chỉ ~4 Hz. Chia {0,2,4} và {1,3,5} thành 2 bus song song thì hai bus **chạy cùng lúc**, nên con 0 và con 1 vẫn nhiễu nhau.
- **Mở rộng cần có:** chế độ "tuần tự theo nhóm". Mỗi lượt bật **cả một nhóm** ở chế độ SINGLE, chờ cả nhóm xong, rồi chuyển nhóm. Có thể viết dựa trên `tof_seq_next`, thay "một cảm biến" bằng "một tập cảm biến".
</details>

---

## Chuyên đề 6 — Kiểm thử bằng bộ giả lập

### 6.1. Vì sao cần giả lập
Lỗi của bài toán nhiều cảm biến (trùng địa chỉ, mất mẫu, treo khi rút dây) **khó tái hiện** trên mạch thật và khó quan sát. Bộ giả lập cho phép chạy **toàn bộ mã thật** (lõi ST + platform + `tof_multi`) trên PC, lặp lại chính xác và đo mọi thứ.

### 6.2. Bộ giả lập mô phỏng những gì

| Thành phần | Mô phỏng |
|---|---|
| Bus I2C | Mỗi giao dịch tìm thiết bị đang bật có đúng địa chỉ. Không có thiết bị: NACK. **Hai thiết bị cùng địa chỉ: ghi nhận xung đột** |
| Thời gian | Đồng hồ µs ảo; mỗi byte 90 µs; NACK dừng sau byte địa chỉ; `HAL_Delay(n)` tăng n+1 ms |
| XSHUT | Thấp: thiết bị reset (xoá thanh ghi, địa chỉ về 0x52). Cao: khởi động |
| Thanh ghi | 7 trang (chọn bằng 0xFF); 0xC0 = 0xEE; 0x8A đổi địa chỉ; 0x00 bắt đầu/dừng đo; 0x0B xoá cờ; 0x13 trạng thái |
| NVM | Đọc qua 0x94/0x83/0x90 trả số SPAD và bản đồ SPAD hợp lý |
| Đo | Single: xong sau 30 ms (1 ms nếu là bước hiệu chuẩn VHV/phase); continuous: cứ 30 ms một số đo; ghi khoảng cách vào 0x1E–0x1F, status 11 |
| GPIO1 | Mức thấp khi có số đo chưa xoá; rút module thì về mức cao |
| Lỗi | Rút module (`present = 0`), cảm biến ngừng đo, độ lệch (`bias`) để thử hiệu chuẩn |

### 6.3. Những gì giả lập **không** kiểm chứng được
- Quang học thật: tín hiệu yếu, ánh nắng, vật tối màu, sigma thật, nhiễu giữa cảm biến.
- Thời gian thật của chip (giả lập đo 30 ms thay vì 33 ms), tốc độ HAL thật, cách Keil tối ưu code.
- Ảnh hưởng thật của các thanh ghi tuning (giả lập chỉ lưu giá trị).

Vì vậy mọi số 🔬 trong sách chỉ là **mức tham chiếu**; phải xác nhận lại trên mạch thật.

### 6.4. Những phát hiện có được nhờ giả lập
1. Chi phí sigma: 23 so với 4 giao dịch (chuyên đề 1), dẫn tới tuỳ chọn `sigma_check`.
2. Một lần `TOF_Poll` từng giữ CPU ~11 ms, dẫn tới `TOF_MAX_READS_PER_POLL` và xoay vòng.
3. Sau thay đổi đó, tốc độ tụt còn 25 Hz vì cả bus chờ chung một chu kỳ hỏi, dẫn tới lịch hỏi **riêng cho từng cảm biến**.
4. Một bẫy của chính chương trình kiểm thử: `printf("%d %d", f(), x)` có thể tính `x` **trước** `f()`, vì C không quy định thứ tự tính tham số. Kết quả kiểm thử sai dù thư viện đúng. Đã sửa bằng cách tách lời gọi ra trước `printf`.

### Bài toán 6.1
Thiết kế một kịch bản giả lập kiểm chứng **bất biến I** (chuyên đề 2) trong suốt quá trình restart liên tục.

<details><summary>Lời giải</summary>

- 4 cảm biến song song.
- Trong 60 s mô phỏng, mỗi 300 ms chọn ngẫu nhiên một cảm biến và gọi `TOF_Restart`; thỉnh thoảng đặt `present = 0` rồi `= 1` để giả rút/cắm.
- Sau **mỗi** giao dịch I2C, đếm số thiết bị đang bật ở 0x52 (phải ≤ 1). Kiểm tra bộ đếm xung đột luôn bằng 0.
- Kết thúc: mọi cảm biến có mặt phải RUNNING, địa chỉ thật trên chip khớp cấu hình.
- Lặp lại với nhiều hạt giống ngẫu nhiên (seed) để phủ nhiều thứ tự sự kiện.
</details>

---

## Chuyên đề 7 — Mở rộng thư viện

### 7.1. Hook sau khởi tạo: giữ profile qua `TOF_Restart`
**Vấn đề:** `TOF_Restart` xoá cấu hình riêng (VD10, Q4 E7). Bọc ngoài bằng `tof_restart_keep_profile` thì được, nhưng phải dừng rồi chạy lại cảm biến thêm một lần, và dễ quên.

**Giải pháp:** thêm con trỏ hàm `on_setup` vào `TOF_Bus`. Thư viện gọi hàm này **cuối mỗi lần khởi tạo một cảm biến**, lúc cảm biến còn IDLE, trước `StartMeasurement`.

**Thay đổi đề xuất** (chưa áp dụng vào thư viện; đã thử trên một bản sao):
```diff
--- tof/tof_multi.h
     uint32_t           poll_ms;
+    VL53L0X_Error    (*on_setup)(TOF_Sensor *s); /* MO RONG: goi cuoi moi lan khoi tao 1 cam bien (NULL = khong) */
     uint8_t            sigma_check;
--- tof/tof_multi.c   (cuoi ham tof_setup)
     if (st == TOF_OK && !b->sigma_check)
         st = VL53L0X_SetLimitCheckEnable(dev, VL53L0X_CHECKENABLE_SIGMA_FINAL_RANGE, 0);
+    if (st == TOF_OK && b->on_setup != NULL)     /* MO RONG: cau hinh rieng, ap lai sau moi lan restart */
+        st = b->on_setup(s);
     return st;
```
**Hàm cấu hình** (không Stop/Start, vì thư viện gọi lúc cảm biến đang IDLE):
```c
static VL53L0X_Error cfg_long_range(TOF_Sensor *s)
{
    VL53L0X_DEV d = &s->dev;
    VL53L0X_Error st;

    st = VL53L0X_SetLimitCheckValue(d, VL53L0X_CHECKENABLE_SIGNAL_RATE_FINAL_RANGE,
                                    (FixPoint1616_t)(0.10 * 65536));
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_SetVcselPulsePeriod(d, VL53L0X_VCSEL_PERIOD_PRE_RANGE, 18);
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_SetVcselPulsePeriod(d, VL53L0X_VCSEL_PERIOD_FINAL_RANGE, 14);
    if (st == VL53L0X_ERROR_NONE)
        st = VL53L0X_SetMeasurementTimingBudgetMicroSeconds(d, 33000);
    return st;
}
/* TOF_Bus bus = { ..., .on_setup = cfg_long_range }; */
```
🔬 Kết quả trên bản sao: sau `TOF_Init`, VCSEL = 18/14; **sau `TOF_Restart` vẫn 18/14**; cảm biến đo tiếp 33 lần/s.

⚠️ Hàm `on_setup` dùng chung cho cả bus. Muốn cấu hình khác nhau cho từng cảm biến, phân biệt bằng `s->addr` hoặc so sánh con trỏ `s` với `&tof[i]`.

### 7.2. Hai bus I2C vật lý
**Đã hỗ trợ sẵn:** tạo `TOF_Bus` thứ hai với `.hi2c = &hi2c2`. Địa chỉ trên hai bus vật lý **độc lập** nhau (có thể trùng). Ví dụ:
- I2C1 100 kHz: LCD.
- I2C2 (PB10/PB11, chịu được 5 V) **400 kHz**: các VL53L0X. Thời gian đọc giảm còn ~0,7 ms, tải bus giảm ~4 lần.

### 7.3. Dùng với RTOS (FreeRTOS)
- Mỗi bus I2C cần một **mutex**: mọi lời gọi `TOF_*` và mọi thư viện khác dùng cùng bus (LCD…) phải nằm trong mutex. `VL53L0X_LockSequenceAccess` của ST **không được gọi**, nên đừng dựa vào nó.
- `VL53L0X_PollingDelay` đổi sang `osDelay(1)` để nhường CPU.
- `HAL_Delay` trong `tof_multi` (`TOF_BOOT_MS`, chờ khi hiệu chuẩn) nên đổi sang `osDelay`, ví dụ bằng một macro `TOF_DELAY_MS(x)`.
- Chạy `TOF_Poll` trong một task riêng, chu kỳ 1–5 ms, hoặc đánh thức bằng EXTI của GPIO1 qua semaphore.

### 7.4. I2C dùng ngắt/DMA (không chặn hẳn)
API của ST là **đồng bộ**: mỗi `VL53L0X_RdByte` phải xong mới chạy tiếp. Muốn I2C không chặn CPU hoàn toàn, phải **bỏ qua ST ở đường đọc**: tự viết máy trạng thái đọc 12 byte kết quả bằng `HAL_I2C_Mem_Read_DMA`, giải mã status `(reg & 0x78) >> 3 == 11`, rồi xoá cờ bằng `HAL_I2C_Mem_Write_IT`. ST vẫn dùng cho khởi tạo và hiệu chuẩn.
**Đánh đổi:** mất phần giải mã trạng thái của ST; phức tạp hơn nhiều; phải quản lý tranh chấp bus bằng hàng đợi. Chỉ đáng làm khi CPU thật sự không chịu được ~2,6 ms chặn mỗi lần đọc.

### 7.5. Ngắt theo ngưỡng khoảng cách
API của ST hỗ trợ GPIO1 báo khi khoảng cách vượt ngưỡng:
```c
VL53L0X_SetGpioConfig(d, 0, VL53L0X_DEVICEMODE_CONTINUOUS_RANGING,
                      VL53L0X_GPIOFUNCTIONALITY_THRESHOLD_CROSSED_LOW,
                      VL53L0X_INTERRUPTPOLARITY_LOW);
VL53L0X_SetInterruptThresholds(d, VL53L0X_DEVICEMODE_CONTINUOUS_RANGING,
                               (FixPoint1616_t)300 << 16, (FixPoint1616_t)400 << 16);
```
⚠️ **Xung đột với `tof_multi`:**
1. Thư viện giả định GPIO1 = "có mẫu mới". Ở chế độ ngưỡng, GPIO1 chỉ xuống thấp khi vượt ngưỡng, nên `tof_service` hiểu nhầm là "không có dữ liệu" và báo TIME_OUT. Phải đặt `gpio1_port = NULL` cho cảm biến đó.
2. Với ngưỡng > 255 mm, ST tự nạp bản vá `InterruptThresholdSettings` khi `StartMeasurement` (vì vậy phải cấu hình **trước** `StartMeasurement`). Có thể đặt vào hook `on_setup` ở 7.1.

---

## Chuyên đề 8 — Bài toán thử thách

### Thử thách 1 — Ước lượng thời gian khởi tạo
8 cảm biến, mỗi con khởi tạo 0,22 s; khi có `calib.valid = 1`, mỗi con 0,17 s. Tính tổng thời gian. Có thể khởi tạo song song để nhanh hơn không?

<details><summary>Lời giải</summary>

- Chưa hiệu chuẩn: 8 × 0,22 ≈ **1,8 s**. Có calib: 8 × 0,17 ≈ **1,4 s**.
- **Không song song được** với API đồng bộ của ST: mỗi hàm khởi tạo chờ chip xong (các lần đo nội bộ) trên **cùng một bus**, CPU không làm việc khác.
- **Có thể** gán địa chỉ cho cả 8 con trước (nhanh, ~vài ms mỗi con). Nhưng phần chậm (đo SPAD, VHV/phase) vẫn phải lần lượt, trừ khi tự viết máy trạng thái không chặn (chuyên đề 7.4).
- Cách thực tế để rút ngắn: lưu `calib`, giảm số lần đo nội bộ; hoặc khởi tạo trong lúc hệ thống còn đang làm việc khác (RTOS).
</details>

### Thử thách 2 — Lỗi "một tháng"
Một robot chạy ổn định suốt 49 ngày, rồi đột nhiên LED trạng thái nháy loạn trong khoảng 1 giây, sau đó lại bình thường. Code đèn LED:
```c
if (HAL_GetTick() > t_next) { t_next = HAL_GetTick() + 500; HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin); }
```
Giải thích và sửa.

<details><summary>Lời giải</summary>

49,7 ngày ≈ 2³² ms: `HAL_GetTick` quay vòng.
- Khi tick gần 2³², `t_next = tick + 500` **tràn** thành số nhỏ. Từ đó `tick > t_next` **luôn đúng**, nên LED đảo **mỗi vòng lặp** (nháy loạn).
- Khi chính tick quay về 0, `tick > t_next` lại sai bình thường. Thời gian "loạn" ≤ 500 ms tính đến lúc tick quay vòng.
- 🔬 Một mô phỏng tương tự với chu kỳ 1000 ms cho 23 lần kích hoạt trong 2 s quanh điểm quay vòng.

**Sửa:** `if (HAL_GetTick() - t_last >= 500U) { t_last = HAL_GetTick(); ... }` [chuyên đề 4.3].
</details>

### Thử thách 3 — Tối ưu tải bus cho 6 cảm biến
6 cảm biến song song trên I2C1 100 kHz, kèm LCD. Đo được tải bus ~60%, LCD giật. Đề xuất ít nhất 4 biện pháp và ước lượng hiệu quả từng biện pháp.

<details><summary>Lời giải</summary>

Gốc:
- Đọc số đo: 6 × 30 × 2,6 ms ≈ 47%.
- Hỏi I2C khi chưa có dữ liệu: 6 cảm biến × ~1 lần mỗi số đo × 0,4 ms × 30 Hz ≈ 7%.
- LCD: ~7%.
- **Tổng ≈ 61%.**

| Biện pháp | Hiệu quả ước lượng |
|---|---|
| Nối **GPIO1** cho cả 6 | Bỏ các lần hỏi I2C: −7% → ~54% |
| Chuyển cảm biến sang **I2C2 ở 400 kHz** | Thời gian đọc ~0,7 ms: phần đọc giảm từ 47% xuống ~13%; I2C1 chỉ còn LCD |
| Tăng budget lên 50 ms nếu ứng dụng chấp nhận 20 Hz | Phần đọc × 20/30: 47% → 31% |
| LCD chỉ ghi phần thay đổi; vẽ 2–4 lần mỗi giây | LCD 7% → ~2–3% |
| Chế độ tuần tự theo nhóm (nếu có nhiễu) | Giảm số lần đọc mỗi giây (đổi lại tốc độ mỗi con thấp hơn) |

Kết hợp **GPIO1 + I2C2 400 kHz** đưa tải bus cảm biến xuống ~13% và tách hẳn LCD sang I2C1.
</details>

### Thử thách 4 — Thiết kế phát hiện "cảm biến bị che"
Một cảm biến bị ai đó dán băng dính che kín. Nó vẫn trả lời I2C, vẫn đo ~30 Hz, nhưng số đo luôn rất gần (~20–40 mm, status 0 hoặc 3). Thiết kế cách phát hiện, chỉ dùng dữ liệu thư viện có sẵn.

<details><summary>Lời giải</summary>

**Dấu hiệu:** khoảng cách gần như không đổi và rất nhỏ trong thời gian dài, hoặc status 3 (min range) kéo dài. Tín hiệu phản xạ rất mạnh.

**Thuật toán:**
1. Mỗi số đo mới: nếu (`range_status == 3`) hoặc (`range_status == 0` và `range_mm < 50`) thì tăng bộ đếm `near_cnt`; ngược lại đặt `near_cnt = 0`.
2. Nếu `near_cnt` > 3 s × 30 Hz = 90 thì báo "cảm biến bị che".
3. Muốn chắc hơn, gọi `VL53L0X_GetRangingMeasurementData` riêng để xem `SignalRateRtnMegaCps`: rất cao (ví dụ > 20 MCPS) ở khoảng cách rất gần là đặc trưng của vật che sát mắt cảm biến.

**Lưu ý:** phải phân biệt với trường hợp thật sự có vật ở sát, ví dụ robot đang áp sát tường. Có thể kết hợp với cảm biến khác, hoặc với trạng thái chuyển động của robot.
</details>

### Thử thách 5 — Chứng minh tính công bằng của xoay vòng
Chứng minh: ở chế độ song song với `TOF_MAX_READS_PER_POLL = 1`, nếu cảm biến k luôn sẵn sàng (có số đo chờ đọc), thì nó được đọc trong tối đa N lần gọi `TOF_Poll` liên tiếp.

<details><summary>Lời giải</summary>

Gọi c_t là chỉ số được phục vụ ở lần gọi thứ t (`active` được gán bằng c_t). Lần gọi t + 1 duyệt bắt đầu từ `(c_t + 1) mod N` theo chiều vòng và phục vụ **cảm biến đầu tiên cho kết quả**. Vì k luôn sẵn sàng, mỗi lần gọi chắc chắn phục vụ một cảm biến nào đó, và cảm biến đó không nằm sau k.

Đặt d_t = khoảng cách theo chiều vòng từ `c_t + 1` tới k, với 0 ≤ d_t ≤ N − 1. Ở lần gọi t + 1:
- Nếu c_{t+1} = k thì xong.
- Nếu không, c_{t+1} nằm **trước** k. Điểm bắt đầu mới `c_{t+1} + 1` gần k hơn ít nhất 1 vị trí, nên d_{t+1} ≤ d_t − 1.

Vì d₀ ≤ N − 1 và d giảm ít nhất 1 sau mỗi lần không phục vụ k, nên sau tối đa N lần gọi, k phải được phục vụ. ∎

Kết hợp với chuyên đề 3.3: thời gian chờ tối đa ≤ N × (T_r + L).
</details>

---

**Hết bộ sách.** Quay lại [Mục lục bộ sách](README.md).
