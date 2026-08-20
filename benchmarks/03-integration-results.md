# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3841.8 | 3841.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 3395.2 | 3395.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 3405.6 | 3405.6 |

Mean per stage (ms): embed **0.0** � retrieve **0.0** �
llm **3547.5** � total **3547.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

Khai báo trung thực từng mảnh:

| Day | Piece | Trạng thái | Thực tế là gì |
|:--|:--|:--|:--|
| N16 | Cloud / IaC | **STUB** | Không có. Chạy local trên laptop, không provision hạ tầng. |
| N17 | Data pipeline | **STUB** | Không có ingestion/transform. Corpus là `TOY_DOCS` hard-code trong `pipeline.py`. |
| N18 | Lakehouse | **STUB** | Không có. 6 document nằm trong list Python trong bộ nhớ. |
| N19 | Vector + features | **STUB** | Không có vector index. `retrieve()` dùng **keyword overlap**, không phải embedding. Không chạy `serve-embed` nên `embed()` trả `None` và code rơi về nhánh so khớp từ khoá. |
| N20 | Serving | **REAL** | `llama-server` thật, Gemma 4 E2B `UD-Q4_K_XL`, gọi qua HTTP `/v1/chat/completions`. |

Nói thẳng: **chỉ có tầng serving là thật.** Bốn tầng còn lại là stub. Bằng chứng
nằm ngay trong header của file này (`retrieval backend: keyword overlap`) và ở cột
`embed = 0.0 ms` — không có model embedding nào được gọi thì mới ra đúng 0.

## Stage nào chi phối, và có đúng như tôi nghĩ không?

**llm chiếm 3547.5 / 3547.6 ms = 100.0% tổng thời gian.** Hai stage kia gộp lại còn
chưa tới 0.1 ms.

**Có đúng kỳ vọng không? Đúng về thứ hạng, nhưng sai về mức độ.** Tôi đoán trước là
LLM sẽ thắng, nhưng không nghĩ nó thắng tuyệt đối đến mức retrieve gần như *không đo
được*. Lý do là stub: so khớp từ khoá trên **6 document trong RAM** là vài chục phép
so sánh chuỗi. Một vector DB thật với hàng triệu vector, cộng một lần gọi embedding
model qua mạng, sẽ đưa retrieve lên hàng chục đến hàng trăm ms. **Con số 0.0 ms này
là hệ quả của stub, không phải bằng chứng rằng retrieval luôn miễn phí** — và đó là
điều dễ kết luận nhầm nhất từ bảng trên.

Nhưng ngay cả khi retrieve tốn 100 ms thật, nó cũng chỉ là **2.8%** của 3547 ms.
Thứ hạng không đổi.

## Nếu phải giảm latency pipeline này đi 2 lần

Bắt buộc phải tấn công vào **stage llm**, vì Amdahl không cho lựa chọn nào khác: nó
chiếm 100%, nên tối ưu hai stage kia xuống 0 tuyệt đối cũng chỉ tiết kiệm 0.1 ms.

Tách `server_timings` của query đầu ra để biết đánh vào đâu **trong** stage llm:

- `prompt_n = 149` token, `prompt_ms = 530.6` ⇒ **prefill ≈ 0.53 s** (15%)
- `predicted_n = 30` token, `predicted_ms = 935.2` ⇒ **decode ≈ 0.94 s** (26%)
- Phần còn lại (~2.4 s) là overhead HTTP, sampling và khởi tạo request.

Vậy **decode đắt gần gấp đôi prefill** dù chỉ sinh 30 token so với 149 token đọc vào.
Mỗi token decode tốn 31 ms, mỗi token prefill tốn 3.6 ms — chênh gần **9 lần**. Đúng
lý thuyết trong deck: prefill xử lý cả 149 token song song trong một lượt (compute-
bound, tận dụng được ALU), còn decode phải đọc lại toàn bộ trọng số model cho *từng*
token một (memory-bandwidth-bound, batch size 1).

Thứ tự tôi sẽ làm, đắt nhất trước:

1. **Giảm số token sinh ra.** Đây là đòn hiệu quả nhất và rẻ nhất, vì latency decode
   tỉ lệ thuận tuyến tính với `predicted_n`. Câu trả lời hiện dài ~30 token; siết
   prompt để trả lời ngắn gọn, hoặc hạ `max_tokens`, cắt gần như trực tiếp phần 26%.
2. **Batch 3 query lại thay vì gửi tuần tự.** Bài load test đã chứng minh server này
   chạy được 4 slot đồng thời ở 98% công suất. Ba query độc lập nhau, không có lý do
   gì phải xếp hàng — riêng điều này đã có thể xấp xỉ chia 3 tổng thời gian.
3. **Streaming.** Không giảm được tổng thời gian, nhưng đưa TTFT xuống ~0.5 s, nên
   *cảm nhận* của người dùng cải thiện nhiều hơn con số 2x.
4. **Chỉ khi đó mới đụng tới prompt caching.** Với RAG, mỗi query có context khác
   nhau nên phần prefix dùng chung chỉ là system prompt — tiết kiệm được ít. Nó chỉ
   đáng nếu corpus lớn lên và context bắt đầu lặp lại giữa các query.
