# Minggu 12 — ESP-IDF Security: HTTPS/TLS, AES, HMAC, Secure Storage, Flash Encryption, Secure Boot

## Target Pembelajaran

Pada akhir Minggu 12, kamu mampu:

- Menjelaskan confidentiality, integrity, authenticity, dan anti-replay.
- Menggunakan HTTPS/TLS client di ESP-IDF.
- Memahami sertifikat server dan certificate bundle.
- Menghitung HMAC-SHA256 untuk payload.
- Membuat backend yang memverifikasi HMAC.
- Memahami AES-CBC + IV + padding + HMAC.
- Memahami secure storage, NVS encryption, flash encryption, dan secure boot.
- Menyusun security plan untuk IoT production.

---

## 1. Konsep Security IoT

### Confidentiality

Data tidak bisa dibaca pihak tidak berhak.

Solusi:

- HTTPS/TLS.
- AES encryption.
- Flash encryption.
- NVS encryption.

### Integrity

Data tidak berubah di tengah jalan.

Solusi:

- HMAC-SHA256.
- TLS integrity.
- Digital signature.

### Authenticity

Server yakin data berasal dari device sah.

Solusi:

- Device token.
- HMAC shared secret.
- Client certificate.
- Digital signature.

### Anti-Replay

Mencegah request valid lama dikirim ulang.

Solusi:

- Timestamp.
- Nonce.
- Counter monotonik.

---

## 2. Threat Model Sederhana

Ancaman jaringan:

```text
Orang membaca traffic WiFi
Orang memodifikasi request HTTP
Orang membuat server palsu
Orang replay payload lama
```

Solusi:

```text
HTTPS/TLS
Certificate verification
HMAC
Timestamp/counter
```

Ancaman device:

```text
Flash dibaca langsung
Firmware diekstrak
Secret dicari dari binary
Device clone
```

Solusi:

```text
Flash encryption
NVS encryption
Secure boot
Per-device secret
Secure element jika perlu
```

---

## 3. HTTPS/TLS Client

Gunakan `https://`, bukan `http://`, untuk production.

Contoh HTTPS POST JSON:

```c
#include <stdio.h>
#include <string.h>

#include "esp_log.h"
#include "esp_http_client.h"
#include "esp_crt_bundle.h"

static const char *TAG = "HTTPS_CLIENT";

esp_err_t https_post_json(const char *url, const char *json_payload)
{
    esp_http_client_config_t config = {
        .url = url,
        .method = HTTP_METHOD_POST,
        .timeout_ms = 10000,
        .crt_bundle_attach = esp_crt_bundle_attach,
    };

    esp_http_client_handle_t client = esp_http_client_init(&config);

    if (client == NULL) {
        ESP_LOGE(TAG, "Failed to init HTTPS client");
        return ESP_FAIL;
    }

    esp_http_client_set_header(client, "Content-Type", "application/json");
    esp_http_client_set_post_field(client, json_payload, strlen(json_payload));

    esp_err_t ret = esp_http_client_perform(client);

    if (ret == ESP_OK) {
        int status = esp_http_client_get_status_code(client);
        ESP_LOGI(TAG, "HTTPS POST status=%d", status);

        if (status < 200 || status >= 300) {
            ret = ESP_FAIL;
        }
    } else {
        ESP_LOGE(TAG, "HTTPS POST failed: %s", esp_err_to_name(ret));
    }

    esp_http_client_cleanup(client);

    return ret;
}
```

`esp_crt_bundle_attach` membantu ESP32 memverifikasi sertifikat server publik.

Jangan disable certificate verification di production.

---

## 4. Sertifikat TLS

Ada dua pendekatan:

### Certificate Bundle

Cocok untuk server dengan CA publik seperti Let's Encrypt, DigiCert, Cloudflare, dan lain-lain.

```c
.crt_bundle_attach = esp_crt_bundle_attach
```

### Embed Root CA Manual

Cocok jika server memakai CA sendiri.

```c
extern const char server_cert_pem_start[] asm("_binary_server_cert_pem_start");

esp_http_client_config_t config = {
    .url = "https://example.com/api/data",
    .cert_pem = server_cert_pem_start,
};
```

`CMakeLists.txt`:

```cmake
idf_component_register(
    SRCS "main.c"
    INCLUDE_DIRS "."
    EMBED_TXTFILES "server_cert.pem"
)
```

---

