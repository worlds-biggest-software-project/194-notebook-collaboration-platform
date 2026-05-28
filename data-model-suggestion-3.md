# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Notebook Collaboration Platform · Created: 2026-05-20

## Philosophy

This model uses a pragmatic hybrid approach: core structural relationships (workspaces, projects, notebooks, users) are modelled as normalised relational tables with foreign keys, while variable, domain-specific, or frequently evolving data is stored in JSONB columns. The notebook content itself — cells, outputs, metadata — lives as a single JSONB document within the notebook row, mirroring how nbformat already stores notebooks as monolithic JSON files.

This design is inspired by how PostgreSQL-backed SaaS platforms handle the tension between structural integrity and flexibility. The notebook document model is inherently JSON (nbformat is a JSON spec), so storing it as JSONB preserves its native format without decomposition overhead. Meanwhile, the organisational layer (who owns what, who can access what, when was it last run) benefits from relational structure for efficient querying and referential integrity.

The hybrid approach is particularly well-suited for a notebook platform because: (a) the notebook document format is already JSON and storing it relationally requires lossy decomposition, (b) different kernel types may produce outputs with wildly different structures, (c) metadata varies by deployment context (enterprise vs. research vs. education), and (d) rapid iteration on the data model is possible without schema migrations for JSONB fields.

**Best for:** Teams prioritising rapid development, MVP-first iteration, and deployment flexibility across diverse environments — particularly when the notebook document format should be preserved in its native JSON form.

**Trade-offs:**
- (+) Fastest path to MVP — notebook content stored as-is from nbformat, no decomposition
- (+) Schema flexibility — new metadata, output types, and kernel-specific fields without migrations
- (+) Excellent read performance for single-notebook operations (one row fetch)
- (+) Natural alignment with nbformat: import/export is trivial (the content column IS the .ipynb)
- (+) JSONB indexing (GIN) enables efficient queries into document structure
- (-) Cross-notebook queries on cell content require JSONB path expressions (slower than relational JOINs)
- (-) No referential integrity within the JSONB document (e.g., cannot FK a cell ID)
- (-) Large JSONB documents (notebooks with many outputs) can cause TOAST bloat
- (-) Concurrent JSONB updates require application-level conflict resolution
- (-) Harder to enforce data quality constraints on JSONB fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| nbformat v4 | The `content` JSONB column stores the exact nbformat v4 JSON structure — zero transformation on import/export |
| JSON Schema Draft 2020-12 | JSONB content can be validated against the official nbformat JSON Schema at the application layer |
| Jupyter Server REST API | API responses can return the `content` JSONB directly, matching Jupyter Server's contents API response format |
| OAuth 2.0 / OIDC | Relational auth tables follow standard OIDC patterns; JSONB `auth_claims` stores provider-specific fields |
| OpenAPI 3.1 | API schema uses JSON Schema (aligned with OpenAPI 3.1) for documenting JSONB structures |
| SCIM 2.0 | User provisioning maps to relational user table; extensible JSONB profile stores SCIM extension attributes |
| FAIR4RS | Notebook-level JSONB metadata supports arbitrary FAIR-compliant provenance fields |

---

## Identity & Multi-Tenancy

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_kernel": "python3",
    --   "max_concurrent_executions": 5,
    --   "allowed_data_connectors": ["postgresql", "snowflake", "bigquery"],
    --   "branding": {"logo_url": "...", "primary_color": "#4F46E5"},
    --   "security": {"ip_allowlist": ["10.0.0.0/8"], "session_timeout_minutes": 480}
    -- }
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
    profile JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "timezone": "America/New_York",
    --   "preferred_kernel": "python3",
    --   "theme": "dark",
    --   "notifications": {"email": true, "slack": false},
    --   "scim_extension": {"department": "Data Science", "cost_center": "DS-001"}
    -- }
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'editor',
    permissions JSONB NOT NULL DEFAULT '{}',
    -- permissions example (for fine-grained overrides beyond role):
    -- {
    --   "can_manage_connections": true,
    --   "can_publish_dashboards": false,
    --   "max_scheduled_runs": 10
    -- }
    joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);

