# 14 — Async Jobs & Scheduling

## The idea

Not all work fits inside a request/response. Some work is **slow** (sending email, processing
video), some is **deferred** (retry later), and some is **periodic** (renew subscriptions every
hour). Async jobs move that work *off* the request path so the user gets a fast response, and
schedulers run work *on a clock* with no user present at all.

Two shapes:
- **Background jobs / queues** — enqueue work now, a worker processes it later, out of band.
- **Scheduled jobs / cron** — work triggered by time (every hour, every night), independent of any
  request.

## Why it exists

If you do slow or periodic work inside a request handler, you block the request (timeouts, bad UX)
and you have nothing to drive work that isn't triggered by a user (a nightly cleanup, a renewal).
Decoupling work from requests lets the system stay responsive, absorb spikes (the queue buffers),
retry failures, and perform maintenance autonomously.

## Patterns & trade-offs

- **Producer → queue → worker.** The request enqueues a message and returns immediately; a separate
  worker consumes it. The queue decouples arrival rate from processing rate (back-pressure) and
  survives worker restarts.
- **Idempotent jobs.** A queue delivers *at least once*; a cron may overlap or double-fire. So every
  job must be safe to run twice — dedup on a stable key (a message id, or a synthetic key derived
  from the work). This is [doc 01](01-idempotency-and-exactly-once-semantics.md) applied to jobs.
- **Synthetic idempotency keys for periodic work.** A scheduled grant has no client to mint a key,
  so derive one deterministically from *what* and *when*: `cron_<entity>_<period>`. Two runs in the
  same period produce the same key → no duplicate effect.
- **Secured trigger endpoints.** A cron that's just an HTTP endpoint must be protected by a shared
  secret so the public can't trigger it.
- **Batch resilience.** Process rows independently; log a per-row failure and continue rather than
  aborting the whole batch on one bad record.
- **Safety-net design.** When two mechanisms can do the same work (a webhook *and* a cron), make the
  slower one a backstop that excludes anything the faster one already handled, so they don't
  double-act.
- **Observability.** Jobs run unattended — log successes at `info`, failures with context, and make
  them traceable.

Trade-off: async adds moving parts (a queue, a worker, monitoring) and *eventual* consistency — the
effect isn't immediate. Don't reach for it until the work genuinely doesn't belong in the request.

## Pitfalls

- **Non-idempotent jobs** → double-credit/double-send on the inevitable retry or overlap.
- **Unsecured cron endpoints** → anyone can trigger privileged work.
- **One bad row aborts the batch** → a single failure halts everything.
- **Doing slow work inline** → blocked requests and timeouts.
- **Two mechanisms double-acting** (webhook + cron both renew) without mutual exclusion.
- **Silent job failures** → unattended work fails and nobody knows.

## Seen in Mythos

- **The `subscription_renew` cron — `BE#86` (MYT-103c).** A near-complete checklist of this doc's
  patterns:
  - *Secured trigger:* `POST /jobs/subscription-renew/run`, protected by `CRON_JOB_SECRET`.
  - *Targeted query:* selects `user_subscriptions WHERE current_period_end <= now()`.
  - *Idempotency via synthetic key:* tags each renewal `cron_<subscription_id>_<period_epoch>`
    stored as `stripe_invoice_id`, so concurrent or repeated runs can't double-grant credits.
  - *Safety-net design:* paid tiers are excluded if Stripe webhooks already advanced the period —
    the cron only backstops what the faster webhook missed (no double-action).
  - *Batch resilience + observability:* logs successful renewals at `info`, logs per-row failures
    **without aborting the batch**.

- **Idempotent jobs proven by test — `BE#70`.** The wallet suite explicitly tests
  "`subscription_renew` behavior and idempotency" — the job's safe-to-rerun property is pinned.

- **Email as out-of-band work — `BE#20`, `BE#25`, constitution.** Email flows for various events
  (`BE#20`) with HTML templates (`BE#25`); the constitution forbids sending email inline in a route
  handler and requires templates to be **pure functions** returning `{ subject, html }` — email is
  treated as async, side-effecting work pushed out of the request path.

- **Queue-first architecture as policy — constitution.** *"Heavy async work → SQS → worker process.
  Never block the request cycle. SQS jobs must be idempotent — use checkout session ID or UUID as
  deduplication key."* The producer→queue→worker + idempotent-job model, written down. (The repo
  even carries `AWS_PREVIEW_SQS_URL` config, and `BE#38` typed the `cron-utils` domain.)

## See also

- [01 — Idempotency & exactly-once](01-idempotency-and-exactly-once-semantics.md)
- [09 — Payments & billing architecture](09-payments-and-billing-architecture.md) (renewal/grants)
- [11 — DevOps, CI/CD & deployment](11-devops-cicd-and-deployment.md)
