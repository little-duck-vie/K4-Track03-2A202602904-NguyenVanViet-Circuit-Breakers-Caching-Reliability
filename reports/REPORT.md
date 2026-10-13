# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Văn Việt — MSV 2A202602904  **Commit:**   **Điểm tự động:** 84 / 88 

> Số liệu lấy từ `reports/gameday/*.json` (seed 25), `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`.
> Starter `gameday.yaml` giữ nguyên phần `client:`; tôi kiểm tra lại trên 5 seed (1, 2, 3, 7, 25) — 6/6 băng PASS ở cả 5 seed.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
| ----- | ------ | -----------: | ------: | ------------: | ----------: | --------: | -----------: | :---: |
| G1    | naive  |       0.7194 |   7.157 |         7.738 |       6.738 |     2.163 |          4.0 |  ❌  |
| G1    | yours  |          1.0 |   4.776 |         1.062 |       0.002 |     2.101 |          0.0 |  ✅  |
| G2    | naive  |        0.456 |  15.034 |         9.904 |       8.904 |     4.627 |        155.0 |  ❌  |
| G2    | yours  |          1.0 |   10.92 |         1.041 |       0.007 |     2.119 |          0.0 |  ✅  |
| G3    | naive  |          1.0 |   4.768 |         1.164 |       0.164 |     2.328 |          0.0 |  ❌  |
| G3    | yours  |          1.0 |    4.66 |          1.06 |       0.034 |     2.145 |          0.0 |  ✅  |
| G4    | naive  |          1.0 |  13.764 |         1.004 |       0.004 |     2.008 |          0.0 |  ❌  |
| G4    | yours  |          1.0 |   5.562 |         1.015 |       0.002 |     2.072 |          0.0 |  ✅  |
| G5    | naive  |       0.3699 |   5.359 |         17.03 |       16.03 |     2.124 |           — |  ❌  |
| G5    | yours  |       0.8179 |   5.477 |         1.003 |       0.002 |     2.504 |           — |  ✅  |
| G6    | naive  |        0.517 |   4.836 |        12.344 |      11.344 |     2.009 |          0.0 |  ❌  |
| G6    | yours  |          1.0 |   5.961 |         1.055 |       0.019 |     2.189 |          0.0 |  ✅  |

Naive fail 6/6 băng, yours pass 6/6. G5: 0.8179 là trần thực tế — 60% request bị malformed trong 2 phút không client nào cứu được; yours không retry lỗi 400 (amplification 1.003) và chỉ 1 attempt sang `alpha-eu`.

## 2. Cấu hình client và lý do

| Tham số                        |                   Giá trị | Vì sao (dựa trên băng nào, số liệu nào)                                                                                                                                                                                                                 |
| ------------------------------- | --------------------------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| attempt_timeout_s               |                         8.0 | Thí nghiệm 2: nâng lên 30 làm G6 tụt availability 1.0 → 0.949 (109 lượt hết deadline vì treo trên`alpha-eu`). 8 s ≈ vài lần p99 bình thường (~1.2–1.8 s service) nhưng còn chừa chỗ cho 1–2 lần fail over trong deadline 30 s.       |
| max_attempts_per_provider       |                           2 | 1 retry rồi fail over. G3 (brown-out 30% lỗi 500) đạt retry_ratio 0.034 ≤ 0.10 và availability 1.0: lỗi độc lập nên 1 retry là đủ; nhiều hơn chỉ khuếch đại tải (thí nghiệm 1 dùng 5 → retry_ratio 0.495 ở G2 khi không có budget). |
| backoff_base_s / cap_s          |                  0.25 / 4.0 | Full jitter: trải các retry ra thay vì dồn cùng nhịp (nguyên nhân sóng retry của naive ở G2). Cap 4 s để delay không ăn hết deadline 30 s.                                                                                                      |
| retry_budget_ratio              |                         0.1 | Giới hạn retry ≲ 10% first-attempt dù sự cố nặng cỡ nào; thí nghiệm 1: tắt budget (khi breaker hỏng) G2 retry_ratio 0.058 → 0.495.                                                                                                                |
| breaker.window_size / min_calls |                     20 / 10 | Đủ nhạy (phát hiện trong ~10 call ≈ vài giây ở 8 call/s) mà không trip vì 2–3 lỗi lẻ. G5 (bad deploy 400) không kích hoạt breaker: chỉ 1 attempt sang`alpha-eu`.                                                                           |
| breaker.failure_rate_threshold  |                         0.5 | G3 chỉ có 30% lỗi → breaker không mở (retry + budget xử lý), tránh bỏ`alpha` khi nó vẫn phục vụ 70%. G2/G6 (≥ 75%/100% lỗi) mở nhanh.                                                                                                        |
| breaker.slow_call_s             |                         5.0 | G4:`alpha` chậm thêm 5 s **không có lỗi** — chỉ slow-call mới bắt được. Kết quả: p95 13.76 s (naive) → 5.56 s.                                                                                                                           |
| breaker.open_duration_s         |                          10 | G2: recovery 0 s và ≥ 80% traffic quay lại`alpha`. 10 s đủ để provider thoát hàng đợi, không đợi quá lâu mới thử lại.                                                                                                                      |
| breaker.half_open_max_probes    | 2 (+ half_open_successes 2) | Chỉ 2 request thăm dò → không thundering herd khi provider vừa hồi phục (đúng cơ chế metastable ở mục 6).                                                                                                                                         |

