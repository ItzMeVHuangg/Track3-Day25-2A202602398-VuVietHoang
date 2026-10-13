# Plan — Lab 25B Game Day

## Phase 0 — Setup
- [x] [run] `python -m venv .venv` + activate
- [x] [run] `pip install -e ".[dev]"`
- [x] [run] `python -m pytest -q` (xem baseline fail ở đâu)
- [x] [run] `python scripts/run_gameday.py --client naive` (xem v1 thất bại trên 6 băng)

## Phase A — Phân loại lỗi & retry (12đ)
- [x] [code] `src/gameday_lab/errors.py`: `classify()`, `is_retryable()`, `counts_toward_breaker()`, `failover_helps()`
- [x] [code] `src/gameday_lab/retry.py`: `FullJitterBackoff.delay()` (full jitter, tôn trọng `retry_after`)
- [x] [code] `src/gameday_lab/retry.py`: `RetryBudget` (token bucket)
- [x] [run] `python -m pytest -q tests/test_errors_retry.py`

## Phase B — Circuit breaker sliding-window (12đ)
- [x] [code] `src/gameday_lab/breaker.py`: trip theo tỷ lệ lỗi (window_size, min_calls)
- [x] [code] `src/gameday_lab/breaker.py`: trip theo slow-call (slow_call_s, slow_rate_threshold)
- [x] [code] `src/gameday_lab/breaker.py`: HALF-OPEN (half_open_max_probes, half_open_successes)
- [x] [code] `src/gameday_lab/breaker.py`: `Permit` + bỏ qua kết quả generation cũ
- [x] [run] `python -m pytest -q tests/test_breaker.py`

## Phase C — ResilientClient (30đ)
- [x] [code] `src/gameday_lab/client.py`: `ladder(task)` (fallback order, lọc theo khả năng tool-calling)
- [x] [code] `src/gameday_lab/client.py`: `start` / `on_success` / `on_error` / `on_cancel` (ghép Part A+B)
- [x] [code] `configs/gameday.yaml`: tinh chỉnh khối `client:`
- [x] [run] `python -m pytest -q tests/test_client.py`
- [x] [run] `python scripts/run_gameday.py --client both` (6/6 băng PASS)

## Phase G — Semantic cache & stale-if-error (10đ, bắt buộc, làm ngay sau C)
- [x] [code] `src/gameday_lab/cache.py`: `SemanticCache.lookup/store/invalidate` (scope, threshold, TTL, LRU)
- [x] [code] `src/gameday_lab/cache.py`: `CachingClient` (hit→Serve, miss→provider, GiveUp→stale-if-error)
- [x] [code] `configs/gameday.yaml`: tinh chỉnh khối `cache:`
- [x] [run] `python -m pytest -q tests/test_cache.py`
- [x] [run] `python scripts/run_gameday.py --scenarios configs/scenarios_cache.yaml --client resilient,cached --out reports/cache` (băng C1 đạt ngưỡng)

## Phase D — Workflow hoàn tiền exactly-once (12đ)
- [x] [code] `src/gameday_lab/workflows/executor.py`: idempotency key ổn định theo (session, step)
- [x] [code] `src/gameday_lab/workflows/executor.py`: checkpoint sau khi step thành công + resume sau crash
- [x] [run] `python -m pytest -q tests/test_executor.py`
- [x] [run] `python scripts/run_workflows.py --executor both`

## Phase E — Burn-rate alerting & error budget (12đ)
- [x] [code] `src/gameday_lab/slo/alerting.py`: burn-rate rules (14.4x/1h-5m, 6x/6h-30m, 1x/3d-6h) với prefix sum
- [x] [code] `src/gameday_lab/slo/alerting.py`: `error_budget_report()`
- [x] [run] `python -m pytest -q tests/test_alerting.py`
- [x] [run] `python scripts/run_slo.py --alerter both`

## Phase F — Report & postmortem (12đ)
- [x] [run] `python scripts/plot_timeseries.py --scenario G2_overload_storm` (cần `pip install -e ".[plot]"`)
- [x] [report] Copy `reports/REPORT_TEMPLATE.md` → `reports/REPORT.md`
- [x] [report] Điền mục 1-7 của report (bao gồm giải thích tinh chỉnh `client:`)
- [x] [report] Điền mục 8 (cache) — giải thích tinh chỉnh `cache:` bằng số liệu `wrong_answers_by_reason`
- [x] [report] Viết postmortem blameless cho băng G2 (client naive, dùng plot + CSV trong `reports/gameday/`)
- [x] [report] Thí nghiệm bật/tắt retry budget, giải thích kết quả (G3 retry_ratio)

## Phase Final — Grading & nộp bài
- [x] [run] `python scripts/grade.py` (tạo `reports/scoreboard.json`, mục tiêu 88/100 tự động)
- [x] [run] `ruff check src tests scripts` (lint sạch)
- [x] [report] Kiểm tra repo/zip đủ: code, `configs/gameday.yaml`, `reports/REPORT.md` (+hình), `reports/scoreboard.json`
