# Minggu 1 — Dasar ESP-IDF: Project Structure, Build System, GPIO, Delay, dan Logging

## Target Minggu 1

Setelah menyelesaikan Minggu 1, kamu diharapkan mampu:

- Memahami struktur dasar project ESP-IDF.
- Membuat, build, flash, dan monitor project ESP-IDF.
- Memahami fungsi `app_main()`.
- Menggunakan GPIO sebagai output untuk LED.
- Menggunakan GPIO sebagai input untuk tombol.
- Menggunakan `vTaskDelay()` dengan benar.
- Menggunakan `ESP_LOGI`, `ESP_LOGW`, dan `ESP_LOGE`.
- Mulai memahami gaya berpikir firmware berbasis ESP-IDF.

Minggu 1 adalah fondasi. Jangan buru-buru masuk WiFi, MQTT, BLE, OTA, atau sensor kompleks sebelum dasar ini benar-benar kuat.

---

## 1. Gambaran Besar ESP-IDF

ESP-IDF adalah framework resmi dari Espressif untuk membuat firmware ESP32, ESP32-C3, ESP32-S3, dan keluarga ESP lain.

Berbeda dengan Arduino, ESP-IDF memberi akses lebih dekat ke sistem asli ESP32:

- FreeRTOS task
- Driver GPIO, UART, I2C, SPI
- WiFi dan BLE native
- NVS storage
- OTA
- Security
- Logging
- Partition table
- Power management

Kalau Arduino cocok untuk prototyping cepat, ESP-IDF lebih cocok untuk firmware yang lebih serius, modular, dan production-grade.

---

## 2. Perbedaan Arduino dan ESP-IDF

| Arduino | ESP-IDF |
|---|---|
| `setup()` dan `loop()` | `app_main()` |
| Abstraksi sederhana | Lebih dekat ke sistem asli |
| Cocok untuk pemula/prototipe | Cocok untuk production firmware |
| Banyak library siap pakai | Driver resmi dan component system |
| Struktur sederhana | Struktur project lebih rapi dan scalable |
| Kontrol terbatas | Kontrol lebih penuh terhadap RTOS, memory, dan peripheral |

Di Arduino:

```cpp
void setup() {
    pinMode(2, OUTPUT);
}

void loop() {
    digitalWrite(2, HIGH);
    delay(1000);
    digitalWrite(2, LOW);
    delay(1000);
}
```

Di ESP-IDF:

```c
void app_main(void)
{
    // program dimulai dari sini
}
```

---

## 3. Struktur Project ESP-IDF

Struktur project minimal:

```txt
my_project/
├── CMakeLists.txt
└── main/
    ├── CMakeLists.txt
    └── main.c
```

Penjelasan:

```txt
CMakeLists.txt          → konfigurasi project utama
main/CMakeLists.txt     → konfigurasi source file di folder main
main/main.c             → kode utama aplikasi
```

Contoh `CMakeLists.txt` root:

```cmake
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(week1_basic)
```

Contoh `main/CMakeLists.txt`:

```cmake
idf_component_register(SRCS "main.c"
                    INCLUDE_DIRS ".")
```

---

## 4. Perintah Dasar ESP-IDF

### 4.1 Membuat Project Baru

```bash
idf.py create-project week1_basic
cd week1_basic
```

Atau bisa copy dari example:

```bash
idf.py create-project-from-example get-started/hello_world
```

### 4.2 Set Target Chip

Untuk ESP32:

```bash
idf.py set-target esp32
```

Untuk ESP32-C3:

```bash
idf.py set-target esp32c3
```

Untuk ESP32-S3:

```bash
idf.py set-target esp32s3
```

### 4.3 Build Project

```bash
idf.py build
```

### 4.4 Flash ke Board

```bash
idf.py flash
```

Kalau perlu port spesifik:

```bash
idf.py -p COM5 flash
```

Linux/macOS:

```bash
idf.py -p /dev/ttyUSB0 flash
```

### 4.5 Monitor Serial

```bash
idf.py monitor
```

Build, flash, dan monitor sekaligus:

```bash
idf.py build flash monitor
```

Keluar dari monitor:

```txt
Ctrl + ]
```

---

## 5. `app_main()`

Di ESP-IDF, entry point utama adalah:

```c
void app_main(void)
{
}
```

