# 📘 Hướng Dẫn Thực Hiện Bài Lab — Cloud Service & Deployment

> **Bài lab K4 — Level 3B, Ngày 12** | Thời lượng: 240 phút | Tổng điểm: 100 (+10 bonus)

---

## 📋 Tổng Quan Bài Lab

Bài lab này hướng dẫn bạn **đưa một AI Agent từ `localhost:8000` lên một địa chỉ công khai** (public URL) với đầy đủ các yếu tố production: bảo mật, giới hạn chi phí, containerization, và không bị mất request khi deploy bản mới.

### Kiến thức bạn sẽ học được

| Checkpoint | Kiến thức cốt lõi | Điểm |
|---|---|---:|
| CP0 | Setup môi trường, hiểu cấu trúc dự án | — |
| CP1 | 12-Factor App, structured logging, health check | 15 |
| CP2 | Docker multi-stage, bảo mật image, Docker Compose | 15 |
| CP3 | API authentication, rate limiting, cost guard | 20 |
| CP4 | Stateless service, liveness/readiness, graceful shutdown | 20 |
| CP5 | Cloud deployment (Railway/Render) | 15 |
| exercises.md | 10 câu phản ánh | 15 |
| Bonus | CI/CD với GitHub Actions | +10 |

### Cấu trúc thư mục — file bạn cần sửa (★)

```
├── app/
│   ├── config.py          ★ CP1 — Settings 12-factor
│   ├── logging_utils.py   ★ CP1 — log JSON
│   ├── main.py            ★ CP1/CP3/CP4 — FastAPI app
│   ├── auth.py            ★ CP3 — xác thực API key
│   ├── rate_limiter.py    ★ CP3 — sliding window
│   ├── cost_guard.py      ★ CP3 — ngân sách theo tháng
│   ├── store.py           ★ CP4 — lịch sử hội thoại
│   └── lifecycle.py       ★ CP4 — graceful shutdown
├── Dockerfile             ★ CP2 — multi-stage build
├── docker-compose.yml     ★ CP2 — thêm service agent
├── .dockerignore          ★ CP2 — bổ sung mục thiếu
├── exercises.md           ★ 10 câu phản ánh
├── DEPLOYMENT.md          ★ CP5 — điền URL deploy
└── utils/mock_llm.py      (cho sẵn — LLM giả)
```

---

## 🟢 CP0 — Setup (Start +0–20 phút)

### Bước 1: Tạo môi trường ảo và cài thư viện

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### Bước 2: Tạo file `.env` từ template

```bash
cp .env.example .env
```

### Bước 3: Sinh API key riêng

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Copy kết quả (ví dụ: `aB3cD4eF5gH6iJ7kL8mN9oP0qR1sT2u`) và dán vào dòng `AGENT_API_KEY=` trong file `.env`.

### Bước 4: Khởi động Redis

```bash
docker compose up -d redis
docker compose ps                  # STATE phải là running/healthy
```

> [!TIP]
> Nếu chưa cài Docker, đặt `REDIS_URL=fake://` trong `.env` để dùng Redis giả tạm. Nhớ cài Docker trước Block 2.

### ✅ Kiểm tra CP0

```bash
pytest tests/ -v -m "not docker"
```

**Kết quả mong đợi:** Hầu hết test **RỚT** (FAILED) — đúng vì bạn chưa viết code. Điều cần xác nhận: pytest **chạy được**, không có `ModuleNotFoundError` hay `ImportError`.

> [!IMPORTANT]
> **Kiến thức cần nắm:**
> - **Vì sao `.env` không được commit?** File `.env` chứa secret (API key, mật khẩu Redis...). Nếu commit, bất cứ ai clone repo đều có secret của bạn. Git lưu lịch sử vĩnh viễn — xóa file ở commit sau **không** làm nó biến mất khỏi lịch sử.
> - **Vì sao test rớt là bình thường?** Các file code có `raise NotImplementedError(...)` — đó là placeholder cho bạn viết code vào. Test rớt vì code chưa được implement, không phải vì môi trường sai.

---

## 🔵 CP1 — 12-Factor Config, Health & Logging (Start +20–60 phút)

### 🎓 Kiến thức nền tảng: 12-Factor App

**12-Factor App** là bộ 12 nguyên tắc thiết kế ứng dụng cho cloud. Nguyên tắc quan trọng nhất trong bài lab:

> **Factor III — Config**: *Lưu cấu hình trong biến môi trường (environment variables)*

```
❌ SAI:  API_KEY = "sk-proj-abc123"         # hardcode trong code
✅ ĐÚNG: API_KEY = os.environ["API_KEY"]    # đọc từ biến môi trường
```

**Tại sao?**
- Cùng 1 image chạy ở laptop, staging, production — chỉ khác biến môi trường
- Secret không nằm trong code → không bị leak khi push lên GitHub
- Thay đổi config không cần build lại code

**Fail Fast** — app nên chết ngay khi thiếu config quan trọng:
- `agent_api_key` **KHÔNG có mặc định** → thiếu biến → app crash ngay lúc khởi động → bạn phát hiện ngay
- Nếu để mặc định `"changeme"` → app chạy bình thường → ai cũng dùng key `"changeme"` để gọi API → bạn chỉ biết khi nhận hóa đơn

---

