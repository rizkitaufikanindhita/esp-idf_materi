# Minggu 3 — ESP-IDF Inter-Task Communication

## Target Minggu 3

Setelah menyelesaikan Minggu 3, kamu diharapkan mampu memahami dan menggunakan mekanisme komunikasi antar-task di ESP-IDF/FreeRTOS, yaitu:

- Queue
- Binary semaphore
- Counting semaphore
- Mutex
- Event group
- Task notification
- Pola producer-consumer
- Pola event-driven firmware sederhana

Minggu 3 adalah jembatan dari sekadar membuat banyak task menuju firmware yang lebih rapi, aman, dan terstruktur.

Pada Minggu 2, kamu sudah belajar membuat banyak task. Masalahnya, dalam firmware nyata task tidak boleh berjalan sendiri-sendiri tanpa koordinasi.

Contoh kasus:

```txt
Button Task membaca tombol
LED Task mengontrol LED
Sensor Task membaca sensor
Network Task mengirim data
Storage Task menyimpan data offline
```

Semua task itu harus bisa saling memberi informasi.

---

## 1. Kenapa Perlu Inter-Task Communication?

Tanpa komunikasi antar-task, firmware akan kacau.

Contoh buruk:

```c
bool button_pressed = false;
bool sensor_ready = false;
bool wifi_connected = false;
```

Semua task membaca dan menulis global variable secara bebas.

Masalah yang muncul:

```txt
1. Race condition
2. Data berubah saat sedang dibaca
3. Sulit tahu task mana yang mengubah data
4. Sulit debug
5. Firmware tidak scalable
```

Solusi FreeRTOS:

```txt
Queue             → kirim data antar-task
Semaphore         → memberi sinyal atau mengatur resource
Mutex             → melindungi shared resource
Event Group       → menyimpan beberapa status event dalam bit
Task Notification → sinyal ringan langsung ke task tertentu
```

---

## 2. Gambaran Minggu 3

| Hari | Fokus | Output |
|---|---|---|
| Hari 1 | Konsep komunikasi antar-task | Paham masalah shared data |
| Hari 2 | Queue | Bisa kirim data antar-task |
| Hari 3 | Semaphore | Bisa memberi sinyal antar-task |
| Hari 4 | Mutex | Bisa melindungi shared resource |
| Hari 5 | Event Group | Bisa mengelola banyak status event |
| Hari 6 | Task Notification | Bisa memakai sinyal ringan antar-task |
| Hari 7 | Mini Project | Button → Queue → LED + Monitor |

---

## 3. Analogi Sederhana

### Queue

Queue seperti antrean pesan.

```txt
Sensor Task menghasilkan data
↓
Masuk ke queue
↓
Network Task mengambil data
```

Cocok untuk mengirim data.

---

### Semaphore

Semaphore seperti bel/sinyal.

```txt
ISR atau Task memberi sinyal
↓
Task lain bangun dan bekerja
```

Cocok untuk memberi tahu bahwa suatu event terjadi.

---

### Mutex

Mutex seperti kunci ruangan.

```txt
Task A mengambil kunci
↓
Task A memakai resource
↓
Task A mengembalikan kunci
↓
Task B baru boleh memakai resource
```

Cocok untuk melindungi resource yang dipakai bersama.

---

### Event Group

Event group seperti papan status.

```txt
Bit 0 = WiFi connected
Bit 1 = MQTT connected
Bit 2 = Sensor ready
Bit 3 = OTA running
```

Cocok untuk menyimpan banyak status boolean.

---

### Task Notification

Task notification seperti pesan langsung khusus ke satu task.

```txt
ISR → notify Button Task
Timer → notify Worker Task
```

Lebih ringan daripada queue/semaphore jika hanya perlu memberi sinyal ke satu task.

---

## 4. Queue

## 4.1 Apa Itu Queue?

Queue adalah mekanisme FreeRTOS untuk mengirim data dari satu task ke task lain.

Pola umum:

```txt
Producer Task
  ↓ xQueueSend()
Queue
  ↓ xQueueReceive()
Consumer Task
```

Producer adalah task yang menghasilkan data. Consumer adalah task yang memproses data.

---

## 4.2 Kapan Menggunakan Queue?

Gunakan queue jika:

```txt
1. Task perlu mengirim data ke task lain
2. Data harus diproses berurutan
3. Producer dan consumer berjalan dengan timing berbeda
4. Ingin menghindari global variable bebas
```

