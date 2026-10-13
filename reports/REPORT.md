# Lab 25B — Report

**Họ tên / nhóm:** Vũ Việt Hoàng (2A202602398)  **Commit:** xem `git log -1`  **Điểm tự động (`make grade`):** 84 / 88 (4 điểm còn lại là dữ liệu ẩn của giảng viên, chưa chấm được ở máy local)

> Số liệu lấy từ `reports/gameday/*.json`, `reports/cache/*.json`, `reports/workflows.json`, `reports/slo.json` (seed 25). Các thí nghiệm ở mục 3 và 8 chạy lại cùng seed bằng cách ghi đè cấu hình trong bộ nhớ.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0,719 | 7,16 | 7,74 | 6,74 | 2,16 | 4,0 | FAIL |
| G1 | yours | 1,000 | 4,78 | 1,06 | 0,002 | 2,10 | 0,0 | PASS |
| G2 | naive | 0,456 | 15,03 | 9,90 | 8,90 | 4,63 | 155,0 | FAIL |
| G2 | yours | 1,000 | 11,22 | 1,04 | 0,005 | 2,13 | 0,0 | PASS |
| G3 | naive | 1,000 | 4,77 | 1,16 | 0,164 | 2,33 | 0,0 | FAIL (retry_ratio) |
| G3 | yours | 1,000 | 4,74 | 1,06 | 0,034 | 2,15 | 0,0 | PASS |
| G4 | naive | 1,000 | 13,76 | 1,00 | 0,004 | 2,01 | 0,0 | FAIL (p95) |
| G4 | yours | 1,000 | 5,68 | 1,01 | 0,002 | 2,07 | 0,0 | PASS |
| G5 | naive | 0,370 | 5,36 | 17,03 | 16,03 | 2,12 | – | FAIL |
| G5 | yours | 0,818 | 5,48 | 1,00 | 0,002 | 2,50 | – | PASS |
| G6 | naive | 0,517 | 4,84 | 12,34 | 11,34 | 2,01 | 0,0 | FAIL |
| G6 | yours | 1,000 | 5,96 | 1,05 | 0,020 | 2,22 | 0,0 | PASS |

Naive đạt 0/6 băng, client của tôi đạt 6/6 (cũng đạt 6/6 với seed 1, 2, 3).

## 2. Cấu hình client và lý do

Tôi giữ nguyên các giá trị mặc định của starter vì chúng đã đạt 6/6 băng trên 4 seed; dưới đây là lý do từng giá trị hợp lý, không phải chỉ "để nguyên".

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8,0 | Lượt user chờ tối đa 30 s với 2 call. Ở G6, `alpha-eu` treo; timeout 8 s fail over sau 8 s. Thử 30 s: availability G6 tụt 1,000 → 0,949 (thí nghiệm 2). |
| max_attempts_per_provider | 2 | 1 lần retry cùng provider rồi fail over. G3 (500 ngẫu nhiên 30%) cần 1 retry để che lỗi lẻ, nhưng retry nhiều hơn chỉ làm tăng tải khi provider quá tải (G2). |
| backoff_base_s / cap_s | 0,25 / 4,0 | Full jitter, trần 4 s để retry không ăn hết deadline 30 s. Có `retry_after` thì dùng `retry_after + jitter(0, base)`: G1 có 0 vi phạm `retry-after`. |
| retry_budget_ratio | 0,1 | Retry không vượt ~10% lượng request. G3 `retry_ratio` = 0,034 (ngưỡng ≤ 0,10). |
| breaker.window_size / min_calls | 20 / 10 | Cửa sổ đủ ngắn để bắt sự cố trong vài giây ở ~8 call/s, `min_calls` = 10 để 2 lỗi lẻ lúc ít traffic không làm trip breaker. |
| breaker.failure_rate_threshold | 0,5 | G3 lỗi 30% **không** trip (đúng: retry + budget xử lý), G2/G6 lỗi gần 100% trip ngay. |
| breaker.slow_call_s | 5,0 | G4 không có lỗi nào, chỉ chậm thêm 5 s. Nhờ slow-call, breaker trip và traffic chuyển sang `alpha-eu`: p95 13,76 → 5,68 s. |
| breaker.open_duration_s | 10,0 | Đủ ngắn để quay lại `alpha` sau sự cố (`primary_share_after_restore` ≥ 0,8 ở G2/G4), đủ dài để không dò liên tục provider đang chết. |
| breaker.half_open_max_probes | 2 | Chỉ 2 request thăm dò cùng lúc; phần còn lại fail fast, không dồn thundering herd vào provider đang hồi phục. |

Sơ đồ quyết định của `on_error`:

