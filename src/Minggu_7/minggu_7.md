# Minggu 7 — ESP-IDF Storage: NVS, Partition Table, SPIFFS/FATFS, dan Offline Buffer

## Target Minggu 7

Setelah menyelesaikan Minggu 7, kamu diharapkan mampu:

1. Memahami perbedaan RAM, flash, NVS, SPIFFS, dan FATFS.
2. Menyimpan konfigurasi kecil menggunakan NVS.
3. Membuat custom partition table.
4. Mount dan menggunakan SPIFFS untuk menyimpan file.
5. Membuat offline queue berbasis file untuk payload IoT.
6. Memahami batas umur flash dan strategi mengurangi flash wear.
7. Menentukan kapan menggunakan NVS, SPIFFS, atau FATFS.

Minggu ini sangat penting untuk firmware IoT production karena device tidak boleh kehilangan data hanya karena WiFi/server sedang mati.

---

## 1. Kenapa Storage Penting di ESP32?

Tanpa storage, device akan kehilangan banyak informasi setiap restart.

Contoh masalah tanpa storage:

```txt
Device restart
↓
SSID/password hilang
↓
Config harus diinput ulang
↓
WiFi mati
↓
Data sensor gagal dikirim dan hilang
```

Dengan storage:

```txt
Device restart
↓
Baca config dari NVS
↓
Sensor tetap berjalan
↓
Jika server gagal, data disimpan offline
↓
Saat koneksi kembali, data dikirim ulang
```

Contoh penggunaan storage pada project IoT:

| Data | Storage yang Cocok |
|---|---|
| SSID/password WiFi | NVS |
| Device ID | NVS |
| Token / key identifier | NVS / secure storage |
| Calibration value | NVS |
| Boot counter | NVS |
| Payload JSON offline | SPIFFS / FATFS |
| Error log | SPIFFS / FATFS |
| File konfigurasi besar | SPIFFS / FATFS |
| OTA firmware slot | Partition app `ota_0` / `ota_1` |

---

## 2. Jenis Storage di ESP-IDF

### 2.1 NVS

NVS adalah **Non-Volatile Storage**.

NVS cocok untuk data kecil berbasis key-value.

Contoh:

```txt
wifi_ssid = "MyWiFi"
wifi_password = "secret"
device_id = "STUNT-001"
boot_count = 12
height_offset_mm = 5
```

Gunakan NVS untuk:

1. Config kecil.
2. String pendek.
3. Integer.
4. Boolean.
5. Blob/struct kecil.
6. Calibration value.
7. Device identity.

Jangan gunakan NVS untuk:

1. Payload JSON yang terus bertambah.
2. Log panjang.
3. File besar.
4. Data yang ditulis sangat sering.

---

### 2.2 Partition Table

Partition table adalah peta pembagian flash.

Contoh isi flash:

```txt
Bootloader
Partition table
NVS
otadata
Firmware app
File system
Core dump
```

Contoh partition sederhana:

```csv
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 1M
storage,  data, spiffs,  ,        1M
```

---

### 2.3 SPIFFS

SPIFFS adalah file system untuk SPI NOR flash.

Cocok untuk:

1. Offline payload.
2. Log sederhana.
3. File kecil/sedang.
4. Static file web kecil.
5. Data line-delimited JSON.

Kekurangan SPIFFS:

1. Tidak punya directory asli.
2. Performa bisa menurun jika partisi terlalu penuh.
3. Kurang cocok untuk banyak file besar dan sering berubah.

---

### 2.4 FATFS

FATFS cocok untuk:

1. SD card.
2. File banyak.
3. Directory/folder.
4. Logging besar.
5. Data yang perlu dibaca PC.

Untuk flash internal, FATFS biasanya dipakai bersama wear levelling.

---

## 3. Flash Wear

Flash punya batas erase/write cycle. Artinya, flash tidak boleh ditulis terlalu sering tanpa strategi.

Buruk:

```txt
Setiap 10 ms tulis counter ke NVS
```

Lebih baik:

```txt
Simpan counter setiap 1 menit
atau saat nilai penting berubah
atau batch beberapa data dulu
```

Prinsip:

1. Jangan menulis flash terlalu sering.
2. Batch data jika memungkinkan.
3. Gunakan file append untuk offline queue.
4. Batasi ukuran log.
5. Jangan biarkan file system penuh.