`app_main()` mirip seperti `main()` dalam program C biasa, tetapi berjalan di atas sistem ESP-IDF dan FreeRTOS.

Contoh paling sederhana:

```c
#include <stdio.h>

void app_main(void)
{
    printf("Hello ESP-IDF!\n");
}
```

Namun dalam ESP-IDF, untuk logging lebih disarankan memakai `ESP_LOGI()` daripada `printf()`.

---

## 6. Logging di ESP-IDF

Header:

```c
#include "esp_log.h"
```

Contoh:

```c
#include "esp_log.h"

static const char *TAG = "MAIN";

void app_main(void)
{
    ESP_LOGI(TAG, "Hello from ESP-IDF");
}
```

Level logging:

| Macro | Level | Fungsi |
|---|---|---|
| `ESP_LOGE` | Error | Error serius |
| `ESP_LOGW` | Warning | Peringatan |
| `ESP_LOGI` | Info | Informasi umum |
| `ESP_LOGD` | Debug | Detail debugging |
| `ESP_LOGV` | Verbose | Sangat detail |

Contoh:

```c
ESP_LOGE(TAG, "Sensor failed");
ESP_LOGW(TAG, "WiFi disconnected");
ESP_LOGI(TAG, "System started");
ESP_LOGD(TAG, "Raw value: %d", value);
```

### Best Practice Logging

Gunakan TAG berbeda untuk setiap module:

```c
static const char *TAG = "APP_WIFI";
static const char *TAG = "APP_SENSOR";
static const char *TAG = "APP_STORAGE";
```

Jangan semua log memakai TAG `MAIN`, karena nanti sulit mengetahui sumber masalah.

---

## 7. Delay di ESP-IDF

Di Arduino kamu biasa memakai:

```cpp
delay(1000);
```

Di ESP-IDF, gunakan:

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

Header:

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
```

Contoh:

```c
while (1) {
    ESP_LOGI(TAG, "Loop running");
    vTaskDelay(pdMS_TO_TICKS(1000));
}
```

### Kenapa Tidak Langsung `vTaskDelay(1000)`?

Karena `vTaskDelay()` menerima satuan tick, bukan milidetik.

Gunakan:

```c
pdMS_TO_TICKS(1000)
```

agar 1000 ms dikonversi ke tick FreeRTOS.

---

## 8. GPIO Output: Blink LED

### 8.1 Konsep GPIO Output

GPIO output berarti pin ESP32 mengeluarkan level digital:

```txt
LOW  = 0V
HIGH = 3.3V
```

LED biasanya dikontrol dengan:

```c
gpio_set_level(LED_GPIO, 1); // ON
gpio_set_level(LED_GPIO, 0); // OFF
```

Namun pada beberapa board, LED bisa active-low. Artinya:

```txt
0 = ON
1 = OFF
```

Untuk latihan awal, sesuaikan dengan board kamu.

---

### 8.2 Full Code Blink LED

File: `main/main.c`

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2

static const char *TAG = "BLINK";

void app_main(void)
{
    ESP_LOGI(TAG, "Blink LED example started");

    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        ESP_LOGI(TAG, "LED ON");
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(1000));

        ESP_LOGI(TAG, "LED OFF");
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

### 8.3 Penjelasan Kode

```c
#define LED_GPIO GPIO_NUM_2
```

Menentukan pin LED.

```c
gpio_reset_pin(LED_GPIO);
```

Mengembalikan konfigurasi GPIO ke default.

```c
gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
```

Mengatur pin sebagai output.

```c
gpio_set_level(LED_GPIO, 1);
```

Mengeluarkan level HIGH.

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

Delay 1000 ms tanpa memblokir scheduler secara buruk.

---

## 9. GPIO Input: Tombol

### 9.1 Konsep Tombol Active-Low

Umumnya tombol disusun active-low:

```txt
Tidak ditekan = HIGH
Ditekan       = LOW
```

Dengan pull-up internal:

```txt
GPIO ---- tombol ---- GND
 |
 pull-up internal
 |
 3.3V
