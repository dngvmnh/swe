# Module 21 — Observability & Operations

> **Core question:** Once your code is running in production, how do you *see* what it's doing, correlate a failure to one request, and operate it safely?
> **Source:** the Mythos crawl (`backend/src/utils/logger.ts`, `middleware/request-id.ts`, `app.ts`, the runbooks, the MYT-101 cost-observability spec).
> **Prerequisites:** [08 — Security](08-security.md), [11 — DevOps/CI-CD](11-devops-cicd.md), [14 — Async jobs](14-async-jobs.md).

## Why this module
Code you can't observe is code you can't operate. When a user reports "my charge failed," you need to find *that request* among millions of log lines, see what it did, and not have leaked their token into the logs while doing it. And beyond logs, operating a credit business means watching **unit economics** (are we losing money on free users?) and having **runbooks** so any engineer can act during an incident. This module is the production-facing layer.

### What you'll be able to do
- Emit structured logs with levels and automatic secret redaction.
- Thread a correlation ID through every log line and response.
- Treat middleware ordering as a correctness/security contract.
- Monitor business/unit-economics metrics as SLOs, not just CPU/memory.
- Write runbooks as executable, copy-paste documentation.

---

## Lesson 21.1 — Structured logging with secret redaction

**Goal:** emit machine-parseable logs that never leak secrets.

### Concept
Use a **structured logger** (Pino) that writes **JSON to stdout** (your platform — CloudWatch/ECS — ingests it), with explicit **levels** (`error/warn/info/debug`). The critical safety feature: **redaction paths** that censor sensitive fields *automatically*, so a stray `logger.info({ req })` can't dump an `Authorization` header or a password. Never `console.*` in production code — you lose structure, levels, and redaction.

### Worked example — `backend/src/utils/logger.ts`
```ts
export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  // pretty in dev, raw JSON in prod (stdout → CloudWatch)
  ...(isProd ? {} : { transport: { target: 'pino-pretty', options: { colorize: true } } }),
  redact: {
    paths: [
      'req.headers.authorization', 'req.headers.cookie',
      '*.password', '*.password_hash', '*.otp',
      '*.token', '*.access_token', '*.refresh_token',
    ],
    censor: '[REDACTED]',          // these fields are auto-scrubbed everywhere they appear
  },
});
```

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — console.log of the whole request: unstructured, no level, leaks the auth header + tokens
console.log('incoming request', req);
```
```ts
// ✅ GOOD — structured, leveled, and redaction strips secrets automatically
logger.info({ req, userId }, 'incoming request');   // authorization/token/password → [REDACTED]
```

### Check yourself
1. Why redact by *path* in the logger instead of remembering to scrub at each call site? (Call-site scrubbing is forgotten exactly once and leaks; central redaction is a guarantee.)
2. Why JSON to stdout rather than writing log files? (12-factor: the platform captures stdout; JSON is queryable in CloudWatch/Datadog. Files are a server you have to manage.)

### ✅ Lesson 21.1 checklist
- [ ] 📖 Read & understood — I can explain structured logging, levels, and path-based redaction.
- [ ] 💻 Applied — I configured a logger with redaction and replaced a `console.log`.
- [ ] 🔍 Found in Mythos — `backend/src/utils/logger.ts` (`redact.paths`); constitution "never `console.*` in new code."

---

## Lesson 21.2 — Correlation IDs (one request, one thread of logs)

**Goal:** tie every log line and the response back to a single request.

### Concept
Generate (or accept) a **request id** per request, attach it to `req`, include it in **every log line**, and return it as a response header (`X-Request-Id`). Now "my charge failed at 3:02pm" becomes a single `reqId` you can grep to see that request's entire journey across services. Accepting an *incoming* `X-Request-Id` lets the id span multiple services (distributed tracing's poor cousin, and often enough).

### Worked example — `request-id.ts` + wiring in `app.ts`
```ts
// middleware/request-id.ts — accept incoming id or mint one; echo it back
export function requestId() {
  return (req, res, next) => {
    const incoming = req.header('X-Request-Id');
    const id = incoming && incoming.length > 0 ? incoming : uuidv4();
    req.id = id;
    res.setHeader('X-Request-Id', id);   // client/caller can quote it in a bug report
    next();
  };
}
// app.ts — every log line carries reqId
app.use(requestId());
app.use(pinoHttp({ logger, customProps: (req) => ({ reqId: req.id }) }));
```

### ✅ Lesson 21.2 checklist
- [ ] 📖 Read & understood — I can explain correlation IDs and why they're returned in a header.
- [ ] 💻 Applied — I added a request-id middleware and stamped it into my log lines.
- [ ] 🔍 Found in Mythos — `middleware/request-id.ts`; `app.ts` `pinoHttp customProps: { reqId }`.

---

## Lesson 21.3 — Middleware ordering is a contract

**Goal:** order cross-cutting middleware so security and correctness hold.

### Concept
The *order* of middleware is not cosmetic — it's a **correctness and security contract**. Some rules are absolute: **helmet first** (headers before anything responds), **error handler last** (it must catch everything downstream), and the **Stripe webhook raw-body route *before* `express.json()`** (signature verification needs the unparsed body — parse it and verification breaks). Reading `app.ts` top-to-bottom *is* the request lifecycle.

### Worked example — `backend/src/app.ts` (numbered, in order)
```ts
app.use(helmet());                       // 1) security headers FIRST
app.use(requestId());                    // 2) correlation id
app.use(pinoHttp({ logger, ... }));      // 3) request logging (now every line has reqId)
app.use(/* Cache-Control: no-store */);  // 4) safe caching defaults
app.post('/api/wallet/stripe/webhook',   // 5) RAW body BEFORE json — signature needs unparsed bytes
  express.raw({ type: 'application/json' }), handleStripeWebhookOnly);
