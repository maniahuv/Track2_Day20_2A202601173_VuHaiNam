# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` � `--parallel 4` � 15 samples over
60s at 2.0s intervals � raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.92 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a � not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 6341 |

Highest sampled value was **3.92 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

**Peak batch width: 3.92 / 4 slots (98%).** Continuous batching hoạt động đúng như
thiết kế — scheduler gần như luôn nhồi đủ 4 request vào cùng một decode step. Trong
cả 15 mẫu, `requests_processing` cố định ở **4.0**, không dao động lần nào. Server
không hề có khoảng nghỉ.

**Hai con số này có mâu thuẫn với `02-server-results.md` không?** Nhìn qua thì có:
file kia báo effective concurrency **27.9**, còn ở đây đỉnh chỉ **3.92**. Chênh gần
7 lần. Nhưng chúng **không** mâu thuẫn — chúng đo hai thứ khác nhau, và chỗ chênh
lệch chính là điều đáng nói nhất của bài này:

- **3.92** là số request đang *thực sự được decode*. Nó bị chặn cứng bởi
  `--parallel 4`; không cách nào vượt quá 4.
- **27.9** (Little's Law = RPS × latency trung bình) là số request đang *nằm trong
  hệ thống*, tính cả những request đang xếp hàng chờ.

Cột `requests_deferred` chứng minh trực tiếp điều đó: nó dao động **44–46** suốt
gần cả run. Cộng lại: **4 đang chạy + 46 đang chờ = 50 users** — khớp chính xác
với số user locust mô phỏng. Vậy tại thời điểm bất kỳ, chỉ **8%** số request được
phục vụ; **92%** còn lại đang đợi.

**Tôi tin con số nào?** Tin cả hai, vì mỗi con số trả lời một câu hỏi khác nhau.
`n_busy_slots_per_decode` là gauge do chính llama.cpp phát ra nên nó là sự thật về
việc *server có bận không* — và câu trả lời là bận tối đa, không lãng phí slot nào.
Little's Law là suy ra từ phía client nên nó là sự thật về việc *người dùng chờ bao
lâu*. Nếu chỉ đọc 3.92 tôi sẽ kết luận sai rằng "server chạy hết công suất, ổn";
chỉ đọc 27.9 thì lại không biết nghẽn nằm ở đâu.

Ghép lại mới ra kết luận đúng: server **không** chậm vì tính toán yếu — nó đã chạy
98% công suất decode. Nó chậm vì **hàng đợi**. Và điều đó xác định luôn hướng sửa:
tăng `--parallel` (nhiều slot hơn) hoặc giảm tokens/request thì có tác dụng; tối ưu
tốc độ decode từng token thì gần như không, vì phần lớn thời gian của một request
là thời gian ngồi chờ chứ không phải thời gian được xử lý.

`kv_cache_usage_ratio` không được build `b10488` export nên tôi không kiểm chứng
được liệu KV cache có phải ràng buộc thứ hai hay không — đó là giới hạn của phép
đo này, không phải kết luận rằng KV cache còn dư.
