# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Notebook Collaboration Platform · Created: 2026-05-20

## Philosophy

This model treats every change to a notebook as an immutable event stored in an append-only event log. The event store is the single source of truth; the current state of any notebook is derived by replaying its events. Materialised read models (following CQRS — Command Query Responsibility Segregation) provide fast queries for the UI, while the event log enables full audit trails, temporal queries ("what did this notebook look like on March 15th?"), and AI-powered analytics on editing patterns.

This approach is directly inspired by how CRDTs already work in collaborative editing. Yjs internally maintains an operation log where each character insertion or deletion is an event with a unique identifier. Extending this philosophy to the entire application — not just text editing but cell creation, execution, sharing, commenting — creates a unified model where the collaboration layer and the persistence layer share the same conceptual foundation.

Event sourcing is used in financial systems (ledgers), healthcare (patient record timelines), and audit-heavy enterprise software. For a notebook platform, it enables unique features: semantic version diffing from event replay, AI training on editing patterns, "time travel" to any point in a notebook's history, and compliance-grade audit trails that record not just what changed but who changed it and why.

**Best for:** Teams that need full audit trails, temporal querying, AI-powered insights on editing patterns, and regulatory compliance — particularly enterprise and research deployments.

**Trade-offs:**
- (+) Complete, immutable audit trail — every action ever taken is recorded
- (+) Time-travel queries: reconstruct notebook state at any point in time
- (+) Natural fit for CRDT-based collaboration — both are operation/event-based
- (+) Rich analytics: AI can learn from editing patterns, execution sequences, collaboration dynamics
- (+) Enables semantic diff summaries by analysing event sequences between versions
- (-) Higher storage requirements — events accumulate indefinitely
- (-) Read queries require materialised views or projections that must be kept in sync
- (-) Eventual consistency between event store and read models adds complexity
- (-) Debugging state issues requires replaying event sequences
- (-) Snapshot management needed for performance — replaying thousands of events is slow

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| nbformat v4 | Events produce/consume nbformat-compatible structures; materialised notebook view is a valid .ipynb |
| Yjs / Y-sync | CRDT operations are a subset of notebook events; the Yjs operation log maps to cell_edit events |
| OCSF (Open Cybersecurity Schema Framework) | Event schema inspired by OCSF's structured event format: actor, action, target, outcome |
| FAIR4RS | Event log provides complete provenance chain for reproducibility and citability |
| OAuth 2.0 / OIDC | Actor identification in events uses authenticated user identity from OIDC claims |
| RFC 7807 (Problem Details) | Error events follow RFC 7807 structure for consistent error reporting |

---

## Event Store

```sql
-- The single source of truth: an append-only event log
CREATE TABLE notebook_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,  -- notebook ID — all events for one notebook share a stream
    sequence_number BIGINT NOT NULL,  -- monotonically increasing per stream
    event_type VARCHAR(100) NOT NULL,  -- e.g. 'cell.created', 'cell.executed', 'notebook.shared'
    event_version INT NOT NULL DEFAULT 1,  -- schema version for this event type
    actor_id UUID NOT NULL,  -- user who caused the event
    actor_type VARCHAR(50) NOT NULL DEFAULT 'user',  -- 'user', 'system', 'scheduler', 'ai_agent'
    payload JSONB NOT NULL,  -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',  -- cross-cutting: ip, user_agent, request_id
    correlation_id UUID,  -- links related events (e.g., all events in a single execution run)
    causation_id UUID REFERENCES notebook_events(event_id),  -- the event that caused this event
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Primary query path: replay events for a notebook in order
CREATE INDEX idx_events_stream_seq ON notebook_events(stream_id, sequence_number);

-- Query by event type across all notebooks (analytics, monitoring)
CREATE INDEX idx_events_type_time ON notebook_events(event_type, created_at);

-- Query by actor (user activity feed)
CREATE INDEX idx_events_actor ON notebook_events(actor_id, created_at);

-- Correlation queries (find all events in an execution run)
CREATE INDEX idx_events_correlation ON notebook_events(correlation_id);

-- Partition by month for performance at scale
-- CREATE TABLE notebook_events PARTITION BY RANGE (created_at);
```

### Event Type Catalogue

