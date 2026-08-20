# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Vũ Hải Nam
**Cohort:** ⬛ TỰ ĐIỀN (A20-K1 hay A20-K2?)
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home (build 10.0.26200)
- **CPU:** AMD Ryzen 7 H 255 w/ Radeon 780M Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2 + AVX-512 (llama.cpp nạp backend `ggml-cpu-zen4.dll`, tức là build Zen 4 có AVX-512)
- **RAM:** 23.3 GB
- **Accelerator:** AMD Radeon 780M (iGPU), qua **Vulkan** — UMA, fp16 + bf16, coopmat
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip` (build `b10488`)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`, mặc định — 23.3 GB RAM nên không cần hạ xuống Qwen3.5 0.8B)
- **Quantization:** `UD-Q4_K_XL` (primary, 2.97 GB) + `UD-Q2_K_XL` (compare, 2.24 GB)

**Chạy ở đâu:** laptop của tôi, hoàn toàn local. Không dùng cloud fallback.

**Setup story:**

Ba thứ phải sửa, cả ba đều là vấn đề riêng của Windows chứ không phải của lab:

1. **`lab.ps1` không parse được.** Windows PowerShell 5.1 đọc file script không có
   BOM theo codepage hệ thống (cp1252), nên ký tự em dash `—` trong một chuỗi bị
   giải mã sai, làm vỡ dấu ngoặc kép và kéo theo lỗi parse dây chuyền
   (`'<' operator is reserved`, `missing terminator`). Sửa bằng cách lưu lại file
   dưới dạng UTF-8 **có BOM**.
2. **`make verify` crash với `UnicodeDecodeError`.** Cùng gốc rễ: Python trên
   Windows mặc định dùng cp1252 cho text I/O, nên `verify.py` chết khi đọc tiếng
   Việt trong `REFLECTION.md`, và các file `benchmarks/*.md` do lab sinh ra bị ghi
   thành cp1252 (dấu `·` hiển thị lỗi khi đẩy lên GitHub). Sửa bằng cách set
   `PYTHONUTF8=1` trong `lab.ps1` và convert lại các file đã sinh sang UTF-8.
3. **iGPU hết VRAM làm `llama-bench` chết im lặng.** Xem chi tiết ở §5 — đây là
   workaround đáng giá nhất, vì nó không báo lỗi mà trả về `0.0 tok/s` trông y hệt
   một kết quả đo thật.

Việc tải model và runtime thì trơn tru, không cần đụng tới `MANUAL-DOWNLOAD.md`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5047 | 408 / 467 | 30.6 / 30.8 | 2337 / 2404 / 2404 | 32.6 |
| UD-Q2_K_XL | 2.24 | 4994 | 380 / 427 | 24.6 / 24.8 | 1932 / 1956 / 1956 | 40.6 |

**Quan sát:**

2-bit decode nhanh hơn **1.25x** (40.6 vs 32.6 tok/s) và nhỏ hơn **0.73 GB** (giảm 25%).
Điều đáng nói là mức tăng tốc gần **trùng khớp** với mức giảm dung lượng — không phải
trùng hợp. Decode ở batch size 1 phải đọc lại toàn bộ trọng số từ RAM cho mỗi token,
nên thời gian mỗi token tỉ lệ thuận với số byte trọng số. Ít hơn 25% byte thì nhanh
hơn xấp xỉ 25%. TPOT xác nhận: 30.6 → 24.6 ms, đúng 1.24x.

Ngược lại **TTFT gần như không đổi** (408 → 380 ms, chỉ 1.07x). Prefill xử lý cả prompt
song song nên nó compute-bound — thu nhỏ trọng số không giúp được nhiều. Cùng một thay
đổi, hai giai đoạn phản ứng hoàn toàn khác nhau: đó là bằng chứng trực tiếp rằng prefill
và decode bị chặn bởi hai tài nguyên khác nhau.

Tôi đã chạy song song hai server để so chất lượng (4-bit ở `:8080`, 2-bit ở `:8090` qua
`serve.py --compare --port 8090`), hỏi cùng 2 câu cho cả hai: một câu về continuous
batching, một câu về KV cache — đều là kiến thức trong lab nên dễ soi sai.