### Bước 1.1 — Cài đặt `app/config.py`

Mở file [app/config.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/config.py) và khai báo 6 trường trong class `Settings`:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    # 6 trường cần khai báo:
    port: int = 8000                                    # cổng HTTP
    agent_api_key: str                                  # BẮT BUỘC, KHÔNG có mặc định
    redis_url: str = "redis://localhost:6379/0"          # URL kết nối Redis
    rate_limit_per_minute: int = 10                     # giới hạn request/phút/user
    monthly_budget_usd: float = 10.0                    # ngân sách tối đa/user/tháng
    log_level: str = "INFO"                             # mức log
```

> [!NOTE]
> **Giải thích cách `pydantic-settings` hoạt động:**
> - Tự đọc file `.env` (nhờ `env_file=".env"` trong `model_config`)
> - Ánh xạ tên trường → biến môi trường viết HOA: `agent_api_key` ← `AGENT_API_KEY`
> - Trường không có mặc định (`agent_api_key: str`) → bắt buộc phải có trong `.env` hoặc biến môi trường → thiếu = `ValidationError`
> - `extra="ignore"` → bỏ qua biến môi trường thừa, không báo lỗi
> - `@lru_cache(maxsize=1)` trên `get_settings()` → chỉ đọc env **một lần** rồi cache, không đọc lại mỗi request

---

### Bước 1.2 — Cài đặt `app/logging_utils.py`

Mở file [app/logging_utils.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/logging_utils.py) và implement hàm `log_event()`:

```python
def log_event(event: str, level: str = "info", **fields) -> str:
    entry = {
        "event": event,
        "level": level.lower(),
        "timestamp": utc_now_iso(),
        **fields,                    # gộp thêm mọi key/value truyền vào
    }
    line = json.dumps(entry, ensure_ascii=False)
    print(line, flush=True)          # in ra stdout, flush để không bị buffer
    return line
```

> [!NOTE]
> **Kiến thức: Tại sao log phải là JSON một dòng?**
>
> | `print("đã trả lời xong")` | `log_event("ask_completed", user_id="sv01", cost_usd=0.001)` |
> |---|---|
> | Người đọc hiểu, máy không hiểu | Máy parse được → filter, count, alert |
> | Không biết ai gọi, lúc nào, tốn bao nhiêu | Có user_id, timestamp, cost_usd |
> | Không lọc được | `jq '. | select(.cost_usd > 0.01)'` |
>
> Cloud platform (Railway, Render, Datadog...) gom log **theo dòng**. JSON xuống dòng (có `indent`) → 1 log bị vỡ thành nhiều mảnh vô nghĩa.
>
> `ensure_ascii=False` → giữ nguyên tiếng Việt, không biến thành `\u1ea3`.

**Kết quả mong đợi — một dòng log mẫu:**
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:00:00+00:00", "user_id": "sv01", "cost_usd": 0.0001}
```

---

### Bước 1.3 — Cài đặt `/health` trong `app/main.py`

Mở file [app/main.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/main.py) và implement endpoint `/health`:

```python
@app.get("/health")
def health():
    if lifecycle.shutting_down:
        return JSONResponse(
            status_code=503,
            content={"status": "shutting_down"},
        )
    return {
        "status": "ok",
        "service": SERVICE_NAME,
        "version": SERVICE_VERSION,
    }
```

> [!WARNING]
> **Quy tắc quan trọng: `/health` KHÔNG được phụ thuộc bất cứ dependency nào** (Redis, database, external API...).
>
> **Tại sao?**
> - `/health` là **liveness probe** — trả lời câu hỏi: "Process này có cần restart không?"
> - Nếu `/health` gọi Redis, Redis chết 1 giây → `/health` trả 503 → orchestrator (Docker/K8s) **restart container**
> - Redis quay lại → 3 container đều đang restart → **không ai phục vụ user**
> - Sự cố Redis 1 giây biến thành sự cố toàn hệ thống nhiều phút
>
> Endpoint kiểm tra dependency là `/ready` (làm ở CP4) — nó có mục đích khác: load balancer dùng nó để **ngừng gửi traffic**, không restart container.

---

### Thử chạy

```bash
uvicorn app.main:app --reload --port 8000
curl -i http://localhost:8000/health
```

**Kết quả mong đợi:**
```
HTTP/1.1 200 OK
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}
```

### ✅ Kiểm tra CP1

```bash
pytest tests/test_cp1.py -v
```

**Kết quả mong đợi:** Tất cả test PASS (xanh).

> [!TIP]
> **Gỡ lỗi thường gặp:**
> - `ValidationError` → `.env` thiếu `AGENT_API_KEY`
> - `test_log_ra_stdout_dung_mot_dong` rớt → bạn dùng `json.dumps(..., indent=2)` — bỏ `indent`
> - `test_health_khong_phu_thuoc_dependency_nao` rớt → hàm `health()` có `Depends(...)` — bỏ đi

**Commit ngay sau khi pass:**
```bash
git add -A
git commit -m "CP1: 12-Factor config, health endpoint, structured logging"
```

---

## 🟣 CP2 — Docker (Start +60–105 phút)

### 🎓 Kiến thức nền tảng: Docker & Containerization