Contoh:

```txt
Sensor Task → kirim data sensor → Network Task
RFID Task → kirim UID → App Task
Button Task → kirim event tombol → LED Task
Prediction Task → kirim hasil diagnosis → Storage Task
```

---

## 4.3 API Queue Penting

```c
xQueueCreate()
xQueueSend()
xQueueReceive()
xQueueSendFromISR()
xQueueReset()
uxQueueMessagesWaiting()
```

---

## 4.4 Contoh Queue Sederhana

Project:

```txt
week3_01_queue_basic
```

Kode:

```c
#include <stdio.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#include "esp_log.h"

static const char *TAG = "QUEUE_BASIC";

static QueueHandle_t number_queue = NULL;

static void producer_task(void *pvParameters)
{
    int counter = 0;

    while (1) {
        counter++;

        ESP_LOGI("PRODUCER", "Sending number: %d", counter);

        if (xQueueSend(number_queue, &counter, pdMS_TO_TICKS(100)) != pdTRUE) {
            ESP_LOGW("PRODUCER", "Queue full, failed to send");
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

static void consumer_task(void *pvParameters)
{
    int received_number = 0;

    while (1) {
        if (xQueueReceive(number_queue, &received_number, portMAX_DELAY) == pdTRUE) {
            ESP_LOGI("CONSUMER", "Received number: %d", received_number);
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Queue basic example started");

    number_queue = xQueueCreate(5, sizeof(int));

    if (number_queue == NULL) {
        ESP_LOGE(TAG, "Failed to create queue");
        return;
    }

    xTaskCreate(producer_task, "Producer Task", 2048, NULL, 2, NULL);
    xTaskCreate(consumer_task, "Consumer Task", 2048, NULL, 2, NULL);
}
```

---

## 4.5 Penjelasan Queue

```c
number_queue = xQueueCreate(5, sizeof(int));
```

Artinya:

```txt
Queue dapat menampung 5 item
Setiap item berukuran sizeof(int)
```

---

```c
xQueueSend(number_queue, &counter, pdMS_TO_TICKS(100));
```

Mengirim nilai `counter` ke queue.

Parameter ketiga adalah timeout. Jika queue penuh, task menunggu maksimal 100 ms.

---

```c
xQueueReceive(number_queue, &received_number, portMAX_DELAY);
```

Task menunggu sampai ada data masuk.

`portMAX_DELAY` artinya menunggu terus sampai data tersedia.

---

## 4.6 Queue dengan Struct

Dalam firmware nyata, jarang hanya mengirim `int`. Biasanya kita kirim struct.

Contoh event:

```c
typedef enum {
    APP_EVENT_BUTTON_PRESSED,
    APP_EVENT_SENSOR_READY,
    APP_EVENT_NETWORK_SEND,
} app_event_type_t;

typedef struct {
    app_event_type_t type;
    int value;
} app_event_t;
```

Full code:

```c
#include <stdio.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#include "esp_log.h"

static const char *TAG = "QUEUE_STRUCT";

#define APP_QUEUE_LENGTH 10

typedef enum {
    APP_EVENT_BUTTON_PRESSED,
    APP_EVENT_SENSOR_READY,
    APP_EVENT_NETWORK_SEND,
} app_event_type_t;

typedef struct {
    app_event_type_t type;
    int value;
} app_event_t;

static QueueHandle_t app_queue = NULL;

static void event_producer_task(void *pvParameters)
{
    int counter = 0;

    while (1) {
        app_event_t event = {
            .type = APP_EVENT_SENSOR_READY,
            .value = counter,
        };

        ESP_LOGI("PRODUCER", "Sending sensor event value=%d", event.value);

        xQueueSend(app_queue, &event, pdMS_TO_TICKS(100));

        counter++;

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

static void app_event_task(void *pvParameters)
{
    app_event_t event;

    while (1) {
        if (xQueueReceive(app_queue, &event, portMAX_DELAY) == pdTRUE) {
            switch (event.type) {
                case APP_EVENT_BUTTON_PRESSED:
                    ESP_LOGI("APP", "Button pressed event");
                    break;

                case APP_EVENT_SENSOR_READY:
                    ESP_LOGI("APP", "Sensor ready value=%d", event.value);
                    break;

                case APP_EVENT_NETWORK_SEND:
                    ESP_LOGI("APP", "Network send event");
                    break;

                default:
                    ESP_LOGW("APP", "Unknown event");
                    break;
            }
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Queue struct example started");

    app_queue = xQueueCreate(APP_QUEUE_LENGTH, sizeof(app_event_t));

    if (app_queue == NULL) {
        ESP_LOGE(TAG, "Failed to create app queue");
        return;
    }

    xTaskCreate(event_producer_task, "Event Producer", 2048, NULL, 2, NULL);
    xTaskCreate(app_event_task, "App Event Task", 4096, NULL, 3, NULL);
}
```

