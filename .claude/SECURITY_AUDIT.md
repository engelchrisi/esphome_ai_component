# Security Audit Report

**Date:** 2026-06-06  
**Audited by:** Claude Code (claude-sonnet-4-6)  
**Scope:** Full repository scan — Python, C++, YAML, configuration files

---

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 1 |
| HIGH     | 3 |
| MEDIUM   | 6 |
| LOW      | 7 |
| **Total**| **17** |

---

## CRITICAL

### 1. Hardcoded WiFi credentials in fuzz test script
- **File:** `fuzz_parallel.py` lines 220, 223
- **Code:**
  ```yaml
  password: "password12345678"
  ```
- **Impact:** Hardcoded credentials injected into generated device configs; accidental inclusion in production builds exposes the device to unauthorized WiFi connections.
- **Fix:** Use environment variables or a `.gitignore`d secrets file; mark clearly as test-only.

---

## HIGH

### 1. Cleartext HTTP uploads — no HTTPS enforcement
- **File:** `components/data_collector/data_collector.cpp` lines 210–273
- **Code:**
  ```cpp
  config.url = this->upload_url_.c_str();
  config.method = HTTP_METHOD_POST;
  esp_http_client_handle_t client = esp_http_client_init(&config);
  ```
- **Impact:** Meter readings and camera images transmitted in plaintext; vulnerable to MITM interception.
- **Fix:** Validate URL scheme at config time; reject non-HTTPS URLs in production builds; document the requirement.

### 2. No TLS certificate validation
- **File:** `components/data_collector/data_collector.cpp` lines 212–223
- **Code:** `esp_http_client_config_t` has no `cert_pem`, `client_cert_pem`, or CA bundle configured.
- **Impact:** Even if HTTPS is used, the server certificate is not verified — MITM attacks still succeed.
- **Fix:** Add ESP-IDF built-in CA bundle or configure `cert_pem`; consider certificate pinning for fixed deployments.

### 3. Weak default API key in Docker Compose
- **File:** `components/data_collector/server/docker-compose.yml` line 15
- **Code:**
  ```yaml
  environment:
    - API_KEY=change-me-to-a-secure-key
  ```
- **Impact:** Deployments that don't change the default key are protected only by a publicly visible, predictable credential.
- **Fix:** Remove the default value; require the env var to be set externally (fail startup if absent); use Docker secrets or a secrets manager.

---

## MEDIUM

### 1. Timing-attack-vulnerable API key comparison
- **File:** `components/data_collector/server/app.py` lines 50–58
- **Code:**
  ```python
  authorized = (client_key and client_key == API_KEY) or \
               (auth_header and auth_header == f"Bearer {API_KEY}")
  ```
- **Impact:** String equality `==` leaks timing information that can be used to guess the key byte-by-byte; no rate limiting compounds this.
- **Fix:** Replace `==` with `secrets.compare_digest()`; add rate limiting (e.g., Flask-Limiter).

### 2. Unauthenticated file download route
- **File:** `components/data_collector/server/app.py` line 37
- **Code:**
  ```python
  @app.route('/uploads/<path:filename>')
  def uploaded_file(filename):
      return send_from_directory(UPLOAD_FOLDER, filename)
  ```
- **Impact:** Anyone who knows (or guesses) a filename can download uploaded meter images without an API key.
- **Fix:** Add the same API key authentication check applied to the upload endpoint.

### 3. Unsanitized EXIF metadata injection
- **File:** `components/data_collector/server/app.py` lines 88–103
- **Code:**
  ```python
  metadata_json = request.headers.get('X-Meter-Json')
  if metadata_json:
      user_comment = piexif.helper.UserComment.dump(metadata_json)
      exif_dict["Exif"][piexif.ExifIFD.UserComment] = user_comment
  ```
- **Impact:** Arbitrary, unsized content from an untrusted header is injected into JPEG EXIF data; potential for file corruption; if EXIF is later parsed by another service, may trigger parsing bugs.
- **Fix:** Validate `metadata_json` is valid JSON; enforce a size cap (e.g., 1 KB); validate against expected schema.

### 4. Credentials stored in plaintext RAM
- **File:** `components/data_collector/data_collector.cpp` lines 26–39
- **Code:**
  ```cpp
  std::string upload_url_;
  std::string username_;
  std::string password_;
  std::string api_key_;
  ```
- **Impact:** If device memory is read (JTAG, core dump, heap inspection), credentials are exposed in plaintext.
- **Fix:** Zero-out credentials after use where possible; document the limitation; consider using NVS encryption on ESP32.

### 5. No input validation on configuration values
- **File:** `components/data_collector/__init__.py` lines 50–57
- **Code:**
  ```python
  cg.add(var.set_upload_url(config[CONF_UPLOAD_URL]))
  cg.add(var.set_auth(config[CONF_USERNAME], config.get(CONF_PASSWORD, "")))
  cg.add(var.set_api_key(config[CONF_API_KEY]))
  ```