**Docker giải quyết vấn đề gì?**
- "Máy tôi chạy được" — máy bạn có Python 3.11, server có 3.9. Docker **đóng gói cả môi trường** vào 1 image → cùng image chạy giống nhau ở mọi nơi.

**Các khái niệm quan trọng:**

| Khái niệm | Giải thích |
|---|---|
| **Image** | Bản chụp bất biến (immutable snapshot) của ứng dụng + dependencies + OS |
| **Container** | Instance đang chạy của image (như process trên hệ điều hành) |
| **Layer** | Mỗi lệnh trong Dockerfile tạo 1 layer. Docker cache layer → build nhanh hơn |
| **Multi-stage build** | Dùng nhiều `FROM` trong 1 Dockerfile. Stage đầu cài compiler, stage cuối chỉ copy kết quả → image nhỏ |
| **Build context** | Thư mục Docker copy vào khi build. `.dockerignore` loại trừ file không cần |

---

### Bước 2.1 — Sửa `Dockerfile` thành multi-stage

Mở file [Dockerfile](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/Dockerfile) và thay toàn bộ nội dung:

```dockerfile
# ── Stage 1: Builder ────────────────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /build

# Copy requirements TRƯỚC → Docker cache layer này
# Sửa code thì không phải cài lại thư viện
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ── Stage 2: Runtime ────────────────────────────────────────
FROM python:3.11-slim AS runtime

WORKDIR /app

# Copy KẾT QUẢ từ builder, không mang theo compiler
COPY --from=builder /install /usr/local

# Tạo user thường — KHÔNG chạy bằng root
RUN useradd --create-home --uid 10001 appuser

# Copy source code
COPY app ./app
COPY utils ./utils

# Chuyển sang user thường
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:${PORT:-8000}/health').read()" || exit 1

EXPOSE ${PORT:-8000}

# Đọc PORT từ biến môi trường — cloud tự gán cổng
CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

> [!NOTE]
> **Giải thích từng điểm quan trọng:**
>
> 1. **Multi-stage build** (`AS builder` → `AS runtime`):
>    - Stage `builder` cài dependency (có thể cần compiler nặng)
>    - Stage `runtime` chỉ COPY kết quả → image tụt từ ~1GB xuống ~200MB
>    - Compiler, header files, cache pip... đều bị vứt đi
>
> 2. **Thứ tự COPY quyết định tốc độ build**:
>    - `COPY requirements.txt` → `pip install` → `COPY app` (code)
>    - Docker cache theo layer, HỦY cache từ layer đầu tiên thay đổi
>    - Sửa 1 dòng code → chỉ layer `COPY app` thay đổi → pip install dùng cache
>    - Nếu `COPY . .` trước `pip install` → sửa code = cài lại tất cả thư viện
>
> 3. **Không chạy root** (`USER appuser`):
>    - Container mặc định chạy root → lỗ hổng trong code Python → kẻ tấn công có quyền root TRONG container
>    - Nếu container share kernel với host → root trong container ≈ root trên host
>    - `USER appuser` → ngay cả khi bị exploit, attacker chỉ là user thường
>
> 4. **`0.0.0.0` thay vì `127.0.0.1`**:
>    - `127.0.0.1` = localhost = chỉ trong container gọi được
>    - `0.0.0.0` = mọi interface = bên ngoài container gọi được
>
> 5. **`${PORT:-8000}`**:
>    - Railway/Render/Cloud Run **tự gán cổng** qua biến `PORT`
>    - `:-8000` = nếu không có biến PORT, dùng 8000 làm mặc định

---

### Bước 2.2 — Bổ sung `.dockerignore`

Mở file [.dockerignore](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/.dockerignore) và thêm các mục:

```
.git
.gitignore
.env
.env.example
.venv
__pycache__
*.pyc
.pytest_cache
screenshots
*.md
tests
nginx
```

> [!WARNING]
> **`.env` PHẢI nằm trong `.dockerignore`!** Không có → `.env` (chứa API key) bị copy vào image → bạn push image lên registry → ai cũng có secret của bạn.
>
> **Cẩn thận không ignore nhầm** thứ image cần: `app/`, `utils/`, `requirements.txt` PHẢI được giữ lại.

---

### Bước 2.3 — Thêm service `agent` vào `docker-compose.yml`

Mở file [docker-compose.yml](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/docker-compose.yml) và thêm service `agent`:

```yaml
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      AGENT_API_KEY: ${AGENT_API_KEY}       # đọc từ .env, KHÔNG viết thẳng khóa
      REDIS_URL: redis://redis:6379/0       # "redis" = tên service = hostname
      RATE_LIMIT_PER_MINUTE: ${RATE_LIMIT_PER_MINUTE:-10}
      MONTHLY_BUDGET_USD: ${MONTHLY_BUDGET_USD:-10.0}
      LOG_LEVEL: ${LOG_LEVEL:-INFO}
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health').read()"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  redis-data:
```

> [!NOTE]
> **Kiến thức Docker Compose:**
>
> - **`${AGENT_API_KEY}`** — Docker Compose tự đọc biến từ file `.env` cùng thư mục. KHÔNG viết secret thẳng vào file YAML (file này được commit).
> - **`redis://redis:6379/0`** — Trong Compose, **tên service = hostname**. `redis` ở đây là tên service, không phải `localhost`. Container `agent` gọi `localhost` sẽ gọi vào chính nó, không phải Redis.
> - **`depends_on: redis`** — Agent chỉ khởi động sau khi Redis healthy.
> - **Ranh giới mạng**: mỗi container có mạng riêng. `localhost` bên trong container = chính container đó. Để giao tiếp, dùng tên service.

