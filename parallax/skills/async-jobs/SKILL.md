---
name: async-jobs
description: How to handle Parallax tools that submit a job and wait for it server-side before returning.
---

# Async Jobs

Eight Parallax tools submit a job and the MCP server waits for it internally. Each call waits up to 45 seconds. A job that finishes in that time returns inline in the same call. A longer job returns its `job_id` with `status: "running"`; pick it up with `check_job_status`.

| Tool | Typical job time | Notes |
|---|---|---|
| `get_stock_report` | ~1-2 min | |
| `get_etf_report` | ~1-2 min | |
| `get_financial_analysis` | ~2-5 min | |
| `get_assessment` | ~3 min | Always the deep tier |
| `analyze_portfolio` | ~1-3 min | |
| `get_news_synthesis` | ~30-90s, instant if cached | |
| `get_technical_analysis` | ~15-30s | |
| `build_stock_universe` | ~15-40s | Check the original job first; a narrower retry only after a failure, as a new billed query (5 credits) — see Failure and Timeout Handling |

`get_telemetry` reads synchronously; callers see an inline result. If it ever returns a `job_id` with `status: "running"`, handle it the same way and do not re-call it.

## Pattern

Call the tool **once**. If the job finished within 45 seconds, use the result directly. If the reply is `status: "running"`, follow the check budget below.

## Check budget

`check_job_status` costs 0 credits. It waits up to 45 seconds per call; while the job is still running it returns after about 30 seconds.

Spend up to 9 `check_job_status` calls per job. With the first 45-second wait, this covers about 5 minutes in total, the same as the old server wait. Space the checks: do other work between them where the workflow has other work, rather than calling back to back. A `running` reply is not a timeout and not a failure.

## Failure and Timeout Handling

**Never re-call the original async tool**: a second call can start and bill a new job. Upstream reuses a job for an identical request only within 5 minutes, and never for `build_stock_universe`, `get_assessment`, or reports called with `force=true`.

Identify each reply by its fields, never by its error text:

| Reply | Meaning | Action |
|---|---|---|
| Tool reply with `success: false`, a `job_id` and `status: "running"` | Still running | The one exception to the rule that `success: false` ends the section. Call `check_job_status` with that `job_id`. |
| `check_job_status` with `success: true`, `status` of `pending`, `queued`, `running` or `processing`, and a `next_step` | Not done | Counts as one check. Check again within the budget. |
| `check_job_status` with `success: false`, a `job_id` and no `status`, error saying the job status could not be read | Status unknown | Counts as one check. Call again. |
| `status: "completed"` with a `result` | Done | Use the `result` field. |
| `status: "failed"`, a completed job with no result, or a job not found or expired | Terminal | Render "Analysis unavailable — <short reason>." Do not re-call. |
| Any other `success: false` from the original tool (no `status: "running"`) | Terminal | Render "Analysis unavailable — <short reason>." Do not re-call. |

A job still running after the budget renders as "Analysis pending — ask me to check job <job_id> later today." The server keeps a job for 24 hours after its last update, so a later `check_job_status` call with the same `job_id` can still pick it up.

**`build_stock_universe` is the one designed exception.** On a "still running" reply, follow the check budget on the returned `job_id`. If it is still running after the budget, report the `job_id` and offer to check later; never start a second job while the first runs. Only if the job failed, run one narrower query. That narrower query is a new billed call (5 credits); say so, and run it at most once.

`check_job_status`'s own description says the same: async tools already poll internally, and it should only be called if a tool returned a `job_id` without a result.

## Parallelism and Latency

Issue independent async calls in one parallel batch, so their server-side waits overlap. Never chain async calls serially; chaining adds the waits together.

A parallel batch returns only when its slowest call returns, up to 45 seconds. "Never blocks the rest of the output" therefore means: an async failure or a still-running job never stops the other sections from rendering. It does not mean instant results arrive before the async ones in the same batch. When a command says to start a later batch "without waiting" for an async tool, put that async tool in its own batch alongside the later work, not in the first batch with the instant identity calls.

## Caching

Many tools cache results. If the same symbol was run recently, the result returns immediately.
