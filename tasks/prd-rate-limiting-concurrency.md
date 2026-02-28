# PRD: Rate Limiting & Concurrency Controls

## Introduction

Address rate limiting and concurrency issues discovered during the system audit to prevent resource exhaustion, ensure fair access, and maintain system stability. This PRD covers fixing rate limiter initialization, adding container concurrency limits, proper Redis connection handling, and queue depth controls.

## Goals

- Fix rate limiter initialization to wait for Redis connection
- Add maximum concurrent analysis container limit
- Add queue depth limit to prevent unbounded growth
- Increase database pool size to match worker concurrency
- Add circuit breaker for downstream service failures
- Ensure graceful degradation when rate limited

## User Stories

### US-001: Fix Rate Limiter Redis Initialization
**Description:** As a developer, I want rate limiters to wait for Redis connection so that they work correctly from startup.

**Acceptance Criteria:**
- [ ] Rate limiters created after Redis client connects (not at import time)
- [ ] Factory function creates rate limiters on first request if not initialized
- [ ] If Redis unavailable, fall back to memory store with warning log
- [ ] Health check reports rate limiter status
- [ ] Unit test: verify rate limiter waits for Redis
- [ ] Integration test: rate limiting works after fresh start

### US-002: Add Container Concurrency Limit
**Description:** As an operator, I want a maximum container limit so that the host system isn't overwhelmed.

**Acceptance Criteria:**
- [ ] Add `MAX_CONCURRENT_CONTAINERS` environment variable (default: 5)
- [ ] Track active containers in Redis counter: `depfender:active_containers`
- [ ] Block new container spawn if limit reached
- [ ] Queue waiting requests with timeout (2 minutes)
- [ ] Decrement counter when container exits (success or failure)
- [ ] Counter decremented in finally block to prevent leaks
- [ ] Log when limit reached and when capacity available
- [ ] Unit test: spawn at limit, verify blocked
- [ ] Integration test: concurrent scans respect container limit

### US-003: Add Queue Depth Limit
**Description:** As an operator, I want queue depth limited so that memory doesn't grow unbounded.

**Acceptance Criteria:**
- [ ] Add `MAX_QUEUE_DEPTH` environment variable (default: 1000)
- [ ] Check queue depth before adding new job
- [ ] If limit reached, return 503 Service Unavailable with retry-after header
- [ ] Expose queue depth in `/api/v1/queue/stats` endpoint
- [ ] Add Prometheus gauge: `depfender_queue_depth`
- [ ] Log warning when queue exceeds 80% of limit
- [ ] Unit test: fill queue to limit, verify rejection
- [ ] Integration test: queue stats shows correct depth

### US-004: Increase Database Pool Size
**Description:** As a developer, I want database pool sized for concurrency so that workers don't wait for connections.

**Acceptance Criteria:**
- [ ] Change default pool size from 10 to `QUEUE_CONCURRENCY * 2 + 5` (minimum 15)
- [ ] Add `DB_POOL_SIZE` environment variable for explicit control
- [ ] Add pool exhaustion logging (when waiting > 1s for connection)
- [ ] Add Prometheus gauge: `depfender_db_pool_waiting`
- [ ] Health check reports pool status (available, waiting, total)
- [ ] Unit test: verify pool size calculation
- [ ] Integration test: concurrent operations don't exhaust pool

### US-005: Add Circuit Breaker for Docker
**Description:** As a developer, I want a circuit breaker so that Docker failures don't cascade.

**Acceptance Criteria:**
- [ ] Implement circuit breaker pattern for Docker operations
- [ ] States: CLOSED (normal), OPEN (failing), HALF_OPEN (testing)
- [ ] Open circuit after 5 consecutive failures
- [ ] Stay open for 30 seconds before testing
- [ ] Close circuit after 2 consecutive successes in half-open state
- [ ] Return 503 immediately when circuit is open
- [ ] Log state transitions
- [ ] Prometheus gauge: `depfender_circuit_breaker_state` (0=closed, 1=open, 2=half-open)
- [ ] Unit test: verify state transitions
- [ ] Integration test: Docker failure opens circuit

### US-006: Add Circuit Breaker for Claude API
**Description:** As a developer, I want a circuit breaker for Claude API so that API outages are handled gracefully.

