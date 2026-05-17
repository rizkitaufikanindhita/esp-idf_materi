# Minggu 13 — ESP-IDF OTA Update: HTTP/HTTPS OTA, Version Check, Rollback, Signed Firmware, Anti-Rollback

## Target Pembelajaran

Pada akhir Minggu 13, kamu mampu:

- Menjelaskan konsep OTA update.
- Mendesain partition table untuk OTA.
- Menggunakan `esp_https_ota()`.
- Membuat firmware manifest dan version check.
- Mengaktifkan rollback.
- Memvalidasi firmware baru setelah boot.
- Memahami signed firmware dan anti-rollback.
- Mendesain OTA manager untuk production device.

---

## 1. Apa Itu OTA?

OTA adalah Over-The-Air update. Firmware diperbarui tanpa kabel USB.

Alur:

```text
Firmware baru diupload ke server
↓
ESP32 cek update
↓
ESP32 download firmware
↓
ESP32 tulis ke slot OTA tidak aktif
↓
ESP32 reboot
↓
Firmware baru aktif
↓
Firmware baru validasi diri
↓
Jika gagal, rollback ke firmware lama
```

OTA penting karena device IoT sering tersebar di banyak lokasi.

---

## 2. Partition OTA

OTA butuh minimal:

```text
otadata
ota_0
ota_1
```

Jika firmware sedang berjalan dari `ota_0`, update akan ditulis ke `ota_1`. Update berikutnya bergantian.

`otadata` menyimpan informasi slot boot aktif.

Contoh `partitions.csv` 4 MB:

```csv
# Name,    Type, SubType, Offset,   Size,     Flags
nvs,       data, nvs,     0x9000,   32K,
otadata,   data, ota,     0x11000,  8K,
phy_init,  data, phy,     0x13000,  4K,
ota_0,     app,  ota_0,   0x20000,  1408K,
ota_1,     app,  ota_1,   ,         1408K,
storage,   data, spiffs,  ,         512K,
```

Untuk 8 MB:

```csv
# Name,    Type, SubType, Offset,   Size,     Flags
nvs,       data, nvs,     0x9000,   32K,
otadata,   data, ota,     0x11000,  8K,
phy_init,  data, phy,     0x13000,  4K,
ota_0,     app,  ota_0,   0x20000,  2M,
ota_1,     app,  ota_1,   ,         2M,
storage,   data, spiffs,  ,         3M,
```

Aktifkan custom partition:

```bash
idf.py menuconfig
```

```text
Partition Table -> Custom partition table CSV
```

Lalu:

```bash
idf.py build
idf.py partition-table
idf.py erase-flash
idf.py flash monitor
```

---

## 3. Cek Partition Berjalan

```c
#include "esp_log.h"
#include "esp_ota_ops.h"

static const char *TAG = "OTA_INFO";

static void print_partition_info(void)
{
    const esp_partition_t *running = esp_ota_get_running_partition();
    const esp_partition_t *boot = esp_ota_get_boot_partition();

    ESP_LOGI(TAG, "Running partition: %s", running->label);
    ESP_LOGI(TAG, "Boot partition: %s", boot->label);
}
```

---

## 4. HTTPS OTA Basic

Gunakan `esp_https_ota()` untuk pendekatan praktis.

```c
#include <stdio.h>
#include <string.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "esp_system.h"
#include "esp_https_ota.h"
#include "esp_http_client.h"
#include "esp_crt_bundle.h"
#include "esp_ota_ops.h"

static const char *TAG = "HTTPS_OTA";

#define OTA_FIRMWARE_URL "https://your-domain.com/firmware/firmware.bin"

static esp_err_t perform_https_ota(const char *firmware_url)
{
    esp_http_client_config_t http_config = {
        .url = firmware_url,
        .timeout_ms = 15000,
        .crt_bundle_attach = esp_crt_bundle_attach,
        .keep_alive_enable = true,
    };

    esp_https_ota_config_t ota_config = {
        .http_config = &http_config,
    };

    ESP_LOGI(TAG, "Starting OTA from URL: %s", firmware_url);

    esp_err_t ret = esp_https_ota(&ota_config);

    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "OTA successful, restarting...");
        esp_restart();
    } else {
        ESP_LOGE(TAG, "OTA failed: %s", esp_err_to_name(ret));
    }

    return ret;
}
```

Catatan:

```text
Panggil OTA hanya setelah WiFi got IP.
Untuk production, gunakan HTTPS dengan certificate verification.
```

---

## 5. Server Firmware Latihan

Build firmware:

```bash
idf.py build
```

File binary:

```text
build/<project_name>.bin
```

Jalankan local server:

```bash
python -m http.server 8000
```

