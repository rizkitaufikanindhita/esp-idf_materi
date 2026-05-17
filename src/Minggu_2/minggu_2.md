# Minggu 2 — ESP-IDF FreeRTOS Multitasking dan Task Management

## Target Minggu 2

Setelah menyelesaikan Minggu 2, kamu diharapkan mampu:

1. Memahami konsep multitasking di ESP-IDF.
2. Memahami hubungan ESP-IDF dengan FreeRTOS.
3. Membuat beberapa task menggunakan `xTaskCreate()`.
4. Menggunakan `vTaskDelay()` dengan benar.
5. Memahami task priority.
6. Memahami task handle.
7. Menggunakan `vTaskSuspend()` dan `vTaskResume()`.
8. Membagi program menjadi beberapa task yang rapi.
9. Membuat project praktik: LED task + button task.

Minggu 1 kamu sudah belajar struktur project, GPIO, delay, dan logging. Minggu 2 mulai masuk ke inti firmware ESP-IDF: **FreeRTOS task**.

---

## 1. Konsep Dasar FreeRTOS di ESP-IDF

ESP-IDF berjalan di atas FreeRTOS. Artinya, program tidak harus ditulis sebagai satu loop besar seperti gaya Arduino.

Di Arduino, biasanya pola program seperti ini:

```cpp
void loop() {
    baca_tombol();
    kedip_led();
    baca_sensor();
    kirim_data();
}
```

Di ESP-IDF, kamu bisa memecah pekerjaan menjadi beberapa task:

```txt
LED Task       → mengatur LED
Button Task    → membaca tombol
Sensor Task    → membaca sensor
Network Task   → mengirim data
Display Task   → update LCD
```

Keuntungan multitasking:

1. Kode lebih rapi.
2. Setiap task punya tanggung jawab jelas.
3. Program lebih mudah dikembangkan.
4. Cocok untuk firmware IoT yang kompleks.
5. Cocok untuk project besar seperti alat diagnosis stunting, drone, sensor node, atau gateway IoT.

---

## 2. Apa Itu Task?

Task adalah fungsi yang berjalan seperti proses kecil di dalam firmware.

Bentuk task FreeRTOS:

