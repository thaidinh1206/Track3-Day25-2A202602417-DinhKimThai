# Lab 25B — Report

**Họ tên / nhóm:** Đinh Kim Thái (2A202602417)  **Commit:** 521113b  **Điểm tự động (`make grade`):** 84.0 / 88 (Đạt điểm tuyệt đối trên bộ dữ liệu public, 4 điểm còn lại thuộc tập dữ liệu ẩn của giảng viên)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.911 | 1.059 | 0.008 | 2.161 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.738 | 1.058 | 0.034 | 2.141 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | **PASS** |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0s | Băng G4 & G6: Tránh để attempt treo tới deadline turn (30s) làm trễ failover. Với timeout 8.0s, p95 ở G4 chỉ đạt 5.562s (ngưỡng ≤ 10s) thay vì 13.764s như client naive. |
| max_attempts_per_provider | 2 | Băng G2 & G3: Giới hạn tối đa 1 lần retry trên cùng một provider (`tries < 2`), sau đó lập tức failover. Ngăn chặn khuếch đại retry storm (amplification giữ ở 1.059 ở G2 so với 9.904 của naive). |
| backoff_base_s / cap_s | 0.25s / 4.0s | Băng G1 & G2: Kết hợp Full Jitter giúp giải tỏa xung đột thundering herd; cap 4.0s vừa đủ để provider xả tải mà không tiêu tốn hết ngân sách 30s của user turn. |
| retry_budget_ratio | 0.1 | Băng G3: Khống chế số lượng retry không vượt quá 10% tổng request, giữ `retry_ratio = 0.034` (đạt yêu cầu ≤ 0.10) trong khi naive bị vượt ngưỡng lên 0.164. |
| breaker.window_size / min_calls | 20 / 10 | Băng G2 & G5: Cửa sổ 20 request với tối thiểu 10 request giúp tránh ngắt nhầm (false trip) do vài lỗi ngẫu nhiên ban đêm, đồng thời đủ nhạy để trip trong vòng 3–5 giây khi có sự cố thực sự. |
| breaker.failure_rate_threshold | 0.5 | Băng G2: Khi 50% call thất bại (529/500/timeout), provider đang bị quá tải nghiêm trọng, cần trip ngay sang OPEN để chuyển hướng traffic sang `alpha-eu`. |
| breaker.slow_call_s | 5.0s | Băng G4: Độ trễ bình thường ~1.2s; khi mạng chậm thêm +5s (tổng ~6.2s ≥ 5.0s), breaker coi đây là tín hiệu suy thoái và kích hoạt trip dù không có lỗi HTTP nào trả về. |
| breaker.open_duration_s | 10.0s | Băng G2: Thời gian chờ 10s đủ dài để hàng đợi của provider giải tỏa tải nghẽn, nhưng không quá lâu để kịp thời đưa traffic quay lại primary sau khi hồi phục (`primary_share = 1.0`). |
| breaker.half_open_max_probes | 2 | Tránh bão probe (thundering herd): Chỉ cho phép tối đa 2 request thăm dò đồng thời khi breaker chuyển sang HALF-OPEN. |

Sơ đồ quyết định của `on_error`:

```
                           +-------------------------+
                           |  Nhận ProviderError     |
                           +------------+------------+
                                        |
                                [ classify(err) ]
                                        |
         +------------------+-----------+-----------+-----------------------+
         |                  |                       |                       |
   CLIENT_ERROR      QUOTA_EXHAUSTED           RATE_LIMITED         OVERLOADED / 5xx / TIMEOUT
         |                  |                       |                       |
  release(permit)    release(permit)         release(permit)         breaker.on_failure()
         |                  |                       |                       |
     GiveUp()        cooldown = now+300s     cooldown = now+retry_after  tries < 2 AND budget ok
   (không retry,            |                       |                    AND deadline ok?
    không failover)     _route() (failover)     _route() (failover)         /          \
                                                                        [CÓ]           [KHÔNG]
                                                                         /                \
                                                             Retry cùng provider    _route() (failover)
                                                             với backoff delay      ngay lập tức
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   Khi vô hiệu hóa circuit breaker (`min_calls: 1_000_000_000`) và tăng `max_attempts_per_provider: 5`:
   - Ở kịch bản **(a) Giữ retry budget**: Trên G2 amplification chỉ là **1.382**, `retry_ratio = 0.058`, `recovery_s = 0.0s`, availability đạt **99.96%**. Trên G3 `amplification = 1.135`, `retry_ratio = 0.067`.
   - Ở kịch bản **(b) Tắt retry budget**: Trên G2 amplification tăng vọt lên **1.672**, `retry_ratio = 0.491`, `recovery_s = 30.0s`, availability tụt xuống **96.49%**. Trên G3 `retry_ratio = 0.162` (vi phạm ngưỡng 0.10).
   *Giải thích cơ chế:* Circuit breaker bảo vệ theo từng provider cục bộ dựa trên tỷ lệ lỗi trong quá khứ. Nếu breaker cấu hình sai ngưỡng hoặc lỗi xuất hiện rải rác dưới ngưỡng trip, Retry Budget đóng vai trò chốt chặn toàn cục khống chế tỷ lệ retry $\le 10\%$ traffic. Cần cả hai vì Breaker cắt đứt lưu lượng đến node chết, còn Budget ngăn chặn toàn bộ hệ thống tự nuốt chửng tài nguyên của chính mình.

2. **`attempt_timeout_s: 30`:**
   Khi nâng attempt timeout lên 30.0s (bằng toàn bộ ngân sách turn deadline):
   - Trên băng G4 (slow region): p95 đạt **5.683s** (vẫn kịp failover nhờ slow-call breaker trip ở 5.0s).
   - Trên băng G6 (provider outage): Availability giảm xuống còn **94.88%** (FAIL vì không đạt yêu cầu $\ge 95\%$), do các attempt gửi sang `alpha-eu` bị treo suốt 30s khiến user turn hết sạch deadline trước khi kịp failover sang provider B.
   *Giải thích cơ chế:* Timeout của một attempt phải dựa trên SLA xử lý p99 của provider bình thường (thường là 3–5 lần service median, ở đây là $1.2s \times 3 \approx 3.6 - 8.0s$), và phải để lại đủ thời gian cho ít nhất 1–2 lần failover trong turn deadline 30s.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   Khi cho phép request tool gọi sang cả `gamma-mini` (model C-mini không hỗ trợ tool calling):
   - Trên băng G6, số lượng `incompatible_routes` tăng vọt từ **0 lên 256 lỗi**, availability sụt từ **100% xuống 89.27%**.
   - *Ngôn ngữ Product Manager:* "Khi hệ thống chính gặp sự cố, chúng ta đã cố gắng định tuyến 256 khách hàng đang yêu cầu hoàn tiền tự động sang một mô hình giá rẻ không có khả năng kích hoạt công cụ tài chính. Hậu quả là 256 khách hàng này nhận được câu trả lời vô nghĩa hoặc báo lỗi hệ thống, khiến tỷ lệ hoàn tất giao dịch sụt giảm hơn 10% và tăng chi phí hỗ trợ thủ công."

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,574,200 đ | 1.123 |
| yours | 1.0 | 66 | **0** | **0** | **0** | **0 đ** | **0.303** |

*Trả lời:*
- **Vì sao chỉ idempotency key là chưa đủ?** Idempotency key chỉ bảo vệ tại biên của dịch vụ bên thứ ba (payment/email). Nếu không có checkpoint lưu trạng thái cục bộ, khi worker bị crash sau bước thanh toán thành công, worker mới khởi động lại sẽ phải chạy lại từ đầu tất cả các bước trước đó, gây lãng phí gọi LLM (`wasted_llm_per_crash > 1.0`).
- **Vì sao chỉ checkpoint là chưa đủ?** Nếu mạng bị timeout (`ServiceTimeout`) sau khi payment gateway đã trừ tiền nhưng response chưa kịp về worker, worker không thể lưu checkpoint. Khi thử lại (retry), nếu không có idempotency key, cổng thanh toán sẽ trừ tiền lần 2 (`duplicate refunds`).
- **Điều gì xảy ra nếu ghi checkpoint trước khi gọi payment?** Nếu ghi checkpoint hoàn tiền thành công *trước* khi gọi lệnh payment thực tế, nếu lệnh payment sau đó gặp lỗi hoặc worker chết ngay trước khi gửi request, worker phục hồi sẽ tưởng rằng bước thanh toán đã xong và chuyển sang gửi email, dẫn đến **mất hoàn tiền** (`lost_refunds > 0`), khách hàng nhận email xác nhận nhưng tiền không bao giờ về tài khoản.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (delay 1m) | page | 27m |
| provider_outage | page | page (delay 0m) | page | 4m |
| deploy_blip | none | **page (false)** | none | Không kích hoạt |
| slow_leak | ticket | none (bỏ sót) | ticket | 1337m (~22.3h) |
| degraded_evening | page | page (delay 214m) | page | 291m |
| night_spike | none | **page (false)** | none | Không kích hoạt |
| small_bump | none | **page (false)** | none | Không kích hoạt |

*Error budget 30 ngày:*
- Tổng request: 25,901,755; Lỗi cho phép (SLO 99.5%): 129,508.8; Lỗi thực tế: 55,618.
- Độ khả dụng thực tế: **99.7853%**; Ngân sách lỗi đã tiêu: **42.95%**, còn lại: **57.05%**.
- Quyết định: **Không đóng băng release** (`freeze_releases: False`) vì còn hơn 57% ngân sách lỗi.
- *Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?* Vì tổng lượng ngân sách lỗi bị tiêu hao trong 20 phút này là rất nhỏ so với toàn bộ tháng (chỉ khoảng 0.5% budget). Một sự cố ngắn tự phục hồi không đe dọa cam kết SLO 30 ngày thì không được phép gây mệt mỏi cảnh báo (alert fatigue) và đánh thức kỹ sư trực ca; một thông báo mức ticket hoặc theo dõi qua dashboard là phù hợp.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Đồ thị trực quan hóa sự cố:*

![G2 Overload Storm](gameday/G2_overload_storm.png)
*(Hình ảnh được sinh từ `python scripts/plot_timeseries.py --scenario G2_overload_storm` và lưu tại `reports/gameday/G2_overload_storm.png`)*

**Tóm tắt (3 câu):**
Vào thời điểm t=120s, cụm model primary `alpha` bị suy giảm 75% năng lực GPU trong 3 phút, làm bùng phát tỷ lệ lỗi 529 và đẩy hàng đợi xử lý lên mức nghẽn nghiêm trọng. Do client naive v1 sử dụng cơ chế retry 3 tầng chồng chéo không có jitter, không circuit breaker và không giới hạn ngân sách retry, lưu lượng request tới `alpha` đã bị khuếch đại gấp 10 lần (lên tới 254 req/s). Sự cố chuyển hóa thành lỗi siêu bền vững (metastable failure), khiến hệ thống không thể tự phục hồi và tiếp tục sập hoàn toàn thêm 155 giây sau khi hạ tầng GPU của provider đã được khôi phục 100%.

**Ảnh hưởng:**
- Tổng cộng 1,285 lượt user thất bại (tỷ lệ khả dụng sụt xuống chỉ còn 45.60%).
- Sự cố kéo dài 335 giây (180 giây suy thoái GPU + 155 giây phục hồi trễ do bão retry).
- Tiêu tốn 9.904 attempts trên mỗi call, vượt quá 100% ngân sách lỗi khả dụng của tháng cho riêng dịch vụ này.

**Timeline (giây ảo — trích xuất từ `reports/gameday/naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% năng lực GPU | Traffic bình thường ~9 req/s; calls_ok = 7; sau t=120 attempts bắt đầu dồn ứ. |
| 136 | Bão retry bắt đầu bùng nổ | attempts_alpha tăng vọt lên 53 req/s; calls_ok giảm về 3 req/s. |
| 144 | Đỉnh điểm của bão retry (Metastable state) | attempts_alpha đạt đỉnh **254 req/s** (khuếch đại > 25 lần); calls_ok = 0. |
| 200 | Hệ thống tê liệt hoàn toàn | attempts_alpha duy trì 135 req/s; calls_ok = 0; toàn bộ request bị 529 hoặc timeout. |
| 300 | alpha hồi phục hoàn toàn năng lực GPU | Hạ tầng GPU hồi phục nhưng attempts_alpha vẫn ở mức 115 req/s; calls_ok vẫn bằng 0. |
| 350 | Hàng đợi vẫn nghẽn vì request rác | attempts_alpha vẫn là 15–30 req/s, server bận xử lý các request mà client đã timeout từ trước; calls_ok = 0. |
| 455 | Hệ thống bắt đầu phục hồi thực sự | Hàng đợi cạn dần, attempts giảm về 7 req/s, calls_ok hồi phục lên 4 req/s (recovery = 155s sau t=300). |