---

## 4. Hari 1 — Konsep Flash dan Partition

ESP32 biasanya memiliki flash 4 MB, 8 MB, 16 MB, atau lebih.

Flash digunakan untuk:

1. Bootloader.
2. Partition table.
3. Firmware aplikasi.
4. NVS.
5. OTA metadata.
6. File system.
7. PHY init data.

Perbedaan RAM dan flash:

| RAM | Flash |
|---|---|
| Hilang saat mati | Tetap tersimpan |
| Cepat | Lebih lambat |
| Untuk variable runtime | Untuk firmware/data permanen |
| Bisa sering diubah | Ada batas write/erase |

---

## 5. Hari 2 — NVS Dasar

### 5.1 Alur NVS

Alur umum memakai NVS:

```txt
1. Init NVS flash
2. Open namespace
3. Read/write key
4. Commit jika menulis
5. Close handle
```

API penting:

```c
nvs_flash_init();
nvs_open();
nvs_set_i32();
nvs_get_i32();
nvs_set_str();
nvs_get_str();
nvs_set_blob();
nvs_get_blob();
nvs_commit();
nvs_close();
```

---

### 5.2 Contoh Boot Counter dengan NVS

Project:

```txt
week7_01_nvs_boot_counter
```

Kode:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "nvs_flash.h"
#include "nvs.h"

static const char *TAG = "NVS_COUNTER";

static void nvs_init_storage(void)
{
    esp_err_t ret = nvs_flash_init();

    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_LOGW(TAG, "NVS issue detected, erasing NVS");
        ESP_ERROR_CHECK(nvs_flash_erase());
        ESP_ERROR_CHECK(nvs_flash_init());
    } else {
        ESP_ERROR_CHECK(ret);
    }

    ESP_LOGI(TAG, "NVS initialized");
}

void app_main(void)
{
    ESP_LOGI(TAG, "NVS boot counter example started");

    nvs_init_storage();

    nvs_handle_t handle;

    esp_err_t ret = nvs_open("storage", NVS_READWRITE, &handle);

    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to open NVS: %s", esp_err_to_name(ret));
        return;
    }

    int32_t boot_count = 0;

    ret = nvs_get_i32(handle, "boot_count", &boot_count);

    if (ret == ESP_ERR_NVS_NOT_FOUND) {
        ESP_LOGI(TAG, "boot_count not found, starting from 0");
        boot_count = 0;
    } else if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to read boot_count: %s", esp_err_to_name(ret));
        nvs_close(handle);
        return;
    }

    boot_count++;

    ESP_LOGI(TAG, "Boot count: %ld", boot_count);

    ret = nvs_set_i32(handle, "boot_count", boot_count);

    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to set boot_count: %s", esp_err_to_name(ret));
        nvs_close(handle);
        return;
    }

    ret = nvs_commit(handle);

    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to commit NVS: %s", esp_err_to_name(ret));
    } else {
        ESP_LOGI(TAG, "NVS committed");
    }

    nvs_close(handle);

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

### 5.3 Penjelasan

```c
nvs_flash_init();
```

Menginisialisasi subsystem NVS.

```c
nvs_open("storage", NVS_READWRITE, &handle);
```

Membuka namespace bernama `storage`.

Namespace adalah grup logical key-value.

Contoh namespace:

```txt
wifi
device
calibration
security
app_state
```

```c
nvs_get_i32(handle, "boot_count", &boot_count);
```

Membaca integer dari NVS.

Jika belum ada, hasilnya:

```c
ESP_ERR_NVS_NOT_FOUND
```

```c
nvs_set_i32(...);
nvs_commit(...);
```

`nvs_set_i32()` menyiapkan perubahan.  
`nvs_commit()` menyimpan permanen ke flash.

Jika lupa `nvs_commit()`, data belum tentu tersimpan permanen.

---

## 6. Hari 3 — NVS String dan Blob

### 6.1 Simpan WiFi Config ke NVS

