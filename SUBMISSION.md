# Bài nộp nhóm - Cloud AI Agent Deployment Review

## Nhóm
Tên thành viên: (điền tên bạn/nhóm)

## Path đã chọn
- [x] A - Local Docker + Architecture Review
- [ ] B - Railway/Render Deploy
- [ ] C - Advanced Design: Cloud Run/ECS + Scaling Plan

## Kết quả chạy

```
$ curl http://localhost:8000/health
{"status":"ok","uptime_seconds":131,"version":"0.1.0","environment":"local"}

$ curl http://localhost:8000/ready
{"status":"ready","checks":{"api_key_configured":true,"mock_agent_loaded":true,"port_from_env":true}}

$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"test"}'
{"detail":"Missing or invalid X-API-Key"}   # thiếu key -> 401, đúng thiết kế

$ curl -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: <APP_API_KEY demo>" \
    -d '{"question":"Summarize production checklist for AI agent"}'
{"request_id":"a635160d-...","answer":"...","citations":["mock_kb:cloud","mock_kb:docker","mock_kb:security"],
 "latency_ms":3,"tokens_estimated":116}

$ python check_production_ready.py
Production readiness: 27/27 checks passed
```

## Public URL (nếu có)
Không có — Path A chỉ chạy local Docker, không deploy public.

## Quyết định kiến trúc
Platform: Local Docker (docker compose). Lý do và chi tiết xem [ADR.md](ADR.md).

## Checklist
- [x] /health hoạt động
- [x] /ready hoạt động
- [x] /ask yêu cầu X-API-Key
- [x] Env vars không hardcode (`.env` gitignored, `docker-compose.yml` đọc từ env)
- [x] Dockerfile không chạy root (`app` user, `USER app`)
- [x] Có ý tưởng rate limit/cost guard (`RATE_LIMIT_PER_MINUTE`, `MAX_QUESTION_CHARS`)
- [x] Có cách xem logs/request_id (`docker compose logs agent`, JSON có `request_id`)
- [x] Bonus +10: CI local (`.github/workflows/ci.yml`) + rate limit/cost guard demo + log latency/status

## 3 điều nhóm học được
1. `/health` chỉ báo process còn sống; `/ready` báo dependency (API key, agent, port) đã sẵn
   sàng phục vụ — hai khái niệm khác nhau, orchestrator (Docker/K8s) dùng cho mục đích khác nhau
   (restart vs. traffic routing).
2. Secret không nên nằm trong `docker-compose.yml`/source — service này fail fast nếu thiếu
   `APP_API_KEY`, buộc phải cấu hình qua env/secret file thay vì key mặc định nguy hiểm.
3. Rate limit hiện tại là in-memory theo process, không dùng được nếu scale ra nhiều instance —
   phải hiểu giới hạn của thiết kế đơn giản trước khi đem lên production thật.

## 1 rủi ro production còn lại
Rate limiting và cost guard đang lưu state trong bộ nhớ của một process — nếu scale ngang
(nhiều container/instance đứng sau load balancer) thì mỗi instance đếm riêng, giới hạn thực tế
sẽ cao hơn cấu hình dự kiến (VD: 20 req/phút × N instance). Cần Redis hoặc store dùng chung để
rate limit đúng trên toàn hệ thống.

## Bonus +10: Cloud-ready workflow & observability

Không deploy cloud thật (free tier); mô phỏng local theo đúng gợi ý của instruction (Path A).

- **CI local**: `.github/workflows/ci.yml` — checkout, cài deps, chạy `check_production_ready.py`,
  build Docker image, chạy container, curl `/health` `/ready` `/ask` (401 khi thiếu key, 200 khi
  đúng key), in `docker logs`, dọn container. Chạy được bằng GitHub Actions khi push, hoặc local
  bằng `act` nếu không có GitHub.
- **Observability**: mỗi request có JSON log kèm `request_id`, `path`, `latency_ms`, `status_code`.
  Bằng chứng từ `docker logs day12inclass-agent-1`:
  ```json
  {"ts":"2026-08-10T04:51:22Z","level":"INFO","logger":"cloud_ai_agent","message":"request_completed","request_id":"31490405-3a3a-44f8-9327-c35cf6797c09","path":"/ask","latency_ms":2,"status_code":200,"client":"172.19.0.1"}
  {"ts":"2026-08-10T04:51:22Z","level":"INFO","logger":"cloud_ai_agent","message":"request_completed","request_id":"bb98547f-db68-4524-b388-bbe417f23d1a","path":"/ask","latency_ms":1,"status_code":429,"client":"172.19.0.1"}
  ```
- **Rate limit demo** (`RATE_LIMIT_PER_MINUTE=20`): gửi 21 request `/ask` liên tiếp cùng key.
  ```
  req 18 -> 200
  req 19 -> 200
  req 20 -> 429
  req 21 -> 429
  ```
- **Cost guard demo** (`MAX_QUESTION_CHARS=2000`): gửi `question` dài 2100 ký tự.
  ```
  HTTP 413
  ```
- **README deploy**: README.md đã có hướng dẫn deploy Railway/Render (env var trong dashboard,
  Dockerfile tự nhận diện, health check path `/health`) — chưa deploy thật vì free tier bị giới
  hạn tài khoản trong buổi học, dùng mô phỏng local + CI thay thế theo đúng lưu ý của đề bài.
