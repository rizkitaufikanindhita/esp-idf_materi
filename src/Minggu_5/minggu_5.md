# Minggu 5 — ESP-IDF Peripheral Communication: UART, I2C, SPI

## Target Minggu 5

Setelah menyelesaikan Minggu 5, kamu diharapkan mampu:

- Menjelaskan perbedaan UART, I2C, dan SPI.
- Menghubungkan ESP32 dengan modul eksternal menggunakan wiring yang benar.
- Menggunakan driver UART native ESP-IDF.
- Membuat UART echo dan command parser sederhana.
- Menggunakan I2C master driver untuk scan device.
- Membaca dan menulis register sensor I2C.
- Menggunakan SPI master driver untuk transaksi data.
- Memahami penggunaan mutex untuk shared bus.
- Membuat mini project peripheral gateway berbasis UART command.

Minggu ini adalah fondasi penting sebelum masuk sensor, RFID, LCD, GPS, dan modul komunikasi eksternal.

---

## 1. Gambaran Besar Minggu 5

| Hari | Fokus | Output |
|---|---|---|
| Hari 1 | Konsep UART, I2C, SPI | Paham kapan memakai masing-masing protokol |
| Hari 2 | UART TX/RX | ESP32 bisa kirim dan terima serial |
| Hari 3 | UART event queue | ESP32 bisa handle event UART |
| Hari 4 | I2C scanner | ESP32 bisa mendeteksi alamat sensor I2C |
| Hari 5 | I2C register read/write | ESP32 bisa baca/tulis register sensor |
| Hari 6 | SPI master | ESP32 bisa transaksi SPI |
| Hari 7 | Mini project | UART command + I2C scanner + SPI test |

---

## 2. Peta Besar UART, I2C, dan SPI

### 2.1 UART

UART cocok untuk:

- GPS module.
- GSM/LTE module.
- RS485/Modbus.
- Debug serial.
- Komunikasi ESP32 dengan mikrokontroler lain.
- Fingerprint sensor.
- Barcode scanner.

Karakteristik UART:

```txt
Jumlah kabel utama : TX, RX, GND
Clock              : Tidak memakai clock eksternal
Topologi           : Point-to-point
Kecepatan umum     : 9600, 115200, 921600 baud
Format umum        : 8N1
```

Format `8N1` artinya:

```txt
8 data bits
No parity
1 stop bit
```

---

### 2.2 I2C

I2C cocok untuk:

- Sensor suhu dan kelembapan.
- Sensor jarak.
- RTC.
- EEPROM.
- LCD I2C.
- OLED.
- IMU.
- ADC eksternal.

Karakteristik I2C:

```txt
Jumlah kabel utama : SDA, SCL, GND
Butuh pull-up      : Ya
Topologi           : Banyak device dalam satu bus
Address device     : Umumnya 7-bit address
Kecepatan umum     : 100 kHz, 400 kHz
```

---

### 2.3 SPI

SPI cocok untuk:

- RFID RC522.
- TFT display.
- SD card.
- External flash.
- High-speed ADC.
- LoRa module.

Karakteristik SPI:

```txt
Jumlah kabel utama : MOSI, MISO, SCLK, CS, GND
Kecepatan          : Lebih tinggi dari I2C
Topologi           : Satu bus, banyak device dengan CS berbeda
Address device     : Tidak pakai address, pakai CS
```

---

## 3. Kapan Pakai UART, I2C, atau SPI?

| Kebutuhan | Pilihan |
|---|---|
| Komunikasi serial dengan GPS/GSM | UART |
| Banyak sensor lambat dalam satu bus | I2C |
| RFID, display cepat, SD card | SPI |
| Hanya ingin dua kabel untuk banyak sensor | I2C |
| Kecepatan tinggi | SPI |
| Modul memakai AT command | UART |
| Sensor register kecil | I2C |
| Transfer blok data besar | SPI |

Sederhananya:

```txt
UART → ngobrol serial dua arah
I2C  → banyak sensor satu bus
SPI  → cepat, cocok display/RFID/storage
```

---

## 4. Konsep Wiring dan Level Tegangan

### 4.1 Common Ground Wajib

Apa pun protokolnya, ground harus sama.

```txt
ESP32 GND ───── Module GND
```

Tanpa common ground, sinyal TX/RX/SDA/SCL/MOSI/MISO bisa tidak terbaca benar.

---

### 4.2 ESP32 Menggunakan Logic 3.3V

ESP32 GPIO umumnya memakai level logika 3.3V.

```txt
ESP32 TX = 3.3V HIGH
ESP32 RX tidak aman menerima 5V langsung
```

Jika modul memakai logic 5V, gunakan:

- Logic level shifter.
- Voltage divider untuk jalur masuk ke ESP32.
- Modul yang sudah 3.3V-compatible.

Jangan langsung hubungkan TX 5V module ke RX ESP32.

---

### 4.3 Pin Bisa Dipilih, tapi Jangan Sembarangan

ESP32 memiliki GPIO matrix, sehingga banyak peripheral bisa dipetakan ke berbagai pin.