CREATE INDEX idx_ws_memberships_workspace ON workspace_memberships(workspace_id);
CREATE INDEX idx_ws_memberships_user ON workspace_memberships(user_id);

CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    token_hash VARCHAR(64) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    scopes JSONB NOT NULL DEFAULT '[]',  -- ["read:notebooks", "execute:notebooks"]
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Projects & Notebooks (JSONB Content)

```sql
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_environment": "docker.io/myorg/ds-env:latest",
    --   "git_repo": {"url": "https://github.com/...", "branch": "main"},
    --   "shared_connections": ["conn-uuid-1", "conn-uuid-2"]
    -- }
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
    
    -- The notebook content as native nbformat v4 JSON
    -- This IS the .ipynb file — no transformation needed for import/export
    content JSONB NOT NULL DEFAULT '{
        "nbformat": 4,
        "nbformat_minor": 5,
        "metadata": {
            "kernelspec": {"display_name": "Python 3", "language": "python", "name": "python3"},
            "language_info": {"name": "python", "version": "3.11"}
        },
        "cells": []
    }',
    -- content follows the nbformat v4 schema exactly:
    -- {
    --   "nbformat": 4,
    --   "nbformat_minor": 5,
    --   "metadata": {
    --     "kernelspec": {"display_name": "Python 3", "language": "python", "name": "python3"},
    --     "language_info": {"name": "python", "version": "3.11.0", "mimetype": "text/x-python"}
    --   },
    --   "cells": [
    --     {
    --       "id": "cell-uuid-1",
    --       "cell_type": "markdown",
    --       "source": "# My Analysis\nExploring Q3 sales data",
    --       "metadata": {}
    --     },
    --     {
    --       "id": "cell-uuid-2",
    --       "cell_type": "code",
    --       "source": "import pandas as pd\ndf = pd.read_sql('SELECT * FROM sales', conn)",
    --       "metadata": {},
    --       "execution_count": 1,
    --       "outputs": [
    --         {"output_type": "execute_result", "execution_count": 1,
    --          "data": {"text/html": "<table>...</table>", "text/plain": "   sales  ..."}}
    --       ]
    --     }
    --   ]
    -- }
    
    -- Extracted/denormalised fields for efficient queries without JSONB traversal
    kernel_name VARCHAR(100) GENERATED ALWAYS AS (content->'metadata'->'kernelspec'->>'name') STORED,
    language VARCHAR(50) GENERATED ALWAYS AS (content->'metadata'->'language_info'->>'name') STORED,
    cell_count INT GENERATED ALWAYS AS (jsonb_array_length(COALESCE(content->'cells', '[]'::jsonb))) STORED,
    
    -- Collaboration metadata
    created_by UUID NOT NULL REFERENCES users(id),
    last_edited_by UUID REFERENCES users(id),
    last_edited_at TIMESTAMPTZ,
    
    -- Tags and custom metadata (outside nbformat, platform-specific)
    tags TEXT[] NOT NULL DEFAULT '{}',
    platform_metadata JSONB NOT NULL DEFAULT '{}',
    -- platform_metadata example:
    -- {
    --   "environment_image": "docker.io/myorg/ds-env:v2",
    --   "fair_metadata": {"doi": "10.1234/...", "license": "CC-BY-4.0"},
    --   "ai_config": {"model": "claude-sonnet-4", "context_depth": "full"}
    -- }
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notebooks_project ON notebooks(project_id);
CREATE INDEX idx_notebooks_kernel ON notebooks(kernel_name);
CREATE INDEX idx_notebooks_tags ON notebooks USING GIN(tags);

-- GIN index for querying into notebook content (e.g., find notebooks containing specific imports)
CREATE INDEX idx_notebooks_content ON notebooks USING GIN(content jsonb_path_ops);
```

