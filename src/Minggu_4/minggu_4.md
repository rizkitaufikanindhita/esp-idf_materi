# Minggu 4 — ESP-IDF Interrupt, Timer, dan Watchdog

## Target Pembelajaran

Pada Minggu 4, kamu belajar membuat firmware yang lebih responsif dan robust dengan menggunakan:

1. GPIO interrupt.
2. ISR atau Interrupt Service Routine.
3. Task notification dari ISR ke task.
4. Queue dari ISR ke task.
5. FreeRTOS software timer.
6. `esp_timer` untuk timer resolusi mikrodetik.
7. Watchdog awareness.
8. Mini project interrupt + timer + queue + monitor task.

Setelah menyelesaikan minggu ini, kamu harus paham pola berikut:

```txt
Event hardware terjadi
    ↓
ISR menangkap event
    ↓
ISR memberi sinyal ke task
    ↓
Task memproses logic aplikasi
```

Prinsip penting minggu ini:

> ISR bukan tempat memproses logic berat. ISR hanya memberi sinyal ke task.

---

## Prasyarat

Sebelum masuk Minggu 4, kamu sebaiknya sudah memahami:

1. Struktur project ESP-IDF.
2. GPIO input/output.
3. `vTaskDelay()`.
4. `xTaskCreate()`.
5. Queue dasar.
6. Semaphore dan mutex dasar.
7. Event group dasar.
8. Task notification dasar.

---

## Gambaran Minggu 4

| Hari | Materi | Output |
|---|---|---|
| Hari 1 | Polling vs interrupt | Paham kapan memakai polling dan interrupt |
| Hari 2 | GPIO interrupt dasar | Tombol memicu ISR |
| Hari 3 | ISR ke task notification | Tombol interrupt membangunkan task |
| Hari 4 | ISR ke queue | ISR mengirim event ke task |
| Hari 5 | FreeRTOS software timer | LED heartbeat dengan timer |
| Hari 6 | `esp_timer` dan watchdog | Timer resolusi mikrodetik dan watchdog awareness |
| Hari 7 | Mini project | Button interrupt + LED + timer + monitor |

---

# 1. Polling vs Interrupt

## 1.1 Polling

Polling berarti task mengecek kondisi berulang-ulang.

Contoh polling tombol:

```c
while (1) {
    if (gpio_get_level(BUTTON_GPIO) == 0) {
        ESP_LOGI(TAG, "Button pressed");
    }

    vTaskDelay(pdMS_TO_TICKS(10));
}
```

Kelebihan polling:

1. Sederhana.
2. Mudah dipahami.
3. Cocok untuk tombol sederhana.
4. Cocok untuk event lambat.

Kekurangan polling:

1. Task harus terus mengecek.
2. Event cepat bisa terlewat.
3. Kurang efisien.
4. Respons tergantung interval polling.

---

## 1.2 Interrupt

Interrupt berarti hardware memberi tahu CPU ketika event terjadi.

Contoh:

```txt
Tombol ditekan
    ↓
GPIO berubah HIGH ke LOW
    ↓
Interrupt terjadi
    ↓
ISR dipanggil
```

Kelebihan interrupt:

1. Respons lebih cepat.
2. Tidak perlu polling terus-menerus.
3. Cocok untuk event hardware.
4. Cocok untuk sensor data-ready, tombol, encoder, dan pulse input.

Kekurangan interrupt:

1. Lebih rawan bug jika ISR terlalu berat.
2. Perlu paham API `FromISR`.
3. Perlu hati-hati dengan debounce.
4. Perlu hati-hati dengan shared data.

---

# 2. ISR

ISR adalah **Interrupt Service Routine**, yaitu fungsi yang dijalankan saat interrupt terjadi.

## 2.1 Aturan Emas ISR

ISR harus:

1. Singkat.
2. Cepat selesai.
3. Tidak blocking.
4. Tidak memanggil `vTaskDelay()`.
5. Tidak melakukan HTTP, MQTT, SPIFFS, I2C/SPI berat.
6. Tidak melakukan parsing JSON.
7. Tidak melakukan logging berlebihan.
8. Menggunakan API khusus `FromISR` jika berinteraksi dengan FreeRTOS.

Pola buruk:

```c
static void IRAM_ATTR button_isr_handler(void *arg)
{
    ESP_LOGI(TAG, "Button pressed");
    vTaskDelay(pdMS_TO_TICKS(50));
    send_http_request();
}
```

Pola benar:

```c
static void IRAM_ATTR button_isr_handler(void *arg)
{
    BaseType_t higher_priority_task_woken = pdFALSE;

    vTaskNotifyGiveFromISR(button_task_handle, &higher_priority_task_woken);

    if (higher_priority_task_woken == pdTRUE) {
        portYIELD_FROM_ISR();
    }
}
```

