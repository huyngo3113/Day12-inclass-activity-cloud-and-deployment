# Architecture Decision Record - Cloud AI Agent

## 1. Scenario
Agent phục vụ nội bộ nhóm/lớp học để demo pattern production-ready cho AI service.
Traffic thấp (vài chục request/phút, đúng với `RATE_LIMIT_PER_MINUTE=20` mặc định).
Không có dữ liệu nhạy cảm thật — câu hỏi người dùng bị giới hạn độ dài (`MAX_QUESTION_CHARS`),
LLM là mock nên không có payload gửi ra ngoài.

## 2. Chọn platform
- Option đã chọn: Local Docker (Path A) — chạy qua `docker compose up --build`.
- Lý do chọn: mục tiêu bài học là hiểu kiến trúc production-ready (health check, auth, rate
  limit, structured log, non-root container) trước khi tốn chi phí/độ phức tạp deploy thật.
  Docker Compose mô phỏng đúng cách container sẽ chạy trên Railway/Render (cùng Dockerfile).
- Trade-off chấp nhận: không có public URL, không test network/DNS/TLS thật của PaaS; không
  có auto-restart/scaling của platform thật — chỉ mô phỏng qua `HEALTHCHECK` trong Dockerfile.

## 3. Kiến trúc tổng quan
Client (curl) -> FastAPI app (`app/main.py`, request-id middleware) -> `require_api_key`
(`app/security.py`, header `X-API-Key`) -> rate limit + cost guard (độ dài input) ->
`MockAgent` (`app/agent.py`) -> structured JSON log (`app/logging_config.py`, có
`request_id`) -> stdout (container log).

## 4. Production checklist
- [x] Env vars, không hardcode secrets — `.env` (gitignored), settings đọc qua `app/settings.py`,
      fail fast nếu thiếu `APP_API_KEY`.
- [x] Dockerfile multi-stage, non-root — build stage tạo venv, runtime stage `chown` cho user `app`.
- [x] /health và /ready — trả `status`, `uptime_seconds`; `/ready` kiểm tra 3 điều kiện.
- [x] API key/JWT — `X-API-Key` header, thiếu/sai trả 401.
- [x] Rate limit/cost guard — `RATE_LIMIT_PER_MINUTE`, `MAX_QUESTION_CHARS` (413 nếu vượt).
- [x] Structured logs + request_id — mỗi request có `request_id` (UUID hoặc header truyền vào),
      log JSON có `ts`, `level`, `path`, `latency_ms`, `status_code`.
- [ ] Rollback/redeploy plan — chưa áp dụng vì chưa deploy thật lên platform (Path A).

## 5. Câu hỏi còn mở
- Nếu tăng từ 1 user lên 100 users: rate limit hiện lưu state in-memory theo process — cần
  chuyển sang Redis/shared store nếu chạy nhiều instance (horizontal scale), vì mỗi container
  sẽ có bộ đếm riêng.
- Nếu LLM API chậm/lỗi: bản mock luôn trả nhanh, không mô phỏng timeout/retry. Production cần
  timeout + retry với backoff + circuit breaker quanh lệnh gọi LLM thật.
- Nếu chi phí token tăng bất thường: cần log `tokens_estimated` (đã có trong response) vào
  metrics/dashboard, cộng dồn theo API key/thời gian để cảnh báo, hiện tại chỉ log per-request.