---

### Thử chạy

```bash
docker build -t day12-agent:prod .
docker images day12-agent:prod          # ghi lại dung lượng (mục tiêu < 500MB)

docker compose up -d
curl http://localhost:8000/health
docker compose logs agent
```

**Kết quả mong đợi:**
```
REPOSITORY       TAG    SIZE
day12-agent      prod   ~180-250MB    (thay vì ~1GB nếu 1 stage)
```

### ✅ Kiểm tra CP2

```bash
pytest tests/test_cp2.py -v
```

Kiểm tra nhanh (không build image thật):
```bash
pytest tests/test_cp2.py -v -m "not docker"
```

**Commit:**
```bash
git add -A
git commit -m "CP2: multi-stage Dockerfile, dockerignore, compose agent service"
```

---

## ☕ Giải lao (Start +105–115 phút)

---

## 🔴 CP3 — API Security (Start +115–160 phút)

### 🎓 Kiến thức nền tảng: Ba lớp bảo vệ API

Khi bạn có URL công khai, bot quét Internet **tìm thấy endpoint mới trong vài giờ**. Không bảo vệ = mỗi request của người lạ là một lần bạn trả tiền.

| Lớp | Câu hỏi | Mã HTTP | Mục đích |
|---|---|---|---|
| **Authentication** | Bạn là ai? | 401 | Chặn người không có quyền |
| **Rate Limiting** | Bạn gọi quá nhanh không? | 429 | Chống DDoS, chống abuse |
| **Cost Guard** | Bạn hết ngân sách chưa? | 402 | Giới hạn chi phí |

**Thứ tự rất quan trọng:** Auth → Rate Limit → Cost Guard → gọi LLM. Chặn **TRƯỚC** khi gọi LLM vì tiền mất ở bước gọi LLM.

---

### Bước 3.1 — Cài đặt `app/auth.py`

Mở file [app/auth.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/auth.py):

```python
def verify_api_key(
    x_api_key: str | None = Header(default=None),
    x_user_id: str | None = Header(default=None),
) -> str:
    settings = get_settings()

    # Thiếu key hoặc sai key → 401
    if x_api_key is None or not secrets.compare_digest(x_api_key, settings.agent_api_key):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="invalid or missing API key",
        )

    # Hợp lệ → trả user_id
    return x_user_id if x_user_id else ANONYMOUS_USER
```

> [!WARNING]
> **Kiến thức: Timing Attack và `secrets.compare_digest`**
>
> ```python
> # ❌ SAI — dễ bị timing attack
> if x_api_key == settings.agent_api_key:
>
> # ✅ ĐÚNG — constant-time comparison
> if secrets.compare_digest(x_api_key, settings.agent_api_key):
> ```
>
> **Toán tử `==` dừng ngay tại ký tự đầu tiên khác nhau:**
> - Key đúng: `"abcdef"` — so sánh 6 ký tự, mất ~60ns
> - Key sai ở ký tự 1: `"Xbcdef"` — so sánh 1 ký tự, mất ~10ns
> - Key sai ở ký tự 5: `"abcdXf"` — so sánh 5 ký tự, mất ~50ns
>
> Kẻ tấn công đo thời gian phản hồi → đoán đúng từng ký tự → bẻ khóa.
>
> **`secrets.compare_digest` luôn chạy hết chuỗi** — dù đúng hay sai đều mất cùng thời gian → không rò rỉ thông tin.

---

### Bước 3.2 — Cài đặt `app/rate_limiter.py`

Mở file [app/rate_limiter.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/rate_limiter.py):

```python
def hit_count(self, user_id: str, now: float | None = None) -> int:
    now = now if now is not None else time.time()
    key = self._key(user_id)
    # Xóa các entry đã ra khỏi cửa sổ 60 giây
    self.client.zremrangebyscore(key, 0, now - WINDOW_SECONDS)
    # Đếm số request còn lại trong cửa sổ
    return self.client.zcard(key)

def check(self, user_id: str, now: float | None = None) -> None:
    now = now if now is not None else time.time()
    key = self._key(user_id)

    # 1. KIỂM TRA TRƯỚC
    count = self.hit_count(user_id, now)
    if count >= self.limit:
        raise HTTPException(
            status_code=429,
            detail="rate limit exceeded",
            headers={"Retry-After": str(WINDOW_SECONDS)},
        )

    # 2. GHI NHẬN SAU — member phải DUY NHẤT
    self.client.zadd(key, {f"{now}:{uuid.uuid4().hex}": now})
    self.client.expire(key, WINDOW_SECONDS)
```