Kết quả hơi khác mình nghĩ trước khi test: **cả hai bản đều trả lời đúng, không đứa
nào bịa hay lạc đề.** Khác biệt chủ yếu nằm ở *văn phong* chứ không phải *độ đúng*.
Bản 4-bit viết gọn, đúng thuật ngữ hơn (câu batching nó chỉ thẳng ra là giải quyết
vấn đề của "static batching" — rất trúng). Bản 2-bit dài dòng hơn chút và có đoạn
dùng từ hơi lỏng, ví dụ gọi KV cache là "a small, high-performance memory structure"
— nghe thì xuôi tai nhưng không chuẩn, vì KV cache thực ra là thứ ăn RAM/VRAM nhiều
nhất khi context dài, chẳng "small" tí nào. Kiểu sai này không đến mức sai kiến thức,
chỉ là chọn từ hơi ẩu.

Nên kết luận thật của tôi: **với câu hỏi ngắn kiểu định nghĩa cơ bản thì 2-bit vẫn
dùng tốt**, không cần tránh. Nhưng tôi mới test 2 câu dễ, chưa thử câu nào cần số
liệu chính xác hay suy luận nhiều bước — nhìn cách 2-bit hay "phóng đại" câu chữ, tôi
đoán nếu hỏi khó hơn thì khoảng cách chất lượng sẽ lộ rõ hơn. Vì máy này có 23.3 GB
RAM nên dung lượng model không phải ràng buộc với tôi, nên tôi vẫn dùng `UD-Q4_K_XL`
(bản primary) cho toàn bộ phần còn lại của lab — chủ yếu vì nó là default của
`models/active.json`, không hẳn vì 2-bit tệ. 2-bit sẽ đáng cân nhắc hơn hẳn nếu máy
RAM thấp (8 GB) hoặc tác vụ chỉ cần trả lời nhanh, không cần chính xác tuyệt đối.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.81 | 9700 | 16000 | 19000 | 8.4 | 0.0% |
| 50 | 0.91 | 30000 | 53000 | 57000 | 27.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.12×
- **P95 tăng:** 3.31×
- **Effective concurrency ở 50 users:** 27.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.92 / 4 slots (98%)

**Saturation reading:**

Server bão hoà ở đâu đó **dưới 10 users** — tức là đã bão hoà từ trước cả run đầu tiên.

Con số thuyết phục tôi không phải P95, mà là **RPS gần như đứng yên**: 0.81 → 0.91 khi
tải tăng 5 lần. 40 user thêm vào chỉ mua được 0.1 RPS. Nếu server còn dư công suất ở 10
users thì RPS phải tăng đáng kể; nó không tăng.

**Queue time hay compute time? Tôi biết chắc là queue time, và không phải bằng suy đoán.**
`make metrics` cho hai cột quyết định: `requests_processing` cố định ở **4.0** trong cả
15 mẫu, `requests_deferred` dao động **44–46**. Cộng lại 4 + 46 = đúng 50 users. Nghĩa
là tại mọi thời điểm chỉ 8% số request được phục vụ, 92% đang xếp hàng.

Phép loại trừ: nếu compute chậm đi thì `n_busy_slots_per_decode` phải **tụt** (server
vật lộn với từng token). Thực tế nó **tăng** lên 3.92/4 và đứng yên ở đó. Server đang
decode ở tốc độ tối đa nó có; phần latency thêm vào chỉ có thể là thời gian nằm chờ
trước khi được cấp slot.

Kiểm chứng bằng số học: 50 users / 4 slots ⇒ mỗi request đợi khoảng 12 lượt trước mình.
Với P50 ở 10 users là 9.7 s, mô hình hàng đợi dự đoán độ trễ ở 50 users vào cỡ hàng
chục giây — đo được đúng **30 s**. Cùng bậc độ lớn, nên mô hình giải thích được dữ liệu.

**Knob tôi đổi trước: `--parallel` 4 → 8** (`LAB_PARALLEL=8`).

