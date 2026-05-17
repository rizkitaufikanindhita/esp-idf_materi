# Minggu 8 — ESP-IDF WiFi, HTTP Client/Server, dan Dasar Konektivitas IoT

## Target Minggu 8

Setelah menyelesaikan minggu ini, kamu mampu:

- Membuat ESP32 terhubung ke WiFi sebagai **Station**.
- Memahami event WiFi dan IP pada ESP-IDF.
- Membuat logic reconnect yang aman.
- Melakukan HTTP GET dan HTTP POST dari ESP32.
- Mengirim payload JSON ke backend.
- Membuat HTTP server lokal di ESP32.
- Memahami SoftAP untuk konfigurasi lokal.
- Menggabungkan konektivitas dengan offline queue dari Minggu 7.

Gambaran besar:

```txt
ESP32 boot
  ↓
Load config dari NVS
  ↓
Connect WiFi
  ↓
Jika WiFi berhasil:
    kirim data ke server
    sync offline queue
  ↓
Jika WiFi gagal:
    tetap baca sensor
    simpan data ke offline storage
```

---

## 1. Gambaran Belajar Minggu 8

| Hari | Fokus | Output |
|---|---|---|
| Hari 1 | Konsep WiFi ESP-IDF | Paham STA, AP, event, IP |
| Hari 2 | WiFi Station Mode | ESP32 connect ke router |
| Hari 3 | Reconnect dan Event Group | WiFi lebih robust |
| Hari 4 | HTTP Client GET/POST | Kirim data ke backend |
| Hari 5 | JSON Payload | Kirim data sensor format JSON |
| Hari 6 | HTTP Server / SoftAP | ESP32 punya endpoint lokal |
| Hari 7 | Mini project | WiFi + HTTP POST + offline sync |

---

## 2. Komponen Konektivitas ESP-IDF

Komponen penting:

```txt
esp_wifi        → driver WiFi
esp_event       → event system
esp_netif       → network interface
lwIP            → TCP/IP stack
esp_http_client → HTTP/HTTPS client
esp_http_server → HTTP server ringan di ESP32
```

Untuk WiFi Station, biasanya kamu memakai:

```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"
#include "esp_log.h"
#include "freertos/event_groups.h"
```

---

## 3. Mode WiFi

### 3.1 Station Mode

Station mode artinya ESP32 menjadi client WiFi.

```txt
Router / Hotspot
      ↑
    WiFi
      ↓
    ESP32
```

Dipakai untuk:

- Kirim data sensor ke backend.
- MQTT.
- OTA update.
- HTTP request.
- Sinkronisasi waktu NTP.

---

### 3.2 SoftAP Mode

SoftAP mode artinya ESP32 menjadi access point.

```txt
Laptop / HP
      ↓
Connect ke WiFi ESP32
      ↓
ESP32 SoftAP
```

Dipakai untuk:

- Provisioning WiFi.
- Setup awal device.
- Local configuration page.
- Maintenance mode.
- Direct local control.

Contoh:

```txt
SSID: STUNTING-DEVICE-001
IP default SoftAP: 192.168.4.1
```

---

### 3.3 Station + SoftAP

ESP32 juga bisa menjalankan Station dan SoftAP bersamaan.

```txt
ESP32 connect ke router
+
ESP32 membuka access point lokal
```

Dipakai untuk device yang tetap online ke cloud tetapi masih bisa dikonfigurasi lokal.

---

## 4. Event Penting WiFi

Event yang sering dipakai:

| Event | Arti |
|---|---|
| `WIFI_EVENT_STA_START` | WiFi station mulai |
| `WIFI_EVENT_STA_CONNECTED` | ESP32 tersambung ke AP |
| `WIFI_EVENT_STA_DISCONNECTED` | ESP32 terputus |
| `IP_EVENT_STA_GOT_IP` | ESP32 mendapat IP dari DHCP |

Event paling penting untuk mulai HTTP/MQTT:

```txt
IP_EVENT_STA_GOT_IP
```

Karena setelah mendapat IP, barulah aman melakukan request jaringan.

---

## 5. Event Group untuk Status WiFi

Biasanya kita memakai Event Group:

```c
#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1
```

Tujuannya:

```txt
WiFi connected + got IP
  ↓
set WIFI_CONNECTED_BIT

WiFi disconnected
  ↓
clear WIFI_CONNECTED_BIT
```

Task lain bisa mengecek:

```c
EventBits_t bits = xEventGroupGetBits(wifi_event_group);

if (bits & WIFI_CONNECTED_BIT) {
    // boleh HTTP/MQTT
}
```

---

## 6. Project 1 — WiFi Station Basic

Nama project:

```txt
week8_01_wifi_station
```

Ganti bagian ini:

```c
#define WIFI_SSID "YOUR_WIFI_SSID"
#define WIFI_PASS "YOUR_WIFI_PASSWORD"
```

### Full Code

```c
#include <stdio.h>
#include <string.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"

#include "esp_log.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"

#define WIFI_SSID "YOUR_WIFI_SSID"
#define WIFI_PASS "YOUR_WIFI_PASSWORD"

#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

#define MAX_RETRY 5

static const char *TAG = "WIFI_STA";

static EventGroupHandle_t wifi_event_group;
static int retry_count = 0;

static void nvs_init_storage(void)
{
    esp_err_t ret = nvs_flash_init();

    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ESP_ERROR_CHECK(nvs_flash_init());
    } else {
        ESP_ERROR_CHECK(ret);
    }
}

static void wifi_event_handler(void *arg,
                               esp_event_base_t event_base,
                               int32_t event_id,
                               void *event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        ESP_LOGI(TAG, "WiFi started, connecting...");
        esp_wifi_connect();

    } else if (event_base == WIFI_EVENT &&
               event_id == WIFI_EVENT_STA_DISCONNECTED) {

        xEventGroupClearBits(wifi_event_group, WIFI_CONNECTED_BIT);

        if (retry_count < MAX_RETRY) {
            esp_wifi_connect();
            retry_count++;
            ESP_LOGW(TAG, "Retry WiFi connection: %d/%d", retry_count, MAX_RETRY);
        } else {
            xEventGroupSetBits(wifi_event_group, WIFI_FAIL_BIT);
            ESP_LOGE(TAG, "WiFi connection failed");
        }

    } else if (event_base == IP_EVENT &&
               event_id == IP_EVENT_STA_GOT_IP) {

        ip_event_got_ip_t *event = (ip_event_got_ip_t *) event_data;

        ESP_LOGI(TAG, "Got IP: " IPSTR, IP2STR(&event->ip_info.ip));

        retry_count = 0;
        xEventGroupSetBits(wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

static esp_err_t wifi_init_sta(void)
{
    wifi_event_group = xEventGroupCreate();

    if (wifi_event_group == NULL) {
        ESP_LOGE(TAG, "Failed to create WiFi event group");
        return ESP_ERR_NO_MEM;
    }

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    esp_netif_create_default_wifi_sta();

    wifi_init_config_t wifi_init_config = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&wifi_init_config));

    ESP_ERROR_CHECK(esp_event_handler_register(
        WIFI_EVENT,
        ESP_EVENT_ANY_ID,
        &wifi_event_handler,
        NULL
    ));

    ESP_ERROR_CHECK(esp_event_handler_register(
        IP_EVENT,
        IP_EVENT_STA_GOT_IP,
        &wifi_event_handler,
        NULL
    ));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
        },
    };

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "WiFi station initialized");

    EventBits_t bits = xEventGroupWaitBits(
        wifi_event_group,
        WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
        pdFALSE,
        pdFALSE,
        portMAX_DELAY
    );

    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "Connected to SSID: %s", WIFI_SSID);
        return ESP_OK;
    }

    if (bits & WIFI_FAIL_BIT) {
        ESP_LOGE(TAG, "Failed to connect to SSID: %s", WIFI_SSID);
        return ESP_FAIL;
    }

    ESP_LOGE(TAG, "Unexpected WiFi event");
    return ESP_FAIL;
}

void app_main(void)
{
    ESP_LOGI(TAG, "WiFi Station example started");

    nvs_init_storage();

    esp_err_t ret = wifi_init_sta();

    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "WiFi ready");
    } else {
        ESP_LOGE(TAG, "WiFi not ready");
    }

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 7. Penjelasan WiFi Station

### 7.1 Init Network Stack

```c
ESP_ERROR_CHECK(esp_netif_init());
ESP_ERROR_CHECK(esp_event_loop_create_default());
```

`esp_netif_init()` menyiapkan network interface layer.

`esp_event_loop_create_default()` membuat default event loop untuk event WiFi/IP.

---

### 7.2 Buat Interface Station

```c
esp_netif_create_default_wifi_sta();
```

Ini membuat network interface default untuk WiFi Station.

---

### 7.3 Init WiFi Driver

```c
wifi_init_config_t wifi_init_config = WIFI_INIT_CONFIG_DEFAULT();
ESP_ERROR_CHECK(esp_wifi_init(&wifi_init_config));
```

`WIFI_INIT_CONFIG_DEFAULT()` memberi konfigurasi default yang aman.

---

### 7.4 Register Event Handler

```c
esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL);
esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, NULL);
```

Handler ini dipanggil saat event WiFi/IP terjadi.

---

### 7.5 Start WiFi

```c
esp_wifi_set_mode(WIFI_MODE_STA);
esp_wifi_set_config(WIFI_IF_STA, &wifi_config);
esp_wifi_start();
```

Setelah `esp_wifi_start()`, event `WIFI_EVENT_STA_START` muncul, lalu handler memanggil:

```c
esp_wifi_connect();
```

---

## 8. WiFi Reconnect yang Lebih Robust

Kode basic berhenti retry setelah `MAX_RETRY`. Untuk IoT production, lebih baik:

```txt
1. Retry cepat beberapa kali.
2. Kalau gagal, masuk offline mode.
3. Tetap jalankan sensor/storage.
4. Retry lagi setelah interval tertentu.
```

Jangan sampai device tidak berfungsi hanya karena WiFi mati.

Pola production:

```txt
WiFi gagal
  ↓