```
loại lỗi          ghi breaker?    retry cùng provider?                 failover?          cooldown?
CLIENT_ERROR      release         không                                không -> GiveUp    không
QUOTA_EXHAUSTED   release         không                                có, ngay           quota_cooldown_s (300 s)
RATE_LIMITED      release         không                                có, ngay           đến now + retry_after
OVERLOADED /
SERVER_ERROR /    on_failure      có nếu tries < 2 && breaker CLOSED   có, ngay, khi hết   không (breaker lo)
TIMEOUT                           && còn token budget && còn deadline  lượt retry
```

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối.** Cố ý làm breaker vô dụng (`min_calls: 1e9`) và cho `max_attempts_per_provider: 5`. G2: có budget thì `retry_ratio` = 0,058, amplification 1,38, availability 0,999; tắt budget thì `retry_ratio` = 0,495, amplification 1,68, availability 0,967, recovery 25 s. G3: `retry_ratio` 0,068 (có budget) so với 0,159 (tắt budget, vượt ngưỡng 0,10). Khi breaker hoạt động đúng thì nó cắt traffic tới provider chết nhanh hơn, nhưng breaker chỉ trip sau khi đã thu đủ `min_calls` kết quả và không bắt được lỗi rải rác (G3, 30% < 50%). Budget giới hạn cứng tổng lượng retry bất kể breaker cấu hình đúng hay sai, nên nên có cả hai.
2. **`attempt_timeout_s: 30`.** G4: p95 5,61 s (không đổi, vì G4 không có lỗi và đã có slow-call). G6: availability tụt từ 1,000 xuống 0,949. Một attempt vào `alpha-eu` đang treo giữ cả lượt 30 s, đúng bằng deadline, nên user đã bỏ đi khi client mới fail over. Timeout đúng phải dựa vào p99 của latency bình thường (vài giây) và ngân sách deadline còn lại, không dựa vào "càng dài càng chắc".
3. **Bỏ lọc tool calling khỏi `ladder()`.** G6: `incompatible_routes` = 235 (so với 0), availability 0,902 (so với 1,000). Nói với product manager: có 235 lần agent đưa yêu cầu "cần tra cứu đơn hàng/hoàn tiền" cho một model rẻ không biết dùng công cụ, nên khách nhận câu trả lời sai hoặc không làm được việc gì, dù hệ thống vẫn "trả lời". Với khách đó đây là lỗi nặng hơn là timeout.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1,0 | 81 | 128 | 81 | 0 | 1.574.200 ¢ | 1,123 |
| yours | 1,0 | 66 | 0 | 0 | 0 | 0 | 0,303 |

**Vì sao chỉ idempotency key là chưa đủ?** Key chặn hoàn tiền trùng khi gọi lại payment, nhưng nếu worker chết thì process mới không biết step nào đã xong: nó chạy lại từ đầu, gọi lại cả `classify_request`/`lookup_order` (tốn LLM, wasted LLM/crash cao) và phụ thuộc hoàn toàn vào phía payment nhớ key.
**Vì sao chỉ checkpoint là chưa đủ?** Giữa lúc payment đã chạy xong và lúc ta kịp ghi checkpoint có một khoảng trống (timeout mất response, hoặc worker chết ngay sau khi tiền đã đi). Không có key thì lần chạy lại sẽ hoàn tiền lần hai: naive có 128 hoàn tiền trùng. Cần cả hai: key đảm bảo gọi lại an toàn, checkpoint tránh làm lại việc đã xong.
**Nếu ghi checkpoint *trước* khi gọi payment?** Nếu worker chết sau khi ghi checkpoint nhưng trước khi payment chạy xong, worker mới thấy "step đã xong" và bỏ qua: khách không bao giờ nhận được tiền (hoàn tiền bị mất). Vì vậy checkpoint phải ghi **sau** khi step thành công.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (1 phút) | page | 27 phút |
| provider_outage | page | page (0 phút) | page | 4 phút |
| deploy_blip | none | **false page** | không báo | – |
| slow_leak | ticket | **bỏ sót** | ticket | 1337 phút |
| degraded_evening | page | page (214 phút) | page | 291 phút |
| night_spike | none | **false page** | không báo | – |
| small_bump | none | **false page** | không báo | – |

Naive: 3 false page, bỏ sót slow_leak. Burn-rate: 0 false page, 3 page, 1 ticket, đạt cả bốn nhóm.