```sql
-- Example event payloads by type:

-- 'notebook.created'
-- { "name": "Analysis Q3", "project_id": "...", "kernel_name": "python3", "nbformat": 4 }

-- 'cell.created'
-- { "cell_id": "...", "cell_type": "code", "position": 3, "source": "" }

-- 'cell.edited'
-- { "cell_id": "...", "operations": [...yjs_ops...], "source_after": "import pandas as pd" }

-- 'cell.executed'
-- { "cell_id": "...", "execution_count": 5, "kernel_id": "...",
--   "outputs": [{"output_type": "execute_result", "data": {"text/plain": "42"}}],
--   "duration_ms": 1234 }

-- 'cell.moved'
-- { "cell_id": "...", "from_position": 3, "to_position": 1 }

-- 'cell.deleted'
-- { "cell_id": "...", "cell_type": "code", "source": "# old code here" }

-- 'notebook.shared'
-- { "shared_with": "user@example.com", "permission": "edit", "link_token": "..." }

-- 'execution.started'
-- { "run_id": "...", "trigger": "scheduled", "parameters": {"date": "2026-05-01"} }

-- 'execution.completed'
-- { "run_id": "...", "status": "completed", "duration_ms": 45000, "cell_count": 12 }

-- 'comment.added'
-- { "comment_id": "...", "cell_id": "...", "body": "Should we use a different model here?" }

-- 'ai.completion_requested'
-- { "cell_id": "...", "prompt_context_hash": "...", "model": "claude-sonnet-4" }

-- 'ai.completion_accepted'
-- { "cell_id": "...", "completion_id": "...", "suggestion_length": 145 }

-- 'anomaly.detected'
-- { "run_id": "...", "cell_id": "...", "alert_type": "distribution_shift",
--   "expected": {"mean": 42.3, "std": 2.1}, "actual": {"mean": 67.8, "std": 15.2} }
```

## Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying the full event history
CREATE TABLE notebook_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,  -- notebook ID
    sequence_number BIGINT NOT NULL,  -- the event sequence this snapshot is built from
    snapshot JSONB NOT NULL,  -- full materialised notebook state as .ipynb-compatible JSON
    cell_count INT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

CREATE INDEX idx_snapshots_stream ON notebook_snapshots(stream_id, sequence_number DESC);

