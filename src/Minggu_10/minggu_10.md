# Minggu 10 — ESP-IDF BLE dan WiFi Provisioning

## Target Pembelajaran

Pada akhir Minggu 10, kamu mampu:

- Menjelaskan BLE, GAP, GATT, service, characteristic.
- Membuat ESP32 advertising sebagai BLE device.
- Membuat BLE GATT Server.
- Menerima command dari HP melalui BLE Write Characteristic.
- Mengirim status ke HP melalui BLE Notify.
- Memahami NimBLE vs Bluedroid.
- Menggunakan BLE WiFi Provisioning agar SSID/password tidak di-hardcode.
- Menyediakan reset provisioning untuk production device.

---

## 1. Kenapa BLE Penting untuk IoT?

Tanpa provisioning:

```text
SSID/password di-hardcode
Jika WiFi berubah, firmware harus di-flash ulang
```

Dengan BLE provisioning:

```text
Device boot
Belum ada WiFi config
Buka BLE provisioning
HP mengirim SSID/password
ESP32 simpan credential
ESP32 connect WiFi
```

BLE juga berguna untuk:

- Setup awal device.
- Command lokal jarak dekat.
- Kalibrasi sensor.
- Debug status.
- Reset WiFi.
- Maintenance mode.

---

## 2. Konsep BLE

BLE adalah Bluetooth Low Energy. BLE berbeda dari Bluetooth Classic.

Bluetooth Classic cocok untuk audio dan transfer data lebih besar. BLE cocok untuk sensor kecil, command ringan, provisioning, dan device low-power.

Istilah penting:

| Istilah | Arti |
|---|---|
| GAP | advertising, scanning, connection |
| GATT | struktur data setelah connected |
| Service | grup fungsi |
| Characteristic | data/endpoint dalam service |
| Read | client membaca data |
| Write | client menulis data |
| Notify | server mengirim update tanpa diminta |
| Peripheral | device yang advertising, biasanya ESP32 |
| Central | device yang connect, biasanya HP |

Struktur GATT:

```text
GATT Server
└── Service
    └── Characteristic
        └── Descriptor
```

Contoh:

```text
ESP32 GATT Server
└── Device Control Service
    ├── Command Characteristic: Write
    └── Status Characteristic: Read + Notify
```

---

## 3. NimBLE vs Bluedroid

ESP-IDF mendukung dua stack:

### Bluedroid

Cocok jika butuh Bluetooth Classic dan BLE.

### NimBLE

Cocok untuk BLE-only IoT karena lebih ringan.

Rekomendasi roadmap ini:

```text
Gunakan NimBLE untuk BLE-only IoT.
Gunakan WiFi Provisioning Manager untuk provisioning.
```

---

## 4. Desain GATT Custom

Service custom:

```text
Service UUID:
12345678-1234-1234-1234-1234567890ab

Command Characteristic UUID:
12345678-1234-1234-1234-1234567890ac
Property: Write

Status Characteristic UUID:
12345678-1234-1234-1234-1234567890ad
Property: Read + Notify
```

Command dari HP:

```text
LED_ON
LED_OFF
STATUS
RESET_WIFI
```

Status dari ESP32:

```json
{"led":"ON","wifi":"connected","heap":123456}
```

Pola benar:

```text
BLE Write Callback
   ↓
Command Queue
   ↓
Command Task
   ↓
GPIO/WiFi/Config action
   ↓
BLE Notify Status
```

---

## 5. Skeleton BLE GATT Server NimBLE

Header umum:

```c
#include "nvs_flash.h"
#include "esp_log.h"
#include "nimble/nimble_port.h"
#include "nimble/nimble_port_freertos.h"
#include "host/ble_hs.h"
#include "host/ble_uuid.h"
#include "host/ble_gatt.h"
#include "services/gap/ble_svc_gap.h"
#include "services/gatt/ble_svc_gatt.h"
```

Global dasar:

```c
#define LED_GPIO GPIO_NUM_2

static const char *TAG = "BLE_GATT";

static uint16_t status_char_handle;
static uint16_t connection_handle;
static bool ble_connected = false;
static uint8_t ble_addr_type;

typedef struct {
    char command[64];
} ble_command_msg_t;

static QueueHandle_t ble_command_queue = NULL;
```

Access callback:

```c
static int gatt_access_callback(uint16_t conn_handle,
                                uint16_t attr_handle,
                                struct ble_gatt_access_ctxt *ctxt,
                                void *arg)
{
    switch (ctxt->op) {
        case BLE_GATT_ACCESS_OP_WRITE_CHR: {
            ble_command_msg_t msg = {0};

            uint16_t len = OS_MBUF_PKTLEN(ctxt->om);

            if (len >= sizeof(msg.command)) {
                len = sizeof(msg.command) - 1;
            }

            int rc = ble_hs_mbuf_to_flat(
                ctxt->om,
                msg.command,
                len,
                NULL
            );

            if (rc != 0) {
                return BLE_ATT_ERR_UNLIKELY;
            }

            msg.command[len] = '\0';

            ESP_LOGI(TAG, "BLE command received: %s", msg.command);

            if (ble_command_queue != NULL) {
                xQueueSend(ble_command_queue, &msg, 0);
            }

            return 0;
        }

        case BLE_GATT_ACCESS_OP_READ_CHR: {
            const char *status = "{\"status\":\"ok\"}";
            os_mbuf_append(ctxt->om, status, strlen(status));
            return 0;
        }

        default:
            return BLE_ATT_ERR_UNLIKELY;
    }
}
```

Command task:

```c
static void ble_command_task(void *pvParameters)
{
    ble_command_msg_t msg;

    while (1) {
        if (xQueueReceive(ble_command_queue, &msg, portMAX_DELAY) == pdTRUE) {
            if (strcmp(msg.command, "LED_ON") == 0) {
                gpio_set_level(LED_GPIO, 1);
            } else if (strcmp(msg.command, "LED_OFF") == 0) {
                gpio_set_level(LED_GPIO, 0);
            } else if (strcmp(msg.command, "STATUS") == 0) {
                ESP_LOGI("BLE_CMD", "STATUS requested");
            } else {
                ESP_LOGW("BLE_CMD", "Unknown command: %s", msg.command);
            }
        }
    }
}
```

---

## 6. BLE Notify

Notify memungkinkan ESP32 mengirim update status ke HP.

```c
static void ble_notify_status(const char *status_json)
{
    if (!ble_connected) {
        ESP_LOGW("BLE", "BLE not connected, skip notify");
        return;
    }

    struct os_mbuf *om = ble_hs_mbuf_from_flat(
        status_json,
        strlen(status_json)
    );

    if (om == NULL) {
        return;
    }

    int rc = ble_gatts_notify_custom(
        connection_handle,
        status_char_handle,
        om
    );

    if (rc != 0) {
        ESP_LOGW("BLE", "Notify failed rc=%d", rc);
    }
}
```

Status task:

```c
static void ble_status_task(void *pvParameters)
{
    while (1) {
        char status[128];

        snprintf(status, sizeof(status),
                 "{\"heap\":%lu,\"led\":\"%s\"}",
                 esp_get_free_heap_size(),
                 gpio_get_level(LED_GPIO) ? "ON" : "OFF");

        ble_notify_status(status);

        vTaskDelay(pdMS_TO_TICKS(3000));
    }
}
```

---

## 7. BLE WiFi Provisioning

WiFi provisioning adalah proses mengirim SSID/password ke ESP32.

Alur:

```text
ESP32 belum provisioned
↓
Start BLE provisioning service
↓
HP connect pakai ESP BLE Provisioning app
↓
HP scan WiFi dan kirim credential
↓
ESP32 connect WiFi
↓
Credential disimpan
```

Header umum:

```c
#include "wifi_provisioning/manager.h"
#include "wifi_provisioning/scheme_ble.h"
```

Skeleton provisioning:

```c
static void start_ble_provisioning(void)
{
    wifi_prov_mgr_config_t config = {
        .scheme = wifi_prov_scheme_ble,
        .scheme_event_handler = WIFI_PROV_SCHEME_BLE_EVENT_HANDLER_FREE_BTDM
    };

    ESP_ERROR_CHECK(wifi_prov_mgr_init(config));

    bool provisioned = false;
    ESP_ERROR_CHECK(wifi_prov_mgr_is_provisioned(&provisioned));

    if (!provisioned) {
        const char *service_name = "PROV_ESP32";
        const char *service_key = NULL;
        const char *pop = "abcd1234";

        wifi_prov_security_t security = WIFI_PROV_SECURITY_1;

        wifi_prov_mgr_start_provisioning(
            security,
            pop,
            service_name,
            service_key
        );

        ESP_LOGI("PROV", "Provisioning started: %s", service_name);
    } else {
        wifi_prov_mgr_deinit();
        ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
        ESP_ERROR_CHECK(esp_wifi_start());
    }
}
```

Aplikasi untuk test:

```text
ESP BLE Provisioning
nRF Connect untuk GATT custom
```

---

## 8. Reset Provisioning

Saat development:

```bash
idf.py erase-flash
```

Untuk production, buat long press button:

```c
esp_wifi_restore();
esp_restart();
```

Atau hapus namespace config sendiri jika credential disimpan manual.

---

## 9. Mini Project Minggu 10

Project:

```text
week10_final_ble_wifi_provisioning_control
```

Fitur wajib:

1. NVS init.
2. Cek apakah device sudah provisioned.
3. Jika belum: start BLE WiFi provisioning dengan PoP.
4. Jika sudah: start WiFi Station.
5. Custom BLE GATT service:
   - command characteristic WRITE.
   - status characteristic READ + NOTIFY.
6. Command BLE:
   - `LED_ON`
   - `LED_OFF`
   - `STATUS`
   - `RESET_WIFI`
7. BLE callback hanya mengirim command ke queue.
8. Command task memproses command.
9. Status task notify heap, WiFi status, LED status.
10. Long press button untuk reset provisioning.

---

## 10. Latihan

1. Buat ESP32 muncul di nRF Connect.
2. Buat GATT service dengan status read.
3. Buat write characteristic untuk `LED_ON` dan `LED_OFF`.
4. Buat notify status setiap 3 detik.
5. Buat command queue.
6. Jalankan BLE WiFi Provisioning.
7. Buat reset provisioning dengan tombol.

---

## 11. Checklist Lulus

- [ ] Menjelaskan BLE vs Bluetooth Classic.
- [ ] Menjelaskan GAP dan GATT.
- [ ] Menjelaskan service dan characteristic.
- [ ] Menjelaskan read/write/notify.
- [ ] Menjelaskan peripheral dan central.
- [ ] Menjelaskan NimBLE vs Bluedroid.
- [ ] Membuat ESP32 advertising.
- [ ] Membuat GATT service.
- [ ] Membuat write characteristic.
- [ ] Membuat notify characteristic.
- [ ] Mengirim BLE command ke queue.
- [ ] Menggunakan BLE WiFi Provisioning.
- [ ] Menggunakan PoP.
- [ ] Reset provisioning.