**Trigger vs root cause:**
- **Trigger:** Provider `alpha` mất 75% năng lực tính toán GPU trong khoảng t=120s – 300s.
- **Root cause:** Lớp client ứng dụng triển khai 3 tầng retry chồng nhau (App × Gateway × SDK = 36 attempts/call), sử dụng khoảng trễ cố định (fixed delay), không có randomized full jitter, không có Circuit Breaker ngắt mạch và không có Retry Budget. Khi server chậm lại, các retry liên tục dội ngược vào hàng đợi, biến sự cố tạm thời thành **metastable failure** — một vòng lặp tự duy trì: hàng đợi đầy $\rightarrow$ client timeout $\rightarrow$ client retry $\rightarrow$ hàng đợi càng đầy hơn bằng các request vô giá trị mà client đã bỏ qua.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Client tiếp tục gửi request vào provider đang quá tải mà không có backoff ngẫu nhiên; thời gian timeout dài khiến client giữ kết nối vô ích; thiếu failover sang vùng dự phòng `alpha-eu`.
- *Giúp ích:* Cơ chế Circuit Breaker trượt nhanh chóng phát hiện failure rate $\ge 50\%$ để ngắt kết nối sang OPEN, chuyển hướng tải sang `alpha-eu` và `beta`, kết hợp Retry Budget giới hạn retry $\le 10\%$.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai ResilientClient với Circuit Breaker sliding window và fallback ladder tương thích | Mitigate | Core Team | Chạy CI Game Day G2 đạt recovery $\le 30s$ và amplification $\le 1.6$. |
| 2 | Áp dụng Full Jitter Exponential Backoff và Retry Budget token-bucket tối đa 10% | Prevent | SDK Team | Đo kiểm unit test `test_budget_caps_retries_to_a_ratio_of_requests` và G3 retry ratio $\le 0.10$. |
| 3 | Xây dựng hệ thống cảnh báo đa cửa sổ Multi-window Burn-rate thay thế ngưỡng cố định 5 phút | Detect | SRE Team | Cảnh báo page chính xác trong 30 phút cho sự cố lớn, 0 false page trong 30 ngày dữ liệu SLI. |

