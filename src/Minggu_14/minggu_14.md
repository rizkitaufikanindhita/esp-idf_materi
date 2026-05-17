# Minggu 14 — ESP-IDF Debugging, Diagnostics, Profiling, dan Reliability

## Target Pembelajaran

Pada akhir Minggu 14, kamu mampu:

- Menggunakan logging ESP-IDF secara rapi.
- Membaca panic log dan Guru Meditation.
- Membedakan `LoadProhibited`, `StoreProhibited`, stack overflow, heap corruption, watchdog.
- Membaca dan decode backtrace.
- Memantau stack setiap task.
- Memantau heap, memory leak, dan heap fragmentation.
- Mendiagnosis watchdog dan deadlock.
- Memahami core dump, JTAG, dan GDB.
- Membuat firmware health monitor.

---

## 1. Mindset Debugging Firmware

Jangan hanya “coba-coba ubah kode”. Urutan berpikir:

```text
Apa gejalanya?
Kapan muncul?
Setelah event apa muncul?
Task mana yang aktif?
Apakah crash, hang, reboot, atau data salah?
Stack cukup?
Heap turun terus?
Ada race condition?
Peripheral dipakai bersamaan?
Watchdog trigger?
```

Jenis masalah:

| Gejala | Kemungkinan |
|---|---|
| Reboot tiba-tiba | panic, watchdog, brownout, stack overflow |
| Hang | deadlock, infinite loop, task priority salah |
| Data kacau | race condition, buffer overflow |
| WiFi/MQTT putus | network, heap kurang, event salah |
| Crash sensor | pointer NULL, SPI/I2C konflik, stack kurang |

---

## 2. Logging ESP-IDF

Gunakan:

```c
#include "esp_log.h"
```

Macro:

```c
ESP_LOGE(TAG, "Error");
ESP_LOGW(TAG, "Warning");
ESP_LOGI(TAG, "Info");
ESP_LOGD(TAG, "Debug");
ESP_LOGV(TAG, "Verbose");
```

Gunakan TAG per module:

```c
static const char *TAG = "APP_WIFI";
ESP_LOGI(TAG, "WiFi started");
```

Contoh TAG:

```text
MAIN
APP_WIFI
APP_MQTT
APP_STORAGE
APP_SENSOR
APP_OTA
APP_SECURITY
APP_MONITOR
```

Set log level:

```c
esp_log_level_set("*", ESP_LOG_INFO);
esp_log_level_set("APP_WIFI", ESP_LOG_DEBUG);
esp_log_level_set("APP_OTA", ESP_LOG_DEBUG);
```

Print error:

```c
ESP_LOGE(TAG, "Failed: %s", esp_err_to_name(ret));
```

---

## 3. Panic dan Guru Meditation

Panic terjadi saat fatal error.

Penyebab umum:

```text
Illegal instruction
LoadProhibited
StoreProhibited
Stack overflow
Stack smashing
Heap corruption
Interrupt watchdog
Task watchdog
Assert failed
Abort called
```

### LoadProhibited

CPU mencoba membaca alamat invalid.

Contoh:

```c
int *ptr = NULL;
int value = *ptr;
```

### StoreProhibited

CPU mencoba menulis alamat invalid.

Contoh:

```c
int *ptr = NULL;
*ptr = 123;
```

### Backtrace

Log panic biasanya menampilkan:

```text
Backtrace: 0x400d1234:0x3ffb1230 0x400d5678:0x3ffb1250
```

`idf.py monitor` sering decode otomatis jika file ELF tersedia.

File penting untuk debugging:

```text
build/project_name.elf
```

Manual decode:

```bash
xtensa-esp32-elf-addr2line -pfiaC -e build/project.elf 0x400d1234
```

Untuk RISC-V seperti ESP32-C3:

```bash
riscv32-esp-elf-addr2line -pfiaC -e build/project.elf 0x42001234
```

---

## 4. Stack Debugging

Setiap task FreeRTOS punya stack sendiri.

Gejala stack terlalu kecil:

```text
Stack canary watchpoint triggered
Stack overflow
Crash saat JSON/HTTP/TLS
Reboot random
```

Cek stack:

```c
UBaseType_t stack_left = uxTaskGetStackHighWaterMark(NULL);
ESP_LOGI("STACK", "stack left=%u", stack_left);
```

Untuk task lain, simpan handle:

```c
static TaskHandle_t sensor_task_handle = NULL;

xTaskCreate(sensor_task, "Sensor Task", 4096, NULL, 2, &sensor_task_handle);

UBaseType_t left = uxTaskGetStackHighWaterMark(sensor_task_handle);
```

Rekomendasi awal:

| Task | Stack Awal |
|---|---|
| LED/Button | 2048 |
| Sensor I2C/SPI | 3072–4096 |
| Storage file | 4096 |
| JSON/cJSON | 4096–6144 |
| HTTP client | 6144–8192 |
| HTTPS/TLS | 8192+ |
| MQTT | 4096–6144 |
| OTA task | 8192+ |

Selalu validasi dengan high water mark.