> [!NOTE]
> **Kiến thức: Sliding Window Rate Limiting**
>
> **Cấu trúc dữ liệu:** Redis Sorted Set (ZSET)
> - Mỗi request = 1 member trong ZSET, score = timestamp
> - Cửa sổ trượt 60 giây: xóa entry cũ hơn `now - 60`, đếm còn lại
>
> ```
> Timeline:  |-----60 giây------|
>            ↑ xóa hết          ↑ now
>            (zremrangebyscore)  (zcard: đếm còn lại)
> ```
>
> **Hai chi tiết dễ sai:**
>
> 1. **Kiểm tra trước, ghi nhận sau:** Nếu ghi nhận trước → request thứ `limit` bị đếm thêm 1 → bị chặn nhầm
> 2. **Member phải duy nhất:** `f"{now}:{uuid4().hex}"`. Nếu 2 request cùng timestamp mà trùng member → ZSET chỉ giữ 1 → đếm thiếu
>
> **Tại sao sliding window chứ không đếm theo phút đồng hồ?**
> - Hạn mức 10/phút. Đếm theo phút đồng hồ:
>   - 10 request lúc 10:00:59 ✅ (trong phút 10:00)
>   - 10 request lúc 10:01:01 ✅ (trong phút 10:01)
>   - → **20 request trong 2 giây!** Vẫn "đúng luật".
> - Sliding window: luôn đếm 60 giây gần nhất → không có kẽ hở

---

### Bước 3.3 — Cài đặt `app/cost_guard.py`

Mở file [app/cost_guard.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/cost_guard.py):

```python
def spent(self, user_id: str, month: str | None = None) -> float:
    value = self.client.get(self._key(user_id, month))
    if value is None:
        return 0.0
    return float(value)

def check(self, user_id: str, estimated_cost: float = 0.0, month: str | None = None) -> None:
    if self.spent(user_id, month) + estimated_cost > self.budget:
        raise HTTPException(
            status_code=402,
            detail="monthly budget exceeded",
        )

def record(self, user_id: str, cost: float, month: str | None = None) -> float:
    key = self._key(user_id, month)
    total = self.client.incrbyfloat(key, cost)
    self.client.expire(key, KEY_TTL_SECONDS)
    return float(total)
```

> [!NOTE]
> **Kiến thức: Rate Limit vs Cost Guard — tại sao cần cả hai?**
>
> | Tình huống | Rate Limit (10 req/phút) | Cost Guard ($10/tháng) |
> |---|---|---|
> | User gửi 100 req ngắn trong 1 phút | ❌ Chặn | ✅ Cho qua (rẻ) |
> | User gửi 1 req dài 50.000 token | ✅ Cho qua | ❌ Chặn (đắt) |
>
> - **Rate limit**: giới hạn **số lượng** request → chống DDoS, abuse
> - **Cost guard**: giới hạn **số tiền** → chống cháy ngân sách
> - 10 req/phút nghe an toàn, nhưng mỗi req 50k token → ngân sách bay trong vài phút
>
> **Key Redis:** `cost:sv01:2026-09` → tự reset sang tháng mới (tên key theo `YYYY-MM`)
>
> **`incrbyfloat`:** cộng dồn atomic trên Redis — an toàn khi nhiều container cùng ghi

---

### Bước 3.4 — Cài đặt `/ask` trong `app/main.py`

Mở file [app/main.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/main.py) và implement endpoint `/ask`:

```python
@app.post("/ask")
def ask(
    payload: AskRequest,
    user_id: str = Depends(verify_api_key),
    store: ConversationStore = Depends(get_store),
    limiter: RateLimiter = Depends(get_rate_limiter),
    guard: CostGuard = Depends(get_cost_guard),
):
    # 1. Rate limit check → 429
    limiter.check(user_id)

    # 2. Cost guard check → 402
    guard.check(user_id)

    # 3. Lấy lịch sử hội thoại
    history = store.get_history(user_id)

    # 4. Gọi LLM (mock)
    result = ask_llm(payload.question, history)

    # 5. Lưu hội thoại
    store.append(user_id, "user", payload.question)
    store.append(user_id, "assistant", result["answer"])

    # 6. Ghi nhận chi phí
    guard.record(user_id, result["cost_usd"])

    # 7. Log
    log_event(
        "ask_completed",
        user_id=user_id,
        tokens_in=result["tokens_in"],
        tokens_out=result["tokens_out"],
        cost_usd=result["cost_usd"],
    )

    # 8. Trả kết quả
    return {
        "answer": result["answer"],
        "user_id": user_id,
        "history_length": len(history),
        "cost_usd": result["cost_usd"],
        "tokens": {"in": result["tokens_in"], "out": result["tokens_out"]},
    }
```

> [!IMPORTANT]
> **Luồng xử lý — thứ tự rất quan trọng:**
> ```
> verify_api_key (Depends)  ──▶  limiter.check  ──▶  guard.check
>                                                        │
>                             store.get_history ◀────────┘
>                                      │
>                                   ask_llm        ← TIỀN MẤT Ở ĐÂY
>                                      │
>                             store.append × 2  ──▶  guard.record  ──▶  log_event
> ```
> - Auth (401) → Rate limit (429) → Cost guard (402) → **rồi mới gọi LLM**
> - Chặn SAU khi gọi LLM = vừa mất tiền vừa trả lỗi cho user

---

### Thử chạy

```bash
# Không key → 401
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" -d '{"question":"Hello"}'

# Có key → 200
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <your-key>" -H "X-User-Id: sv01" \
  -d '{"question":"Docker là gì?"}'

# Gọi 15 lần → cuối cùng phải 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: <your-key>" -H "X-User-Id: sv01" \
    -d '{"question":"test"}'
done; echo
```

