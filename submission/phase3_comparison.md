
# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the grader finds your answers. Anything you write outside the markers is ignored, so add extra prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->60.9<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.01<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->8.3<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->10<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and explain what each is spending its time on. Say which comparison is like-for-like, and what the Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes. (~80 words)

<!--answer:timing_explanation-->
The local word count was the fastest at 0.01 seconds, followed by Spark TF-IDF at 8.3 seconds and cloud MapReduce at 60.9 seconds. The local and cloud runs perform the same word count, so they can be compared directly. The cloud version takes longer because of task scheduling, worker initialization and reading and writing files in Cloud Storage. Spark performs a more complex computation with two reduceByKey operations and a join, but it runs locally. This suggests that much of the cloud execution time comes from orchestration and communication overhead.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have, and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce? (~50 words)

<!--answer:counts_interpretation-->
The four Spark stages correspond to three shuffles, caused by the two reduceByKey operations and the join. In the chaos run, I counted 10 retries for 16 original tasks, or 62.5% relative to the initial task count. Across all 26 attempts, 38.5% failed, which is reasonably close to the configured 30% failure probability.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells. Name each RDD operation and say whether it is a transformation or an action. What happened when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
In spark_tfidf.py, flatMap, map, reduceByKey and join are transformations. They create new RDDs without executing the computation immediately. parallelize creates the initial RDD, while toDebugString displays the lineage. When I called build_tfidf_rdd, Spark only built the dependency graph. The collect action triggered the actual computation, ran the stages and returned the TF-IDF rows to the driver. The resulting rows were then saved as Parquet.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the worker holding one output partition of the *first* `reduceByKey` died, what would Spark have to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery starts. (~50 words)

<!--answer:lineage_recovery-->
If a worker lost an output partition from the first reduceByKey, Spark could rebuild it using the RDD lineage. The recovery would involve the first shuffle boundary, ShuffledRDD[3], after stage 0. Spark could reuse shuffle data that was still available and recompute the missing data from upstream partitions if necessary, rather than restarting the entire job.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Google Cloud Workflows (workflow.yaml), which schedules parallel tasks, waits for completion and retries failures.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The mr-worker Cloud Run function, which runs the map and reduce tasks defined in main.py.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage, which holds the input splits, intermediate shuffle files and final outputs.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The partitioning logic in mapreduce.py, which assigns each intermediate key to a reducer partition.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where, exactly, is the barrier between the map and reduce phases — which construct provides it? (~30 words)

<!--answer:barrier_location-->
The barrier is at the end of the parallel map_phase step in workflow.yaml. The parallel-for construct waits until every map branch has finished before the workflow can continue to reduce_phase.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead of overwriting it? (~60 words)

<!--answer:idempotence-->
Retries are safe because each task writes deterministic results to its assigned Cloud Storage object and overwrites existing contents. Running the same task again therefore produces the same output instead of duplicating data. If run_map appended pairs to its shuffle files, retrying a task could add the same intermediate records multiple times. The reducers would then count these duplicates and produce incorrect word frequencies.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3, the Cloud Run request logs showed 10 HTTP 503 failures and 16 successful HTTP 200 responses. Workflows retried the failed attempts and the execution eventually succeeded in 60.9 seconds. The final word count contained 2,221 distinct words and matched the local reference exactly (matches_local: true), showing that the retry mechanism handled the simulated failures.
<!--/answer-->
