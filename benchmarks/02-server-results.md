# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=8` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 47 | 0.81 | 9700 | 16000 | 19000 | 8.4 | 0.0% |
| 50 | 53 | 0.91 | 30000 | 53000 | 57000 | 27.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.12x** (22% of linear) |
| P95 latency | **3.31x** |
| Effective concurrency at 50 users | 27.9 vs `--parallel 4` slots (occupancy/slot ratio 6.98) |

**Saturated.** Throughput delivered only 1.12x for 5x the offered load, and effective concurrency (27.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.12x while P95 moved 3.31x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server bão hoà ở đâu đó dưới 10 users — tức là đã bão hoà từ trước cả run đầu tiên.**

**Con số thuyết phục tôi:** không phải P95, mà là **RPS gần như đứng yên**: 0.81 →
0.91 khi offered load tăng 5 lần. Chỉ **1.12x throughput cho 5x tải**. Nếu server
còn dư công suất ở 10 users, RPS phải tăng đáng kể khi thêm user vào. Nó không tăng.
Toàn bộ 40 user thêm vào chỉ tạo ra thêm 0.1 RPS — phần còn lại biến thành thời gian
chờ.

**Bằng chứng phụ, và là bằng chứng quyết định về *bản chất* của độ trễ thêm vào:**
từ `02-server-batching-u50.md`, `requests_processing` cố định ở **4.0** trong cả 15
mẫu, còn `requests_deferred` là **44–46**. 4 chạy + 46 chờ = đúng 50 users.

Đây chính là cách tôi biết phần latency tăng thêm là **queue time chứ không phải
compute time**, và tôi không cần suy đoán:

- Nếu là compute chậm đi, `n_busy_slots_per_decode` sẽ tụt xuống (server vật lộn với
  từng token) và số token/giây tổng sẽ giảm.
- Thực tế ngược lại: busy slots **tăng** lên 3.92/4 (98%) và giữ nguyên ở đó. Server
  đang decode với tốc độ tối đa nó có.
- P95 tăng 3.31x trong khi số slot không đổi ⇒ phần thời gian thêm vào là thời gian
  request nằm trong hàng đợi trước khi được cấp slot.

Kiểm chứng bằng số học: 50 users chia cho 4 slot ⇒ mỗi request đợi trung bình khoảng
12 lượt phục vụ trước mình. Với P50 ở 10 users là 9.7 s, con số đó dự đoán độ trễ ở
50 users vào cỡ hàng chục giây — và P50 đo được đúng là **30 s**. Cùng một bậc độ
lớn, nên mô hình hàng đợi giải thích được dữ liệu.

**Nếu phải nâng goodput@SLO, tôi đổi knob nào trước?**

Đặt SLO cụ thể để có cái mà đo: **P95 ≤ 20 s**. Ở 10 users P95 = 16 s → đạt. Ở 50
users P95 = 53 s → trượt 2.65 lần. Nên goodput@SLO hiện tại rơi từ ~0.81 RPS xuống
gần **0** khi lên 50 users, dù throughput thô vẫn báo 0.91 RPS. Đó chính là lý do
throughput thô là con số gây hiểu nhầm.

Knob đầu tiên: **tăng `--parallel` từ 4 lên 8** (`LAB_PARALLEL=8`).

Lý do chọn knob này chứ không phải knob khác:

1. Nghẽn đã được định vị là **hàng đợi**, và `--parallel` là knob *duy nhất* tác động
   trực tiếp vào độ sâu hàng đợi. Gấp đôi số slot thì gấp đôi số request được phục vụ
   đồng thời, kéo thẳng queue time xuống.
2. **Không** chọn tối ưu tốc độ decode (đổi quantization, chỉnh threads): bước
   `make bench` cho thấy 2-bit chỉ nhanh hơn 1.25x, mà mỗi request lại chỉ dành một
   phần nhỏ vòng đời của nó để được decode — phần lớn là ngồi chờ. Rút ngắn 25% phần
   nhỏ đó gần như không dịch chuyển P95.
3. **Không** chọn thêm phần cứng: đây là bài đo trên laptop, và dữ liệu chưa chứng
   minh được phần cứng là giới hạn.

Nhưng `--parallel` **không miễn phí**, và tôi dự đoán trước giới hạn của nó: 8 slot
chia đôi cùng một băng thông bộ nhớ và cùng một ngân sách KV cache trong `ctx=2048`.
Nên TPOT mỗi request sẽ *xấu đi* ngay cả khi throughput tổng tăng. Nói cách khác,
`--parallel` đổi TPOT lấy queue time. Vì SLO tôi đặt là P95 end-to-end — mà P95 hiện
đang bị queue time chi phối áp đảo — thì đó là một cuộc đổi chác có lời, **cho đến
một ngưỡng nào đó**. Cách kiểm chứng: chạy lại `make load-50` với `LAB_PARALLEL=8`
và xem P95 có thực sự xuống dưới 20 s không, hay TPOT xấu đi đủ để triệt tiêu phần
lợi. Đó là phép đo tôi sẽ làm tiếp nếu có thêm thời gian.
