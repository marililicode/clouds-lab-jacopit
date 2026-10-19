# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
49.98948
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->237.3<!--/answer--> | <!--answer:vm_warm_median_ms-->210.6<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1796.0<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->632.9<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1943.6<!--/answer--> | <!--answer:function_warm_median_ms-->636.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```text
VM          cold=  237.3ms  warm_median=  210.6ms
            #############
Cloud Run   cold= 1796.0ms  warm_median=  632.9ms
            ########################################
Function    cold= 1943.6ms  warm_median=  636.5ms
            ########################################
```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The IaaS deployment does not create additional resources automatically: every request is handled by the single e2-micro VM, so its capacity is limited by the CPU and memory of the machine I selected. Cloud Run automatically creates container instances as load increases and scales back to zero when idle; I explicitly capped it at three instances with --max=3. The gen-2 Cloud Function also creates managed instances automatically and can scale to zero. I did not configure an explicit maximum instance count for the Function, so its limit comes from the managed service configuration and platform quotas rather than from a limit I set myself.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM required the most manual work: I created a custom VPC and subnet, configured separate firewall rules for SSH and port 8080, created an e2-micro Debian VM, granted its service account access to Artifact Registry, installed and authenticated Docker, pulled the image and started the container myself. Cloud Run was simpler because I only had to deploy the existing container image, expose port 8080, allow unauthenticated access and configure scaling. For the Cloud Function, I deployed the Python source using Python 3.11 and the echo entry point, while Google handled the build, containerization and managed runtime.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```text
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-5980566790.us-central1.run.app
LAST DEPLOYED BY: service-5980566790@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-19T15:39:44.076608Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-5980566790.us-central1.run.app
LAST DEPLOYED BY: marie.jacopit@eurecom.fr
LAST DEPLOYED AT: 2026-10-19T15:22:18.061832Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows two Cloud Run services: echo-bot, which I deployed directly, and echo, which was deployed by the Cloud Functions administration service account. This shows that the gen-2 Cloud Function ultimately runs on Cloud Run infrastructure. That explains why their latency profiles are so similar: Cloud Run had a 1796.0 ms cold start and a 632.9 ms warm median, while the Function had a 1943.6 ms cold start and a 636.5 ms warm median. Both can scale to zero, so after being idle they need to start a new runtime instance. The main difference is the deployment abstraction: for Cloud Run I supplied my own container image, whereas for the Function I supplied Python source and Google built and deployed the underlying managed service.
<!--/answer-->