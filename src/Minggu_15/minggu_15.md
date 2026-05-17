# Minggu 15 — Production Firmware Architecture ESP-IDF

## Target Pembelajaran

Pada akhir Minggu 15, kamu mampu:

- Menyusun firmware ESP-IDF yang modular, scalable, dan siap production.
- Memahami component structure ESP-IDF.
- Memisahkan layer board, driver, service, dan application.
- Membuat event bus internal.
- Membuat state machine.
- Membuat config manager, error manager, command queue, dan health report.
- Membuat template project firmware production.

---

## 1. Masalah Firmware Tidak Terstruktur

Firmware pemula biasanya menumpuk semua di `main.c`:

```text
main.c
├── init GPIO
├── init WiFi
├── init MQTT
├── init sensor
├── init storage
├── task sensor
├── task network
├── callback MQTT
├── parser JSON
├── AES/HMAC
└── semua logic aplikasi
```

Masalahnya:

- Sulit dicari bug.
- Sulit dites.
- Banyak global variable.
- Module saling ketergantungan.
- Perubahan kecil merusak bagian lain.
- Tidak jelas task mana pemilik data.
- Tidak jelas siapa boleh akses hardware.
- Sulit dipakai ulang.

Firmware production harus punya struktur, ownership, state, event flow, error handling, dan observability.

---

## 2. Prinsip Utama Arsitektur Firmware

### Separation of Concerns

Setiap module punya satu tanggung jawab.

```text
app_wifi        -> urusan WiFi
app_mqtt        -> urusan MQTT
app_storage     -> urusan storage/offline queue
app_sensor      -> urusan sensor
app_crypto      -> urusan HMAC/AES
app_ota         -> urusan OTA
app_display     -> urusan LCD/OLED
app_main_fsm    -> alur aplikasi
```

### Single Owner

Setiap resource punya pemilik.

| Resource | Owner |
|---|---|
| I2C bus | app_i2c_bus |
| RFID RC522 | driver_rc522 |
| VL53L0X | driver_vl53l0x |
| LCD | app_display |
| Offline file | app_offline_queue |
| MQTT client | app_mqtt |
| Device config | app_config |

Buruk:

```c
FILE *f = fopen("/spiffs/offline_queue.txt", "a");
```

di sembarang task.

Baik:

```c
app_offline_queue_append(payload);
```

### Event-Driven

```text
Button ISR
  ↓
Input Task
  ↓ event queue
App FSM
  ↓ event
Network / Display / Storage
```

### Callback Ringan

Callback ISR, WiFi event, MQTT event, BLE write, timer, HTTP server handler tidak boleh kerja berat.

Callback cukup:

- Copy data kecil.
- Set event bit.
- Send queue.
- Notify task.

### Explicit State

Device harus punya state jelas:

```text
BOOTING
PROVISIONING
WIFI_CONNECTING
IDLE
MEASURING
PREDICTING
SENDING
OFFLINE_SAVE
OTA_UPDATING
ERROR
SLEEPING
```

### Observable

Firmware harus bisa menjelaskan kondisinya:

- health monitor.
- reset reason.
- boot count.
- heap/stack report.
- last error.
- WiFi/MQTT status.
- offline queue count.
- firmware version.

---

## 3. Layer Arsitektur Firmware

Gunakan 5 layer:

```text
Application Layer
Service Layer
Driver Layer
HAL / Board Layer
System Layer
```

### Application Layer

Logic utama produk.

Contoh stunting:

- Input umur.
- Baca UID RFID.
- Baca tinggi badan.
- Prediksi status.
- Bangun payload.
- Kirim/simpan data.

### Service Layer

Layanan umum:

```text
app_wifi
app_mqtt
app_http
app_ota
app_config
app_storage
app_offline_queue
app_crypto
app_health
app_time
```

### Driver Layer

Driver device eksternal:

```text
driver_rc522
driver_vl53l0x
driver_lcd_i2c
driver_button
driver_buzzer
driver_battery
```

Driver tidak tahu logic bisnis.

### Board Layer

Mapping pin dan konfigurasi hardware:

```text
board_pins.h
board_config.h
board_power.c
```

### System Layer

Helper sistem:

```text
sys_event_bus
sys_error
sys_time
sys_log
sys_memory
```

---

## 4. Struktur Project Production