**Kết quả mong đợi:**
```
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

### ✅ Kiểm tra CP3

```bash
pytest tests/test_cp3.py -v
```

**Commit:**
```bash
git add -A
git commit -m "CP3: API auth, sliding window rate limit, cost guard"
```

---

## 🟡 CP4 — Scaling & Reliability (Start +160–200 phút)

### 🎓 Kiến thức nền tảng: Stateless Service & Scale ngang

**Vấn đề:** Một instance không đủ. Cloud có thể restart container bất cứ lúc nào (vá lỗi, dời máy, deploy mới). Hệ thống phải chịu được mà user không nhận ra.

**Stateless Service** — nguyên tắc cốt lõi:

```
❌ State trong RAM (dict Python):
  Container A: history = {"sv01": [...]}
  Container B: history = {}                 ← sv01 "mất trí nhớ"

✅ State trong Redis (chia sẻ):
  Container A ──┐
  Container B ──┼── Redis: history:sv01 = [...]
  Container C ──┘
```

Với 3 instance sau load balancer, request 1 vào container A, request 2 vào container B. Nếu lịch sử nằm trong RAM của A → B không biết gì → agent "mất trí nhớ" ngẫu nhiên.

---

### Bước 4.1 — Cài đặt `app/store.py`

Mở file [app/store.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/store.py):

```python
def ping(self) -> bool:
    try:
        return self.client.ping()
    except Exception:
        return False

def append(self, user_id: str, role: str, content: str) -> None:
    key = self._key(user_id)
    self.client.rpush(key, json.dumps({"role": role, "content": content}, ensure_ascii=False))
    # Giữ tối đa HISTORY_MAX_MESSAGES message MỚI NHẤT
    self.client.ltrim(key, -HISTORY_MAX_MESSAGES, -1)
    # Hội thoại cũ tự hết hạn
    self.client.expire(key, HISTORY_TTL_SECONDS)

def get_history(self, user_id: str) -> list[dict]:
    key = self._key(user_id)
    items = self.client.lrange(key, 0, -1)
    return [json.loads(item) for item in items]
```

> [!NOTE]
> **Chi tiết quan trọng:**
>
> - **`rpush`**: thêm vào cuối list (message mới nhất ở cuối)
> - **`ltrim(key, -HISTORY_MAX_MESSAGES, -1)`**: giữ N phần tử **cuối** (mới nhất)
>   - `-HISTORY_MAX_MESSAGES` = vị trí bắt đầu tính từ cuối
>   - `-1` = phần tử cuối cùng
>   - ⚠️ `ltrim(key, 0, N-1)` sẽ giữ N phần tử **đầu** (cũ nhất) — SAI!
> - **`expire`**: key tự xóa sau 7 ngày → Redis không đầy dần
> - **`ping()` phải nuốt mọi exception**: nó dùng cho `/ready`. Exception thoát ra → 500 thay vì 503
>
> **Tại sao giới hạn history?**
> - Prompt gửi tới LLM = câu hỏi + history
> - History không giới hạn → prompt dài vô hạn → tiền token vô hạn
> - 20 message gần nhất là đủ context

---

### Bước 4.2 — Cài đặt `/ready` trong `app/main.py`

```python
@app.get("/ready")
def ready(store: ConversationStore = Depends(get_store)):
    if lifecycle.shutting_down:
        return JSONResponse(
            status_code=503,
            content={"status": "shutting_down"},
        )
    if not store.ping():
        return JSONResponse(
            status_code=503,
            content={"status": "not ready", "redis": False},
        )
    return {"status": "ready", "redis": True}
```

> [!IMPORTANT]
> **Kiến thức: Liveness vs Readiness Probe**
>
> | | `/health` (Liveness) | `/ready` (Readiness) |
> |---|---|---|
> | **Câu hỏi** | Process còn sống không? | Nhận traffic được chưa? |
> | **Kiểm tra dependency** | ❌ **KHÔNG** | ✅ **CÓ** (Redis) |
> | **Trả 503 thì sao** | Orchestrator **RESTART** container | LB **NGỪNG GỬI** request |
> | **Hậu quả nếu gộp** | Redis chết 30s → restart tất cả → sập hoàn toàn |
>
> **Kịch bản thảm họa khi gộp làm một:**
> 1. Redis mất kết nối 30 giây
> 2. `/health` (gộp check Redis) → trả 503 cho cả 3 container
> 3. Orchestrator restart cả 3 container cùng lúc
> 4. Redis quay lại → nhưng không container nào đang chạy → **sập toàn hệ thống**
>
> **Tách ra:**
> 1. Redis mất kết nối 30 giây
> 2. `/health` → 200 (process vẫn sống) → không restart
> 3. `/ready` → 503 → LB ngừng gửi traffic (nhưng container vẫn chạy)
> 4. Redis quay lại → `/ready` → 200 → LB gửi traffic lại → **tự hồi phục**

---

### Bước 4.3 — Cài đặt `app/lifecycle.py`

Mở file [app/lifecycle.py](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/app/lifecycle.py):

```python
def request_shutdown(self, signum=None, frame=None) -> None:
    self.shutting_down = True
    # Gọi lại handler cũ (của uvicorn)
    previous = self._previous.get(signum)
    if callable(previous):
        previous(signum, frame)

