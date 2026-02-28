# PRD: SSE Integration Fix

## Introduction

Fix the Server-Sent Events (SSE) integration between the frontend and backend that prevents real-time scan progress updates from being displayed. The current implementation has an event type mismatch where the backend sends named events but the frontend only listens to the default message event.

## Goals

- Fix SSE event handling so frontend receives progress updates
- Standardize event types between backend and frontend
- Add proper error handling and reconnection logic
- Prevent memory leaks in event listeners
- Improve progress display UX with status messages

## User Stories

### US-001: Fix SSE Event Type Mismatch
**Description:** As a user, I want to see real-time progress updates so that I know my scan is progressing.

**Acceptance Criteria:**
- [ ] Backend sends events with `event: progress` type
- [ ] Frontend uses `eventSource.addEventListener('progress', handler)` instead of `onmessage`
- [ ] Progress updates display in UI as they arrive
- [ ] Typecheck passes
- [ ] Unit test: mock EventSource, verify progress handler called
- [ ] Integration test: start scan, verify progress events received
- [ ] Verify in browser using dev-browser skill

### US-002: Add SSE Error Event Handling
**Description:** As a developer, I want SSE errors handled properly so that the UI doesn't break silently.

**Acceptance Criteria:**
- [ ] Frontend listens to `eventSource.onerror`
- [ ] Display user-friendly error message on connection failure
- [ ] Automatically attempt reconnection with exponential backoff (1s, 2s, 4s, max 30s)
- [ ] Maximum 5 reconnection attempts before showing "Connection lost" message
- [ ] Log errors to console for debugging
- [ ] Typecheck passes
- [ ] Unit test: simulate connection error, verify retry logic
- [ ] Verify in browser using dev-browser skill

### US-003: Add SSE Completion Event
**Description:** As a developer, I want a completion event so that the frontend knows when to stop listening.

**Acceptance Criteria:**
- [ ] Backend sends `event: complete` when scan finishes (success or failure)
- [ ] Complete event includes final scan status and verdict
- [ ] Frontend closes EventSource connection on complete event
- [ ] Frontend fetches final scan results after complete event
- [ ] Typecheck passes
- [ ] Unit test: verify complete event closes connection
- [ ] Integration test: scan completes, verify connection closed

### US-004: Fix Memory Leak in SSE Cleanup
**Description:** As a developer, I want proper cleanup so that event listeners don't leak memory.

**Acceptance Criteria:**
- [ ] Return cleanup function from SSE setup
- [ ] Cleanup function removes all event listeners
- [ ] Cleanup function closes EventSource connection
- [ ] useEffect cleanup calls the returned function
- [ ] No event listeners remain after component unmount
- [ ] Typecheck passes
- [ ] Unit test: mount/unmount component, verify no leaked listeners

### US-005: Standardize SSE Event Payload Format
**Description:** As a developer, I want consistent event payloads so that parsing is reliable.

**Acceptance Criteria:**
- [ ] All SSE events are valid JSON
- [ ] Define TypeScript types for each event type:
  - `ProgressEvent: { type: 'progress', stage: string, message: string, percent?: number }`
  - `CompleteEvent: { type: 'complete', status: 'completed'|'failed', verdict?: string }`
  - `ErrorEvent: { type: 'error', message: string, code?: string }`
- [ ] Backend validates payload before sending
- [ ] Frontend parses with proper error handling for malformed JSON
- [ ] Typecheck passes
- [ ] Unit test: verify event parsing handles all types

### US-006: Add Progress Stage Indicators
**Description:** As a user, I want to see what stage my scan is in so that I understand the progress.

**Acceptance Criteria:**
- [ ] Backend sends progress events at key stages:
  - `{ stage: 'queued', message: 'Waiting in queue...', percent: 0 }`
  - `{ stage: 'downloading', message: 'Downloading package...', percent: 10 }`
  - `{ stage: 'analyzing', message: 'Running security agents...', percent: 30 }`
  - `{ stage: 'agent_N', message: 'Agent X analyzing...', percent: 30+(N*10) }`
  - `{ stage: 'finalizing', message: 'Generating report...', percent: 90 }`
- [ ] Frontend displays current stage and message
- [ ] Progress bar updates with percent value
- [ ] Typecheck passes
- [ ] Unit test: verify all stages emit correct events
- [ ] Verify in browser using dev-browser skill

### US-007: Add SSE Connection Timeout
**Description:** As a developer, I want SSE connections to timeout so that stale connections are cleaned up.

**Acceptance Criteria:**
- [ ] Backend closes SSE connection after scan completes
- [ ] Backend closes SSE connection after 10 minutes (max scan time + buffer)
- [ ] Frontend handles server-initiated close gracefully
- [ ] Display "Scan timed out" message if timeout occurs
- [ ] Typecheck passes
- [ ] Unit test: simulate timeout, verify connection closed
- [ ] Verify in browser using dev-browser skill

### US-008: Add SSE Heartbeat
**Description:** As a developer, I want heartbeat events so that connections stay alive through proxies.

**Acceptance Criteria:**
- [ ] Backend sends `event: heartbeat` every 30 seconds
- [ ] Heartbeat payload: `{ timestamp: ISO8601 }`
- [ ] Frontend ignores heartbeat events (no UI update)
- [ ] Heartbeat keeps connection alive through nginx/cloudflare proxies
- [ ] Typecheck passes
- [ ] Unit test: verify heartbeat events sent at interval

## Functional Requirements

- FR-1: Backend must send SSE events with explicit `event:` type field
- FR-2: Frontend must use `addEventListener` for named event types, not `onmessage`
- FR-3: Frontend must implement exponential backoff reconnection (1s, 2s, 4s, max 30s)
- FR-4: Frontend must close EventSource on component unmount
- FR-5: Backend must send `complete` event when scan finishes
- FR-6: Backend must send `heartbeat` event every 30 seconds
- FR-7: Backend must close SSE connections after 10 minutes maximum
- FR-8: All SSE event payloads must be valid JSON matching defined TypeScript types

## Non-Goals

- WebSocket implementation (SSE is sufficient for one-way updates)
- Push notifications for completed scans
- Multiple concurrent scan progress tracking
- Historical progress replay
- Progress persistence across page refreshes

## Design Considerations

- **Progress Bar:** Use existing Tailwind progress bar component
- **Stage Display:** Show stage name and message below progress bar
- **Error State:** Red alert box with retry button
- **Loading State:** Pulsing animation while waiting for first event
- **Completed State:** Green checkmark with verdict badge

## Technical Considerations

- **Event Type Standards:** Use lowercase event names: `progress`, `complete`, `error`, `heartbeat`
- **JSON Parsing:** Always wrap `JSON.parse` in try/catch
- **EventSource Polyfill:** Not needed - native support in all modern browsers
- **Proxy Timeout:** Nginx default timeout is 60s; heartbeat prevents disconnect
- **React Strict Mode:** SSE setup in useEffect will run twice in dev; handle gracefully

## Success Metrics

- Progress updates visible within 1 second of backend event
- Zero memory leaks (event listeners cleaned up on unmount)
- Connection stays alive for full scan duration (no proxy timeouts)
- Error recovery successful in < 5 seconds
- Users report understanding scan progress (qualitative feedback)

## Open Questions

- Should we show estimated time remaining based on average scan duration?
- Should progress events include agent names for detailed progress?
- Should we persist progress state to survive page refresh?