```text
stunt_device_firmware/
├── CMakeLists.txt
├── partitions.csv
├── sdkconfig.defaults
├── README.md
├── docs/
│   ├── architecture.md
│   ├── state_machine.md
│   ├── ota.md
│   ├── security.md
│   └── manufacturing.md
├── components/
│   ├── board/
│   ├── app_types/
│   ├── app_config/
│   ├── app_event/
│   ├── app_error/
│   ├── app_health/
│   ├── app_storage/
│   ├── app_offline_queue/
│   ├── app_wifi/
│   ├── app_mqtt/
│   ├── app_http/
│   ├── app_ota/
│   ├── app_crypto/
│   ├── app_payload/
│   ├── app_prediction/
│   ├── app_command/
│   ├── app_fsm/
│   ├── app_display/
│   ├── driver_button/
│   ├── driver_battery/
│   ├── driver_vl53l0x/
│   ├── driver_rc522/
│   └── driver_lcd_i2c/
└── main/
    ├── CMakeLists.txt
    └── main.c
```

Root `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(stunt_device VERSION 1.0.0)
```

`main/CMakeLists.txt`:

```cmake
idf_component_register(
    SRCS "main.c"
    INCLUDE_DIRS "."
    REQUIRES
        board
        app_config
        app_event
        app_health
        app_storage
        app_wifi
        app_mqtt
        app_ota
        app_command
        app_fsm
)
```

---

## 5. Board Component

`components/board/include/board_pins.h`:

```c
#ifndef BOARD_PINS_H
#define BOARD_PINS_H

#include "driver/gpio.h"
#include "driver/i2c_master.h"
#include "driver/spi_master.h"
#include "esp_adc/adc_oneshot.h"

#define BOARD_LED_GPIO              GPIO_NUM_2
#define BOARD_BUTTON_GPIO           GPIO_NUM_0

#define BOARD_I2C_PORT              I2C_NUM_0
#define BOARD_I2C_SDA_GPIO          GPIO_NUM_8
#define BOARD_I2C_SCL_GPIO          GPIO_NUM_9

#define BOARD_SPI_HOST              SPI2_HOST
#define BOARD_SPI_MOSI_GPIO         GPIO_NUM_11
#define BOARD_SPI_MISO_GPIO         GPIO_NUM_13
#define BOARD_SPI_SCLK_GPIO         GPIO_NUM_12

#define BOARD_RFID_CS_GPIO          GPIO_NUM_10
#define BOARD_RFID_RST_GPIO         GPIO_NUM_14

#define BOARD_BATTERY_ADC_CHANNEL   ADC_CHANNEL_0

#endif
```

Jika pindah board, ubah satu file ini.

---

## 6. App Config Component

`app_config.h`:

```c
#ifndef APP_CONFIG_H
#define APP_CONFIG_H

#include <stdbool.h>
#include <stdint.h>
#include "esp_err.h"

#define APP_DEVICE_ID_MAX_LEN 32
#define APP_WIFI_SSID_MAX_LEN 32
#define APP_WIFI_PASS_MAX_LEN 64
#define APP_CONFIG_VERSION 1

typedef struct {
    uint32_t version;
    char device_id[APP_DEVICE_ID_MAX_LEN];
    char wifi_ssid[APP_WIFI_SSID_MAX_LEN];
    char wifi_password[APP_WIFI_PASS_MAX_LEN];
    uint32_t sample_interval_ms;
    bool offline_enabled;
    int32_t height_offset_mm;
} app_config_t;

esp_err_t app_config_init(void);
esp_err_t app_config_load(app_config_t *out_config);
esp_err_t app_config_save(const app_config_t *config);
void app_config_set_default(app_config_t *config);

#endif
```

`app_config.c` inti:

```c
void app_config_set_default(app_config_t *config)
{
    memset(config, 0, sizeof(app_config_t));

    config->version = APP_CONFIG_VERSION;
    snprintf(config->device_id, sizeof(config->device_id), "STUNT-001");
    config->sample_interval_ms = 1000;
    config->offline_enabled = true;
    config->height_offset_mm = 0;
}
```

Config harus terpusat. Jangan menyebar SSID/device_id/server_url di banyak file.

---

## 7. App Types

`app_types.h`:

```c
#ifndef APP_TYPES_H
#define APP_TYPES_H

#include <stdint.h>
#include <stdbool.h>

#define APP_UID_MAX_LEN 16
#define APP_DEVICE_ID_MAX_LEN 32

typedef enum {
    STUNTING_STATUS_NORMAL = 0,
    STUNTING_STATUS_RISK,
    STUNTING_STATUS_STUNTED,
    STUNTING_STATUS_UNKNOWN,
} stunting_status_t;

typedef struct {
    char uid[APP_UID_MAX_LEN];
    int age_month;
    float height_cm;
} diagnosis_input_t;

typedef struct {
    stunting_status_t status;
    float z_score;
    char recommendation[128];
} diagnosis_result_t;

typedef struct {
    char device_id[APP_DEVICE_ID_MAX_LEN];
    diagnosis_input_t input;
    diagnosis_result_t result;
    uint32_t counter;
    int64_t timestamp_us;
} diagnosis_record_t;

#endif
```