## Version Control

```sql
CREATE TABLE notebook_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    content JSONB NOT NULL,  -- full .ipynb snapshot at this version
    message TEXT,
    diff_summary TEXT,  -- AI-generated plain-language summary of changes
    diff_stats JSONB,
    -- diff_stats example:
    -- {
    --   "cells_added": 2,
    --   "cells_deleted": 0,
    --   "cells_modified": 3,
    --   "lines_added": 45,
    --   "lines_deleted": 12
    -- }
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (notebook_id, version_number)
);

CREATE INDEX idx_versions_notebook ON notebook_versions(notebook_id, version_number DESC);
```

## Scheduling & Execution

```sql
CREATE TABLE scheduled_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    name VARCHAR(255),
    cron_expression VARCHAR(100) NOT NULL,
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT true,
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "parameters": {"date_range": "last_7_days", "region": "us-east"},
    --   "compute": {"cpu": "2", "memory": "4Gi", "gpu": null, "timeout_seconds": 3600},
    --   "notifications": [
    --     {"on": "failure", "channel": "email", "target": "team@company.com"},
    --     {"on": "success", "channel": "webhook", "url": "https://hooks.slack.com/..."}
    --   ],
    --   "retry": {"max_attempts": 3, "delay_seconds": 60}
    -- }
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scheduled_runs_notebook ON scheduled_runs(notebook_id);

CREATE TABLE execution_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    scheduled_run_id UUID REFERENCES scheduled_runs(id) ON DELETE SET NULL,
    trigger_type VARCHAR(50) NOT NULL,  -- 'manual', 'scheduled', 'api', 'webhook'
    triggered_by UUID REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'queued',
    
    -- Execution context and results as JSONB
    context JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "parameters": {"date_range": "last_7_days"},
    --   "environment": {"image": "python:3.11", "packages": ["pandas==2.1", "scikit-learn==1.4"]},
    --   "compute": {"cpu": "2", "memory": "4Gi"},
    --   "kernel_id": "kernel-uuid"
    -- }
    
    result JSONB,
    -- result example:
    -- {
    --   "notebook_output": <full .ipynb with outputs>,
    --   "cell_results": [
    --     {"cell_id": "...", "status": "ok", "duration_ms": 234},
    --     {"cell_id": "...", "status": "error", "error": {"ename": "ValueError", "evalue": "..."}}
    --   ],
    --   "metrics": {"peak_memory_mb": 1024, "total_cpu_seconds": 45.2}
    -- }
    
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_ms BIGINT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_execution_runs_notebook ON execution_runs(notebook_id, created_at DESC);
CREATE INDEX idx_execution_runs_status ON execution_runs(status);
```

## Data Connections

```sql
CREATE TABLE data_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    connector_type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,
    -- config structure varies by connector_type:
    -- PostgreSQL: {"host": "...", "port": 5432, "database": "...", "schema": "public", "ssl": true}
    -- Snowflake:  {"account": "...", "warehouse": "...", "database": "...", "schema": "..."}
    -- BigQuery:   {"project_id": "...", "dataset": "...", "location": "us-east1"}
    -- S3:         {"bucket": "...", "region": "us-east-1", "prefix": "data/"}
    credentials_encrypted BYTEA,
    schema_cache JSONB,  -- cached database schema for AI context
    -- schema_cache example:
    -- {
    --   "tables": [
    --     {"name": "sales", "columns": [
    --       {"name": "id", "type": "integer", "nullable": false},
    --       {"name": "amount", "type": "numeric(10,2)", "nullable": false},
    --       {"name": "created_at", "type": "timestamptz", "nullable": false}
    --     ]}
    --   ],
    --   "cached_at": "2026-05-20T10:00:00Z"
    -- }
    created_by UUID NOT NULL REFERENCES users(id),
    last_tested_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_connections_workspace ON data_connections(workspace_id);
```

