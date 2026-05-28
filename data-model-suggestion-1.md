# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Notebook Collaboration Platform · Created: 2026-05-20

## Philosophy

This model follows a fully normalized relational design where every domain concept gets its own table with strict foreign key relationships. The notebook document model (aligned with nbformat v4) is decomposed into relational tables: notebooks contain ordered cells, cells produce outputs, and versions track the full state at each save point. Collaboration, scheduling, and AI features each have dedicated table groups.

The approach mirrors how traditional content management systems and enterprise SaaS platforms model structured documents. JupyterHub's internal database (SQLAlchemy-based) uses a similar normalized pattern for users, servers, and tokens. Databricks' workspace API exposes notebooks, clusters, and jobs as distinct REST resources, implying a normalized backend.

This design prioritises data integrity, query flexibility, and clear separation of concerns. Every relationship is explicit via foreign keys, making it straightforward to enforce referential integrity and write complex cross-entity queries (e.g., "find all notebooks in workspace X that were last executed by user Y with failing outputs").

**Best for:** Teams that value strict data integrity, need complex ad-hoc queries across entities, and operate in environments where schema migrations are manageable.

**Trade-offs:**
- (+) Maximum query flexibility — any cross-entity question is a JOIN away
- (+) Strong referential integrity enforced at the database level
- (+) Clear, auditable schema that maps directly to domain concepts
- (+) Easy to reason about for new developers
- (-) High table count increases migration complexity
- (-) Cell ordering requires explicit `position` columns with reordering overhead
- (-) Notebook reconstruction requires multiple JOINs (notebook + cells + outputs)
- (-) Schema changes needed for every new cell type or output format
- (-) Real-time collaboration state (CRDT) must be stored separately from relational data

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| nbformat v4 | Notebook, Cell, and Output tables mirror the nbformat JSON structure — cell_type enum matches nbformat cell types, output MIME types stored relationally |
| Jupyter Server REST API | Table structure maps 1:1 to Jupyter Server API resources: kernels, sessions, contents |
| OAuth 2.0 / OIDC | auth_provider and auth_tokens tables support federated identity per OIDC spec |
| SCIM 2.0 | User and workspace_membership tables support SCIM provisioning/deprovisioning patterns |
| OpenAPI 3.1 | Each table maps cleanly to an OpenAPI resource schema for the platform's REST API |
| FAIR4RS | Version and notebook_metadata tables support findability, accessibility, and reproducibility metadata |
| JSON Schema | Cell output validation can reference JSON Schema definitions for structured output types |

---

## Identity & Multi-Tenancy

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',  -- 'free', 'team', 'enterprise'
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    auth_provider VARCHAR(50) NOT NULL DEFAULT 'local',  -- 'local', 'google', 'github', 'saml'
    auth_provider_id VARCHAR(255),
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'editor',  -- 'viewer', 'editor', 'admin', 'owner'
    invited_by UUID REFERENCES users(id),
    joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);

CREATE INDEX idx_workspace_memberships_workspace ON workspace_memberships(workspace_id);
CREATE INDEX idx_workspace_memberships_user ON workspace_memberships(user_id);

CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    token_hash VARCHAR(64) NOT NULL UNIQUE,  -- SHA-256 of the token
    name VARCHAR(255) NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',  -- e.g. '{read:notebooks, execute:notebooks}'
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_tokens_user ON api_tokens(user_id);
CREATE INDEX idx_api_tokens_hash ON api_tokens(token_hash);
```

## Projects & Notebooks

```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    archived_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_workspace ON projects(workspace_id);

CREATE TABLE notebooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    nbformat_major INT NOT NULL DEFAULT 4,
    nbformat_minor INT NOT NULL DEFAULT 5,
    kernel_name VARCHAR(100) NOT NULL DEFAULT 'python3',
    language VARCHAR(50) NOT NULL DEFAULT 'python',
    created_by UUID NOT NULL REFERENCES users(id),
    last_edited_by UUID REFERENCES users(id),
    last_edited_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notebooks_project ON notebooks(project_id);

CREATE TABLE notebook_metadata (
    notebook_id UUID PRIMARY KEY REFERENCES notebooks(id) ON DELETE CASCADE,
    kernel_display_name VARCHAR(255),
    language_version VARCHAR(50),
    environment_image VARCHAR(500),  -- Docker image for repo2docker reproducibility
    environment_spec JSONB,  -- requirements.txt / environment.yml parsed
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_metadata JSONB NOT NULL DEFAULT '{}'
);
```

## Cells & Outputs (nbformat-aligned)

```sql
CREATE TYPE cell_type AS ENUM ('code', 'markdown', 'raw', 'sql');