Keuntungan:

- Payload builder jelas.
- Storage jelas.
- MQTT/HTTP jelas.
- Unit test lebih mudah.

---

## 8. Event Bus Internal

`app_event.h`:

```c
#ifndef APP_EVENT_H
#define APP_EVENT_H

#include "esp_event.h"
#include "esp_err.h"

ESP_EVENT_DECLARE_BASE(APP_EVENT);

typedef enum {
    APP_EVENT_BOOT_DONE = 1,
    APP_EVENT_WIFI_CONNECTED,
    APP_EVENT_WIFI_DISCONNECTED,
    APP_EVENT_MQTT_CONNECTED,
    APP_EVENT_MQTT_DISCONNECTED,
    APP_EVENT_BUTTON_PRESSED,
    APP_EVENT_RFID_READ,
    APP_EVENT_HEIGHT_MEASURED,
    APP_EVENT_DIAGNOSIS_READY,
    APP_EVENT_PAYLOAD_SENT,
    APP_EVENT_PAYLOAD_SAVE_OFFLINE,
    APP_EVENT_OTA_STARTED,
    APP_EVENT_OTA_FINISHED,
    APP_EVENT_ERROR,
} app_event_id_t;

esp_err_t app_event_init(void);
esp_err_t app_event_post(app_event_id_t event_id,
                         const void *event_data,
                         size_t event_data_size);

#endif
```

`app_event.c`:

```c
#include "app_event.h"

ESP_EVENT_DEFINE_BASE(APP_EVENT);

esp_err_t app_event_init(void)
{
    return ESP_OK;
}

esp_err_t app_event_post(app_event_id_t event_id,
                         const void *event_data,
                         size_t event_data_size)
{
    return esp_event_post(
        APP_EVENT,
        event_id,
        event_data,
        event_data_size,
        pdMS_TO_TICKS(100)
    );
}
```

---

## 9. State Machine

State untuk stunting device:

```c
typedef enum {
    APP_STATE_BOOTING = 0,
    APP_STATE_PROVISIONING,
    APP_STATE_WIFI_CONNECTING,
    APP_STATE_IDLE,
    APP_STATE_WAITING_RFID,
    APP_STATE_WAITING_INPUT,
    APP_STATE_MEASURING_HEIGHT,
    APP_STATE_PREDICTING,
    APP_STATE_SENDING,
    APP_STATE_SAVING_OFFLINE,
    APP_STATE_SYNCING_OFFLINE,
    APP_STATE_OTA_UPDATING,
    APP_STATE_ERROR,
    APP_STATE_SLEEPING,
} app_state_t;
```

Transisi contoh:

| Current State | Event | Next State |
|---|---|---|
| BOOTING | BOOT_OK | WIFI_CONNECTING |
| BOOTING | PROVISIONING_REQUIRED | PROVISIONING |
| WIFI_CONNECTING | WIFI_CONNECTED | IDLE |
| IDLE | RFID_OK | WAITING_INPUT |
| WAITING_INPUT | INPUT_OK | MEASURING_HEIGHT |
| MEASURING_HEIGHT | HEIGHT_OK | PREDICTING |
| PREDICTING | PREDICTION_OK | SENDING |
| SENDING | SEND_OK | IDLE |
| SENDING | SEND_FAILED | SAVING_OFFLINE |
| ANY | OTA_REQUEST | OTA_UPDATING |
| ANY | ERROR | ERROR |

Log setiap transisi state:

```c
static void app_fsm_set_state(app_state_t new_state)
{
    if (current_state == new_state) {
        return;
    }

    ESP_LOGI("APP_FSM", "State: %s -> %s",
             app_state_to_str(current_state),
             app_state_to_str(new_state));

    current_state = new_state;
}
```

---

## 10. Command Pattern

Command bisa datang dari BLE, MQTT, HTTP server, UART, atau button.

Semua masuk ke command queue yang sama.

`app_command.h`:

```c
#ifndef APP_COMMAND_H
#define APP_COMMAND_H

#include "esp_err.h"

typedef enum {
    APP_COMMAND_LED_ON = 1,
    APP_COMMAND_LED_OFF,
    APP_COMMAND_SYNC_NOW,
    APP_COMMAND_OTA_CHECK,
    APP_COMMAND_RESET_WIFI,
    APP_COMMAND_FACTORY_RESET,
    APP_COMMAND_STATUS,
    APP_COMMAND_CALIBRATE_HEIGHT,
} app_command_type_t;

typedef struct {
    app_command_type_t type;
    int32_t int_value;
    char string_value[64];
} app_command_t;

esp_err_t app_command_init(void);
esp_err_t app_command_send(const app_command_t *command);

#endif
```

Callback BLE/MQTT cukup convert payload menjadi `app_command_t`, lalu kirim ke queue.

---

## 11. Error Manager

`app_error.h`:

```c
#ifndef APP_ERROR_H
#define APP_ERROR_H

typedef enum {
    APP_ERR_NONE = 0,
    APP_ERR_WIFI_CONNECT_FAILED,
    APP_ERR_SENSOR_READ_FAILED,
    APP_ERR_STORAGE_FULL,
    APP_ERR_OTA_FAILED,
    APP_ERR_LOW_HEAP,
} app_error_code_t;

void app_error_set(app_error_code_t code, const char *message);
app_error_code_t app_error_get_last(void);
const char *app_error_to_str(app_error_code_t code);

#endif
```

Manfaat:

- Error terakhir bisa tampil di LCD.
- Bisa dikirim ke MQTT health report.
- Debug lapangan lebih mudah.

---

## 12. Payload Builder

Network layer tidak boleh membuat JSON manual dari awal. Pisahkan payload builder.

`app_payload.h`:

```c
#ifndef APP_PAYLOAD_H
#define APP_PAYLOAD_H

#include "app_types.h"

char *app_payload_build_diagnosis_json(const diagnosis_record_t *record);

#endif
```

`app_payload.c`:

```c
#include "app_payload.h"
#include "cJSON.h"

static const char *status_to_str(stunting_status_t status)
{
    switch (status) {
        case STUNTING_STATUS_NORMAL: return "normal";
        case STUNTING_STATUS_RISK: return "risk";
        case STUNTING_STATUS_STUNTED: return "stunted";
        default: return "unknown";
    }
}

char *app_payload_build_diagnosis_json(const diagnosis_record_t *record)
{
    if (record == NULL) {
        return NULL;
    }

    cJSON *root = cJSON_CreateObject();

    if (root == NULL) {
        return NULL;
    }

    cJSON_AddStringToObject(root, "device_id", record->device_id);
    cJSON_AddStringToObject(root, "uid", record->input.uid);
    cJSON_AddNumberToObject(root, "age_month", record->input.age_month);
    cJSON_AddNumberToObject(root, "height_cm", record->input.height_cm);
    cJSON_AddStringToObject(root, "status", status_to_str(record->result.status));
    cJSON_AddNumberToObject(root, "z_score", record->result.z_score);
    cJSON_AddStringToObject(root, "recommendation", record->result.recommendation);
    cJSON_AddNumberToObject(root, "counter", record->counter);
    cJSON_AddNumberToObject(root, "timestamp_us", record->timestamp_us);

    char *json = cJSON_PrintUnformatted(root);

    cJSON_Delete(root);

    return json;
}
```

Pemanggil wajib `free(json)`.

---

## 13. Main.c yang Bersih

`main.c` hanya orkestrasi init/start.

```c
#include "esp_log.h"

#include "board.h"
#include "app_config.h"
#include "app_event.h"
#include "app_health.h"
#include "app_storage.h"
#include "app_wifi.h"
#include "app_mqtt.h"
#include "app_command.h"
#include "app_fsm.h"
#include "app_ota.h"

static const char *TAG = "MAIN";

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Stunt Device Firmware");

    ESP_ERROR_CHECK(board_init());
    ESP_ERROR_CHECK(app_config_init());
    ESP_ERROR_CHECK(app_event_init());
    ESP_ERROR_CHECK(app_storage_init());
    ESP_ERROR_CHECK(app_health_init());

    ESP_ERROR_CHECK(app_command_init());
    ESP_ERROR_CHECK(app_wifi_init());
    ESP_ERROR_CHECK(app_mqtt_init());
    ESP_ERROR_CHECK(app_ota_init());

    ESP_ERROR_CHECK(app_fsm_start());

    ESP_LOGI(TAG, "System started");
}
```

Jika `main.c` terlalu panjang, arsitektur mulai bocor.