URL contoh:

```text
http://192.168.1.10:8000/firmware.bin
```

Untuk production, gunakan HTTPS valid.

---

## 6. Version Check

Jangan OTA setiap boot. Device harus cek manifest.

Manifest contoh:

```json
{
  "version": "1.0.2",
  "url": "https://firmware.example.com/stunt-device/1.0.2/firmware.bin",
  "sha256": "abc123...",
  "min_battery_mv": 3600,
  "notes": "Fix offline queue bug"
}
```

Cek versi firmware saat ini:

```c
#include "esp_app_desc.h"
#include "esp_log.h"

static void print_app_version(void)
{
    const esp_app_desc_t *app_desc = esp_app_get_description();

    ESP_LOGI("APP", "Project name: %s", app_desc->project_name);
    ESP_LOGI("APP", "Version: %s", app_desc->version);
    ESP_LOGI("APP", "IDF version: %s", app_desc->idf_ver);
}
```

Set versi di root `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(stunt_device VERSION 1.0.1)
```

Compare version sederhana:

```c
static int parse_version_part(const char **str)
{
    int value = 0;

    while (**str >= '0' && **str <= '9') {
        value = value * 10 + (**str - '0');
        (*str)++;
    }

    if (**str == '.') {
        (*str)++;
    }

    return value;
}

static bool is_version_newer(const char *server, const char *current)
{
    const char *s = server;
    const char *c = current;

    for (int i = 0; i < 3; i++) {
        int sv = parse_version_part(&s);
        int cv = parse_version_part(&c);

        if (sv > cv) return true;
        if (sv < cv) return false;
    }

    return false;
}
```

---

## 7. Parse Manifest JSON

```c
#include "cJSON.h"

typedef struct {
    char version[32];
    char url[256];
    char sha256[65];
    int min_battery_mv;
} firmware_manifest_t;

static bool parse_firmware_manifest(const char *json,
                                    firmware_manifest_t *manifest)
{
    cJSON *root = cJSON_Parse(json);

    if (root == NULL) {
        return false;
    }

    cJSON *version = cJSON_GetObjectItem(root, "version");
    cJSON *url = cJSON_GetObjectItem(root, "url");
    cJSON *sha256 = cJSON_GetObjectItem(root, "sha256");
    cJSON *min_battery_mv = cJSON_GetObjectItem(root, "min_battery_mv");

    if (!cJSON_IsString(version) || !cJSON_IsString(url)) {
        cJSON_Delete(root);
        return false;
    }

    snprintf(manifest->version, sizeof(manifest->version), "%s", version->valuestring);
    snprintf(manifest->url, sizeof(manifest->url), "%s", url->valuestring);

    if (cJSON_IsString(sha256)) {
        snprintf(manifest->sha256, sizeof(manifest->sha256), "%s", sha256->valuestring);
    }

    if (cJSON_IsNumber(min_battery_mv)) {
        manifest->min_battery_mv = min_battery_mv->valueint;
    }

    cJSON_Delete(root);
    return true;
}
```

---

## 8. Rollback

Rollback memastikan firmware baru harus membuktikan diri valid.

Aktifkan:

```bash
idf.py menuconfig
```

```text
Bootloader config -> Enable app rollback support
```

State OTA penting:

```text
ESP_OTA_IMG_NEW
ESP_OTA_IMG_PENDING_VERIFY
ESP_OTA_IMG_VALID
ESP_OTA_IMG_INVALID
ESP_OTA_IMG_ABORTED
```

Firmware baru pertama boot berada di `PENDING_VERIFY`. Aplikasi harus self-test.

```c
#include "esp_ota_ops.h"
#include "esp_log.h"

static void validate_new_firmware(void)
{
    const esp_partition_t *running = esp_ota_get_running_partition();

    esp_ota_img_states_t ota_state;

    esp_err_t ret = esp_ota_get_state_partition(running, &ota_state);

    if (ret == ESP_OK && ota_state == ESP_OTA_IMG_PENDING_VERIFY) {
        ESP_LOGW("OTA", "New firmware pending verification");

        bool self_test_ok = true;

        /*
         * Self-test contoh:
         * - NVS OK
         * - SPIFFS OK
         * - Sensor utama OK
         * - WiFi manager tidak crash
         * - Free heap cukup
         */

        if (self_test_ok) {
            ESP_LOGI("OTA", "Self-test OK, marking app valid");
            esp_ota_mark_app_valid_cancel_rollback();
        } else {
            ESP_LOGE("OTA", "Self-test failed, rollback and reboot");
            esp_ota_mark_app_invalid_rollback_and_reboot();
        }
    }
}
```

Panggil seawal mungkin di `app_main()`.