CREATE TABLE cells (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    cell_type cell_type NOT NULL,
    source TEXT NOT NULL DEFAULT '',
    position INT NOT NULL,  -- ordering within notebook
    metadata JSONB NOT NULL DEFAULT '{}',
    execution_count INT,  -- NULL for non-code cells
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cells_notebook_position ON cells(notebook_id, position);

CREATE TYPE output_type AS ENUM (
    'execute_result', 'display_data', 'stream', 'error'
);

CREATE TABLE cell_outputs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cell_id UUID NOT NULL REFERENCES cells(id) ON DELETE CASCADE,
    output_type output_type NOT NULL,
    position INT NOT NULL,  -- ordering within cell outputs
    execution_count INT,
    text TEXT,  -- for stream outputs
    traceback TEXT[],  -- for error outputs
    ename VARCHAR(255),  -- error name
    evalue TEXT,  -- error value
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cell_outputs_cell ON cell_outputs(cell_id, position);

CREATE TABLE output_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cell_output_id UUID NOT NULL REFERENCES cell_outputs(id) ON DELETE CASCADE,
    mime_type VARCHAR(255) NOT NULL,  -- e.g. 'text/plain', 'image/png', 'application/json'
    data TEXT NOT NULL,  -- base64-encoded for binary, text for text types
    is_binary BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_output_data_output ON output_data(cell_output_id);
```

## Version Control

```sql
CREATE TABLE notebook_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    snapshot JSONB NOT NULL,  -- full .ipynb JSON at this version
    message TEXT,  -- optional commit message
    created_by UUID NOT NULL REFERENCES users(id),
    parent_version_id UUID REFERENCES notebook_versions(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (notebook_id, version_number)
);

CREATE INDEX idx_notebook_versions_notebook ON notebook_versions(notebook_id, version_number DESC);

CREATE TABLE git_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,  -- 'github', 'gitlab', 'bitbucket'
    repo_url TEXT NOT NULL,
    branch VARCHAR(255) NOT NULL DEFAULT 'main',
    access_token_encrypted BYTEA,
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Real-Time Collaboration

```sql
CREATE TABLE collaboration_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    yjs_document_id VARCHAR(255) NOT NULL UNIQUE,  -- Yjs document identifier
    yjs_state BYTEA,  -- serialised Yjs document state vector
    last_update_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collab_sessions_notebook ON collaboration_sessions(notebook_id);

CREATE TABLE presence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collaboration_session_id UUID NOT NULL REFERENCES collaboration_sessions(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    cursor_cell_id UUID REFERENCES cells(id),
    cursor_offset INT,
    color VARCHAR(7),  -- hex color for cursor display
    connected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (collaboration_session_id, user_id)
);
```

## Scheduling & Execution

```sql
CREATE TYPE schedule_status AS ENUM ('active', 'paused', 'disabled');
CREATE TYPE run_status AS ENUM ('queued', 'running', 'completed', 'failed', 'cancelled', 'timed_out');

CREATE TABLE scheduled_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    name VARCHAR(255),
    cron_expression VARCHAR(100) NOT NULL,  -- standard cron syntax
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    status schedule_status NOT NULL DEFAULT 'active',
    parameters JSONB NOT NULL DEFAULT '{}',  -- parameterised notebook inputs
    notify_on TEXT[] NOT NULL DEFAULT '{failure}',  -- 'success', 'failure'
    notification_channels JSONB NOT NULL DEFAULT '[]',  -- [{type: 'email', target: '...'}, {type: 'webhook', url: '...'}]
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scheduled_runs_notebook ON scheduled_runs(notebook_id);

CREATE TABLE execution_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    scheduled_run_id UUID REFERENCES scheduled_runs(id) ON DELETE SET NULL,
    triggered_by UUID REFERENCES users(id),  -- NULL for scheduled runs
    trigger_type VARCHAR(50) NOT NULL,  -- 'manual', 'scheduled', 'api', 'webhook'
    status run_status NOT NULL DEFAULT 'queued',
    notebook_snapshot JSONB,  -- frozen notebook state at execution start
    parameters JSONB NOT NULL DEFAULT '{}',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_ms BIGINT,
    error_message TEXT,
    kernel_id VARCHAR(255),  -- kernel used for this execution
    compute_resources JSONB,  -- {cpu: '2', memory: '4Gi', gpu: null}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_execution_runs_notebook ON execution_runs(notebook_id);
CREATE INDEX idx_execution_runs_status ON execution_runs(status);
CREATE INDEX idx_execution_runs_scheduled ON execution_runs(scheduled_run_id);
```

## Data Connections

```sql
CREATE TABLE data_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    connector_type VARCHAR(50) NOT NULL,  -- 'postgresql', 'snowflake', 'bigquery', 's3', etc.
    host VARCHAR(500),
    port INT,
    database_name VARCHAR(255),
    credentials_encrypted BYTEA,  -- encrypted connection credentials
    connection_options JSONB NOT NULL DEFAULT '{}',
    created_by UUID NOT NULL REFERENCES users(id),
    last_tested_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_connections_workspace ON data_connections(workspace_id);
```

## Comments & Collaboration

```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    cell_id UUID REFERENCES cells(id) ON DELETE CASCADE,
    parent_comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id),
    body TEXT NOT NULL,
    resolved BOOLEAN NOT NULL DEFAULT false,
    resolved_by UUID REFERENCES users(id),
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_notebook ON comments(notebook_id);
CREATE INDEX idx_comments_cell ON comments(cell_id);
```

## AI Features

```sql
CREATE TABLE ai_completions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    cell_id UUID NOT NULL REFERENCES cells(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    prompt_context JSONB NOT NULL,  -- {variables: [...], schemas: [...], prior_outputs: [...]}
    suggestion TEXT NOT NULL,
    accepted BOOLEAN,
    model_id VARCHAR(100) NOT NULL,
    latency_ms INT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_completions_notebook ON ai_completions(notebook_id);

CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_run_id UUID NOT NULL REFERENCES execution_runs(id) ON DELETE CASCADE,
    cell_id UUID NOT NULL REFERENCES cells(id) ON DELETE CASCADE,
    alert_type VARCHAR(50) NOT NULL,  -- 'distribution_shift', 'value_out_of_range', 'schema_change'
    severity VARCHAR(20) NOT NULL DEFAULT 'warning',  -- 'info', 'warning', 'critical'
    description TEXT NOT NULL,
    expected_value JSONB,
    actual_value JSONB,
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_alerts_run ON anomaly_alerts(execution_run_id);
```

## Sharing & Publishing

```sql
CREATE TYPE share_permission AS ENUM ('view', 'comment', 'edit');

CREATE TABLE notebook_shares (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    shared_with_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    shared_with_email VARCHAR(255),  -- for inviting non-registered users
    permission share_permission NOT NULL DEFAULT 'view',
    link_token VARCHAR(64) UNIQUE,  -- for share-by-link
    password_hash VARCHAR(255),  -- optional password protection
    expires_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notebook_shares_notebook ON notebook_shares(notebook_id);
CREATE INDEX idx_notebook_shares_link ON notebook_shares(link_token);

CREATE TABLE published_dashboards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    layout JSONB NOT NULL,  -- dashboard layout definition
    auto_refresh_seconds INT,
    published_by UUID NOT NULL REFERENCES users(id),
    published_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | workspaces, users, workspace_memberships, api_tokens |
| Projects & Notebooks | 3 | projects, notebooks, notebook_metadata |
| Cells & Outputs | 3 | cells, cell_outputs, output_data |
| Version Control | 2 | notebook_versions, git_connections |
| Real-Time Collaboration | 2 | collaboration_sessions, presence |
| Scheduling & Execution | 2 | scheduled_runs, execution_runs |
| Data Connections | 1 | data_connections |
| Comments | 1 | comments |
| AI Features | 2 | ai_completions, anomaly_alerts |
| Sharing & Publishing | 2 | notebook_shares, published_dashboards |
| **Total** | **22** | |

---

## Key Design Decisions

1. **Cells stored as separate rows with position ordering** — mirrors nbformat's cell array but allows individual cell queries, per-cell comments, and per-cell AI completions. The trade-off is that reconstructing a full notebook requires a JOIN, and reordering cells means updating position values.

2. **Output data separated into its own table with MIME types** — nbformat stores outputs as a dictionary of MIME types. Normalizing this into output_data rows allows querying by MIME type (e.g., "find all cells that produced image outputs") and avoids storing large binary blobs inline with cell metadata.

3. **Notebook versions store full .ipynb snapshots as JSONB** — rather than storing diffs between versions, each version captures the complete notebook state. This trades storage space for simplicity: any version can be exported directly as a valid .ipynb file without replay logic.

4. **CRDT state stored as binary (BYTEA)** — Yjs document state is an opaque binary blob that cannot be meaningfully queried in SQL. The relational tables represent the "settled" state of the notebook; the CRDT state is only used during active collaboration sessions and is periodically flushed to the relational cell/output tables.

5. **Workspace-scoped multi-tenancy with row-level foreign keys** — every resource traces back to a workspace via project. This enables PostgreSQL Row-Level Security policies scoped to workspace_id for tenant isolation.

6. **Separate tables for scheduled_runs (configuration) and execution_runs (instances)** — follows the template/instance pattern. A scheduled_run defines the recurring configuration; each actual execution creates an execution_run record with its own status, timing, and output.

7. **AI completions tracked as first-class entities** — capturing prompt context, suggestions, and acceptance rates enables analytics on AI feature effectiveness and model performance over time.

8. **Share-by-link with optional password** — addresses the gap identified in the features survey where Hex exposes entire workspace projects when link-sharing. Each notebook can have independent share links with granular permissions.
