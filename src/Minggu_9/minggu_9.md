# Minggu 9 — ESP-IDF MQTT dan WebSocket untuk IoT Realtime

## Target Pembelajaran

Pada akhir Minggu 9, kamu mampu:

- Menjelaskan perbedaan HTTP, MQTT, dan WebSocket.
- Membuat ESP32 connect ke MQTT broker.
- Publish telemetry dari ESP32 ke broker.
- Subscribe command dari server/backend.
- Mendesain topic MQTT yang rapi.
- Memahami QoS, retained message, Last Will and Testament, dan reconnect.
- Membuat dasar WebSocket client/server untuk komunikasi realtime.
- Menggabungkan MQTT dengan offline queue dari Minggu 7–8.

---

## 1. HTTP vs MQTT vs WebSocket

### HTTP

HTTP cocok untuk pola request-response.

```text
ESP32 -> HTTP POST -> Server
ESP32 <- Response <- Server
```

Cocok untuk:

- REST API.
- Upload data periodik.
- Cek update OTA.
- Provisioning sederhana.

Kelemahan HTTP untuk IoT realtime:

- Server tidak mudah mengirim command langsung ke ESP32.
- Overhead cukup besar jika data sering.
- Kurang ideal untuk telemetry realtime banyak device.

### MQTT

MQTT cocok untuk IoT publish/subscribe.

```text
ESP32 publish telemetry -> MQTT Broker -> Backend subscribe
Backend publish command -> MQTT Broker -> ESP32 subscribe
```

Cocok untuk:

- Telemetry sensor.
- Command device.
- Heartbeat.
- Status online/offline.
- Banyak device.

### WebSocket

WebSocket cocok untuk komunikasi dua arah realtime berbasis koneksi yang tetap terbuka.

```text
Client connect sekali
Lalu client dan server bisa saling kirim data realtime
```

Cocok untuk:

- Dashboard realtime.
- Local control dari browser.
- Streaming log/status.
- Debug UI lokal.

---

## 2. Konsep MQTT

Komponen utama MQTT:

1. **Broker**: pusat pertukaran pesan.
2. **Publisher**: pengirim pesan.
3. **Subscriber**: penerima pesan.
4. **Topic**: alamat pesan.

Contoh topic:

```text
devices/STUNT-001/telemetry
devices/STUNT-001/status
devices/STUNT-001/command
devices/STUNT-001/heartbeat
devices/STUNT-001/config
devices/STUNT-001/ota
```

Desain topic production:

```text
devices/{device_id}/telemetry
devices/{device_id}/diagnosis
devices/{device_id}/status
devices/{device_id}/heartbeat
devices/{device_id}/command
devices/{device_id}/config
devices/{device_id}/ota
devices/{device_id}/ack
```

Wildcard:

```text
+  = satu level
#  = banyak level
```

Contoh backend subscribe semua telemetry:

```text
devices/+/telemetry
```

---

## 3. MQTT Connect ESP-IDF

Untuk ESP-IDF v6.x, MQTT menjadi dependency terpisah:

```bash
idf.py add-dependency espressif/mqtt
```

Untuk ESP-IDF v5.x, biasanya MQTT masih tersedia sebagai komponen bawaan.

Contoh kode MQTT basic:

```c
#include <stdio.h>
#include <string.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "mqtt_client.h"

static const char *TAG = "MQTT_BASIC";

#define MQTT_BROKER_URI "mqtt://192.168.1.10:1883"

static esp_mqtt_client_handle_t mqtt_client = NULL;
static bool mqtt_connected = false;

static void mqtt_event_handler(void *handler_args,
                               esp_event_base_t base,
                               int32_t event_id,
                               void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;

    switch ((esp_mqtt_event_id_t)event_id) {
        case MQTT_EVENT_CONNECTED:
            ESP_LOGI(TAG, "MQTT connected");
            mqtt_connected = true;
            break;

        case MQTT_EVENT_DISCONNECTED:
            ESP_LOGW(TAG, "MQTT disconnected");
            mqtt_connected = false;
            break;

        case MQTT_EVENT_SUBSCRIBED:
            ESP_LOGI(TAG, "MQTT subscribed, msg_id=%d", event->msg_id);
            break;

        case MQTT_EVENT_PUBLISHED:
            ESP_LOGI(TAG, "MQTT published, msg_id=%d", event->msg_id);
            break;

        case MQTT_EVENT_DATA:
            ESP_LOGI(TAG, "TOPIC=%.*s", event->topic_len, event->topic);
            ESP_LOGI(TAG, "DATA=%.*s", event->data_len, event->data);
            break;

        case MQTT_EVENT_ERROR:
            ESP_LOGE(TAG, "MQTT error");
            break;

        default:
            ESP_LOGI(TAG, "MQTT event id=%ld", event_id);
            break;
    }
}

static void mqtt_app_start(void)
{
    esp_mqtt_client_config_t mqtt_config = {
        .broker.address.uri = MQTT_BROKER_URI,
    };

    mqtt_client = esp_mqtt_client_init(&mqtt_config);

    if (mqtt_client == NULL) {
        ESP_LOGE(TAG, "Failed to init MQTT client");
        return;
    }

    esp_mqtt_client_register_event(
        mqtt_client,
        ESP_EVENT_ANY_ID,
        mqtt_event_handler,
        NULL
    );

    esp_mqtt_client_start(mqtt_client);
}
```

Panggil `mqtt_app_start()` hanya setelah WiFi sudah mendapat IP.

---

## 4. MQTT Publish Telemetry

Contoh publish telemetry:

```c
static void mqtt_publish_telemetry(const char *device_id,
                                   const char *json_payload)
{
    if (!mqtt_connected || mqtt_client == NULL) {
        ESP_LOGW("MQTT", "MQTT not connected, skip publish");
        return;
    }

    char topic[128];

    snprintf(topic, sizeof(topic),
             "devices/%s/telemetry",
             device_id);

    int msg_id = esp_mqtt_client_publish(
        mqtt_client,
        topic,
        json_payload,
        0,
        1,
        0
    );

    ESP_LOGI("MQTT", "Published telemetry, msg_id=%d", msg_id);
}
```

Parameter penting:

```c
esp_mqtt_client_publish(client, topic, data, len, qos, retain);
```

- `qos = 0`: cepat, tidak dijamin sampai.
- `qos = 1`: minimal sampai sekali, bisa duplicate.
- `retain = 1`: broker menyimpan pesan terakhir pada topic.

---

## 5. MQTT Subscribe Command

ESP32 subscribe:

```text
devices/ESP32-001/command
```

Backend publish:

```json
{"cmd":"LED_ON"}
```

Subscribe saat connected:

```c
case MQTT_EVENT_CONNECTED:
    mqtt_connected = true;
    esp_mqtt_client_subscribe(
        mqtt_client,
        "devices/ESP32-001/command",
        1
    );
    break;
```

Command jangan diproses berat di event handler. Gunakan queue.

```c
typedef struct {
    char topic[128];
    char payload[256];
} mqtt_command_msg_t;

static QueueHandle_t mqtt_command_queue = NULL;
```

Di event handler:

```c
case MQTT_EVENT_DATA: {
    mqtt_command_msg_t msg = {0};

    int topic_len = event->topic_len;
    int data_len = event->data_len;

    if (topic_len >= sizeof(msg.topic)) {
        topic_len = sizeof(msg.topic) - 1;
    }

    if (data_len >= sizeof(msg.payload)) {
        data_len = sizeof(msg.payload) - 1;
    }

    memcpy(msg.topic, event->topic, topic_len);
    msg.topic[topic_len] = '\0';

    memcpy(msg.payload, event->data, data_len);
    msg.payload[data_len] = '\0';

    xQueueSend(mqtt_command_queue, &msg, 0);
    break;
}
```

Command task:

```c
static void mqtt_command_task(void *pvParameters)
{
    mqtt_command_msg_t msg;

    while (1) {
        if (xQueueReceive(mqtt_command_queue, &msg, portMAX_DELAY) == pdTRUE) {
            ESP_LOGI("MQTT_CMD", "Topic: %s", msg.topic);
            ESP_LOGI("MQTT_CMD", "Payload: %s", msg.payload);

            /* parse JSON command di sini */
        }
    }
}
```