Sơ đồ quyết định của `on_error`:

```
loại lỗi          ghi breaker?   retry cùng provider?                 failover?   cooldown?
CLIENT_ERROR      release        không                                 không       -> GiveUp("client_error")
QUOTA_EXHAUSTED   release        không                                 ngay        quota_cooldown_s (300 s)
RATE_LIMITED      release        không                                 ngay        retry_after (>= 1 s nếu thiếu header)
OVERLOADED        on_failure     có nếu tries<2 & budget & còn hạn     khi hết     không
SERVER_ERROR      on_failure     (như trên)                            khi hết     không
TIMEOUT           on_failure     (như trên)                            khi hết     không
Hết ladder -> chờ cooldown rate-limit ngắn (<= 5 s, breaker không OPEN, còn deadline) hoặc GiveUp.
```

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối** (breaker `min_calls: 1e9`, `max_attempts_per_provider: 5`).

   | Băng | Budget | availability | amplification | retry_ratio | recovery (s) |
   | ----- | ------ | -----------: | ------------: | ----------: | -----------: |
   | G2    | bật   |       0.9987 |          1.38 |       0.058 |            0 |
   | G2    | tắt   |       0.9666 |         1.675 |       0.495 |           25 |
   | G3    | bật   |          1.0 |         1.136 |       0.068 |            0 |
   | G3    | tắt   |          1.0 |         1.162 |       0.159 |            0 |

   Khi breaker bị vô hiệu, chỉ budget còn ngăn retry storm: ở G2 retry_ratio tăng gấp ~8,5 lần và 79 lượt hết deadline. Ở G3 (lỗi độc lập 30%) khác biệt nhỏ hơn vì retry thật sự có ích. Vẫn cần cả hai: breaker phản ứng theo *sức khỏe provider* và fail-fast; budget giới hạn *tổng lượng retry theo traffic* kể cả khi breaker cấu hình sai, bị trễ, hoặc lỗi không rơi vào tiêu chí của breaker.
2. **`attempt_timeout_s: 30`.**

   | Băng | timeout | availability | p95 (s) | give_up_reasons                        |
   | ----- | ------: | -----------: | ------: | -------------------------------------- |
   | G6    |       8 |          1.0 |   5.961 | —                                     |
   | G6    |      30 |       0.9488 |   5.315 | deadline 109, no_provider_available 13 |
   | G4    |       8 |          1.0 |   5.562 | —                                     |
   | G4    |      30 |          1.0 |   5.608 | —                                     |

   p95 chỉ tính trên lượt thành công nên *giảm* (5.96 → 5.32) trong khi availability tụt 5 điểm — survivorship bias: những lượt treo 30 s đã chết và bị loại khỏi p95. G4 gần như không đổi vì provider chậm nhưng vẫn trả lời (5 s < 8 s) — timeout không là cơ chế chặn ở đó, breaker slow-call mới là. Timeout nên bám vào p99 lúc bình thường × hệ số và quan trọng hơn là **thời gian cần cho một lần fail over trong deadline của user**, không phải deadline toàn bộ.