app.use(cors({ origin: whitelist }));     // 6) CORS (origin allow-list, no '*')
app.use(express.json({ limit: MAX_JSON_BODY_BYTES }));  // 7) body parser WITH size limit
app.use('/api', apiLimiter);             // 8) rate limits
app.use('/api/apps', appsRoutes);        // 9) routes
app.use(errorHandler);                   // 10) error handler LAST — catches everything above
```

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — json() before the raw webhook route → Stripe signature verification fails (body already parsed)
app.use(express.json());
app.post('/api/wallet/stripe/webhook', express.raw(...), handleWebhook);   // too late
// ❌ BAD — error handler mounted before routes → it never catches their errors
app.use(errorHandler);
app.use('/api/apps', appsRoutes);
```
```ts
// ✅ GOOD — raw webhook before json(); error handler last
app.post('/api/wallet/stripe/webhook', express.raw(...), handleWebhook);
app.use(express.json({ limit: '1mb' }));
app.use('/api/apps', appsRoutes);
app.use(errorHandler);   // last
```

### Check yourself
1. Why must the Stripe webhook route come before `express.json()`? (Signature verification hashes the *raw* bytes; once `json()` parses+discards the raw body, the hash won't match.)
2. Why is the error handler mounted last? (Express error middleware only catches errors from middleware/routes registered *before* it.)

### ✅ Lesson 21.3 checklist
- [ ] 📖 Read & understood — I can explain helmet-first, error-handler-last, and raw-webhook-before-json.
- [ ] 💻 Applied — I ordered an app's middleware and justified each position.
- [ ] 🔍 Found in Mythos — `backend/src/app.ts` (the numbered middleware stack).

---

## Lesson 21.4 — Monitor unit economics as SLOs

**Goal:** observe the *business*, not just the servers.

### Concept
For a credit/usage business, the metrics that matter aren't only CPU and latency — they're **unit economics**: credits debited, real provider cost, and **gross margin per plan**. The MYT-101 plan treats these as first-class emitted metrics with **alert thresholds** (e.g. free-tier cost/day over budget, margin on a paid tier dropping below a floor). This is observability pointed at "are we making money / is someone abusing the free tier," which CPU graphs never tell you.

### Worked example — MYT-101 plan §7 (Cost Observability)
```
Emit metrics:
  credits_debited_total{plan, kind}
  provider_cost_usd_total{plan, provider, model}
  gross_margin_pct{plan}            where margin = (revenue - cost) / revenue
  blocked_requests_total{reason}    reason ∈ {quota_exceeded, insufficient_funds, rate_limited}

Alert when:
  free-tier cost/day exceeds budget cap
  cost per active free user exceeds target
  margin on Builder or Pro drops below threshold
```

### Check yourself
1. Why label metrics by `plan`/`provider`/`model`? (Dimensions let you slice: "Pro margin is fine but Free is bleeding on model X" — an unlabeled aggregate hides it.)
2. What does `blocked_requests_total{reason}` tell you that a latency graph can't? (Whether users are hitting quota/funds/rate limits — a product + economics signal, not a perf one.)

### ✅ Lesson 21.4 checklist
- [ ] 📖 Read & understood — I can name business/unit-economics metrics and alert thresholds.
- [ ] 💻 Applied — I defined one labeled business metric and an alert condition for a project.
- [ ] 🔍 Found in Mythos — MYT-101 `plan.md` §7 (Cost Observability).

---

## Lesson 21.5 — Runbooks as executable documentation

**Goal:** let any engineer operate the system from copy-paste steps.

### Concept
A **runbook** is step-by-step, **copy-pasteable** instructions for an operational task — verify the API, exercise a flow, validate a migration. Good runbooks are executable (real `curl`/SQL you can paste), start with a **liveness check**, and state **expected output** so you know each step worked. They turn tribal knowledge into something the on-call engineer (or a new hire) can follow at 3am. Reproducible **migration-validation** runbooks (role/extension stubs, `ON_ERROR_STOP`) do the same for schema changes ([Module 03](03-database-migrations.md)).

### Worked example — `backend/REVIEWS_BACKEND_RUNBOOK.md`
```bash
# 1) Confirm API is alive  — always start with a liveness check
curl -i http://localhost:5000/health
# Expected: HTTP/1.1 200 OK and body {"ok":true}

# mint a test token, then exercise the flow step by step with stated preconditions:
#   - token user is the order buyer
#   - order status is 'paid' or 'fulfilled'
#   - no existing review for that order
```

### ❌ Bad → ✅ Good
- ❌ "To test reviews, set up auth and call the review endpoint." (Vague — every reader re-derives it.)
- ✅ A runbook with the exact token-minting command, a `/health` check with expected output, and the precise preconditions for each step.

### ✅ Lesson 21.5 checklist
- [ ] 📖 Read & understood — I can explain what makes a runbook executable (copy-paste, liveness first, expected output).
- [ ] 💻 Applied — I wrote a short runbook for an operational task with expected outputs.
- [ ] 🔍 Found in Mythos — `REVIEWS_BACKEND_RUNBOOK.md`; `docs/runbooks/myt101-docker-migration-validation.md`.

---

## 🎯 Module 21 mastery checklist
- [ ] I can configure structured, leveled logging with automatic secret redaction.
- [ ] I can thread a correlation ID through logs and responses.
- [ ] I can order middleware so security/correctness invariants (helmet-first, handler-last, raw-webhook-before-json) hold.
- [ ] I can define and alert on unit-economics SLOs, not just system metrics.
- [ ] I can write an executable runbook with liveness checks and expected output.

## 🛠️ Mini-project
Operationalize a small API:
1. Add a Pino (or equivalent) logger with redaction paths for `authorization`/`password`/`token`; prove a logged request object scrubs them.
2. Add a request-id middleware that accepts/echoes `X-Request-Id` and stamps `reqId` into every log line. Trace one request end-to-end by its id.
3. Order your middleware deliberately (helmet first, error handler last) and write a comment explaining each position; if you have a webhook, mount its raw-body route before `json()`.
4. Emit one business metric (e.g. `requests_total{route,status}`) and define an alert threshold.
5. Write a `RUNBOOK.md` with a `/health` check (expected output) and the steps to exercise your main flow.

## 🔗 Mythos source map
| File / PR | What it demonstrates |
|-----------|----------------------|
| `backend/src/utils/logger.ts` | Structured Pino logging + redaction paths + levels. |
| `backend/src/middleware/request-id.ts` | Correlation IDs (accept/mint/echo `X-Request-Id`). |
| `backend/src/app.ts` | Middleware ordering as a contract (numbered 1–10). |
| `backend/specs/myt-101-.../plan.md` §7 | Unit-economics SLOs + alert thresholds. |
| `backend/REVIEWS_BACKEND_RUNBOOK.md`, `docs/runbooks/*` | Executable runbooks. |

## See also
- [08 — Security engineering](08-security.md) — redaction, header ordering, CORS, rate limits.
- [11 — DevOps, CI/CD & deployment](11-devops-cicd.md) — where logs/metrics are shipped.
- [18 — Resilience & failure modes](18-resilience-failure-modes.md) — the failures you'll be observing.