```c
static esp_err_t save_wifi_config(const char *ssid, const char *password)
{
    nvs_handle_t handle;

    esp_err_t ret = nvs_open("wifi", NVS_READWRITE, &handle);
    if (ret != ESP_OK) {
        return ret;
    }

    ret = nvs_set_str(handle, "ssid", ssid);
    if (ret != ESP_OK) {
        nvs_close(handle);
        return ret;
    }

    ret = nvs_set_str(handle, "password", password);
    if (ret != ESP_OK) {
        nvs_close(handle);
        return ret;
    }

    ret = nvs_commit(handle);

    nvs_close(handle);

    return ret;
}
```

---

### 6.2 Baca WiFi Config dari NVS

```c
static esp_err_t load_wifi_config(char *ssid,
                                  size_t ssid_size,
                                  char *password,
                                  size_t password_size)
{
    nvs_handle_t handle;

    esp_err_t ret = nvs_open("wifi", NVS_READONLY, &handle);
    if (ret != ESP_OK) {
        return ret;
    }

    size_t required_size = ssid_size;
    ret = nvs_get_str(handle, "ssid", ssid, &required_size);

    if (ret != ESP_OK) {
        nvs_close(handle);
        return ret;
    }

    required_size = password_size;
    ret = nvs_get_str(handle, "password", password, &required_size);

    nvs_close(handle);

    return ret;
}
```

---

### 6.3 Simpan Struct sebagai Blob

Contoh config:

```c
typedef struct {
    uint32_t version;
    char device_id[32];
    int32_t height_offset_mm;
    int32_t weight_offset_g;
    uint32_t sample_interval_ms;
    bool offline_mode_enabled;
} device_config_t;
```

Simpan:

```c
static esp_err_t save_device_config(const device_config_t *config)
{
    nvs_handle_t handle;

    esp_err_t ret = nvs_open("device", NVS_READWRITE, &handle);
    if (ret != ESP_OK) {
        return ret;
    }

    ret = nvs_set_blob(handle, "config", config, sizeof(device_config_t));

    if (ret == ESP_OK) {
        ret = nvs_commit(handle);
    }

    nvs_close(handle);

    return ret;
}
```

Baca:

```c
static esp_err_t load_device_config(device_config_t *config)
{
    nvs_handle_t handle;

    esp_err_t ret = nvs_open("device", NVS_READONLY, &handle);
    if (ret != ESP_OK) {
        return ret;
    }

    size_t required_size = sizeof(device_config_t);

    ret = nvs_get_blob(handle, "config", config, &required_size);

    nvs_close(handle);

    return ret;
}
```

Catatan penting:

1. Tambahkan field `version` di struct.
2. Jika struktur berubah, data lama mungkin tidak cocok.
3. Jangan simpan data besar di NVS.

---

## 7. App Config Manager

Untuk project nyata, buat layer config.

### 7.1 Struct Config

```c
#define DEVICE_ID_MAX_LEN 32
#define WIFI_SSID_MAX_LEN 32
#define WIFI_PASS_MAX_LEN 64

#define APP_CONFIG_VERSION 1

typedef struct {
    uint32_t version;
    char device_id[DEVICE_ID_MAX_LEN];
    char wifi_ssid[WIFI_SSID_MAX_LEN];
    char wifi_password[WIFI_PASS_MAX_LEN];
    int32_t height_offset_mm;
    uint32_t sample_interval_ms;
    bool offline_buffer_enabled;
} app_config_t;
```

---

### 7.2 Default Config

```c
static void app_config_set_default(app_config_t *config)
{
    memset(config, 0, sizeof(app_config_t));

    config->version = APP_CONFIG_VERSION;
    snprintf(config->device_id, sizeof(config->device_id), "ESP32-DEV-001");
    config->height_offset_mm = 0;
    config->sample_interval_ms = 1000;
    config->offline_buffer_enabled = true;
}
```

---

### 7.3 Load Config dengan Fallback Default

```c
static esp_err_t app_config_load(app_config_t *config)
{
    nvs_handle_t handle;

    esp_err_t ret = nvs_open("app_cfg", NVS_READONLY, &handle);

    if (ret != ESP_OK) {
        app_config_set_default(config);
        return ret;
    }

    size_t size = sizeof(app_config_t);
    ret = nvs_get_blob(handle, "config", config, &size);

    nvs_close(handle);

    if (ret != ESP_OK || config->version != APP_CONFIG_VERSION) {
        app_config_set_default(config);
    }

    return ret;
}
```

---

## 8. Hari 4 — Custom Partition Table