---

## 5. Semaphore

## 5.1 Apa Itu Semaphore?

Semaphore adalah mekanisme sinkronisasi.

Semaphore sering dipakai untuk memberi sinyal:

```txt
Event terjadi
↓
Semaphore diberikan
↓
Task yang menunggu semaphore bangun
```

---

## 5.2 Binary Semaphore

Binary semaphore hanya punya dua kondisi:

```txt
0 = tidak tersedia
1 = tersedia
```

Cocok untuk event signal.

Contoh:

```txt
Button Task mendeteksi tombol
↓
Memberi semaphore
↓
LED Task bangun
```

---

## 5.3 API Binary Semaphore

```c
xSemaphoreCreateBinary()
xSemaphoreGive()
xSemaphoreTake()
xSemaphoreGiveFromISR()
```

Header:

```c
#include "freertos/semphr.h"
```

---

## 5.4 Contoh Binary Semaphore

Project:

```txt
week3_02_binary_semaphore
```

Kode:

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2

static const char *TAG = "BINARY_SEM";

static SemaphoreHandle_t led_semaphore = NULL;

static void signal_task(void *pvParameters)
{
    while (1) {
        ESP_LOGI("SIGNAL", "Giving semaphore");
        xSemaphoreGive(led_semaphore);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

static void led_task(void *pvParameters)
{
    bool led_state = false;

    while (1) {
        if (xSemaphoreTake(led_semaphore, portMAX_DELAY) == pdTRUE) {
            led_state = !led_state;
            gpio_set_level(LED_GPIO, led_state ? 1 : 0);
            ESP_LOGI("LED", "Semaphore received, LED=%s", led_state ? "ON" : "OFF");
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Binary semaphore example started");

    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    led_semaphore = xSemaphoreCreateBinary();

    if (led_semaphore == NULL) {
        ESP_LOGE(TAG, "Failed to create semaphore");
        return;
    }

    xTaskCreate(signal_task, "Signal Task", 2048, NULL, 2, NULL);
    xTaskCreate(led_task, "LED Task", 2048, NULL, 2, NULL);
}
```

---

## 5.5 Counting Semaphore

Counting semaphore punya nilai lebih dari 1.

Cocok untuk:

```txt
1. Menghitung jumlah event
2. Mengatur jumlah resource terbatas
3. Menampung beberapa sinyal yang belum diproses
```

Contoh:

```txt
Tombol ditekan 5 kali cepat
Counting semaphore menyimpan count 5
Task memproses 5 kali
```

API:

```c
xSemaphoreCreateCounting(max_count, initial_count)
```

Contoh:

```c
SemaphoreHandle_t sem = xSemaphoreCreateCounting(10, 0);
```

Artinya semaphore bisa menghitung sampai 10, awalnya 0.

---

## 6. Mutex

## 6.1 Apa Itu Mutex?

Mutex adalah kunci untuk melindungi shared resource.

Shared resource adalah resource yang bisa dipakai lebih dari satu task.

Contoh:

```txt
1. I2C bus
2. SPI bus
3. File SPIFFS
4. LCD
5. Shared struct status
6. UART debug console custom
```

---

## 6.2 Masalah Tanpa Mutex

Misalnya dua task menulis ke file yang sama:

```txt
Task A menulis payload A
Task B menulis payload B
```

Tanpa mutex, data bisa bercampur.

Contoh:

```txt
{"payload":"A{"payload":"B"}}
```

Dengan mutex:

```txt
Task A ambil kunci
Task A menulis file
Task A lepas kunci
Task B baru boleh menulis
```

---

## 6.3 API Mutex

```c
xSemaphoreCreateMutex()
xSemaphoreTake()
xSemaphoreGive()
```

Mutex memakai header yang sama:

```c
#include "freertos/semphr.h"
```

---

## 6.4 Contoh Mutex Shared Counter

Project:

```txt
week3_03_mutex_shared_counter
```

Kode:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"

#include "esp_log.h"

static const char *TAG = "MUTEX_COUNTER";

static SemaphoreHandle_t counter_mutex = NULL;
static int shared_counter = 0;

static void counter_task_a(void *pvParameters)
{
    while (1) {
        if (xSemaphoreTake(counter_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
            shared_counter++;
            ESP_LOGI("TASK_A", "Counter=%d", shared_counter);
            xSemaphoreGive(counter_mutex);
        }

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

static void counter_task_b(void *pvParameters)
{
    while (1) {
        if (xSemaphoreTake(counter_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
            shared_counter += 10;
            ESP_LOGI("TASK_B", "Counter=%d", shared_counter);
            xSemaphoreGive(counter_mutex);
        }

        vTaskDelay(pdMS_TO_TICKS(700));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Mutex shared counter example started");

    counter_mutex = xSemaphoreCreateMutex();

    if (counter_mutex == NULL) {
        ESP_LOGE(TAG, "Failed to create mutex");
        return;
    }

    xTaskCreate(counter_task_a, "Counter A", 2048, NULL, 2, NULL);
    xTaskCreate(counter_task_b, "Counter B", 2048, NULL, 2, NULL);
}
```

---

## 6.5 Best Practice Mutex

```txt
1. Pegang mutex sesingkat mungkin
2. Jangan vTaskDelay() saat memegang mutex
3. Jangan HTTP request saat memegang mutex
4. Gunakan timeout, jangan selalu portMAX_DELAY
5. Selalu xSemaphoreGive() setelah selesai
6. Tentukan urutan lock jika memakai banyak mutex
```

Buruk:

```c
xSemaphoreTake(file_mutex, portMAX_DELAY);
vTaskDelay(pdMS_TO_TICKS(1000));
xSemaphoreGive(file_mutex);
```

Baik:

```c
if (xSemaphoreTake(file_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
    // tulis file singkat
    xSemaphoreGive(file_mutex);
}
```

---

## 7. Event Group

## 7.1 Apa Itu Event Group?

Event group adalah kumpulan bit status.

Satu event group bisa menyimpan banyak status.

Contoh:

```c
#define WIFI_CONNECTED_BIT BIT0
#define MQTT_CONNECTED_BIT BIT1
#define SENSOR_READY_BIT   BIT2
#define OTA_RUNNING_BIT    BIT3
```

Jika WiFi connected:

```c
xEventGroupSetBits(event_group, WIFI_CONNECTED_BIT);
```

Jika WiFi disconnected:

```c
xEventGroupClearBits(event_group, WIFI_CONNECTED_BIT);
```

Task bisa menunggu satu atau beberapa bit.

---

## 7.2 Kapan Pakai Event Group?

Gunakan event group untuk status sistem seperti:

```txt
1. WiFi connected
2. MQTT connected
3. Time synchronized
4. Sensor ready
5. OTA running
6. Provisioning done
7. Storage mounted
```

Jangan pakai event group untuk mengirim data besar. Untuk data, gunakan queue.

---

## 7.3 API Event Group

```c
xEventGroupCreate()
xEventGroupSetBits()
xEventGroupClearBits()
xEventGroupGetBits()
xEventGroupWaitBits()
```

Header:

```c
#include "freertos/event_groups.h"
```

---

## 7.4 Contoh Event Group

Project:

```txt
week3_04_event_group
```

Kode:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"

#include "esp_log.h"

static const char *TAG = "EVENT_GROUP";

#define WIFI_CONNECTED_BIT BIT0
#define MQTT_CONNECTED_BIT BIT1

static EventGroupHandle_t system_event_group = NULL;

static void wifi_simulator_task(void *pvParameters)
{
    while (1) {
        ESP_LOGI("WIFI", "WiFi connected");
        xEventGroupSetBits(system_event_group, WIFI_CONNECTED_BIT);

        vTaskDelay(pdMS_TO_TICKS(5000));

        ESP_LOGW("WIFI", "WiFi disconnected");
        xEventGroupClearBits(system_event_group, WIFI_CONNECTED_BIT | MQTT_CONNECTED_BIT);

        vTaskDelay(pdMS_TO_TICKS(3000));
    }
}

static void mqtt_simulator_task(void *pvParameters)
{
    while (1) {
        xEventGroupWaitBits(
            system_event_group,
            WIFI_CONNECTED_BIT,
            pdFALSE,
            pdTRUE,
            portMAX_DELAY
        );

        ESP_LOGI("MQTT", "WiFi ready, connecting MQTT...");
        vTaskDelay(pdMS_TO_TICKS(1000));

        ESP_LOGI("MQTT", "MQTT connected");
        xEventGroupSetBits(system_event_group, MQTT_CONNECTED_BIT);

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

static void network_worker_task(void *pvParameters)
{
    while (1) {
        xEventGroupWaitBits(
            system_event_group,
            WIFI_CONNECTED_BIT | MQTT_CONNECTED_BIT,
            pdFALSE,
            pdTRUE,
            portMAX_DELAY
        );

        ESP_LOGI("WORKER", "WiFi and MQTT ready, sending data...");

        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Event group example started");

    system_event_group = xEventGroupCreate();

    if (system_event_group == NULL) {
        ESP_LOGE(TAG, "Failed to create event group");
        return;
    }

    xTaskCreate(wifi_simulator_task, "WiFi Simulator", 2048, NULL, 2, NULL);
    xTaskCreate(mqtt_simulator_task, "MQTT Simulator", 2048, NULL, 2, NULL);
    xTaskCreate(network_worker_task, "Network Worker", 2048, NULL, 2, NULL);
}
```

---

## 7.5 Penjelasan `xEventGroupWaitBits`

```c
xEventGroupWaitBits(
    system_event_group,
    WIFI_CONNECTED_BIT | MQTT_CONNECTED_BIT,
    pdFALSE,
    pdTRUE,
    portMAX_DELAY
);
```

Artinya:

```txt
Tunggu sampai WIFI_CONNECTED_BIT dan MQTT_CONNECTED_BIT aktif.
```

Parameter penting:

```txt
pdFALSE → jangan clear bit setelah berhasil
pdTRUE  → tunggu semua bit aktif
```

Jika pakai `pdFALSE` pada parameter wait all:

```txt
Task bangun jika salah satu bit aktif.
```

---

## 8. Task Notification

## 8.1 Apa Itu Task Notification?

Task notification adalah mekanisme sinyal langsung ke task tertentu.

Lebih ringan daripada queue/semaphore.

Cocok untuk:

```txt
1. ISR membangunkan task
2. Timer membangunkan worker task
3. Satu task memberi sinyal ke satu task lain
4. Event sederhana tanpa data kompleks
```

---

## 8.2 API Task Notification

```c
xTaskNotifyGive()
ulTaskNotifyTake()
vTaskNotifyGiveFromISR()
xTaskNotifyFromISR()
```

---

## 8.3 Contoh Task Notification

Project:

```txt
week3_05_task_notification
```

Kode:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"

static const char *TAG = "TASK_NOTIFY";

static TaskHandle_t worker_task_handle = NULL;

static void signal_task(void *pvParameters)
{
    while (1) {
        ESP_LOGI("SIGNAL", "Notify worker task");
        xTaskNotifyGive(worker_task_handle);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

static void worker_task(void *pvParameters)
{
    while (1) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        ESP_LOGI("WORKER", "Notification received, doing work");

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Task notification example started");

    xTaskCreate(worker_task, "Worker Task", 2048, NULL, 2, &worker_task_handle);

    if (worker_task_handle == NULL) {
        ESP_LOGE(TAG, "Failed to create worker task");
        return;
    }

    xTaskCreate(signal_task, "Signal Task", 2048, NULL, 2, NULL);
}
```

---

## 8.4 Task Notification vs Semaphore

| Kebutuhan | Pilihan |
|---|---|
| Sinyal ke satu task | Task notification |
| Sinyal dari ISR ke satu task | Task notification |
| Sinkronisasi resource umum | Semaphore/mutex |
| Data kompleks | Queue |
| Banyak status bit | Event group |

---

## 9. Mini Project Minggu 3

Project:

```txt
week3_final_inter_task_app
```

Fitur:

```txt
1. Button Task membaca tombol dengan polling sederhana.
2. Jika tombol valid ditekan, Button Task mengirim event ke Queue.
3. App Task menerima event.
4. App Task mengubah LED state.
5. Monitor Task membaca shared status dengan mutex.
6. Event Group menyimpan status LED aktif/nonaktif.
```

Arsitektur:

```txt
Button Task
   ↓ Queue
App Task
   ↓ update LED
Shared Status
   ↑ Mutex
Monitor Task

Event Group
   └── LED_ON_BIT
```

---

## 9.1 Full Code Mini Project

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"
#include "freertos/event_groups.h"

#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_heap_caps.h"

#define LED_GPIO        GPIO_NUM_2
#define BUTTON_GPIO     GPIO_NUM_0

#define LED_ON_BIT      BIT0

#define BUTTON_TASK_STACK   2048
#define APP_TASK_STACK      3072
#define MONITOR_TASK_STACK  3072

#define BUTTON_TASK_PRIORITY    3
#define APP_TASK_PRIORITY       3
#define MONITOR_TASK_PRIORITY   1

typedef enum {
    APP_EVENT_BUTTON_PRESSED,
} app_event_type_t;

typedef struct {
    app_event_type_t type;
    int64_t timestamp_ms;
} app_event_t;

typedef struct {
    bool led_state;
    uint32_t button_press_count;
} app_status_t;

static const char *TAG = "WEEK3_APP";

static QueueHandle_t app_queue = NULL;
static SemaphoreHandle_t status_mutex = NULL;
static EventGroupHandle_t app_event_group = NULL;

static app_status_t app_status = {
    .led_state = false,
    .button_press_count = 0,
};

static void gpio_init_all(void)
{
    gpio_config_t led_config = {
        .pin_bit_mask = (1ULL << LED_GPIO),
        .mode = GPIO_MODE_OUTPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE,
    };
    gpio_config(&led_config);
    gpio_set_level(LED_GPIO, 0);

    gpio_config_t button_config = {
        .pin_bit_mask = (1ULL << BUTTON_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE,
    };
    gpio_config(&button_config);
}

static void status_set_led(bool led_state)
{
    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        app_status.led_state = led_state;
        xSemaphoreGive(status_mutex);
    }
}

static void status_increment_button_count(void)
{
    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        app_status.button_press_count++;
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
    bool last_button_state = true;

    while (1) {
        bool current_button_state = gpio_get_level(BUTTON_GPIO);

        if (last_button_state == true && current_button_state == false) {
            vTaskDelay(pdMS_TO_TICKS(50));

            if (gpio_get_level(BUTTON_GPIO) == 0) {
                app_event_t event = {
                    .type = APP_EVENT_BUTTON_PRESSED,
                    .timestamp_ms = xTaskGetTickCount() * portTICK_PERIOD_MS,
                };

                if (xQueueSend(app_queue, &event, pdMS_TO_TICKS(100)) == pdTRUE) {
                    ESP_LOGI("BUTTON", "Button event sent");
                } else {
                    ESP_LOGW("BUTTON", "Failed to send button event");
                }

                while (gpio_get_level(BUTTON_GPIO) == 0) {
                    vTaskDelay(pdMS_TO_TICKS(10));
                }
            }
        }

        last_button_state = current_button_state;

        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

static void app_task(void *pvParameters)
{
    app_event_t event;
    bool led_state = false;

    while (1) {
        if (xQueueReceive(app_queue, &event, portMAX_DELAY) == pdTRUE) {
            switch (event.type) {
                case APP_EVENT_BUTTON_PRESSED:
                    led_state = !led_state;
                    gpio_set_level(LED_GPIO, led_state ? 1 : 0);

                    status_set_led(led_state);
                    status_increment_button_count();

                    if (led_state) {
                        xEventGroupSetBits(app_event_group, LED_ON_BIT);
                    } else {
                        xEventGroupClearBits(app_event_group, LED_ON_BIT);
                    }

                    ESP_LOGI("APP", "Button pressed at %lld ms, LED=%s",
                             event.timestamp_ms,
                             led_state ? "ON" : "OFF");
                    break;

                default:
                    ESP_LOGW("APP", "Unknown event");
                    break;
            }
        }
    }
}

static void monitor_task(void *pvParameters)
{
    app_status_t local_status;

    while (1) {
        if (status_get(&local_status)) {
            EventBits_t bits = xEventGroupGetBits(app_event_group);
            uint32_t free_heap = heap_caps_get_free_size(MALLOC_CAP_DEFAULT);
            UBaseType_t stack_left = uxTaskGetStackHighWaterMark(NULL);

            ESP_LOGI("MONITOR",
                     "LED=%s, LED_BIT=%s, button_count=%lu, free_heap=%lu, stack_left=%u",
                     local_status.led_state ? "ON" : "OFF",
                     (bits & LED_ON_BIT) ? "SET" : "CLEAR",
                     local_status.button_press_count,
                     free_heap,
                     stack_left);
        }

        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Week 3 final inter-task communication app started");

    gpio_init_all();

    app_queue = xQueueCreate(10, sizeof(app_event_t));
    if (app_queue == NULL) {
        ESP_LOGE(TAG, "Failed to create app queue");
        return;
    }

    status_mutex = xSemaphoreCreateMutex();
    if (status_mutex == NULL) {
        ESP_LOGE(TAG, "Failed to create status mutex");
        return;
    }

    app_event_group = xEventGroupCreate();
    if (app_event_group == NULL) {
        ESP_LOGE(TAG, "Failed to create event group");
        return;
    }

    xTaskCreate(button_task, "Button Task", BUTTON_TASK_STACK, NULL,
                BUTTON_TASK_PRIORITY, NULL);

    xTaskCreate(app_task, "App Task", APP_TASK_STACK, NULL,
                APP_TASK_PRIORITY, NULL);

    xTaskCreate(monitor_task, "Monitor Task", MONITOR_TASK_STACK, NULL,
                MONITOR_TASK_PRIORITY, NULL);

    ESP_LOGI(TAG, "System started");
}
```

---

## 10. Penjelasan Mini Project

## 10.1 Button Task sebagai Producer

Button Task membaca tombol lalu membuat event:

```c
app_event_t event = {
    .type = APP_EVENT_BUTTON_PRESSED,
    .timestamp_ms = xTaskGetTickCount() * portTICK_PERIOD_MS,
};
```

Event dikirim ke queue:

```c
xQueueSend(app_queue, &event, pdMS_TO_TICKS(100));
```

Button Task tidak langsung mengontrol LED. Ini pola yang baik.

---

## 10.2 App Task sebagai Consumer

App Task menerima event:

```c
xQueueReceive(app_queue, &event, portMAX_DELAY)
```

Lalu memproses:

```c
led_state = !led_state;
gpio_set_level(LED_GPIO, led_state ? 1 : 0);
```

---

## 10.3 Mutex untuk Shared Status

Status dibaca oleh Monitor Task dan diubah oleh App Task.

Karena shared, harus dilindungi mutex:

```c
xSemaphoreTake(status_mutex, ...)
xSemaphoreGive(status_mutex)
```

---

## 10.4 Event Group untuk Status LED

Jika LED ON:

```c
xEventGroupSetBits(app_event_group, LED_ON_BIT);
```

Jika LED OFF:

```c
xEventGroupClearBits(app_event_group, LED_ON_BIT);
```

Monitor Task bisa membaca status bit.

---

## 11. Best Practice Minggu 3

## 11.1 Queue

```txt
1. Pakai queue untuk mengirim data antar-task.
2. Gunakan struct untuk event yang jelas.
3. Jangan kirim pointer ke local variable.
4. Cek return value xQueueSend().
5. Tentukan panjang queue sesuai kebutuhan.
```

---

## 11.2 Semaphore

```txt
1. Binary semaphore untuk sinyal event.
2. Counting semaphore untuk menghitung banyak event.
3. Gunakan FromISR API jika dari ISR.
4. Jangan gunakan semaphore untuk data kompleks.
```

---

## 11.3 Mutex

```txt
1. Pakai mutex untuk shared resource.
2. Pegang mutex sesingkat mungkin.
3. Gunakan timeout.
4. Hindari deadlock.
5. Jangan lupa give setelah take.
```

---

## 11.4 Event Group

```txt
1. Pakai event group untuk status sistem.
2. Jangan pakai event group untuk data besar.
3. Gunakan bit naming yang jelas.
4. Clear bit saat status tidak berlaku lagi.
```

---

## 11.5 Task Notification

```txt
1. Gunakan untuk sinyal ringan ke satu task.
2. Cocok untuk ISR → task.
3. Lebih ringan daripada semaphore.
4. Tidak cocok untuk banyak consumer.
```

---

## 12. Error Umum Minggu 3

## 12.1 Queue NULL

Penyebab:

```txt
Queue gagal dibuat atau belum dibuat.
```

Solusi:

```c
if (queue == NULL) {
    ESP_LOGE(TAG, "Queue creation failed");
    return;
}
```

---

## 12.2 Queue Full

Penyebab:

```txt
Producer lebih cepat daripada consumer.
```

Solusi:

```txt
1. Perbesar queue
2. Percepat consumer
3. Kurangi rate producer
4. Handle xQueueSend gagal
```

---

## 12.3 Deadlock Mutex

Penyebab:

```txt
Task saling menunggu mutex.
```

Solusi:

```txt
1. Gunakan timeout
2. Jangan pegang mutex lama
3. Tentukan urutan lock
```

---

## 12.4 Data Race

Penyebab:

```txt
Shared variable diakses banyak task tanpa mutex.
```

Solusi:

```txt
Pakai mutex atau ubah pola menjadi queue/event.
```

---

## 12.5 Salah API dari ISR

Buruk:

```c
xQueueSend(queue, &data, 0);
```

di ISR.

Benar:

```c
xQueueSendFromISR(queue, &data, &higher_priority_task_woken);
```

---

## 13. Latihan Minggu 3

## Latihan 1 — Queue Number

Buat:

```txt
Producer Task mengirim angka naik setiap 1 detik.
Consumer Task menerima dan print angka.
```

---

## Latihan 2 — Queue Struct Event

Buat struct event:

```c
typedef struct {
    int type;
    int value;
} event_t;
```

Kirim dari producer ke consumer.

---

## Latihan 3 — Binary Semaphore

Buat Signal Task memberi semaphore setiap 1 detik. LED Task toggle LED saat menerima semaphore.

---

## Latihan 4 — Mutex Shared Counter

Buat dua task menaikkan counter yang sama. Lindungi counter dengan mutex.

---

## Latihan 5 — Event Group WiFi/MQTT Simulation

Buat simulasi:

```txt
WiFi Task set WIFI_CONNECTED_BIT
MQTT Task menunggu WiFi bit lalu set MQTT_CONNECTED_BIT
Network Task menunggu kedua bit lalu print "network ready"
```

---

## Latihan 6 — Task Notification

Buat Signal Task memberi notification ke Worker Task setiap 1 detik.

---

## Latihan 7 — Button Event Queue

Button Task membaca tombol. Saat ditekan, kirim event ke App Task. App Task toggle LED.

---

## 14. Tugas Akhir Minggu 3

Buat project:

```txt
week3_final_inter_task_communication
```

Fitur wajib:

```txt
1. Button Task membaca tombol active-low.
2. Button Task debounce sederhana.
3. Button Task mengirim event ke queue.
4. App Task menerima event dan toggle LED.
5. Shared status berisi:
   - led_state
   - button_press_count
6. Shared status dilindungi mutex.
7. Event group punya LED_ON_BIT.
8. Monitor Task print setiap 2 detik:
   - LED state
   - button count
   - LED_ON_BIT
   - free heap
   - stack left
9. Semua object dicek NULL:
   - queue
   - mutex
   - event group
10. Tidak ada loop tanpa delay atau blocking.
```

---

## 15. Checklist Lulus Minggu 3

Kamu dianggap lulus Minggu 3 kalau bisa:

```txt
[ ] Menjelaskan kenapa task perlu komunikasi
[ ] Menjelaskan queue
[ ] Membuat queue untuk int
[ ] Membuat queue untuk struct
[ ] Menjelaskan producer-consumer
[ ] Menjelaskan binary semaphore
[ ] Menjelaskan counting semaphore
[ ] Menjelaskan mutex
[ ] Melindungi shared variable dengan mutex
[ ] Menjelaskan event group
[ ] Menggunakan xEventGroupSetBits()
[ ] Menggunakan xEventGroupWaitBits()
[ ] Menjelaskan task notification
[ ] Menggunakan xTaskNotifyGive()
[ ] Menggunakan ulTaskNotifyTake()
[ ] Menentukan kapan pakai queue/semaphore/mutex/event group/task notification
[ ] Membuat mini project Button → Queue → App Task → LED
```

---

## 16. Inti Pemahaman Minggu 3

Kalimat penting:

> Task tidak boleh saling mengubah data sembarangan. Gunakan mekanisme komunikasi yang tepat.

Ringkasan pemilihan:

| Kebutuhan | Gunakan |
|---|---|
| Kirim data antar-task | Queue |
| Sinyal event sederhana | Binary semaphore |
| Hitung banyak event/resource | Counting semaphore |
| Lindungi shared resource | Mutex |
| Simpan banyak status bit | Event group |
| Sinyal ringan ke satu task | Task notification |

Pola firmware yang benar:

```txt
Hardware/Input Task
   ↓ Queue/Semaphore/Notification
Application Task
   ↓ Queue
Network/Storage/Display Task
   ↓
Monitor Task mengawasi status
```

Setelah Minggu 3 kuat, kamu siap masuk Minggu 4:

```txt
Interrupt, Timer, dan Watchdog
```