Tetap hati-hati dengan:

- Strapping pins.
- Pin USB native.
- Pin flash/PSRAM internal.
- Input-only pins pada chip tertentu.
- Pin yang sudah dipakai board.

Untuk ESP32-S3, ESP32-C3, dan ESP32 klasik, mapping pin bisa berbeda. Selalu cek pinout board.

---

# Hari 2 — UART Dasar

## 5. Konsep UART

UART minimal membutuhkan:

```txt
ESP32 TX → Module RX
ESP32 RX ← Module TX
ESP32 GND ↔ Module GND
```

TX dan RX harus silang:

```txt
TX device A → RX device B
RX device A ← TX device B
```

Parameter UART:

```txt
Baudrate  : 9600 / 115200 / 921600 / dll
Data bit  : biasanya 8
Parity    : biasanya none
Stop bit  : biasanya 1
Flow ctrl : biasanya disabled
```

---

## 6. Alur UART Driver ESP-IDF

Alur umum:

```txt
1. Tentukan UART port
2. Konfigurasi parameter UART
3. Set pin TX/RX
4. Install driver
5. Read/write data
```

API utama:

```c
uart_param_config()
uart_set_pin()
uart_driver_install()
uart_read_bytes()
uart_write_bytes()
```

---

## 7. Contoh UART Echo

Project:

```txt
week5_01_uart_echo
```

Pin contoh:

```txt
UART_NUM_1
TX = GPIO17
RX = GPIO18
Baudrate = 115200
```

Kode:

```c
#include <stdio.h>
#include <string.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/uart.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define UART_PORT       UART_NUM_1
#define UART_TX_PIN     GPIO_NUM_17
#define UART_RX_PIN     GPIO_NUM_18
#define UART_BAUD_RATE  115200

#define UART_RX_BUF_SIZE 1024
#define UART_TX_BUF_SIZE 1024

static const char *TAG = "UART_ECHO";

static void uart_init(void)
{
    uart_config_t uart_config = {
        .baud_rate = UART_BAUD_RATE,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_DEFAULT,
    };

    ESP_ERROR_CHECK(uart_param_config(UART_PORT, &uart_config));

    ESP_ERROR_CHECK(uart_set_pin(
        UART_PORT,
        UART_TX_PIN,
        UART_RX_PIN,
        UART_PIN_NO_CHANGE,
        UART_PIN_NO_CHANGE
    ));

    ESP_ERROR_CHECK(uart_driver_install(
        UART_PORT,
        UART_RX_BUF_SIZE,
        UART_TX_BUF_SIZE,
        0,
        NULL,
        0
    ));

    ESP_LOGI(TAG, "UART initialized");
}

static void uart_echo_task(void *pvParameters)
{
    uint8_t data[128];

    while (1) {
        int length = uart_read_bytes(
            UART_PORT,
            data,
            sizeof(data) - 1,
            pdMS_TO_TICKS(1000)
        );

        if (length > 0) {
            data[length] = '\0';

            ESP_LOGI(TAG, "Received: %s", (char *)data);

            uart_write_bytes(UART_PORT, (const char *)data, length);
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "UART echo example started");

    uart_init();

    xTaskCreate(
        uart_echo_task,
        "UART Echo Task",
        4096,
        NULL,
        2,
        NULL
    );
}
```

---

## 8. Penjelasan UART Echo

```c
uart_param_config(UART_PORT, &uart_config);
```

Mengatur baudrate, data bits, parity, stop bit, dan flow control.

---

```c
uart_set_pin(UART_PORT, UART_TX_PIN, UART_RX_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
```

Mengatur pin TX dan RX.

Parameter keempat dan kelima adalah RTS/CTS. Karena tidak memakai hardware flow control, gunakan:

```c
UART_PIN_NO_CHANGE
```

---

```c
uart_driver_install(UART_PORT, UART_RX_BUF_SIZE, UART_TX_BUF_SIZE, 0, NULL, 0);
```

Menginstall driver UART.

Parameter penting:

```txt
UART_RX_BUF_SIZE → buffer terima
UART_TX_BUF_SIZE → buffer kirim
0                → tidak pakai event queue
NULL             → tidak menyimpan queue handle
0                → interrupt allocation flags default
```

---

```c
uart_read_bytes(..., pdMS_TO_TICKS(1000));
```

Membaca data UART dengan timeout 1 detik.

---

```c
uart_write_bytes(UART_PORT, (const char *)data, length);
```

Mengirim balik data yang diterima.

---

# Hari 3 — UART Event Queue

## 9. Kenapa Perlu UART Event Queue?

UART sederhana bisa memakai `uart_read_bytes()` langsung. Namun untuk aplikasi lebih serius, kamu perlu menangani event seperti:

- Data masuk.
- Buffer penuh.
- FIFO overflow.
- Break detected.
- Parity error.
- Frame error.

Dengan event queue, task bisa menunggu event UART.

---

## 10. Contoh UART Event Task