## Collaboration & Sharing

```sql
CREATE TABLE collaboration_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    yjs_document_id VARCHAR(255) NOT NULL UNIQUE,
    yjs_state BYTEA,
    awareness JSONB NOT NULL DEFAULT '{}',
    -- awareness example (Yjs Awareness CRDT state, denormalised for API responses):
    -- {
    --   "user-uuid-1": {"cursor": {"cell_id": "...", "offset": 42}, "color": "#E11D48", "name": "Alice"},
    --   "user-uuid-2": {"cursor": {"cell_id": "...", "offset": 10}, "color": "#2563EB", "name": "Bob"}
    -- }
    last_update_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notebook_shares (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    share_config JSONB NOT NULL,
    -- share_config example:
    -- {
    --   "type": "link",
    --   "permission": "view",
    --   "link_token": "abc123...",
    --   "password_hash": "...",
    --   "expires_at": "2026-06-01T00:00:00Z",
    --   "allow_download": true,
    --   "require_auth": false
    -- }
    -- OR:
    -- {
    --   "type": "user",
    --   "user_id": "...",
    --   "user_email": "user@example.com",
    --   "permission": "edit"
    -- }
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notebook_shares_notebook ON notebook_shares(notebook_id);
-- GIN index for querying share config (e.g., find all link shares)
CREATE INDEX idx_notebook_shares_config ON notebook_shares USING GIN(share_config jsonb_path_ops);

CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    cell_id VARCHAR(100),  -- references cell ID within the JSONB content
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
```

## AI Features

```sql
CREATE TABLE ai_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    interaction_type VARCHAR(50) NOT NULL,  -- 'completion', 'narrative', 'diff_summary', 'anomaly'
    details JSONB NOT NULL,
    -- completion example:
    -- {
    --   "cell_id": "...",
    --   "prompt_hash": "...",
    --   "context": {"variables": ["df", "model"], "schemas": ["sales"], "prior_outputs": 3},
    --   "suggestion": "df.groupby('region').agg({'sales': 'sum'})",
    --   "accepted": true,
    --   "model": "claude-sonnet-4",
    --   "latency_ms": 450
    -- }
    -- anomaly example:
    -- {
    --   "run_id": "...",
    --   "cell_id": "...",
    --   "alert_type": "distribution_shift",
    --   "severity": "warning",
    --   "expected": {"mean": 42.3, "std": 2.1},
    --   "actual": {"mean": 67.8, "std": 15.2},
    --   "acknowledged": false
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_interactions_notebook ON ai_interactions(notebook_id, created_at DESC);
CREATE INDEX idx_ai_interactions_type ON ai_interactions(interaction_type, created_at DESC);
```

## Published Dashboards

```sql
CREATE TABLE published_dashboards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    config JSONB NOT NULL,
    -- config example:
    -- {
    --   "layout": [
    --     {"cell_id": "cell-uuid-1", "type": "chart", "x": 0, "y": 0, "w": 6, "h": 4},
    --     {"cell_id": "cell-uuid-2", "type": "table", "x": 6, "y": 0, "w": 6, "h": 4},
    --     {"cell_id": "cell-uuid-3", "type": "metric", "x": 0, "y": 4, "w": 3, "h": 2}
    --   ],
    --   "inputs": [
    --     {"name": "date_range", "type": "date_picker", "default": "last_30_days"},
    --     {"name": "region", "type": "dropdown", "options": ["us", "eu", "apac"]}
    --   ],
    --   "auto_refresh_seconds": 300,
    --   "theme": "light"
    -- }
    published_by UUID NOT NULL REFERENCES users(id),
    published_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

```sql
-- Import a .ipynb file: simply INSERT the JSON as-is
INSERT INTO notebooks (project_id, name, content, created_by)
VALUES ('project-uuid', 'imported.ipynb', $ipynb_json$::jsonb, 'user-uuid');

