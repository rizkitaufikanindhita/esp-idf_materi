# Minggu 11 — ESP-IDF Power Management, Light Sleep, Deep Sleep, dan Battery-Powered IoT

## Target Pembelajaran

Pada akhir Minggu 11, kamu mampu:

- Menjelaskan active mode, modem sleep, light sleep, dan deep sleep.
- Menggunakan timer wakeup untuk deep sleep.
- Menggunakan GPIO/tombol sebagai wakeup source.
- Memahami RTC memory.
- Membaca battery voltage dengan ADC dan voltage divider.
- Membuat pola firmware battery-powered: wake → work → sleep.
- Mendesain firmware yang tetap aman saat WiFi/server gagal.

---

## 1. Mode Daya ESP32

### Active Mode

ESP32 aktif penuh.

```text
CPU aktif
WiFi/BLE bisa aktif
Task FreeRTOS berjalan
Konsumsi daya paling besar
```

Dipakai saat membaca sensor, connect WiFi, HTTP/MQTT publish, BLE provisioning, OTA, dan processing data.

### Modem Sleep

CPU masih berjalan, tetapi radio WiFi/BLE bisa dihemat saat idle. Cocok untuk device yang tetap connected tetapi ingin mengurangi konsumsi radio.

### Light Sleep

```text
CPU berhenti sementara
RAM masih dipertahankan
Program lanjut setelah wakeup
Wakeup cepat
```

Setelah light sleep, kode lanjut dari baris setelah:

```c
esp_light_sleep_start();
```

### Deep Sleep

```text
CPU mati
Sebagian besar RAM hilang
Program boot ulang setelah wakeup
Konsumsi jauh lebih rendah
```

Setelah deep sleep, `app_main()` berjalan ulang.

Kalimat penting:

```text
Light sleep = pause lalu lanjut.
Deep sleep = tidur dalam lalu boot ulang.
```

---

## 2. Pola Low Power IoT

Pola umum:

```text
Boot / Wakeup
↓
Init minimal
↓
Baca wakeup cause
↓
Baca sensor
↓
Connect WiFi jika perlu
↓
Kirim data
↓
Jika gagal, simpan offline
↓
Matikan peripheral eksternal
↓
Deep sleep
```

Untuk device stunting portable:

```text
Deep sleep saat idle
Tombol ditekan -> wakeup
Baca RFID dan sensor tinggi
Prediksi
Kirim/simpan data
Sleep lagi
```

---

## 3. Deep Sleep Timer Wakeup

Contoh deep sleep 10 detik:

```c
#include <stdio.h>
#include <inttypes.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "esp_sleep.h"
#include "esp_timer.h"

#define SLEEP_TIME_SECONDS 10

static const char *TAG = "DEEP_SLEEP_TIMER";

void app_main(void)
{
    ESP_LOGI(TAG, "Device woke up");

    esp_sleep_wakeup_cause_t wakeup_reason = esp_sleep_get_wakeup_cause();

    switch (wakeup_reason) {
        case ESP_SLEEP_WAKEUP_TIMER:
            ESP_LOGI(TAG, "Wakeup caused by timer");
            break;

        case ESP_SLEEP_WAKEUP_UNDEFINED:
            ESP_LOGI(TAG, "First boot or reset");
            break;

        default:
            ESP_LOGI(TAG, "Wakeup cause: %d", wakeup_reason);
            break;
    }

    ESP_LOGI(TAG, "Working for 2 seconds...");
    vTaskDelay(pdMS_TO_TICKS(2000));

    ESP_LOGI(TAG, "Going to deep sleep for %d seconds", SLEEP_TIME_SECONDS);

    ESP_ERROR_CHECK(
        esp_sleep_enable_timer_wakeup(SLEEP_TIME_SECONDS * 1000000ULL)
    );

    esp_deep_sleep_start();
}
```

Parameter timer wakeup memakai mikrodetik.

```text
10 detik = 10 * 1.000.000 us
```

---

## 4. GPIO Wakeup

Untuk tombol active-low:

```text
Tidak ditekan = HIGH
Ditekan       = LOW
```

Konsep:

```text
ESP32 deep sleep
Tombol ditekan
GPIO menjadi LOW
ESP32 wakeup
```