clear WIFI_CONNECTED_BIT
  ↓
network task tidak kirim HTTP
  ↓
payload masuk offline queue
  ↓
retry WiFi berkala
```

---

## 9. Jangan HTTP Sebelum Got IP

Kesalahan umum:

```txt
WiFi started
  ↓
langsung HTTP POST
  ↓
gagal karena belum dapat IP
```

Yang benar:

```txt
Tunggu IP_EVENT_STA_GOT_IP
  ↓
set WIFI_CONNECTED_BIT
  ↓
Network task boleh HTTP POST
```

---

## 10. HTTP Client GET

Project:

```txt
week8_02_http_get
```

### Kode HTTP GET

```c
#include <stdio.h>
#include <string.h>

#include "esp_log.h"
#include "esp_http_client.h"

static const char *TAG = "HTTP_GET";

static esp_err_t http_event_handler(esp_http_client_event_t *evt)
{
    switch (evt->event_id) {
        case HTTP_EVENT_ERROR:
            ESP_LOGW(TAG, "HTTP_EVENT_ERROR");
            break;

        case HTTP_EVENT_ON_CONNECTED:
            ESP_LOGI(TAG, "HTTP_EVENT_ON_CONNECTED");
            break;

        case HTTP_EVENT_HEADER_SENT:
            ESP_LOGI(TAG, "HTTP_EVENT_HEADER_SENT");
            break;

        case HTTP_EVENT_ON_HEADER:
            ESP_LOGI(TAG, "HEADER: %.*s: %.*s",
                     evt->header_key_len,
                     evt->header_key,
                     evt->header_value_len,
                     evt->header_value);
            break;

        case HTTP_EVENT_ON_DATA:
            ESP_LOGI(TAG, "DATA len=%d", evt->data_len);

            if (!esp_http_client_is_chunked_response(evt->client)) {
                ESP_LOGI(TAG, "%.*s", evt->data_len, (char *)evt->data);
            }
            break;

        case HTTP_EVENT_ON_FINISH:
            ESP_LOGI(TAG, "HTTP_EVENT_ON_FINISH");
            break;

        case HTTP_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "HTTP_EVENT_DISCONNECTED");
            break;

        default:
            break;
    }

    return ESP_OK;
}

esp_err_t app_http_get(const char *url)
{
    esp_http_client_config_t config = {
        .url = url,
        .event_handler = http_event_handler,
        .timeout_ms = 5000,
    };

    esp_http_client_handle_t client = esp_http_client_init(&config);

    if (client == NULL) {
        ESP_LOGE(TAG, "Failed to init HTTP client");
        return ESP_FAIL;
    }

    esp_err_t ret = esp_http_client_perform(client);

    if (ret == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);
        int content_length = esp_http_client_get_content_length(client);

        ESP_LOGI(TAG,
                 "HTTP GET status=%d, content_length=%d",
                 status_code,
                 content_length);
    } else {
        ESP_LOGE(TAG, "HTTP GET failed: %s", esp_err_to_name(ret));
    }

    esp_http_client_cleanup(client);

    return ret;
}
```

Pemakaian setelah WiFi connected:

```c
app_http_get("http://httpbin.org/get");
```

---

## 11. HTTP Client POST JSON

HTTP POST dipakai untuk mengirim data ke backend.

Contoh endpoint:

```txt
http://192.168.1.10:3000/api/sensor
```

Contoh payload:

```json
{
  "device_id": "ESP32-001",
  "temperature": 30.5,
  "humidity": 70.2
}
```

### Kode HTTP POST JSON

```c
#include <stdio.h>
#include <string.h>