---

## 5. Heap Debugging

Heap dipakai untuk:

```text
malloc/calloc/free
cJSON_PrintUnformatted
HTTP/TLS buffer
MQTT buffer
BLE allocation
file system buffer
task stack allocation
```

Monitoring heap:

```c
#include "esp_heap_caps.h"

static void heap_monitor_task(void *pvParameters)
{
    while (1) {
        uint32_t free_heap = heap_caps_get_free_size(MALLOC_CAP_DEFAULT);
        uint32_t min_free_heap = heap_caps_get_minimum_free_size(MALLOC_CAP_DEFAULT);
        uint32_t largest_block = heap_caps_get_largest_free_block(MALLOC_CAP_DEFAULT);

        ESP_LOGI("HEAP_MON",
                 "free=%lu, min_free=%lu, largest_block=%lu",
                 free_heap,
                 min_free_heap,
                 largest_block);

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

Makna:

| Metric | Arti |
|---|---|
| free_heap | heap bebas sekarang |
| min_free_heap | heap bebas terendah sejak boot |
| largest_block | blok alokasi terbesar yang masih tersedia |

Kalau free heap besar tapi largest block kecil, bisa terjadi fragmentasi.

---

## 6. Memory Leak

Contoh leak:

```c
char *buffer = malloc(256);
snprintf(buffer, 256, "hello");
/* lupa free(buffer) */
```

Perbaikan:

```c
free(buffer);
```

cJSON leak umum:

```c
char *json = cJSON_PrintUnformatted(root);
/* pakai json */
free(json);
cJSON_Delete(root);
```

Wajib:

```text
cJSON_Delete(root)
free(json hasil cJSON_Print*)
esp_http_client_cleanup(client)
fclose(file)
```

---

## 7. Heap Integrity Check

```c
bool ok = heap_caps_check_integrity_all(true);

if (!ok) {
    ESP_LOGE("HEAP", "Heap corruption detected");
}
```

Gunakan saat debugging. Jangan terlalu sering di production karena berat.

Menuconfig:

```text
Component config -> Heap memory debugging
```

Mode:

```text
Basic
Light Impact
Comprehensive
```

---

## 8. Watchdog Debugging

Penyebab task watchdog:

```text
while(1) tanpa delay
Task priority tinggi tidak pernah blocking
Parsing file besar tanpa yield
Loop retry WiFi tanpa delay
Mutex deadlock
HTTP/TLS blocking terlalu lama
Task menunggu hardware tanpa timeout
```

Pola baik:

```c
vTaskDelay(pdMS_TO_TICKS(10));
```

atau:

```c
xQueueReceive(queue, &msg, portMAX_DELAY);
```

atau timeout:

```c
xSemaphoreTake(mutex, pdMS_TO_TICKS(100));
```

Untuk loop panjang:

```c
for (int i = 0; i < huge_count; i++) {
    do_work(i);

    if (i % 100 == 0) {
        vTaskDelay(pdMS_TO_TICKS(1));
    }
}
```

---

## 9. Deadlock Mutex

Contoh deadlock:

```text
Task A pegang mutex I2C, menunggu mutex storage
Task B pegang mutex storage, menunggu mutex I2C
```

Solusi:

- Tentukan urutan lock konsisten.
- Jangan pegang mutex saat operasi lama.
- Pakai timeout.
- Log sebelum/sesudah take/give saat debugging.

```c
if (xSemaphoreTake(i2c_mutex, pdMS_TO_TICKS(500)) == pdTRUE) {
    /* transaksi I2C */
    xSemaphoreGive(i2c_mutex);
} else {
    ESP_LOGW(TAG, "Timeout taking i2c_mutex");
}
```

---

## 10. Core Dump

Core dump adalah snapshot saat crash:

```text
register
stack task
backtrace
task info
memory tertentu
```

Aktifkan:

```text
idf.py menuconfig
Component config -> Core dump
```

Destinasi:

```text
UART
Flash
```

Jika ke flash, tambahkan partition:

```csv
coredump, data, coredump, , 64K,
```

Analisis:

```bash
idf.py coredump-info
idf.py coredump-debug
```

Simpan file ELF untuk setiap release firmware.

---

## 11. JTAG dan GDB

JTAG/GDB memberi kemampuan:

- Breakpoint.
- Step into/over.
- Inspect variable.
- Inspect stack.
- Pause CPU.
- Backtrace live.

Alur:

```text
ESP32 target
↓ JTAG
OpenOCD server
↓
GDB client
```

Perintah:

```bash
idf.py openocd
idf.py gdb
```

Pakai JTAG jika:

- Crash sulit dipahami dari log.
- Firmware hang tanpa log.
- Perlu inspect variable runtime.
- Race condition sulit.

---

## 12. Reset Reason

```c
#include "esp_system.h"