### 8.1 Kenapa Perlu Custom Partition?

Default partition cukup untuk project kecil, tetapi project IoT production sering butuh:

1. NVS lebih besar.
2. OTA slot.
3. Storage file system.
4. Core dump.
5. Factory data.
6. Certificate partition.

---

### 8.2 Contoh `partitions.csv` 4 MB tanpa OTA

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  24K,
phy_init, data, phy,     0xf000,  4K,
factory,  app,  factory, 0x10000, 1M,
storage,  data, spiffs,  ,        1M,
```

---

### 8.3 Contoh `partitions.csv` 8 MB dengan OTA

```csv
# Name,    Type, SubType, Offset,   Size,     Flags
nvs,       data, nvs,     0x9000,   32K,
otadata,   data, ota,     0x11000,  8K,
phy_init,  data, phy,     0x13000,  4K,
factory,   app,  factory, 0x20000,  1536K,
ota_0,     app,  ota_0,   ,         1536K,
ota_1,     app,  ota_1,   ,         1536K,
storage,   data, spiffs,  ,         2M,
```

---

### 8.4 Aktifkan Custom Partition

Jalankan:

```bash
idf.py menuconfig
```

Masuk:

```txt
Partition Table
→ Partition Table
→ Custom partition table CSV
```

Isi:

```txt
partitions.csv
```

Lalu:

```bash
idf.py build
idf.py partition-table
idf.py erase-flash
idf.py flash monitor
```

---

## 9. Hari 5 — SPIFFS Dasar

### 9.1 Mount SPIFFS

Pastikan partition ada:

```csv
storage, data, spiffs, , 1M,
```

Kode:

```c
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "esp_spiffs.h"

static const char *TAG = "SPIFFS_BASIC";

