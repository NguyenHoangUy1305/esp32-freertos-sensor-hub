# 📘 SỔ TAY KỸ THUẬT & LỘ TRÌNH THỰC HIỆN DỰ ÁN 3
## Đề tài: Hub cảm biến đa tác vụ sử dụng FreeRTOS & ESP-IDF (Multi-tasking Sensor Hub)

> **Người thực hiện:** Kỹ sư IoT / Embedded  
> **Thời gian:** 4 - 6 tuần (Tháng 04/2027 - Tháng 05/2027)  
> **Ghi chú tác giả:** Dự án chuyển dịch từ môi trường Arduino cơ bản sang **ESP-IDF** (Espressif IoT Development Framework) chuẩn chuyên nghiệp, làm chủ lập trình đa nhân vi điều khiển và hệ điều hành thời gian thực (RTOS).

---

## PHẦN 1: CƠ SỞ LÝ THUYẾT & NGUYÊN LÝ HOẠT ĐỘNG

### 1.1. Tại sao phải dùng FreeRTOS thay cho Arduino Super-Loop?
* **Hạn chế chết người của hàm `loop()` và `delay()` trong Arduino:**
  * Lệnh `delay(1000)` làm vi điều khiển bị "bắt cóc" hoàn toàn: CPU chạy các vòng lặp vô nghĩa (Nop) để chờ hết thời gian, không thể đọc nút nhấn khẩn cấp, không thể xử lý gói tin mạng đến, làm trễ hạn phản hồi (Miss Deadline).
* **Bản chất của RTOS (Real-Time Operating System):**
  * RTOS cung cấp một bộ điều phối (**Scheduler**) hoạt động theo cơ chế định thời ngắt nhịp (**SysTick Timer**).
  * Bộ điều phối luân chuyển CPU giữa các tác vụ (**Tasks**) dựa trên **Mức ưu tiên (Priority)** và cơ chế **Preemptive Multitasking** (Tác vụ ưu tiên cao hơn có quyền giành quyền thực thi ngay lập tức khi sẵn sàng).
  * Hàm `vTaskDelay(pdMS_TO_TICKS(100))` không làm đơ CPU; nó chuyển Task hiện tại sang trạng thái `BLOCKED` để CPU nhường tài nguyên cho Task khác làm việc!

### 1.2. Tận dụng kiến trúc 2 nhân (Dual-Core Xtensa LX6) của ESP32
* ESP32 tích hợp 2 nhân xử lý độc lập:
  * **Core 0 (PRO_CPU):** Mặc định xử lý các tác vụ nền tảng nặng về kết nối mạng (Wi-Fi, Bluetooth stack).
  * **Core 1 (APP_CPU):** Mặc định xử lý logic ứng dụng người dùng.
* Bằng hàm `xTaskCreatePinnedToCore()`, ta có thể phân bổ nhiệm vụ chính xác:
  * Ghim Task đọc cảm biến và bắt ngắt phần cứng vào **Core 0** để đảm bảo chu kỳ lấy mẫu chính xác từng mili-giây.
  * Ghim Task vẽ giao diện đồ họa OLED và tính toán thuật toán vào **Core 1** để không làm nghẽn tiến trình đọc cảm biến!

---

## PHẦN 2: CÁC CƠ CHẾ ĐỒNG BỘ HÓA CỐT LÕI (INTER-TASK COMMUNICATION)

### 2.1. FreeRTOS Queue (Hàng đợi an toàn dữ liệu)
* Tuyệt đối không dùng biến toàn cục (`global variables`) để chia sẻ dữ liệu giữa 2 task chạy song song, vì sẽ dẫn đến lỗi tranh chấp dữ liệu (**Race Condition**).
* **Cơ chế Queue:** Truyền dữ liệu theo nguyên tắc **Copy-by-Value** (sao chép nội dung struct vào vùng nhớ hàng đợi). Task nhận dữ liệu sẽ tự động chuyển sang trạng thái `BLOCKED` khi Queue rỗng, và tự động thức tỉnh ngay khi có phần tử mới được đẩy vào!