static void print_reset_reason(void)
{
    esp_reset_reason_t reason = esp_reset_reason();

    switch (reason) {
        case ESP_RST_POWERON:
            ESP_LOGI("RESET", "Power-on reset");
            break;
        case ESP_RST_SW:
            ESP_LOGI("RESET", "Software reset");
            break;
        case ESP_RST_PANIC:
            ESP_LOGI("RESET", "Panic reset");
            break;
        case ESP_RST_TASK_WDT:
            ESP_LOGI("RESET", "Task watchdog reset");
            break;
        case ESP_RST_BROWNOUT:
            ESP_LOGI("RESET", "Brownout reset");
            break;
        case ESP_RST_DEEPSLEEP:
            ESP_LOGI("RESET", "Deep sleep wakeup");
            break;
        default:
            ESP_LOGI("RESET", "Other reset reason: %d", reason);
            break;
    }
}
```

---

## 13. Firmware Health Monitor

Health monitor task:

```c
#include "esp_log.h"
#include "esp_timer.h"
#include "esp_heap_caps.h"

static void health_monitor_task(void *pvParameters)
{
    while (1) {
        uint32_t free_heap = heap_caps_get_free_size(MALLOC_CAP_DEFAULT);
        uint32_t min_free_heap = heap_caps_get_minimum_free_size(MALLOC_CAP_DEFAULT);
        uint32_t largest_block = heap_caps_get_largest_free_block(MALLOC_CAP_DEFAULT);
        int64_t uptime_ms = esp_timer_get_time() / 1000;

        ESP_LOGI("HEALTH",
                 "uptime=%lld ms, free=%lu, min_free=%lu, largest=%lu",
                 uptime_ms,
                 free_heap,
                 min_free_heap,
                 largest_block);

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

Health report production:

```json
{
  "device_id": "STUNT-001",
  "firmware": "1.2.0",
  "uptime_ms": 123456,
  "free_heap": 145000,
  "min_free_heap": 110000,
  "largest_block": 80000,
  "reset_reason": "TASK_WDT",
  "boot_count": 34,
  "panic_count": 1,
  "wdt_count": 2,
  "offline_queue_count": 5,
  "wifi": "connected",
  "mqtt": "connected"
}
```

---

## 14. Debugging Checklist

### Reboot tiba-tiba

Cek:

1. Reset reason.
2. Panic log.
3. Backtrace.
4. Stack high water mark.
5. Heap min free.
6. Brownout.
7. Watchdog.
8. Power supply.

### Device hang

Cek:

1. Task terakhir yang log.
2. Deadlock mutex.
3. Queue receive tanpa producer.
4. Semaphore tidak pernah give.
5. Infinite loop tanpa delay.
6. Priority task terlalu tinggi.

### Heap turun terus

Cek:

1. malloc tanpa free.
2. cJSON_Print tanpa free.
3. HTTP client tidak cleanup.
4. File tidak fclose.
5. Task dibuat berulang tanpa delete.

### Stack overflow

Cek:

1. Local buffer besar.
2. Recursion.
3. Stack task terlalu kecil.
4. HTTP/TLS/cJSON butuh stack besar.

### Brownout

Cek:

1. USB lemah.
2. Kabel USB jelek.
3. Regulator tidak cukup.
4. WiFi transmit menarik arus tinggi.
5. Servo/motor disupply dari 3V3.

---

## 15. Mini Project Minggu 14

Project:

```text
week14_final_reliability_diagnostics
```

Fitur wajib:

1. Print reset reason saat boot.
2. Simpan boot_count ke NVS.
3. Simpan panic_count / wdt_count / brownout_count.
4. Health monitor setiap 5 detik:
   - uptime.
   - free heap.
   - min free heap.
   - largest free block.
   - stack left task penting.
5. Dummy sensor task.
6. Dummy network task.
7. Dummy storage task.
8. Semua task punya handle.
9. Debug macro:
   - `ENABLE_HEAP_TEST`
   - `ENABLE_STACK_TEST`
   - `ENABLE_WATCHDOG_TEST`
10. Health report bisa dipublish via MQTT/HTTP.

---

## 16. Latihan

1. Log level per module.
2. Crash null pointer dan baca backtrace.
3. Stack monitor 3 task.
4. Heap leak test.
5. cJSON leak test.
6. Watchdog test.
7. Reset reason + reboot stats NVS.
8. Core dump.
9. JTAG basic jika board mendukung.

---

## 17. Checklist Lulus

- [ ] Menggunakan ESP_LOGE/W/I/D/V.
- [ ] Membuat TAG per module.
- [ ] Mengatur log level.
- [ ] Menjelaskan Guru Meditation.
- [ ] Membedakan LoadProhibited dan StoreProhibited.
- [ ] Membaca backtrace.
- [ ] Menjelaskan pentingnya file `.elf`.
- [ ] Menggunakan `uxTaskGetStackHighWaterMark()`.
- [ ] Menggunakan heap monitor.
- [ ] Menjelaskan memory leak.
- [ ] Menjelaskan heap corruption.
- [ ] Menjelaskan watchdog.
- [ ] Menjelaskan deadlock mutex.
- [ ] Menjelaskan core dump.
- [ ] Menjelaskan JTAG/GDB.
- [ ] Membuat firmware health monitor.