def install(self) -> None:
    for sig in (signal.SIGTERM, signal.SIGINT):
        self._previous[sig] = signal.getsignal(sig)   # nhớ handler cũ
        signal.signal(sig, self.request_shutdown)      # rồi mới ghi đè
```

> [!NOTE]
> **Kiến thức: Graceful Shutdown**
>
> **Luồng deploy mới:**
> 1. Bạn push code mới → platform build image mới
> 2. Platform gửi **SIGTERM** cho container cũ
> 3. Container nhận SIGTERM → `shutting_down = True`
> 4. `/health` trả 503 → load balancer **rút container ra khỏi rotation**
> 5. Container xử lý nốt request đang chạy
> 6. Uvicorn handler (được gọi lại) tắt server gracefully
> 7. Container mới lên thay
>
> **Không xử lý SIGTERM?**
> - Container bỏ qua SIGTERM → vẫn nhận request mới
> - Platform đợi 10-30 giây → hết kiên nhẫn → **SIGKILL** (giết cứng)
> - Request đang xử lý bị cắt giữa chừng → user thấy 502
>
> **Cái bẫy:** Mỗi tín hiệu chỉ có **1 handler**. Đăng ký handler của bạn = ghi đè handler của uvicorn. Quên gọi lại handler cũ → app bật cờ "đang tắt" rồi... **chạy tiếp mãi mãi** cho đến khi bị SIGKILL.

---

### Thử chạy

```bash
docker compose up -d --scale agent=3
docker compose ps                      # 3 container agent

# Gọi nhiều lần — history_length phải TĂNG DẦN dù đổi container
for i in $(seq 1 5); do
  curl -s -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: <your-key>" -H "X-User-Id: sv01" \
    -d '{"question":"lượt '$i'"}' | python -c "import json,sys; print(json.load(sys.stdin)['history_length'])"
done
```

**Kết quả mong đợi:** `0 2 4 6 8` (tăng dần đều dù có 3 container)

### ✅ Kiểm tra CP4

```bash
pytest tests/test_cp4.py -v
```

**Commit:**
```bash
git add -A
git commit -m "CP4: Redis store, readiness probe, graceful shutdown"
```

---

## 🟠 CP5 — Cloud Deployment (Start +200–230 phút)

### 🎓 Kiến thức nền tảng: Triển khai lên Cloud

**Mục tiêu:** Có 1 URL HTTPS công khai (`https://xxx.up.railway.app`) mà ai cũng gọi được.

### Chọn Platform

| Platform | Độ khó | Free tier | Redis |
|---|---|---|---|
| **Railway** ⭐ | Dễ nhất | $5 credit | Có, 1 click |
| **Render** ⭐⭐ | Trung bình | 750h/tháng | Có (Key Value) |
| Cloud Run ⭐⭐⭐ | Khó | 2M req/tháng | Cần thêm Upstash |

> [!TIP]
> **Chọn Railway nếu muốn xong nhanh.** Cả Railway và Render đều đọc `Dockerfile` bạn vừa viết.

---

### Đường Railway (khuyến nghị)

```bash
# 1. Cài CLI và đăng nhập
npm i -g @railway/cli
railway login

# 2. Tạo project
railway init                       # đặt tên project

# 3. Thêm Redis
railway add --database redis       # tự sinh biến REDIS_URL

# 4. Set biến môi trường
railway variables --set AGENT_API_KEY=<khóa-của-bạn> \
                  --set RATE_LIMIT_PER_MINUTE=10 \
                  --set MONTHLY_BUDGET_USD=10.0 \
                  --set LOG_LEVEL=INFO

# 5. Deploy
railway up                         # build từ Dockerfile

# 6. Tạo domain công khai
railway domain                     # sinh URL HTTPS
```

> [!IMPORTANT]
> - Railway **tự set biến `PORT`** — đừng ghi đè
> - Kiểm tra `REDIS_URL` đã gắn vào service agent (dashboard → service → Variables)
> - URL Railway có dạng: `https://xxx.up.railway.app`

### Đường Render

