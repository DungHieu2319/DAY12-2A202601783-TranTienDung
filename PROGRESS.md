# Tiến độ làm bài — K3 Ngày 12

Nhật ký các bước đã thực hiện trong project này. Cập nhật thêm mỗi khi hoàn
thành một checkpoint mới.

## Môi trường

- Cài đặt **uv** làm trình quản lý dependency Python:
  - `pyproject.toml` — project `day12-cloud-deployment`, `requires-python >= 3.11`.
  - Dependencies runtime (fastapi, uvicorn, pydantic, pydantic-settings, redis,
    python-dotenv) và dev (pytest, httpx, fakeredis, PyYAML) lấy từ
    `requirements.txt` cũ, tách vào `[dependency-groups]`.
  - `uv.lock` đã generate, `.venv` đã sync (31 packages).
  - Chạy code: `uv run <cmd>` (vd: `uv run pytest`, `uv run uvicorn app.main:app --reload`).
- Tạo file `.env` từ `.env.example` (đã nằm trong `.gitignore`, không bị commit).
- Chạy Redis cục bộ qua Docker: `docker compose up -d redis` (container
  `...-redis-1`, port `6379`, có healthcheck).

## Checkpoint 1 — 12-Factor Config, Health Check & Structured Logging

**Trạng thái: ✅ Hoàn thành — 13/13 test pass (`pytest tests/test_cp1.py`)**

File đã sửa:

- **`app/config.py`** — khai báo 6 field trong class `Settings`:
  `port` (mặc định 8000), `agent_api_key` (bắt buộc, không mặc định — để
  fail-fast khi thiếu secret), `redis_url`, `rate_limit_per_minute`,
  `monthly_budget_usd`, `log_level`.
- **`app/logging_utils.py`** — hàm `log_event()`: build dict
  `{event, level (viết thường), timestamp, **fields}`, in ra stdout dưới
  dạng JSON một dòng (`json.dumps(..., ensure_ascii=False)`), trả về chuỗi đó.
- **`app/main.py`** — endpoint `/health`: trả `503
  {"status": "shutting_down"}` khi `lifecycle.shutting_down = True`, ngược
  lại trả `200 {"status": "ok", "service": ..., "version": ...}`; không phụ
  thuộc Redis hay bất kỳ dependency nào (liveness probe phải nhẹ).

## Checkpoint 2 — Docker & Compose

**Trạng thái: ✅ Hoàn thành — 16/16 test pass (`pytest tests/test_cp2.py -v`,
bao gồm cả build Docker image thật). Image dựng ra: 271MB (< 500MB yêu cầu).**

File đã sửa:

- **`Dockerfile`** — multi-stage build:
  - Stage `builder` (`python:3.11-slim AS builder`): `COPY requirements.txt`
    rồi `pip install --user` trước khi có source code, tận dụng layer cache.
  - Stage runtime (`python:3.11-slim`): tạo user thường `appuser` (uid 1000),
    copy kết quả cài đặt + source từ builder, `chown` rồi `USER appuser` —
    không chạy bằng root.
  - `HEALTHCHECK` gọi `GET /health` qua `urllib.request` trong Python (không
    cần cài `curl` thêm vào image).
  - Đọc cổng từ biến môi trường `PORT` (mặc định 8000), `CMD` dùng shell form
    `--port ${PORT:-8000}` để cloud tự gán cổng khác vẫn chạy đúng.
- **`.dockerignore`** — loại `.env`, `.git`, `__pycache__`, `.venv`,
  `tests/`, `screenshots/`, các file `.md`... khỏi build context; không loại
  nhầm `app/`, `utils/`, `requirements.txt`.
- **`docker-compose.yml`** — thêm service `agent`: `build: .`, map cổng
  `8000:8000`, `AGENT_API_KEY: ${AGENT_API_KEY}` (đọc từ `.env`, không
  hardcode), `REDIS_URL: redis://redis:6379/0` (hostname = tên service
  trong compose), `depends_on: redis` (chờ `service_healthy`), có
  `healthcheck` riêng gọi `/health`.

## Checkpoint 3 — Auth, Rate Limit, Cost Guard

**Trạng thái: ✅ Hoàn thành — 22/22 test pass (`pytest tests/test_cp3.py -v`).
Không regress CP1/CP2 (27/27 test static vẫn xanh).**

File đã sửa:

- **`app/auth.py`** — `verify_api_key()`: lấy khóa đúng từ
  `get_settings().agent_api_key`, so sánh với `X-API-Key` bằng
  `secrets.compare_digest` (chống timing attack, không dùng `==`); thiếu/sai
  key → `401`. Trả về `X-User-Id` nếu client gửi, ngược lại `ANONYMOUS_USER`.
- **`app/rate_limiter.py`** — sliding window bằng Redis ZSET:
  - `hit_count()`: xoá entry cũ hơn `WINDOW_SECONDS` (`zremrangebyscore`),
    đếm bằng `zcard`.
  - `check()`: đếm trước — nếu `>= limit` raise `429` (kèm header
    `Retry-After`); chưa vượt thì `zadd` member duy nhất
    (`f"{now}:{uuid4().hex}"` để 2 request cùng timestamp không đè nhau) rồi
    `expire` key theo cửa sổ.
