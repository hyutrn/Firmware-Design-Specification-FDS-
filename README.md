# Firmware-Design-Specification-FDS
**Architecture: 3-Tier, Event-driven, Bare-metal (No RTOS)**
---
## 1. MỤC TIÊU THIẾT KẾ
Firmware VD26 Display phải đáp ứng:
* Đúng 100% yêu cầu chức năng
* Không block CPU, không delay busy-wait
* Không xử lý logic nghiệp vụ trong ISR
* Có khả năng mở rộng, bảo trì, debug
* Hoạt động ổn định trên MCU **không RTOS**
Thiết kế hướng tới:
* Event-driven architecture
* Finite State Machine (FSM)
* Timer-based scheduling
* Tách biệt phần cứng và logic
---
## 2. PHẠM VI HỆ THỐNG
### 2.1 Phần cứng sử dụng
* GPIO (LED, Button, HOLD_PWR)
* Timer
* External Interrupt
* UART
* ADC
* DMA (cho ADC)
### 2.2 Không sử dụng
* RTOS / FreeRTOS
* Dynamic memory allocation
* Blocking delay
---
## 3. KIẾN TRÚC TỔNG THỂ (3-TIER ARCHITECTURE)
```
┌───────────────────────────────┐
│ APPLICATION TIER (APP)        │
│ - FSM (SYS / MODE / LIGHT)    │
│ - Mode management             │
│ - Lighting logic              │
│ - Battery level logic         │
├───────────────────────────────┤
│ SERVICE TIER (SVC)            │
│ - Button service (Power/Mode) │
│ - LED service                 │
│ - Timer service               │
│ - UART service                │
│ - ADC service (DMA optional)  │
│ - Event manager               │
├───────────────────────────────┤
│ DRIVER / HAL TIER (DRV)       │
│ - GPIO                        │
│ - External Interrupt          │
│ - Timer HW                    │
│ - UART HW                     │
│ - ADC HW                      │
│ - DMA controller              │
└───────────────────────────────┘
```
### 3.1 Nguyên tắc bắt buộc

* APP **không gọi register**
* ISR **chỉ sinh event**
* Service **không chứa logic nghiệp vụ**
* Mỗi tầng có trách nhiệm duy nhất
---

## 4. CÁC CÔNG NGHỆ & VAI TRÒ
| Công nghệ          | Vai trò trong hệ thống               |
| ------------------ | ------------------------------------ |
| External Interrupt | Phát hiện nút nhấn                   |
| Timer              | Time base, hold detection, animation |
| Event              | Giao tiếp giữa các tầng              |
| State Machine      | Quản lý trạng thái hệ thống          |
| UART               | Truyền trạng thái ra ngoài           |
| ADC                | Đọc pin, quang trở                   |
| DMA                | Thu thập ADC liên tục, giảm tải CPU  |
---
## 5. EVENT SYSTEM DESIGN
### 5.1 Nhóm Event chính
**Button Events**
* EVT_POWER_PRESS
* EVT_POWER_RELEASE
* EVT_POWER_HOLD_5S
* EVT_MODE_SINGLE
* EVT_MODE_HOLD_3S
**Timer Events**
* EVT_TICK_1MS
* EVT_TICK_10MS
* EVT_TICK_100MS
**ADC Events**
* EVT_BATTERY_UPDATE
* EVT_LDR_DARK
* EVT_LDR_BRIGHT
**System Events**
* EVT_SYS_ON_COMPLETE
* EVT_SYS_OFF_COMPLETE
---
### 5.2 Luồng xử lý chuẩn
```
Interrupt / DMA Complete
        ↓
   Service Layer
        ↓
     Set Event
        ↓
 Application FSM
        ↓
   Action / Transition
```
---
## 6. STATE MACHINE THIẾT KẾ
### 6.1 System State (FSM cấp cao)

