# 📘 SỔ TAY KỸ THUẬT & LỘ TRÌNH THỰC HIỆN CHI TIẾT DỰ ÁN 3
## Đề tài: Hub cảm biến đa tác vụ sử dụng FreeRTOS & ESP-IDF (Multi-tasking Sensor Hub)

> **Tác giả:** Kỹ sư IoT & Hệ thống nhúng (NguyenHoangUy1305)  
> **Repository:** [NguyenHoangUy1305/esp32-freertos-sensor-hub](https://github.com/NguyenHoangUy1305/esp32-freertos-sensor-hub)  
> **Thời gian:** 4 - 6 tuần (Tháng 04/2027 - Tháng 05/2027)  
> **Mục tiêu:** Cung cấp cẩm nang lộ trình chuyển dịch từ môi trường Arduino cơ bản sang **ESP-IDF** chuyên nghiệp, làm chủ hệ điều hành thời gian thực (RTOS), lập trình đa nhân vi điều khiển (Dual-Core) và các công cụ đồng bộ hóa liên tác vụ IPC.

---

## MỤC LỤC
1. [PHẦN 1: TỔNG QUAN LỘ TRÌNH & MỤC TIÊU KỸ THUẬT](#phần-1-tổng-quan-lộ-trình--mục-tiêu-kỹ-thuật)
2. [PHẦN 2: LỘ TRÌNH CHI TIẾT 6 TUẦN TRIỂN KHAI (16/04/2027 - 31/05/2027)](#phần-2-lộ-trình-chi-tiết-6-tuần-triển-khai)
   - Tuần 1: Khởi tạo ESP-IDF v5.x, CMakeLists & sdkconfig (16/04 - 22/04/2027)
   - Tuần 2: Xây dựng Task đa cảm biến trên Core 0 với vTaskDelayUntil (23/04 - 29/04/2027)
   - Tuần 3: Đồng bộ hóa liên tác vụ với FreeRTOS Queue & Mutex I2C (30/04 - 06/05/2027)
   - Tuần 4: Xây dựng Task giao diện OLED & Thuật toán lọc số EMA trên Core 1 (07/05 - 13/05/2027)
   - Tuần 5: Giám sát độ tin cậy bằng Task Watchdog Timer (TWDT) (14/05 - 20/05/2027)
   - Tuần 6: Kiểm thử tải nặng, đo lường Stack High Water Mark & Nghiệm thu (21/05 - 31/05/2027)
3. [PHẦN 3: BẢNG BẪY LẬP TRÌNH NHÚNG RTOS & CÁCH PHÒNG TRÁNH CỐT LÕI](#phần-3-bảng-bẫy-lập-trình-nhúng-rtos--cách-phòng-tránh-cốt-lõi)
4. [PHẦN 4: BẢNG PHÂN BỔ BỘ NHỚ STACK & MỨC ĐỘ ƯU TIÊN CỦA CÁC TÁC VỤ](#phần-4-bảng-phân-bổ-bộ-nhớ-stack--mức-độ-ưu-tiên)
5. [PHẦN 5: BỘ CÂU HỎI PHỎNG VẤN KỸ THUẬT DÀNH CHO DỰ ÁN 3](#phần-5-bộ-câu-hỏi-phỏng-vấn-kỹ-thuật)

---

# PHẦN 1: TỔNG QUAN LỘ TRÌNH & MỤC TIÊU KỸ THUẬT

Dự án này là bước chuyển mình quyết định từ một "người dùng Arduino nghiệp dư" thành một **"Kỹ sư phần mềm nhúng chuyên nghiệp (Firmware Engineer)"**:

* **Nền tảng phát triển:** Chuyển sang **ESP-IDF (Espressif IoT Development Framework)** chuẩn doanh nghiệp, sử dụng build system `CMake` và cấu hình hệ thống bằng `menuconfig`.
* **Khai thác phần cứng tối đa:** Tận dụng trọn vẹn sức mạnh của kiến trúc 2 nhân độc lập Xtensa LX6 32-bit (Core 0 chuyên đo lường cảm biến tốc độ cao; Core 1 chuyên tính toán thuật toán và đồ họa).
* **Tính tất định (Determinism):** Loại bỏ hoàn toàn các hàm dừng `delay()` chết người, thay thế bằng bộ điều phối Preemptive Priority-Based Scheduler của FreeRTOS với độ chính xác chu kỳ $1\text{ ms}$.
* **An toàn luồng dữ liệu (Thread Safety):** Làm chủ các cơ chế IPC (Inter-Process Communication): Queue, Mutex, Priority Inheritance, Event Groups, Task Notifications.
* **Độ tin cậy tuyệt đối:** Giám sát liên tục nhịp sống của các Task bằng Task Watchdog Timer (TWDT), tự động phát hiện và phục hồi khi xảy ra Deadlock.

---

# PHẦN 2: LỘ TRÌNH CHI TIẾT 6 TUẦN TRIỂN KHAI

### Tuần 1: Khởi tạo ESP-IDF v5.x, CMakeLists & sdkconfig (16/04 - 22/04/2027)
* **Mục tiêu:** Cài đặt bộ công cụ biên dịch ESP-IDF, nắm vững cấu trúc CMake modular và tinh chỉnh các tham số hệ điều hành trong `sdkconfig`.
* **Công việc chi tiết:**
  * Cài đặt ESP-IDF v5.x extension trên VS Code (hoặc cài đặt qua dòng lệnh CLI trên Windows/Linux).
  * Tạo cấu trúc dự án chuẩn ESP-IDF:
    ```text
    03-freertos-sensor-hub/
    ├── CMakeLists.txt
    ├── sdkconfig
    ├── main/
    │   ├── CMakeLists.txt
    │   ├── main.c
    │   ├── sensor_tasks.c
    │   └── display_tasks.c
    └── components/
    ```
  * Chạy `idf.py menuconfig`:
    - Cấu hình tần số ngắt nhịp FreeRTOS: `CONFIG_FREERTOS_HZ = 1000` (1 tick = $1\text{ ms}$).
    - Bật hỗ trợ Task Watchdog Timer: `CONFIG_ESP_TASK_WDT_EN = y`.
    - Bật kiểm tra tràn ngăn xếp: `CONFIG_FREERTOS_CHECK_STACKOVERFLOW = 2`.
  * Viết chương trình `main.c` tối giản: Khởi tạo hàm `app_main()` và in thông số vi xử lý (tần số CPU, dung lượng RAM khả dụng).
* **Nghiệm thu:** Biên dịch và nạp thành công chương trình bằng lệnh `idf.py build flash monitor`.

---

### Tuần 2: Xây dựng Task đa cảm biến trên Core 0 với vTaskDelayUntil (23/04 - 29/04/2027)
* **Mục tiêu:** Lập trình các tác vụ thu thập dữ liệu cảm biến chạy song song trên Core 0 với chu kỳ lấy mẫu tuần hoàn chính xác tuyệt đối.
* **Công việc chi tiết:**
  * Tìm hiểu sự khác nhau giữa `vTaskDelay()` (chỉ dừng tương đối) và `vTaskDelayUntil()` (đảm bảo tần số chu kỳ tuần hoàn chính xác, triệt tiêu trôi thời gian tích lũy).
  * Viết `Task_Sensor_BME`: Ghim vào **Core 0**, Priority 3, chu kỳ tuần hoàn $200\text{ ms}$.
  * Viết `Task_Sensor_MPU`: Ghim vào **Core 0**, Priority 4 (ưu tiên cao nhất), chu kỳ lấy mẫu gia tốc cao tốc $50\text{ ms}$.
  * Viết `Task_ADC_LDR`: Ghim vào **Core 0**, Priority 2, đọc điện áp ADC 12-bit từ quang trở mỗi $100\text{ ms}$.
  * Sử dụng hàm `xTaskCreatePinnedToCore()` để chỉ định rõ nhân xử lý `PRO_CPU_NUM` (Core 0).
  * Dùng chân GPIO toggle và máy đo xung Logic Analyzer để kiểm tra độ chính xác chu kỳ $50\text{ ms}$ của tác vụ.
* **Nghiệm thu:** Cả 3 tác vụ chạy trơn tru trên Core 0; chu kỳ lấy mẫu đo được sai số dưới $0.1\text{ ms}$.

---

### Tuần 3: Đồng bộ hóa liên tác vụ với FreeRTOS Queue & Mutex I2C (30/04 - 06/05/2027)
* **Mục tiêu:** Xây dựng cầu nối truyền dữ liệu an toàn tuyến trình giữa các tác vụ và bảo vệ tài nguyên bus phần cứng dùng chung.
* **Công việc chi tiết:**
  * Định nghĩa cấu trúc dữ liệu chung `SensorPayload_t`: Chứa nhiệt độ, độ ẩm, áp suất, gia tốc 3 trục (ax, ay, az), cường độ ánh sáng và timestamp.
  * Khởi tạo hàng đợi FreeRTOS Queue: `xQueue_SensorData = xQueueCreate(10, sizeof(SensorPayload_t))`.
  * Lập trình các Task cảm biến trên Core 0 đóng gói dữ liệu và gọi `xQueueSend(xQueue_SensorData, &payload, pdMS_TO_TICKS(10))`.
  * Khởi tạo khóa loại trừ tương hỗ Mutex: `xMutex_I2C = xSemaphoreCreateMutex()`.
  * Bọc tất cả các thao tác đọc ghi bus $I^2C$ của BME280, MPU6050 và OLED bên trong cặp lệnh:
    ```c
    if (xSemaphoreTake(xMutex_I2C, pdMS_TO_TICKS(50)) == pdTRUE) {
        // Thực hiện đọc/ghi I2C an toàn
        xSemaphoreGive(xMutex_I2C);
    }
    ```
  * Chứng minh thực nghiệm: Cố tình cho 3 tác vụ cùng truy cập $I^2C$ đồng thời mà không bị lỗi xung đột hoặc đè dữ liệu lên nhau.
* **Nghiệm thu:** Hàng đợi Queue truyền nhận dữ liệu mượt mà; không xảy ra lỗi tranh chấp bus phần cứng (Race Condition).

---

### Tuần 4: Xây dựng Task giao diện OLED & Thuật toán lọc số EMA trên Core 1 (07/05 - 13/05/2027)
* **Mục tiêu:** Tách biệt hoàn toàn tác vụ tính toán nặng và vẽ màn hình sang Core 1 để không làm nghẽn tiến trình lấy mẫu của Core 0.
* **Công việc chi tiết:**
  * Viết `Task_DSP_Filter` ghim vào **Core 1**, Priority 2:
    - Đọc dữ liệu thô từ `xQueue_SensorData`.
    - Áp dụng bộ lọc trung bình động lũy thừa (Exponential Moving Average - EMA) với hệ số $\alpha = 0.2$ để làm mượt nhiễu nhiệt độ và gia tốc:
      $$S_t = \alpha \cdot Y_t + (1 - \alpha) \cdot S_{t-1}$$
    - Tính toán thuật toán phát hiện rơi ngã (Fall Detection) dựa trên độ lớn gia tốc tổng hợp: $A = \sqrt{a_x^2 + a_y^2 + a_z^2}$.
  * Viết `Task_Display_OLED` ghim vào **Core 1**, Priority 1 (chu kỳ $100\text{ ms}$):
    - Vẽ biểu đồ đường dao động gia tốc thời gian thực trên màn hình OLED SSD1306.
    - Hiển thị nhiệt độ, độ ẩm và biểu tượng nhịp tim CPU Core 0 / Core 1.
  * Viết `Task_Telemetry_UART` ghim vào **Core 1**, Priority 1: Xuất dữ liệu định dạng JSON qua cổng Serial để giao tiếp máy tính.
* **Nghiệm thu:** Màn hình OLED vẽ đồ thị động $10\text{ FPS}$ trên Core 1 mà chu kỳ đọc cảm biến cao tốc trên Core 0 hoàn toàn không bị ảnh hưởng.

---

### Tuần 5: Giám sát độ tin cậy bằng Task Watchdog Timer (TWDT) (14/05 - 20/05/2027)
* **Mục tiêu:** Tích hợp bộ định thời giám sát phần cứng, đảm bảo hệ thống có khả năng tự phục hồi tức thì khi xảy ra lỗi kẹt tác vụ hoặc Deadlock.
* **Công việc chi tiết:**
  * Khởi tạo Task Watchdog Timer với chu kỳ timeout $3000\text{ ms}$ bằng hàm `esp_task_wdt_init()`.
  * Đăng ký các tác vụ quan trọng vào danh sách theo dõi của TWDT: `esp_task_wdt_add(NULL)`.
  * Định kỳ trong mỗi vòng lặp `while(1)`, các tác vụ phải gọi lệnh `esp_task_wdt_reset()` để xác nhận còn sống.
  * Thử nghiệm mô phỏng sự cố:
    - Cố tình viết một vòng lặp vô tận `while(1) {}` bên trong `Task_Sensor_BME` để giả lập lỗi phần mềm kẹt cứng.
    - Quan sát phản ứng của TWDT: Sau đúng 3 giây, ngắt Watchdog được kích hoạt, in toàn bộ Stack Trace ra Serial và tự động Reset vi điều khiển để cứu hệ thống.
  * Tìm hiểu cơ chế Kế thừa độ ưu tiên (Priority Inheritance) của FreeRTOS Mutex trong việc ngăn chặn thảm họa Nghịch đảo độ ưu tiên (Priority Inversion).
* **Nghiệm thu:** Watchdog hoạt động chuẩn xác; tự động phát hiện và khởi động lại vi điều khiển trong vòng 3 giây khi có tác vụ bị kẹt.

---

### Tuần 6: Kiểm thử tải nặng, đo lường Stack High Water Mark & Nghiệm thu (21/05 - 31/05/2027)
* **Mục tiêu:** Đo lường tối ưu hóa bộ nhớ RAM, hoàn thiện hồ sơ kỹ thuật chuyên sâu và đóng gói dự án tốt nghiệp.
* **Công việc chi tiết:**
  * Sử dụng hàm `uxTaskGetStackHighWaterMark()` để đo lượng RAM ngăn xếp tối thiểu còn dư thừa của từng Task trong suốt quá trình hoạt động.
  * Tinh chỉnh lại kích thước Stack cấp phát ban đầu: Cắt giảm RAM thừa của các Task nhẹ, tăng RAM cho Task vẽ đồ thị OLED để tránh lỗi tràn ngăn xếp (Stack Overflow).
  * Chạy bài kiểm thử tải nặng liên tục 48 tiếng (Stress Test): Đo nhiệt độ chip ESP32, kiểm tra dung lượng Heap RAM tự do ổn định.
  * Biên tập video quay demo: Trình bày chi tiết cơ chế chạy song song 2 nhân, mô phỏng tháo lỏng dây cảm biến để kiểm tra cơ chế chống lỗi, hiển thị đồ họa OLED.
  * Hoàn thiện tài liệu kỹ thuật cuối cùng, chuẩn bị sẵn sàng cho Portfolio tuyển dụng kỹ sư nhúng chuyên nghiệp.
* **Nghiệm thu:** Hệ thống chạy ổn định 48h không sập; bộ nhớ RAM được tối ưu hóa tối đa; hồ sơ dự án đạt chuẩn kỹ sư Firmware chuyên nghiệp.

---

# PHẦN 3: BẢNG BẪY LẬP TRÌNH NHÚNG RTOS & CÁCH PHÒNG TRÁNH CỐT LÕI

| Bẫy lập trình FreeRTOS | Hậu quả nghiêm trọng | Bản chất lỗi kỹ thuật | Giải pháp kỹ thuật chuẩn xác |
| :--- | :--- | :--- | :--- |
| **Bẫy gọi hàm FreeRTOS trong ISR** | Vi điều khiển bị Panic Crash ngay lập tức, báo lỗi Guru Meditation | Các hàm thông thường như `xQueueSend()` hoặc `xSemaphoreGive()` có cơ chế chặn (blocking), tuyệt đối không được gọi trong ngắt phần cứng (ISR). | **Bắt buộc dùng phiên bản chuyên biệt cho ngắt:** `xQueueSendFromISR()`, `xSemaphoreGiveFromISR()` kèm cờ `pxHigherPriorityTaskWoken`. |
| **Bẫy tràn ngăn xếp (Stack Overflow)** | Biến nhớ toàn cục bị ghi đè, hệ thống khởi động lại ngẫu nhiên khó hiểu | Cấp phát mảng cục bộ quá lớn bên trong hàm của Task (ví dụ `char buffer[2048]`) vượt quá dung lượng Stack cấp phát ban đầu khi tạo Task. | Cấp phát bộ nhớ Stack đủ lớn khi gọi `xTaskCreate`. Bật cờ `configCHECK_FOR_STACK_OVERFLOW = 2` để phát hiện lỗi ngay lập tức. |
| **Bẫy trôi thời gian khi dùng `vTaskDelay`** | Chu kỳ đọc cảm biến bị giãn nở dần theo thời gian, không còn chuẩn | `vTaskDelay(100)` chỉ tạm dừng $100\text{ ms}$ kể từ thời điểm kết thúc xử lý code, tổng chu kỳ thực tế sẽ là: $100\text{ ms} + \text{Execution Time}$. | **Bắt buộc dùng `vTaskDelayUntil()`**: Hàm tự động trừ đi thời gian thực thi của code, đảm bảo chu kỳ tuần hoàn chuẩn xác từng mili-giây. |
| **Bẫy thảm họa Nghịch đảo độ ưu tiên** | Task ưu tiên cao nhất bị kẹt cứng vô thời hạn chờ Task ưu tiên thấp | Task thấp đang giữ tài nguyên, Task cao bị chặn chờ, Task trung bình nhảy vào chiếm CPU không cho Task thấp chạy để nhả khóa. | **Luôn dùng Mutex (`xSemaphoreCreateMutex()`)** vì nó có sẵn cơ chế Kế thừa độ ưu tiên (Priority Inheritance). Không dùng Binary Semaphore để bảo vệ tài nguyên. |
| **Bẫy chia sẻ biến toàn cục không khóa** | Số liệu đo đạc bị sai lệch, byte cao của lần đo này dính byte thấp của lần đo trước | Hai Task chạy trên 2 Core khác nhau cùng đọc/ghi một biến toàn cục `float temperature` tại cùng một chu kỳ xung nhịp (Race Condition). | **Tuyệt đối không dùng biến toàn cục trần**. Truyền dữ liệu an toàn qua **FreeRTOS Queue** hoặc khóa bảo vệ bằng Spinlock / Mutex. |

---

# PHẦN 4: BẢNG PHÂN BỔ BỘ NHỚ STACK & MỨC ĐỘ ƯU TIÊN

```text
+-----------------------+--------+----------+------------+------------------------------------------------------+
| Tên Tác Vụ (Task)     | Nhân   | Priority | Stack Size | Nhiệm vụ & Vai trò kỹ thuật                          |
+-----------------------+--------+----------+------------+------------------------------------------------------+
| Task_Sensor_MPU       | Core 0 |    4     |  3072 B    | Đọc gia tốc cao tốc (50ms) - Cần ưu tiên cao nhất     |
| Task_Sensor_BME       | Core 0 |    3     |  3072 B    | Đọc nhiệt độ, độ ẩm, áp suất (200ms)                 |
| Task_ADC_LDR          | Core 0 |    2     |  2048 B    | Lấy mẫu kênh ADC quang trở (100ms)                   |
| Task_DSP_Filter       | Core 1 |    2     |  4096 B    | Xử lý lọc số EMA, thuật toán phát hiện ngã           |
| Task_Display_OLED     | Core 1 |    1     |  4096 B    | Vẽ đồ thị giao diện đồ họa OLED SSD1306 (100ms)       |
| Task_Telemetry_UART   | Core 1 |    1     |  2048 B    | Đóng gói và xuất chuỗi JSON qua cổng Serial          |
| Task Watchdog (TWDT)  | Cả 2   |   ISR    |  Phần cứng | Bộ đếm định thời phần cứng giám sát nhịp sống 3000ms |
+-----------------------+--------+----------+------------+------------------------------------------------------+
```

---

# PHẦN 5: BỘ CÂU HỎI PHỎNG VẤN KỸ THUẬT DÀNH CHO DỰ ÁN 3

1. **Câu hỏi:** *Sự khác biệt cốt lõi giữa hệ điều hành đa nhiệm thông thường (Windows/Linux) và RTOS là gì?*  
   **Trả lời:** Hệ điều hành thông thường (GPOS) được thiết kế tối ưu hóa cho thông lượng tổng thể (Throughput) và sự công bằng giữa các tiến trình. Trong khi đó, RTOS được thiết kế để đảm bảo tính tất định (Determinism) và thời gian đáp ứng thời gian thực (Guaranteed Response Time). Trong RTOS, một tác vụ ưu tiên cao hơn đang ở trạng thái sẵn sàng (Ready) sẽ lập tức chiếm quyền CPU (Preemptive Scheduling) để thực thi ngay trong chu kỳ ngắt tiếp theo mà không bị trì hoãn, đảm bảo hoàn thành trước thời hạn chót (Deadline).

2. **Câu hỏi:** *Hãy giải thích hiện tượng Nghịch đảo độ ưu tiên (Priority Inversion) và cơ chế Kế thừa độ ưu tiên (Priority Inheritance) đã cứu robot tự hành Mars Pathfinder của NASA như thế nào?*  
   **Trả lời:** Hiện tượng xảy ra khi có 3 Task: Task Cao, Task Trung bình và Task Thấp. Task Thấp chiếm giữ Mutex tài nguyên dùng chung. Task Cao cần tài nguyên này nên bị chuyển sang trạng thái Blocked. Lúc này, Task Trung bình (không cần tài nguyên) xuất hiện, vì có mức ưu tiên cao hơn Task Thấp nên Task Trung bình chiếm CPU chạy miệt mài. Hậu quả là Task Cao (ưu tiên cao nhất) bị kẹt gián tiếp chờ Task Trung bình chạy xong! Trong vụ Mars Pathfinder năm 1997, lỗi này khiến hệ thống bị kích hoạt Watchdog Reset liên tục. Giải pháp là cơ chế Kế thừa độ ưu tiên (Priority Inheritance): Khi Task Cao bị kẹt chờ Mutex từ Task Thấp, FreeRTOS sẽ tự động nâng tạm thời mức ưu tiên của Task Thấp lên bằng Task Cao, giúp Task Thấp không bị Task Trung bình chen ngang, nhanh chóng hoàn thành và nhả Mutex cho Task Cao.

3. **Câu hỏi:** *Tại sao khi lập trình tác vụ định kỳ trong FreeRTOS lại bắt buộc dùng `vTaskDelayUntil()` thay vì `vTaskDelay()`?*  
   **Trả lời:** Hàm `vTaskDelay(period)` chỉ tạm dừng tác vụ một khoảng thời gian tương đối kể từ khi hàm được gọi. Do đó, tổng chu kỳ thực tế sẽ bằng thời gian tạm dừng cộng với thời gian CPU thực thi mã lệnh của Task (Execution Time). Sau nhiều chu kỳ, thời gian thực thi này sẽ tích lũy tạo ra hiện tượng trôi chu kỳ (Cumulative Drift). Ngược lại, `vTaskDelayUntil(&xLastWakeTime, period)` lưu giữ mốc thời gian đánh thức tuyệt đối, nó tự động trừ đi thời gian thực thi của code, đảm bảo chu kỳ lặp lại luôn chính xác tuyệt đối từng mili-giây, cực kỳ quan trọng cho các bộ lọc số DSP và lấy mẫu cảm biến.

4. **Câu hỏi:** *Tại sao việc truyền dữ liệu giữa các Task qua FreeRTOS Queue lại an toàn hơn biến toàn cục?*  
   **Trả lời:** FreeRTOS Queue sử dụng cơ chế truyền sao chép giá trị (Copy by Value), dữ liệu được sao chép nguyên vẹn vào bộ đệm nội của Queue, tránh nguy cơ vùng nhớ bị Task khác sửa đổi khi đang truyền. Đồng thời, Queue tích hợp sẵn cơ chế khóa ngắt an toàn tuyến trình (Thread-Safe) và cơ chế Blocking: Khi Queue rỗng, Task đọc tự động chuyển sang trạng thái Blocked chờ dữ liệu mà không tiêu tốn chu kỳ CPU nào. Sử dụng biến toàn cục trần trên vi điều khiển đa nhân (Dual-Core) rất dễ bị lỗi Race Condition khi 2 nhân cùng ghi đè dữ liệu tại cùng một chu kỳ xung nhịp.
