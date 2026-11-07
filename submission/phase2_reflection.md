# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->243.047<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.999<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->428242.5<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12993.9<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5017<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2268.489 ms; 2 threads: 2067.609 ms; 4 threads: 2295.831 ms; 8 threads: 2300.662 ms. 2 threads gave the lowest time.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
With the naive version, there are lots of Python loops doing small calculations one by one. This takes time even before considering the actual maths. The vectorized version sends matrix operations to NumPy and OpenBLAS instead. These libraries are optimized and can use SIMD instructions, so several values can be processed together. In my run, the time went from 243.047 ms to 0.999 ms. The improvement isn't only due to SIMD, since avoiding Python loops also helps.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
I got the best time with 2 threads: 2067.609 ms per step, compared to 2268.489 ms with just 1. I expected 4 or 8 threads to be faster, but the times actually increased to 2295.831 and 2300.662 ms. The notebook mentions a 2-vCPU Colab environment, so asking for more threads doesn't mean there are more CPU cores available. The threads also need to share resources and coordinate their work, which can end up costing more time than it saves.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
The GPU was much faster for this model: 12993.9 ms for 100 steps, while the CPU took 428242.5 ms. A transformer does many matrix multiplications during training, and the GPU can do a lot of these calculations in parallel. However, I wouldn't expect the same improvement for a very small task, such as processing a tiny matrix once. Moving data to the GPU and launching the operations also takes time, which might be more than the actual computation.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
What I noticed is that parallelism helps, but only up to a point. For attention, the vectorized code took 0.999 ms instead of 243.047 ms. It was much faster because the work was done in optimized matrix operations rather than lots of Python loops. But some work, like setting up operations, still has to happen, so the gain isn't unlimited.

For CPU threading, 2 threads were best at 2067.609 ms per step. With 4 threads it went up to 2295.831 ms. Since the notebook describes the Colab runtime as having 2 vCPUs, the extra threads probably spent more time competing for CPU resources and coordinating than helping.

The GPU comparison was different: 100 steps took 12993.9 ms on GPU instead of 428242.5 ms on CPU. Here there was enough matrix computation to benefit from the GPU. Still, data transfers, launching GPU operations and parts that must run in sequence can limit the benefit, especially for smaller workloads.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would try the threading experiment again with different batch sizes, repeating each measurement a few times. I'd like to see if 2 threads are still the best when the amount of work changes. If 4 threads became consistently faster for larger batches on the same runtime, my explanation for the slowdown would need to be reconsidered.
<!--/answer-->