#include "esp_log.h"
#include "esp_http_client.h"

static const char *TAG = "HTTP_POST";

esp_err_t app_http_post_json(const char *url, const char *json_payload)
{
    esp_http_client_config_t config = {
        .url = url,
        .method = HTTP_METHOD_POST,
        .timeout_ms = 5000,
    };

    esp_http_client_handle_t client = esp_http_client_init(&config);

    if (client == NULL) {
        ESP_LOGE(TAG, "Failed to init HTTP client");
        return ESP_FAIL;
    }

    esp_http_client_set_header(client, "Content-Type", "application/json");
    esp_http_client_set_post_field(client, json_payload, strlen(json_payload));

    esp_err_t ret = esp_http_client_perform(client);

    if (ret == ESP_OK) {
        int status_code = esp_http_client_get_status_code(client);

        ESP_LOGI(TAG, "HTTP POST status=%d", status_code);

        if (status_code >= 200 && status_code < 300) {
            ret = ESP_OK;
        } else {
            ret = ESP_FAIL;
        }
    } else {
        ESP_LOGE(TAG, "HTTP POST failed: %s", esp_err_to_name(ret));
    }

    esp_http_client_cleanup(client);

    return ret;
}
```

Pemakaian:

```c
const char *payload =
    "{\"device_id\":\"ESP32-001\",\"temperature\":30.5,\"humidity\":70.2}";

app_http_post_json("http://192.168.1.10:3000/api/sensor", payload);
```

---

## 12. JSON Payload dengan cJSON

ESP-IDF menyertakan `cJSON`.

### Membuat JSON Sensor

```c
#include "cJSON.h"

static char *create_sensor_json(const char *device_id,
                                float temperature,
                                float humidity,
                                int64_t timestamp)
{
    cJSON *root = cJSON_CreateObject();

    if (root == NULL) {
        return NULL;
    }

    cJSON_AddStringToObject(root, "device_id", device_id);
    cJSON_AddNumberToObject(root, "temperature", temperature);
    cJSON_AddNumberToObject(root, "humidity", humidity);
    cJSON_AddNumberToObject(root, "timestamp", timestamp);

    char *json_string = cJSON_PrintUnformatted(root);

    cJSON_Delete(root);

    return json_string;
}
```

Pemakaian:

```c
char *payload = create_sensor_json("ESP32-001", 30.5, 70.2, 123456);

if (payload != NULL) {
    app_http_post_json("http://192.168.1.10:3000/api/sensor", payload);
    free(payload);
}
```

Penting:

```txt
cJSON_PrintUnformatted() mengalokasikan memory.
Setelah selesai, panggil free(payload).
```

---

## 13. Payload untuk Project Stunting

Contoh payload:

```json
{
  "device_id": "STUNT-001",
  "uid": "A1B2C3D4",
  "age_month": 24,
  "height_cm": 82.5,
  "status": "risk",
  "firmware_version": "1.0.0"
}
```

Fungsi builder:

```c
static char *create_stunting_payload(const char *device_id,
                                     const char *uid,
                                     int age_month,
                                     float height_cm,
                                     const char *status,
                                     const char *firmware_version)
{
    cJSON *root = cJSON_CreateObject();

    if (root == NULL) {
        return NULL;
    }

    cJSON_AddStringToObject(root, "device_id", device_id);
    cJSON_AddStringToObject(root, "uid", uid);
    cJSON_AddNumberToObject(root, "age_month", age_month);
    cJSON_AddNumberToObject(root, "height_cm", height_cm);
    cJSON_AddStringToObject(root, "status", status);
    cJSON_AddStringToObject(root, "firmware_version", firmware_version);

    char *json = cJSON_PrintUnformatted(root);

    cJSON_Delete(root);

    return json;
}
```

---

## 14. HTTP Server ESP32

Use case:

- Local status page.
- WiFi provisioning sederhana.
- Local API endpoint.
- Debug endpoint.
- Factory test page.

Project:

```txt
week8_03_http_server
```

### Kode HTTP Server Basic

```c
#include <stdio.h>
#include <string.h>