1. Push repo lên GitHub (public)
2. [render.com](https://render.com) → **New** → **Blueprint** → chọn repo
3. Render đọc `render.yaml`, tạo web service + Redis
4. Điền `AGENT_API_KEY` khi Render hỏi
5. Chờ build xong

---

### Kiểm tra bản deploy

```bash
URL=https://<domain-cua-ban>

# 1. Liveness — mong đợi 200
curl -i $URL/health

# 2. Readiness — mong đợi 200 (đã nối Redis)
curl -i $URL/ready

# 3. Không có key — mong đợi 401
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

---

### Điền `DEPLOYMENT.md`

Mở [DEPLOYMENT.md](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/DEPLOYMENT.md) và điền:
- Họ tên, MSSV
- Public URL thật
- Platform đã chọn
- Danh sách biến môi trường (chỉ TÊN, **KHÔNG dán giá trị API key**)
- Output các lệnh curl
- Ảnh chụp dashboard → `screenshots/`

Thêm vào `.env` (ở máy bạn, KHÔNG commit):
```bash
DEPLOY_API_KEY=<đúng giá trị AGENT_API_KEY bạn set trên cloud>
```

### Không deploy được?

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. `docker compose up -d` → chụp screenshot
3. Ghi lý do vào `DEPLOYMENT.md`
4. CP5 tối đa 9/15 điểm

### ✅ Kiểm tra CP5

```bash
pytest tests/test_cp5.py -v
```

**Commit:**
```bash
git add -A
git commit -m "CP5: deployed to Railway/Render, DEPLOYMENT.md updated"
```

---

## 🌟 Bonus — CI/CD với GitHub Actions (+10 điểm)

> Chỉ làm khi CP1–CP5 đã ổn. Có thể làm ở nhà.

### 🎓 Kiến thức: CI/CD là gì?

| Viết tắt | Nghĩa | Mục đích |
|---|---|---|
| **CI** | Continuous Integration | Tự động kiểm tra mọi thay đổi (test, build) |
| **CD** | Continuous Deployment | Tự động đưa code đã kiểm tra lên production |

**Vấn đề khi deploy tay:**
1. Deploy code chưa test → production lỗi
2. Không ai biết ai deploy commit nào lúc nào
3. "Máy tôi build được" → hỏng trên server

### Tạo `.github/workflows/ci.yml`

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pytest tests/ -v --ignore=tests/test_cp5.py --ignore=tests/test_bonus_cicd.py
        env:
          AGENT_API_KEY: ci-dummy
          REDIS_URL: "fake://"

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t day12-agent:ci .

  deploy:
    needs: [test, build]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Thêm step deploy Railway/Render ở đây
      # Railway: railway up
      # Render: curl deploy hook
      - name: Smoke test
        run: |
          sleep 45
          curl -fsS "${{ vars.PUBLIC_URL }}/health"
```

### Thêm badge vào README.md

```markdown
![CI](https://github.com/<username>/<repo-name>/actions/workflows/ci.yml/badge.svg)
```

### ✅ Kiểm tra Bonus

```bash
pytest tests/test_bonus_cicd.py -v
```

---

## 📝 Wrap-up (Start +230–240 phút)

### 1. Trả lời 10 câu trong `exercises.md`

Mở [exercises.md](file:///home/lqaq/PROJECT/AI20K/WEEK01/day12_28092026/LAB/K4-L3B-LamQuangAnhQuan-2A202602467-Cloud-Service-And-Deployment/exercises.md) và trả lời bằng lời của bạn, dựa trên quan sát thực tế.

### 2. Chấm điểm

```bash
python grade.py
```

### 3. Kiểm tra an toàn

```bash
# .env không bị commit?
git ls-files | grep "^\.env$" && echo "NGUY HIỂM: .env đang bị theo dõi!"
```

### 4. Nộp bài

```bash
git add -A
git commit -m "Hoàn thành lab Day 12"
git push
```

---

## 📊 Bảng Tổng Kết HTTP Status Code

| Mã | Ý nghĩa | Khi nào xuất hiện |
|---|---|---|
| 200 | OK | Mọi thứ ổn |
| 401 | Unauthorized | Thiếu/sai API key |
| 402 | Payment Required | Hết ngân sách tháng |
| 422 | Unprocessable Entity | Body sai định dạng |
| 429 | Too Many Requests | Vượt rate limit |
| 503 | Service Unavailable | Chưa ready hoặc đang tắt dần |

---

## 🔧 Bảng Tra Lỗi Thường Gặp

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| `ValidationError: agent_api_key Field required` | Thiếu `.env` hoặc thiếu biến | `cp .env.example .env` rồi điền khóa |
| `ConnectionError: localhost:6379` | Redis chưa chạy | `docker compose up -d redis` hoặc `REDIS_URL=fake://` |
| `ModuleNotFoundError: No module named 'app'` | Chạy pytest từ thư mục con | Chạy từ gốc repo |
| `curl: (7) Failed to connect` | uvicorn bind `127.0.0.1` trong container | Đổi sang `--host 0.0.0.0` |
| Container tắt ngay | Thiếu biến môi trường | `docker compose logs agent` |
| Image > 500MB | 1 stage hoặc không dùng slim | Multi-stage + `python:3.11-slim` |
| 429 xuất hiện quá sớm | `zadd` trước `zcard` | Kiểm tra trước, ghi nhận sau |
| `/ready` luôn 200 dù Redis chết | Không dùng kết quả `ping()` | `if not store.ping(): return 503` |
| Deploy xong health fail | App không đọc `$PORT` | `--port ${PORT:-8000}` |

---

## 📋 Checklist Trước Khi Nộp

- [ ] Repo đúng tên `K4-L3B-DAY12-<HoVaTen>-<MSSV>-CloudServicesAndDeployment`
- [ ] `pytest tests/ -v` — biết rõ test nào pass, nào fail, vì sao
- [ ] `python grade.py` — xem điểm, mục tiêu ≥ 75/100
- [ ] `exercises.md` — đủ 10 câu, viết bằng lời của mình
- [ ] `DEPLOYMENT.md` — có Public URL thật, không dán giá trị API key
- [ ] `screenshots/` — có ảnh dashboard và ảnh gọi `/health`
- [ ] `.env` **không** nằm trong repo
- [ ] Không còn `NotImplementedError` nào trong `app/`
- [ ] Có commit ở nhiều mốc thời gian
- [ ] *(Bonus)* `.github/workflows/ci.yml` chạy xanh, README có badge `passing`
