# Serverless

Functions (Lambda, Cloud Functions, Azure Functions) or managed compute (Cloud Run, Fargate)
where the cloud vendor handles provisioning, scaling to zero, and patching — you ship code, not
servers. "Serverless" is a deployment/ops model, not a separate architecture style from the
others; it's usually applied to an event-driven or microservices design.

## Why reach for it

- **Zero idle cost**: scales to zero between invocations — genuinely cheap for spiky or
  low-volume workloads (a webhook handler called a few times a day shouldn't need an always-on
  server).
- **No capacity planning**: the platform scales out per-invocation up to account limits; you
  don't pre-provision for peak load.
- **Fastest path from code to running endpoint**: no cluster, no base image patching, no OS-level
  ops — genuinely faster to ship a small, self-contained piece of logic.

## What it costs

- **Cold starts**: a function with no warm instance pays initialization latency (runtime boot +
  your app's init code) on the first request after idle — can be 100ms-several seconds depending
  on runtime and package size. This tail latency is often unacceptable for user-facing
  synchronous paths with tight SLAs; provisioned concurrency (pre-warmed instances) trades back
  some of the cost savings to fix it.
- **Execution time / resource limits**: hard caps (e.g. Lambda's 15-minute max runtime, memory
  ceilings) rule it out for long-running jobs or heavy in-memory workloads without restructuring
  the work into smaller steps.
- **Vendor lock-in is real**: function signatures, event source integrations, and IAM models
  differ enough across AWS/GCP/Azure that "just redeploy elsewhere" is rarely actually simple.
- **Local dev/debugging friction**: emulating the full event-source + IAM + networking
  environment locally is imperfect (LocalStack, SAM CLI, etc. all have gaps).
- **Cost at high, steady volume can flip against you**: per-invocation pricing that's cheap at
  low volume can exceed the cost of a small always-on fleet once traffic is high and constant —
  model this explicitly rather than assuming serverless is always cheaper.

## Where it's a strong default

- Event handlers with bursty/unpredictable traffic (webhooks, image/file processing triggered by
  upload, scheduled/cron jobs).
- Glue code between managed services (S3 event → transform → write to another store).
- Low-traffic internal tools and admin endpoints where idle cost matters more than latency.

## Staff-engineer notes

- Model cost as a function of traffic shape before committing: plot (requests/month, avg
  duration, memory) against both the serverless per-invocation price and an equivalent
  always-on-fleet price — the crossover point is usually lower than people expect for
  steady-traffic services.
- For latency-sensitive synchronous user-facing paths, treat cold starts as a real SLA risk, not
  a rounding error — either provision concurrency, keep the deployment package/runtime minimal,
  or don't use serverless for that specific path even if the rest of the system is serverless.
- Serverless doesn't remove the need for the rest of this repo's concepts — you still need
  idempotency (a retried invocation must be safe), timeouts/circuit breakers on downstream calls,
  and distributed tracing across function boundaries.