#include "esp_log.h"
#include "esp_http_server.h"

static const char *TAG = "HTTP_SERVER";

static esp_err_t root_get_handler(httpd_req_t *req)
{
    const char *response =
        "<!DOCTYPE html>"
        "<html>"
        "<head><title>ESP32</title></head>"
        "<body>"
        "<h1>ESP32 HTTP Server</h1>"
        "<p>Status: OK</p>"
        "</body>"
        "</html>";

    httpd_resp_set_type(req, "text/html");
    httpd_resp_send(req, response, HTTPD_RESP_USE_STRLEN);

    return ESP_OK;
}

static esp_err_t status_get_handler(httpd_req_t *req)
{
    const char *json =
        "{\"device_id\":\"ESP32-001\",\"status\":\"ok\",\"wifi\":\"connected\"}";

    httpd_resp_set_type(req, "application/json");
    httpd_resp_send(req, json, HTTPD_RESP_USE_STRLEN);

    return ESP_OK;
}

httpd_handle_t app_http_server_start(void)
{
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();

    httpd_handle_t server = NULL;

    esp_err_t ret = httpd_start(&server, &config);

    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to start HTTP server: %s", esp_err_to_name(ret));
        return NULL;
    }

    httpd_uri_t root_uri = {
        .uri = "/",
        .method = HTTP_GET,
        .handler = root_get_handler,
        .user_ctx = NULL
    };

    httpd_uri_t status_uri = {
        .uri = "/api/status",
        .method = HTTP_GET,
        .handler = status_get_handler,
        .user_ctx = NULL
    };

    httpd_register_uri_handler(server, &root_uri);
    httpd_register_uri_handler(server, &status_uri);

    ESP_LOGI(TAG, "HTTP server started");

    return server;
}
```

Setelah WiFi connected, panggil:

```c
app_http_server_start();
```

Lalu buka dari browser:

```txt
http://IP_ESP32/
http://IP_ESP32/api/status
```

---

## 15. SoftAP untuk Setup Lokal

### Kapan Butuh SoftAP?

Misalnya device baru belum tahu SSID/password WiFi rumah/kantor.

Alur provisioning sederhana:

```txt
1. ESP32 nyala.
2. Cek NVS, belum ada WiFi config.
3. ESP32 membuat SoftAP: STUNT-SETUP-001.
4. User connect HP/laptop ke SoftAP.
5. User buka halaman config.
6. User submit SSID/password.
7. ESP32 simpan ke NVS.
8. ESP32 restart/connect sebagai Station.
```

---

### Kode SoftAP Basic

```c
#include <stdio.h>
#include <string.h>

#include "esp_log.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"

#define AP_SSID      "ESP32-SETUP"
#define AP_PASSWORD  "12345678"
#define AP_CHANNEL   1
#define AP_MAX_CONN  4

static const char *TAG = "WIFI_AP";

static void wifi_init_softap(void)
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    esp_netif_create_default_wifi_ap();

    wifi_init_config_t wifi_init_config = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&wifi_init_config));

    wifi_config_t wifi_config = {
        .ap = {
            .ssid = AP_SSID,
            .ssid_len = strlen(AP_SSID),
            .channel = AP_CHANNEL,
            .password = AP_PASSWORD,
            .max_connection = AP_MAX_CONN,
            .authmode = WIFI_AUTH_WPA_WPA2_PSK,
        },
    };

    if (strlen(AP_PASSWORD) == 0) {
        wifi_config.ap.authmode = WIFI_AUTH_OPEN;
    }

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "SoftAP started. SSID=%s password=%s", AP_SSID, AP_PASSWORD);
}
```

Default IP SoftAP biasanya:

```txt
192.168.4.1
```

---

## 16. Mini Project Minggu 8 — WiFi HTTP Offline Sync

Nama project:

```txt
week8_final_wifi_http_offline_sync
```

Fitur:

```txt
1. WiFi Station connect ke router.
2. Event Group menyimpan status WiFi connected.
3. Payload generator membuat dummy JSON setiap 10 detik.
4. Network task mencoba HTTP POST.
5. Jika HTTP POST sukses → selesai.
6. Jika gagal / WiFi offline → simpan ke offline queue SPIFFS.
7. Sync task mencoba kirim offline queue saat WiFi connected.
8. Monitor task print WiFi status, heap, queue count.
```

Arsitektur:

```txt
Payload Task
   ↓ queue