```c
static void my_task(void *pvParameters)
{
    while (1) {
        // pekerjaan task
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

Ciri penting task:

1. Bentuknya fungsi `void`.
2. Parameternya `void *pvParameters`.
3. Biasanya berisi `while (1)`.
4. Harus punya delay, blocking call, atau yield.
5. Jangan membuat task loop kosong tanpa delay.

Contoh buruk:

```c
static void bad_task(void *pvParameters)
{
    while (1) {
        // tidak ada delay
    }
}
```

Contoh baik:

```c
static void good_task(void *pvParameters)
{
    while (1) {
        ESP_LOGI("TASK", "Running");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 3. Membuat Task dengan `xTaskCreate()`

Format umum:

```c
xTaskCreate(
    task_function,
    "Task Name",
    stack_size,
    parameter,
    priority,
    task_handle
);
```

Penjelasan parameter:

| Parameter | Fungsi |
|---|---|
| `task_function` | Fungsi task yang akan dijalankan |
| `"Task Name"` | Nama task untuk debugging |
| `stack_size` | Ukuran stack task dalam byte |
| `parameter` | Data yang dikirim ke task |
| `priority` | Prioritas task |
| `task_handle` | Handle untuk mengontrol task |

Contoh:

```c
xTaskCreate(
    led_task,
    "LED Task",
    2048,
    NULL,
    2,
    NULL
);
```

Artinya:

```txt
Buat task led_task
Nama task: LED Task
Stack: 2048 byte
Parameter: tidak ada
Priority: 2
Handle: tidak disimpan
```

---

## 4. `vTaskDelay()`

`vTaskDelay()` digunakan untuk membuat task tidur sementara.

Contoh:

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

Artinya task tidur selama 1000 ms atau 1 detik.

Kenapa pakai `pdMS_TO_TICKS()`?

FreeRTOS bekerja dalam satuan tick, bukan langsung milidetik. Macro `pdMS_TO_TICKS()` mengubah milidetik menjadi tick.

Buruk:

```c
vTaskDelay(1000);
```

Lebih benar:

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

---

## 5. Perbedaan `vTaskDelay()` dan Delay Biasa

Di ESP-IDF, jangan berpikir delay sebagai menghentikan seluruh program.

Saat satu task menjalankan:

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

Task tersebut tidur, tetapi task lain tetap bisa berjalan.

Contoh:

```txt
LED Task tidur 1000 ms
Button Task tetap membaca tombol
Network Task tetap bisa berjalan
```

Ini inti multitasking.

---

## 6. Project 1 — Dua Task LED Berbeda

### Tujuan

Membuat dua task:

1. Task pertama print log setiap 1 detik.
2. Task kedua print log setiap 2 detik.

### Struktur Project

```txt
week2_01_two_tasks/
├── CMakeLists.txt
└── main/
    ├── CMakeLists.txt
    └── main.c
```

### `main.c`

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

static const char *TAG_TASK1 = "TASK_1";
static const char *TAG_TASK2 = "TASK_2";

static void task_1(void *pvParameters)
{
    while (1) {
        ESP_LOGI(TAG_TASK1, "Task 1 running every 1 second");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

static void task_2(void *pvParameters)
{
    while (1) {
        ESP_LOGI(TAG_TASK2, "Task 2 running every 2 seconds");
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

void app_main(void)
{
    ESP_LOGI("MAIN", "Week 2 two tasks example started");

    xTaskCreate(task_1, "Task 1", 2048, NULL, 2, NULL);
    xTaskCreate(task_2, "Task 2", 2048, NULL, 2, NULL);
}
```

### Output yang Diharapkan

```txt
I (xxx) MAIN: Week 2 two tasks example started
I (xxx) TASK_1: Task 1 running every 1 second
I (xxx) TASK_2: Task 2 running every 2 seconds
I (xxx) TASK_1: Task 1 running every 1 second
I (xxx) TASK_1: Task 1 running every 1 second
I (xxx) TASK_2: Task 2 running every 2 seconds
```

---

## 7. Task Priority

Setiap task punya prioritas.

Contoh:

```c
xTaskCreate(sensor_task, "Sensor", 4096, NULL, 3, NULL);
xTaskCreate(display_task, "Display", 4096, NULL, 2, NULL);
xTaskCreate(logger_task, "Logger", 4096, NULL, 1, NULL);
```

Artinya:

```txt
Sensor Task priority 3
Display Task priority 2
Logger Task priority 1
```

Task dengan priority lebih tinggi akan lebih diprioritaskan oleh scheduler.

Namun jangan asal memberi priority tinggi. Jika task priority tinggi tidak pernah delay/blocking, task lain bisa kelaparan.

Contoh buruk:

```c
static void high_priority_bad_task(void *pvParameters)
{
    while (1) {
        // kerja terus tanpa delay
    }
}
```

Ini bisa membuat task lain tidak berjalan dan memicu watchdog.

---

## 8. Panduan Priority Awal

Untuk latihan:

| Jenis Task | Priority Awal |
|---|---|
| Monitor/log ringan | 1 |
| Display task | 1–2 |
| Button/input task | 2–3 |
| Sensor task | 2–3 |
| Network task | 3–5 |
| Critical control task | 4–6 |

Untuk pemula, gunakan priority sederhana dulu:

```txt
Monitor Task : 1
LED Task     : 2
Button Task  : 3
```

---

## 9. Stack Size Task

Stack adalah memory kerja internal task.

Contoh:

```c
xTaskCreate(network_task, "Network", 8192, NULL, 4, NULL);
```

`8192` adalah ukuran stack dalam byte.

Task sederhana bisa memakai 2048–3072 byte. Task yang memakai JSON, HTTP, TLS, atau file system butuh lebih besar.

Panduan awal:

| Task | Stack Awal |
|---|---|
| LED blink | 2048 |
| Button polling | 2048 |
| Sensor I2C sederhana | 3072–4096 |
| Display task | 3072–4096 |
| File system task | 4096 |
| JSON task | 4096–6144 |
| HTTP task | 6144–8192 |
| HTTPS/TLS task | 8192+ |

---

## 10. Mengecek Sisa Stack

Gunakan:

```c
uxTaskGetStackHighWaterMark(NULL)
```

Contoh:

```c
static void monitor_task(void *pvParameters)
{
    while (1) {
        UBaseType_t stack_left = uxTaskGetStackHighWaterMark(NULL);
        ESP_LOGI("MONITOR", "Stack left: %u", stack_left);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

Kalau sisa stack terlalu kecil, task rawan crash.

---

## 11. Project 2 — LED Task

### Tujuan

Membuat LED berkedip di dalam task.

### Wiring

Untuk banyak board ESP32, LED onboard sering di GPIO2. Sesuaikan dengan board kamu.

```c
#define LED_PIN GPIO_NUM_2
```

### Kode

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_PIN GPIO_NUM_2

static const char *TAG = "LED_TASK";

static void led_task(void *pvParameters)
{
    bool led_state = false;

    while (1) {
        led_state = !led_state;
        gpio_set_level(LED_PIN, led_state ? 1 : 0);

        ESP_LOGI(TAG, "LED %s", led_state ? "ON" : "OFF");

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void app_main(void)
{
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_PIN, 0);

    xTaskCreate(
        led_task,
        "LED Task",
        2048,
        NULL,
        2,
        NULL
    );
}
```

---

## 12. Task Handle

Task handle digunakan untuk mengontrol task dari task lain.

Contoh:

```c
static TaskHandle_t led_task_handle = NULL;
```

Saat membuat task:

```c
xTaskCreate(
    led_task,
    "LED Task",
    2048,
    NULL,
    2,
    &led_task_handle
);
```

Sekarang task lain bisa mengontrol LED Task menggunakan handle tersebut.

---

## 13. Suspend dan Resume Task

### Suspend

```c
vTaskSuspend(led_task_handle);
```

Task berhenti sementara.

### Resume

```c
vTaskResume(led_task_handle);
```

Task berjalan lagi.

Catatan:

1. Jangan suspend task sembarangan di firmware production.
2. Untuk komunikasi antar-task, nanti lebih baik pakai queue, semaphore, event group, atau task notification.
3. Suspend/resume bagus untuk latihan memahami task handle.

---

## 14. Project 3 — Button Mengontrol LED Task

### Target

1. LED Task berkedip setiap 500 ms.
2. Button Task membaca tombol.
3. Jika tombol ditekan, LED Task di-suspend atau di-resume.

### Wiring Tombol

Gunakan active-low:

```txt
GPIO button ---- tombol ---- GND
GPIO memakai pull-up internal
```

Saat tidak ditekan:

```txt
GPIO = HIGH
```

Saat ditekan:

```txt
GPIO = LOW
```

### Kode Lengkap

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "driver/gpio.h"

#define LED_PIN     GPIO_NUM_2
#define BUTTON_PIN  GPIO_NUM_0

static const char *TAG = "MAIN";
static const char *TAG_LED = "LED_TASK";
static const char *TAG_BUTTON = "BUTTON_TASK";

static TaskHandle_t led_task_handle = NULL;
static bool led_task_active = true;

static void gpio_init_all(void)
{
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_PIN, 0);

    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_PIN, GPIO_PULLUP_ONLY);
}

static void led_task(void *pvParameters)
{
    bool led_state = false;

    while (1) {
        led_state = !led_state;
        gpio_set_level(LED_PIN, led_state ? 1 : 0);

        ESP_LOGI(TAG_LED, "LED %s", led_state ? "ON" : "OFF");

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

static void button_task(void *pvParameters)
{
    int last_button_state = 1;

    while (1) {
        int current_button_state = gpio_get_level(BUTTON_PIN);

        if (last_button_state == 1 && current_button_state == 0) {
            vTaskDelay(pdMS_TO_TICKS(50));

            if (gpio_get_level(BUTTON_PIN) == 0) {
                if (led_task_active) {
                    ESP_LOGI(TAG_BUTTON, "Button pressed: suspend LED task");
                    vTaskSuspend(led_task_handle);
                    gpio_set_level(LED_PIN, 0);
                    led_task_active = false;
                } else {
                    ESP_LOGI(TAG_BUTTON, "Button pressed: resume LED task");
                    vTaskResume(led_task_handle);
                    led_task_active = true;
                }

                while (gpio_get_level(BUTTON_PIN) == 0) {
                    vTaskDelay(pdMS_TO_TICKS(10));
                }
            }
        }

        last_button_state = current_button_state;
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Week 2 Button + LED Task example started");

    gpio_init_all();

    xTaskCreate(
        led_task,
        "LED Task",
        2048,
        NULL,
        2,
        &led_task_handle
    );

    xTaskCreate(
        button_task,
        "Button Task",
        2048,
        NULL,
        3,
        NULL
    );
}
```

---

## 15. Penjelasan Project Button + LED

### `led_task_handle`

```c
static TaskHandle_t led_task_handle = NULL;
```

Dipakai agar Button Task bisa mengontrol LED Task.

---

### `led_task_active`

```c
static bool led_task_active = true;
```

Dipakai untuk menyimpan status apakah LED Task sedang aktif atau suspend.

---

### Debounce Sederhana

```c
vTaskDelay(pdMS_TO_TICKS(50));
```

Tombol mekanik bisa bouncing. Jadi setelah terdeteksi ditekan, tunggu 50 ms lalu cek ulang.

---

### Menunggu Tombol Dilepas

```c
while (gpio_get_level(BUTTON_PIN) == 0) {
    vTaskDelay(pdMS_TO_TICKS(10));
}
```

Tujuannya agar satu tekan tidak terbaca berkali-kali.

---

## 16. Common Mistakes Minggu 2

### 16.1 Task Tidak Punya Delay

Buruk:

```c
while (1) {
    gpio_set_level(LED_PIN, 1);
}
```

Akibat:

```txt
CPU penuh
Task lain terganggu
Watchdog bisa trigger
```

---

### 16.2 Stack Terlalu Kecil

Buruk:

```c
xTaskCreate(http_task, "HTTP", 2048, NULL, 3, NULL);
```

HTTP/HTTPS biasanya butuh stack lebih besar.

---

### 16.3 Terlalu Banyak Global Variable

Awal belajar boleh, tapi di project besar nanti harus dikontrol dengan queue, mutex, dan event.

---

### 16.4 Priority Terlalu Tinggi Semua

Buruk:

```c
xTaskCreate(task1, "Task1", 2048, NULL, 10, NULL);
xTaskCreate(task2, "Task2", 2048, NULL, 10, NULL);
xTaskCreate(task3, "Task3", 2048, NULL, 10, NULL);
```

Untuk latihan, cukup priority 1–4.

---

## 17. Latihan Minggu 2

### Latihan 1 — Dua Task Log

Buat dua task:

```txt
Task A print setiap 1 detik
Task B print setiap 3 detik
```

---

### Latihan 2 — Tiga LED Pattern

Buat tiga task:

```txt
Task 1: LED blink cepat
Task 2: print heartbeat setiap 2 detik
Task 3: print uptime setiap 5 detik
```

---

### Latihan 3 — Button Polling Task

Buat Button Task yang membaca tombol setiap 10 ms dan print saat tombol ditekan.

---

### Latihan 4 — Suspend/Resume LED Task

Gunakan tombol untuk suspend/resume LED Task.

---

### Latihan 5 — Stack Monitor

Tambahkan Monitor Task:

```c
uxTaskGetStackHighWaterMark(NULL)
```

Print sisa stack setiap 2 detik.

---

### Latihan 6 — Priority Experiment

Buat dua task dengan priority berbeda. Amati log.

Jangan buat infinite loop tanpa delay kecuali hanya untuk eksperimen singkat memahami masalah watchdog.

---

## 18. Tugas Akhir Minggu 2

Buat project:

```txt
week2_final_task_management
```

Fitur wajib:

```txt
1. LED Task berkedip setiap 500 ms.
2. Button Task membaca tombol active-low.
3. Satu kali tekan tombol suspend/resume LED Task.
4. Monitor Task print setiap 2 detik:
   - status LED Task aktif/tidak
   - free heap
   - stack high water mark Monitor Task
5. Semua task menggunakan vTaskDelay().
6. Tidak ada while(1) kosong tanpa delay.
7. Gunakan ESP_LOGI dengan TAG berbeda.
```

Contoh TAG:

```c
static const char *TAG_MAIN = "MAIN";
static const char *TAG_LED = "LED_TASK";
static const char *TAG_BUTTON = "BUTTON_TASK";
static const char *TAG_MONITOR = "MONITOR_TASK";
```

---

## 19. Checklist Lulus Minggu 2

Kamu dianggap lulus Minggu 2 kalau bisa:

```txt
[ ] Menjelaskan apa itu FreeRTOS task
[ ] Membuat task dengan xTaskCreate()
[ ] Menjelaskan parameter xTaskCreate()
[ ] Menggunakan vTaskDelay(pdMS_TO_TICKS())
[ ] Menjelaskan kenapa task harus delay/blocking
[ ] Membuat dua task berjalan bersamaan
[ ] Menjelaskan task priority
[ ] Menjelaskan stack size task
[ ] Menggunakan TaskHandle_t
[ ] Menggunakan vTaskSuspend()
[ ] Menggunakan vTaskResume()
[ ] Membuat Button Task mengontrol LED Task
[ ] Melakukan debounce sederhana pada tombol
[ ] Mengecek stack dengan uxTaskGetStackHighWaterMark()
```

---

## 20. Inti Pemahaman Minggu 2

Kalimat penting:

> Di ESP-IDF, program besar tidak ditulis sebagai satu loop utama, tetapi dipecah menjadi beberapa task yang masing-masing punya tanggung jawab.

Pola yang harus mulai kamu biasakan:

```txt
Task input     → membaca tombol/input
Task output    → mengatur LED/display
Task sensor    → membaca sensor
Task network   → mengirim data
Task monitor   → memantau sistem
```

Minggu 2 adalah fondasi untuk Minggu 3, yaitu:

```txt
Inter-task Communication:
Queue, Semaphore, Mutex, Event Group, dan Task Notification
```