```c
#include <stdio.h>
#include <string.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#include "driver/uart.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define UART_PORT       UART_NUM_1
#define UART_TX_PIN     GPIO_NUM_17
#define UART_RX_PIN     GPIO_NUM_18
#define UART_BAUD_RATE  115200

#define UART_RX_BUF_SIZE 1024
#define UART_TX_BUF_SIZE 1024
#define UART_EVENT_QUEUE_SIZE 20

static const char *TAG = "UART_EVENT";

static QueueHandle_t uart_event_queue = NULL;

static void uart_init(void)
{
    uart_config_t uart_config = {
        .baud_rate = UART_BAUD_RATE,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_DEFAULT,
    };

    ESP_ERROR_CHECK(uart_param_config(UART_PORT, &uart_config));

    ESP_ERROR_CHECK(uart_set_pin(
        UART_PORT,
        UART_TX_PIN,
        UART_RX_PIN,
        UART_PIN_NO_CHANGE,
        UART_PIN_NO_CHANGE
    ));

    ESP_ERROR_CHECK(uart_driver_install(
        UART_PORT,
        UART_RX_BUF_SIZE,
        UART_TX_BUF_SIZE,
        UART_EVENT_QUEUE_SIZE,
        &uart_event_queue,
        0
    ));

    ESP_LOGI(TAG, "UART initialized with event queue");
}

static void uart_event_task(void *pvParameters)
{
    uart_event_t event;
    uint8_t data[256];

    while (1) {
        if (xQueueReceive(uart_event_queue, &event, portMAX_DELAY) == pdTRUE) {
            switch (event.type) {
                case UART_DATA: {
                    int read_len = event.size;

                    if (read_len >= sizeof(data)) {
                        read_len = sizeof(data) - 1;
                    }

                    int length = uart_read_bytes(
                        UART_PORT,
                        data,
                        read_len,
                        pdMS_TO_TICKS(100)
                    );

                    if (length > 0) {
                        data[length] = '\0';
                        ESP_LOGI(TAG, "UART DATA: %s", (char *)data);
                    }
                    break;
                }

                case UART_FIFO_OVF:
                    ESP_LOGW(TAG, "UART FIFO overflow");
                    uart_flush_input(UART_PORT);
                    xQueueReset(uart_event_queue);
                    break;

                case UART_BUFFER_FULL:
                    ESP_LOGW(TAG, "UART ring buffer full");
                    uart_flush_input(UART_PORT);
                    xQueueReset(uart_event_queue);
                    break;

                case UART_BREAK:
                    ESP_LOGW(TAG, "UART break detected");
                    break;

                case UART_PARITY_ERR:
                    ESP_LOGW(TAG, "UART parity error");
                    break;

                case UART_FRAME_ERR:
                    ESP_LOGW(TAG, "UART frame error");
                    break;

                default:
                    ESP_LOGI(TAG, "UART event type: %d", event.type);
                    break;
            }
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "UART event example started");

    uart_init();

    xTaskCreate(
        uart_event_task,
        "UART Event Task",
        4096,
        NULL,
        3,
        NULL
    );
}
```

---

## 11. Kapan Pakai UART Event Queue?

Gunakan UART event queue jika:

- Data UART masuk tidak teratur.
- Perlu handle overflow atau error.
- Perlu parser command.
- Komunikasi dengan modem, GPS, atau RS485.
- Aplikasi sudah mendekati production.

Untuk latihan awal, `uart_read_bytes()` cukup. Untuk firmware serius, event queue lebih baik.

---

## 12. UART Command Parser Sederhana

Command:

```txt
LED ON
LED OFF
STATUS
```

Parser:

```c
#include <string.h>
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2

static void process_command(const char *cmd)
{
    if (strcmp(cmd, "LED ON") == 0) {
        gpio_set_level(LED_GPIO, 1);
        ESP_LOGI("CMD", "LED turned ON");
    } else if (strcmp(cmd, "LED OFF") == 0) {
        gpio_set_level(LED_GPIO, 0);
        ESP_LOGI("CMD", "LED turned OFF");
    } else if (strcmp(cmd, "STATUS") == 0) {
        ESP_LOGI("CMD", "System OK");
    } else {
        ESP_LOGW("CMD", "Unknown command: %s", cmd);
    }
}
```

UART command cocok untuk:

- Factory test.
- Debug console.
- Konfigurasi awal.
- Komunikasi AT command.
- RS485 protocol.

---

# Hari 4 — I2C Dasar

## 13. Konsep I2C

I2C memakai dua kabel utama:

```txt
SDA → data
SCL → clock
```

Wiring:

```txt
ESP32 SDA ───── SDA Sensor
ESP32 SCL ───── SCL Sensor
ESP32 GND ───── GND Sensor
ESP32 3V3 ───── VCC Sensor
```

I2C butuh pull-up resistor:

```txt
SDA → pull-up ke 3.3V
SCL → pull-up ke 3.3V
```

Banyak breakout sensor sudah memiliki pull-up onboard.

---

## 14. Address I2C

Setiap device I2C punya alamat.

Contoh umum:

```txt
0x27 → LCD I2C umum
0x3C → OLED SSD1306 umum
0x68 → MPU6050 / DS3231 umum
0x76 → BME280/BMP280 umum
0x29 → VL53L0X umum
```

Alamat bisa berbeda tergantung modul. Karena itu latihan pertama I2C adalah **I2C scanner**.

---

## 15. I2C Scanner dengan Driver Modern ESP-IDF

Project:

```txt
week5_02_i2c_scanner
```

Pin contoh:

```txt
SDA = GPIO8
SCL = GPIO9
```

Kode:

```c
#include <stdio.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/i2c_master.h"
#include "esp_log.h"

#define I2C_MASTER_SCL_IO       GPIO_NUM_9
#define I2C_MASTER_SDA_IO       GPIO_NUM_8
#define I2C_MASTER_PORT         I2C_NUM_0
#define I2C_MASTER_FREQ_HZ      100000

static const char *TAG = "I2C_SCANNER";

static i2c_master_bus_handle_t i2c_bus_handle = NULL;

static void i2c_master_init(void)
{
    i2c_master_bus_config_t bus_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_MASTER_PORT,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };

    ESP_ERROR_CHECK(i2c_new_master_bus(&bus_config, &i2c_bus_handle));

    ESP_LOGI(TAG, "I2C master bus initialized");
}

void app_main(void)
{
    ESP_LOGI(TAG, "I2C scanner started");

    i2c_master_init();

    while (1) {
        ESP_LOGI(TAG, "Scanning I2C bus...");

        int found_count = 0;

        for (uint8_t address = 1; address < 127; address++) {
            esp_err_t ret = i2c_master_probe(
                i2c_bus_handle,
                address,
                100
            );

            if (ret == ESP_OK) {
                ESP_LOGI(TAG, "Found I2C device at address 0x%02X", address);
                found_count++;
            }
        }

        if (found_count == 0) {
            ESP_LOGW(TAG, "No I2C devices found");
        } else {
            ESP_LOGI(TAG, "Scan done, found %d device(s)", found_count);
        }

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

---

## 16. Jika I2C Scanner Tidak Menemukan Device

Cek:

- SDA/SCL tertukar.
- VCC/GND salah.
- Ground tidak common.
- Pull-up tidak ada.
- Pin yang dipakai bukan pin yang benar.
- Alamat device berbeda.
- Sensor butuh 5V power tetapi logic harus tetap aman untuk ESP32.
- Kabel terlalu panjang.

Jika bus error:

- Turunkan speed ke 100 kHz.
- Cek apakah SDA tertahan LOW.
- Cek pull-up resistor.
- Power cycle sensor.

---

# Hari 5 — I2C Register Read/Write

## 17. Konsep Register Sensor I2C

Banyak sensor I2C bekerja dengan register.

Pola membaca register:

```txt
1. Kirim alamat register
2. Baca data dari register tersebut
```

Contoh:

```txt
Device address  : 0x68
Register WHOAMI : 0x75
```

---

## 18. Helper I2C Read Register

```c
static esp_err_t i2c_read_register(
    i2c_master_dev_handle_t dev_handle,
    uint8_t reg_addr,
    uint8_t *data,
    size_t data_len
)
{
    return i2c_master_transmit_receive(
        dev_handle,
        &reg_addr,
        1,
        data,
        data_len,
        100
    );
}
```

---

## 19. Helper I2C Write Register

```c
static esp_err_t i2c_write_register(
    i2c_master_dev_handle_t dev_handle,
    uint8_t reg_addr,
    uint8_t value
)
{
    uint8_t write_buf[2] = { reg_addr, value };

    return i2c_master_transmit(
        dev_handle,
        write_buf,
        sizeof(write_buf),
        100
    );
}
```

---

## 20. Contoh Baca Register WHO_AM_I

Misal address sensor:

```txt
0x68
```

Kode:

```c
#include <stdio.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/i2c_master.h"
#include "esp_log.h"

#define I2C_MASTER_SCL_IO       GPIO_NUM_9
#define I2C_MASTER_SDA_IO       GPIO_NUM_8
#define I2C_MASTER_PORT         I2C_NUM_0
#define I2C_MASTER_FREQ_HZ      100000

#define SENSOR_ADDR             0x68
#define WHO_AM_I_REG            0x75

static const char *TAG = "I2C_REG";

static i2c_master_bus_handle_t i2c_bus_handle = NULL;
static i2c_master_dev_handle_t sensor_handle = NULL;

static void i2c_sensor_init(void)
{
    i2c_master_bus_config_t bus_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_MASTER_PORT,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };

    ESP_ERROR_CHECK(i2c_new_master_bus(&bus_config, &i2c_bus_handle));

    i2c_device_config_t dev_config = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = SENSOR_ADDR,
        .scl_speed_hz = I2C_MASTER_FREQ_HZ,
    };

    ESP_ERROR_CHECK(i2c_master_bus_add_device(
        i2c_bus_handle,
        &dev_config,
        &sensor_handle
    ));

    ESP_LOGI(TAG, "I2C sensor initialized");
}