---

## 7. Một điểm yếu còn lại

*Điểm yếu:*
Khi xảy ra sự cố quá tải diện rộng hoặc toàn bộ các provider trong cùng vùng địa lý đều gặp sự cố (như G6 khi cả `alpha` và `alpha-eu` đều sập), toàn bộ lưu lượng fallback đổ dồn về provider `beta`. `beta` có chi phí cao hơn đáng kể (cost factor 1.4 so với 1.0) và giới hạn rate limit thấp hơn (16 rps). Nếu đợt bùng phát lưu lượng kéo dài nhiều giờ, chi phí vận hành sẽ tăng vọt và `beta` cũng sẽ có nguy cơ bị sập dây chuyền theo kiểu domino.

*Cách khắc phục:*
Áp dụng **Load shedding có phân cấp ưu tiên** (Request Prioritization) kết hợp với **Cost-aware Adaptive Rate Limiting**: Trong thời gian failover sang provider đắt tiền, chỉ ưu tiên các luồng giao dịch thanh toán quan trọng, tạm thời hạ cấp hoặc trả về câu trả lời tĩnh cho các tác vụ không khẩn cấp; đồng thời mở rộng thêm các provider thứ ba độc lập khác vào fallback ladder.

---

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9124** | **0.9645** | **0.652** | **157** | **0** | **3** | **0.915** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ở ngưỡng 0.80, hệ thống trả sai 157 câu (`wrong_answers = 157` do nhầm lẫn giữa các câu hỏi FAQ tương tự). Nâng lên 0.92 giúp triệt tiêu hoàn toàn lỗi (`wrong_answers = 0`) mà vẫn duy trì hit ratio cao (65.2%). |
| ttl_s | 60.0s | Ở TTL = 300s, khi biểu phí ship đổi ngầm không có webhook ở t=480s, số câu trả lời cũ bị lỗi thời vọt lên 95 câu. Giảm TTL xuống 60s giúp câu trả lời cũ hết hạn kịp thời, giảm `outdated_answers` xuống chỉ còn 3 (đáp ứng điều kiện $\le 15$). |
| max_stale_s | 300.0s | Sự cố sập toàn bộ provider kéo dài 120s (t=240..360). Đặt `max_stale_s = 300s` cho phép phục vụ 157 câu trả lời cũ (`stale_served = 157`), nâng `faq_availability` từ 87.33% lên 96.45% (đạt yêu cầu $\ge 93\%$). |

