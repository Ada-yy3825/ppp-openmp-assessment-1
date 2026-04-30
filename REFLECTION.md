# A1 REFLECTION

> Complete every section. CI will:
>
> 1. Verify all `## Section` headers below are present.
> 2. Verify each section has **at least 50 words**.
>
> No automatic content grading: the prose is read by a human, and the short
> prompt at the end is marked on a 0 / 0.5 / 1 scale. The numbers you quote
> in your reflection do **not** have to match canonical times exactly — HPC
> queue variance is real. Be concise, ground claims in your measurements, show your working.

## Section 1 — Schedule choice and why

Which schedule (`static` / `dynamic` / `guided` / chunk size) did you end up with, and why? Reference the cost structure of `f(x)` and what the measured timings told you. Mention at least one schedule you tried and discarded, and what the measured evidence was. Minimum 50 words.

I selected `schedule(guided)` for the final implementation because it gave the best measured performance on the Rome node, especially at high thread counts. The cost of `f(x)` is deliberately non-uniform, with an expensive spike region between `x = 0.3` and `x = 0.4`, so static scheduling caused load imbalance. I also tested `dynamic,64`, which performed well locally but measured about 0.099 s at 128 threads on Rome, while `guided` reached about 0.045 s. This showed that guided scheduling provided a better balance between reducing load imbalance and avoiding excessive scheduling overhead.

## Section 2 — Scaling behaviour

Looking at your `tables.csv`, where does your speedup curve depart from ideal (linear)? What does that tell you about overhead, memory bandwidth, or load balance for this kernel? Minimum 50 words.

The speedup curve scaled well initially but departed from ideal linear scaling as thread count increased. At 16 threads, the measured speedup was about 8.3×, while at 128 threads it reached about 42.9× instead of the ideal 128×. Efficiency also decreased from about 0.52 to 0.33. This suggests that parallel overheads became more significant at higher thread counts. Although guided scheduling reduced load imbalance from the expensive spike region in `f(x)`, synchronization, thread management, and memory-system effects still limited scalability.

## Section 3 — Roofline position

Pick your best thread count. Using the Rome roofline constants from the day-2 slides (theoretical peak 4608 GFLOPs, HPL-achievable 2896 GFLOPs, STREAM triad 246 GB/s), what roofline fraction did you achieve against the *theoretical* and the *HPL-achievable* compute ceilings? Most non-DGEMM code (including A1) lands well below both — explain why your kernel doesn't approach DGEMM-class efficiency. If you want to argue your kernel is bandwidth-bound rather than compute-bound, justify it. Minimum 50 words.

Using a conservative operation-equivalent estimate, I counted about 6 operations for a light iteration, including loop arithmetic and the basic arithmetic in `f(x)`. In the heavy region `0.3 < x < 0.4`, each iteration performs ten extra `sqrt(abs(y) + 1.0)` steps, giving about 36 operations for a heavy iteration. Since the heavy region covers about 10% of the domain, the total work is roughly `90,000,000 × 6 + 10,000,000 × 36 = 0.9 GFLOPs`. With my best 128-thread time of 0.0446 s, this gives about 20.2 GFLOPs/s, or around 0.44% of the theoretical peak and 0.70% of the HPL-achievable peak.

## Section 4 — What you'd try next
 
You have two more days. What would you change about `integrate.cpp`? Pick one concrete change and predict its effect. Minimum 50 words.

If I had more time, I would experiment with different chunk sizes for guided and dynamic scheduling to further reduce scheduling overhead at high thread counts. Although `schedule(guided)` performed best overall, there may still be opportunities to improve cache locality and thread utilization by tuning how iterations are distributed. I would also profile the runtime more carefully on the Rome node to identify whether synchronization overhead or memory-system effects become the dominant bottleneck at 128 threads. This could potentially improve scaling efficiency further.

## Reasoning question (instructor-marked, ≤100 words)

**In at most 100 words, explain why your chosen schedule is appropriate for the cost structure of this particular `f(x)`.**

`f(x)` has a deliberately non-uniform cost structure because the region between `x = 0.3` and `x = 0.4` performs ten additional nested square-root operations. This creates load imbalance when using static scheduling, since some threads receive significantly more expensive iterations than others. I chose `schedule(guided)` because it dynamically redistributes work while gradually reducing chunk size, improving load balance without introducing as much scheduling overhead as very small dynamic chunks. The measured timings on the Rome node showed that guided scheduling achieved the best scalability and lowest runtime, especially at high thread counts.