- **`app/cost_guard.py`** — ngân sách theo tháng (key theo `user_id` +
  tháng UTC):
  - `spent()`: đọc từ Redis, ép `float`, chưa có key → `0.0`.
  - `check()`: `spent + estimated_cost > budget` → raise `402`.
  - `record()`: `incrbyfloat` cộng dồn chi phí, refresh `expire` (TTL ~40
    ngày để còn đối soát sang tháng sau).
- **`app/main.py`** — endpoint `/ask`, đúng thứ tự: `limiter.check` (429) →
  `guard.check` (402) → lấy lịch sử → gọi `ask_llm` (mock) → ghi lịch sử 2
  lượt (user + assistant) vào store → `guard.record` chi phí thật →
  `log_event("ask_completed", ...)` → trả `answer`, `user_id`,
  `history_length`, `cost_usd`, `tokens {in, out}`. Check trước khi gọi LLM
  để không mất tiền rồi mới báo lỗi.

## Checkpoint 4 — Statelessness, Readiness, Graceful Shutdown

**Trạng thái: ✅ Hoàn thành — 19/19 test pass (`pytest tests/test_cp4.py -v`).
Regression toàn bộ CP1–CP4: 68/68 pass.**

File đã sửa:

- **`app/store.py`** (`ConversationStore`) — lịch sử hội thoại lưu trong
  Redis List, không nằm trong RAM của process (stateless — nhiều instance
  cùng thấy một dữ liệu):
  - `ping()`: gọi `client.ping()` trong try/except, Redis chết → trả
    `False` thay vì để exception văng ra (dùng cho `/ready`).
  - `append()`: `rpush` lượt chat mới → `ltrim` chỉ giữ
    `HISTORY_MAX_MESSAGES` (20) tin gần nhất → `expire` theo
    `HISTORY_TTL_SECONDS` (7 ngày) để tự dọn.
  - `get_history()`: `lrange` toàn bộ rồi `json.loads` từng phần tử, cũ
    nhất trước; chưa có gì → list rỗng.
- **`app/lifecycle.py`** (`Lifecycle`) — graceful shutdown khi nhận SIGTERM
  (Docker/Railway/Cloud Run gửi lúc deploy bản mới) hoặc SIGINT (Ctrl+C):
  - `install()`: với mỗi signal, lưu lại handler cũ (`signal.getsignal`)
    rồi ghi đè bằng `request_shutdown`.
  - `request_shutdown()`: bật `shutting_down = True`, rồi **gọi lại handler
    cũ** (quan trọng — nếu không gọi lại thì ghi đè mất handler dừng server
    thật sự của uvicorn, server treo tới khi bị SIGKILL).
- **`app/main.py`** — endpoint `/ready`: 503 `shutting_down` nếu đang tắt
  dần → 503 `{"status": "not ready", "redis": false}` nếu `store.ping()`
  fail → ngược lại 200 `{"status": "ready", "redis": true}`. Khác `/health`
  (CP1) ở chỗ `/ready` ĐƯỢC PHÉP kiểm tra dependency (Redis), vì load
  balancer dùng nó để quyết định có đẩy traffic vào instance không.

## Checkpoint 5 — Deploy thật lên cloud

**Trạng thái: ⬜ Chưa làm** — `DEPLOYMENT.md` chưa điền thông tin cá nhân /
platform, chưa có deployment public.

## Bonus — CI/CD với GitHub Actions

**Trạng thái: ⬜ Chưa làm** — chưa có thư mục `.github/workflows/`.

## Baseline test toàn repo

Chạy `uv run pytest -q` từ gốc repo (trước khi sửa CP1):
**67 failed, 8 passed, 5 skipped, 16 errors** — toàn bộ do các TODO của
CP1–CP5/bonus chưa cài đặt, không có lỗi setup/import/kết nối Redis.

## Số liệu để làm `exercises.md`

- **Câu 3 — Kích thước image (CP2)**: đo thật bằng `docker images | grep agent`.

  | Bản | Dung lượng |
  |-----|-----------|
  | 1 stage (`python:3.11` đầy đủ, `COPY . .` rồi mới `pip install`) | 1.73GB |
  | Multi-stage (`python:3.11-slim`, builder + runtime, hiện tại) | 271MB |

  Chênh lệch ≈ 1.46GB. Phần lớn là do: (1) base image `python:3.11` đầy đủ
  mang theo build toolchain (gcc, make, headers...) không cần lúc chạy, còn
  `-slim` thì không; (2) stage builder (chứa pip cache, compiler tạm dùng để
  build một số wheel) bị bỏ lại hoàn toàn, runtime chỉ copy đúng thư mục
  `~/.local` (package đã cài) sang, không mang theo cache/tooling đó.

- **Câu 2 — Log JSON mẫu (CP1)**: ví dụ output thật của `log_event`:
  ```json
  {"event": "service_started", "level": "info", "timestamp": "2026-08-10T...", "service": "day12-agent", "version": "1.0.0"}
  ```
  (test `test_log_event_gan_them_truong_tuy_y` xác nhận field tuỳ ý như
  `user_id`, `cost_usd` được gộp vào JSON).