Contoh untuk ESP32 klasik dengan EXT0:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_sleep.h"

#define WAKEUP_GPIO GPIO_NUM_0

static const char *TAG = "GPIO_WAKEUP";

void app_main(void)
{
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();

    if (cause == ESP_SLEEP_WAKEUP_EXT0) {
        ESP_LOGI(TAG, "Wakeup caused by external GPIO");
    } else {
        ESP_LOGI(TAG, "Wakeup cause: %d", cause);
    }

    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << WAKEUP_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE
    };

    gpio_config(&io_conf);

    ESP_LOGI(TAG, "Going to deep sleep. Press button to wake up.");
    vTaskDelay(pdMS_TO_TICKS(1000));

    ESP_ERROR_CHECK(esp_sleep_enable_ext0_wakeup(WAKEUP_GPIO, 0));

    esp_deep_sleep_start();
}
```

Catatan:

```text
Dukungan EXT0/EXT1/GPIO wakeup bergantung pada chip dan pin.
Selalu cek dokumentasi target ESP32, ESP32-C3, atau ESP32-S3.
```

---

## 5. Light Sleep

Contoh light sleep timer:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "esp_sleep.h"

#define LIGHT_SLEEP_TIME_SECONDS 5

static const char *TAG = "LIGHT_SLEEP";

void app_main(void)
{
    while (1) {
        ESP_LOGI(TAG, "Entering light sleep");

        ESP_ERROR_CHECK(
            esp_sleep_enable_timer_wakeup(LIGHT_SLEEP_TIME_SECONDS * 1000000ULL)
        );

        esp_err_t ret = esp_light_sleep_start();

        if (ret == ESP_OK) {
            ESP_LOGI(TAG, "Woke up from light sleep");
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

Perbedaan utama:

```text
esp_light_sleep_start() kembali ke baris berikutnya.
esp_deep_sleep_start() tidak kembali; device boot ulang.
```

---

## 6. RTC Memory

Variable biasa hilang setelah deep sleep. Agar data bertahan selama deep sleep, gunakan:

```c
RTC_DATA_ATTR
```

Contoh wake counter:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "esp_sleep.h"

#define SLEEP_TIME_SECONDS 5

static const char *TAG = "RTC_MEMORY";

RTC_DATA_ATTR static int boot_count = 0;

void app_main(void)
{
    boot_count++;

    ESP_LOGI(TAG, "Boot count in RTC memory: %d", boot_count);

    vTaskDelay(pdMS_TO_TICKS(1000));

    ESP_ERROR_CHECK(
        esp_sleep_enable_timer_wakeup(SLEEP_TIME_SECONDS * 1000000ULL)
    );

    esp_deep_sleep_start();
}
```

RTC memory vs NVS:

| Kebutuhan | Pilih |
|---|---|
| Bertahan saat deep sleep | RTC memory |
| Bertahan setelah power off | NVS |
| Counter wake sementara | RTC memory |
| Config device permanen | NVS |
| Data diagnosis/offline | SPIFFS/FATFS |

---

## 7. Battery Measurement dengan ADC

Baterai Li-ion 1 cell bisa mencapai 4.2V, sedangkan ADC ESP32 tidak boleh menerima tegangan lebih tinggi dari batas pin.

Gunakan voltage divider:

```text
Battery+ -- R1 --+-- ADC GPIO
                 |
                R2
                 |
                GND
```

Rumus:

```text
Vadc = Vbat * R2 / (R1 + R2)
Vbat = Vadc * (R1 + R2) / R2
```

Contoh R1 = 100k dan R2 = 100k:

```text
Vadc = Vbat * 0.5
Jika Vbat = 4.2V, Vadc = 2.1V
```

Arus divider:

```text
I = Vbat / (R1 + R2)
```

Dengan 100k + 100k:

```text
I = 4.2V / 200k = 21 uA
```

Untuk low-power serius, gunakan resistor lebih besar atau MOSFET agar divider hanya aktif saat pengukuran.

---

## 8. Battery Percent Sederhana