Đặt SLO để có cái mà đo: **P95 ≤ 20 s**. Ở 10 users P95 = 16 s → đạt. Ở 50 users P95 =
53 s → trượt 2.65 lần, nên goodput@SLO rơi từ ~0.81 RPS xuống gần **0**, dù throughput
thô vẫn báo 0.91 RPS. Đó chính là lý do throughput thô là con số gây hiểu nhầm.

Chọn `--parallel` vì nghẽn đã được định vị là **hàng đợi**, và đây là knob duy nhất tác
động trực tiếp vào độ sâu hàng đợi. **Không** chọn tối ưu tốc độ decode (quantization,
threads): 2-bit chỉ nhanh hơn 1.25x, mà mỗi request chỉ dành một phần nhỏ vòng đời để
được decode — rút ngắn 25% của phần nhỏ đó gần như không dịch chuyển P95.

Nhưng `--parallel` không miễn phí, và tôi dự đoán trước giới hạn: 8 slot chia đôi cùng
một băng thông bộ nhớ và cùng ngân sách KV cache trong `ctx=2048`, nên TPOT mỗi request
sẽ **xấu đi** ngay cả khi throughput tổng tăng. Nó đổi TPOT lấy queue time. Vì P95 hiện
đang bị queue time chi phối áp đảo nên đó là cuộc đổi chác có lời — **cho đến một ngưỡng
nào đó**. Phép đo để kiểm chứng: chạy lại `make load-50` với `LAB_PARALLEL=8`, xem P95
có xuống dưới 20 s không hay TPOT xấu đi đủ để triệt tiêu phần lợi.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không có | **stub** — chạy local trên laptop, không provision hạ tầng |
| N17 Data pipeline | không có | **stub** — corpus là `TOY_DOCS` hard-code trong `pipeline.py` |
| N18 Lakehouse | không có | **stub** — 6 document nằm trong list Python trong RAM |
| N19 Vector + features | keyword overlap | **stub** — không có vector index; `embed()` trả `None` nên rơi về so khớp từ khoá |
| N20 Serving | `llama-server` | **real** — Gemma 4 E2B `UD-Q4_K_XL` qua HTTP `/v1/chat/completions` |

Nói thẳng: **chỉ tầng serving là thật**, bốn tầng còn lại là stub. Bằng chứng nằm ngay
trong report: header ghi `retrieval backend: keyword overlap`, và cột `embed = 0.0 ms`
— không gọi model embedding nào thì mới ra đúng 0.

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 3547.5 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection:**

**Đúng về thứ hạng, nhưng sai về mức độ.** Tôi đoán trước LLM sẽ thắng, nhưng không nghĩ
nó thắng tuyệt đối tới mức retrieve gần như *không đo được*. Lý do là stub: so khớp từ
khoá trên 6 document trong RAM chỉ là vài chục phép so sánh chuỗi. Một vector DB thật
với hàng triệu vector cộng một lần gọi embedding qua mạng sẽ đưa retrieve lên hàng chục
đến hàng trăm ms. **Con số 0.0 ms là hệ quả của stub, không phải bằng chứng rằng
retrieval luôn miễn phí** — đó là điều dễ kết luận nhầm nhất từ bảng này. Dù vậy, ngay
cả khi retrieve tốn 100 ms thật thì nó vẫn chỉ là 2.8% của 3547 ms; thứ hạng không đổi.

Nếu phải giảm 2×, bắt buộc đánh vào **stage llm** — Amdahl không cho lựa chọn khác.
Tách `server_timings` để biết đánh vào đâu *bên trong* nó:

- prefill: 149 token trong 530.6 ms ⇒ **3.6 ms/token**
- decode: 30 token trong 935.2 ms ⇒ **31 ms/token**

Decode đắt hơn prefill **~9 lần trên mỗi token**, dù chỉ sinh 30 token so với 149 token
đọc vào. Đúng lý thuyết: prefill xử lý cả 149 token song song trong một lượt nên
compute-bound và tận dụng được ALU; decode phải đọc lại toàn bộ trọng số cho *từng*
token nên memory-bandwidth-bound.

