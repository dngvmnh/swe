# Module 14 — Async Jobs & Scheduling

> **Core question:** How do you do work outside the request/response cycle?
> **Reference doc:** [../14-async-jobs-and-scheduling.md](../14-async-jobs-and-scheduling.md)
> **Prerequisites:** [01 — Idempotency](01-idempotency.md) (applied to jobs); pairs with [09 — Payments](09-payments.md).

## Why this module
If you do slow or periodic work inside a request handler, you block the request (timeouts, bad UX)
and you have nothing to drive work that isn't triggered by a user (a nightly cleanup, a renewal).
This module teaches you to move work *off* the request path and run work *on a clock.*

### What you'll be able to do
- Decide what belongs in a background job vs the request.
- Build a producer → queue → worker pipeline.
- Make jobs idempotent (including with synthetic keys for periodic work).
- Secure trigger endpoints, make batches resilient, and design safety nets.

---

## Lesson 14.1 — Why async (slow / deferred / periodic)

**Goal:** recognize work that doesn't belong in a request.

### Concept
Two shapes of out-of-band work:
- **Background jobs / queues** — enqueue work now, a worker processes it later, out of band.
- **Scheduled jobs / cron** — work triggered by **time** (every hour, nightly), no user present.

Decoupling work from requests lets the system stay responsive, absorb spikes (the queue buffers),
retry failures, and perform maintenance autonomously.

> **Trade-off:** async adds moving parts (a queue, a worker, monitoring) and **eventual
> consistency** — the effect isn't immediate. Don't reach for it until the work genuinely doesn't
> belong in the request.

### ✅ Lesson 14.1 checklist
- [ ] 📖 Read & understood — I can classify work as request-bound, queued, or scheduled.
- [ ] 💻 Applied — I identified slow work in a handler that should be moved off the request path.
- [ ] 🔍 Found in Mythos — email is treated as out-of-band work; the constitution forbids sending email inline in a route.

---

## Lesson 14.2 — Producer → queue → worker

**Goal:** build the canonical pipeline.

### Concept
The request **enqueues a message and returns immediately**; a separate **worker** consumes it. The
queue decouples arrival rate from processing rate (**back-pressure**) and survives worker restarts.

```ts
// producer (in the request) — enqueue and return fast
await sqs.send({ type: "process_video", videoId, dedupKey: videoId });
return res.json({ success: true, data: { status: "queued" } });

// worker (separate process) — consume and process out of band
worker.on("process_video", async (msg) => { await transcode(msg.videoId); });
```

### ✅ Lesson 14.2 checklist
- [ ] 📖 Read & understood — I can explain producer/queue/worker and back-pressure.
- [ ] 💻 Applied — I sketched a queue-backed flow for slow work.
- [ ] 🔍 Found in Mythos — constitution: "Heavy async work → SQS → worker process. Never block the request cycle."

---

## Lesson 14.3 — Idempotent jobs & synthetic keys

**Goal:** survive at-least-once delivery and overlapping cron runs.

### Concept
A queue delivers **at least once**; a cron may **overlap or double-fire.** So **every job must be
safe to run twice** — dedup on a stable key (a message id, or a **synthetic key** derived from the
work). This is [Module 01](01-idempotency.md) applied to jobs.

```ts
// periodic grant has no client → derive a deterministic key from WHAT + WHEN
const key = `cron_${subscriptionId}_${Math.floor(periodEnd.getTime() / 1000)}`;
// stored in a UNIQUE column → a second run in the same period is blocked (23505 → skip)
```

### ✅ Lesson 14.3 checklist
- [ ] 📖 Read & understood — I can explain at-least-once delivery and synthetic keys.
- [ ] 💻 Applied — I made a job idempotent with a message id or synthetic key.
- [ ] 🔍 Found in Mythos — `BE#86` tags each renewal `cron_<subscription_id>_<period_epoch>`; `BE#70` tests `subscription_renew` idempotency.

---

## Lesson 14.4 — Secured trigger endpoints

**Goal:** stop the public from triggering privileged work.

### Concept
A cron that's just an HTTP endpoint must be **protected by a shared secret** so the public can't
trigger it.

