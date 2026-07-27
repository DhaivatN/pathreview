## Solution plan

**Issue:** #86 — Add rate-limit headers to API responses  
https://github.com/ascherj/pathreview/issues/86

### Understand
The backend already enforces rate limits and calculates remaining quota, but clients do not consistently receive that information through HTTP response headers. Successful responses now include `X-RateLimit-Limit` and `X-RateLimit-Remaining`, but the 429 early-return path still does not attach those headers, so the implementation is incomplete.

### Map
The likely files and modules involved are:
- `api/middleware/request_id.py` — current middleware that checks the limiter and attaches response headers
- `safety/rate_limiter.py` — existing Redis-backed rolling-window logic that returns whether a request is allowed and how many requests remain
- Middleware registration path (for example `main.py`) — to confirm the middleware is wired into every API request
- Relevant test files under `tests/` — to verify both successful and 429 responses include the required headers

### Plan
1. Inspect the 429 early-return path in `api/middleware/request_id.py` and refactor it so blocked responses also receive `X-Request-ID`, `X-RateLimit-Limit`, and `X-RateLimit-Remaining`.
2. Keep `safety/rate_limiter.py` unchanged unless needed, since it already returns the limiter state required by middleware.
3. Verify that both successful responses and 429 responses expose consistent header values.
4. Review or add tests under `tests/` to cover both the allowed-request path and rate-limit-exceeded path.
5. Run `make check` and `make test-unit` before final PR submission.

### Inputs & outputs
**Inputs:**
- Incoming HTTP request
- Client identifier (currently IP address)
- Existing `RateLimiter.check_rate_limit()` result: `(allowed, remaining_requests)`

**Outputs:**
- Successful responses with `X-RateLimit-Limit` and `X-RateLimit-Remaining`
- 429 responses with the same rate-limit headers
- Existing request ID header behavior preserved
- No change to Redis rolling-window enforcement logic

### Risks & unknowns
- `api/middleware/request_id.py` currently returns a `JSONResponse` early for blocked requests, so header logic must be carefully applied to avoid skipping required headers.
- The remaining value on blocked requests may need confirmation to ensure it matches the intended contract (`0` vs another derived value).
- I still need to confirm whether existing tests already cover middleware headers or whether I need to add new ones.

### Edge cases
- The final allowed request in the window should return a normal response with `X-RateLimit-Remaining: 0`.
- A blocked request should return 429 and still include both rate-limit headers.
- Repeated rapid requests from the same client should show remaining counts decreasing consistently.
- Missing client connection information should not break the middleware and should fall back safely.