---

## 9. Progress OTA

Jika ingin progress, gunakan API bertahap:

```text
esp_https_ota_begin()
esp_https_ota_perform()
esp_https_ota_finish()
```

Konsep:

```c
while (1) {
    ret = esp_https_ota_perform(https_ota_handle);

    if (ret != ESP_ERR_HTTPS_OTA_IN_PROGRESS) {
        break;
    }

    int image_len = esp_https_ota_get_image_len_read(https_ota_handle);
    ESP_LOGI("OTA", "Downloaded %d bytes", image_len);
}
```

Manfaat:

- Menampilkan progress di LCD.
- Publish progress ke MQTT/WebSocket.
- Abort jika battery rendah.

---

## 10. Signed Firmware dan Anti-Rollback

OTA production sebaiknya:

```text
HTTPS + signed firmware + rollback
```

Signed firmware memastikan firmware palsu ditolak. Secure Boot memastikan hanya firmware sah yang bisa boot.

Anti-rollback mencegah downgrade ke firmware lama yang rentan.

Hati-hati:

```text
Anti-rollback berhubungan dengan eFuse.
Beberapa setting bersifat irreversible.
Jangan eksperimen di board utama.
```

---

## 11. OTA State Machine

```c
typedef enum {
    OTA_STATE_IDLE,
    OTA_STATE_CHECKING,
    OTA_STATE_DOWNLOADING,
    OTA_STATE_VERIFYING,
    OTA_STATE_SUCCESS,
    OTA_STATE_FAILED
} ota_state_t;
```

Alur:

```text
IDLE
↓
CHECKING manifest
↓
DOWNLOADING firmware
↓
VERIFYING
↓
SUCCESS -> reboot
```

Jika gagal:

```text
FAILED -> retry nanti
```

---

## 12. Mini Project Minggu 13

Project:

```text
week13_final_secure_ota_manager
```

Fitur wajib:

1. Custom partition OTA:
   - nvs.
   - otadata.
   - ota_0.
   - ota_1.
   - storage.
2. Print current app version.
3. Print running partition.
4. WiFi Station connect.
5. HTTP/HTTPS GET manifest.
6. Parse manifest JSON.
7. Compare server version vs current version.
8. Jika versi baru:
   - cek battery minimal jika ada.
   - jalankan HTTPS OTA.
9. Aktifkan rollback.
10. Firmware baru melakukan self-test.
11. Jika self-test sukses, mark app valid.
12. Jika self-test gagal, rollback and reboot.
13. OTA state machine.
14. Monitor task print app version, partition, OTA state, free heap.
15. OTA bisa dipicu periodic atau command.

---

## 13. Error Umum

### OTA Partition Tidak Ada

Solusi:

- Pastikan `otadata`, `ota_0`, `ota_1` ada.
- Aktifkan custom partition.
- `erase-flash` setelah ganti partition.

### Firmware Terlalu Besar

Solusi:

- Besarkan `ota_0` dan `ota_1`.
- Kurangi storage.
- Gunakan flash lebih besar.
- Optimasi size.

### OTA Download Gagal

Cek:

- WiFi connected.
- URL benar.
- DNS.
- TLS certificate.
- Timeout.
- Heap cukup.

### Bootloop Setelah OTA

Solusi:

- Aktifkan rollback.
- Self-test sebelum mark valid.
- Jangan ubah partition table sembarangan via OTA.

---

## 14. Latihan

1. Print running partition dan app version.
2. Buat partition OTA.
3. HTTP OTA lokal.
4. HTTPS OTA.
5. Version check via manifest.
6. Rollback test dengan firmware sengaja gagal self-test.
7. Mark valid setelah self-test.
8. OTA via MQTT command `OTA_CHECK`.

---

## 15. Checklist Lulus

- [ ] Menjelaskan OTA.
- [ ] Menjelaskan `ota_0`, `ota_1`, dan `otadata`.
- [ ] Membuat partition OTA.
- [ ] Menggunakan `esp_ota_get_running_partition()`.
- [ ] Menggunakan `esp_app_get_description()`.
- [ ] Menjalankan `esp_https_ota()`.
- [ ] Membuat firmware manifest.
- [ ] Membandingkan version.
- [ ] Mengaktifkan rollback.
- [ ] Mengecek `ESP_OTA_IMG_PENDING_VERIFY`.
- [ ] Memanggil `esp_ota_mark_app_valid_cancel_rollback()`.
- [ ] Memanggil `esp_ota_mark_app_invalid_rollback_and_reboot()`.
- [ ] Menjelaskan signed firmware.
- [ ] Menjelaskan anti-rollback.
- [ ] Membuat OTA manager state machine.