static esp_err_t sensor_read_register(uint8_t reg_addr, uint8_t *data)
{
    return i2c_master_transmit_receive(
        sensor_handle,
        &reg_addr,
        1,
        data,
        1,
        100
    );
}

void app_main(void)
{
    ESP_LOGI(TAG, "I2C register read example started");

    i2c_sensor_init();

    while (1) {
        uint8_t who_am_i = 0;

        esp_err_t ret = sensor_read_register(WHO_AM_I_REG, &who_am_i);

        if (ret == ESP_OK) {
            ESP_LOGI(TAG, "WHO_AM_I = 0x%02X", who_am_i);
        } else {
            ESP_LOGE(TAG, "Failed to read WHO_AM_I: %s", esp_err_to_name(ret));
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 21. I2C Bus Mutex

Jika satu I2C bus dipakai banyak task, gunakan mutex.

Contoh:

```txt
Sensor Task  → baca sensor
Display Task → update LCD I2C
RTC Task     → baca DS3231
```

Semua memakai SDA/SCL yang sama.

Pola:

```c
static SemaphoreHandle_t i2c_mutex = NULL;

if (xSemaphoreTake(i2c_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
    // transaksi I2C
    xSemaphoreGive(i2c_mutex);
}
```

Tujuannya agar dua task tidak melakukan transaksi I2C bersamaan.

---

# Hari 6 — SPI Dasar

## 22. Konsep SPI

SPI memakai:

```txt
MOSI → Master Out Slave In
MISO → Master In Slave Out
SCLK → Serial Clock
CS   → Chip Select
GND  → Ground
```

Wiring:

```txt
ESP32 MOSI → Device MOSI / DI
ESP32 MISO ← Device MISO / DO
ESP32 SCLK → Device SCK
ESP32 CS   → Device CS / SS
ESP32 GND  ↔ Device GND
```

Catatan RC522:

```txt
SDA  → CS
SCK  → SCLK
MOSI → MOSI
MISO → MISO
RST  → GPIO reset
```

---

## 23. SPI Mode

SPI punya mode berdasarkan CPOL dan CPHA:

```txt
Mode 0: CPOL=0, CPHA=0
Mode 1: CPOL=0, CPHA=1
Mode 2: CPOL=1, CPHA=0
Mode 3: CPOL=1, CPHA=1
```

Banyak device memakai mode 0, tetapi selalu cek datasheet.

---

## 24. Alur SPI Master ESP-IDF

```txt
1. Konfigurasi bus SPI
2. Initialize bus dengan spi_bus_initialize()
3. Konfigurasi device SPI
4. Add device dengan spi_bus_add_device()
5. Buat transaction
6. Kirim dengan spi_device_transmit()
```

API utama:

```c
spi_bus_initialize()
spi_bus_add_device()
spi_device_transmit()
```

---

## 25. Contoh SPI Master Transaction

Project:

```txt
week5_03_spi_master_basic
```

Pin contoh:

```txt
MOSI = GPIO11
MISO = GPIO13
SCLK = GPIO12
CS   = GPIO10
```

Kode:

```c
#include <stdio.h>
#include <string.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/spi_master.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define SPI_HOST_USED   SPI2_HOST

#define PIN_NUM_MOSI    GPIO_NUM_11
#define PIN_NUM_MISO    GPIO_NUM_13
#define PIN_NUM_CLK     GPIO_NUM_12
#define PIN_NUM_CS      GPIO_NUM_10

static const char *TAG = "SPI_BASIC";

static spi_device_handle_t spi_device = NULL;

static void spi_init(void)
{
    spi_bus_config_t bus_config = {
        .mosi_io_num = PIN_NUM_MOSI,
        .miso_io_num = PIN_NUM_MISO,
        .sclk_io_num = PIN_NUM_CLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 64,
    };

    ESP_ERROR_CHECK(spi_bus_initialize(
        SPI_HOST_USED,
        &bus_config,
        SPI_DMA_CH_AUTO
    ));

    spi_device_interface_config_t dev_config = {
        .clock_speed_hz = 1 * 1000 * 1000,
        .mode = 0,
        .spics_io_num = PIN_NUM_CS,
        .queue_size = 3,
    };

    ESP_ERROR_CHECK(spi_bus_add_device(
        SPI_HOST_USED,
        &dev_config,
        &spi_device
    ));

    ESP_LOGI(TAG, "SPI initialized");
}

static esp_err_t spi_send_receive(
    const uint8_t *tx_data,
    uint8_t *rx_data,
    size_t length
)
{
    spi_transaction_t transaction = {
        .length = length * 8,
        .tx_buffer = tx_data,
        .rx_buffer = rx_data,
    };

    return spi_device_transmit(spi_device, &transaction);
}

void app_main(void)
{
    ESP_LOGI(TAG, "SPI basic example started");

    spi_init();

    while (1) {
        uint8_t tx_data[2] = { 0x9F, 0x00 };
        uint8_t rx_data[2] = { 0 };

        esp_err_t ret = spi_send_receive(tx_data, rx_data, sizeof(tx_data));

        if (ret == ESP_OK) {
            ESP_LOGI(TAG, "RX: 0x%02X 0x%02X", rx_data[0], rx_data[1]);
        } else {
            ESP_LOGE(TAG, "SPI transmit failed: %s", esp_err_to_name(ret));
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 26. Penjelasan SPI

```c
spi_bus_initialize(SPI_HOST_USED, &bus_config, SPI_DMA_CH_AUTO);
```

Menginisialisasi SPI bus.

---

```c
spi_bus_add_device(SPI_HOST_USED, &dev_config, &spi_device);
```

Menambahkan device SPI ke bus.

Jika ada banyak device:

```txt
Device 1: CS GPIO10
Device 2: CS GPIO14
Device 3: CS GPIO15
```

MOSI/MISO/SCLK sama, CS berbeda.

---

```c
spi_transaction_t transaction = {
    .length = length * 8,
    .tx_buffer = tx_data,
    .rx_buffer = rx_data,
};
```

`length` memakai satuan bit, jadi jumlah byte dikali 8.

---

## 27. SPI Register Read/Write Pattern

Banyak device SPI memakai register.

Pola write register:

```txt
Byte 1: register address + write bit
Byte 2: value
```

Pola read register:

```txt
Byte 1: register address + read bit
Byte 2: dummy byte untuk clock out data
```

Helper generik:

```c
static esp_err_t spi_write_register(uint8_t reg, uint8_t value)
{
    uint8_t tx_data[2] = { reg, value };

    spi_transaction_t transaction = {
        .length = 16,
        .tx_buffer = tx_data,
    };

    return spi_device_transmit(spi_device, &transaction);
}

static esp_err_t spi_read_register(uint8_t reg, uint8_t *value)
{
    uint8_t tx_data[2] = { reg | 0x80, 0x00 };
    uint8_t rx_data[2] = { 0 };

    spi_transaction_t transaction = {
        .length = 16,
        .tx_buffer = tx_data,
        .rx_buffer = rx_data,
    };

    esp_err_t ret = spi_device_transmit(spi_device, &transaction);

    if (ret == ESP_OK) {
        *value = rx_data[1];
    }

    return ret;
}
```

Catatan:

```txt
reg | 0x80 hanya contoh umum.
Setiap chip punya format read/write berbeda.
Selalu cek datasheet device.
```

Untuk RC522, register access memiliki format khusus:

```txt
Write address = (reg << 1) & 0x7E
Read address  = ((reg << 1) & 0x7E) | 0x80
```

---

## 28. SPI Bus Mutex

Jika SPI dipakai beberapa task, gunakan mutex.

Contoh:

```txt
RFID Task    → pakai SPI
Display Task → pakai SPI
SD Card Task → pakai SPI
```

Pola:

```c
static SemaphoreHandle_t spi_mutex = NULL;

if (xSemaphoreTake(spi_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
    spi_device_transmit(spi_device, &transaction);
    xSemaphoreGive(spi_mutex);
}
```

---

# Hari 7 — Mini Project Minggu 5

## 29. Target Mini Project

Project:

```txt
week5_final_peripheral_gateway
```

Fitur:

```txt
1. UART menerima command dari PC.
2. Command "SCAN I2C" menjalankan I2C scanner.
3. Command "LED ON" menyalakan LED.
4. Command "LED OFF" mematikan LED.
5. Command "SPI TEST" mengirim transaksi SPI dummy.
6. Monitor task print free heap.
7. Semua komunikasi memakai task dan queue.
```

Arsitektur:

```txt
UART Task
   ↓ queue command
Command Task
   ├── LED GPIO
   ├── I2C Scanner
   └── SPI Test

Monitor Task
   └── print heap/status
```

---

## 30. Tipe Event Command

```c
typedef enum {
    CMD_LED_ON,
    CMD_LED_OFF,
    CMD_SCAN_I2C,
    CMD_SPI_TEST,
    CMD_STATUS,
    CMD_UNKNOWN
} app_command_type_t;

typedef struct {
    app_command_type_t type;
    char raw[64];
} app_command_t;
```

---

## 31. Parser Command

```c
static app_command_type_t parse_command(const char *cmd)
{
    if (strcmp(cmd, "LED ON") == 0) {
        return CMD_LED_ON;
    }

    if (strcmp(cmd, "LED OFF") == 0) {
        return CMD_LED_OFF;
    }

    if (strcmp(cmd, "SCAN I2C") == 0) {
        return CMD_SCAN_I2C;
    }

    if (strcmp(cmd, "SPI TEST") == 0) {
        return CMD_SPI_TEST;
    }

    if (strcmp(cmd, "STATUS") == 0) {
        return CMD_STATUS;
    }

    return CMD_UNKNOWN;
}
```

---

## 32. UART Task Mengirim Command ke Queue

```c
static void uart_task(void *pvParameters)
{
    uint8_t data[64];

    while (1) {
        int length = uart_read_bytes(
            UART_PORT,
            data,
            sizeof(data) - 1,
            pdMS_TO_TICKS(1000)
        );

        if (length > 0) {
            data[length] = '\0';

            for (int i = 0; i < length; i++) {
                if (data[i] == '\r' || data[i] == '\n') {
                    data[i] = '\0';
                    break;
                }
            }

            app_command_t command = {
                .type = parse_command((char *)data),
            };

            strncpy(command.raw, (char *)data, sizeof(command.raw) - 1);
            command.raw[sizeof(command.raw) - 1] = '\0';

            xQueueSend(command_queue, &command, pdMS_TO_TICKS(100));
        }
    }
}
```

---

## 33. Command Task

```c
static void command_task(void *pvParameters)
{
    app_command_t command;

    while (1) {
        if (xQueueReceive(command_queue, &command, portMAX_DELAY) == pdTRUE) {
            switch (command.type) {
                case CMD_LED_ON:
                    gpio_set_level(LED_GPIO, 1);
                    ESP_LOGI("CMD", "LED ON");
                    break;

                case CMD_LED_OFF:
                    gpio_set_level(LED_GPIO, 0);
                    ESP_LOGI("CMD", "LED OFF");
                    break;

                case CMD_SCAN_I2C:
                    ESP_LOGI("CMD", "I2C scan requested");
                    i2c_scan_once();
                    break;

                case CMD_SPI_TEST:
                    ESP_LOGI("CMD", "SPI test requested");
                    spi_test_once();
                    break;

                case CMD_STATUS:
                    ESP_LOGI("CMD", "System status OK");
                    break;

                default:
                    ESP_LOGW("CMD", "Unknown command: %s", command.raw);
                    break;
            }
        }
    }
}
```

---

## 34. Struktur File yang Direkomendasikan

```txt
week5_final_peripheral_gateway/
├── CMakeLists.txt
└── main/
    ├── CMakeLists.txt
    ├── main.c
    ├── app_uart.c
    ├── app_uart.h
    ├── app_i2c.c
    ├── app_i2c.h
    ├── app_spi.c
    ├── app_spi.h
    ├── app_commands.c
    ├── app_commands.h
    ├── app_gpio.c
    └── app_gpio.h
```

`main.c` ideal:

```c
#include "esp_log.h"

#include "app_gpio.h"
#include "app_uart.h"
#include "app_i2c.h"
#include "app_spi.h"
#include "app_commands.h"

static const char *TAG = "MAIN";

void app_main(void)
{
    ESP_LOGI(TAG, "Week 5 peripheral gateway started");

    app_gpio_init();
    app_uart_init();
    app_i2c_init();
    app_spi_init();
    app_commands_start();
    app_uart_start();
}
```

`main.c` jangan terlalu penuh. Detail peripheral sebaiknya ada di file masing-masing.

---

# 35. Error Umum UART

## Tidak Ada Data Masuk

Cek:

- TX/RX belum silang.
- GND belum common.
- Baudrate beda.
- Salah UART port.
- Salah pin.
- Modul belum powered.
- Level logic 5V/3.3V bermasalah.

## Data Aneh atau Karakter Rusak

Kemungkinan:

- Baudrate salah.
- GND tidak tersambung.
- Noise kabel.
- Level tegangan tidak cocok.
- Format parity/stop bit beda.

## UART0 Bentrok dengan Monitor

UART0 sering dipakai untuk log serial dan flashing.

Untuk komunikasi eksternal, lebih aman pakai:

```txt
UART_NUM_1
UART_NUM_2 jika tersedia pada chip/board
```

---

# 36. Error Umum I2C

## Device Tidak Terdeteksi Scanner

Cek:

- SDA/SCL tertukar.
- Pull-up tidak ada.
- Alamat device berbeda.
- Power sensor salah.
- GND belum tersambung.
- Pin ESP32 salah.
- Sensor rusak.
- Kabel terlalu panjang.

## I2C Timeout

Kemungkinan:

- SDA tertahan LOW.
- Device hang.
- Pull-up bermasalah.
- Bus speed terlalu tinggi.
- Kabel terlalu panjang.

Solusi:

- Turunkan speed ke 100 kHz.
- Cek pull-up 4.7k–10k.
- Power cycle sensor.
- Cek wiring dengan multimeter.

---

# 37. Error Umum SPI

## Tidak Ada Response

Cek:

- MOSI/MISO tertukar.
- CS salah.
- SPI mode salah.
- Clock terlalu cepat.
- Device belum reset.
- GND belum common.
- Format register salah.

## Data Selalu 0xFF atau 0x00

Kemungkinan 0xFF:

- MISO floating.
- Device tidak aktif.
- CS tidak benar.

Kemungkinan 0x00:

- Device tidak memberi data.
- Wiring salah.
- Register read salah.

## SPI Clock Terlalu Cepat

Untuk awal, jangan langsung memakai clock tinggi.

Mulai dari:

```txt
100 kHz
1 MHz
```

Setelah stabil, naikkan:

```txt
4 MHz
8 MHz
10 MHz+
```

Tergantung device.

---

# 38. Best Practice Minggu 5

## UART

- Cocok untuk komunikasi serial point-to-point.
- Selalu cek baudrate.
- TX/RX harus silang.
- Gunakan event queue untuk aplikasi serius.
- Jangan lupa buffer cukup.

## I2C

- Pastikan ada pull-up.
- Gunakan scanner untuk debugging awal.
- Gunakan mutex jika bus dipakai banyak task.
- Mulai dari 100 kHz dulu.
- Jangan akses I2C bersamaan dari banyak task tanpa pengaman.

## SPI

- Pastikan CS benar.
- Mulai dari clock rendah.
- Cek SPI mode dari datasheet.
- Gunakan mutex untuk shared SPI bus.
- Pisahkan helper register read/write per device.

---

# 39. Latihan Minggu 5

## Latihan 1 — UART Echo

Buat ESP32 menerima data UART dan mengirim balik data tersebut.

Target:

```txt
Ketik "hello" dari serial tool
ESP32 membalas "hello"
```

---

## Latihan 2 — UART Command

Buat command:

```txt
LED ON
LED OFF
STATUS
```

Target:

```txt
Command dari UART bisa mengontrol LED.
```

---

## Latihan 3 — I2C Scanner

Buat scanner yang mencetak semua alamat I2C yang ditemukan.

Target output:

```txt
Found I2C device at 0x3C
Found I2C device at 0x68
```

---

## Latihan 4 — I2C Register Read

Baca register dari sensor I2C.

Target:

```txt
WHO_AM_I = 0xXX
```

Jika belum punya sensor register, cukup jalankan scanner dulu.

---

## Latihan 5 — SPI Dummy Transaction

Kirim byte dummy lewat SPI dan log hasil RX.

Target:

```txt
TX: 0x9F 0x00
RX: 0x?? 0x??
```

---

## Latihan 6 — SPI Register Helper

Buat helper:

```c
spi_read_register()
spi_write_register()
```

Target:

```txt
Siap dipakai untuk device seperti RC522 atau display.
```

---

## Latihan 7 — Peripheral Gateway

Buat UART command:

```txt
SCAN I2C
SPI TEST
LED ON
LED OFF
STATUS
```

Target:

```txt
UART menjadi console untuk menjalankan peripheral test.
```

---

# 40. Tugas Akhir Minggu 5

Buat project:

```txt
week5_final_uart_i2c_spi_gateway
```

Fitur wajib:

```txt
1. UART command parser.
2. Command LED ON.
3. Command LED OFF.
4. Command STATUS.
5. Command SCAN I2C.
6. Command SPI TEST.
7. I2C scanner berjalan saat command diterima.
8. SPI dummy transaction berjalan saat command diterima.
9. Monitor task print free heap setiap 5 detik.
10. Struktur file terpisah:
    - app_uart.c/h
    - app_i2c.c/h
    - app_spi.c/h
    - app_gpio.c/h
    - app_commands.c/h
```

---

# 41. Checklist Lulus Minggu 5

Kamu dianggap lulus Minggu 5 kalau bisa:

```txt
[ ] Menjelaskan perbedaan UART, I2C, SPI
[ ] Menjelaskan wiring UART TX/RX silang
[ ] Menjelaskan kenapa GND harus common
[ ] Menginisialisasi UART ESP-IDF
[ ] Membaca data UART
[ ] Mengirim data UART
[ ] Membuat UART command parser
[ ] Menjelaskan SDA/SCL dan pull-up I2C
[ ] Membuat I2C scanner
[ ] Membaca register I2C
[ ] Menulis register I2C
[ ] Menjelaskan MOSI/MISO/SCLK/CS
[ ] Menginisialisasi SPI master
[ ] Menambahkan SPI device
[ ] Mengirim SPI transaction
[ ] Membuat helper SPI register read/write
[ ] Menggunakan mutex untuk shared bus
[ ] Membuat mini project UART + I2C + SPI
```

---

# 42. Penerapan untuk Project Stunting IoT

Mapping peripheral untuk project stunting:

| Modul | Protokol |
|---|---|
| RFID RC522 | SPI |
| LCD I2C | I2C |
| VL53L0X | I2C |
| GPS | UART |
| RS485/Modbus | UART |
| Debug command | UART |
| Display TFT | SPI |

Arsitektur contoh:

```txt
RFID Task
   ↓ SPI
App Task

VL53L0X Task
   ↓ I2C
App Task

UART Command Task
   ↓ Queue
Command Task

App Task
   ↓ Network/Storage
Backend / Offline Queue
```

---

# 43. Inti Pemahaman Minggu 5

Kalimat penting:

> Peripheral communication bukan hanya soal kode, tetapi juga wiring, level tegangan, timing, address/register, dan error handling.

Pola berpikir:

```txt
UART:
Data serial masuk → UART task → parser → command

I2C:
Task → ambil mutex bus → transmit/register read → release mutex

SPI:
Task → pilih device via CS → transaction → proses response
```

Setelah Minggu 5 kuat, kamu siap masuk Minggu 6:

```txt
ADC, PWM/LEDC, RMT, dan kontrol sinyal analog/digital
```
