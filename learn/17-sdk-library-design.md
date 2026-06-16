# Module 17 — SDK & Library Design

> **Core question:** How do you design a library that *other people* integrate — one that's secure by default, caches expensive work, fails clearly, and survives the dependency it's built on?
> **Source:** the Mythos Python SDK crawl (`mythos-sdk/packages/python/mythos_sdk/*`).
> **Prerequisites:** [04 — Auth](04-auth.md) (JWT/JWKS), [06 — Error handling](06-error-handling.md), [07 — API design](07-api-design.md) (layering).

## Why this module
Writing an *application* and writing a *library* are different jobs. A library is a **public contract** other engineers depend on: your function names, your error types, and your defaults are now an API you can't casually change. The Mythos SDK lets a creator's app verify a launch token and meter usage in three calls — and every design choice (what's exported, how JWKS is cached, what happens on a network blip) is a lesson in library design.

### What you'll be able to do
- Design a small, intentional public surface over private internals.
- Cache an expensive remote resource (JWKS) with a TTL and a targeted refresh.
- Verify JWTs safely: algorithm allow-listing and full audience validation.
- Map transport errors to typed domain errors so callers can branch on meaning.
- Configure via environment (12-factor) with fail-fast, and ship a capability handshake.
- Recognize fragile couplings (string-matching a dependency's error text) and pin against them.

---

## Lesson 17.1 — A thin public facade over private internals

**Goal:** expose the few things integrators need; hide everything else.

### Concept
A library's `__init__` / index is its **public API** — keep it tiny and intentional. Internals (transport, caching, verification) stay private modules that you can refactor freely. Mythos exposes exactly **six names**; the `jwks_cache`, `api_client`, and `config` modules are implementation detail.

### Worked example — `mythos_sdk/__init__.py`
```python
from .errors import InsufficientFundsError, MythosError, SessionNotFoundError
from .handshake import handshake_router
from .middleware import require_launch_token
from .report_usage import report_usage
from .types import MythosSession
from .verify import verify_launch_token

__all__ = [
    "verify_launch_token", "require_launch_token", "report_usage",
    "handshake_router", "MythosSession",
    "MythosError", "InsufficientFundsError", "SessionNotFoundError",
]
```
Note the composition: `report_usage` is a one-line wrapper over the private `meter_session` transport — the public verb (`report_usage`) is stable even if the internal call changes.

### ❌ Bad → ✅ Good
```python
# ❌ BAD — no __all__, everything importable; integrators reach into api_client/jwks_cache
#          and now your "internal" function signatures are a contract you can't change.
from mythos_sdk.jwks_cache import _fetch_jwks   # users will do this if you let them
```
```python
# ✅ GOOD — a curated __all__; internals are free to change
from mythos_sdk import verify_launch_token, report_usage, MythosSession
```

### ✅ Lesson 17.1 checklist
- [ ] 📖 Read & understood — I can explain why a small public surface protects future refactors.
- [ ] 💻 Applied — I curated an `__all__` / index exporting only the intended API.
- [ ] 🔍 Found in Mythos — `mythos_sdk/__init__.py` (`__all__`), `report_usage.py` wrapping `meter_session`.

---

## Lesson 17.2 — Cache an expensive remote resource (JWKS) with a TTL

**Goal:** don't refetch the public keys on every request, but recover when they rotate.

### Concept
Verifying a token needs the issuer's JWKS — a network fetch. Doing that per request is slow and hammers the issuer. **Cache it with a TTL**, measured on a **monotonic clock** (immune to wall-clock jumps), and expose a **forced refresh** for the one case where the cache is legitimately stale: a **`kid` miss** after the issuer rotated keys ([Module 16](16-cryptography-key-management.md)).

### Worked example — `mythos_sdk/jwks_cache.py`
```python
CACHE_TTL = 600  # 10 minutes
_cache: dict | None = None
_fetched_at: float = 0.0

def _is_stale() -> bool:
    return _cache is None or time.monotonic() - _fetched_at > CACHE_TTL   # monotonic, not time.time()

async def get_jwks(api_url: str, force_refresh: bool = False) -> dict:
    if not force_refresh and not _is_stale() and _cache is not None:
        return _cache                      # fast path: serve from cache
    return await _fetch_jwks(api_url)      # fetch + repopulate

async def get_jwks_with_kid_fallback(api_url: str) -> dict:
    return await get_jwks(api_url, force_refresh=True)   # used only on a kid miss
```

### Check yourself
1. Why `time.monotonic()` instead of `time.time()`? (Wall-clock can jump backwards (NTP, DST); monotonic only moves forward, so TTL math is correct.)
2. When is a forced refresh justified — every verify failure? (No — only a *kid miss* (new key, stale cache). Bad-signature/expired tokens should NOT trigger a refetch — Lesson 17.3.)

### ✅ Lesson 17.2 checklist
- [ ] 📖 Read & understood — I can explain TTL caching, monotonic time, and targeted refresh.
- [ ] 💻 Applied — I cached a remote resource with a TTL + a force-refresh path.
- [ ] 🔍 Found in Mythos — `jwks_cache.py` (`_is_stale`, `get_jwks`, `get_jwks_with_kid_fallback`).

---

## Lesson 17.3 — Verify JWTs safely: allow-list algorithms, validate every audience

**Goal:** avoid the two classic JWT verification holes.

### Concept
- **Algorithm allow-listing.** Pass an explicit `algorithms=["RS256"]`. If you don't, a library may accept a token whose header says `alg: none` (no signature) or `HS256` signed with your *public* key (algorithm-confusion). The allow-list closes both.
- **Validate the *whole* audience.** A token's `aud` can be a single string *or* a list. Checking only `aud[0]` is a real bug (you reject valid multi-listing tokens, or worse). Check **membership across all elements**.
- **Exception triage before retry.** On failure, decide *which* failures are worth a JWKS refetch. A **bad signature is terminal** — refetching keys won't help and wastes a round trip; only a **kid miss** warrants a refetch.

### Worked example — `mythos_sdk/verify.py`
```python
ALGORITHMS = ["RS256"]   # allow-list: rejects alg:none and HS256 confusion

def _validate_audience(payload, listing_ids: list[str]) -> None:
    aud = payload.get("aud")
    if aud is None: raise JWTClaimsError("Missing audience claim")
    aud_values = aud if isinstance(aud, list) else [aud]     # normalize str | list
    if not any(a in listing_ids for a in aud_values):        # check ALL elements
        raise JWTClaimsError("Invalid audience")

async def verify_launch_token(token: str) -> MythosSession:
    config = load_config()
    jwks = await get_jwks(config.api_url)
    try:
        payload = jwt.decode(token, jwks, algorithms=ALGORITHMS, issuer=config.api_url, options={"verify_aud": False})
    except JWTError as e:
        if isinstance(e, (ExpiredSignatureError, JWTClaimsError)): raise   # terminal
        if "Signature verification failed" in str(e): raise                # terminal — don't refetch
        jwks = await get_jwks_with_kid_fallback(config.api_url)            # only here: kid miss → refetch once
        payload = jwt.decode(token, jwks, algorithms=ALGORITHMS, issuer=config.api_url, options={"verify_aud": False})
    _validate_audience(payload, config.listing_ids)
    return _build_session(payload)
```

### ❌ Bad → ✅ Good
```python
# ❌ BAD — no algorithms list (accepts alg:none); only checks aud[0]
jwt.decode(token, jwks)                       # algorithm-confusion / alg:none hole
if payload["aud"][0] in listing_ids: ...      # breaks for list-valued or reordered aud
```
```python
# ✅ GOOD — explicit allow-list + membership over all aud values
jwt.decode(token, jwks, algorithms=["RS256"], issuer=config.api_url, ...)
if any(a in listing_ids for a in aud_values): ...
```

### Check yourself
1. What attack does `algorithms=["RS256"]` prevent? (`alg:none` (unsigned tokens) and HS256/RS256 algorithm-confusion.)
2. Why not refetch JWKS on *every* verify failure? (A bad signature or expired token is terminal — refetching keys can't fix it and adds latency; reserve refetch for kid misses.)

### ✅ Lesson 17.3 checklist
- [ ] 📖 Read & understood — I can explain algorithm allow-listing and full-audience validation.
- [ ] 💻 Applied — I verified a JWT with an explicit algorithm list and rejected an `alg:none` token.
- [ ] 🔍 Found in Mythos — `verify.py` (`ALGORITHMS`, `_validate_audience`, the triage in `verify_launch_token`).

---

## Lesson 17.4 — Map transport errors to typed domain errors

**Goal:** let callers branch on *meaning*, not on HTTP status codes.

### Concept
An SDK shouldn't make every integrator learn your status codes. At the transport boundary, **translate HTTP statuses into typed domain errors** that carry machine-readable codes. The caller catches `InsufficientFundsError`, not `if resp.status_code == 402`.

### Worked example — `errors.py` + `api_client.py`
```python
# errors.py — a small typed hierarchy, each carrying a stable code
class MythosError(Exception):
    def __init__(self, message, code): super().__init__(message); self.code = code
class InsufficientFundsError(MythosError):
    def __init__(self): super().__init__("Insufficient funds in wallet", "INSUFFICIENT_FUNDS")
class SessionNotFoundError(MythosError):
    def __init__(self, jti): super().__init__(f"Session not found: {jti}", "SESSION_NOT_FOUND")

# api_client.py — translate status → typed error at the boundary
async def meter_session(jti, credits, reason=None):
    resp = await client.post(f"{config.api_url}/api/apps/sessions/{jti}/meter", json=body)
    if resp.status_code == 402: raise InsufficientFundsError()
    if resp.status_code == 404: raise SessionNotFoundError(jti)
    resp.raise_for_status()
```
This is the SDK mirror of the backend's typed-error hierarchy ([Module 06](06-error-handling.md)) — the *contract* survives across the network.

### ✅ Lesson 17.4 checklist
- [ ] 📖 Read & understood — I can explain translating HTTP status into typed domain errors at the boundary.
- [ ] 💻 Applied — I mapped two status codes to typed errors a caller can `catch`.
- [ ] 🔍 Found in Mythos — `errors.py` hierarchy; `api_client.py` `meter_session` 402/404 mapping.

---

## Lesson 17.5 — 12-factor config (fail-fast) + a capability handshake

**Goal:** configure from the environment, and let the platform discover your SDK.

### Concept
- **Config from env, fail-fast.** Read settings from environment variables; accept both **single and multi** forms; and **throw immediately** if a required value is missing rather than failing mysteriously on first use.
- **Capability handshake.** Ship a well-known endpoint that advertises the SDK version, so the platform can detect integration and version-skew.

### Worked example — `config.py` + `handshake.py`
```python
def load_config() -> MythosConfig:
    api_url = os.environ.get("MYTHOS_API_URL", DEFAULT_API_URL)
    multi  = os.environ.get("MYTHOS_LISTING_IDS")
    single = os.environ.get("MYTHOS_LISTING_ID")
    if multi:    listing_ids = [l.strip() for l in multi.split(",") if l.strip()]
    elif single: listing_ids = [single]
    else:        listing_ids = []
    if not listing_ids:                                  # fail-fast, clear message
        raise RuntimeError("MYTHOS_LISTING_ID or MYTHOS_LISTING_IDS env var is required")
    return MythosConfig(listing_ids=listing_ids, api_url=api_url)
```
```python
# handshake.py — a versioned capability endpoint the platform can probe
SDK_VERSION = "0.1.0"
@handshake_router.get("/.well-known/mythos-handshake")
async def mythos_handshake():
    return JSONResponse({"ok": True, "sdk_version": SDK_VERSION})
```

### ✅ Lesson 17.5 checklist
- [ ] 📖 Read & understood — I can explain 12-factor env config, fail-fast, and a capability handshake.
- [ ] 💻 Applied — I wrote a config loader that fails fast on a missing required var.
- [ ] 🔍 Found in Mythos — `config.py` (`load_config`), `handshake.py` (`/.well-known/mythos-handshake`).

---

## Lesson 17.6 — Fragile couplings: string-matching a dependency, and pinning against it

**Goal:** recognize when you've coupled to a dependency's *incidental* behavior, and contain the risk.

### Concept
Sometimes a library gives you no typed way to distinguish two failures, so you're forced to **match on its error *message string*** — which can change between versions. That's a real coupling. The honest way to handle it: **pin the dependency version** so the string stays stable, and **document the coupling** at the call site. (Bonus lesson from the SDK audit: a dynamically-typed `audience` parameter "accepted anything" but then rejected *every* token — proof that "accepts anything" isn't the same as "works," and why typed contracts and adversarial tests matter — [Module 10](10-testing.md).)

### Worked example — `verify.py` (the documented coupling)
```python
# python-jose 3.x raises a BARE JWTError with this message for signature failures.
# Version pinned <4 in pyproject.toml so this text stays stable.
# A bad signature is not a kid miss — fail immediately, don't re-fetch JWKS.
if "Signature verification failed" in str(e):
    raise
```
```toml
# pyproject.toml — the pin that keeps the string above stable
"python-jose[cryptography]>=3.3,<4",
```

### Check yourself
1. Why is matching on `str(e)` risky, and what makes it *acceptable* here? (The message is incidental and could change on upgrade — acceptable only because the version is pinned `<4` and the coupling is documented.)
2. The Python SDK once "accepted any audience type" but rejected every token. What's the lesson? (Loose/dynamic typing hides hard rejections; a real type contract + an adversarial test would have caught it. "Accepts anything" ≠ "works".)

### ✅ Lesson 17.6 checklist
- [ ] 📖 Read & understood — I can explain message-string coupling and why pinning + documenting contains it.
- [ ] 💻 Applied — I found (or wrote) a dependency coupling and pinned/documented it.
- [ ] 🔍 Found in Mythos — `verify.py` signature-string check + the `python-jose <4` pin in `pyproject.toml`; the audience-type bug in `mythos-sdk-demo/FINDINGS.md` (F1).

---

## 🎯 Module 17 mastery checklist
- [ ] I can design a small, intentional public API over private internals.
- [ ] I can cache a remote resource with a TTL on a monotonic clock and refresh it only when warranted.
- [ ] I can verify JWTs with an algorithm allow-list and full audience validation.
- [ ] I can translate transport errors into typed domain errors at the boundary.
- [ ] I can load config from the environment with fail-fast and ship a capability handshake.
- [ ] I can spot a fragile dependency coupling and contain it with a pin + a comment.

## 🛠️ Mini-project
Write a tiny verification SDK for a mock issuer:
1. Public API: `verify(token)`, `report(jti, n)`, plus a typed error hierarchy — curate `__all__`.
2. Fetch the issuer's JWKS and cache it with a 10-minute TTL (monotonic clock) + a force-refresh used only on a `kid` miss.
3. `verify` allow-lists `RS256`, validates the full `aud`, and does *not* refetch on a bad signature. Add an adversarial test: an `alg:none` token must be rejected.
4. Map `402`/`404` to typed errors; config from env with fail-fast; add a `/.well-known/<your>-handshake` advertising a version.

## 🔗 Mythos source map
| File / PR | What it demonstrates |
|-----------|----------------------|
| `mythos_sdk/__init__.py` | Curated public surface (`__all__`); facade over internals. |
| `mythos_sdk/jwks_cache.py` | TTL cache, monotonic clock, targeted force-refresh. |
| `mythos_sdk/verify.py` | Algorithm allow-list, full-audience check, exception triage, documented dep coupling. |
| `mythos_sdk/errors.py` + `api_client.py` | Typed domain errors mapped from HTTP status. |
| `mythos_sdk/config.py` + `handshake.py` | 12-factor config with fail-fast; capability handshake. |
| `mythos-sdk-demo/FINDINGS.md` | The audience-type bug and the fault-injection audit. |

## See also
- [16 — Cryptography & key management](16-cryptography-key-management.md) — the JWKS this SDK consumes.
- [06 — Error handling & typed errors](06-error-handling.md) — the pattern, mirrored client-side.
- [18 — Resilience & failure modes](18-resilience-failure-modes.md) — what the SDK middleware does on a network blip.
- [19 — Integration & protocol design](19-integration-protocol-design.md) — the handshake this SDK takes part in.
