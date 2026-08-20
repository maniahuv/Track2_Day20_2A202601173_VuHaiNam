# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 7.8 | 40% |
| 4 | 17.1 | 88% |
| 8 | 19.6 | 100% |
| 16 | 17.0 | 87% |
| 32 | 11.5 | 59% |

**Best**: `-t 8` at 19.6 tok/s
**Slowest tested**: `-t 1` at 7.8 tok/s (2.51x spread)
**Against the physical-core default** (`-t 8`, 19.6 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

> **Ghi chú về cách chạy:** sweep này chạy với `ngl=0` (CPU-only), *khác* với phần
> còn lại của lab (`ngl=99`). Lý do và ý nghĩa của việc đó nằm ở mục "Chạy lại cùng
> sweep với GPU offload" bên dưới — đó cũng là phần đáng giá nhất của bài đo này.

### Knee nằm ở đâu

**Knee ở đúng `-t 8`, bằng chính xác số core vật lý (8 physical / 16 logical).**
Curve có hình chữ nón rất rõ, spread **2.51x** từ đáy lên đỉnh:

```
 1 thread  →  7.8 tok/s   (40%)
 4 threads → 17.1 tok/s   (88%)
 8 threads → 19.6 tok/s   (100%)  ← knee, = physical cores
16 threads → 17.0 tok/s   (87%)   ← hyperthreads, chậm hơn
32 threads → 11.5 tok/s   (59%)   ← oversubscribe, sụp hẳn
```

### Vì sao lại ở đó — ba cơ chế khác nhau trên ba đoạn của curve

**Đoạn 1 → 4 threads: tăng gần tuyến tính (7.8 → 17.1, tức 2.19x cho 4x thread).**
Ở đây thread thực sự mua thêm được thứ đang thiếu. Với 1 thread, một core đơn lẻ
không tạo đủ số request bộ nhớ đang chờ (memory-level parallelism) để lấp đầy băng
thông của controller — CPU dành phần lớn thời gian *đứng chờ* dữ liệu về chứ không
phải tính toán. Thêm core là thêm luồng nạp song song, nên băng thông thực tế khai
thác được tăng lên. Nhưng đã không đạt 4x: hiệu suất tụt từ 100% xuống 55%, dấu hiệu
cho thấy điểm bão hoà đang tới gần.

**Đoạn 4 → 8: chững lại (17.1 → 19.6, chỉ +15% cho 2x thread).** Đây là lúc chạm
trần băng thông bộ nhớ. Decode với batch size 1 phải đọc lại **toàn bộ 2.97 GB
trọng số cho mỗi token sinh ra**, trong khi lượng phép tính trên mỗi byte đọc vào
lại cực thấp. Bài toán vì thế bị chặn bởi tốc độ RAM, không phải bởi số ALU. Khi
băng thông đã bão hoà thì thêm core không giúp gì — đó chính là định nghĩa của knee.

**Đoạn 8 → 16: tụt (19.6 → 17.0, mất 13%).** 16 là số logical core, tức là bật SMT.
Hai hyperthread trên cùng một core vật lý **không** có thêm đơn vị tính toán riêng —
chúng dùng chung ALU, chung port nạp/lưu, và quan trọng nhất là **chung L1/L2 cache**.
Với workload streaming toàn bộ trọng số qua cache như thế này, hai thread cùng core
liên tục đẩy dữ liệu của nhau ra khỏi cache. Kết quả là tỉ lệ cache miss tăng, và
phần thiệt do miss lớn hơn phần lợi do che giấu độ trễ. SMT giúp ích khi các thread
hay bị stall vì lý do khác nhau; ở đây chúng stall vì **cùng một lý do** (chờ RAM),
nên chẳng có gì để xen kẽ.

**Đoạn 16 → 32: sụp mạnh (17.0 → 11.5, mất 32%).** 32 thread trên 16 logical core
là oversubscribe gấp đôi. Giờ có thêm chi phí ở tầng OS: scheduler của Windows phải
liên tục context-switch, mỗi lần switch lại làm nguội cache của thread vừa bị đá ra,
và llama.cpp còn phải đồng bộ *tất cả* thread ở cuối mỗi layer — nên toàn bộ batch
chỉ nhanh bằng thread chậm nhất. Càng nhiều thread thì xác suất có một thread bị OS
cướp mất lượt chạy càng cao, và mọi thread khác phải đứng chờ nó ở rào đồng bộ.

Tóm lại, ba đoạn bị chặn bởi ba thứ khác nhau: **thiếu song song bộ nhớ** → **trần
băng thông** → **tranh chấp cache** → **overhead scheduling**.

### Chạy lại cùng sweep với GPU offload — và đây mới là phần bất ngờ

Sweep phía trên chạy `ngl=0`. Khi chạy lại đúng nó với `ngl=99` (mặc định của lab
trên máy này, đẩy layer lên iGPU Radeon 780M qua Vulkan), curve **biến mất hoàn toàn**:

| threads (-t) | tg128, `ngl=0` (CPU) | tg128, `ngl=99` (Vulkan) |
|:--|--:|--:|
| 1 | 7.8 | 34.7 |
| 4 | 17.1 | 33.4 |
| 8 | **19.6** | 32.5 |
| 16 | 17.0 | 33.1 |
| 32 | 11.5 | 33.9 |
| **spread** | **2.51x** | **1.07x** |

Với GPU offload, spread chỉ còn **1.07x** — nằm gọn trong sai số đo (llama-bench báo
± khoảng 1.5 tok/s ở các điểm này). Nói cách khác: **`-t` không còn là knob nữa.**
Điều đó hợp lý, vì lúc này decode chạy trên shader của iGPU; CPU thread chỉ còn làm
nhiệm vụ xếp lệnh và đồng bộ, một công việc gần như không tốn gì. Chỉnh `-t` lúc này
là đang chỉnh một thứ không nằm trên đường tới hạn.

Bài học thực tế rút ra: **giá trị của một knob phụ thuộc vào cấu hình, không phải
vào bản thân knob.** Cùng một tham số `-t`, cùng một model, cùng một máy — đáng 2.51x
ở cấu hình này và đáng 1.00x ở cấu hình kia. Nếu tôi chỉ chạy sweep ở chế độ mặc
định `ngl=99`, tôi sẽ kết luận sai rằng "thread count không quan trọng", trong khi
sự thật là "thread count không quan trọng *khi đã có GPU offload*".

Cũng đáng ghi nhận: iGPU cho ~33 tok/s so với ~19.6 tok/s của cấu hình CPU tốt nhất
— nhanh hơn **1.7x**. Nên với máy này, quyết định đúng đắn nhất không phải là chọn
`-t` nào, mà là **có bật offload hay không**.

### Một cái bẫy đo lường đã suýt làm hỏng kết quả

Lần chạy `make tune` đầu tiên của tôi trả về `0.0 tok/s` ở 4/5 điểm đo. Nguyên nhân
không phải thread count: hai tiến trình `llama-server` cũ (port 8080 và 8090, còn sót
lại từ bước so sánh quantization ở 1.1) vẫn đang giữ bộ nhớ của iGPU, nên `llama-bench`
chết với `ggml_vulkan: vk::Device::allocateMemory: ErrorOutOfDeviceMemory`. Sau khi
tắt hai tiến trình đó, cả hai sweep chạy sạch không lỗi.

Điều này khớp trực tiếp với cảnh báo trong `labs/01-measure/README.md` rằng chạy
benchmark cạnh 40 tab Chrome là đang đo Chrome. Ở đây thậm chí còn tệ hơn: phép đo
không sai lệch một chút — nó **fail hẳn**, nhưng lại fail *im lặng* thành số `0.0`
trong bảng, trông y hệt một kết quả đo thật. Nếu không kiểm tra lại, tôi đã viết cả
một đoạn phân tích cho một con số không hề tồn tại.