static esp_err_t spiffs_init(void)
{
    esp_vfs_spiffs_conf_t conf = {
        .base_path = "/spiffs",
        .partition_label = "storage",
        .max_files = 5,
        .format_if_mount_failed = true
    };

    esp_err_t ret = esp_vfs_spiffs_register(&conf);

    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to mount SPIFFS: %s", esp_err_to_name(ret));
        return ret;
    }

    size_t total = 0;
    size_t used = 0;

    ret = esp_spiffs_info("storage", &total, &used);

    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "SPIFFS total=%d, used=%d", total, used);
    }

    return ESP_OK;
}
```

---

### 9.2 Write dan Read File

```c
void app_main(void)
{
    ESP_LOGI(TAG, "SPIFFS example started");

    if (spiffs_init() != ESP_OK) {
        return;
    }

    FILE *file = fopen("/spiffs/hello.txt", "w");

    if (file == NULL) {
        ESP_LOGE(TAG, "Failed to open file for writing");
        return;
    }

    fprintf(file, "Hello from ESP-IDF SPIFFS!\n");
    fclose(file);

    ESP_LOGI(TAG, "File written");

    file = fopen("/spiffs/hello.txt", "r");

    if (file == NULL) {
        ESP_LOGE(TAG, "Failed to open file for reading");
        return;
    }

    char line[128];

    if (fgets(line, sizeof(line), file) != NULL) {
        ESP_LOGI(TAG, "Read from file: %s", line);
    }

    fclose(file);

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 10. SPIFFS Append Log

Menulis log append:

```c
static esp_err_t append_log_line(const char *line)
{
    FILE *file = fopen("/spiffs/log.txt", "a");

    if (file == NULL) {
        return ESP_FAIL;
    }

    fprintf(file, "%s\n", line);
    fclose(file);

    return ESP_OK;
}
```

Membaca log:

```c
static void read_log_file(void)
{
    FILE *file = fopen("/spiffs/log.txt", "r");

    if (file == NULL) {
        ESP_LOGW("LOG", "No log file found");
        return;
    }

    char line[128];

    while (fgets(line, sizeof(line), file) != NULL) {
        ESP_LOGI("LOG", "%s", line);
    }

    fclose(file);
}
```

---

## 11. Offline Queue Berbasis File

### 11.1 Konsep

Saat HTTP/MQTT gagal:

```txt
Payload JSON disimpan ke file
```

Contoh file:

```txt
/spiffs/offline_queue.txt
```

Isi:

```json
{"uid":"A1B2","height":95.2,"status":"normal"}
{"uid":"C3D4","height":82.1,"status":"stunted"}
{"uid":"E5F6","height":88.0,"status":"risk"}
```

Satu baris = satu payload JSON.

Format ini disebut line-delimited JSON atau NDJSON.

Keuntungan:

1. Mudah append.
2. Mudah dibaca baris per baris.
3. Cocok untuk offline buffer sederhana.

---

### 11.2 Simpan Payload Offline

```c
static esp_err_t offline_queue_append(const char *json_payload)
{
    FILE *file = fopen("/spiffs/offline_queue.txt", "a");

    if (file == NULL) {
        ESP_LOGE("OFFLINE", "Failed to open offline queue");
        return ESP_FAIL;
    }

    fprintf(file, "%s\n", json_payload);
    fclose(file);

    ESP_LOGI("OFFLINE", "Payload saved offline");

    return ESP_OK;
}
```

---

### 11.3 Baca Payload Offline

```c
static void offline_queue_print_all(void)
{
    FILE *file = fopen("/spiffs/offline_queue.txt", "r");

    if (file == NULL) {
        ESP_LOGW("OFFLINE", "No offline queue file");
        return;
    }

    char line[256];

    while (fgets(line, sizeof(line), file) != NULL) {
        ESP_LOGI("OFFLINE", "Payload: %s", line);
    }

    fclose(file);
}
```

---

### 11.4 Hitung Jumlah Payload Offline

```c
static int offline_queue_count_lines(void)
{
    int count = 0;

    FILE *file = fopen("/spiffs/offline_queue.txt", "r");

    if (file == NULL) {
        return 0;
    }

    char line[256];

    while (fgets(line, sizeof(line), file) != NULL) {
        count++;
    }

    fclose(file);

    return count;
}
```

---

### 11.5 Hapus Queue Setelah Sync Berhasil

```c
static esp_err_t offline_queue_clear(void)
{
    int ret = remove("/spiffs/offline_queue.txt");

    if (ret == 0) {
        return ESP_OK;
    }

    return ESP_FAIL;
}
```

Strategi lebih aman:

```txt
1. Baca offline_queue.txt
2. Kirim payload satu-satu
3. Payload yang gagal ditulis ke offline_tmp.txt
4. Hapus offline_queue.txt
5. Rename offline_tmp.txt menjadi offline_queue.txt
```

Ini mencegah data hilang jika sebagian sync berhasil dan sebagian gagal.

---

## 12. Mutex untuk File Access

Kalau file diakses beberapa task, gunakan mutex.

Contoh:

```txt
Payload Task → append offline queue
Monitor Task → count queue lines
Sync Task → read queue
```

Tanpa mutex, file access bisa bentrok.

```c
static SemaphoreHandle_t file_mutex = NULL;
```

Pola:

```c
if (xSemaphoreTake(file_mutex, pdMS_TO_TICKS(1000)) == pdTRUE) {
    // fopen / fwrite / fgets / remove
    xSemaphoreGive(file_mutex);
}
```

---

## 13. Hari 6 — FATFS dan Wear Levelling

Gunakan FATFS jika:

1. Perlu directory/folder.
2. File banyak.
3. Logging besar.
4. Pakai SD card.
5. Data perlu dibaca PC.

Untuk flash internal, gunakan FATFS bersama wear levelling.

Untuk SD card, SD card umumnya sudah memiliki wear levelling internal.

---

## 14. Mini Project Minggu 7

Project:

```txt
week7_storage_offline_queue
```

Fitur:

1. NVS menyimpan `boot_count`.
2. NVS menyimpan `device_id`.
3. SPIFFS menyimpan offline payload JSON.
4. Timer/task membuat dummy payload setiap 5 detik.
5. Monitor task print storage info.
6. Queue printer task print semua offline payload setiap 30 detik.

---

### 14.1 Partition Table

`partitions.csv`:

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  32K,
phy_init, data, phy,     0xf000,  4K,
factory,  app,  factory, 0x10000, 1M,
storage,  data, spiffs,  ,        1M,
```

---

### 14.2 Arsitektur Mini Project

```txt
app_main
 ├── init NVS
 ├── load config
 ├── init SPIFFS
 ├── create file mutex
 └── start tasks

Payload Generator Task
 └── generate dummy JSON → offline_queue_append()

Monitor Task
 └── print boot_count, storage used, queue count, heap

Queue Printer Task
 └── print isi offline_queue.txt
```

---

## 15. Struktur File yang Direkomendasikan

```txt
week7_storage_offline_queue/
├── CMakeLists.txt
├── partitions.csv
└── main/
    ├── CMakeLists.txt
    ├── main.c
    ├── app_nvs.c
    ├── app_nvs.h
    ├── app_config.c
    ├── app_config.h
    ├── app_fs.c
    ├── app_fs.h
    ├── offline_queue.c
    ├── offline_queue.h
    ├── app_tasks.c
    └── app_tasks.h
```

`main.c` ideal:

```c
#include "esp_log.h"

#include "app_nvs.h"
#include "app_config.h"
#include "app_fs.h"
#include "offline_queue.h"
#include "app_tasks.h"

static const char *TAG = "MAIN";

void app_main(void)
{
    ESP_LOGI(TAG, "Week 7 app started");

    app_nvs_init();
    app_config_load();
    app_fs_init();
    offline_queue_init();
    app_tasks_start();
}
```

---

## 16. Error Umum Minggu 7

### 16.1 NVS Init Error

Error:

```txt
ESP_ERR_NVS_NO_FREE_PAGES
ESP_ERR_NVS_NEW_VERSION_FOUND
```

Solusi development:

```c
nvs_flash_erase();
nvs_flash_init();
```

Catatan production:

Jangan sembarangan erase NVS karena config user bisa hilang.

---

### 16.2 Lupa `nvs_commit()`

Buruk:

```c
nvs_set_i32(handle, "boot_count", count);
nvs_close(handle);
```

Benar:

```c
nvs_set_i32(handle, "boot_count", count);
nvs_commit(handle);
nvs_close(handle);
```

---

### 16.3 SPIFFS Gagal Mount

Cek:

1. Partition `storage` belum ada.
2. Label partition salah.
3. Size partition terlalu kecil.
4. Flash belum di-erase setelah ganti partition.
5. Path salah.

Solusi development:

```bash
idf.py fullclean
idf.py erase-flash
idf.py flash monitor
```

---

### 16.4 File Gagal Dibuka

Benar:

```c
fopen("/spiffs/data.txt", "w");
```

Salah:

```c
fopen("data.txt", "w");
```

Karena SPIFFS dimount di:

```txt
/spiffs
```

---

### 16.5 App Partition Terlalu Kecil

Gejala:

```txt
app partition is too small
```

Solusi:

1. Besarkan app partition.
2. Kurangi storage partition.
3. Gunakan flash lebih besar.
4. Aktifkan optimization for size.

---

## 17. Best Practice Storage

### 17.1 NVS

1. Gunakan NVS untuk config kecil.
2. Jangan simpan payload besar di NVS.
3. Jangan tulis NVS terlalu sering.
4. Selalu commit setelah write.
5. Pakai namespace rapi.
6. Tambahkan version untuk blob config.

---

### 17.2 File System

1. Gunakan mutex jika file diakses banyak task.
2. Jangan biarkan storage penuh.
3. Simpan satu payload per baris untuk offline queue.
4. Gunakan temporary file untuk sync aman.
5. Batasi ukuran log.
6. Jangan append terlalu sering dalam interval sangat pendek.

---

### 17.3 Partition Table

1. Rencanakan OTA sejak awal.
2. Sisakan ruang cukup untuk firmware app.
3. Storage jangan terlalu kecil.
4. Gunakan label partition yang jelas.
5. Setelah ubah partition, erase flash saat development.

---

## 18. Latihan Minggu 7

### Latihan 1 — Boot Counter NVS

Buat boot counter.

Target:

```txt
Boot count: 1
Boot count: 2
Boot count: 3
```

Setiap reset, angka naik.

---

### Latihan 2 — WiFi Config NVS

Simpan:

```txt
ssid
password
```

Lalu baca dan print.

---

### Latihan 3 — Device Config Blob

Simpan struct:

```c
typedef struct {
    uint32_t version;
    char device_id[32];
    int32_t sensor_offset;
    bool offline_enabled;
} device_config_t;
```

---

### Latihan 4 — Custom Partition Table

Buat partition:

```csv
storage, data, spiffs, , 1M,
```

Build dan cek:

```bash
idf.py partition-table
```

---

### Latihan 5 — SPIFFS Write/Read

Buat file:

```txt
/spiffs/hello.txt
```

Tulis dan baca ulang.

---

### Latihan 6 — Offline Queue Append

Buat fungsi:

```c
offline_queue_append("{\"temp\":30}");
```

Baca semua isi file.

---

### Latihan 7 — Queue Count

Hitung jumlah payload offline berdasarkan jumlah baris file.

---

### Latihan 8 — Simulasi Sync

Buat fungsi:

```txt
1. Baca offline_queue.txt
2. Print semua payload seolah-olah dikirim
3. Hapus file setelah selesai
```

---

## 19. Tugas Akhir Minggu 7

Buat project:

```txt
week7_final_storage_offline_sync
```

Fitur wajib:

1. Custom partition table dengan NVS dan storage SPIFFS.
2. NVS menyimpan:
   - device_id
   - boot_count
   - sample_interval_ms
3. Jika config belum ada, buat default config.
4. SPIFFS menyimpan `offline_queue.txt`.
5. Payload generator membuat dummy JSON setiap 5 detik.
6. Payload disimpan ke offline queue.
7. Monitor task print:
   - device_id
   - boot_count
   - storage used/total
   - queue count
   - free heap
8. File access dilindungi mutex.
9. Ada fungsi print semua offline queue.
10. Ada fungsi clear offline queue.

---

## 20. Checklist Lulus Minggu 7

Kamu dianggap lulus Minggu 7 kalau bisa:

```txt
[ ] Menjelaskan perbedaan RAM dan flash
[ ] Menjelaskan fungsi partition table
[ ] Membuat custom partitions.csv
[ ] Menjelaskan NVS untuk key-value
[ ] Init NVS dengan nvs_flash_init()
[ ] Open namespace dengan nvs_open()
[ ] Simpan integer ke NVS
[ ] Simpan string ke NVS
[ ] Simpan blob/struct ke NVS
[ ] Commit perubahan NVS
[ ] Mount SPIFFS
[ ] Write file ke SPIFFS
[ ] Read file dari SPIFFS
[ ] Append file log
[ ] Membuat offline queue berbasis file
[ ] Menggunakan mutex untuk file access
[ ] Menjelaskan kapan pakai NVS vs SPIFFS vs FATFS
[ ] Menjelaskan flash wear dan kenapa tidak boleh terlalu sering write
```

---

## 21. Penerapan untuk Project Stunting IoT

Mapping storage untuk project stunting:

| Data | Storage |
|---|---|
| Device ID | NVS |
| WiFi SSID/password | NVS |
| AES/HMAC key identifier | NVS / secure storage |
| Calibration tinggi badan | NVS |
| Last sync timestamp | NVS |
| Data diagnosis offline | SPIFFS/FATFS |
| Error log | SPIFFS/FATFS |
| OTA slot | Partition app `ota_0` / `ota_1` |
| Factory settings | NVS namespace factory |

Contoh offline payload:

```json
{
  "device_id": "STUNT-001",
  "uid": "A1B2C3D4",
  "age_month": 24,
  "height_cm": 82.5,
  "status": "risk",
  "timestamp": 123456789
}
```

Disimpan sebagai satu baris di:

```txt
/spiffs/offline_queue.txt
```

Saat WiFi/server tersedia:

```txt
1. Baca payload pertama
2. Kirim ke server
3. Kalau berhasil, lanjut
4. Kalau gagal, sisakan di queue
5. Jangan hapus data yang belum terkirim
```

---

## 22. Inti Pemahaman Minggu 7

Kalimat penting:

> NVS untuk konfigurasi kecil. File system untuk data/file. Partition table menentukan ruang hidup semuanya.

Pola production IoT:

```txt
Boot
 ↓
NVS init
 ↓
Load config
 ↓
Mount file system
 ↓
Sensor read
 ↓
Try send to server
 ├── success → selesai
 └── failed  → append offline queue
 ↓
Later sync offline queue
```

Setelah Minggu 7 kuat, kamu siap masuk Minggu 8:

```txt
WiFi Station/AP, HTTP Client/Server, dan dasar konektivitas IoT
```
