# PRD: Comprehensive Test Coverage Implementation

## Introduction

Implement comprehensive test coverage across the Depfender platform based on the detailed test plans created for the backend (unit, integration, E2E) and website (component, page, API routes). The primary goal is to increase code quality and reliability by systematically covering all critical code paths with automated tests.

This PRD consolidates the four test plan documents:
- `depfender-dev-backend/docs/UNIT_TEST_PLAN.md` (~345 tests)
- `depfender-dev-backend/docs/INTEGRATION_TEST_PLAN.md` (~29 scenarios)
- `depfender-dev-backend/docs/E2E_TEST_PLAN.md` (~135 tests)
- `depfender-dev-website/docs/TEST_PLAN.md` (~200+ test cases)

## Goals

- Achieve comprehensive test coverage for all backend services, middleware, and workers
- Cover all frontend components, pages, and API routes with unit/integration tests
- Enable confident refactoring by establishing a robust test safety net
- Document testing patterns and conventions for consistency
- Generate coverage reports for visibility (advisory, not enforced)

## User Stories

### Phase 1: Backend Unit Tests - Core Services

---

### US-001: DatabaseService Unit Tests
**Description:** As a developer, I want unit tests for DatabaseService so that database operations are verified in isolation.

**Acceptance Criteria:**
- [ ] Create `tests/unit/database-service.test.ts`
- [ ] Test `createOrGetPackage()` - creates new package, returns existing on conflict
- [ ] Test `createOrGetVersion()` - handles upsert with COALESCE for sha values
- [ ] Test `getScanById()` - returns scan with joins, null for non-existent
- [ ] Test `updateScanStatus()` - sets timestamps for status transitions, invalidates cache
- [ ] Test `createScanWithTransaction()` - rollback on any insert failure
- [ ] Test `storeFindingsWithTransaction()` - batch insert, rollback on failure
- [ ] Test `storeDependenciesBatch()` - batch upsert packages and dependencies
- [ ] Test `getDependencyGraph()` - handles circular refs, respects maxDepth
- [ ] Test `findExistingScan()` - finds pending/running/completed, not failed
- [ ] Mock `db.query()` and `db.getClient()` appropriately
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-002: JobQueueService Unit Tests
**Description:** As a developer, I want unit tests for JobQueueService so that queue operations are verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/job-queue-service.test.ts`
- [ ] Test constructor initializes Redis and creates queues
- [ ] Test `initializeWorkers()` - creates workers, sets up handlers, idempotent
- [ ] Test `enqueueScan()` - checks depth limit, throws QueueFullError when full
- [ ] Test `enqueueDependency()` - creates composite jobId, sets priority by depth
- [ ] Test `getQueueStats()` - calculates depth correctly, updates gauge
- [ ] Test `getQueueDepthStatus()` - returns all fields, isAtLimit logic
- [ ] Test `checkQueueDepthLimit()` - warns at 80%, throws at limit
- [ ] Test `QueueFullError` class properties
- [ ] Mock `Redis`, `Queue`, `Worker` from BullMQ
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-003: ClaudeOrchestrator Unit Tests
**Description:** As a developer, I want unit tests for ClaudeOrchestrator so that AI analysis orchestration is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/claude-orchestrator.test.ts`
- [ ] Test `detectClaudeApiError()` - identifies timeout, rate limit, 5xx, 4xx errors
- [ ] Test `parseAnalysisOutputWithStatus()` - parses JSON wrapper, direct JSON, embedded JSON
- [ ] Test parse failure handling - returns parseFailure=true, creates fallback findings
- [ ] Test severity extraction - maps 'severe' to 'critical', 'warning' to 'medium'
- [ ] Test confidence extraction - converts 0-1 to 0-100, handles strings
- [ ] Test `analyzePackage()` - handles timeout, non-zero exit code
- [ ] Test `orchestrateAnalysis()` - stores findings, re-throws CircuitOpenError
- [ ] Mock `spawnContainer()`, `dbService`
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-004: VerdictCalculator Unit Tests
**Description:** As a developer, I want unit tests for VerdictCalculator so that verdict logic is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/verdict-calculator.test.ts`
- [ ] Test `calculateVerdict()` returns 'safe' with 1.0 confidence for empty findings
- [ ] Test returns 'malicious' for any critical finding
- [ ] Test returns 'malicious' for 2+ high findings
- [ ] Test returns 'suspicious' for 1 high finding
- [ ] Test returns 'suspicious' for 2+ medium findings
- [ ] Test returns 'safe' for only low findings
- [ ] Test confidence increases with agent agreement
- [ ] Test `confidenceToPercentage()` converts 0.85 to 85, handles null
- [ ] Test `percentageToConfidence()` converts 85 to 0.85, handles edge values
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-005: PolicyEngine Unit Tests
**Description:** As a developer, I want unit tests for PolicyEngine so that policy evaluation is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/policy-engine.test.ts`
- [ ] Test constructor uses default policy when none provided
- [ ] Test `evaluatePolicy()` returns correct verdict from matching rule
- [ ] Test falls back to defaultAction when no rules match
- [ ] Test generates stats and reasoning correctly
- [ ] Test `findMatchingRule()` returns highest priority enabled rule
- [ ] Test `ruleMatches()` checks all thresholds correctly
- [ ] Test `calculateConfidence()` applies weights, maps to confidence levels
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-006: ScanWorker Unit Tests
**Description:** As a developer, I want unit tests for ScanWorker so that job processing logic is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/scan-worker.test.ts`
- [ ] Test constructor initializes all services
- [ ] Test `start()` creates Worker with correct concurrency, idempotent
- [ ] Test job processing updates status to 'running', resolves package, runs analysis
- [ ] Test sets 'error' verdict on parse failure
- [ ] Test stores findings and calculates verdict
- [ ] Test `enqueueDependencyScans()` respects limits, skips already-scanned
- [ ] Test `isHealthy()` returns correct state
- [ ] Test `stop()` closes worker, queue, disconnects Redis
- [ ] Mock all services appropriately
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### Phase 1: Backend Unit Tests - Supporting Services

---

### US-007: PackageResolver Unit Tests
**Description:** As a developer, I want unit tests for PackageResolver so that package resolution is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/package-resolver.test.ts`
- [ ] Test `resolveMetadata()` fetches from npm registry, resolves tags and semver
- [ ] Test throws for non-existent package/version
- [ ] Test `downloadAndExtract()` validates URL against allowlist (SSRF prevention)
- [ ] Test `resolve()` full flow with source code retrieval fallback
- [ ] Test `cleanup()` removes directories, handles failure gracefully
- [ ] Mock `fetch()`, `fs`, `tar.extract()`
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-008: ExportService Unit Tests
**Description:** As a developer, I want unit tests for ExportService so that export generation is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/export-service.test.ts`
- [ ] Test `exportScan()` returns correct format (SARIF, JSON, CSV)
- [ ] Test throws for non-existent scan or unsupported format
- [ ] Test `generateSarifReport()` creates valid SARIF 2.1.0 structure
- [ ] Test severity to SARIF level mapping (critical/high -> error)
- [ ] Test `generateJsonExport()` includes metadata, evidence when requested
- [ ] Test `generateCsvExport()` escapes quotes, handles missing fields
- [ ] Mock `dbService`
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-009: RedisService Unit Tests
**Description:** As a developer, I want unit tests for RedisService so that caching and locking are verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/redis-service.test.ts`
- [ ] Test `connect()` connects, returns early if already connected
- [ ] Test `acquireLock()` acquires with SETNX, waits and retries, returns false on timeout
- [ ] Test `releaseLock()` deletes key, doesn't throw on failure
- [ ] Test `cacheScanResult()` stores JSON with TTL
- [ ] Test `getCachedScanResult()` returns parsed JSON or null
- [ ] Test `waitForContainerSlot()` waits when at limit, returns false on timeout
- [ ] Test `incrementActiveContainers()` / `decrementActiveContainers()` with Lua script
- [ ] Mock Redis client
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-010: DependencyResolver Unit Tests
**Description:** As a developer, I want unit tests for DependencyResolver so that dependency parsing is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/dependency-resolver.test.ts`
- [ ] Test `resolveDependencies()` returns empty for missing package.json
- [ ] Test parses production and dev dependencies
- [ ] Test handles malformed package.json
- [ ] Test skips invalid version values
- [ ] Test `getProductionDependencies()` filters to 'prod' kind
- [ ] Mock `fs.existsSync()`, `fs.readFileSync()`
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-011: GitRepositoryService Unit Tests
**Description:** As a developer, I want unit tests for GitRepositoryService so that git operations are verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/git-repository-service.test.ts`
- [ ] Test `cloneRepository()` creates directory, clones, checkouts commit
- [ ] Test fetches more history when commit not found
- [ ] Test throws on invalid repo URL or commit SHA
- [ ] Test `cleanup()` removes directory, logs warning on failure
- [ ] Test `parseGitUrl()` parses HTTPS, SSH, SSH alternate formats
- [ ] Test `isRepositoryAccessible()` returns true/false correctly
- [ ] Mock `simpleGit`, `fs`
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### Phase 1: Backend Unit Tests - Middleware & Utils