## 5. HMAC-SHA256

HMAC membuat signature dari payload dan secret.

```text
signature = HMAC_SHA256(secret, raw_json_body)
```

Header request:

```text
X-Device-Id: STUNT-001
X-Signature: hex_hmac_sha256
```

Penting:

```text
HMAC harus dihitung dari raw body string yang benar-benar dikirim.
Jangan hitung dari JSON yang sudah diparse dan stringify ulang.
```

Kode HMAC:

```c
#include <stdio.h>
#include <string.h>

#include "esp_log.h"
#include "mbedtls/md.h"

static void bytes_to_hex(const uint8_t *input, size_t input_len,
                         char *output, size_t output_len)
{
    static const char hex_chars[] = "0123456789abcdef";

    if (output_len < (input_len * 2 + 1)) {
        return;
    }

    for (size_t i = 0; i < input_len; i++) {
        output[i * 2] = hex_chars[(input[i] >> 4) & 0x0F];
        output[i * 2 + 1] = hex_chars[input[i] & 0x0F];
    }

    output[input_len * 2] = '\0';
}

esp_err_t hmac_sha256_hex(const uint8_t *key,
                          size_t key_len,
                          const uint8_t *message,
                          size_t message_len,
                          char *hex_output,
                          size_t hex_output_len)
{
    uint8_t hmac_result[32];

    const mbedtls_md_info_t *md_info =
        mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);

    if (md_info == NULL) {
        return ESP_FAIL;
    }

    int ret = mbedtls_md_hmac(
        md_info,
        key,
        key_len,
        message,
        message_len,
        hmac_result
    );

    if (ret != 0) {
        return ESP_FAIL;
    }

    bytes_to_hex(hmac_result, sizeof(hmac_result),
                 hex_output, hex_output_len);

    return ESP_OK;
}
```

---

## 6. HTTPS + HMAC

```c
esp_err_t https_post_json_signed(const char *url,
                                 const char *device_id,
                                 const char *secret,
                                 const char *json_payload)
{
    char signature_hex[65];

    esp_err_t ret = hmac_sha256_hex(
        (const uint8_t *)secret,
        strlen(secret),
        (const uint8_t *)json_payload,
        strlen(json_payload),
        signature_hex,
        sizeof(signature_hex)
    );

    if (ret != ESP_OK) {
        return ret;
    }

    esp_http_client_config_t config = {
        .url = url,
        .method = HTTP_METHOD_POST,
        .timeout_ms = 10000,
        .crt_bundle_attach = esp_crt_bundle_attach,
    };

    esp_http_client_handle_t client = esp_http_client_init(&config);

    if (client == NULL) {
        return ESP_FAIL;
    }

    esp_http_client_set_header(client, "Content-Type", "application/json");
    esp_http_client_set_header(client, "X-Device-Id", device_id);
    esp_http_client_set_header(client, "X-Signature", signature_hex);
    esp_http_client_set_post_field(client, json_payload, strlen(json_payload));

    ret = esp_http_client_perform(client);

    if (ret == ESP_OK) {
        int status = esp_http_client_get_status_code(client);
        if (status < 200 || status >= 300) {
            ret = ESP_FAIL;
        }
    }

    esp_http_client_cleanup(client);
    return ret;
}
```

---

## 7. Backend Express Verifikasi HMAC

```js
import express from "express";
import crypto from "crypto";

const app = express();

const deviceSecrets = {
  "STUNT-001": "super_secret_key",
};

app.use(express.raw({ type: "application/json", limit: "64kb" }));

function timingSafeEqualHex(a, b) {
  const bufferA = Buffer.from(a, "hex");
  const bufferB = Buffer.from(b, "hex");

  if (bufferA.length !== bufferB.length) {
    return false;
  }

  return crypto.timingSafeEqual(bufferA, bufferB);
}

app.post("/api/diagnosis", (req, res) => {
  const deviceId = req.header("X-Device-Id");
  const signature = req.header("X-Signature");

  if (!deviceId || !signature) {
    return res.status(401).json({ ok: false, error: "Missing auth headers" });
  }

  const secret = deviceSecrets[deviceId];

  if (!secret) {
    return res.status(401).json({ ok: false, error: "Unknown device" });
  }

  const expectedSignature = crypto
    .createHmac("sha256", secret)
    .update(req.body)
    .digest("hex");

  if (!timingSafeEqualHex(signature, expectedSignature)) {
    return res.status(401).json({ ok: false, error: "Invalid signature" });
  }

  const payload = JSON.parse(req.body.toString("utf8"));

  console.log("Valid payload:", payload);

  return res.json({ ok: true });
});

app.listen(3000, "0.0.0.0", () => {
  console.log("Server running on port 3000");
});
```