---

## 6. QoS, Retained Message, dan LWT

### QoS

| QoS | Arti | Cocok untuk |
|---|---|---|
| 0 | At most once | telemetry sering |
| 1 | At least once | command/data penting |
| 2 | Exactly once | jarang dipakai di ESP32 karena lebih berat |

Rekomendasi:

```text
Telemetry biasa        -> QoS 0 atau QoS 1
Command penting        -> QoS 1
Heartbeat              -> QoS 0
Diagnosis penting      -> QoS 1 + offline queue
OTA command            -> QoS 1
```

### Retained Message

Retained message cocok untuk:

- Status terakhir.
- Desired state.
- Config terakhir.

Tidak cocok untuk command sekali jalan seperti `OTA_CHECK` atau `RESET_WIFI`, karena bisa dieksekusi ulang saat reconnect.

### Last Will and Testament

LWT membuat broker mengirim pesan jika device disconnect tidak normal.

Contoh:

```text
Topic: devices/STUNT-001/status
Payload: {"online":false}
```

Saat connect normal, ESP32 publish:

```json
{"online":true}
```

---

## 7. WebSocket Dasar

WebSocket cocok untuk realtime dashboard atau local control.

Dependency:

```bash
idf.py add-dependency espressif/esp_websocket_client
```

Contoh WebSocket client:

```c
#include "esp_log.h"
#include "esp_websocket_client.h"

static const char *TAG = "WS_CLIENT";
#define WEBSOCKET_URI "ws://192.168.1.10:8080/ws"

static esp_websocket_client_handle_t ws_client = NULL;
static bool ws_connected = false;

static void websocket_event_handler(void *handler_args,
                                    esp_event_base_t base,
                                    int32_t event_id,
                                    void *event_data)
{
    esp_websocket_event_data_t *data =
        (esp_websocket_event_data_t *)event_data;

    switch (event_id) {
        case WEBSOCKET_EVENT_CONNECTED:
            ESP_LOGI(TAG, "WebSocket connected");
            ws_connected = true;
            break;

        case WEBSOCKET_EVENT_DISCONNECTED:
            ESP_LOGW(TAG, "WebSocket disconnected");
            ws_connected = false;
            break;

        case WEBSOCKET_EVENT_DATA:
            ESP_LOGI(TAG, "DATA=%.*s", data->data_len, (char *)data->data_ptr);
            break;

        case WEBSOCKET_EVENT_ERROR:
            ESP_LOGE(TAG, "WebSocket error");
            break;
    }
}
```

---

## 8. Mini Project Minggu 9

Project:

```text
week9_final_mqtt_realtime_device
```

Fitur wajib:

1. WiFi Station connect.
2. MQTT connect ke broker.
3. Publish telemetry setiap 5 detik.
4. Subscribe command topic.
5. Command `LED_ON` dan `LED_OFF`.
6. Publish status setelah command.
7. Status topic memakai retained message.
8. Event Group untuk WiFi dan MQTT connected.
9. MQTT event handler hanya mengirim command ke queue.
10. Jika MQTT offline, payload masuk offline queue.
11. Saat MQTT reconnect, sync offline queue.
12. Monitor task print WiFi, MQTT, heap, offline queue count.

---

## 9. Latihan

1. Connect MQTT broker lokal.
2. Publish telemetry sederhana.
3. Subscribe command.
4. Parse command JSON.
5. Buat command queue.
6. Publish status retained.
7. Simulasikan broker mati dan gunakan offline queue.
8. Buat WebSocket client dasar.

---

## 10. Checklist Lulus

- [ ] Menjelaskan MQTT broker, publisher, subscriber.
- [ ] Menjelaskan topic dan wildcard.
- [ ] Connect MQTT dari ESP-IDF.
- [ ] Publish telemetry.
- [ ] Subscribe command.
- [ ] Menggunakan command queue.
- [ ] Menjelaskan QoS, retained, dan LWT.
- [ ] Membuat offline fallback saat MQTT disconnect.
- [ ] Menjelaskan kapan pakai MQTT vs HTTP vs WebSocket.
- [ ] Membuat WebSocket client dasar.