```ts
// ❌ BAD — anyone who finds the URL can run the renewal job
app.post("/jobs/subscription-renew/run", runRenewal);
// ✅ GOOD — gate on a shared secret
app.post("/jobs/subscription-renew/run", (req, res, next) => {
  if (req.headers["x-cron-secret"] !== process.env.CRON_JOB_SECRET) return res.sendStatus(401);
  next();
}, runRenewal);
```

### ✅ Lesson 14.4 checklist
- [ ] 📖 Read & understood — I can explain why cron endpoints need a shared secret.
- [ ] 💻 Applied — I gated a trigger endpoint on a secret.
- [ ] 🔍 Found in Mythos — `BE#86` `POST /jobs/subscription-renew/run` protected by `CRON_JOB_SECRET`.

---

## Lesson 14.5 — Batch resilience & observability

**Goal:** don't let one bad row halt the batch.

### Concept
- **Batch resilience.** Process rows **independently**; log a per-row failure and **continue** rather
  than aborting the whole batch on one bad record.
- **Observability.** Jobs run unattended — log successes at `info`, failures with context, make them
  traceable. **Silent job failures = unattended work fails and nobody knows.**

```ts
// ✅ one bad row doesn't kill the batch
for (const sub of dueSubscriptions) {
  try { await renew(sub); logger.info("renewed", { id: sub.id }); }
  catch (e) { logger.error("renew failed", { id: sub.id, err: e }); /* continue */ }
}
```

### ✅ Lesson 14.5 checklist
- [ ] 📖 Read & understood — I can explain per-row resilience and job observability.
- [ ] 💻 Applied — I wrote a batch loop that logs and continues on per-row failure.
- [ ] 🔍 Found in Mythos — `BE#86` logs successful renewals at `info` and per-row failures without aborting the batch.

---

## Lesson 14.6 — Safety-net design (two mechanisms, no double-action)

**Goal:** combine a fast path and a backstop without double-acting.

### Concept
When **two mechanisms can do the same work** (a webhook *and* a cron), make the slower one a
**backstop that excludes anything the faster one already handled**, so they don't double-act.

> Mythos: paid-tier renewals are handled by Stripe webhooks (fast); the hourly cron only renews rows
> the webhook **hasn't** already advanced — it backstops missed webhooks, never duplicates them.

### ✅ Lesson 14.6 checklist
- [ ] 📖 Read & understood — I can design a backstop that excludes already-handled work.
- [ ] 💻 Applied — I described a fast-path + backstop pair with mutual exclusion.
- [ ] 🔍 Found in Mythos — `BE#86`: cron excludes paid tiers already advanced by Stripe webhooks.

---

## 🎯 Module 14 mastery checklist
- [ ] I can decide what work belongs off the request path.
- [ ] I can build a producer → queue → worker pipeline with back-pressure.
- [ ] I can make jobs idempotent, including synthetic keys for periodic work.
- [ ] I can secure a trigger endpoint with a shared secret.
- [ ] I can write resilient, observable batch jobs.
- [ ] I can design a safety-net backstop that never double-acts.

## 🛠️ Mini-project
Build a "daily digest" job:
1. A secured `POST /jobs/digest/run` (shared-secret gated).
2. It selects due users, sends each a digest **independently** — log + continue on per-row failure.
3. Make it **idempotent** with a synthetic key `digest_<userId>_<yyyy-mm-dd>` so running it twice in a
   day sends once. Test the replay.
4. Bonus: add a queue (or in-memory stand-in) so the request enqueues and a worker processes.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#86` | The full cron checklist: secured trigger, targeted query, synthetic-key idempotency, safety-net, batch resilience, observability. |
| `BE#70` | `subscription_renew` idempotency pinned by test. |
| `BE#20`, `BE#25` | Email as out-of-band work + HTML templates (pure functions). |
| `BE#38` | Typed the `cron-utils` domain. |

## See also
- [01 — Idempotency & exactly-once](01-idempotency.md)
- [09 — Payments & billing](09-payments.md) — renewal grants.
- [11 — DevOps, CI/CD](11-devops-cicd.md)
