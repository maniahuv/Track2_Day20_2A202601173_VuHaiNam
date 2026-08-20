# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 14.7 | 1.00x | 44% |
| 8 | 16.9 | 1.15x | 51% |
| 16 | 17.2 | 1.17x | 52% |
| 24 | 18.2 | 1.24x | 55% |
| 32 | 20.5 | 1.39x | 62% |
| 99 | 33.1 | 2.25x | 100% |

Best: `-ngl 99` at 33.1 tok/s
-- 2.25x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

**Full offload thắng tuyệt đối, và curve không hề peak ở giá trị partial.**
`-ngl 99` cho 33.1 tok/s so với 14.7 tok/s CPU-only — nhanh hơn **2.25x**. Toàn bộ
6 điểm đo tăng đơn điệu, không có chỗ nào tụt xuống. Điều này khác với kỳ vọng "curve
peak dưới full offload rồi tụt lại" mà mô tả của script đặt ra cho GPU rời có VRAM
giới hạn.

**`-ngl 99` không có nghĩa là 99 layer.** Đọc trực tiếp header của file GGUF
(`gemma4.block_count`) thì model này chỉ có **35 layer**. `-ngl 99` chỉ là quy ước
"yêu cầu nhiều hơn số layer thực có" để llama.cpp tự clamp về full offload — không
phải một con số có ý nghĩa riêng.

**Vì sao không có ngưỡng "hết VRAM"?** Radeon 780M là **iGPU dùng UMA** (unified
memory architecture — thấy rõ trong log Vulkan lúc `serve`: `uma: 1`), tức là GPU
và CPU dùng chung một pool RAM vật lý (23.3 GB) thay vì có VRAM riêng biệt và giới
hạn cứng như GPU rời. Nên "hết VRAM" không phải là ràng buộc ở đây — ràng buộc thật
là **băng thông bộ nhớ dùng chung** giữa CPU và GPU, và cả model 2.97 GB lẫn KV cache
đều nằm thoải mái trong 23.3 GB đó.

**Nhưng đường cong không tuyến tính — và đoạn phi tuyến đó mới là điều đáng chú ý
nhất.** Từ `-ngl 0` đến `-ngl 32` (91% số layer đã offload), tốc độ chỉ tăng
**1.39x** (14.7 → 20.5). Nhưng từ `-ngl 32` lên `-ngl 99` (tức 35, chỉ thêm 3 layer
cuối), tốc độ nhảy thêm **1.61x** nữa (20.5 → 33.1) — mức tăng của 3 layer cuối lớn
hơn cả mức tăng của 32 layer trước đó cộng lại. Cách giải thích khớp với cơ chế
partial offload của llama.cpp: khi còn layer nào đó chạy trên CPU, activation phải
đi qua ranh giới CPU↔GPU ở đúng layer đó **mỗi lượt forward pass** — một lần đồng bộ
và truyền dữ liệu qua PCIe/driver overhead. Chừng nào còn dù chỉ 1 layer trên CPU,
chi phí handoff đó vẫn tồn tại. Chỉ khi **toàn bộ 35/35 layer** đều trên GPU thì
chi phí đó mới biến mất hoàn toàn, nên bước nhảy cuối cùng (offload nốt 3 layer) lại
lớn hơn hẳn phần còn lại của đường cong.

**Một điểm đáng ghi nhận về độ nhiễu của phép đo:** cùng cấu hình `-t 8 -ngl 0`, sweep
này đo được **14.7 tok/s**, trong khi sweep thread ở bước 1.2 (`01-tuning-tg128.md`,
CPU-only) đo cùng điểm đó ra **19.6 tok/s** — chênh **~25%** dù chạy cùng máy, cùng
build, cách nhau chỉ vài phút. Tôi không có lời giải thích chắc chắn cho khoảng chênh
này (có thể là nhiễu lịch trình của Windows, hoặc iGPU chưa "nguội" hoàn toàn giữa hai
lần đo liên tiếp), nên tôi báo cáo nó như một giới hạn của phép đo trên laptop chia sẻ
tài nguyên với OS, chứ không làm tròn hay chọn số đẹp hơn để báo cáo.