Error budget 30 ngày: 25.901.755 request, cho phép 129.508,8 lỗi, thực tế 55.618 lỗi (availability 0,997853). Đã tiêu **42,95%**, còn **57,05%**, **không** đóng băng release. `small_bump` (20 phút ở 12% lỗi) tiêu khoảng 12%/0,5% = 24× tốc độ budget trong 20 phút, nhưng tổng chỉ là một phần rất nhỏ của budget cả tháng; tự hết, khách gần như không cảm nhận được, nên không đáng đánh thức on-call lúc 11 giờ trưa. Cảnh báo theo ngưỡng 5 phút của naive nhạy với blip ngắn mà không quan tâm đến tổng budget bị đốt.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình: `reports/gameday/G2_overload_storm.png`; CSV: `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt:** Từ t=120 s provider `alpha` mất 75% GPU trong 3 phút. Client v1 với 3 tầng retry chồng nhau, delay cố định, không breaker đã khuếch đại tải gấp ~9,9 lần, nên dù `alpha` hồi phục đúng lúc t=300 s hệ thống vẫn nghẹt thêm hơn 2 phút (recovery 155 s). Trong 10 phút thử nghiệm, chỉ 1.077 trên 2.362 lượt user (45,6%) thành công.

**Ảnh hưởng:** 1.285 lượt user thất bại (availability 0,456) trong khoảng t=120 → ~450 s, tức ~5,5 phút. Giả sử tháng chỉ có sự cố này với 4 lượt/s (~10,4 triệu lượt/tháng) thì budget 0,5% là ~51.800 lượt lỗi; sự cố tiêu ~2,5% budget tháng. (Sự cố thật 5 phút trên traffic thật có thể cao hơn nhiều.)

**Timeline** (giây ảo — số từ CSV, trung bình theo cửa sổ 30 s):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 0–120 | Bình thường | ~7,5 attempt/s, ~7,5 call hoàn tất/s, ok_ratio 1,00 |
| 120 | `alpha` mất 75% năng lực | attempts/s vọt 7,9 → 90,8; ok_ratio 0,52 |
| 150–270 | Retry storm | 140–156 attempt/s (gấp ~19× bình thường), ok_ratio 0,00; 26.298 lỗi `rate_limit_exceeded` và 3.194 `overloaded_error` |
| 300 | `alpha` hồi phục hoàn toàn | attempts/s hạ xuống 23, nhưng ok_ratio vẫn 0,00 |
| 360 | Đợt retry thứ hai | attempts/s tăng lại 113, ok_ratio vẫn 0,00: hàng đợi vẫn đầy |
| 450 | Bắt đầu hồi phục | ok_ratio 0,90, attempts/s 11,5 |
| 480+ | Hồi phục | ok_ratio 1,00, attempts/s 10,4 → 7,3 |

**Trigger vs root cause:** *Trigger* là mất 75% GPU của `alpha`. Đó chỉ là sự cố 3 phút. *Root cause* là chính sách retry của client: 3 tầng retry (app × gateway × SDK, tối đa 36 attempt mỗi call), delay cố định không jitter, không retry budget, không breaker, không fallback sang provider khác, nên mỗi lượt lỗi tạo ~10 attempt (amplification 9,9). Đây là metastable failure: lượng retry duy trì tải cao hơn năng lực ngay cả khi nguyên nhân gốc (GPU) đã hết.

**Vì sao không tự hồi phục sau t=300:** Server vẫn tiếp tục xử lý những request mà client đã timeout và bỏ, nên hàng đợi của `alpha` đầy các request "ma" không còn ai chờ. Mỗi client timeout lại sinh retry mới, giữ tải đầu vào cao hơn năng lực xử lý, hàng đợi không bao giờ vơi, request mới tiếp tục quá hạn (529 hoặc timeout) và sinh thêm retry. Vòng lặp tự duy trì này chỉ dừng khi các lượt user hết hạn (30 s deadline) và lượng retry tự giảm, nên mất ~150 s.

**Điều gì giúp / điều gì làm tệ hơn:** Giúp: deadline 30 s của user đặt giới hạn cho vòng retry; `alpha` hồi phục đúng hẹn. Làm tệ hơn: retry xếp chồng (nhân với nhau), delay cố định làm mọi client quay lại cùng lúc (không jitter), không phân biệt lỗi 429/529 nên không tôn trọng `retry-after`, không có fallback nên toàn bộ tải dồn vào một provider đang yếu.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Chỉ giữ 1 tầng retry; dùng full jitter + tôn trọng `retry-after`; retry budget 10% | prevent | Nhóm platform / Reliability | G2 trong CI: amplification ≤ 1,6 (hiện tại 1,04) |
| 2 | Thêm circuit breaker sliding-window và fallback ladder `alpha → alpha-eu → beta` | mitigate | Nhóm agent runtime | G2/G6: availability ≥ 95%, recovery ≤ 30 s |
| 3 | Alert burn-rate đa cửa sổ (14,4× 1h/5m) thay cho ngưỡng 5 phút | detect | SRE on-call | `make slo`: page đúng hạn, 0 false page |
| 4 | Dashboard hiển thị amplification (attempt/call) và `retry_ratio` theo provider; cảnh báo khi amplification > 1,5 trong 2 phút | detect | SRE on-call | Alert bắn trong ≤ 2 phút khi chạy lại băng G2 với client naive |

Postmortem mang tính blameless: không có cá nhân nào "sai"; thiết kế mặc định (3 tầng retry riêng biệt, ai cũng thêm một tầng cho an toàn) khiến lỗi khuếch đại, và quy trình review không bắt được điều này.

## 7. Một điểm yếu còn lại

G2 p95 vẫn cao nhất trong 6 băng (11,2 s so với 4–6 s các băng khác): trong 3 phút `alpha` quá tải, các call bị 529 chờ hàng đợi tới 10 s trước khi fail over. Client chỉ biết `alpha` kém sau khi thu đủ `min_calls` kết quả. Khi `beta` gánh tải thì cost/turn tăng (1,4 so với 1,0 mỗi call). Cải tiến: giảm `attempt_timeout_s` cho call đầu tiên khi `alpha` đang có tỷ lệ chậm cao, hoặc hedge (gửi song song sang `alpha-eu` nếu sau ~3 s chưa có phản hồi) với giới hạn bằng retry budget. Điểm yếu thứ hai: nếu cả 4 provider cùng chậm, client chỉ còn `GiveUp`.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0,837 | 0,842 | 0,0 | 0 | 0 | 0 | 1,386 | FAIL |
| cached (của tôi) | 0,912 | 0,9645 | 0,635 | 174 | 0 | 6 | 0,927 | PASS |

Starter (threshold 0,80, ttl 600, max_stale 0) cho `wrong_answers` = 181 (`other_question`), `outdated_answers` = 80, `faq_availability` = 0,859: FAIL cả 3 check.

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0,95 | Ở 0,80 có 142 câu trả nhầm (`other_question`: câu gần nghĩa nhưng khác ý). Ở 0,95 `wrong_answers` = 0 và hit ratio 0,635. 0,97 cũng 0 lỗi nhưng hit ratio hạ còn 0,55, sát ngưỡng 0,50 (rủi ro rớt trên băng ẩn). |
| ttl_s | 45 | Webhook chỉ xử lý được thay đổi có báo. Thay đổi bảng phí ship không báo trước chỉ bị TTL giới hạn: ttl 300 → `outdated_answers` = 95 (vượt ngưỡng 15); ttl 30 → 2, 45 → 6, 60 → 3. 45 giữ hit ratio cao (0,635) mà vẫn còn biên an toàn dưới 15. |
| max_stale_s | 900 | Phải lớn hơn thời gian sập của mọi provider (120 s trong băng C1; chọn 900 để dư biên cho băng ẩn kéo dài hơn). Với `max_stale_s: 0` `faq_availability` tụt 0,9645 → 0,863 và `stale_served` = 0. |

Thí nghiệm:

1. **Ngưỡng.** 0,80 / 0,95 / 0,97: hit ratio 0,63 / 0,635 / 0,55; `wrong_answers` 142 / 0 / 0; cost/turn 0,985 / 0,927 / 0,985. Ở 0,80 các câu hỏi gần nghĩa mà khác ý vượt ngưỡng nên bị trả nhầm câu trả lời của câu khác (`other_question`).
2. **TTL.** 30 / 45 / 60 / 300 s: `outdated_answers` 2 / 6 / 3 / 95; hit ratio 0,592 / 0,635 / 0,652 / 0,731. Webhook chỉ kích hoạt khi hệ thống biết tài liệu đổi; thay đổi không báo trước (bảng phí ship) không có tín hiệu nào, nên cách duy nhất là giới hạn tuổi của entry bằng TTL.
3. **Stale-if-error.** `max_stale_s: 0`: `faq_availability` 0,863, `stale_served` 0; với 900: 0,9645, `stale_served` 174. Trong 120 s mọi provider sập, các câu FAQ chỉ trả lời được nhờ câu trả lời cũ.

Trả lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả theo từng user?** Câu trả lời phụ thuộc vào dữ liệu thay đổi liên tục của chính user (trạng thái đơn, số tiền hoàn, điểm) nên bản cũ có thể sai ngay sau khi lưu; cache theo user còn kéo theo rủi ro lộ dữ liệu của user khác nếu khóa/scope có lỗi (`other_user`). Giá trị tiết kiệm thấp (mỗi user hỏi câu của riêng mình, ít lặp) trong khi rủi ro cao.
- **Khi mọi provider sập, dashboard vẫn xanh nhờ cache — alert theo số nào?** Theo `stale_served` (tỷ lệ câu trả lời stale / tổng), `cache_hit_ratio` giảm, tỷ lệ attempt lỗi và `GiveUp` ở tầng provider (breaker OPEN), và availability của các call không đi qua cache (tool call). Alert khi `stale_served` > 0 kéo dài vài phút hoặc breaker của tất cả provider OPEN, vì khi đó "availability" của người dùng không còn phản ánh sức khỏe thật của hệ thống.