---

### US-012: Validation Middleware Unit Tests
**Description:** As a developer, I want unit tests for validation middleware so that input validation is verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/validation-middleware.test.ts`
- [ ] Test `isValidNpmPackageName()` rejects invalid names, accepts valid
- [ ] Test `isValidVersionString()` rejects path traversal, accepts semver
- [ ] Test `hasPathTraversal()` detects basic, URL encoded, double encoded
- [ ] Test `validate()` middleware validates body/query/params/all
- [ ] Test returns 400 with details on validation failure
- [ ] Test Zod schemas validate required fields, apply defaults
- [ ] Mock Request, Response, NextFunction
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-013: Metrics Helpers Unit Tests
**Description:** As a developer, I want unit tests for metrics helpers so that observability functions are verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/metrics-helpers.test.ts`
- [ ] Test `trackScanCacheHit()` / `trackScanCacheMiss()` increment counters
- [ ] Test `trackScanDedupNew()` / `trackScanDedupExisting()` / `trackScanDedupRace()`
- [ ] Test `updateQueueDepth()` sets gauge
- [ ] Test `trackClaudeApiError()` increments with type label
- [ ] Test `trackRequestTimeout()` normalizes path
- [ ] Test `trackScanStarted()` / `trackScanCompleted()` / `trackScanFailed()`
- [ ] Test `getMetrics()` returns Prometheus format
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-014: TaskRunner Unit Tests
**Description:** As a developer, I want unit tests for TaskRunner so that container operations are verified.