**Acceptance Criteria:**
- [ ] Implement circuit breaker for Anthropic API calls
- [ ] Open circuit after 3 consecutive 5xx errors or timeouts
- [ ] Stay open for 60 seconds before testing
- [ ] Distinguish between rate limits (429) and failures (5xx)
- [ ] Rate limits should not open circuit
- [ ] Log API errors with request ID for debugging
- [ ] Prometheus counter: `depfender_claude_api_errors_total` with label `type: rate_limit|server_error|timeout`
- [ ] Unit test: verify circuit opens on 5xx
- [ ] Integration test: API timeout handled gracefully

### US-007: Add Graceful Rate Limit Response
**Description:** As a user, I want friendly rate limit messages so that I know when to retry.

**Acceptance Criteria:**
- [ ] 429 response includes `Retry-After` header with seconds until reset
- [ ] Response body includes: `{ error: 'rate_limited', retryAfter: 60, message: 'Too many requests...' }`
- [ ] Frontend displays user-friendly "Please wait X seconds" message
- [ ] Frontend auto-retries after delay (optional, with user consent)
- [ ] Different limits have different messages (scan creation vs general API)
- [ ] Typecheck passes
- [ ] Unit test: verify Retry-After header present
- [ ] Verify in browser using dev-browser skill

### US-008: Add Request Timeout Middleware
**Description:** As a developer, I want request timeouts so that slow requests don't hang forever.

**Acceptance Criteria:**
- [ ] Add configurable request timeout middleware (default: 30 seconds)
- [ ] SSE endpoints exempt from timeout (they have their own)
- [ ] Timeout returns 504 Gateway Timeout
- [ ] Log timed out requests with path and duration
- [ ] Add Prometheus counter: `depfender_request_timeouts_total`
- [ ] Unit test: slow handler triggers timeout
- [ ] Integration test: timeout middleware applied to routes

### US-009: Kill Orphaned Containers on Shutdown
**Description:** As an operator, I want orphaned containers cleaned up so that resources aren't leaked.

**Acceptance Criteria:**
- [ ] Track spawned container IDs in memory
- [ ] On SIGTERM/SIGINT, send SIGKILL to all tracked containers
- [ ] Wait up to 10 seconds for containers to stop
- [ ] Force kill any remaining containers
- [ ] Clear active_containers Redis counter on clean shutdown
- [ ] Log orphaned containers killed at shutdown
- [ ] Unit test: verify containers tracked on spawn
- [ ] Integration test: shutdown kills orphaned containers

## Functional Requirements

- FR-1: Rate limiters must be initialized after Redis client connects
- FR-2: System must limit concurrent analysis containers to configurable maximum
- FR-3: System must reject new jobs when queue exceeds configurable depth
- FR-4: Database pool size must be at least `QUEUE_CONCURRENCY * 2 + 5`
- FR-5: Docker operations must use circuit breaker pattern
- FR-6: Claude API calls must use circuit breaker pattern (excluding rate limits)
- FR-7: 429 responses must include Retry-After header
- FR-8: Non-SSE requests must timeout after configurable duration
- FR-9: Orphaned containers must be killed on process shutdown

## Non-Goals

- Distributed rate limiting with sliding window (current fixed window is acceptable)
- Per-user or per-IP rate limiting (only API-level for now)
- Adaptive rate limiting based on system load
- Container resource usage monitoring
- Auto-scaling based on queue depth
- Request prioritization in queue

## Technical Considerations

- **Circuit Breaker Library:** Consider `opossum` npm package or implement simple state machine
- **Redis Counter:** Use `INCR`/`DECR` with TTL for active container tracking
- **Pool Size Formula:** `concurrency * 2` handles workers + API, `+5` handles health checks and overhead
- **Timeout Middleware:** Use `express-timeout-handler` or custom middleware
- **Container Tracking:** Use Map<containerId, timestamp> for in-memory tracking
- **Graceful Shutdown Order:** Stop accepting requests → drain queue → kill containers → close DB

## Success Metrics

- Zero memory store fallback in production (Redis always available)
- Container count never exceeds limit (verified by monitoring)
- Queue depth stays below 80% of limit during normal operation
- Zero database pool exhaustion events
- Circuit breaker activations < 1 per day (indicates upstream issues)
- 429 responses include Retry-After header 100% of time

## Open Questions

- Should we implement adaptive concurrency based on container memory usage?
- Should circuit breaker state be shared across instances (Redis)?
- Should we add per-IP rate limiting for scan creation?
- Should queue depth limit be dynamic based on available memory?