- **Impact:** Malformed URLs and unchecked credential lengths accepted without error; may cause runtime failures or unexpected behavior on-device.
- **Fix:** Validate URL scheme (must be `http://` or `https://`); enforce max length on all credential fields; add ESPHome `cv.url` validator.

### 6. Unsafe pointer cast in drawing utilities — potential buffer overflow
- **File:** `components/esp32_camera_utils/drawing_utils.cpp` lines 15–40
- **Code:**
  ```cpp
  *(uint16_t*)(&buffer[index]) = color;  // unaligned cast
  ```
- **Impact:** Unaligned memory access is undefined behavior on some architectures; no buffer-size parameter means bounds checking relies entirely on caller correctness.
- **Fix:** Use `memcpy` for the 2-byte write to avoid alignment issues; add a `buffer_size` parameter and validate that `index + channels <= buffer_size`.

---

## LOW

### 1. Logging sensitive meter values
- **File:** `components/data_collector/data_collector.cpp` line 264
- **Code:**
  ```cpp
  ESP_LOGD(TAG, "Full upload metrics: Value=%s, Conf=%.4f, Size=%zu", raw_value.c_str(), confidence, len);
  ```
- **Fix:** Remove raw meter values from debug logs; log only non-sensitive diagnostics (response code, upload duration).

### 2. Unrestricted upload file size (DoS)
- **File:** `components/data_collector/server/app.py` lines 82–83
- **Code:**
  ```python
  with open(filepath, "wb") as f:
      f.write(request.data)
  ```
- **Fix:** Check `Content-Length` header; set `MAX_CONTENT_LENGTH` on the Flask app; implement per-device quotas.

### 3. EXIF metadata header size not capped
- **File:** `components/data_collector/server/app.py` line 88
- **Fix:** Reject requests where `X-Meter-Json` exceeds a reasonable limit (e.g., 1 KB) before processing.

### 4. Race condition on task teardown — potential use-after-free
- **File:** `components/data_collector/data_collector.cpp` lines 180–205
- **Code:**
  ```cpp
  this->task_running_ = false;
  vTaskDelay(1100 / portTICK_PERIOD_MS);  // hope task exits in time
  ```
- **Fix:** Use a FreeRTOS `EventGroupHandle_t` or `SemaphoreHandle_t` to signal and confirm task exit before freeing resources.

### 5. Predictable filenames using non-cryptographic random
- **File:** `components/data_collector/data_extractor/1_extractor.py` line 81
- **Code:**
  ```python
  random_str = ''.join(random.choices(string.ascii_lowercase + string.digits, k=3))
  ```
- **Fix:** Replace with `secrets.token_urlsafe(4)` for unpredictable filenames.

### 6. SSRF risk in data extractor scripts
- **File:** `components/data_collector/data_extractor/3_correct_errors.py`, `5_low_confidence.py`
- **Code:**
  ```python
  if image_source_str.startswith(('http://', 'https://')):
      response = requests.get(image_source_str, timeout=10)
  ```
- **Fix:** Block requests to private IP ranges (`127.x`, `10.x`, `172.16-31.x`, `192.168.x`); consider an allowlist of valid hostnames.

### 7. HTTP-only examples in documentation
- **Files:** `README.md` line 166, `components/data_collector/README.md` lines 25, 125
- **Fix:** Update examples to use HTTPS; add a security note explaining why plaintext HTTP should not be used in production.

---

## Positive Findings

- WiFi credentials correctly use ESPHome's `!secret` mechanism
- Flask uses `secure_filename()` — basic path traversal protection in place
- HTML output appears escaped — no obvious XSS surface
- `heap_caps_malloc` used appropriately for large image buffers
- Bounds checking present in drawing functions (partial mitigation)

---

## Prioritised Remediation Roadmap

**P0 — Immediate**
1. Remove hardcoded WiFi credentials from `fuzz_parallel.py`
2. Remove default `API_KEY` value from `docker-compose.yml`
3. Add auth check to `/uploads/` download route

**P1 — Short term**
4. Replace API key `==` comparison with `secrets.compare_digest()`
5. Enforce HTTPS and add TLS certificate validation in C++ HTTP client
6. Validate and size-cap `X-Meter-Json` EXIF injection
7. Set `MAX_CONTENT_LENGTH` on Flask app

**P2 — Medium term**
8. Add URL scheme validation in ESPHome component config
9. Fix unaligned pointer cast in `drawing_utils.cpp`
10. Replace `vTaskDelay` teardown with FreeRTOS synchronization primitive

**P3 — Long term**
11. Strip sensitive values from debug logs
12. Replace `random.choices` with `secrets` in extractor scripts
13. Add SSRF hostname validation to extractor scripts
14. Update documentation examples to HTTPS