**Acceptance Criteria:**
- [ ] Create `tests/unit/task-runner.test.ts`
- [ ] Test `spawnContainer()` creates container, enforces timeout, returns output
- [ ] Test container killed on timeout, timedOut=true
- [ ] Test container tracking functions (register, unregister, getCount)
- [ ] Test `killTrackedContainers()` sends SIGKILL, force removes
- [ ] Test `Task.invoke()` waits for slot, throws on slot timeout
- [ ] Test `ensureImageExists()` throws when image missing
- [ ] Mock Docker client
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### Phase 2: Backend Integration Tests

---

### US-015: Queue-to-Worker Flow Integration Tests
**Description:** As a developer, I want integration tests for job processing flow so that the complete scan pipeline is verified.

**Acceptance Criteria:**
- [ ] Create `tests/integration/queue-worker-flow.test.ts`
- [ ] Test job enqueue to completion flow with status transitions
- [ ] Test job retry on transient failure with exponential backoff
- [ ] Test job failure after max retries, scan status 'failed'
- [ ] Test queue depth limit enforcement, 503 response
- [ ] Test graceful worker shutdown completes in-progress jobs
- [ ] Requires running PostgreSQL, Redis, Docker
- [ ] All tests pass with `npm run test:integration`
- [ ] Typecheck passes