Thứ tự tôi làm, hiệu quả nhất trước: (1) **giảm số token sinh ra** — latency decode tỉ
lệ tuyến tính với `predicted_n`, nên siết prompt hoặc hạ `max_tokens` cắt thẳng phần
26%; (2) **batch 3 query thay vì gửi tuần tự** — load test đã chứng minh server chạy 4
slot ở 98% công suất, ba query độc lập nhau thì không có lý do gì phải xếp hàng, riêng
điều này xấp xỉ chia 3 tổng thời gian; (3) **streaming** — không giảm tổng thời gian
nhưng đưa TTFT xuống ~0.5 s nên cảm nhận cải thiện hơn cả 2×; (4) **prompt caching sau
cùng** — với RAG mỗi query có context khác nhau nên prefix dùng chung chỉ là system
prompt, tiết kiệm ít.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** đặt `-t` bằng đúng số core **vật lý** (8) thay vì để 1 thread, khi decode
chạy trên CPU (`ngl=0`).

```
before:  7.8 tok/s   (-t 1,  ngl=0)
after:   19.6 tok/s  (-t 8,  ngl=0)
speedup: 2.51×
```

**Tại sao nó work:**

Curve từ `make tune` có hình chữ nón rất rõ, và ba đoạn của nó bị chặn bởi **ba cơ chế
khác nhau** — đó mới là phần đáng nói, chứ không phải con số 2.51×:

```
 1 thread  →  7.8 tok/s   (40%)
 4 threads → 17.1 tok/s   (88%)
 8 threads → 19.6 tok/s   (100%)  ← knee, = physical cores
16 threads → 17.0 tok/s   (87%)   ← hyperthreads, chậm hơn
32 threads → 11.5 tok/s   (59%)   ← oversubscribe, sụp hẳn
```

**1 → 4 threads (+2.19× cho 4× thread):** ở đây thread mua được thứ đang thiếu thật. Một
core đơn lẻ không tạo đủ số request bộ nhớ đang chờ (memory-level parallelism) để lấp
đầy băng thông controller — CPU dành phần lớn thời gian *đứng chờ* dữ liệu chứ không
phải tính toán. Thêm core là thêm luồng nạp song song. Nhưng đã không đạt 4×: hiệu suất
tụt từ 100% xuống 55%, báo hiệu điểm bão hoà đang tới.

**4 → 8 (+15% cho 2× thread):** chạm trần băng thông bộ nhớ. Decode ở batch size 1 phải
đọc lại **toàn bộ 2.97 GB trọng số cho mỗi token**, trong khi số phép tính trên mỗi byte
đọc vào cực thấp. Bài toán bị chặn bởi tốc độ RAM, không phải số ALU. Băng thông đã bão
hoà thì thêm core không giúp gì — đó chính là định nghĩa của knee.

**8 → 16 (−13%):** bật SMT. Hai hyperthread trên cùng một core vật lý không có thêm đơn
vị tính toán riêng — chúng dùng chung ALU, chung port nạp/lưu, và quan trọng nhất là
**chung L1/L2**. Với workload streaming toàn bộ trọng số qua cache, hai thread cùng core
liên tục đẩy dữ liệu của nhau ra khỏi cache; phần thiệt do cache miss lớn hơn phần lợi
do che giấu độ trễ. SMT giúp ích khi các thread stall vì lý do *khác nhau*; ở đây chúng
stall vì **cùng một lý do** (chờ RAM), nên chẳng có gì để xen kẽ.

**16 → 32 (−32%):** oversubscribe gấp đôi. Thêm chi phí ở tầng OS: scheduler phải liên
tục context-switch, mỗi lần switch lại làm nguội cache của thread vừa bị đá ra. Tệ hơn,
llama.cpp phải đồng bộ *tất cả* thread ở cuối mỗi layer, nên cả batch chỉ nhanh bằng
thread chậm nhất — càng nhiều thread thì xác suất có một thread bị OS cướp lượt càng
cao, và mọi thread khác đứng chờ nó ở rào đồng bộ.

**Kết quả khác kỳ vọng ở chỗ nào — và đây mới là điều tôi học được nhiều nhất**

Deck nói `-t` là một trong những knob quan trọng nhất. Trên máy này điều đó **chỉ đúng
một nửa**. Tôi chạy lại đúng sweep đó ở cấu hình mặc định của lab (`ngl=99`, đẩy layer
lên iGPU Radeon 780M qua Vulkan) và curve **biến mất hoàn toàn**:

| threads (-t) | `ngl=0` (CPU) | `ngl=99` (Vulkan) |
|--:|--:|--:|
| 1 | 7.8 | 34.7 |
| 4 | 17.1 | 33.4 |
| 8 | **19.6** | 32.5 |
| 16 | 17.0 | 33.1 |
| 32 | 11.5 | 33.9 |
| **spread** | **2.51×** | **1.07×** |

Với GPU offload, spread chỉ còn 1.07× — nằm gọn trong sai số đo (llama-bench báo ±1.5
tok/s ở các điểm này). **`-t` không còn là knob nữa**, vì decode đã chạy trên shader của
iGPU; CPU thread chỉ còn xếp lệnh và đồng bộ, một công việc gần như không tốn gì.

Bài học không phải "thread quan trọng" mà là: **giá trị của một knob phụ thuộc vào cấu
hình, không phải vào bản thân knob.** Cùng một tham số, cùng model, cùng máy — đáng
2.51× ở cấu hình này, đáng 1.00× ở cấu hình kia. Nếu tôi chỉ chạy sweep ở chế độ mặc
định, tôi đã kết luận sai rằng "thread count không quan trọng", trong khi sự thật là
"thread count không quan trọng *khi đã có GPU offload*". Muốn biết một knob có đáng
chỉnh không thì phải biết trước **cái gì đang là nút thắt** — chỉnh mù thì 4 trong 5 lần
là đang chỉnh thứ không nằm trên đường tới hạn.

Cũng đáng ghi nhận: iGPU cho ~33 tok/s so với ~19.6 tok/s của cấu hình CPU tốt nhất
(1.7×). Nên với máy này, quyết định lớn nhất không phải chọn `-t` nào, mà là **có bật
offload hay không**.

**Cái bẫy đo lường suýt làm hỏng toàn bộ phần này**

Lần chạy `make tune` đầu tiên trả về `0.0 tok/s` ở **4/5 điểm đo**. Nguyên nhân không
liên quan gì tới thread: hai tiến trình `llama-server` cũ (port 8080 và 8090, sót lại từ
bước so sánh quantization ở 1.1) vẫn giữ bộ nhớ iGPU, nên `llama-bench` chết với
`ggml_vulkan: vk::Device::allocateMemory: ErrorOutOfDeviceMemory`. Tắt hai tiến trình
đó xong thì cả hai sweep chạy sạch.

Điều này khớp trực tiếp với cảnh báo trong `labs/01-measure/README.md` rằng benchmark
cạnh 40 tab Chrome là đang đo Chrome. Ở đây còn tệ hơn: phép đo không sai lệch một chút
— nó **fail hẳn**, nhưng fail *im lặng* thành số `0.0` trong bảng, trông y hệt một kết
quả đo thật. Nếu không kiểm tra lại, tôi đã viết cả một đoạn phân tích cho một con số
không hề tồn tại. Bài học vận hành: khi một điểm đo trả về 0, mặc định phải coi đó là
**công cụ hỏng**, không phải **kết quả bằng 0**.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Việc một phép đo có thể **fail im lặng thành `0.0`** thay vì báo lỗi. Bốn trong năm
điểm đo đầu tiên của tôi là số không, và bảng markdown do lab sinh ra hiển thị chúng
y hệt dữ liệu thật — có cả cột "vs best" tính phần trăm đàng hoàng. Nếu tôi tin bảng
đó, tôi đã kết luận rằng 16 thread là tối ưu, trong khi sự thật là 8, và lý do thật
sự chỉ là hai server cũ đang giữ VRAM.

Điều thứ hai: **`n_busy_slots_per_decode = 3.92/4` trông như một tin tốt** ("server
chạy 98% công suất!") nhưng đặt cạnh `requests_deferred = 46` thì nó lại là bằng
chứng của một hệ thống đang quá tải nặng. Cùng một con số, hai kết luận trái ngược,
tuỳ vào việc bạn có nhìn con số bên cạnh nó hay không.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