---

# 3. GPIO Interrupt Dasar

## 3.1 Jenis Trigger Interrupt

| Mode | Arti |
|---|---|
| `GPIO_INTR_POSEDGE` | Interrupt saat LOW ke HIGH |
| `GPIO_INTR_NEGEDGE` | Interrupt saat HIGH ke LOW |
| `GPIO_INTR_ANYEDGE` | Interrupt saat berubah naik atau turun |
| `GPIO_INTR_LOW_LEVEL` | Interrupt saat level LOW |
| `GPIO_INTR_HIGH_LEVEL` | Interrupt saat level HIGH |

Untuk tombol active-low:

```txt
Tidak ditekan = HIGH
Ditekan       = LOW
```

Maka saat tombol ditekan terjadi perubahan:

```txt
HIGH → LOW
```

Gunakan:

```c
GPIO_INTR_NEGEDGE
```

---

## 3.2 Contoh GPIO Interrupt dengan Flag

Project:

```txt
week4_01_gpio_interrupt_flag
```

Kode:

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define BUTTON_GPIO GPIO_NUM_0

static const char *TAG = "GPIO_INT";

static volatile bool button_interrupt_flag = false;

static void IRAM_ATTR button_isr_handler(void *arg)
{
    button_interrupt_flag = true;
}

