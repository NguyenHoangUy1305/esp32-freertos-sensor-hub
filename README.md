# ⚙️ MULTI-TASKING FREERTOS SENSOR HUB (ESP-IDF)
> **Tên đề tài:** High-Performance Dual-Core Sensor Hub with FreeRTOS Inter-Task Communication and Fault Recovery  
> **Thời gian:** Tháng 05/2027 (3 tuần)  
> **Mục tiêu:** Chinh phục kỹ năng Firmware Engineer chuyên sâu; làm chủ hệ điều hành thời gian thực (RTOS), lập trình đa nhân vi điều khiển, đồng bộ tài nguyên và xử lý lỗi phần cứng.

---

> 📘 **SỔ TAY KỸ THUẬT & LỘ TRÌNH 3 TUẦN CHI TIẾT:** Xem toàn bộ lý thuyết ESP-IDF, phân bổ Dual-core, FreeRTOS Queue, Mutex và Watchdog tại [`docs/ROADMAP_KY_THUAT.md`](./docs/ROADMAP_KY_THUAT.md)


## 1. CẤU TRÚC THƯ MỤC DỰ ÁN (CHUẨN ESP-IDF)
```text
03-freertos-sensor-hub/
├── main/           # Mã nguồn chính theo kiến trúc ESP-IDF
│   ├── main.c      # Điểm khởi chạy app_main, tạo task, queue, mutex
│   ├── sensor_task.c   # Task đọc cảm biến (Core 0)
│   ├── display_task.c  # Task hiển thị OLED & in Serial (Core 1)
│   ├── comms_task.c    # Task xử lý truyền tin
│   └── idf_component.yml
├── components/     # Các module ngoại vi tự viết (OLED SSD1306, BME280 Driver)
├── docs/           # Sơ đồ phân bổ nhân (Dual Core Map), biểu đồ chu kỳ Task
└── README.md       # Tài liệu đặc tả dự án
```

---

## 2. NỀN TẢNG KỸ THUẬT & KHÁI NIỆM CỐT LÕI
* **Nền tảng:** C/C++ lập trình trực tiếp trên **ESP-IDF v5.x** (Espressif IoT Development Framework).
* **Hệ điều hành:** FreeRTOS Kernel được tối ưu hóa cho vi xử lý 2 nhân Xtensa LX6 của ESP32.
* **Xóa bỏ hoàn toàn lệnh `delay()`:** Thay thế 100% bằng cơ chế Non-blocking: `vTaskDelayUntil()`, `xQueueReceive()`, `xSemaphoreTake()`.

---

## 3. THIẾT KẾ PHÂN BỔ NHÂN VÀ TÁC VỤ (DUAL-CORE ARCHITECTURE)

```text
 ┌─────────────────────────────────────────┐       ┌─────────────────────────────────────────┐
 │               CORE 0 (PRO_CPU)          │       │               CORE 1 (APP_CPU)          │
 │                                         │       │                                         │
 │  [Task 1: SensorAcquisitionTask]        │       │  [Task 3: DisplayAndUiTask]             │
 │  - Chu kỳ: 100ms                        │       │  - Nhận data từ Queue                   │
 │  - Đọc BME280 / ADC Biến trở            │       │  - Vẽ giao diện đồ họa lên OLED         │
 │  - Đóng gói struct SensorData           │       │  - Ưu tiên: Priority 2                  │
 │  - Ưu tiên: Priority 4 (Cao)            │       │                                         │
 │                    │                    │       └────────────────────▲────────────────────┘
 └────────────────────┼────────────────────┘                            │
                      │                                                 │
                      ▼ xQueueSend()                                    │ xQueueReceive()
            ┌───────────────────────────────────────────────────────────┴────────────────────┐
            │                 FreeRTOS Queue: sensor_data_queue                              │
            │                 (Kích thước hàng đợi: 10 phần tử struct)                       │
            └───────────────────────────────────────────────────────────┬────────────────────┘
                                                                        │
 ┌─────────────────────────────────────────┐                            │
 │  [Task 2: HardwareInterruptHandler]     │                            │
 │  - Bắt ngắt nút nhấn GPIO (ISR)         │                            ▼
 │  - Gửi tín hiệu xSemaphoreGiveFromISR() │               [Task 4: DiagnosticsAndLoggingTask]
 │  - Xử lý đổi chế độ đo tức thì          │               - Giám sát RAM còn lại (Free Heap)
 └─────────────────────────────────────────┘               - In log qua UART / Serial
```

---

## 4. CƠ CHẾ ĐỒNG BỘ HÓA & AN TOÀN BỘ NHỚ
1. **FreeRTOS Queue (`xQueueCreate`):** Truyền dữ liệu kiểu giá trị an toàn giữa Task đọc cảm biến và Task hiển thị, loại bỏ hoàn toàn biến toàn cục (Global Variables) dễ gây tranh chấp dữ liệu (Race Condition).
2. **Mutex (`xSemaphoreCreateMutex`):** Khóa quyền truy cập Bus I2C dùng chung khi có nhiều hơn 1 task muốn giao tiếp ngoại vi.
3. **Event Groups (`xEventGroupCreate`):** Đồng bộ trạng thái: Chỉ kích hoạt chu trình tính toán khi cờ `EVENT_WIFI_CONNECTED` và `EVENT_SENSOR_CALIBRATED` đều được bật.
4. **Giám sát Stack (`uxTaskGetStackHighWaterMark`):** Đo đạc chính xác lượng byte RAM tối thiểu còn lại của từng task trong suốt quá trình chạy để phòng chống triệt để lỗi tràn Stack (Stack Overflow Crash).

---

## 5. TỰ PHỤC HỒI LỖI PHẦN CỨNG (WATCHDOG TIMER & FAULT RECOVERY)
* Đăng ký từng task vào **Task Watchdog Timer (TWDT)** với chu kỳ giám sát 3000ms.
* Mỗi task phải gọi `esp_task_wdt_reset()` trong chu kỳ hoạt động. Nếu một task bị treo cứng (Deadlock hoặc vòng lặp vô hạn), vi điều khiển sẽ tự động ghi vết lỗi (Core Dump) và kích hoạt Reset hệ thống an toàn.