---

## 14. Testing Strategy

### Unit Test

Cocok untuk:

- version compare.
- payload builder.
- HMAC output.
- prediction function.
- config migration.
- parser command.
- state transition.
- battery percentage calculation.

Contoh:

```c
#include "unity.h"
#include "app_version.h"

TEST_CASE("version newer works", "[app_version]")
{
    TEST_ASSERT_TRUE(app_version_is_newer("1.0.2", "1.0.1"));
    TEST_ASSERT_FALSE(app_version_is_newer("1.0.1", "1.0.1"));
    TEST_ASSERT_FALSE(app_version_is_newer("1.0.0", "1.0.1"));
}
```

### Integration Test

Uji di board:

- WiFi connect.
- MQTT publish/subscribe.
- HTTP POST.
- SPIFFS offline queue.
- OTA update.
- BLE provisioning.
- Sensor read.
- LCD update.

### Fault Injection Test

Sengaja rusak:

- WiFi mati.
- Server mati.
- MQTT broker mati.
- Sensor dilepas.
- Storage penuh.
- Payload terlalu besar.
- Baterai rendah.
- OTA firmware salah.

Firmware production harus tidak crash dan tetap menyimpan data penting.

---

## 15. Final Project Template

```text
firmware/
├── CMakeLists.txt
├── partitions.csv
├── sdkconfig.defaults
├── README.md
├── docs/
│   ├── architecture.md
│   ├── state_machine.md
│   ├── error_handling.md
│   ├── ota.md
│   └── security.md
├── components/
│   ├── board/
│   ├── app_types/
│   ├── app_config/
│   ├── app_event/
│   ├── app_error/
│   ├── app_health/
│   ├── app_storage/
│   ├── app_offline_queue/
│   ├── app_wifi/
│   ├── app_mqtt/
│   ├── app_http/
│   ├── app_ota/
│   ├── app_crypto/
│   ├── app_payload/
│   ├── app_prediction/
│   ├── app_command/
│   ├── app_fsm/
│   ├── app_display/
│   ├── driver_button/
│   ├── driver_battery/
│   ├── driver_vl53l0x/
│   ├── driver_rc522/
│   └── driver_lcd_i2c/
└── main/
    ├── CMakeLists.txt
    └── main.c
```

---

## 16. Tugas Akhir Minggu 15

Project:

```text
week15_final_production_firmware_template
```

Fitur wajib:

1. Struktur component:
   - board.
   - app_config.
   - app_event.
   - app_error.
   - app_health.
   - app_storage.
   - app_offline_queue.
   - app_wifi.
   - app_payload.
   - app_command.
   - app_fsm.
   - main.
2. `main.c` hanya orkestrasi init/start.
3. `app_config` load default config dan save/load NVS.
4. `app_event` punya event base internal.
5. `app_command` punya command queue dan command task.
6. `app_fsm` punya minimal 6 state dan log transisi.
7. `app_payload` build JSON dari struct data.
8. `app_offline_queue` append dan count payload.
9. `app_health` report heap, uptime, reset reason, last error.
10. `board` menyimpan semua pin mapping.
11. Dokumentasi:
    - `architecture.md`
    - `state_machine.md`
    - `error_handling.md`

---

## 17. Latihan

1. Pecah project Minggu 8/9 menjadi component.
2. Buat board abstraction.
3. Buat `app_types.h`.
4. Buat payload builder testable.
5. Buat command queue terpadu.
6. Implementasi FSM minimal.
7. Buat error manager.
8. Buat health report JSON.

---

## 18. Checklist Lulus

- [ ] Menjelaskan kenapa `main.c` tidak boleh terlalu besar.
- [ ] Membuat ESP-IDF component.
- [ ] Menggunakan `idf_component_register()`.
- [ ] Menggunakan `REQUIRES` dan `PRIV_REQUIRES`.
- [ ] Membuat board abstraction.
- [ ] Memisahkan driver layer dan app layer.
- [ ] Membuat service layer.
- [ ] Membuat `app_types.h`.
- [ ] Membuat payload builder yang testable.
- [ ] Membuat command queue terpadu.
- [ ] Membuat state machine.
- [ ] Membuat event bus internal.
- [ ] Membuat config manager.
- [ ] Membuat error manager.
- [ ] Membuat health report.
- [ ] Menjelaskan ownership resource.
- [ ] Menjelaskan callback tidak boleh kerja berat.
- [ ] Menjelaskan testing strategy.
- [ ] Membuat template firmware production.