---

## 8. Anti-Replay

Tambahkan counter atau timestamp ke payload:

```json
{
  "device_id": "STUNT-001",
  "counter": 12345,
  "height_cm": 82.5
}
```

Backend menolak counter yang lebih kecil/sama dari counter terakhir.

Untuk device offline-first, counter monotonik sering lebih mudah daripada timestamp.

---

## 9. AES Encryption

HTTPS sudah mengenkripsi transport. AES tambahan berguna jika:

- Payload disimpan offline dan ingin encrypted.
- Ada sistem perantara.
- Field tertentu harus tetap encrypted bahkan di layer lain.

Jangan pakai AES-ECB.

Skema yang lebih aman untuk latihan:

```text
AES-256-CBC + random IV + PKCS#7 padding + HMAC
```

Alur:

```text
plaintext JSON
↓
generate IV random 16 byte
↓
AES-256-CBC encrypt
↓
HMAC over IV + ciphertext
↓
kirim iv + ciphertext + hmac
```

IV harus random:

```c
esp_fill_random(iv, 16);
```

---

## 10. Secure Storage

Masalah jika secret hardcoded:

```c
const char *secret = "super_secret_key";
```

Secret bisa ditemukan dari binary firmware.

Pilihan lebih baik:

- Secret unik per device.
- Provision secret saat manufacturing.
- Simpan di NVS encrypted.
- Aktifkan flash encryption.
- Gunakan secure element untuk keamanan lebih tinggi.

### NVS Encryption

Melindungi data key-value di NVS.

### Flash Encryption

Mengenkripsi isi off-chip flash agar firmware/data sulit dibaca secara fisik.

### Secure Boot

Memastikan hanya firmware yang sah/ditandatangani yang bisa boot.

---

## 11. Hati-hati eFuse

Fitur seperti:

- Flash encryption release mode.
- Secure boot.
- Disable JTAG.
- Disable UART bootloader access.
- Burn key ke eFuse.
- Anti-rollback.

bisa irreversible.

Aturan:

```text
Jangan eksperimen di board utama.
Gunakan development mode dulu.
Baca dokumentasi chip target.
Simpan key dengan aman.
```

---

## 12. Mini Project Minggu 12

Project:

```text
week12_final_secure_iot_payload
```

Fitur wajib:

1. WiFi Station connect.
2. HTTPS POST ke backend.
3. JSON dibuat dengan cJSON.
4. Payload berisi:
   - device_id.
   - counter.
   - timestamp/time_us.
   - sensor_value.
5. HMAC-SHA256 dihitung dari raw JSON body.
6. Header:
   - `X-Device-Id`
   - `X-Signature`
7. Backend Express memverifikasi HMAC.
8. Jika HMAC salah, backend menolak.
9. Jika HTTP gagal, payload masuk offline queue.
10. Jangan log secret.
11. Tulis threat model singkat.

---

## 13. Latihan

1. HTTPS GET.
2. HTTPS POST JSON.
3. Hitung HMAC-SHA256 string `hello` dengan secret `secret`.
4. Backend verify HMAC.
5. Gabungkan HTTPS + HMAC.
6. AES-CBC encrypt JSON kecil.
7. Anti-replay counter.
8. Security review project Minggu 8/9.

---

## 14. Checklist Lulus

- [ ] Menjelaskan confidentiality.
- [ ] Menjelaskan integrity.
- [ ] Menjelaskan authenticity.
- [ ] Menggunakan HTTPS.
- [ ] Menggunakan certificate bundle.
- [ ] Menghitung HMAC-SHA256.
- [ ] Mengirim HMAC di header.
- [ ] Backend verifikasi raw body HMAC.
- [ ] Menjelaskan anti-replay.
- [ ] Menjelaskan AES-CBC + IV + padding.
- [ ] Menjelaskan kenapa AES-ECB buruk.
- [ ] Menjelaskan NVS encryption.
- [ ] Menjelaskan flash encryption.
- [ ] Menjelaskan secure boot.
- [ ] Menjelaskan risiko eFuse irreversible.
