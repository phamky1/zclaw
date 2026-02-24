# Tổng quan zclaw

## 1. Chức năng chính

**zclaw** là firmware trợ lý AI cá nhân tối giản chạy trực tiếp trên vi điều khiển ESP32, viết bằng C. Toàn bộ firmware (bao gồm runtime, Wi-Fi, TLS, và cert bundle) bị giới hạn chặt ở mức **≤ 888 KiB**.

Các tính năng chính:

- **Giao tiếp ngôn ngữ tự nhiên** — kết nối với các LLM backend: Anthropic Claude, OpenAI, OpenRouter, hoặc Ollama (local).
- **Điều khiển GPIO** — đọc/ghi chân GPIO với giới hạn an toàn; hỗ trợ đọc toàn bộ (`gpio_read_all`).
- **Bộ nhớ bền vững** — lưu trữ key-value trong flash NVS, tồn tại qua các lần khởi động lại.
- **Lên lịch tác vụ** — hỗ trợ `periodic` (chu kỳ N phút), `daily` (theo giờ cụ thể, nhận biết múi giờ), `once` (một lần), và `condition` (có điều kiện). Tối đa 16 tác vụ.
- **Tạo công cụ tùy chỉnh** — người dùng tạo công cụ mới bằng ngôn ngữ tự nhiên thông qua `create_tool`; tối đa 8 công cụ động.
- **Lịch sử hội thoại** — bộ đệm cuộn 12 lượt người dùng/trợ lý.
- **Persona** — 4 kiểu trả lời: `neutral`, `friendly`, `technical`, `witty`.
- **Giới hạn tốc độ** — mặc định 100 yêu cầu/giờ và 1.000 yêu cầu/ngày.
- **OTA** — cập nhật firmware từ xa với cơ chế rollback tự động.
- **Bảo vệ boot loop** — phát hiện khởi động thất bại liên tiếp và chuyển sang safe mode.

---

## 2. Cách tương tác với ESP32

Có ba kênh chính để giao tiếp với thiết bị:

### A. Telegram
- Gửi tin nhắn tới bot Telegram đã cấu hình trên thiết bị.
- Thiết bị nhận qua long-polling (timeout 30 giây) và phản hồi trực tiếp.
- Hỗ trợ tối đa 4 chat ID được phép (`allowlist`).
- Thiết bị gửi thông báo khởi động khi bật nguồn.
- Cấu hình token và chat ID qua `./scripts/provision.sh`.

### B. Web Relay (HTTP)
- Chạy `./scripts/web-relay.sh` trên máy tính để khởi động relay server.
- Relay cung cấp endpoint `/api/chat` và giao diện web di động.
- Tiến trình Python chuyển tiếp tin nhắn giữa trình duyệt/client và thiết bị qua serial.
- Hỗ trợ API key và CORS origin để bảo mật.
- Dùng để kiểm thử mà không cần cài đặt Telegram.

### C. Serial / JTAG (phát triển)
- Giao tiếp trực tiếp qua cổng USB serial trong quá trình phát triển.
- Dùng `./scripts/monitor.sh <port>` để xem log và nhập lệnh.
- Trong chế độ emulator (QEMU), sử dụng UART0.

### Quy trình cài đặt ban đầu

```bash
# 1. Flash firmware
./scripts/flash.sh --kill-monitor /dev/cu.usbmodem1101

# 2. Cấu hình thông tin (WiFi + LLM + Telegram)
./scripts/provision.sh --port /dev/cu.usbmodem1101

# 3. Kiểm tra hoạt động
./scripts/web-relay.sh
```

---

## 3. Sử dụng LLM cục bộ (Ollama) với ESP32

