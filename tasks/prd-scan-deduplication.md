# PRD: Scan Deduplication & Race Condition Fix

## Introduction

Address the race condition in scan creation that allows duplicate scans when concurrent requests arrive for the same package/version. Implement distributed locking to ensure exactly one scan is created, with other requests receiving the existing scan ID.

## Goals

- Prevent duplicate scans for the same package@version
- Implement distributed locking using Redis SETNX
- Return existing scan ID when duplicate request detected
- Handle lock expiration gracefully
- Maintain performance for non-duplicate requests

## User Stories

### US-001: Implement Redis Distributed Lock
**Description:** As a developer, I need a distributed locking mechanism so that concurrent scan requests are serialized.

**Acceptance Criteria:**
- [ ] Create `acquireLock(key, ttlMs)` function using Redis SETNX
- [ ] Create `releaseLock(key, token)` function that only releases if token matches
- [ ] Lock key format: `lock:scan:{ecosystem}:{package}:{version}`
- [ ] Default TTL: 30 seconds (prevents deadlocks)
- [ ] Returns unique token on success, null on failure
- [ ] Unit tests for acquire, release, expiration
- [ ] Integration test: concurrent lock attempts, only one succeeds

### US-002: Add Lock to Scan Creation Flow
**Description:** As a developer, I want scan creation to acquire a lock so that duplicate scans are prevented.

**Acceptance Criteria:**
- [ ] Scan creation attempts to acquire lock before database check
- [ ] If lock acquired: proceed with creation, release lock after commit
- [ ] If lock not acquired: wait briefly (100ms), then check for existing scan
- [ ] Lock released in finally block (even on error)
- [ ] Timeout after 5 seconds of waiting, return 429 Too Many Requests
- [ ] Unit test: simulate concurrent requests, verify one scan created
- [ ] Integration test: parallel scan requests, all receive same scan ID

### US-003: Check for Existing Scan Before Creation
**Description:** As a developer, I want to check for existing scans so that we return existing results instead of duplicating work.

**Acceptance Criteria:**
- [ ] Query for existing scan with matching package, version, ecosystem
- [ ] Only match scans in status: pending, processing, or completed (not failed)
- [ ] If exists: return 200 with existing scan ID (not 201)
- [ ] If not exists: create new scan, return 201
- [ ] Response includes `existed: true|false` field
- [ ] Unit test: create scan, request same package, verify same ID returned
- [ ] Integration test: existing completed scan, new request returns it

### US-004: Handle Lock Expiration Gracefully
**Description:** As a developer, I want lock expiration handled so that deadlocks don't block the system.

**Acceptance Criteria:**
- [ ] If lock expires during creation, log warning and continue
- [ ] Use atomic check-and-delete for lock release (Lua script)
- [ ] If creation fails after lock expiry, handle potential duplicate gracefully
- [ ] Database unique constraint serves as final safeguard
- [ ] Log lock expiration events for monitoring
- [ ] Unit test: simulate slow creation exceeding TTL
- [ ] Integration test: verify system recovers from expired lock

### US-005: Add Scan Dedup Cache Layer
**Description:** As a developer, I want recently created scans cached so that duplicate checks are fast.

**Acceptance Criteria:**
- [ ] Cache recent scan IDs in Redis: `scan:{ecosystem}:{package}:{version} -> scanId`
- [ ] Cache TTL: 1 hour (matches typical scan duration + buffer)
- [ ] Check cache before database query
- [ ] Populate cache after successful scan creation
- [ ] Cache invalidated if scan fails
- [ ] Unit test: verify cache hit avoids database query
- [ ] Integration test: create scan, verify cached, request returns cached

### US-006: Add Metrics for Deduplication
**Description:** As a developer, I want metrics on deduplication so that I can monitor effectiveness.

**Acceptance Criteria:**
- [ ] Counter: `depfender_scan_requests_total` with label `result: new|existing|locked`
- [ ] Histogram: `depfender_lock_acquire_duration_seconds`
- [ ] Gauge: `depfender_active_locks`
- [ ] Counter: `depfender_lock_expirations_total`
- [ ] Metrics visible at `/metrics` endpoint
- [ ] Unit test: verify metrics increment correctly
- [ ] Integration test: create scans, verify metrics reflect operations

## Functional Requirements

- FR-1: The system must acquire a distributed lock before creating a scan
- FR-2: The system must return existing scan ID if one exists for the same package@version
- FR-3: The system must release locks in finally blocks to prevent deadlocks
- FR-4: Lock TTL must be 30 seconds to auto-recover from crashed processes
- FR-5: The system must wait up to 5 seconds for lock, then return 429
- FR-6: The system must cache scan IDs in Redis for 1 hour after creation
- FR-7: Response must indicate whether scan was newly created or already existed
- FR-8: Database unique constraint must serve as final deduplication safeguard

## Non-Goals

- Cross-datacenter lock coordination
- Scan result caching (only scan ID caching)
- Automatic retry of failed scans
- User-specific scan isolation (all users share scan results)
- Priority queue for duplicate requests

## Technical Considerations

- **Redis Lua Script:** Use Lua script for atomic check-and-delete:
  ```lua
  if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
  else
    return 0
  end
  ```
- **Lock Token:** Use UUID v4 for lock tokens to prevent accidental release
- **Cache Key Design:** Include ecosystem to support future multi-ecosystem expansion
- **Database Constraint:** Add `UNIQUE(package_id, version_id)` on scans if not present
- **Idempotency:** BullMQ already uses scanId as jobId for queue deduplication

## Success Metrics

- Zero duplicate scans created (verified by unique constraint violations = 0)
- Cache hit rate > 80% for popular packages
- Lock acquisition p99 < 10ms
- Zero deadlocks (lock expirations indicate potential issues)
- Duplicate request response time < 100ms (cached path)

## Open Questions

- Should we allow re-scanning a package that was scanned > 24 hours ago?
- Should we expose "force rescan" option to users?
- How should we handle version "latest" vs specific version deduplication?