Network Task
   ├── WiFi connected → HTTP POST
   └── WiFi offline/fail → SPIFFS offline queue

Sync Task
   ├── wait WiFi connected
   └── sync offline queue

Monitor Task
   └── status
```

---

## 17. Tipe Data Payload

```c
typedef struct {
    char json[256];
} payload_msg_t;
```

---

## 18. Payload Generator Task

```c
static void payload_generator_task(void *pvParameters)
{
    int counter = 0;

    while (1) {
        payload_msg_t msg;

        snprintf(msg.json, sizeof(msg.json),
                 "{\"device_id\":\"ESP32-001\",\"counter\":%d,\"value\":%.2f}",
                 counter,
                 30.0f + (counter % 10));

        if (xQueueSend(payload_queue, &msg, pdMS_TO_TICKS(100)) != pdTRUE) {
            ESP_LOGW("PAYLOAD", "Payload queue full");
        } else {
            ESP_LOGI("PAYLOAD", "Payload generated: %s", msg.json);
        }

        counter++;

        vTaskDelay(pdMS_TO_TICKS(10000));
    }
}
```

---

## 19. Network Task

```c
static void network_task(void *pvParameters)
{
    payload_msg_t msg;

    while (1) {
        if (xQueueReceive(payload_queue, &msg, portMAX_DELAY) == pdTRUE) {
            EventBits_t bits = xEventGroupGetBits(wifi_event_group);

            if (bits & WIFI_CONNECTED_BIT) {
                esp_err_t ret = app_http_post_json(SERVER_URL, msg.json);

                if (ret == ESP_OK) {
                    ESP_LOGI("NETWORK", "Payload sent successfully");
                } else {
                    ESP_LOGW("NETWORK", "POST failed, saving offline");
                    offline_queue_append(msg.json);
                }
            } else {
                ESP_LOGW("NETWORK", "WiFi offline, saving offline");
                offline_queue_append(msg.json);
            }
        }
    }
}
```

---

## 20. Sync Task Konsep

Strategi sederhana:

```txt
1. Tunggu WiFi connected.
2. Baca offline_queue.txt.
3. Kirim satu per satu.
4. Kalau sukses semua, hapus file.
5. Kalau ada yang gagal, simpan ulang sisanya.
```

Pseudo-code:

```c
static void offline_sync_task(void *pvParameters)
{
    while (1) {
        xEventGroupWaitBits(
            wifi_event_group,
            WIFI_CONNECTED_BIT,
            pdFALSE,
            pdTRUE,
            portMAX_DELAY
        );

        ESP_LOGI("SYNC", "WiFi connected, trying offline sync");

        offline_queue_sync_to_server(SERVER_URL);

        vTaskDelay(pdMS_TO_TICKS(30000));
    }
}
```

---

## 21. Contoh Backend Express untuk Test

```js
import express from "express";

const app = express();

app.use(express.json());

app.post("/api/sensor", (req, res) => {
  console.log("Payload received:", req.body);

  res.json({
    ok: true,
    message: "Payload received",
  });
});

app.get("/api/status", (req, res) => {
  res.json({
    ok: true,
    server: "running",
  });
});

app.listen(3000, "0.0.0.0", () => {
  console.log("Server running on port 3000");
});
```

Jalankan:

```bash
node server.js
```

Cari IP laptop:

```txt
Windows: ipconfig
Linux/macOS: ifconfig / ip addr
```

Misal IP laptop:

```txt
192.168.1.10
```

Maka ESP32 POST ke:

```txt
http://192.168.1.10:3000/api/sensor
```

Catatan:

```txt
ESP32 dan laptop harus berada di jaringan WiFi yang sama.
Firewall laptop harus mengizinkan port 3000.
Backend harus listen di 0.0.0.0, bukan localhost saja.
```

---

## 22. Struktur File yang Direkomendasikan

```txt
week8_final_wifi_http_offline_sync/
├── CMakeLists.txt
├── partitions.csv
└── main/
    ├── CMakeLists.txt
    ├── main.c
    ├── app_wifi.c
    ├── app_wifi.h
    ├── app_http_client.c
    ├── app_http_client.h
    ├── app_http_server.c
    ├── app_http_server.h
    ├── app_nvs.c
    ├── app_nvs.h
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
#include "app_fs.h"
#include "app_wifi.h"
#include "app_http_server.h"
#include "app_tasks.h"