-- To reconstruct current state:
-- 1. Find latest snapshot for stream_id
-- 2. Replay all events after that snapshot's sequence_number
-- 3. Apply events to snapshot to get current state
```

## Materialised Read Models

```sql
-- Denormalised read model for the notebook list view
-- Updated asynchronously by event projections
CREATE TABLE mv_notebooks (
    id UUID PRIMARY KEY,
    workspace_id UUID NOT NULL,
    project_id UUID NOT NULL,
    project_name VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    kernel_name VARCHAR(100) NOT NULL,
    language VARCHAR(50) NOT NULL,
    cell_count INT NOT NULL DEFAULT 0,
    created_by_id UUID NOT NULL,
    created_by_name VARCHAR(255) NOT NULL,
    last_edited_by_id UUID,
    last_edited_by_name VARCHAR(255),
    last_edited_at TIMESTAMPTZ,
    last_executed_at TIMESTAMPTZ,
    last_execution_status VARCHAR(50),
    version_count INT NOT NULL DEFAULT 1,
    is_shared BOOLEAN NOT NULL DEFAULT false,
    tags TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mv_notebooks_workspace ON mv_notebooks(workspace_id);
CREATE INDEX idx_mv_notebooks_project ON mv_notebooks(project_id);

-- Denormalised read model for workspace members
CREATE TABLE mv_workspace_members (
    workspace_id UUID NOT NULL,
    user_id UUID NOT NULL,
    user_email VARCHAR(255) NOT NULL,
    user_display_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL,
    joined_at TIMESTAMPTZ NOT NULL,
    last_active_at TIMESTAMPTZ,
    notebooks_edited INT NOT NULL DEFAULT 0,
    PRIMARY KEY (workspace_id, user_id)
);

-- Read model for execution history
CREATE TABLE mv_execution_history (
    run_id UUID PRIMARY KEY,
    notebook_id UUID NOT NULL,
    notebook_name VARCHAR(255) NOT NULL,
    trigger_type VARCHAR(50) NOT NULL,
    triggered_by_id UUID,
    triggered_by_name VARCHAR(255),
    status VARCHAR(50) NOT NULL,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_ms BIGINT,
    cell_count INT,
    error_summary TEXT,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_mv_execution_notebook ON mv_execution_history(notebook_id, created_at DESC);

-- Read model for comments (denormalised for fast rendering)
CREATE TABLE mv_comments (
    comment_id UUID PRIMARY KEY,
    notebook_id UUID NOT NULL,
    cell_id UUID,
    parent_comment_id UUID,
    author_id UUID NOT NULL,
    author_name VARCHAR(255) NOT NULL,
    author_avatar_url TEXT,
    body TEXT NOT NULL,
    is_resolved BOOLEAN NOT NULL DEFAULT false,
    resolved_by_name VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_mv_comments_notebook ON mv_comments(notebook_id);
CREATE INDEX idx_mv_comments_cell ON mv_comments(cell_id);
```

## Supporting Tables (Non-Event-Sourced)

```sql
-- These tables store configuration that doesn't benefit from event sourcing

CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    auth_provider VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_provider_id VARCHAR(255),
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE data_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    connector_type VARCHAR(50) NOT NULL,
    connection_config_encrypted BYTEA,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scheduled_run_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_stream_id UUID NOT NULL,  -- references the event stream, not a FK
    cron_expression VARCHAR(100) NOT NULL,
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT true,
    parameters JSONB NOT NULL DEFAULT '{}',
    notification_channels JSONB NOT NULL DEFAULT '[]',
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Real-time collaboration: Yjs state is ephemeral but persisted for crash recovery
CREATE TABLE collaboration_sessions (
    notebook_stream_id UUID PRIMARY KEY,
    yjs_state BYTEA,
    active_user_count INT NOT NULL DEFAULT 0,
    last_update_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

```sql
-- Time travel: reconstruct notebook state at a specific point in time
SELECT payload
FROM notebook_events
WHERE stream_id = '...'
  AND created_at <= '2026-03-15T14:30:00Z'
ORDER BY sequence_number;
-- Application code replays these events against the latest snapshot before that date

-- Semantic diff between two points: what happened between version A and version B?
SELECT event_type, actor_id, payload, created_at
FROM notebook_events
WHERE stream_id = '...'
  AND sequence_number BETWEEN 100 AND 150
ORDER BY sequence_number;
-- AI summariser processes this event sequence into natural language

-- User activity feed: recent actions by a user across all notebooks
SELECT e.event_type, e.payload, e.created_at, n.name AS notebook_name
FROM notebook_events e
JOIN mv_notebooks n ON n.id = e.stream_id
WHERE e.actor_id = '...'
ORDER BY e.created_at DESC
LIMIT 50;

-- AI analytics: code completion acceptance rate by model
SELECT
    payload->>'model' AS model,
    COUNT(*) FILTER (WHERE event_type = 'ai.completion_requested') AS requests,
    COUNT(*) FILTER (WHERE event_type = 'ai.completion_accepted') AS acceptances,
    ROUND(
        COUNT(*) FILTER (WHERE event_type = 'ai.completion_accepted')::NUMERIC /
        NULLIF(COUNT(*) FILTER (WHERE event_type = 'ai.completion_requested'), 0) * 100, 1
    ) AS acceptance_rate
FROM notebook_events
WHERE event_type IN ('ai.completion_requested', 'ai.completion_accepted')
  AND created_at >= now() - INTERVAL '30 days'
GROUP BY payload->>'model';

-- Anomaly frequency by notebook over the last 7 days
SELECT
    stream_id AS notebook_id,
    COUNT(*) AS anomaly_count,
    COUNT(DISTINCT payload->>'cell_id') AS affected_cells
FROM notebook_events
WHERE event_type = 'anomaly.detected'
  AND created_at >= now() - INTERVAL '7 days'
GROUP BY stream_id
ORDER BY anomaly_count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | notebook_events — the single source of truth |
| Snapshots | 1 | notebook_snapshots — performance optimisation |
| Materialised Read Models | 4 | mv_notebooks, mv_workspace_members, mv_execution_history, mv_comments |
| Supporting Configuration | 6 | workspaces, users, data_connections, scheduled_run_configs, collaboration_sessions |
| **Total** | **12** | Significantly fewer tables than the normalized model |

---

## Key Design Decisions

1. **Single event table rather than per-type tables** — all events go into `notebook_events` with the event type as a discriminator. This simplifies the write path (one INSERT per action), enables cross-type queries (user activity feeds), and avoids schema changes when new event types are added. The trade-off is that payload structure varies by event_type, requiring application-level validation.

2. **Sequence numbers per stream, not global** — each notebook has its own monotonically increasing sequence, enabling efficient stream replay without gaps. Global ordering is available via `created_at` but is not guaranteed to be gap-free.

3. **Snapshots at configurable intervals** — without snapshots, reconstructing a notebook that has accumulated 10,000 events would be prohibitively slow. The application creates snapshots every N events (e.g., 100) or on explicit version saves. Reconstruction loads the latest snapshot and replays only subsequent events.

4. **Materialised views are denormalised and eventually consistent** — the `mv_` tables are projections of the event stream, updated asynchronously by event handlers. They may lag the event store by milliseconds. The UI reads from these views for list/search operations and from the event store (via snapshot + replay) for the notebook editor.

5. **CRDT operations as events** — Yjs cell-edit operations map naturally to `cell.edited` events. The Yjs binary state is stored for crash recovery, but the event log captures a higher-level representation of each edit for audit and analytics purposes.

6. **Correlation and causation IDs for event chains** — `correlation_id` groups related events (e.g., all events during a single execution run). `causation_id` tracks direct cause-effect relationships (e.g., an `anomaly.detected` event was caused by an `execution.completed` event). This enables root-cause analysis and debugging.

7. **Event versioning for schema evolution** — `event_version` allows the payload schema to evolve over time. Event handlers can branch on version number to process old and new formats. This avoids the need for data migrations on the event store.

8. **Non-event-sourced supporting tables** — user profiles, workspace settings, and data connection credentials are stored conventionally. These are configuration data where full event history adds no value and would complicate GDPR deletion requests.