static void button_gpio_init(void)
{
    gpio_config_t button_config = {
        .pin_bit_mask = (1ULL << BUTTON_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };

    ESP_ERROR_CHECK(gpio_config(&button_config));

    ESP_ERROR_CHECK(gpio_install_isr_service(0));
    ESP_ERROR_CHECK(gpio_isr_handler_add(BUTTON_GPIO, button_isr_handler, NULL));
}

void app_main(void)
{
    ESP_LOGI(TAG, "GPIO interrupt flag example started");

    button_gpio_init();

    while (1) {
        if (button_interrupt_flag) {
            button_interrupt_flag = false;
            ESP_LOGI(TAG, "Button interrupt detected");
        }

        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

## 3.3 Penjelasan

```c
static volatile bool button_interrupt_flag = false;
```

`volatile` dipakai karena variable diubah oleh ISR dan dibaca oleh task/main loop.

```c
static void IRAM_ATTR button_isr_handler(void *arg)
```

`IRAM_ATTR` umum dipakai untuk ISR agar fungsi ditempatkan di IRAM.

```c
gpio_install_isr_service(0);
gpio_isr_handler_add(BUTTON_GPIO, button_isr_handler, NULL);
```

Dua fungsi ini mengaktifkan ISR service dan mendaftarkan handler untuk GPIO tertentu.

Catatan:

Contoh flag cocok untuk pengenalan. Untuk project lebih serius, gunakan task notification atau queue.

---

# 4. ISR ke Task Notification

Task notification cocok jika ISR hanya perlu membangunkan satu task.

Pola:

```txt
GPIO interrupt
    ↓
ISR
    ↓
vTaskNotifyGiveFromISR()
    ↓
Button Task bangun
    ↓
Debounce dan proses logic
```

---

## 4.1 Contoh Button Interrupt Toggle LED

Project:

```txt
week4_02_isr_task_notification
```

Kode:

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO     GPIO_NUM_2
#define BUTTON_GPIO  GPIO_NUM_0

static const char *TAG = "ISR_NOTIFY";

static TaskHandle_t button_task_handle = NULL;

static void IRAM_ATTR button_isr_handler(void *arg)
{
    BaseType_t higher_priority_task_woken = pdFALSE;

    vTaskNotifyGiveFromISR(button_task_handle, &higher_priority_task_woken);

    if (higher_priority_task_woken == pdTRUE) {
        portYIELD_FROM_ISR();
    }
}

static void gpio_init_all(void)
{
    gpio_config_t led_config = {
        .pin_bit_mask = (1ULL << LED_GPIO),
        .mode = GPIO_MODE_OUTPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE,
    };

    ESP_ERROR_CHECK(gpio_config(&led_config));
    gpio_set_level(LED_GPIO, 0);

    gpio_config_t button_config = {
        .pin_bit_mask = (1ULL << BUTTON_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };

    ESP_ERROR_CHECK(gpio_config(&button_config));

    ESP_ERROR_CHECK(gpio_install_isr_service(0));
    ESP_ERROR_CHECK(gpio_isr_handler_add(BUTTON_GPIO, button_isr_handler, NULL));
}

static void button_task(void *pvParameters)
{
    bool led_state = false;

    ESP_LOGI("BUTTON_TASK", "Started");

    while (1) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        // Debounce di task, bukan di ISR
        vTaskDelay(pdMS_TO_TICKS(50));

        if (gpio_get_level(BUTTON_GPIO) == 0) {
            led_state = !led_state;
            gpio_set_level(LED_GPIO, led_state ? 1 : 0);

            ESP_LOGI("BUTTON_TASK", "Valid press, LED=%s", led_state ? "ON" : "OFF");

            // Tunggu tombol dilepas agar satu tekan tidak terbaca berkali-kali
            while (gpio_get_level(BUTTON_GPIO) == 0) {
                vTaskDelay(pdMS_TO_TICKS(10));
            }

            ESP_LOGI("BUTTON_TASK", "Button released");
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "ISR to task notification example started");

    BaseType_t ret = xTaskCreate(
        button_task,
        "Button Task",
        2048,
        NULL,
        3,
        &button_task_handle
    );

    if (ret != pdPASS) {
        ESP_LOGE(TAG, "Failed to create button task");
        return;
    }

    gpio_init_all();
}
```

---

## 4.2 Penjelasan Penting

```c
vTaskNotifyGiveFromISR(button_task_handle, &higher_priority_task_woken);
```

Ini adalah API yang aman dipakai dari ISR.

Jangan memakai API biasa di ISR jika ada versi `FromISR`.

```c
ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
```

Task tidur sampai ISR memberi notification.

Ini lebih efisien daripada polling.

---

# 5. ISR ke Queue

Gunakan queue jika ISR perlu mengirim data event.

Contoh data:

```c
typedef struct {
    gpio_num_t gpio_num;
    uint32_t tick;
} gpio_event_t;
```

Pola:

```txt
ISR
    ↓
xQueueSendFromISR()
    ↓
GPIO Event Task
```

---

## 5.1 Contoh ISR Kirim Event ke Queue

Project:

```txt
week4_03_isr_queue
```

Kode:

```c
#include <stdio.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define BUTTON_GPIO GPIO_NUM_0

static const char *TAG = "ISR_QUEUE";

typedef struct {
    gpio_num_t gpio_num;
    uint32_t tick;
} gpio_event_t;

static QueueHandle_t gpio_event_queue = NULL;

static void IRAM_ATTR gpio_isr_handler(void *arg)
{
    gpio_event_t event = {
        .gpio_num = (gpio_num_t)(int)arg,
        .tick = xTaskGetTickCountFromISR(),
    };

    BaseType_t higher_priority_task_woken = pdFALSE;

    xQueueSendFromISR(gpio_event_queue, &event, &higher_priority_task_woken);

    if (higher_priority_task_woken == pdTRUE) {
        portYIELD_FROM_ISR();
    }
}

static void gpio_event_task(void *pvParameters)
{
    gpio_event_t event;

    while (1) {
        if (xQueueReceive(gpio_event_queue, &event, portMAX_DELAY) == pdTRUE) {
            ESP_LOGI("GPIO_TASK",
                     "GPIO interrupt on GPIO %d at tick %lu",
                     event.gpio_num,
                     event.tick);
        }
    }
}

static void gpio_init_button(void)
{
    gpio_config_t button_config = {
        .pin_bit_mask = (1ULL << BUTTON_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };

    ESP_ERROR_CHECK(gpio_config(&button_config));

    ESP_ERROR_CHECK(gpio_install_isr_service(0));
    ESP_ERROR_CHECK(gpio_isr_handler_add(BUTTON_GPIO, gpio_isr_handler, (void *)BUTTON_GPIO));
}

void app_main(void)
{
    ESP_LOGI(TAG, "ISR to queue example started");

    gpio_event_queue = xQueueCreate(10, sizeof(gpio_event_t));

    if (gpio_event_queue == NULL) {
        ESP_LOGE(TAG, "Failed to create queue");
        return;
    }

    xTaskCreate(gpio_event_task, "GPIO Event Task", 2048, NULL, 3, NULL);

    gpio_init_button();
}
```

---

# 6. Debounce Tombol

Mechanical button sering bouncing.

Satu kali tekan bisa menghasilkan beberapa edge:

```txt
HIGH → LOW → HIGH → LOW → LOW
```

Maka perlu debounce.

## 6.1 Debounce Delay di Task

```c
ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
vTaskDelay(pdMS_TO_TICKS(50));

if (gpio_get_level(BUTTON_GPIO) == 0) {
    // valid press
}
```

## 6.2 Debounce Berbasis Timestamp

```c
static int64_t last_press_time_us = 0;

int64_t now_us = esp_timer_get_time();

if ((now_us - last_press_time_us) > 200000) {
    last_press_time_us = now_us;
    // valid event
}
```

Debounce 200 ms cocok untuk tombol menu agar tidak double-trigger.

---

# 7. FreeRTOS Software Timer

Software timer adalah timer yang dikelola FreeRTOS.

Cocok untuk:

1. Timeout.
2. Heartbeat.
3. LED blink periodik.
4. Retry interval.
5. Delay aksi satu kali.

Jenis timer:

| Jenis | Arti |
|---|---|
| One-shot | Jalan sekali |
| Periodic / auto-reload | Jalan berulang |

Header:

```c
#include "freertos/timers.h"
```

API utama:

```c
xTimerCreate()
xTimerStart()
xTimerStop()
xTimerReset()
xTimerDelete()
```

---

## 7.1 Contoh Periodic Timer Blink LED

Project:

```txt
week4_04_freertos_timer
```

Kode:

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/timers.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2

static const char *TAG = "FTIMER";

static TimerHandle_t led_timer = NULL;
static bool led_state = false;

static void led_timer_callback(TimerHandle_t xTimer)
{
    led_state = !led_state;
    gpio_set_level(LED_GPIO, led_state ? 1 : 0);
}

void app_main(void)
{
    ESP_LOGI(TAG, "FreeRTOS software timer example started");

    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    led_timer = xTimerCreate(
        "LED Timer",
        pdMS_TO_TICKS(500),
        pdTRUE,
        NULL,
        led_timer_callback
    );

    if (led_timer == NULL) {
        ESP_LOGE(TAG, "Failed to create LED timer");
        return;
    }

    if (xTimerStart(led_timer, 0) != pdPASS) {
        ESP_LOGE(TAG, "Failed to start LED timer");
        return;
    }

    ESP_LOGI(TAG, "LED timer started");
}
```

---

## 7.2 Catatan Timer Callback

Timer callback jangan dipakai untuk kerja berat.

Buruk:

```c
static void timer_callback(TimerHandle_t xTimer)
{
    http_post();
    write_file();
    vTaskDelay(pdMS_TO_TICKS(100));
}
```

Baik:

```c
static void timer_callback(TimerHandle_t xTimer)
{
    app_event_t event = APP_EVENT_HEARTBEAT;
    xQueueSend(app_queue, &event, 0);
}
```

---

# 8. `esp_timer`

`esp_timer` adalah timer resolusi tinggi dari ESP-IDF.

Cocok untuk:

1. Timestamp mikrodetik.
2. Debounce berbasis waktu.
3. Pengukuran durasi.
4. Timer presisi lebih tinggi daripada FreeRTOS timer.

Header:

```c
#include "esp_timer.h"
```

API umum:

```c
esp_timer_create()
esp_timer_start_once()
esp_timer_start_periodic()
esp_timer_stop()
esp_timer_delete()
esp_timer_get_time()
```

---

## 8.1 Contoh `esp_timer_get_time()`

```c
int64_t now_us = esp_timer_get_time();
ESP_LOGI(TAG, "Time since boot: %lld us", now_us);
```

## 8.2 Contoh Periodic `esp_timer`

Project:

```txt
week4_05_esp_timer
```

Kode:

```c
#include <stdio.h>
#include <stdbool.h>

#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_timer.h"

#define LED_GPIO GPIO_NUM_2

static const char *TAG = "ESP_TIMER";

static esp_timer_handle_t periodic_timer;
static bool led_state = false;

static void periodic_timer_callback(void *arg)
{
    led_state = !led_state;
    gpio_set_level(LED_GPIO, led_state ? 1 : 0);
}

void app_main(void)
{
    ESP_LOGI(TAG, "esp_timer example started");

    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    const esp_timer_create_args_t periodic_timer_args = {
        .callback = &periodic_timer_callback,
        .arg = NULL,
        .name = "periodic_led_timer",
    };

    ESP_ERROR_CHECK(esp_timer_create(&periodic_timer_args, &periodic_timer));
    ESP_ERROR_CHECK(esp_timer_start_periodic(periodic_timer, 500000));

    ESP_LOGI(TAG, "Periodic esp_timer started every 500 ms");
}
```

---

# 9. FreeRTOS Timer vs `esp_timer`

| Kebutuhan | Pilihan |
|---|---|
| Blink LED tiap 500 ms | FreeRTOS timer atau task delay |
| Heartbeat 1 detik | FreeRTOS timer |
| Retry network tiap 30 detik | FreeRTOS timer |
| Timestamp mikrodetik | `esp_timer_get_time()` |
| Mengukur durasi tekan tombol | `esp_timer_get_time()` |
| Timer presisi mikrodetik | `esp_timer` |
| Sampling sangat cepat | Peripheral hardware, bukan callback berat |

---

# 10. Watchdog Awareness

Watchdog mendeteksi firmware yang macet.

Penyebab umum watchdog:

1. `while(1)` tanpa delay.
2. Task priority tinggi tidak pernah blocking.
3. ISR terlalu lama.
4. Timer callback terlalu berat.
5. Mutex deadlock.
6. Critical section terlalu lama.
7. Loop menunggu hardware tanpa timeout.

Contoh buruk:

```c
static void bad_task(void *pvParameters)
{
    while (1) {
        // Tidak ada delay, tidak ada blocking
    }
}
```

Contoh baik:

```c
static void good_task(void *pvParameters)
{
    while (1) {
        // kerja singkat
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

Contoh event-driven yang baik:

```c
static void worker_task(void *pvParameters)
{
    app_event_t event;

    while (1) {
        if (xQueueReceive(app_queue, &event, portMAX_DELAY) == pdTRUE) {
            process_event(&event);
        }
    }
}
```

Task ini aman karena blocking saat tidak ada event.

---

# 11. Mini Project Minggu 4

Project:

```txt
week4_final_interrupt_timer_watchdog_safe
```

## 11.1 Fitur

1. Tombol menggunakan GPIO interrupt.
2. ISR memberi notification ke Button Task.
3. Button Task melakukan debounce.
4. Button Task mengirim event ke queue.
5. LED Task menerima event dan toggle LED.
6. FreeRTOS timer membuat heartbeat tiap 1 detik.
7. Monitor Task print status tiap 2 detik.
8. Semua task watchdog-friendly.
9. Shared status dilindungi mutex.

Arsitektur:

```txt
Button GPIO
    ↓ interrupt
ISR
    ↓ task notification
Button Task
    ↓ queue
LED Task
    ↓ update LED/status

FreeRTOS Timer
    ↓ queue
LED Task / Monitor Status

Monitor Task
    ↓ print status
```

---

## 11.2 Full Code Mini Project

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"
#include "freertos/timers.h"

#include "driver/gpio.h"

#include "esp_log.h"
#include "esp_heap_caps.h"
#include "esp_timer.h"

#define LED_GPIO        GPIO_NUM_2
#define BUTTON_GPIO     GPIO_NUM_0

#define BUTTON_TASK_STACK   2048
#define LED_TASK_STACK      2048
#define MONITOR_TASK_STACK  3072

#define BUTTON_TASK_PRIORITY    4
#define LED_TASK_PRIORITY       3
#define MONITOR_TASK_PRIORITY   1

typedef enum {
    APP_EVENT_BUTTON_PRESSED,
    APP_EVENT_HEARTBEAT,
} app_event_type_t;

typedef struct {
    app_event_type_t type;
    int64_t time_us;
} app_event_t;

typedef struct {
    bool led_state;
    uint32_t button_press_count;
    uint32_t heartbeat_count;
    int64_t last_button_time_us;
} app_status_t;

static const char *TAG = "WEEK4_APP";

static TaskHandle_t button_task_handle = NULL;
static QueueHandle_t app_event_queue = NULL;
static SemaphoreHandle_t status_mutex = NULL;
static TimerHandle_t heartbeat_timer = NULL;

static app_status_t app_status = {
    .led_state = false,
    .button_press_count = 0,
    .heartbeat_count = 0,
    .last_button_time_us = 0,
};

static void IRAM_ATTR button_isr_handler(void *arg)
{
    BaseType_t higher_priority_task_woken = pdFALSE;

    vTaskNotifyGiveFromISR(button_task_handle, &higher_priority_task_woken);

    if (higher_priority_task_woken == pdTRUE) {
        portYIELD_FROM_ISR();
    }
}

static void heartbeat_timer_callback(TimerHandle_t xTimer)
{
    app_event_t event = {
        .type = APP_EVENT_HEARTBEAT,
        .time_us = esp_timer_get_time(),
    };

    xQueueSend(app_event_queue, &event, 0);
}

static void gpio_init_all(void)
{
    gpio_config_t led_config = {
        .pin_bit_mask = (1ULL << LED_GPIO),
        .mode = GPIO_MODE_OUTPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE,
    };

    ESP_ERROR_CHECK(gpio_config(&led_config));
    gpio_set_level(LED_GPIO, 0);

    gpio_config_t button_config = {
        .pin_bit_mask = (1ULL << BUTTON_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };

    ESP_ERROR_CHECK(gpio_config(&button_config));

    ESP_ERROR_CHECK(gpio_install_isr_service(0));
    ESP_ERROR_CHECK(gpio_isr_handler_add(BUTTON_GPIO, button_isr_handler, NULL));

    ESP_LOGI(TAG, "GPIO initialized");
}

static void status_update_button_press(int64_t time_us)
{
    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        app_status.button_press_count++;
        app_status.last_button_time_us = time_us;
        xSemaphoreGive(status_mutex);
    }
}

static void status_update_led(bool state)
{
    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        app_status.led_state = state;
        xSemaphoreGive(status_mutex);
    }
}

static void status_update_heartbeat(void)
{
    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        app_status.heartbeat_count++;
        xSemaphoreGive(status_mutex);
    }
}

static bool status_get(app_status_t *out_status)
{
    if (out_status == NULL) {
        return false;
    }

    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        *out_status = app_status;
        xSemaphoreGive(status_mutex);
        return true;
    }

    return false;
}

static void button_task(void *pvParameters)
{
    ESP_LOGI("BUTTON_TASK", "Started");

    while (1) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        vTaskDelay(pdMS_TO_TICKS(50));

        if (gpio_get_level(BUTTON_GPIO) == 0) {
            app_event_t event = {
                .type = APP_EVENT_BUTTON_PRESSED,
                .time_us = esp_timer_get_time(),
            };

            if (xQueueSend(app_event_queue, &event, pdMS_TO_TICKS(100)) == pdTRUE) {
                status_update_button_press(event.time_us);
                ESP_LOGI("BUTTON_TASK", "Valid button press sent");
            } else {
                ESP_LOGW("BUTTON_TASK", "Failed to send button event");
            }

            while (gpio_get_level(BUTTON_GPIO) == 0) {
                vTaskDelay(pdMS_TO_TICKS(10));
            }

            ESP_LOGI("BUTTON_TASK", "Button released");
        }
    }
}

static void led_task(void *pvParameters)
{
    ESP_LOGI("LED_TASK", "Started");

    bool led_state = false;
    app_event_t event;

    while (1) {
        if (xQueueReceive(app_event_queue, &event, portMAX_DELAY) == pdTRUE) {
            switch (event.type) {
                case APP_EVENT_BUTTON_PRESSED:
                    led_state = !led_state;
                    gpio_set_level(LED_GPIO, led_state ? 1 : 0);
                    status_update_led(led_state);

                    ESP_LOGI("LED_TASK",
                             "Button event at %lld us, LED %s",
                             event.time_us,
                             led_state ? "ON" : "OFF");
                    break;

                case APP_EVENT_HEARTBEAT:
                    status_update_heartbeat();
                    break;

                default:
                    ESP_LOGW("LED_TASK", "Unknown event type");
                    break;
            }
        }
    }
}

static void monitor_task(void *pvParameters)
{
    ESP_LOGI("MONITOR_TASK", "Started");

    app_status_t local_status;

    while (1) {
        if (status_get(&local_status)) {
            uint32_t free_heap = heap_caps_get_free_size(MALLOC_CAP_DEFAULT);
            UBaseType_t stack_left = uxTaskGetStackHighWaterMark(NULL);

            ESP_LOGI("MONITOR_TASK",
                     "LED=%s, button_count=%lu, heartbeat=%lu, last_button=%lld us, free_heap=%lu, stack_left=%u",
                     local_status.led_state ? "ON" : "OFF",
                     local_status.button_press_count,
                     local_status.heartbeat_count,
                     local_status.last_button_time_us,
                     free_heap,
                     stack_left);
        } else {
            ESP_LOGW("MONITOR_TASK", "Failed to read status");
        }

        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Week 4 interrupt/timer/watchdog-safe project started");

    app_event_queue = xQueueCreate(10, sizeof(app_event_t));
    if (app_event_queue == NULL) {
        ESP_LOGE(TAG, "Failed to create app event queue");
        return;
    }

    status_mutex = xSemaphoreCreateMutex();
    if (status_mutex == NULL) {
        ESP_LOGE(TAG, "Failed to create status mutex");
        return;
    }

    if (xTaskCreate(button_task,
                    "Button Task",
                    BUTTON_TASK_STACK,
                    NULL,
                    BUTTON_TASK_PRIORITY,
                    &button_task_handle) != pdPASS) {
        ESP_LOGE(TAG, "Failed to create Button Task");
        return;
    }

    if (xTaskCreate(led_task,
                    "LED Task",
                    LED_TASK_STACK,
                    NULL,
                    LED_TASK_PRIORITY,
                    NULL) != pdPASS) {
        ESP_LOGE(TAG, "Failed to create LED Task");
        return;
    }

    if (xTaskCreate(monitor_task,
                    "Monitor Task",
                    MONITOR_TASK_STACK,
                    NULL,
                    MONITOR_TASK_PRIORITY,
                    NULL) != pdPASS) {
        ESP_LOGE(TAG, "Failed to create Monitor Task");
        return;
    }

    gpio_init_all();

    heartbeat_timer = xTimerCreate(
        "Heartbeat Timer",
        pdMS_TO_TICKS(1000),
        pdTRUE,
        NULL,
        heartbeat_timer_callback
    );

    if (heartbeat_timer == NULL) {
        ESP_LOGE(TAG, "Failed to create heartbeat timer");
        return;
    }

    if (xTimerStart(heartbeat_timer, 0) != pdPASS) {
        ESP_LOGE(TAG, "Failed to start heartbeat timer");
        return;
    }

    ESP_LOGI(TAG, "System started successfully");
}
```

---

# 12. Struktur File yang Direkomendasikan

Setelah kode satu file berhasil, pecah menjadi:

```txt
week4_interrupt_timer_watchdog_safe/
├── CMakeLists.txt
└── main/
    ├── CMakeLists.txt
    ├── main.c
    ├── app_gpio.c
    ├── app_gpio.h
    ├── app_events.h
    ├── app_status.c
    ├── app_status.h
    ├── app_tasks.c
    ├── app_tasks.h
    ├── app_timer.c
    └── app_timer.h
```

Pembagian:

| File | Isi |
|---|---|
| `app_gpio.c/h` | GPIO LED, tombol, ISR |
| `app_events.h` | enum dan struct event |
| `app_status.c/h` | shared status dan mutex |
| `app_tasks.c/h` | button task, LED task, monitor task |
| `app_timer.c/h` | heartbeat timer |
| `main.c` | init dan start module |

---

# 13. Error Umum Minggu 4

## 13.1 ISR Tidak Terpanggil

Cek:

1. GPIO salah.
2. Pull-up belum aktif.
3. Salah interrupt type.
4. `gpio_install_isr_service()` belum dipanggil.
5. `gpio_isr_handler_add()` belum dipanggil.
6. Wiring tombol salah.
7. Tombol tidak common ground.

Debug awal:

```c
ESP_LOGI(TAG, "Button level=%d", gpio_get_level(BUTTON_GPIO));
```

Cek dulu dengan polling sebelum interrupt.

---

## 13.2 Crash Saat ISR

Penyebab:

1. Memanggil API yang tidak aman dari ISR.
2. Queue handle masih NULL.
3. Task handle masih NULL.
4. Logging dari ISR.
5. ISR terlalu berat.

Solusi:

1. Buat task/queue sebelum enable interrupt.
2. Gunakan API `FromISR`.
3. Hindari logic berat di ISR.

---

## 13.3 Tombol Sekali Tekan Terbaca Banyak

Penyebab:

```txt
Mechanical bouncing
```

Solusi:

1. Debounce di task.
2. Gunakan timestamp.
3. Tunggu tombol dilepas.
4. Optional: disable interrupt sementara.

---

## 13.4 Timer Callback Membuat Sistem Lambat

Penyebab:

1. Callback melakukan kerja berat.
2. Callback blocking.
3. Callback melakukan file/network.

Solusi:

```txt
Timer callback → send queue/notify task
Task → proses kerja berat
```

---

## 13.5 Watchdog Reset

Cek:

1. Task tanpa delay.
2. ISR terlalu lama.
3. Callback timer berat.
4. Mutex deadlock.
5. Critical section terlalu lama.
6. Loop menunggu hardware tanpa timeout.

---

# 14. Best Practice Minggu 4

1. ISR harus pendek.
2. Jangan debounce di ISR.
3. Jangan logging berat di ISR.
4. Gunakan `FromISR` API.
5. Gunakan task notification jika hanya membangunkan satu task.
6. Gunakan queue jika perlu mengirim data event.
7. Timer callback jangan melakukan kerja berat.
8. Semua task harus blocking atau delay.
9. Gunakan timeout saat menunggu mutex/semaphore.
10. Monitor heap dan stack untuk project yang lebih besar.

---

# 15. Latihan Minggu 4

## Latihan 1 — GPIO Interrupt Flag

Buat tombol active-low dengan interrupt falling edge.

Target:

```txt
Button interrupt detected
```

---

## Latihan 2 — ISR ke Task Notification

Buat tombol interrupt membangunkan Button Task.

Target:

```txt
Valid button press
```

---

## Latihan 3 — Toggle LED dengan Interrupt

Tombol ditekan → LED toggle.

Syarat:

1. ISR tidak toggle LED langsung.
2. Toggle LED dilakukan di task.
3. Debounce di task.

---

## Latihan 4 — ISR ke Queue

ISR mengirim struct event:

```c
typedef struct {
    gpio_num_t gpio_num;
    uint32_t tick;
} gpio_event_t;
```

Task menerima dan print event.

---

## Latihan 5 — FreeRTOS One-Shot Timer

Tombol ditekan:

```txt
LED ON
3 detik kemudian LED OFF
```

Gunakan one-shot timer:

```c
pdFALSE
```

---

## Latihan 6 — FreeRTOS Periodic Timer

Timer periodik 1 detik mengirim heartbeat event ke queue.

Target:

```txt
heartbeat_count bertambah setiap 1 detik
```

---

## Latihan 7 — `esp_timer_get_time()`

Hitung durasi tombol ditekan.

Rumus:

```c
duration_ms = (release_time_us - press_time_us) / 1000;
```

---

## Latihan 8 — Watchdog Awareness

Buat dua task:

Task buruk:

```c
while (1) {
}
```

Task baik:

```c
while (1) {
    vTaskDelay(pdMS_TO_TICKS(10));
}
```

Amati perbedaannya.

---

# 16. Tugas Akhir Minggu 4

Buat project:

```txt
week4_final_interrupt_timer_watchdog
```

Fitur wajib:

1. GPIO tombol memakai interrupt falling edge.
2. ISR tidak melakukan logging.
3. ISR memberi notification ke Button Task.
4. Button Task debounce 50 ms.
5. Button Task mengirim event ke App Queue.
6. LED Task menerima event dan toggle LED.
7. FreeRTOS timer mengirim heartbeat event setiap 1 detik.
8. Monitor Task print setiap 2 detik:
   - LED state.
   - Button press count.
   - Heartbeat count.
   - Free heap.
   - Stack high water mark.
9. Shared status dilindungi mutex.
10. Tidak ada loop tanpa delay/blocking.
11. Semua object dicek return value-nya:
   - Queue.
   - Mutex.
   - Timer.
   - Task.

---

# 17. Checklist Lulus Minggu 4

Kamu dianggap lulus Minggu 4 kalau bisa:

```txt
[ ] Menjelaskan polling vs interrupt
[ ] Menjelaskan apa itu ISR
[ ] Menjelaskan kenapa ISR harus pendek
[ ] Menggunakan gpio_config untuk interrupt
[ ] Menggunakan gpio_install_isr_service()
[ ] Menggunakan gpio_isr_handler_add()
[ ] Menggunakan vTaskNotifyGiveFromISR()
[ ] Menggunakan xQueueSendFromISR()
[ ] Melakukan debounce di task
[ ] Membuat FreeRTOS software timer
[ ] Membuat one-shot timer
[ ] Membuat periodic timer
[ ] Menggunakan esp_timer_get_time()
[ ] Menjelaskan FreeRTOS timer vs esp_timer
[ ] Menjelaskan penyebab watchdog trigger
[ ] Membuat task watchdog-friendly
[ ] Membuat mini project interrupt + timer + queue
```

---

# 18. Penerapan ke Project IoT Stunting

Untuk project diagnosis stunting berbasis ESP32, Minggu 4 berguna untuk:

| Kebutuhan | Mekanisme |
|---|---|
| Tombol menu | GPIO interrupt + debounce task |
| Tombol start pengukuran | GPIO interrupt + task notification |
| Rotary encoder | GPIO interrupt + queue |
| RFID IRQ pin | GPIO interrupt + task notification |
| Heartbeat device | FreeRTOS timer |
| Retry sync offline | FreeRTOS timer |
| Timestamp event | `esp_timer_get_time()` |
| Watchdog-safe firmware | Semua task blocking/delay |

Arsitektur contoh:

```txt
Button ISR
    ↓
Input Task
    ↓ queue
App Task
    ↓
Sensor Task / Network Task / Display Task

Timer
    ↓
Heartbeat Event
    ↓
Monitor Task
```

---

# 19. Ringkasan Inti

Minggu 4 mengajarkan pola firmware event-driven.

Intinya:

```txt
Interrupt cepat
    ↓
ISR pendek
    ↓
Task notification / queue
    ↓
Task memproses logic
    ↓
Status dimonitor
```

Kalimat yang harus diingat:

> ISR hanya menangkap event. Task yang memproses logic aplikasi.

Setelah Minggu 4 kuat, kamu siap masuk Minggu 5:

```txt
UART, I2C, SPI, dan komunikasi dengan peripheral eksternal.
```