3. **Bỏ lọc tool calling khỏi `ladder()`** (G6): `incompatible_routes` = **234**, availability 1.0 → 0.9019, toàn bộ 234 lượt `give_up = incompatible_model_output`. Với PM: 234 lần hệ thống đưa yêu cầu "hoàn tiền" cho một model không biết gọi công cụ, nên nó trả lời mà không làm gì — khách tưởng đã xong nhưng không có hoàn tiền. Rẻ hơn (0.2/call) nhưng sai; thà fail over sang `beta` (đắt hơn 1.4) còn hơn.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds |    overpaid | wasted LLM / crash |
| -------- | ---------: | ------: | ----------------: | ---------------: | -----------: | ----------: | -----------------: |
| naive    |        1.0 |      81 |               128 |               81 |            0 | 1,496,700¢ |              1.123 |
| yours    |        1.0 |      66 |                 0 |                0 |            0 |           0 |              0.303 |

- **Chỉ idempotency key là chưa đủ**: sau crash, worker mới không biết step nào đã chạy nên chạy lại `classify`/`lookup`/`summarize` (tốn LLM), và nếu key sinh theo thời điểm/instance thì lại hoàn tiền trùng. Key đảm bảo *không lặp tác dụng phụ* nhưng không tránh lặp công việc.
- **Chỉ checkpoint là chưa đủ**: nếu crash/timeout *sau khi* payment commit nhưng *trước khi* ghi checkpoint (hoặc response bị mất — `ServiceTimeout`), worker mới sẽ gọi lại; không có key thì tiền đi lần 2. Key ổn định `session:step` làm lần gọi lại trả kết quả đã lưu của provider.
- **Ghi checkpoint trước khi gọi payment**: crash giữa chừng → worker mới thấy "đã làm" và bỏ qua → **hoàn tiền bị mất** mà không ai biết. Vì vậy thứ tự là: chạy step → thành công → mới ghi checkpoint.

## 5. Alerting & error budget

| Sự cố          | Mong đợi | Naive alerter        | Burn-rate alerter | Độ trễ phát hiện (burn-rate) |
| ---------------- | ---------- | -------------------- | ----------------- | --------------------------------: |
| checkout_bug     | page       | page (delay 1)       | page              |                          27 phút |
| provider_outage  | page       | page (delay 0)       | page              |                           4 phút |
| deploy_blip      | none       | **page (sai)** | không alert      |                                — |
| slow_leak        | ticket     | không phát hiện   | ticket            |                        1337 phút |
| degraded_evening | page       | page (delay 214)     | page              |                         291 phút |
| night_spike      | none       | **page (sai)** | không alert      |                                — |
| small_bump       | none       | **page (sai)** | không alert      |                                — |

Naive: 7 page, **3 false page**, bỏ sót slow_leak. Burn-rate: 3 page, 1 ticket, **0 false page**. Đổi lại, burn-rate chậm hơn ở `checkout_bug` (27 vs 1 phút) và `degraded_evening` (291 vs 214 phút) — đây là cái giá của việc không đánh thức người vì blip.