---

### US-016: SSE Stream Integration Tests
**Description:** As a developer, I want integration tests for SSE streaming so that real-time progress is verified.

**Acceptance Criteria:**
- [ ] Create `tests/integration/sse-stream.test.ts`
- [ ] Test SSE connection for pending, running, completed, failed scans
- [ ] Test heartbeat comments sent every 15 seconds
- [ ] Test progress events received during processing
- [ ] Test complete/error events sent appropriately
- [ ] Test connection timeout after SSE_CONNECTION_TIMEOUT_MS
- [ ] Test client disconnect cleanup (QueueEvents closed)
- [ ] Requires running PostgreSQL, Redis
- [ ] All tests pass with `npm run test:integration`
- [ ] Typecheck passes

---

### US-017: Circuit Breaker Integration Tests
**Description:** As a developer, I want integration tests for circuit breaker so that Claude API resilience is verified.

**Acceptance Criteria:**
- [ ] Create `tests/integration/circuit-breaker.test.ts`
- [ ] Test circuit opens on consecutive 5xx failures
- [ ] Test circuit does NOT trip on rate limits (429)
- [ ] Test circuit half-open test request after timeout
- [ ] Test circuit closes on successful request
- [ ] Test metrics tracking for API error types
- [ ] Requires mock Claude container
- [ ] All tests pass with `npm run test:integration`
- [ ] Typecheck passes

---

### US-018: Transaction Conflict Integration Tests
**Description:** As a developer, I want integration tests for transaction edge cases so that concurrent operations are verified.

**Acceptance Criteria:**
- [ ] Create `tests/integration/transaction-conflicts.test.ts`
- [ ] Test concurrent package upsert race - only one record created
- [ ] Test concurrent version upsert with different metadata - COALESCE behavior
- [ ] Test connection pool exhaustion recovery
- [ ] Test long-running transaction warning
- [ ] Requires running PostgreSQL
- [ ] All tests pass with `npm run test:integration`
- [ ] Typecheck passes

---

### US-019: Error Propagation Integration Tests
**Description:** As a developer, I want integration tests for error handling so that error consistency is verified.

**Acceptance Criteria:**
- [ ] Create `tests/integration/error-propagation.test.ts`
- [ ] Test database connection failure during scan creation
- [ ] Test Redis failure during lock acquisition
- [ ] Test queue service failure during enqueue - scan rolled back
- [ ] Test worker failure propagates to API with correct status
- [ ] Test export service failure handling
- [ ] Requires controlled infrastructure failures
- [ ] All tests pass with `npm run test:integration`
- [ ] Typecheck passes

---

### US-020: Export Endpoints Integration Tests
**Description:** As a developer, I want integration tests for export endpoints so that all formats are verified.

**Acceptance Criteria:**
- [ ] Create `tests/integration/export-endpoints.test.ts`
- [ ] Test JSON export with all options (evidence, policy, metadata)
- [ ] Test SARIF 2.1.0 export validates against schema
- [ ] Test CSV export escapes special characters correctly
- [ ] Test export formats endpoint returns all formats
- [ ] Requires running PostgreSQL with test data
- [ ] All tests pass with `npm run test:integration`
- [ ] Typecheck passes

---

### Phase 3: Frontend Component Tests

---

### US-021: StatsBar Component Tests
**Description:** As a developer, I want unit tests for StatsBar so that statistics display is verified.

