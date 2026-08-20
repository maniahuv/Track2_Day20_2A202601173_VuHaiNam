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

## Your finding (required -- replace this line)

_Is full offload best on your machine? If the curve peaked at a partial value,
what ran out first -- VRAM, or bandwidth between host and device?_