```

Saat tombol tidak ditekan, pin ditarik HIGH oleh pull-up.

Saat tombol ditekan, pin tersambung ke GND, sehingga LOW.

---

### 9.2 Full Code Button Polling

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define BUTTON_GPIO GPIO_NUM_0

static const char *TAG = "BUTTON";

void app_main(void)
{
    ESP_LOGI(TAG, "Button input example started");

    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);

    while (1) {
        int level = gpio_get_level(BUTTON_GPIO);

        if (level == 0) {
            ESP_LOGI(TAG, "Button pressed");
        } else {
            ESP_LOGI(TAG, "Button released");
        }

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

---

### 9.3 Penjelasan

```c
gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
```

Pin dijadikan input.

```c
gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);
```

Mengaktifkan pull-up internal.

```c
int level = gpio_get_level(BUTTON_GPIO);
```

Membaca level pin.

```c
if (level == 0)
```

Karena tombol active-low, nilai 0 berarti tombol ditekan.

---

## 10. Button Mengontrol LED

Project ini menggabungkan input dan output.

Target:

```txt
Tombol ditekan   → LED ON
Tombol dilepas   → LED OFF
```

Kode:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO    GPIO_NUM_2
#define BUTTON_GPIO GPIO_NUM_0

static const char *TAG = "BUTTON_LED";

static void gpio_init_all(void)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);
}

void app_main(void)
{
    ESP_LOGI(TAG, "Button controls LED example started");

    gpio_init_all();

    while (1) {
        int button_level = gpio_get_level(BUTTON_GPIO);

        if (button_level == 0) {
            gpio_set_level(LED_GPIO, 1);
            ESP_LOGI(TAG, "Button pressed, LED ON");
        } else {
            gpio_set_level(LED_GPIO, 0);
            ESP_LOGI(TAG, "Button released, LED OFF");
        }

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

---

## 11. Toggle LED dengan Tombol

Target:

```txt
Tekan tombol sekali → LED berubah ON
Tekan lagi         → LED berubah OFF
```

Masalah utama: mechanical button memiliki bouncing. Untuk Minggu 1, kita pakai debounce sederhana dengan delay.

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO    GPIO_NUM_2
#define BUTTON_GPIO GPIO_NUM_0

static const char *TAG = "TOGGLE";

static void gpio_init_all(void)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);
}

void app_main(void)
{
    ESP_LOGI(TAG, "Button toggle LED example started");

    gpio_init_all();

    bool led_state = false;
    int last_button_level = 1;

    while (1) {
        int current_button_level = gpio_get_level(BUTTON_GPIO);

        if (last_button_level == 1 && current_button_level == 0) {
            vTaskDelay(pdMS_TO_TICKS(50));

            if (gpio_get_level(BUTTON_GPIO) == 0) {
                led_state = !led_state;
                gpio_set_level(LED_GPIO, led_state ? 1 : 0);

                ESP_LOGI(TAG, "Button pressed, LED is now %s",
                         led_state ? "ON" : "OFF");
            }
        }

        last_button_level = current_button_level;

        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

---

## 12. Debounce Dasar

Tombol mekanik tidak langsung stabil saat ditekan.

Saat tombol ditekan, sinyal bisa seperti ini:

```txt
HIGH LOW HIGH LOW LOW LOW
```

Padahal secara manusia hanya satu kali tekan.

Debounce sederhana:

```txt
1. Deteksi perubahan HIGH → LOW
2. Tunggu 50 ms
3. Baca lagi
4. Jika masih LOW, anggap valid
```

Kode inti:

```c
if (last_button_level == 1 && current_button_level == 0) {
    vTaskDelay(pdMS_TO_TICKS(50));

    if (gpio_get_level(BUTTON_GPIO) == 0) {
        // valid press
    }
}
```

Nanti di Minggu 4, debounce akan dibuat lebih baik menggunakan interrupt dan task.

---

## 13. Kesalahan Umum Minggu 1

### 13.1 Salah Pilih GPIO

Tidak semua GPIO aman dipakai.

Hati-hati dengan:

- pin strapping
- pin flash/PSRAM
- pin USB native
- pin input-only pada chip tertentu
- pin yang sudah dipakai board

Untuk latihan, gunakan pin yang aman dari board kamu.

---

### 13.2 Lupa Pull-up Tombol

Gejala:

```txt
Tombol terbaca random
Kadang pressed sendiri
Nilai GPIO loncat-loncat
```

Solusi:

```c
gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);
```

---

### 13.3 Salah Memahami Active-Low

Tombol active-low:

```txt
0 = ditekan
1 = tidak ditekan
```

Jangan terbalik.

---

### 13.4 Salah Satuan Delay

Buruk:

```c
vTaskDelay(1000);
```

Lebih benar:

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

---

### 13.5 LED Tidak Menyala

Cek:

```txt
1. GPIO salah
2. LED active-low
3. Board tidak punya LED di GPIO itu
4. LED eksternal belum pakai resistor
5. GND belum tersambung
6. Program belum ter-flash
```

---

## 14. Best Practice Minggu 1

1. Selalu pakai `ESP_LOGI()` untuk status penting.
2. Selalu cek GPIO sesuai board.
3. Gunakan `pdMS_TO_TICKS()` untuk delay.
4. Pisahkan init GPIO ke fungsi sendiri.
5. Jangan menulis semua logic langsung tanpa struktur.
6. Mulai biasakan nama TAG yang jelas.
7. Jangan abaikan warning saat build.
8. Pahami active-high dan active-low.

---

## 15. Latihan Minggu 1

### Latihan 1 — Hello ESP-IDF

Buat project yang print:

```txt
Hello ESP-IDF
```

menggunakan:

```c
ESP_LOGI()
```

---

### Latihan 2 — Blink LED

Buat LED berkedip:

```txt
ON  500 ms
OFF 500 ms
```

---

### Latihan 3 — Blink dengan 3 Pola

Buat pola:

```txt
1. Blink lambat 1000 ms
2. Blink sedang 500 ms
3. Blink cepat 100 ms
```

---

### Latihan 4 — Baca Tombol

Print status tombol:

```txt
Button pressed
Button released
```

---

### Latihan 5 — Tombol Mengontrol LED

Tombol ditekan LED ON, dilepas LED OFF.

---

### Latihan 6 — Toggle LED

Tekan tombol sekali, LED toggle.

---

### Latihan 7 — Debounce

Tambahkan debounce 50 ms agar satu tekan tidak terbaca berkali-kali.

---

## 16. Tugas Akhir Minggu 1

Buat project:

```txt
week1_final_gpio_logging
```

Fitur wajib:

```txt
1. LED output.
2. Button input active-low.
3. Pull-up internal aktif.
4. Tombol toggle LED.
5. Debounce sederhana 50 ms.
6. Log saat:
   - system start
   - button valid pressed
   - LED ON
   - LED OFF
