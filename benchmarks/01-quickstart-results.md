# 01 - Measure: latency baseline

Model `Gemma 4 E2B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5047 | 408 / 467 | 30.6 / 30.8 | 2337 / 2404 / 2404 | 32.6 |
| UD-Q2_K_XL | 2.24 | 4994 | 380 / 427 | 24.6 / 24.8 | 1932 / 1956 / 1956 | 40.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.25x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

**Số liệu:** `UD-Q2_K_XL` decode 40.6 tok/s so với 32.6 tok/s của `UD-Q4_K_XL` —
nhanh hơn **1.25x**, đổi lại nhỏ hơn **0.73 GB** (2.24 vs 2.97 GB, giảm 25%).

Điều đáng chú ý là **mức tăng tốc gần như trùng khớp với mức giảm dung lượng**:
1.25x nhanh hơn cho 1.33x nhỏ hơn. Đó không phải trùng hợp. Decode chỉ sinh 1 token
mỗi bước nên mỗi bước phải đọc lại **toàn bộ** trọng số từ RAM; với batch size 1,
số phép nhân cộng quá ít để làm ALU bận, nên thời gian mỗi token bị chặn bởi
băng thông bộ nhớ chứ không phải FLOPs. Weights nhỏ hơn 25% thì mỗi bước decode
đọc ít hơn 25% byte, và tốc độ tăng gần đúng bằng tỉ lệ đó. TPOT P50 xác nhận:
30.6 ms → 24.6 ms, đúng bằng 1.24x.

Ngược lại, **TTFT gần như không đổi** (408 → 380 ms, chỉ 1.07x) và **load time
cũng vậy** (5047 → 4994 ms). Điều này hợp lý: prefill xử lý cả prompt cùng lúc nên
là compute-bound — nó bị chặn bởi FLOPs chứ không phải băng thông, nên thu nhỏ
trọng số không giúp được nhiều. Cùng một model, cùng một máy, nhưng hai giai đoạn
phản ứng hoàn toàn khác nhau trước cùng một thay đổi.

**Có đáng dùng không?** Tôi đã chạy song song hai server để tự kiểm chứng —
4-bit ở `:8080` (`make serve`) và 2-bit ở `:8090`
(`serve.py --compare --port 8090`) — rồi hỏi cùng một câu cho cả hai.

Câu hỏi mình dùng để test: *"What problem does continuous batching solve in LLM
serving?"* và *"what is the KV cache and why does it grow with context length?"*
— toàn kiến thức nằm trong nội dung lab nên dễ soi sai.

Kết quả thì hơi bất ngờ so với mình nghĩ trước khi test: **cả hai đều trả lời
đúng, không đứa nào bịa hay lạc đề.** Bản 4-bit viết gọn và "chuẩn textbook" hơn
— ví dụ câu batching nó nói thẳng là giải quyết vấn đề của *static batching*,
nghe rất trúng ý. Bản 2-bit thì dài dòng hơn một tí và có chỗ dùng từ hơi lỏng
lẻo — kiểu gọi KV cache là "a small, high-performance memory structure", nghe
thì không sai hẳn nhưng không chuẩn xác lắm (KV cache đâu có "small", nó còn
là thứ ăn RAM/VRAM nhiều nhất khi context dài). Nhìn chung là khác biệt về
*văn phong* nhiều hơn là khác biệt về *độ đúng*.

Nên câu trả lời thật của mình cho "có đáng dùng không" là: **với câu hỏi ngắn,
kiến thức cơ bản kiểu này thì 2-bit vẫn ổn**, không đến mức phải tránh. Chỗ mình
lo hơn là mấy câu hỏi cần chi tiết số liệu chính xác hoặc suy luận nhiều bước —
kiểu đó chưa test nên chưa dám kết luận, nhưng dựa trên cách 2-bit hay "phóng
đại" ngôn từ (thấy rõ ở câu KV cache) thì mình đoán càng hỏi khó, khoảng cách
chất lượng sẽ càng rõ hơn là ở mấy câu định nghĩa cơ bản này.

Điều kiện để kết luận theo hướng nào: máy này có **23.3 GB RAM**, nghĩa là dung
lượng model **không** phải ràng buộc — bản 4-bit 2.97 GB nạp thoải mái. Nên nếu
bạn thấy chất lượng giảm dù chỉ chút ít, 25% tốc độ không đủ để bù. 2-bit chỉ
thực sự đáng khi RAM là ràng buộc cứng (máy 8 GB không nạp nổi bản 4-bit) hoặc
khi tác vụ chịu được sai sót (tóm tắt, phân loại thô).