Error budget 30 ngày: requests 25,901,755; allowed_bad 129,508.8; bad 55,618; availability 0.997853 → đã tiêu **42.95%**, còn **57.05%**. `freeze_releases = False` — không đóng băng release. `small_bump` (20 phút ở 12% lỗi) chỉ tiêu một phần nhỏ budget và không thể cạn budget trong vài giờ (burn rate chưa tới ngưỡng 6× ở cửa sổ 6 giờ); không có hành động nào cần làm lúc 11 giờ trưa — xử lý trong giờ làm việc.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình: `reports/gameday/G2_overload_storm.png`; CSV: `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt:** `alpha` mất 75% GPU lúc t=120 và hồi phục lúc t=300 nhưng hệ thống vẫn không phục vụ được tới ~t=455 (recovery 155 s). Cấu hình retry 3 tầng không jitter, không budget, không breaker khuếch đại tải gấp ~9,9 lần và giữ hàng đợi `alpha` luôn đầy. Chỉ 45,6% lượt user thành công.

**Ảnh hưởng:** 2,362 lượt, 1,285 thất bại (54,4%). Giả sử tháng chỉ có sự cố này và lưu lượng 4 lượt/s (~10,4M lượt/tháng, allowed bad 0,5% ≈ 51,840 lượt): tiêu ~2,5% budget — nhỏ về tuyệt đối vì tape ngắn, nhưng nếu giữ tỷ lệ lỗi 54% trong cửa sổ 10 phút thì burn rate = 0,544/0,005 ≈ **109×**, vượt xa ngưỡng page 14,4×.

**Timeline** (từ CSV, attempts/s tổng các provider; success ratio là 10 s trượt):

|        t | Sự kiện                                      | Bằng chứng (số)                                                           |
| -------: | ---------------------------------------------- | ---------------------------------------------------------------------------- |
|      100 | Bình thường                                 | ~10 attempts/s; success 1.00                                                 |
|      120 | `alpha` mất 75% năng lực                  | 5 attempts/s, 7/7 call OK (hàng đợi chưa đầy)                          |
|      140 | Retry storm bắt đầu                         | **216 attempts/s** (~25× bình thường); 0 call thành công         |
| 160–280 | Metastable: tải gửi vào luôn ≫ năng lực | 105–182 attempts/s; success ratio 0.00                                      |
|      300 | `alpha` hồi phục hoàn toàn               | vẫn 115 attempts/s; 0 call thành công                                     |
| 320–440 | Hệ thống vẫn chưa hồi phục — vì sao?   | 19–107 attempts/s, ratio 0.00; phải xả hết hàng đợi + retry còn lại |
|      460 | Hồi phục                                     | 14 attempts/s, ratio 0.75 (t=480: 1.00); recovery_s = 155                    |

**Trigger vs root cause:** *Trigger* là mất 75% GPU của `alpha`. *Root cause* là cấu hình retry: 3 tầng chồng (app × gateway × SDK → tối đa 36 attempt/call), delay cố định (không jitter nên các client đồng bộ), không retry budget, không breaker, không fallback. Mỗi call thất bại sinh thêm nhiều attempt vào chính provider đang quá tải → hàng đợi dài hơn → thêm timeout → thêm retry: vòng lặp tự duy trì (metastable failure).

**Vì sao không tự hồi phục sau t=300:** năng lực `alpha` đã đủ cho traffic gốc (~8–10 call/s), nhưng hàng đợi đầy các request mà client đã timeout bỏ đi (server vẫn xử lý — công sức phí). Retry của client giữ tải đầu vào ≫ năng lực (115 attempts/s ngay t=300), nên hàng đợi không bao giờ rỗng cho tới khi các call cũ hết deadline (30 s) và tải retry giảm tự nhiên.

**Điều gì giúp / làm tệ hơn:** Giúp: server vẫn trả 529 nhanh khi hàng đợi > 10 s (không treo vô hạn); deadline 30 s của user cuối cùng cắt đuôi retry. Tệ hơn: retry không jitter đồng bộ; 3 tầng nhân tải; không có fail over nên `alpha-eu`/`beta` nhàn rỗi trong khi `alpha` chết.

**Action items:**

| # | Hành động                                                                   | Loại    | Owner                 | Cách kiểm chứng                                                                               |
| - | ------------------------------------------------------------------------------ | -------- | --------------------- | ------------------------------------------------------------------------------------------------ |
| 1 | Chỉ còn 1 tầng retry (ở`ResilientClient`); tắt retry ở gateway và SDK | prevent  | Platform/Gateway team | CI chạy G2:`amplification ≤ 1.2` (hiện 1.041)                                               |
| 2 | Retry budget 10% + full jitter + tôn trọng`retry-after`                    | prevent  | Reliability/SDK owner | Unit test`RetryBudget`; G1–G3: `retry_ratio ≤ 0.10`, 0 `retry_after_violations`          |
| 3 | Circuit breaker sliding-window + fallback ladder có kiểm tra tool-calling    | mitigate | Reliability owner     | G2:`recovery_s ≤ 30`, `primary_share_after_restore ≥ 0.8`; G6: `incompatible_routes = 0` |
| 4 | Page theo burn-rate đa cửa sổ (14.4× 1h/5m) thay vì ngưỡng 5 phút      | detect   | SRE on-call           | `make slo`: 0 false page, page `provider_outage` trong ≤ 5 phút                            |
| 5 | Dashboard hiển thị attempts/s so với calls/s (tỷ lệ khuếch đại)        | detect   | Observability         | Alert khi`attempts/calls > 2` trong 1 phút; diễn tập G2 hàng quý                          |

## 7. Một điểm yếu còn lại

G2 p95 vẫn 10.9 s (so với ~4.7 s các băng khác): trong lúc `alpha` còn đang nghẹt và breaker chưa mở, một số call chờ hàng đợi tới 8 s trước khi timeout rồi mới fail over. Có thể sửa bằng *hedged request* có budget (gửi bản sao sang `alpha-eu` khi latency vượt p95) hoặc giảm `slow_call_s` có điều kiện theo tải; cái giá là thêm chi phí và cần chứng minh bằng G2/G4 để tránh tự tạo khuếch đại. Điểm yếu thứ hai: `quota_cooldown_s` cố định 300 s — nếu quota được nạp lại sớm, ta vẫn tránh provider đó vô ích; cần probe nhẹ định kỳ.

## 8. Cache (Part G)

| Client                   | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
| ------------------------ | -----------: | ---------------: | --------------: | -----------: | ------------: | ---------------: | --------: | :---: |
| resilient (không cache) |       0.8373 |           0.8424 |             0.0 |            0 |             0 |                0 |     1.386 |  ❌  |
| cached (của bạn)       |       0.9124 |           0.9645 |           0.592 |          192 |             0 |                2 |     0.955 |  ✅  |

| Tham số`cache:`   | Giá trị | Vì sao (số liệu nào)                                                                                                                                                                                                                          |
| -------------------- | --------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| similarity_threshold |      0.93 | Starter 0.80 →**182 `wrong_answers`** (`other_question`). Quét 0.90–0.95 đều cho 0 wrong và hit ratio như nhau; 0.97 bắt đầu mất hit (hit 0.495 < 0.5 ở ttl 30). Chọn giữa vùng ổn định để chịu được băng ẩn. |
| ttl_s                |        30 | `outdated_answers` do thay đổi **không báo trước** (shipping) chỉ TTL giới hạn được. ttl 300 → 95 outdated; ttl 40 → tối đa 19 (> 15) trên 12 seed; ttl 30 → tối đa 10, hit ≥ 0.55; ttl 15 → hit 0.49 (< 0.5).      |
| max_stale_s          |       300 | Sự cố 120 s;`max_stale_s: 0` → faq_availability 0.8529 (chỉ 0.9645 khi có stale-if-error). 300 ≈ 2,5× độ dài sự cố mà vẫn bị ràng buộc bởi prompt_version/invalidation.                                                       |

Thí nghiệm:

1. **Ngưỡng.** 0.80: hit 0.587, `wrong_answers` 182 (`other_question`), faq_availability 0.8587 — các câu hỏi "gần mà khác" (vd. chính sách hoàn tiền vs thời hạn hoàn tiền) bị gộp, và lượt bị đánh dấu sai tính như thất bại. 0.93: hit 0.592, wrong 0, faq_avail 0.9645. 0.97: hit 0.495, wrong 0, cost/turn 1.02 (tăng vì miss nhiều hơn). Ngưỡng cao hơn không làm sai thêm nhưng tốn tiền và mất hit.
2. **TTL.** 30: outdated 2, hit 0.592; 60: outdated 3, hit 0.652; 300: **outdated 95**, hit 0.731. Webhook `on_content_update` chỉ có tác dụng với thay đổi *có báo trước* (policy/refund tại t=150 → 0 `ignored_invalidation`); thay đổi shipping lúc t=480 không có webhook nên cache không thể biết — chỉ tuổi tối đa của entry (TTL) giới hạn mức độ lỗi thời.
3. **Stale-if-error.** `max_stale_s: 0`: faq_availability **0.8529** (so với 0.9645), stale_served 0, give_up `no_provider_available` 438 (so với 246). Stale-if-error cứu ~11 điểm availability FAQ trong sự cố 120 s.

Trả lời:

- **Vì sao câu hỏi cá nhân không được cache, kể cả theo từng user:** câu trả lời phụ thuộc trạng thái thay đổi liên tục (đơn hàng đang ở đâu, tiền hoàn, điểm) — không có webhook theo user/đơn hàng, nên TTL hay stale đều có thể trả thông tin sai (và nhạy cảm). Cache theo user_id vẫn dễ lỗi do key/embedding gộp nhầm giữa người dùng hay phiên bản dữ liệu; lợi ích (hit) nhỏ vì mỗi user hiếm khi hỏi lại đúng câu đó. Rủi ro lộ dữ liệu người khác (`other_user`) nghiêm trọng hơn lợi ích.
- **Alert khi dashboard xanh nhờ cache:** dùng số *không* bị cache che: `stale_served` / tỷ lệ stale trên tổng FAQ, và **provider-side availability** (tỷ lệ call tới provider thành công, `no_provider_available`/giây, breaker OPEN count) tách khỏi availability của user. Page khi `breaker OPEN` cho mọi provider hoặc `stale_served` vượt ngưỡng (vd. > 10% trong 5 phút); đồng thời theo dõi tuổi lớn nhất của câu trả lời stale để biết còn bao lâu trước khi hết `max_stale_s`.