7. Semua GPIO init di fungsi gpio_init_all().
8. Delay memakai pdMS_TO_TICKS().
```

Contoh output log:

```txt
I (300) MAIN: Week 1 final project started
I (1500) MAIN: Button valid pressed
I (1501) MAIN: LED ON
I (3500) MAIN: Button valid pressed
I (3501) MAIN: LED OFF
```

---

## 17. Checklist Lulus Minggu 1

Kamu dianggap lulus Minggu 1 kalau bisa:

```txt
[ ] Menjelaskan apa itu ESP-IDF
[ ] Menjelaskan perbedaan Arduino dan ESP-IDF
[ ] Menjelaskan fungsi app_main()
[ ] Membuat project ESP-IDF baru
[ ] Menjalankan idf.py build
[ ] Menjalankan idf.py flash
[ ] Menjalankan idf.py monitor
[ ] Menggunakan ESP_LOGI()
[ ] Menggunakan GPIO output
[ ] Menggunakan GPIO input
[ ] Mengaktifkan pull-up internal
[ ] Membaca tombol active-low
[ ] Mengontrol LED
[ ] Menggunakan vTaskDelay(pdMS_TO_TICKS())
[ ] Membuat toggle LED dengan debounce sederhana
```

---

## 18. Inti Pemahaman Minggu 1

Kalimat penting:

> ESP-IDF dimulai dari pemahaman struktur project, `app_main()`, GPIO, delay FreeRTOS, dan logging.

Pola dasar firmware:

```txt
Init hardware
↓
Masuk loop/task
↓
Baca input
↓
Proses logic
↓
Update output
↓
Delay/yield
```

Contoh paling dasar:

```txt
Init LED + tombol
↓
Baca tombol
↓
Toggle LED
↓
Log status
↓
Delay kecil
```

Setelah Minggu 1 kuat, kamu siap masuk Minggu 2:

```txt
FreeRTOS Task, multitasking, task priority, dan task management
```