| State            | Mô tả        |
| ---------------- | ------------ |
| SYS_OFF          | Hệ thống tắt |
| SYS_POWERING_ON  | Đang bật     |
| SYS_ON           | Hoạt động    |
| SYS_POWERING_OFF | Đang tắt     |
➡ Hệ thống **luôn chỉ ở 1 state**.
---
### 6.2 Mode State (khi SYS_ON)
| Mode       | LED  |
| ---------- | ---- |
| MODE_ECO   | LED2 |
| MODE_SPORT | LED3 |
Quy tắc:
* Chỉ tồn tại khi `SYS_ON`
* Không được đồng thời ECO + SPORT
---
### 6.3 Lighting State
| State      | Mô tả                     |
| ---------- | ------------------------- |
| LIGHT_OFF  | Đèn tắt                   |
| LIGHT_ON   | Đèn bật thủ công          |
| LIGHT_AUTO | Điều khiển bằng quang trở |
Manual override:
* MODE hold 3s → toggle LIGHT_ON / OFF
---
## 7. BUTTON HANDLING STRATEGY
### Power Button
* External interrupt phát hiện cạnh
* Timer dùng để đo thời gian giữ
* Sau 5s → EVT_POWER_HOLD_5S
### Mode Button
* External interrupt
* Timer phân biệt:
  * Nhấn nhả → EVT_MODE_SINGLE
  * Giữ ≥ 3s → EVT_MODE_HOLD_3S
➡ Toàn bộ debounce & timing nằm ở **Service Button**
---
## 8. TIMER SYSTEM DESIGN
### 8.1 Time base
* 1ms hardware timer interrupt
### 8.2 Timer dùng cho
* Button hold detection
* LED animation
* Blink 3 lần
* Periodic ADC sampling
* FSM timeout
⛔ Không dùng delay blocking
---
## 9. LED MANAGEMENT DESIGN
### 9.1 Phân loại LED
| LED    | Chức năng     |
| ------ | ------------- |
| LED1   | Lighting      |
| LED2   | ECO           |
| LED3   | SPORT         |
| LED7–4 | Battery level |
### 9.2 LED Animation
* Điều khiển bởi Service LED
* APP chỉ ra lệnh logic (start / stop / pattern)
---
## 10. BATTERY MANAGEMENT (ADC + DMA)
### 10.1 Thu thập dữ liệu
* ADC chạy định kỳ
* DMA ghi dữ liệu ADC vào buffer vòng
### 10.2 Xử lý
1. DMA complete / half complete → event
2. Service ADC lọc & tính điện áp
3. Application tính % pin
4. Mapping % → LED7–4
| %      | LED  |
| ------ | ---- |
| 0–24   | LED7 |
| 25–49  | LED6 |
| 50–74  | LED5 |
| 75–100 | LED4 |
📌 DMA giúp:
* Không mất mẫu
* CPU rảnh
* Sampling ổn định
---
## 11. LIGHT AUTO (LDR) DESIGN
### 11.1 Điều kiện
* Chỉ active khi `LIGHT_AUTO`
* SYS phải ở `SYS_ON`
### 11.2 Logic
* Tối → LED1 ON + UART "on light"
* Sáng → LED1 OFF + UART "off light"
Manual override:
* MODE hold 3s → ưu tiên hơn AUTO
---
## 12. UART COMMUNICATION DESIGN
### 12.1 Chức năng
* Chỉ truyền dữ liệu
* Không nhận command
### 12.2 Message
* "mode ECO"
* "mode SPORT"
* "on light"
* "off light"
UART được gọi từ **Application**, thực thi bởi **Service UART**
---
## 13. POWER ON / OFF SEQUENCE (FSM)
### 13.1 Power ON
```
SYS_OFF
 ↓ EVT_POWER_HOLD_5S
SYS_POWERING_ON
 ↓ HOLD_PWR = HIGH
 ↓ LED 7 → 4 sáng dần
 ↓ Blink 3 lần
 ↓ Set MODE_ECO
 ↓ Enable MODE button
SYS_ON
```
### 13.2 Power OFF
```
SYS_ON
 ↓ EVT_POWER_HOLD_5S
SYS_POWERING_OFF
 ↓ Disable MODE button
 ↓ LED1/2/3 OFF
 ↓ LED 4 → 7 tắt dần
 ↓ Blink 3 lần
 ↓ HOLD_PWR = LOW
SYS_OFF
```
---
## 14. ĐẢM BẢO TÍNH ĐÚNG & ỔN ĐỊNH
* FSM loại bỏ trạng thái không hợp lệ
* Event tránh xử lý trong ISR
* DMA giảm tải CPU
* Timer thay thế delay
* Tier Architecture cô lập rủi ro
---
## 15. Pin config 
### Button 
| Button Pin| MCU Pin   |Function             |Internal Configuration |
|-----------|-----------|---------------------|-----------------------|
| SW_SIG    | PA0       |Power control button |Input with pull-up     |
| MODE_SIG  | PA1       |Mode selection button|Input with pull-up     |
### LED
| Led Pin   | MCU Pin   |Function             |Active State       |
|-----------|-----------|---------------------|-------------------|
| LED 1     | PA7       |Lighting indicator   | Output active low |
| LED 2     | PA8       |ECO mode indicator   | Output active low |
| LED 3     | PA9       |SPORT mode indicator | Output active low |
| LED 4     | PA12      |75-100% battery      | Output active low |
| LED 5     | PA13      |50-74% battery       | Output active low |
| LED 6     | PA14      |25-49% battery       | Output active low |
| LED 7     | PA15      |0-24% battery        | Output active low |
### Power Control Signal
|Signal Name| MCU Pin   |Function             |Active State       |
|-----------|-----------|---------------------|-------------------|
| HOLD_PWR  | PA22      |Power enable output  | Output active high|
### Communications
|UART - Comm| MCU Pin   |
|-----------|-----------|
| UART0_TX  | PA10      |
| UART0_RX  | PA11      |
### LDR & BATTERY
| Name          | MCU Pin   |
|---------------|-----------|
|LDR            | PA24      |
|V_BATTERY      | PA25      |
---
## 16. Mode Button Design
### 16.1 Behavior
| Nhấn	           | Hành động                              |
|------------------|----------------------------------------|
|Nhấn nhả (single) | Chuyển qua lại MODE_ECO / MODE_SPORT:  |
|                  | - ECO ON → SPORT ON                    |
|                  | - Sport ON → ECO ON                    | 
|                  | → Bật LED2 / LED3 tương ứng            |
|                  | → Gọi UART "mode ECO"/"mode SPORT"     |
| Nhấn giữ ≥ 3s    |	Toggle LED1 (Lighting)                |
|                  | → Gọi UART "on light"/"off light"      |

