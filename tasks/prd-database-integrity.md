# PRD: Database Integrity Improvements

## Introduction

Address database integrity issues discovered during the system audit to prevent data corruption, silent deduplication failures, and performance degradation. This PRD covers adding missing constraints, wrapping multi-table operations in transactions, optimizing indexes, and adding proper enum validation.

## Goals

- Add missing unique constraint for evidence_blobs deduplication
- Wrap all multi-table write operations in database transactions
- Add indexes for common query patterns to improve performance
- Add CHECK constraints for enum fields (status, verdict, severity)
- Eliminate N+1 query patterns in dependency storage
- Ensure referential integrity across all tables

## User Stories

### US-001: Add Evidence Blob Unique Constraint
**Description:** As a developer, I need the evidence_blobs table to have a unique constraint so that ON CONFLICT deduplication works correctly.

**Acceptance Criteria:**
- [ ] Create migration adding `UNIQUE(finding_id, snippet_sha)` constraint to `evidence_blobs`
- [ ] Migration handles existing duplicate data (keep first, delete others)
- [ ] Verify `ON CONFLICT (finding_id, snippet_sha) DO NOTHING` works after migration
- [ ] Migration is reversible
- [ ] Unit test: insert duplicate evidence blob, verify only one stored
- [ ] Integration test: run migration on test database with existing data

### US-002: Add Database Transactions for Scan Creation
**Description:** As a developer, I want scan creation to be atomic so that partial data is never written on failure.

**Acceptance Criteria:**
- [ ] Wrap scan creation (scan + version + package records) in transaction
- [ ] Transaction rolls back if any insert fails
- [ ] Connection released back to pool on success or failure
- [ ] Error includes transaction ID for debugging
- [ ] Unit test: simulate failure mid-transaction, verify rollback
- [ ] Integration test: create scan, verify all related records exist

### US-003: Add Database Transactions for Findings Storage
**Description:** As a developer, I want findings storage to be atomic so that partial results are never stored.

**Acceptance Criteria:**
- [ ] Wrap all findings + evidence_blobs inserts in single transaction
- [ ] Batch insert findings (not one-by-one) within transaction
- [ ] Transaction rolls back if any insert fails
- [ ] Scan status only updated to "completed" after successful commit
- [ ] Unit test: simulate failure during findings storage, verify rollback
- [ ] Integration test: store findings, verify atomicity

### US-004: Add Indexes for Common Query Patterns
**Description:** As a developer, I want proper indexes so that common queries perform well.

**Acceptance Criteria:**
- [ ] Add index on `scans.status` for queue queries
- [ ] Add index on `scans.finished_at` for recent scans queries
- [ ] Add index on `scans.created_at` for ordering
- [ ] Add composite index on `findings(scan_id, severity)` for findings queries
- [ ] Add index on `findings.agent_name` for per-agent queries
- [ ] Migration adds indexes concurrently (no table locks)
- [ ] Verify EXPLAIN ANALYZE shows index usage for common queries
- [ ] Unit test: verify indexes exist after migration

### US-005: Add CHECK Constraints for Enum Fields
**Description:** As a developer, I want enum fields validated at the database level so that invalid values cannot be stored.

**Acceptance Criteria:**
- [ ] Add CHECK constraint on `scans.status`: `('pending', 'processing', 'completed', 'failed')`
- [ ] Add CHECK constraint on `scans.verdict`: `('safe', 'suspicious', 'malicious')` or NULL
- [ ] Add CHECK constraint on `findings.severity`: `('critical', 'high', 'medium', 'low', 'info')`
- [ ] Migration handles any existing invalid data before adding constraint
- [ ] Unit test: attempt to insert invalid enum value, verify rejected
- [ ] Integration test: verify constraints exist in database schema

### US-006: Batch Insert Dependencies (Fix N+1)
**Description:** As a developer, I want dependencies inserted in batches so that we don't execute 100+ queries per scan.

**Acceptance Criteria:**
- [ ] Replace loop of single INSERTs with single batch INSERT
- [ ] Use `unnest()` or multi-value INSERT for PostgreSQL
- [ ] Batch size configurable (default 100)
- [ ] Maintain ON CONFLICT behavior for deduplication
- [ ] Log query count before/after for verification
- [ ] Unit test: insert 50 dependencies, verify single query
- [ ] Integration test: store scan with dependencies, verify correct data

### US-007: Add Foreign Key Cascade Rules
**Description:** As a developer, I want proper cascade rules so that orphaned data is automatically cleaned up.

**Acceptance Criteria:**
- [ ] Add `ON DELETE CASCADE` from `findings` to `scans`
- [ ] Add `ON DELETE CASCADE` from `evidence_blobs` to `findings`
- [ ] Add `ON DELETE CASCADE` from `scans` to `versions`
- [ ] Document cascade behavior in schema comments
- [ ] Unit test: delete scan, verify findings and evidence deleted
- [ ] Integration test: verify no orphaned records after deletions

### US-008: Add Connection Pool Health Checks
**Description:** As a developer, I want database connections validated so that stale connections don't cause failures.

**Acceptance Criteria:**
- [ ] Configure pg pool with `idleTimeoutMillis: 30000`
- [ ] Configure pg pool with `connectionTimeoutMillis: 5000`
- [ ] Add query to `/health/ready` that tests database connection
- [ ] Log pool statistics periodically (available, waiting, total)
- [ ] Unit test: verify pool configuration applied
- [ ] Integration test: health check returns database status

## Functional Requirements

- FR-1: The `evidence_blobs` table must have `UNIQUE(finding_id, snippet_sha)` constraint
- FR-2: Scan creation must be atomic - all records created or none
- FR-3: Findings storage must be atomic - all findings/evidence stored or none
- FR-4: The system must use indexed queries for status, finished_at, and severity filters
- FR-5: The system must reject invalid enum values at the database level
- FR-6: Dependency storage must use batch inserts, not individual queries
- FR-7: Deleting a scan must automatically delete related findings and evidence
- FR-8: Database health check must verify connection before reporting ready

## Non-Goals

- Database sharding or partitioning
- Read replicas for query distribution
- Automated backup system
- Point-in-time recovery setup
- Database migration CI/CD pipeline
- ORM integration (continue using raw SQL with pg)

## Technical Considerations

- **Migration Tool:** Use existing `db/migrations/` structure with numbered files
- **Concurrent Index:** Use `CREATE INDEX CONCURRENTLY` to avoid table locks
- **Batch Size:** 100 items per batch balances memory and network efficiency
- **Pool Size:** Current pool size (10) may need increase for concurrent workers
- **Transaction Isolation:** Use default READ COMMITTED isolation level
- **Constraint Names:** Use descriptive names like `chk_scans_status_valid`

## Success Metrics

- Zero duplicate evidence_blobs (verified by `SELECT COUNT(*) = COUNT(DISTINCT ...)`)
- Query time for recent scans < 50ms with 100k+ records
- N+1 pattern eliminated: dependency storage uses ≤ 2 queries regardless of count
- Zero orphaned records after scan deletions
- Database constraint violations caught before application logic

## Open Questions

- Should we add partial indexes for `status = 'pending'` queries (small subset)?
- What's the data retention policy? Should we add `deleted_at` soft-delete column?
- Should pool size be configurable per environment or auto-scaled?