Thí nghiệm:

1. **Ngưỡng similarity:**
   - Với `threshold = 0.80`: hit_ratio = 66.6%, **wrong_answers = 157** (tất cả là lỗi `other_question`), cost/turn = 0.965.
   - Với `threshold = 0.92`: hit_ratio = 65.2%, **wrong_answers = 0**, cost/turn = 0.915, faq_availability = 96.45%.
   - Với `threshold = 0.97`: hit_ratio tụt xuống 58.1%, wrong_answers = 0, cost/turn = 0.964.
   *Giải thích:* Ở ngưỡng 0.80, các câu hỏi ngữ nghĩa gần nhau nhưng khác mục đích (ví dụ: "Thời gian xử lý hoàn tiền là bao lâu?" vs "Điều kiện chính sách hoàn tiền?") có độ tương đồng cosine ~0.83–0.88 bị cache nhận nhầm, dẫn đến câu trả lời sai nghiêm trọng. Ngưỡng 0.92 phân biệt chính xác ngữ nghĩa câu hỏi.

2. **TTL:**
   - Khi chạy với `ttl_s = 30s`: `outdated_answers = 2`, hit_ratio = 59.2%.
   - Khi chạy với `ttl_s = 60s`: `outdated_answers = 3`, hit_ratio = 65.2%.
   - Khi chạy với `ttl_s = 300s`: `outdated_answers = 95` (vượt ngưỡng 15), hit_ratio = 73.1%.
   *Giải thích:* Đối với các thay đổi nội dung không báo trước (silent updates như biểu phí đối tác vận chuyển không có webhook), hệ thống không hề nhận được sự kiện để kích hoạt invalidate. Do đó, thời gian sống TTL chính là cơ chế phòng vệ duy nhất để ép cache phải làm mới dữ liệu từ nguồn.

3. **Stale-if-error:**
   - Khi đặt `max_stale_s: 0`: Số câu trả lời stale phục vụ bằng 0 (`stale_served = 0`), khiến `faq_availability` tụt từ 96.45% xuống chỉ còn **87.33%** (thất bại thảm hại vì không đạt chuẩn 93%).
   - Cơ chế stale-if-error cứu sống 157 yêu cầu FAQ trong 2 phút mất điện toàn bộ provider.

*Trả lời:*
- **Vì sao câu hỏi cá nhân không được cache, kể cả khi cache theo từng user?**
  Dữ liệu cá nhân (trạng thái đơn hàng, số dư hoàn tiền, điểm thưởng) thay đổi liên tục theo thời gian thực (stateful). Nếu cache, người dùng có thể thấy trạng thái cũ vừa được cập nhật 1 giây trước. Hơn nữa, việc cache dữ liệu định danh theo người dùng tiềm ẩn rủi ro rò rỉ dữ liệu chéo rất nghiêm trọng nếu logic mapping user_id gặp bug bảo mật.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  Không được chỉ dựa vào metric HTTP status 200 hay availability tổng thể. Cần alert dựa trên **tỷ lệ lỗi backend gốc của provider (upstream provider error rate / 5xx rate)** và tốc độ tăng trưởng của metric **`stale_served_rate` (tỷ lệ request đang phải trả lời bằng dữ liệu cũ do backend chết)**. Khi `stale_served_rate > 0` vượt quá ngưỡng quy định, hệ thống phải kích hoạt page cảnh báo kỹ sư trực ca ngay lập tức.