-- Export a notebook: the content column IS the .ipynb
SELECT content FROM notebooks WHERE id = 'notebook-uuid';

-- Find all Python notebooks with more than 10 cells
SELECT id, name, cell_count
FROM notebooks
WHERE kernel_name = 'python3'
  AND cell_count > 10;

-- Search for notebooks containing a specific import (uses GIN index)
SELECT id, name
FROM notebooks
WHERE content @? '$.cells[*] ? (@.source like_regex "import tensorflow")';

-- Find all code cells across a project that use pandas
SELECT n.id AS notebook_id, n.name,
       cell->>'id' AS cell_id,
       cell->>'source' AS source
FROM notebooks n,
     jsonb_array_elements(n.content->'cells') AS cell
WHERE n.project_id = 'project-uuid'
  AND cell->>'cell_type' = 'code'
  AND cell->>'source' LIKE '%import pandas%';

-- Dashboard with notebook content — single query, no JOINs for content
SELECT d.title, d.config, n.content
FROM published_dashboards d
JOIN notebooks n ON n.id = d.notebook_id
WHERE d.slug = 'q3-sales-dashboard';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | workspaces, users, workspace_memberships, api_tokens |
| Projects & Notebooks | 2 | projects, notebooks (content as JSONB) |
| Version Control | 1 | notebook_versions |
| Scheduling & Execution | 2 | scheduled_runs, execution_runs |
| Data Connections | 1 | data_connections |
| Collaboration & Sharing | 3 | collaboration_sessions, notebook_shares, comments |
| AI Features | 1 | ai_interactions |
| Publishing | 1 | published_dashboards |
| **Total** | **15** | Fewer than normalized (22) due to JSONB consolidation |

---

## Key Design Decisions

1. **Notebook content stored as a single JSONB column matching nbformat exactly** — this is the defining choice. Import is `INSERT ... content = $file_json`, export is `SELECT content`. No ORM mapping, no decomposition, no reconstruction. The trade-off is that cell-level queries require JSONB path expressions, which are slower than relational JOINs but adequate for most workloads.

2. **Generated columns extract frequently-queried fields** — `kernel_name`, `language`, and `cell_count` are `GENERATED ALWAYS AS ... STORED` columns derived from the JSONB content. This avoids the need to traverse JSONB for common filter/sort operations while keeping the content column as the single source of truth.

3. **GIN index on content enables full-text search within notebooks** — the `jsonb_path_ops` GIN index supports containment queries (`@>`, `@?`) for searching cell source code across notebooks. This enables features like "find all notebooks that import tensorflow" without a separate search index.

4. **Configuration-heavy tables use JSONB for variability** — `scheduled_runs.config`, `data_connections.config`, `notebook_shares.share_config`, and `published_dashboards.config` all use JSONB because their structure varies by type. A PostgreSQL schedule notification differs from a Snowflake connection config, and JSONB handles this without subtype tables.

5. **AI interactions unified into a single table with type discriminator** — rather than separate tables for completions, anomalies, and narrative generation, a single `ai_interactions` table with JSONB details handles all AI feature types. New AI features (e.g., "smart scheduling") can be added by defining a new `interaction_type` without schema changes.

6. **Comments reference cell IDs as strings, not foreign keys** — since cells live inside the JSONB content column (not in their own table), comments reference cell IDs by string value. This means a deleted cell leaves orphaned comments, which the application must handle. The trade-off is acceptable because cells are rarely deleted, and orphaned comments can be cleaned up asynchronously.

7. **Version snapshots also store JSONB content** — each version is a complete .ipynb snapshot. Combined with `diff_stats` and `diff_summary` (AI-generated), this enables both precise reconstruction and human-readable version history without diff computation at read time.

8. **Schema cache on data connections for AI context** — the `schema_cache` JSONB column on `data_connections` stores a cached representation of the connected database's schema. This cache is passed to the AI model as context for code completion, enabling schema-aware suggestions without querying the remote database on every keystroke.