static const char *TAG = "MAIN";

void app_main(void)
{
    ESP_LOGI(TAG, "Week 8 app started");

    app_nvs_init();
    app_fs_init();
    app_wifi_start_sta();
    app_http_server_start();
    app_tasks_start();
}
```

---

## 23. Error Umum WiFi

### 23.1 Tidak Bisa Connect

Cek:

```txt
1. SSID salah.
2. Password salah.
3. Router 5 GHz only, ESP32 umumnya butuh 2.4 GHz.
4. Sinyal lemah.
5. Auth mode tidak cocok.
6. MAC filtering router.
7. NVS belum init.
```

---

### 23.2 Sudah Connected tapi HTTP Gagal

Cek:

```txt
1. Belum dapat IP.
2. Server URL salah.
3. Laptop dan ESP32 beda jaringan.
4. Firewall laptop menutup port.
5. Backend hanya listen di localhost.
6. HTTP timeout terlalu pendek.
7. DNS gagal.
```

Kalau backend di laptop, pastikan:

```js
app.listen(3000, "0.0.0.0")
```

Bukan hanya:

```js
app.listen(3000)
```

---

### 23.3 `esp_event_loop_create_default()` Error

Kalau dipanggil dua kali, bisa error:

```txt
ESP_ERR_INVALID_STATE
```

Solusi:

```txt
Pastikan hanya dibuat sekali.
Atau handle error tersebut dengan sadar.
```

---

### 23.4 SoftAP Bisa Connect tapi Halaman Tidak Terbuka

Cek:

```txt
1. HTTP server belum start.
2. IP salah, coba 192.168.4.1.
3. Handler URI belum diregister.
4. Device belum benar-benar connect ke AP ESP32.
```

---

## 24. Best Practice Minggu 8

### 24.1 Jangan Hardcode WiFi untuk Production

Untuk latihan boleh:

```c
#define WIFI_SSID "..."
#define WIFI_PASS "..."
```

Untuk production:

```txt
1. Simpan WiFi config di NVS.
2. Jika belum ada config, masuk SoftAP/BLE provisioning.
3. Jika WiFi gagal, tetap offline mode.
```

---

### 24.2 Selalu Event-Driven

Baik:

```txt
WiFi event handler → set Event Group
Network task → cek/wait Event Group
```

Buruk:

```txt
Task menebak-nebak status WiFi dari delay acak.
```

---

### 24.3 Jangan Kirim HTTP dari Event Handler

Buruk:

```txt
IP_EVENT_STA_GOT_IP handler langsung HTTP POST besar.
```

Baik:

```txt
IP_EVENT_STA_GOT_IP handler set WIFI_CONNECTED_BIT.
Network task yang melakukan HTTP.
```

---

### 24.4 Gunakan Timeout

HTTP harus punya timeout.

```c
.timeout_ms = 5000
```

Jangan biarkan network task menggantung terlalu lama.

---

### 24.5 Offline-First

Device IoT production sebaiknya berpikir:

```txt
Internet adalah bonus.
Data tidak boleh hilang saat internet mati.
```

Pola:

```txt
HTTP sukses → selesai
HTTP gagal  → simpan offline
WiFi balik  → sync ulang
```

---

## 25. Latihan Minggu 8

### Latihan 1 — WiFi Station

Buat ESP32 connect ke WiFi dan print IP.

Target output:

```txt
Got IP: 192.168.1.xxx
```

---

### Latihan 2 — WiFi Reconnect

Matikan hotspot/router, lalu nyalakan lagi.

Target:

```txt
ESP32 reconnect otomatis.
```

---

### Latihan 3 — HTTP GET

Setelah WiFi connected, lakukan GET ke endpoint test.

Target:

```txt
HTTP GET status=200
```

---

### Latihan 4 — HTTP POST JSON

Kirim JSON sederhana ke backend Express.

Payload:

```json
{
  "device_id": "ESP32-001",
  "value": 123
}
```

---

### Latihan 5 — cJSON

Buat JSON memakai cJSON, bukan string manual.

Target:

```txt
Payload valid dan bisa diterima backend.
```

---

### Latihan 6 — HTTP Server

Buat endpoint:

```txt
GET /
GET /api/status
```

Target:

```txt
Browser bisa membuka status ESP32.
```

---

### Latihan 7 — SoftAP

Buat ESP32 menjadi access point:

```txt
SSID: ESP32-SETUP
Password: 12345678
```

Lalu akses:

```txt
http://192.168.4.1/
```

---

### Latihan 8 — Offline Sync

Simulasikan server mati.

Target:

```txt
1. Payload gagal dikirim.
2. Payload masuk offline_queue.txt.
3. Server hidup lagi.
4. Payload offline dikirim ulang.
```

---

## 26. Tugas Akhir Minggu 8

Buat project:

```txt
week8_final_wifi_http_offline_sync
```

Fitur wajib:

```txt
1. WiFi Station connect ke SSID dari NVS atau macro.
2. Event Group untuk WIFI_CONNECTED_BIT dan WIFI_FAIL_BIT.
3. HTTP POST JSON ke backend lokal.
4. Payload dibuat memakai cJSON.
5. Jika HTTP gagal, payload disimpan ke SPIFFS offline queue.
6. Offline queue memakai satu JSON per baris.
7. Sync task mengirim ulang offline queue saat WiFi connected.
8. Monitor task print:
   - WiFi connected atau offline
   - free heap
   - offline queue count
   - last HTTP status