zclaw hỗ trợ kết nối tới một **LLM chạy trên máy tính hoặc server local** thông qua [Ollama](https://ollama.com), không cần API key của bên thứ ba. ESP32 gửi yêu cầu qua WiFi tới server Ollama trong cùng mạng nội bộ.

### Cách thức hoạt động

```
ESP32 ──WiFi──► Máy tính/Server chạy Ollama (cùng mạng LAN)
                  http://<IP>:11434/v1/chat/completions
```

ESP32 sử dụng giao thức **OpenAI-compatible** (giống OpenAI và OpenRouter), do đó không cần thay đổi firmware để dùng Ollama.

### Yêu cầu

1. Máy tính hoặc server cùng mạng WiFi với ESP32.
2. [Ollama](https://ollama.com/download) đã được cài đặt và đang chạy.
3. Model đã được tải về trên máy Ollama (mặc định: `qwen3:8b`).

### Bước 1 — Cài đặt và chạy Ollama

```bash
# Tải và cài Ollama (macOS/Linux)
curl -fsSL https://ollama.com/install.sh | sh

# Tải model (ví dụ: qwen3:8b — phù hợp với phần cứng RAM thấp)
ollama pull qwen3:8b

# Chạy Ollama, cho phép kết nối từ mạng LAN
OLLAMA_HOST=0.0.0.0 ollama serve
```

> **Lưu ý:** Mặc định Ollama chỉ lắng nghe trên `127.0.0.1`. Cần đặt `OLLAMA_HOST=0.0.0.0` để ESP32 truy cập từ mạng nội bộ.

### Bước 2 — Tìm địa chỉ IP của máy chạy Ollama

```bash
# macOS
ipconfig getifaddr en0

# Linux
ip route get 1 | awk '{print $NF; exit}'
```

### Bước 3 — Nạp cấu hình vào ESP32

```bash
./scripts/provision.sh --port /dev/cu.usbmodem1101 \
  --backend ollama \
  --api-url http://<IP-máy-Ollama>:11434 \
  --model qwen3:8b
```

Script tự chuẩn hóa URL: chấp nhận dạng base (`http://192.168.1.10:11434`), `/v1`, hoặc đường dẫn đầy đủ (`/v1/chat/completions`).

Hoặc chạy theo chế độ tương tác (không dùng `--yes`):

```bash
./scripts/provision.sh --port /dev/cu.usbmodem1101
# Chọn "ollama" khi được hỏi về LLM provider
# Nhập URL endpoint của Ollama
```

### Bước 4 — Kiểm tra kết nối

```bash
./scripts/web-relay.sh
# Gửi tin nhắn thử: "xin chào"
```

Hoặc quan sát log trực tiếp:

```bash
./scripts/monitor.sh /dev/cu.usbmodem1101
```

Log thành công sẽ hiển thị:
```
Backend: Ollama, Model: qwen3:8b
```

### Lựa chọn model

| Model | RAM tối thiểu máy | Đặc điểm |
|---|---|---|
| `qwen3:8b` (mặc định) | ~6 GB | Cân bằng tốc độ/chất lượng |
| `llama3.2:3b` | ~2 GB | Nhẹ hơn, phù hợp máy yếu |
| `phi4-mini` | ~2 GB | Nhỏ gọn, phản hồi nhanh |
| `mistral:7b` | ~5 GB | Chất lượng cao |

Thay đổi model bất kỳ lúc nào (không cần flash lại):

```bash
./scripts/provision.sh --port /dev/cu.usbmodem1101 \
  --backend ollama \
  --api-url http://<IP>:11434 \
  --model llama3.2:3b
```

### Lưu ý quan trọng

- **Không cần API key**: Ollama backend bỏ qua trường `api_key`. Nếu Ollama server yêu cầu xác thực, truyền token qua `--api-key`.
- **Chỉ dùng HTTP (không TLS)**: zclaw kết nối Ollama qua HTTP thuần. Không dùng HTTPS trừ khi có reverse proxy với cert hợp lệ trong cert bundle của ESP-IDF.
- **Độ trễ phụ thuộc phần cứng**: Tốc độ sinh token của Ollama phụ thuộc vào CPU/GPU máy host, không liên quan tới ESP32.
- **Cùng mạng WiFi**: ESP32 và máy Ollama phải ở cùng mạng nội bộ (hoặc ESP32 có thể định tuyến tới server Ollama).
- **URL được lưu trong NVS**: Sau khi provision, URL Ollama được lưu vào flash NVS với key `llm_api_url` và tồn tại qua các lần reboot.

---

## 4. Phân tích kiến trúc

### Sơ đồ tổng quan

```
┌─────────────────────────────────────────────────────────────────┐
│                  ESP32 Firmware (C + FreeRTOS)                   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Khởi động (app_main)                                     │  │
│  │  NVS → OTA check → factory reset → WiFi → NTP            │  │
│  │  → LLM init → Telegram init → tools init → start tasks   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────┐   ┌──────────────────┐                    │
│  │  channel_task    │   │  telegram_task   │                    │
│  │  (Serial I/O)    │   │  (Bot polling)   │                    │
│  └────────┬─────────┘   └────────┬─────────┘                    │
│           │                      │                               │
│           └──────────┬───────────┘                               │
│                      ▼                                           │
│              input_queue (sâu 8)                                 │
│                      │                                           │
│  ┌───────────────────▼───────────────────────┐                  │
│  │  agent_task (logic chính)                 │                  │
│  │  - Giới hạn tốc độ                        │                  │
│  │  - Lịch sử hội thoại (12 lượt)            │                  │
│  │  - Vòng lặp gọi công cụ (tối đa 5 vòng)  │                  │
│  │  - Phân tích phản hồi LLM                 │                  │
│  └─────────┬──────────────────────┬───────────┘                  │
│            │                      │                              │
│            ▼                      ▼                              │
│  channel_output_queue   telegram_output_queue                    │
│  (phản hồi serial)      (tin nhắn Telegram)                      │
│                                                                   │
│  ┌──────────────────────────┐  ┌──────────────────────────┐     │
│  │  Hệ thống công cụ        │  │  cron_task (lên lịch)    │     │
│  │                          │  │  - periodic / daily      │     │
│  │  Có sẵn:                 │  │  - once / condition      │     │
│  │  gpio_read/write/all     │  │  - Nhận biết múi giờ     │     │
│  │  memory_get/set          │  │  - Đưa hành động vào     │     │
│  │  cron_set/list/delete    │  │    input_queue           │     │
│  │  i2c_scan                │  └──────────────────────────┘     │
│  │  create_tool             │                                    │
│  │  set/get/reset_persona   │                                    │
│  │  get_system_time         │                                    │
│  │                          │                                    │
│  │  Tùy chỉnh (do người dùng│                                    │
│  │  tạo qua ngôn ngữ tự nhiên│                                   │
│  └──────────────────────────┘                                    │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NVS Flash (lưu trữ bền vững)                            │   │
│  │  WiFi credentials · LLM config · Telegram config         │   │
│  │  User memory · Cron schedules · Persona setting          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
          │  WiFi / HTTPS                         │  Serial / USB
          ▼                                        ▼
  ┌────────────────────┐                  ┌──────────────────────┐
  │  LLM Backend       │                  │  Web Relay (Python)  │
  │  Anthropic / OpenAI│                  │  /api/chat endpoint  │
  │  OpenRouter / Ollama│                 │  Giao diện web mobile│
  └────────────────────┘                  └──────────────────────┘
          │
          ▼
  ┌────────────────────┐
  │  Telegram Bot API  │
  │  (long-polling)    │
  └────────────────────┘
```

### Các mẫu kiến trúc quan trọng

| Thành phần | Mô tả |
|---|---|
| **Message Queue IPC** | Input queue → Agent → Output queues (channel + Telegram). Tách biệt nguồn nhận tin và xử lý. |
| **Stateless tool execution** | Mỗi công cụ nhận JSON đầu vào, trả về JSON kết quả. Không trạng thái nội bộ. |
| **History-based conversation** | Bộ đệm cuộn 12 lượt được nạp vào context LLM mỗi yêu cầu. |
| **NVS-backed persistence** | Mọi trạng thái người dùng (memory, cron, persona, credentials) tồn tại qua reboot. |
| **Rate-limited requests** | Theo dõi số lượng yêu cầu theo phút và theo ngày để bảo vệ API key. |
| **Timezone-aware scheduling** | Đồng bộ NTP khi khởi động; chuỗi POSIX TZ lưu trong NVS; cron task kiểm tra mỗi 10 giây. |
| **Boot loop protection** | Đếm số lần khởi động thất bại liên tiếp; sau `MAX_BOOT_FAILURES` (mặc định 4) chuyển sang safe mode. |
| **OTA với rollback** | Xác nhận image ổn định sau 30 giây hoạt động bình thường; tự rollback nếu thất bại. |
| **Emulator mode (QEMU)** | Bỏ qua WiFi/Telegram; dùng stub LLM; hỗ trợ live-bridge để kiểm thử không cần phần cứng. |

### Phân bổ kích thước firmware (ESP32-S3, mặc định ~850 KiB)

| Thành phần | Kích thước | Tỷ lệ |
|---|---:|---:|
| zclaw app logic (`libmain.a`) | ~34,9 KiB | ~4,1% |
| Wi-Fi + networking stack | ~388,0 KiB | ~45,7% |
| TLS/crypto stack | ~110,3 KiB | ~13,0% |
| Cert bundle + app metadata | ~97,4 KiB | ~11,5% |
| ESP-IDF runtime / drivers / libc | ~218,8 KiB | ~25,8% |

Logic ứng dụng thực tế (zclaw) chỉ chiếm khoảng **4%** tổng dung lượng firmware; phần còn lại là overhead của stack mạng và TLS.

### Phần cứng được hỗ trợ

- **ESP32-C3**, **ESP32-S3**, **ESP32-C6** (đã kiểm thử).
- Bo mạch khuyến nghị để bắt đầu: [Seeed XIAO ESP32-C3](https://www.seeedstudio.com/Seeed-XIAO-ESP32C3-p-5431.html).