**Acceptance Criteria:**
- [ ] Create `src/components/__tests__/StatsBar.test.tsx`
- [ ] Test renders all four stat sections
- [ ] Test calculates verdicts correctly from scans array
- [ ] Test uses fallbackTotal when scans empty
- [ ] Test formats large numbers with locale string
- [ ] Test applies correct CSS classes for each verdict
- [ ] Test handles empty scans array and null verdicts
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-022: ScanTable Component Tests
**Description:** As a developer, I want unit tests for ScanTable so that scan list display is verified.

**Acceptance Criteria:**
- [ ] Create `src/components/__tests__/ScanTable.test.tsx`
- [ ] Test renders table headers correctly
- [ ] Test renders loading state with spinner
- [ ] Test renders empty state when scans empty
- [ ] Test renders correct number of rows
- [ ] Test package links to correct scan detail page
- [ ] Test source links open in new tab
- [ ] Test time formatting for all ranges (just now, min, hours, days, weeks)
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-023: StatusBadge Component Tests
**Description:** As a developer, I want unit tests for StatusBadge so that status display is verified.

**Acceptance Criteria:**
- [ ] Create `src/components/__tests__/StatusBadge.test.tsx`
- [ ] Test renders "Scanning" with spinner for running/pending
- [ ] Test renders correct text and icon for each verdict
- [ ] Test renders "Analysis Error" for error verdict and parse_error status
- [ ] Test renders "Unknown" for completed without verdict
- [ ] Test applies correct CSS class for each state
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-024: HeroSection Component Tests (Expansion)
**Description:** As a developer, I want expanded unit tests for HeroSection so that form behavior is verified.

**Acceptance Criteria:**
- [ ] Expand `src/components/__tests__/HeroSection.test.tsx`
- [ ] Test renders main title and subtitle
- [ ] Test input accepts package name and GitHub URL
- [ ] Test form submission with valid input navigates to scan page
- [ ] Test empty/whitespace input prevented
- [ ] Test loading state shows spinner, disables input/button
- [ ] Test generic error message for non-rate-limit errors
- [ ] Test keyboard accessibility (Enter submits)
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-025: Footer Component Tests
**Description:** As a developer, I want unit tests for Footer so that footer links are verified.

**Acceptance Criteria:**
- [ ] Create `src/components/__tests__/Footer.test.tsx`
- [ ] Test renders Depfender description
- [ ] Test Documentation link points to correct GitHub URL
- [ ] Test Discussions link points to correct URL
- [ ] Test Twitter/X link points to correct URL
- [ ] Test all external links have target="_blank" and rel="noopener noreferrer"
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### Phase 3: Frontend Page Tests

---