9. HTTP server lokal menyediakan:
   - GET /
   - GET /api/status
10. Tidak ada HTTP request langsung dari WiFi event handler.
```

---

## 27. Checklist Lulus Minggu 8

Kamu dianggap lulus Minggu 8 kalau bisa:

```txt
[ ] Menjelaskan Station mode
[ ] Menjelaskan SoftAP mode
[ ] Menjelaskan WiFi event handler
[ ] Menjelaskan IP_EVENT_STA_GOT_IP
[ ] Menggunakan esp_netif_init()
[ ] Menggunakan esp_event_loop_create_default()
[ ] Menggunakan esp_wifi_init()
[ ] Menggunakan esp_wifi_set_mode()
[ ] Menggunakan esp_wifi_set_config()
[ ] Menggunakan esp_wifi_start()
[ ] Membuat reconnect logic
[ ] Menggunakan Event Group untuk WiFi status
[ ] Membuat HTTP GET
[ ] Membuat HTTP POST JSON
[ ] Membuat JSON dengan cJSON
[ ] Membuat HTTP server ESP32
[ ] Membuat endpoint /api/status
[ ] Membuat SoftAP dasar
[ ] Menggabungkan WiFi + HTTP + offline queue
```

---

## 28. Penerapan untuk Project Stunting IoT

Arsitektur cloud sync:

```txt
RFID Task
   ↓
Sensor Task
   ↓
Prediction Task
   ↓
Payload Builder
   ↓
Network Task
   ├── WiFi connected → HTTP POST ke backend
   └── WiFi offline   → simpan SPIFFS
```

Payload contoh:

```json
{
  "device_id": "STUNT-001",
  "uid": "A1B2C3D4",
  "age_month": 24,
  "height_cm": 82.5,
  "prediction": "risk",
  "firmware_version": "1.0.0"
}
```

Endpoint backend:

```txt
POST /api/diagnosis
GET  /api/device/:deviceId/status
```

Mode production:

```txt
1. Device boot.
2. Load WiFi dari NVS.
3. Kalau WiFi ada, connect.
4. Kalau WiFi tidak ada, buka SoftAP/BLE provisioning.
5. Hasil diagnosis dikirim HTTP POST.
6. Kalau gagal, simpan offline.
7. Sync ulang saat online.
```

---

## 29. Inti Pemahaman Minggu 8

Kalimat penting:

> WiFi event hanya memberi status. Network task yang melakukan kerja berat seperti HTTP, retry, dan sync.

Pola firmware yang benar:

```txt
WiFi Event Handler
   ↓
Set/Clear Event Group

Payload Task
   ↓
Queue

Network Task
   ↓
HTTP POST
   ├── success
   └── fail → offline queue

Sync Task
   ↓
Wait WIFI_CONNECTED_BIT
   ↓
Sync offline queue
```

Setelah Minggu 8 kuat, lanjut ke Minggu 9:

```txt
MQTT dan WebSocket untuk komunikasi IoT realtime
```
