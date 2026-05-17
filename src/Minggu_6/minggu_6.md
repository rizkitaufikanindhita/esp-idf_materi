# Minggu 6 — ESP-IDF ADC, PWM/LEDC, RMT, dan Kontrol Sinyal

## Target Minggu 6

Pada akhir minggu ini, kamu diharapkan mampu:

1. Memahami fungsi ADC, PWM/LEDC, dan RMT pada ESP-IDF.
2. Membaca sinyal analog menggunakan ADC oneshot driver.
3. Menggunakan ADC calibration untuk mengubah nilai raw menjadi tegangan mV.
4. Membuat filter sederhana seperti moving average untuk menstabilkan pembacaan ADC.
5. Menghasilkan PWM menggunakan LEDC.
6. Mengatur brightness LED dengan duty cycle.
7. Mengontrol servo dan buzzer menggunakan LEDC.
8. Memahami fungsi dasar RMT untuk sinyal pulse yang butuh timing presisi.
9. Membuat mini project: potensiometer/ADC mengontrol brightness LED PWM.

---

## 1. Gambaran Besar Minggu 6

| Hari | Materi | Output |
|---|---|---|
| Hari 1 | Konsep ADC, PWM, RMT | Paham fungsi dan kapan digunakan |
| Hari 2 | ADC basic | Bisa membaca nilai analog/raw |
| Hari 3 | ADC calibration dan filtering | Bisa membaca tegangan mV dan menstabilkan data |
| Hari 4 | PWM dengan LEDC | Bisa mengatur brightness LED |
| Hari 5 | Servo dan buzzer | Bisa menghasilkan sinyal PWM untuk aktuator sederhana |
| Hari 6 | RMT dasar | Paham sinyal pulse presisi |
| Hari 7 | Mini project | ADC mengontrol PWM LED |

---

## 2. Konsep Dasar

### 2.1 ADC

ADC adalah **Analog-to-Digital Converter**.

Fungsinya:

```txt
Tegangan analog → nilai digital
```

Contoh:

```txt
0.0V  → nilai kecil
1.65V → nilai tengah
3.3V  → nilai besar
```

ADC digunakan untuk:

- Potensiometer
- Sensor baterai
- LDR
- NTC thermistor
- Joystick analog
- Sensor tekanan analog
- Sensor analog 0–3.3V

Penting:

```txt
Jangan pernah memberi input 5V langsung ke pin ADC ESP32.
```

Jika ingin membaca tegangan lebih tinggi dari batas ADC, gunakan **voltage divider**.

---

### 2.2 PWM / LEDC

PWM adalah **Pulse Width Modulation**.

PWM mengatur rasio waktu sinyal HIGH dan LOW.

```txt
0% duty   → selalu OFF
25% duty  → ON seperempat periode
50% duty  → ON setengah periode
75% duty  → ON tiga perempat periode
100% duty → selalu ON
```

PWM digunakan untuk:

- Brightness LED
- Speed motor DC melalui driver
- Servo
- Passive buzzer
- Backlight LCD

Di ESP-IDF, PWM biasanya dibuat menggunakan peripheral **LEDC**.

---

### 2.3 RMT

RMT adalah **Remote Control Transceiver**.

RMT cocok untuk membuat atau membaca sinyal pulse dengan timing presisi.

Digunakan untuk:

- IR remote
- WS2812/NeoPixel
- Protocol pulse custom
- Timing one-wire-like
- Sinyal digital presisi

RMT lebih cocok daripada bit-banging manual jika timing harus presisi.

---

## 3. Hari 1 — Perbedaan ADC, PWM, dan RMT

| Peripheral | Arah | Fungsi |
|---|---|---|
| ADC | Input | Membaca tegangan analog |
| PWM/LEDC | Output | Menghasilkan sinyal duty cycle |
| RMT | Input/Output | Membaca/membuat pulse timing presisi |

Contoh hubungan:

```txt
Potensiometer → ADC → nilai 0–100% → PWM LED brightness
```

---

## 4. Hari 2 — ADC Basic dengan Oneshot Driver

### 4.1 Header ADC

```c
#include "esp_adc/adc_oneshot.h"
```

### 4.2 Contoh ADC Read Raw

Project:

```txt
week6_01_adc_raw
```

Kode:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "esp_adc/adc_oneshot.h"

#define ADC_UNIT        ADC_UNIT_1
#define ADC_CHANNEL     ADC_CHANNEL_0
#define ADC_ATTEN       ADC_ATTEN_DB_12
#define ADC_BITWIDTH    ADC_BITWIDTH_DEFAULT