Lưu ý:
* Override AUTO LIGHTING khi nhấn giữ 3s
* Single click không ảnh hưởng LIGHT_AUTO.

### 16.2 Event & Timer Handling
* External Interrupt phát hiện cạnh nhấn/nhã
* Timer đo thời gian giữ (3s threshold)
* Service Button phân loại:
* - < 3s → EVT_MODE_SINGLE
* - ≥ 3s → EVT_MODE_HOLD_3S
* Debounce 10–50ms để tránh nhiễu

### 16.3 Tier Responsibilities
|Tier	        | Vai trò Mode Button                      |
|-------------|------------------------------------------|
|Driver	      | External Interrupt, đọc GPIO             |
|Service	    | Debounce, hold detection, set Event      |
|Application	| FSM xử lý:                               |
|             | - EVT_MODE_SINGLE → toggle ECO/SPORT     |
|             | - EVT_MODE_HOLD_3S → toggle LIGHT + UART |
### 16.4 FSM Integration (Mode Button)

* Khi SYS_ON:
```
EVT_MODE_SINGLE → Kiểm tra mode_state:
  Nếu MODE_ECO → chuyển MODE_SPORT → LED3 ON, LED2 OFF → UART "mode SPORT"
  Nếu MODE_SPORT → chuyển MODE_ECO → LED2 ON, LED3 OFF → UART "mode ECO"
EVT_MODE_HOLD_3S → Kiểm tra lighting_state:
  Nếu LIGHT_OFF / LIGHT_AUTO → LIGHT_ON → LED1 ON → UART "on light"
  Nếu LIGHT_ON → LIGHT_OFF → LED1 OFF → UART "off light"
```
* Tóm tắt FSM
```
State: SYS_ON
Event: EVT_MODE_SINGLE
  Action: Toggle MODE_ECO / MODE_SPORT
Event: EVT_MODE_HOLD_3S
  Action: Toggle LIGHT
```
---
**END OF FIRMWARE DESIGN SPECIFICATION**
