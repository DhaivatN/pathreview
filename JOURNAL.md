## Week 7 — Issue understanding

**Issue link:** [https://github.com/ascherj/pathreview/issues/86](https://github.com/ascherj/pathreview/issues/86)

**Issue summary:**
The issue is that the API is enforcing rate limits but not exposing rate-limit information to clients until they actually get blocked. Currently, responses do not include headers telling the client what the configured limit is or how many requests remain in the current window, so clients can only discover the limit by hitting a 429. A successful fix will wire the existing `RateLimiter.check_rate_limit()` logic into middleware that runs on every request, then attach `X-RateLimit-Limit` and `X-RateLimit-Remaining` to both successful responses and 429 responses. This will let clients proactively manage their usage and avoid unintentionally crossing the rate limit, while keeping the server-side logic and Redis-based rolling-window behavior unchanged.

**Current broken behavior:**
Successful responses do not expose rate-limit headers, so clients have no visibility into remaining quota until the server rejects a request with 429.

**Successful fix outcome:**
Normal responses and 429 responses both include `X-RateLimit-Limit` and `X-RateLimit-Remaining`, with values consistent with the existing limiter behavior.

## Week 8 — Reproducing the issue and planning the fix

**Reproduction commit link:** <https://github.com/DhaivatN/pathreview/commit/e4c2799f28d09afe0d8c58bfa5ed1901685a2e4b>

**Reproduction summary:**
I inspected the rate-limiting flow across `safety/rate_limiter.py` and `api/middleware/request_id.py`. The existing `RateLimiter.check_rate_limit()` logic already calculates whether a request is allowed and how many requests remain, so the server-side rate-limit state was already available. The gap was in the middleware/response layer: clients were not receiving `X-RateLimit-Limit` and `X-RateLimit-Remaining` on responses. In the current implementation, successful responses now include those headers, but the 429 early-return path still exits before adding them, which confirms the exact issue and shows the remaining work needed to fully satisfy issue #86.

**PLAN.md link:** <https://github.com/DhaivatN/pathreview/blob/fix/86-api-rate-limit-headers/PLAN.md>

**Blockers or open questions:**
The main unknown is how the remaining quota should be represented on blocked requests (e.g., `0` vs another value), and whether existing tests already cover the middleware header behavior or if new tests are needed.

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the request middleware changes so rate-limit headers are attached to both successful responses and 429 responses. Added unit tests in `tests/unit/test_request_id_middleware.py` to verify the headers on allowed and blocked requests.

**Next steps:**
Finalize the PR description, confirm the branch is pushed, and complete the Week 9 submission details in `JOURNAL.md`.

**Blockers:**
None.

---

### Check-in 2 (end of week)

**PR link:** [https://github.com/ascherj/pathreview/pull/233](https://github.com/ascherj/pathreview/pull/233)

**Branch:** `fix/86-api-rate-limit-headers`

**What you built:**
I updated the request middleware so API responses consistently expose `X-RateLimit-Limit` and `X-RateLimit-Remaining` on both successful and blocked requests. The middleware uses the existing `RateLimiter` logic and attaches those headers alongside `X-Request-ID`.

**Tests added or updated:**
Added `tests/unit/test_request_id_middleware.py`, which verifies that both 2xx responses and 429 responses include the request ID and rate-limit headers.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes

**Draft PR feedback received from:** none