static const char *TAG = "ADC_RAW";
static adc_oneshot_unit_handle_t adc_handle;

static void adc_init(void)
{
    adc_oneshot_unit_init_cfg_t init_config = {
        .unit_id = ADC_UNIT,
    };

    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config, &adc_handle));

    adc_oneshot_chan_cfg_t channel_config = {
        .bitwidth = ADC_BITWIDTH,
        .atten = ADC_ATTEN,
    };

    ESP_ERROR_CHECK(adc_oneshot_config_channel(
        adc_handle,
        ADC_CHANNEL,
        &channel_config
    ));

    ESP_LOGI(TAG, "ADC initialized");
}

void app_main(void)
{
    ESP_LOGI(TAG, "ADC raw example started");

    adc_init();

    while (1) {
        int raw = 0;

        esp_err_t ret = adc_oneshot_read(
            adc_handle,
            ADC_CHANNEL,
            &raw
        );

        if (ret == ESP_OK) {
            ESP_LOGI(TAG, "ADC raw = %d", raw);
        } else {
            ESP_LOGE(TAG, "ADC read failed: %s", esp_err_to_name(ret));
        }

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

---

## 5. Penjelasan ADC Basic

### 5.1 `adc_oneshot_new_unit()`

Membuat handle ADC unit.

```c
adc_oneshot_new_unit(&init_config, &adc_handle);
```

---

### 5.2 `adc_oneshot_config_channel()`

Mengatur channel ADC.

```c
adc_oneshot_config_channel(adc_handle, ADC_CHANNEL, &channel_config);
```

---

### 5.3 `adc_oneshot_read()`

Membaca nilai ADC raw.

```c
adc_oneshot_read(adc_handle, ADC_CHANNEL, &raw);
```

---

### 5.4 Attenuation

Attenuation mengatur rentang tegangan input ADC.

Umum digunakan:

```c
ADC_ATTEN_DB_12
```

Untuk potensiometer 0–3.3V, biasanya gunakan `ADC_ATTEN_DB_12`.

---

## 6. Hari 3 — ADC Calibration

Nilai raw ADC belum langsung berarti tegangan.

Contoh:

```txt
raw = 2048
```

Belum tentu sama dengan tepat 1650 mV.

Dengan calibration:

```txt
raw ADC → voltage mV
```

### 6.1 Header Calibration

```c
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"
```

### 6.2 Contoh ADC Calibration

```c
#include <stdio.h>
#include <stdbool.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_log.h"
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

#define ADC_UNIT        ADC_UNIT_1
#define ADC_CHANNEL     ADC_CHANNEL_0
#define ADC_ATTEN       ADC_ATTEN_DB_12
#define ADC_BITWIDTH    ADC_BITWIDTH_DEFAULT

static const char *TAG = "ADC_CALI";

static adc_oneshot_unit_handle_t adc_handle;
static adc_cali_handle_t adc_cali_handle = NULL;
static bool adc_calibration_enabled = false;

static bool adc_calibration_init(void)
{
    adc_cali_line_fitting_config_t cali_config = {
        .unit_id = ADC_UNIT,
        .atten = ADC_ATTEN,
        .bitwidth = ADC_BITWIDTH,
    };

    esp_err_t ret = adc_cali_create_scheme_line_fitting(
        &cali_config,
        &adc_cali_handle
    );

    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "ADC calibration enabled");
        return true;
    }

    ESP_LOGW(TAG, "ADC calibration unavailable: %s", esp_err_to_name(ret));
    return false;
}

static void adc_init(void)
{
    adc_oneshot_unit_init_cfg_t init_config = {
        .unit_id = ADC_UNIT,
    };

    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config, &adc_handle));

    adc_oneshot_chan_cfg_t channel_config = {
        .bitwidth = ADC_BITWIDTH,
        .atten = ADC_ATTEN,
    };

    ESP_ERROR_CHECK(adc_oneshot_config_channel(
        adc_handle,
        ADC_CHANNEL,
        &channel_config
    ));

    adc_calibration_enabled = adc_calibration_init();
}

void app_main(void)
{
    adc_init();

    while (1) {
        int raw = 0;
        int voltage_mv = 0;

        if (adc_oneshot_read(adc_handle, ADC_CHANNEL, &raw) == ESP_OK) {
            if (adc_calibration_enabled) {
                if (adc_cali_raw_to_voltage(adc_cali_handle, raw, &voltage_mv) == ESP_OK) {
                    ESP_LOGI(TAG, "raw=%d, voltage=%d mV", raw, voltage_mv);
                }
            } else {
                ESP_LOGI(TAG, "raw=%d", raw);
            }
        }

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

---

## 7. ADC Filtering — Moving Average

ADC sering noise. Solusi sederhana: baca beberapa kali lalu ambil rata-rata.

```c
static int adc_read_average_raw(int sample_count)
{
    int total = 0;
    int valid_count = 0;

    for (int i = 0; i < sample_count; i++) {
        int raw = 0;

        if (adc_oneshot_read(adc_handle, ADC_CHANNEL, &raw) == ESP_OK) {
            total += raw;
            valid_count++;
        }

        vTaskDelay(pdMS_TO_TICKS(2));
    }

    if (valid_count == 0) {
        return 0;
    }

    return total / valid_count;
}
```

Pemakaian:

```c
int raw_avg = adc_read_average_raw(16);
```

Manfaat:

- Nilai lebih stabil
- Noise berkurang
- Cocok untuk baterai dan potensiometer

Kekurangan:

- Respons lebih lambat
- Ada delay sampling

---

## 8. Voltage Divider untuk Membaca Baterai

ESP32 tidak boleh membaca baterai Li-ion 4.2V langsung.

Gunakan divider:

```txt
Battery+ ── R1 ──┬── ADC
                 |
                R2
                 |
                GND
```

Rumus:

```txt
Vadc = Vbat × R2 / (R1 + R2)
```

Rumus balik:

```txt
Vbat = Vadc × (R1 + R2) / R2
```

Contoh:

```txt
R1 = 100k
R2 = 100k
Vbat = Vadc × 2
```

Kode:

```c
static float calculate_battery_voltage_mv(float adc_voltage_mv)
{
    const float r1 = 100000.0f;
    const float r2 = 100000.0f;

    return adc_voltage_mv * (r1 + r2) / r2;
}
```

---

## 9. Hari 4 — PWM dengan LEDC

### 9.1 Konsep LEDC

LEDC punya dua bagian:

```txt
Timer   → menentukan frequency dan resolution
Channel → menghasilkan PWM ke GPIO
```

Alur:

```txt
1. Configure LEDC timer
2. Configure LEDC channel
3. Set duty
4. Update duty
```

---

### 9.2 Contoh LED Fade PWM

Project:

```txt
week6_03_ledc_pwm_led
```

Kode:

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/ledc.h"
#include "esp_log.h"

#define PWM_GPIO            GPIO_NUM_2
#define LEDC_TIMER          LEDC_TIMER_0
#define LEDC_MODE           LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL        LEDC_CHANNEL_0
#define LEDC_DUTY_RES       LEDC_TIMER_10_BIT
#define LEDC_FREQUENCY      5000

static const char *TAG = "LEDC_PWM";

static void ledc_init(void)
{
    ledc_timer_config_t timer_config = {
        .speed_mode = LEDC_MODE,
        .timer_num = LEDC_TIMER,
        .duty_resolution = LEDC_DUTY_RES,
        .freq_hz = LEDC_FREQUENCY,
        .clk_cfg = LEDC_AUTO_CLK,
    };

    ESP_ERROR_CHECK(ledc_timer_config(&timer_config));

    ledc_channel_config_t channel_config = {
        .speed_mode = LEDC_MODE,
        .channel = LEDC_CHANNEL,
        .timer_sel = LEDC_TIMER,
        .intr_type = LEDC_INTR_DISABLE,
        .gpio_num = PWM_GPIO,
        .duty = 0,
        .hpoint = 0,
    };

    ESP_ERROR_CHECK(ledc_channel_config(&channel_config));
}

static void ledc_set_percent(uint32_t percent)
{
    if (percent > 100) {
        percent = 100;
    }

    uint32_t max_duty = (1 << 10) - 1;
    uint32_t duty = (max_duty * percent) / 100;

    ESP_ERROR_CHECK(ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, duty));
    ESP_ERROR_CHECK(ledc_update_duty(LEDC_MODE, LEDC_CHANNEL));
}

void app_main(void)
{
    ESP_LOGI(TAG, "LEDC PWM LED example started");

    ledc_init();

    while (1) {
        for (int percent = 0; percent <= 100; percent += 5) {
            ledc_set_percent(percent);
            ESP_LOGI(TAG, "Brightness: %d%%", percent);
            vTaskDelay(pdMS_TO_TICKS(100));
        }

        for (int percent = 100; percent >= 0; percent -= 5) {
            ledc_set_percent(percent);
            ESP_LOGI(TAG, "Brightness: %d%%", percent);
            vTaskDelay(pdMS_TO_TICKS(100));
        }
    }
}
```

---

## 10. Penjelasan LEDC

### 10.1 Duty Resolution

Kode memakai:

```c
LEDC_TIMER_10_BIT
```

Artinya duty range:

```txt
0 sampai 1023
```

Rumus:

```txt
max_duty = 2^10 - 1 = 1023
```

Jika 50%:

```txt
duty = 1023 × 50 / 100 = 511
```

---

### 10.2 Frequency vs Resolution

PWM punya trade-off:

```txt
Frequency tinggi → resolution maksimum lebih rendah
Resolution tinggi → frequency maksimum lebih rendah
```

Rekomendasi awal:

| Kebutuhan | Frequency | Resolution |
|---|---:|---:|
| LED brightness | 1–5 kHz | 10 bit |
| Servo | 50 Hz | 14 bit |
| Buzzer | sesuai nada | 10 bit |

---

## 11. Hari 5 — Servo dengan LEDC

Servo hobby biasanya memakai:

```txt
Frequency: 50 Hz
Period   : 20 ms
Pulse    : 500–2500 us
```

Contoh sudut:

```txt
0°   → 500 us
90°  → 1500 us
180° → 2500 us
```

### 11.1 Contoh Servo Sweep

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/ledc.h"
#include "esp_log.h"

#define SERVO_GPIO          GPIO_NUM_2

#define LEDC_TIMER          LEDC_TIMER_0
#define LEDC_MODE           LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL        LEDC_CHANNEL_0
#define LEDC_DUTY_RES       LEDC_TIMER_14_BIT
#define SERVO_FREQ_HZ       50

#define SERVO_MIN_US        500
#define SERVO_MAX_US        2500
#define SERVO_PERIOD_US     20000

static const char *TAG = "SERVO";

static void servo_init(void)
{
    ledc_timer_config_t timer_config = {
        .speed_mode = LEDC_MODE,
        .timer_num = LEDC_TIMER,
        .duty_resolution = LEDC_DUTY_RES,
        .freq_hz = SERVO_FREQ_HZ,
        .clk_cfg = LEDC_AUTO_CLK,
    };

    ESP_ERROR_CHECK(ledc_timer_config(&timer_config));

    ledc_channel_config_t channel_config = {
        .speed_mode = LEDC_MODE,
        .channel = LEDC_CHANNEL,
        .timer_sel = LEDC_TIMER,
        .intr_type = LEDC_INTR_DISABLE,
        .gpio_num = SERVO_GPIO,
        .duty = 0,
        .hpoint = 0,
    };

    ESP_ERROR_CHECK(ledc_channel_config(&channel_config));
}

static void servo_set_angle(uint32_t angle)
{
    if (angle > 180) {
        angle = 180;
    }

    uint32_t pulse_us = SERVO_MIN_US +
        ((SERVO_MAX_US - SERVO_MIN_US) * angle) / 180;

    uint32_t max_duty = (1 << 14) - 1;
    uint32_t duty = (max_duty * pulse_us) / SERVO_PERIOD_US;

    ESP_ERROR_CHECK(ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, duty));
    ESP_ERROR_CHECK(ledc_update_duty(LEDC_MODE, LEDC_CHANNEL));

    ESP_LOGI(TAG, "Angle=%lu, pulse=%lu us, duty=%lu", angle, pulse_us, duty);
}

void app_main(void)
{
    servo_init();

    while (1) {
        servo_set_angle(0);
        vTaskDelay(pdMS_TO_TICKS(1000));

        servo_set_angle(90);
        vTaskDelay(pdMS_TO_TICKS(1000));

        servo_set_angle(180);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

Penting:

```txt
Jangan supply servo dari pin 3.3V ESP32.
Gunakan supply eksternal dan common ground.
```

---

## 12. Passive Buzzer dengan LEDC

Passive buzzer perlu sinyal PWM.

Frequency menentukan nada.

Contoh nada:

```txt
262 Hz → C4
294 Hz → D4
330 Hz → E4
349 Hz → F4
392 Hz → G4
440 Hz → A4
494 Hz → B4
523 Hz → C5
```

### 12.1 Contoh Buzzer

```c
#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/ledc.h"
#include "esp_log.h"

#define BUZZER_GPIO         GPIO_NUM_2

#define LEDC_TIMER          LEDC_TIMER_0
#define LEDC_MODE           LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL        LEDC_CHANNEL_0
#define LEDC_DUTY_RES       LEDC_TIMER_10_BIT

static const char *TAG = "BUZZER";

static void buzzer_init(uint32_t freq_hz)
{
    ledc_timer_config_t timer_config = {
        .speed_mode = LEDC_MODE,
        .timer_num = LEDC_TIMER,
        .duty_resolution = LEDC_DUTY_RES,
        .freq_hz = freq_hz,
        .clk_cfg = LEDC_AUTO_CLK,
    };

    ESP_ERROR_CHECK(ledc_timer_config(&timer_config));

    ledc_channel_config_t channel_config = {
        .speed_mode = LEDC_MODE,
        .channel = LEDC_CHANNEL,
        .timer_sel = LEDC_TIMER,
        .intr_type = LEDC_INTR_DISABLE,
        .gpio_num = BUZZER_GPIO,
        .duty = 0,
        .hpoint = 0,
    };

    ESP_ERROR_CHECK(ledc_channel_config(&channel_config));
}

static void buzzer_play(uint32_t freq_hz, uint32_t duration_ms)
{
    uint32_t max_duty = (1 << 10) - 1;
    uint32_t duty_50_percent = max_duty / 2;

    ESP_ERROR_CHECK(ledc_set_freq(LEDC_MODE, LEDC_TIMER, freq_hz));
    ESP_ERROR_CHECK(ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, duty_50_percent));
    ESP_ERROR_CHECK(ledc_update_duty(LEDC_MODE, LEDC_CHANNEL));

    ESP_LOGI(TAG, "Play %lu Hz", freq_hz);

    vTaskDelay(pdMS_TO_TICKS(duration_ms));

    ESP_ERROR_CHECK(ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, 0));
    ESP_ERROR_CHECK(ledc_update_duty(LEDC_MODE, LEDC_CHANNEL));
}

void app_main(void)
{
    buzzer_init(440);

    while (1) {
        buzzer_play(262, 300);
        vTaskDelay(pdMS_TO_TICKS(100));

        buzzer_play(330, 300);
        vTaskDelay(pdMS_TO_TICKS(100));

        buzzer_play(392, 300);
        vTaskDelay(pdMS_TO_TICKS(100));

        buzzer_play(523, 500);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 13. Hari 6 — RMT Dasar

RMT digunakan untuk sinyal pulse dengan timing presisi.

Contoh sinyal:

```txt
HIGH 800 ns
LOW  450 ns
HIGH 400 ns
LOW  850 ns
```

Jika pakai `gpio_set_level()` dan delay manual, timing bisa tidak stabil karena FreeRTOS scheduling.

Gunakan RMT untuk:

- IR remote
- WS2812/NeoPixel
- Pulse protocol custom
- Timing digital presisi

Konsep RMT TX:

```txt
Task menyiapkan array durasi pulse
↓
RMT hardware mengirim waveform
```

Untuk Minggu 6, cukup pahami konsep. Implementasi WS2812/IR bisa dibuat di minggu/project lanjutan.

---

## 14. Hari 7 — Mini Project: ADC Mengontrol PWM LED

Project:

```txt
week6_adc_pwm_control
```

Fitur:

1. ADC membaca potensiometer.
2. Nilai ADC difilter moving average.
3. Nilai ADC dikonversi menjadi 0–100%.
4. LEDC PWM mengatur brightness LED.
5. Monitor task print raw ADC, voltage, percent, duty, dan heap.
6. ADC task mengirim data ke PWM task melalui queue.
7. Shared status dilindungi mutex.

---

## 15. Full Code Mini Project

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdint.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"

#include "esp_log.h"
#include "esp_heap_caps.h"

#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

#include "driver/ledc.h"

#define ADC_UNIT        ADC_UNIT_1
#define ADC_CHANNEL     ADC_CHANNEL_0
#define ADC_ATTEN       ADC_ATTEN_DB_12
#define ADC_BITWIDTH    ADC_BITWIDTH_DEFAULT

#define PWM_GPIO            GPIO_NUM_2
#define LEDC_TIMER          LEDC_TIMER_0
#define LEDC_MODE           LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL        LEDC_CHANNEL_0
#define LEDC_DUTY_RES       LEDC_TIMER_10_BIT
#define LEDC_FREQUENCY      5000

#define ADC_TASK_STACK      4096
#define PWM_TASK_STACK      2048
#define MONITOR_TASK_STACK  3072

#define ADC_TASK_PRIORITY      3
#define PWM_TASK_PRIORITY      2
#define MONITOR_TASK_PRIORITY  1

typedef struct {
    int raw;
    int voltage_mv;
    uint32_t percent;
} adc_data_t;

typedef struct {
    int raw;
    int voltage_mv;
    uint32_t percent;
    uint32_t pwm_duty;
} app_status_t;

static const char *TAG = "WEEK6_APP";

static adc_oneshot_unit_handle_t adc_handle;
static adc_cali_handle_t adc_cali_handle = NULL;
static bool adc_calibration_enabled = false;

static QueueHandle_t adc_queue = NULL;
static SemaphoreHandle_t status_mutex = NULL;

static app_status_t app_status = {
    .raw = 0,
    .voltage_mv = 0,
    .percent = 0,
    .pwm_duty = 0
};

static bool adc_calibration_init(void)
{
    adc_cali_line_fitting_config_t cali_config = {
        .unit_id = ADC_UNIT,
        .atten = ADC_ATTEN,
        .bitwidth = ADC_BITWIDTH,
    };

    esp_err_t ret = adc_cali_create_scheme_line_fitting(
        &cali_config,
        &adc_cali_handle
    );

    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "ADC calibration enabled");
        return true;
    }

    ESP_LOGW(TAG, "ADC calibration unavailable: %s", esp_err_to_name(ret));
    return false;
}

static void adc_init(void)
{
    adc_oneshot_unit_init_cfg_t init_config = {
        .unit_id = ADC_UNIT,
    };

    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config, &adc_handle));

    adc_oneshot_chan_cfg_t channel_config = {
        .bitwidth = ADC_BITWIDTH,
        .atten = ADC_ATTEN,
    };

    ESP_ERROR_CHECK(adc_oneshot_config_channel(
        adc_handle,
        ADC_CHANNEL,
        &channel_config
    ));

    adc_calibration_enabled = adc_calibration_init();
}

static void pwm_init(void)
{
    ledc_timer_config_t timer_config = {
        .speed_mode = LEDC_MODE,
        .timer_num = LEDC_TIMER,
        .duty_resolution = LEDC_DUTY_RES,
        .freq_hz = LEDC_FREQUENCY,
        .clk_cfg = LEDC_AUTO_CLK,
    };

    ESP_ERROR_CHECK(ledc_timer_config(&timer_config));

    ledc_channel_config_t channel_config = {
        .speed_mode = LEDC_MODE,
        .channel = LEDC_CHANNEL,
        .timer_sel = LEDC_TIMER,
        .intr_type = LEDC_INTR_DISABLE,
        .gpio_num = PWM_GPIO,
        .duty = 0,
        .hpoint = 0,
    };

    ESP_ERROR_CHECK(ledc_channel_config(&channel_config));
}

static int adc_read_average_raw(int sample_count)
{
    int total = 0;
    int valid_count = 0;

    for (int i = 0; i < sample_count; i++) {
        int raw = 0;

        if (adc_oneshot_read(adc_handle, ADC_CHANNEL, &raw) == ESP_OK) {
            total += raw;
            valid_count++;
        }

        vTaskDelay(pdMS_TO_TICKS(2));
    }

    if (valid_count == 0) {
        return 0;
    }

    return total / valid_count;
}

static uint32_t raw_to_percent(int raw)
{
    const int raw_min = 0;
    const int raw_max = 4095;

    if (raw < raw_min) {
        raw = raw_min;
    }

    if (raw > raw_max) {
        raw = raw_max;
    }

    return ((uint32_t)(raw - raw_min) * 100) / (raw_max - raw_min);
}

static uint32_t percent_to_duty(uint32_t percent)
{
    if (percent > 100) {
        percent = 100;
    }

    uint32_t max_duty = (1 << 10) - 1;
    return (max_duty * percent) / 100;
}

static void status_update(const adc_data_t *data, uint32_t duty)
{
    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        app_status.raw = data->raw;
        app_status.voltage_mv = data->voltage_mv;
        app_status.percent = data->percent;
        app_status.pwm_duty = duty;
        xSemaphoreGive(status_mutex);
    }
}

static bool status_get(app_status_t *out)
{
    if (out == NULL) {
        return false;
    }

    if (xSemaphoreTake(status_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        *out = app_status;
        xSemaphoreGive(status_mutex);
        return true;
    }

    return false;
}

static void adc_task(void *pvParameters)
{
    while (1) {
        adc_data_t data = {
            .raw = 0,
            .voltage_mv = 0,
            .percent = 0
        };

        data.raw = adc_read_average_raw(16);

        if (adc_calibration_enabled) {
            if (adc_cali_raw_to_voltage(
                    adc_cali_handle,
                    data.raw,
                    &data.voltage_mv
                ) != ESP_OK) {
                data.voltage_mv = 0;
            }
        }

        data.percent = raw_to_percent(data.raw);

        if (xQueueSend(adc_queue, &data, pdMS_TO_TICKS(100)) != pdTRUE) {
            ESP_LOGW("ADC_TASK", "ADC queue full");
        }

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

static void pwm_task(void *pvParameters)
{
    adc_data_t data;

    while (1) {
        if (xQueueReceive(adc_queue, &data, portMAX_DELAY) == pdTRUE) {
            uint32_t duty = percent_to_duty(data.percent);

            ESP_ERROR_CHECK(ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, duty));
            ESP_ERROR_CHECK(ledc_update_duty(LEDC_MODE, LEDC_CHANNEL));

            status_update(&data, duty);
        }
    }
}

static void monitor_task(void *pvParameters)
{
    app_status_t local_status;

    while (1) {
        if (status_get(&local_status)) {
            uint32_t free_heap = heap_caps_get_free_size(MALLOC_CAP_DEFAULT);
            UBaseType_t stack_left = uxTaskGetStackHighWaterMark(NULL);

            ESP_LOGI("MONITOR_TASK",
                     "raw=%d, voltage=%d mV, percent=%lu%%, duty=%lu, free_heap=%lu, stack_left=%u",
                     local_status.raw,
                     local_status.voltage_mv,
                     local_status.percent,
                     local_status.pwm_duty,
                     free_heap,
                     stack_left);
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Week 6 ADC PWM control project started");

    adc_queue = xQueueCreate(5, sizeof(adc_data_t));
    if (adc_queue == NULL) {
        ESP_LOGE(TAG, "Failed to create ADC queue");
        return;
    }

    status_mutex = xSemaphoreCreateMutex();
    if (status_mutex == NULL) {
        ESP_LOGE(TAG, "Failed to create status mutex");
        return;
    }

    adc_init();
    pwm_init();

    xTaskCreate(adc_task, "ADC Task", ADC_TASK_STACK, NULL, ADC_TASK_PRIORITY, NULL);
    xTaskCreate(pwm_task, "PWM Task", PWM_TASK_STACK, NULL, PWM_TASK_PRIORITY, NULL);
    xTaskCreate(monitor_task, "Monitor Task", MONITOR_TASK_STACK, NULL, MONITOR_TASK_PRIORITY, NULL);
}
```

---

## 16. Error Umum ADC

### Nilai ADC Selalu 0

Kemungkinan:

- Salah channel ADC
- Pin bukan ADC
- Sensor tidak diberi power
- GND tidak common
- Input benar-benar 0V
- Wiring salah

---

### Nilai ADC Selalu Maksimum

Kemungkinan:

- Input tersambung ke 3.3V
- Input floating
- Attenuation tidak sesuai
- Tegangan input terlalu tinggi
- Voltage divider salah

---

### Nilai ADC Loncat-loncat

Kemungkinan:

- Noise
- Kabel panjang
- Input floating
- Tidak ada filter
- Power supply tidak stabil

Solusi:

- Moving average
- ADC calibration
- Kapasitor kecil ke GND
- Wiring pendek
- Ground baik

---

## 17. Error Umum PWM/LEDC

### LED Tidak Berubah Brightness

Cek:

- Pin GPIO salah
- LED active-low
- Duty tidak di-update dengan `ledc_update_duty()`
- Timer/channel salah
- Wiring LED salah

---

### Servo Bergetar

Kemungkinan:

- Supply servo lemah
- Ground tidak common
- Pulse range tidak cocok
- Frequency bukan 50 Hz
- Kabel terlalu panjang

Solusi:

- Gunakan supply 5V eksternal
- Common ground
- Coba pulse 1000–2000 us dulu
- Tambahkan kapasitor pada supply servo

---

### Buzzer Tidak Bunyi

Cek:

- Buzzer aktif atau pasif?
- Passive buzzer perlu PWM
- Active buzzer cukup ON/OFF
- Duty terlalu kecil
- Frequency di luar range buzzer

---

## 18. Best Practice Minggu 6

### ADC

- Jangan input 5V ke ADC.
- Gunakan voltage divider untuk tegangan tinggi.
- Gunakan calibration driver.
- Ambil beberapa sampel lalu rata-rata.
- Hindari input floating.
- Gunakan common ground.

### PWM

- Pilih frequency sesuai beban.
- Untuk LED, gunakan 1–5 kHz.
- Untuk servo, gunakan 50 Hz.
- Untuk buzzer, frequency menentukan nada.
- Jangan drive motor langsung dari GPIO.
- Gunakan MOSFET/driver untuk beban besar.

### Servo dan Motor

- Jangan supply servo/motor dari 3.3V ESP32.
- Gunakan power eksternal.
- Satukan ground.
- Gunakan driver.

### RMT

- Gunakan RMT untuk timing pulse presisi.
- Jangan bit-banging manual untuk protocol cepat.
- Pastikan level sinyal dan supply sesuai.

---

## 19. Latihan Minggu 6

### Latihan 1 — ADC Raw Reader

Buat program membaca ADC raw setiap 500 ms.

Target output:

```txt
ADC raw = 1234
ADC raw = 1250
ADC raw = 1242
```

---

### Latihan 2 — ADC Voltage Reader

Tambahkan calibration.

Target output:

```txt
ADC raw=2048, voltage=1650 mV
```

---

### Latihan 3 — Moving Average

Ambil 16 sampel ADC lalu rata-rata.

Target:

```txt
Nilai lebih stabil daripada pembacaan tunggal.
```

---

### Latihan 4 — Battery Voltage Reader

Gunakan voltage divider.

Target:

```txt
ADC voltage = 2100 mV
Battery voltage = 4200 mV
```

---

### Latihan 5 — LED Fade PWM

Buat LED brightness naik dari 0% ke 100%, lalu turun lagi.

---

### Latihan 6 — Servo Sweep

Buat servo bergerak:

```txt
0° → 90° → 180° → 90° → 0°
```

---

### Latihan 7 — Buzzer Melody

Buat buzzer memainkan nada:

```txt
C4 E4 G4 C5
```

---

### Latihan 8 — ADC Control PWM

Potensiometer mengontrol brightness LED.

Target:

```txt
Potensiometer diputar → LED makin terang/redup.
```

---

## 20. Tugas Akhir Minggu 6

Buat project:

```txt
week6_final_adc_pwm_monitor
```

Fitur wajib:

1. ADC membaca potensiometer atau voltage divider.
2. ADC memakai moving average minimal 16 sampel.
3. Jika calibration tersedia, tampilkan voltage mV.
4. Nilai ADC dikonversi menjadi persen 0–100%.
5. LEDC PWM mengatur brightness LED berdasarkan persen ADC.
6. Monitor task print setiap 1 detik:
   - raw ADC
   - voltage mV
   - percent
   - PWM duty
   - free heap
   - stack high water mark
7. Gunakan queue antara ADC task dan PWM task.
8. Gunakan mutex untuk shared status.
9. Tidak ada loop tanpa delay/blocking.
10. Struktur file dipisah:
    - `app_adc.c/h`
    - `app_pwm.c/h`
    - `app_status.c/h`
    - `app_tasks.c/h`

---

## 21. Checklist Lulus Minggu 6

Kamu dianggap lulus Minggu 6 kalau bisa:

```txt
[ ] Menjelaskan apa itu ADC
[ ] Menjelaskan raw ADC vs voltage mV
[ ] Menjelaskan attenuation ADC
[ ] Menggunakan ADC oneshot driver
[ ] Menggunakan ADC calibration driver
[ ] Membuat moving average filter
[ ] Menghitung voltage divider
[ ] Menjelaskan apa itu PWM
[ ] Menjelaskan frequency dan duty cycle
[ ] Menggunakan LEDC timer
[ ] Menggunakan LEDC channel
[ ] Mengubah duty PWM
[ ] Membuat LED fade
[ ] Mengontrol servo dengan PWM 50 Hz
[ ] Mengontrol buzzer dengan PWM frequency
[ ] Menjelaskan fungsi RMT
[ ] Menjelaskan kapan RMT lebih baik daripada GPIO manual
[ ] Membuat project ADC mengontrol PWM
```

---

## 22. Penerapan untuk Project Stunting IoT

Untuk proyek stunting berbasis ESP32:

### ADC

- Membaca baterai
- Membaca sensor analog tambahan
- Monitoring supply device

### PWM/LEDC

- Buzzer indikator
- LED status
- Backlight LCD jika dikontrol PWM

### RMT

- LED addressable untuk status device
- IR atau pulse protocol jika dibutuhkan

Contoh arsitektur:

```txt
Battery ADC Task
   ↓
Health Monitor
   ↓
Display/MQTT Health Report

Sensor/Input Task
   ↓
App Task
   ↓
Buzzer/LED PWM Indicator
```

---

## 23. Inti Pemahaman Minggu 6

Kalimat penting:

> ADC membaca dunia analog, PWM mengontrol output berbasis duty cycle, dan RMT menangani sinyal pulse yang butuh timing lebih presisi.

Pola firmware yang baik:

```txt
ADC Task
   ↓ queue
Control/PWM Task
   ↓ LEDC output

Status
   ↓ mutex
Monitor Task
```

Setelah Minggu 6 kuat, lanjut ke Minggu 7:

```txt
NVS, Partition Table, SPIFFS/LittleFS/FATFS, dan offline storage.
```