```c
static int battery_estimate_percent(float mv)
{
    if (mv >= 4200) {
        return 100;
    }

    if (mv <= 3300) {
        return 0;
    }

    return (int)((mv - 3300) * 100 / (4200 - 3300));
}
```

Catatan:

```text
Ini estimasi kasar. Kurva Li-ion tidak linear.
Untuk akurasi tinggi, gunakan fuel gauge IC.
```

---

## 9. WiFi dengan Timeout

Device battery-powered tidak boleh menunggu WiFi selamanya.

```c
EventBits_t bits = xEventGroupWaitBits(
    wifi_event_group,
    WIFI_CONNECTED_BIT,
    pdFALSE,
    pdTRUE,
    pdMS_TO_TICKS(10000)
);

if (bits & WIFI_CONNECTED_BIT) {
    // kirim data
} else {
    // simpan offline lalu sleep
}
```

Pola:

```text
Coba connect 10 detik
Jika berhasil -> kirim data
Jika gagal -> simpan offline
Deep sleep
```

---

## 10. Matikan Peripheral Eksternal

ESP32 bisa deep sleep hemat, tetapi sensor eksternal tetap boros kalau tidak dimatikan.

Gunakan MOSFET/load switch:

```text
ESP32 GPIO -> gate MOSFET/load switch
MOSFET/load switch -> power sensor
```

Firmware:

```c
gpio_set_level(SENSOR_POWER_GPIO, 1);
vTaskDelay(pdMS_TO_TICKS(100));
read_sensor();
gpio_set_level(SENSOR_POWER_GPIO, 0);
```

---

## 11. Mini Project Minggu 11

Project:

```text
week11_final_battery_powered_iot_node
```

Fitur wajib:

1. Deep sleep timer wakeup setiap 60 detik.
2. RTC wake counter.
3. Print wakeup cause.
4. Battery voltage reader dengan ADC + calibration.
5. Battery percentage estimation.
6. Payload JSON berisi:
   - `device_id`
   - `wake_count`
   - `battery_mv`
   - `battery_percent`
7. WiFi connect dengan timeout 10 detik.
8. Jika WiFi connected: HTTP POST payload.
9. Jika WiFi gagal atau HTTP gagal: simpan payload ke offline queue.
10. Setelah selesai:
    - stop WiFi.
    - matikan sensor power pin jika ada.
    - deep sleep lagi.

---

## 12. Error Umum

### Device Tidak Bangun

Cek:

- Wakeup source belum di-enable.
- Salah GPIO.
- Pin tidak mendukung wakeup.
- Pull-up/pull-down salah.
- Tombol wiring salah.

### Konsumsi Deep Sleep Tetap Besar

Cek:

- Power LED dev board.
- USB-UART chip.
- Regulator boros.
- Sensor eksternal tetap aktif.
- Voltage divider selalu membuang arus.
- Pull-up/pull-down terlalu kecil.

### Battery Reading Tidak Stabil

Solusi:

- Moving average 16–64 sampel.
- ADC calibration.
- Kapasitor kecil di node ADC.
- Divider dikontrol MOSFET.
- Sampling setelah supply stabil.

---

## 13. Latihan

1. Deep sleep 10 detik.
2. RTC wake counter.
3. Print wakeup cause.
4. GPIO wakeup dari tombol.
5. Light sleep 5 detik.
6. Battery ADC dengan voltage divider.
7. WiFi connect timeout.
8. Wake → read battery → try WiFi → send/save → sleep.

---

## 14. Checklist Lulus

- [ ] Menjelaskan active mode.
- [ ] Menjelaskan modem sleep.
- [ ] Menjelaskan light sleep.
- [ ] Menjelaskan deep sleep.
- [ ] Menggunakan `esp_sleep_enable_timer_wakeup()`.
- [ ] Menggunakan `esp_deep_sleep_start()`.
- [ ] Menggunakan `esp_light_sleep_start()`.
- [ ] Membaca wakeup cause.
- [ ] Menggunakan `RTC_DATA_ATTR`.
- [ ] Membaca baterai dengan ADC.
- [ ] Menghitung voltage divider.
- [ ] Membuat WiFi timeout.
- [ ] Membuat pola wake → work → sleep.
- [ ] Menjelaskan kenapa dev board sering tidak hemat daya.