### US-026: Home Page Integration Tests
**Description:** As a developer, I want integration tests for the home page so that data loading is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/__tests__/page.test.tsx`
- [ ] Test renders all main components (header, HeroSection, StatsBar, ScanTable, Footer)
- [ ] Test fetches queue stats and recent scans on mount
- [ ] Test handles API errors gracefully
- [ ] Test sets up 30-second polling interval
- [ ] Test clears polling on unmount
- [ ] Test updates display when data changes
- [ ] Mock apiClient methods
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-027: Scan Detail Page Tests
**Description:** As a developer, I want tests for scan detail page so that all states are verified.

**Acceptance Criteria:**
- [ ] Create `src/app/scan/[id]/__tests__/page.test.tsx`
- [ ] Test loading state with progress indicator
- [ ] Test error state for invalid ID or failed request
- [ ] Test completed scan displays verdict banner, findings, summary cards
- [ ] Test correct styling for safe/suspicious/malicious verdicts
- [ ] Test "Copy Report Link" copies URL to clipboard
- [ ] Test SSE progress updates reflected in UI
- [ ] Test manual retry button when disconnected
- [ ] Mock apiClient and EventSource
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-028: Error Page Tests
**Description:** As a developer, I want tests for error page so that error handling UI is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/__tests__/error.test.tsx`
- [ ] Test renders error icon and title
- [ ] Test displays error digest when provided
- [ ] Test "Try Again" button calls reset function
- [ ] Test "Go Home" link navigates to home
- [ ] Test logs error to console on mount
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-029: Not Found Page Tests
**Description:** As a developer, I want tests for 404 page so that navigation is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/__tests__/not-found.test.tsx`
- [ ] Test renders 404 heading and description
- [ ] Test search input accepts text
- [ ] Test search button disabled when empty
- [ ] Test form submission navigates with search query
- [ ] Test "Go Home" link navigates to home
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### Phase 3: Frontend API Route Tests

---

### US-030: POST /api/v1/scans Route Tests
**Description:** As a developer, I want tests for scan creation route so that request handling is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/api/v1/scans/__tests__/route.test.ts`
- [ ] Test returns 400 when package name missing
- [ ] Test returns 201 with scan data on success
- [ ] Test passes ecosystem, version, display_name to backend
- [ ] Test returns 500 when backend fails
- [ ] Test backend receives x-internal-secret header
- [ ] Mock proxyJsonToBackend
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-031: GET /api/v1/scans/[id] Route Tests
**Description:** As a developer, I want tests for scan retrieval route so that response handling is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/api/v1/scans/[id]/__tests__/route.test.ts`
- [ ] Test returns 400 when ID missing
- [ ] Test returns 200 with scan details on success
- [ ] Test returns 500 when backend fails
- [ ] Test correctly extracts ID from params
- [ ] Mock proxyJsonToBackend
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-032: GET /api/v1/scans/[id]/stream Route Tests
**Description:** As a developer, I want tests for SSE stream route so that proxy behavior is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/api/v1/scans/[id]/stream/__tests__/route.test.ts`
- [ ] Test returns 500 when env vars not configured
- [ ] Test returns proper SSE headers
- [ ] Test proxies SSE data from backend
- [ ] Test closes stream when backend ends
- [ ] Test handles client disconnect (abort signal)
- [ ] Test handles backend connection error
- [ ] Mock fetch and ReadableStream
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-033: GET /api/v1/scans/recent Route Tests
**Description:** As a developer, I want tests for recent scans route so that pagination is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/api/v1/scans/recent/__tests__/route.test.ts`
- [ ] Test returns default limit of 5
- [ ] Test accepts custom limit from query
- [ ] Test returns 400 for invalid limit (non-numeric, <1, >100)
- [ ] Test returns 200 with array on success
- [ ] Test returns 500 when backend fails
- [ ] Mock proxyJsonToBackend
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-034: GET /api/v1/queue/stats Route Tests
**Description:** As a developer, I want tests for queue stats route so that response is verified.

**Acceptance Criteria:**
- [ ] Create `src/app/api/v1/queue/stats/__tests__/route.test.ts`
- [ ] Test returns 200 with stats on success
- [ ] Test returns 500 when backend fails
- [ ] Test response includes pending, completed_today, recent_completions
- [ ] Mock proxyJsonToBackend
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### Phase 3: Frontend Library Tests

---

### US-035: Backend Client Tests
**Description:** As a developer, I want tests for backend-client so that proxy utilities are verified.

**Acceptance Criteria:**
- [ ] Create `src/lib/__tests__/backend-client.test.ts`
- [ ] Test throws error when BACKEND_URL missing
- [ ] Test throws error when INTERNAL_API_SECRET missing
- [ ] Test `proxyToBackend()` adds headers, constructs URL
- [ ] Test `proxyJsonToBackend()` returns parsed JSON
- [ ] Test throws error on non-OK response with status info
- [ ] Mock environment variables
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### US-036: API Client Expansion Tests
**Description:** As a developer, I want expanded tests for api.ts so that all methods are verified.

**Acceptance Criteria:**
- [ ] Expand `src/lib/__tests__/api-client.test.ts`
- [ ] Test `createScan()` sends POST, handles rate limit, throws errors
- [ ] Test `getScan()` sends GET, returns ScanDetails
- [ ] Test `getQueueStats()` sends GET, returns QueueStats
- [ ] Test `getRecentScans()` sends GET with limit, returns array
- [ ] Test `createScanProgressStream()` returns EventSource with correct URL
- [ ] All tests pass with `npm test`
- [ ] Typecheck passes

---

### Phase 4: Test Infrastructure & Documentation

---

### US-037: Test Data Factories
**Description:** As a developer, I want test data factories so that test data is consistent.

**Acceptance Criteria:**
- [ ] Create `tests/factories/scan.ts` for backend
- [ ] Create `src/__tests__/factories/scan.ts` for frontend
- [ ] Factory functions for Package, Version, Scan, Finding, Evidence
- [ ] Factory functions for ScanResponse, ScanDetails, QueueStats
- [ ] Support for overrides parameter
- [ ] Use UUID generation for IDs
- [ ] All factories properly typed

---

### US-038: Test Utilities
**Description:** As a developer, I want shared test utilities so that common patterns are reusable.

**Acceptance Criteria:**
- [ ] Create `tests/utils/database.ts` for backend - seeding, cleanup helpers
- [ ] Create `tests/utils/queue.ts` for backend - job tracking helpers
- [ ] Create `tests/utils/sse.ts` for backend - SSE client helper
- [ ] Create `src/__tests__/utils/mocks.ts` for frontend - common mocks
- [ ] Document usage in test files

---

### US-039: Coverage Reporting Setup
**Description:** As a developer, I want coverage reporting configured so that I can see coverage metrics.

**Acceptance Criteria:**
- [ ] Configure vitest coverage in backend `vitest.config.ts`
- [ ] Configure vitest coverage in website `vitest.config.ts`
- [ ] Generate text, JSON, and HTML reports
- [ ] Add `npm run test:coverage` scripts
- [ ] Document how to view coverage reports in README or test plans
- [ ] Coverage runs successfully and generates reports

---

## Functional Requirements

- FR-1: All unit tests must mock external dependencies (database, Redis, Docker, fetch)
- FR-2: All integration tests must check infrastructure availability and skip gracefully
- FR-3: All frontend tests must mock next/navigation and API client
- FR-4: Test files must be co-located with source code in `__tests__/` directories
- FR-5: Test naming convention: `{component-name}.test.ts(x)`
- FR-6: All tests must pass typecheck before merge
- FR-7: Coverage reports generated in text, JSON, and HTML formats
- FR-8: Test data factories must support override parameters for flexibility

## Non-Goals

- No CI/CD pipeline integration (manual runs only)
- No enforcement of coverage thresholds
- No visual regression testing
- No performance/load testing beyond what's in E2E plan
- No browser-based E2E testing (Playwright/Cypress)

## Technical Considerations

- **Backend Testing Framework:** Vitest 4.x with supertest for HTTP testing
- **Frontend Testing Framework:** Vitest 4.x with React Testing Library and jsdom
- **Mocking:** Use `vi.mock()` for module mocking, `vi.fn()` for function mocks
- **Test Isolation:** Each test should be independent, use `beforeEach` for setup
- **Async Testing:** Use `async/await` pattern, avoid callbacks
- **Infrastructure Tests:** Use `beforeAll` to check availability, skip if unavailable

## Success Metrics

- All ~345 backend unit tests implemented and passing
- All ~29 backend integration scenarios implemented and passing
- All ~100+ frontend component/page tests implemented and passing
- Coverage reports available via `npm run test:coverage`
- No flaky tests (deterministic outcomes)
- Test execution time: Unit < 60s, Integration < 5min

## Open Questions

1. Should we add snapshot testing for React components?
2. Should we create a mock Claude container for integration tests?
3. Should we add mutation testing (Stryker) for test quality validation?
4. Should we implement visual regression testing later (Chromatic/Percy)?