### 2.2. Mutex vs Binary Semaphore (Bảo vệ tài nguyên dùng chung)
* Khi cả 2 task cùng muốn ghi dữ liệu lên bus I2C hoặc cùng in ra cổng Serial:
  * **Mutex (Mutual Exclusion):** Đóng vai trò như "chìa khóa phòng vệ sinh". Task nào muốn dùng bus I2C phải lấy khóa (`xSemaphoreTake`). Dùng xong bắt buộc phải trả khóa (`xSemaphoreGive`).
  * **Cơ chế Priority Inheritance (Kế thừa mức ưu tiên):** Khác với Semaphore thông thường, Mutex của FreeRTOS tự động nâng mức ưu tiên của task đang giữ khóa lên bằng với task ưu tiên cao đang chờ khóa, ngăn ngừa triệt để lỗi kinh điển **Priority Inversion** làm treo hệ thống nhúng!

### 2.3. Task Watchdog Timer (TWDT - Tự phục hồi lỗi)
* Hệ thống nhúng công nghiệp phải hoạt động 24/7 không người giám sát. Nếu code bị rơi vào vòng lặp vô hạn hoặc deadlock, Watchdog Timer sẽ tự động can thiệp.
* Mỗi task quan trọng phải đăng ký với TWDT. Định kỳ mỗi chu kỳ, task phải gọi lệnh xóa chó canh cổng (`esp_task_wdt_reset()`). Nếu task bị đơ quá 3000ms không xóa, chip sẽ tự động reset và ghi lại nhật ký lỗi (Core Dump) vào bộ nhớ Flash.

---

## PHẦN 3: LỘ TRÌNH THỰC HIỆN CHI TIẾT 3 TUẦN

### Tuần 1: Nhập môn ESP-IDF & Phân chia Tác vụ Đa nhân
* [ ] Cài đặt tiện ích mở rộng **ESP-IDF v5.x** trên VS Code. Cấu hình công cụ biên dịch `CMake` và `ninja`.
* [ ] Tạo project mẫu đầu tiên với hàm khởi chạy chuẩn `app_main()`.
* [ ] Tạo 2 Task chạy trên 2 nhân khác nhau:
  * `Task_Sensor` (Priority 4, Core 0): Đọc thông số cảm biến BME280 mỗi 100ms.
  * `Task_Display` (Priority 2, Core 1): Vẽ giao diện lên màn hình OLED mỗi 500ms.
* [ ] Đo đạc mức sử dụng bộ nhớ: Dùng hàm `uxTaskGetStackHighWaterMark()` để xác định chính xác số byte RAM còn dư thừa của từng task, tối ưu hóa kích thước cấp phát Stack.

### Tuần 2: Đồng bộ Dữ liệu với Queue & Mutex
* [ ] Định nghĩa struct dữ liệu chuẩn:
  ```c
  typedef struct {
      float temperature;
      float humidity;
      float pressure;
      uint32_t timestamp;
  } sensor_data_t;
  ```
* [ ] Khởi tạo hàng đợi `sensor_queue = xQueueCreate(10, sizeof(sensor_data_t))`.
* [ ] Task đọc cảm biến gọi `xQueueSend()` để đẩy dữ liệu vào hàng đợi.
* [ ] Task hiển thị gọi `xQueueReceive()` để lấy dữ liệu ra vẽ đồ thị.
* [ ] Khởi tạo `i2c_mutex = xSemaphoreCreateMutex()` để bảo vệ an toàn tuyệt đối khi giao tiếp với màn hình OLED và cảm biến trên cùng đường truyền I2C.

### Tuần 3: Ngắt Phần Cứng (ISR) & Chống Treo với Watchdog Timer
* [ ] Cấu hình chân nút bấm GPIO với ngắt sườn xuống (**Falling Edge Interrupt**).
* [ ] Trong hàm ngắt ISR: Dùng `xSemaphoreGiveFromISR()` để gửi tín hiệu đánh thức Task xử lý khẩn cấp (không chạy vòng lặp trong ISR!).
* [ ] Đăng ký các task vào hệ thống giám sát **Task Watchdog Timer (TWDT)** với thời gian timeout 3 giây.
* [ ] Tạo một kịch bản giả lập lỗi (cố tình tạo vòng lặp `while(1)` trong 1 task): Quan sát Watchdog kích hoạt reset vi điều khiển và in ra thông báo lỗi stack trace an toàn.
* [ ] Hoàn thiện mã nguồn chuẩn C99, viết chú thích kỹ thuật từng hàm, cập nhật file `README.md`.
