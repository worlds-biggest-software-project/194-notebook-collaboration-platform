# Data Model Suggestion 4: Graph-Relational (Relationship-First)

> Project: Notebook Collaboration Platform · Created: 2026-05-20

## Philosophy

This model adds a property graph layer on top of a relational foundation to capture the rich, traversable relationships that emerge in a collaborative notebook platform. While the operational CRUD (creating notebooks, running cells, managing users) uses standard relational tables, a `graph_nodes` / `graph_edges` overlay models the connections between entities: notebooks that share data connections, users who collaborate frequently, cells that reference outputs from other cells, execution dependencies between notebooks, and data lineage from source tables through transformations to published dashboards.

The motivation is that notebook platforms are fundamentally about relationships: a data scientist's work connects people (collaborators), data (sources, transformations, outputs), code (imports, dependencies), and artefacts (dashboards, reports, models). These relationships are first-class query targets — "which notebooks depend on this data source?", "who else has worked with this dataset?", "what is the data lineage from raw table to published dashboard?". In a purely relational model, these queries require multi-hop JOINs across many tables. In a graph model, they are natural traversals.

This approach is used by Databricks Unity Catalog for data lineage, by GitHub for repository/contributor networks, and by knowledge management platforms for concept graphs. It is implementable in PostgreSQL using ltree for hierarchies and adjacency tables for general graphs, or with a dedicated graph database (Neo4j, Apache AGE) alongside PostgreSQL.

**Best for:** Teams that need data lineage tracking, cross-notebook dependency analysis, AI-powered recommendations ("users who used this dataset also used..."), and conflict-of-interest or access-pattern analysis across the workspace.

**Trade-offs:**
- (+) Natural modelling of relationships: lineage, dependencies, collaboration patterns
- (+) Efficient graph traversal queries for multi-hop relationships
- (+) Enables AI-powered recommendations and insights based on graph analysis
- (+) Data lineage visualization from source to dashboard is a direct graph query
- (+) Flexible: new relationship types added as edge types without schema changes
- (-) Dual storage: relational tables for CRUD, graph layer for relationships — data can drift
- (-) Graph query language (Cypher, GQL, or recursive CTEs) adds learning curve
- (-) Graph indexes and traversals have different performance characteristics than B-tree indexes
- (-) Maintaining the graph layer requires event-driven updates or triggers
- (-) Overkill if the platform is small-scale or relationship queries are not a priority

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| nbformat v4 | Notebook content stored as JSONB (same as Hybrid model); graph layer adds relationship metadata on top |
| GQL (ISO/IEC 39075) | Graph queries follow the emerging GQL standard for property graph querying |
| Apache TinkerPop / Gremlin | Edge/node schema compatible with TinkerPop property graph model if migrating to a dedicated graph DB |
| OpenLineage | Data lineage edges follow OpenLineage's job → dataset → job model for interoperability |
| OAuth 2.0 / OIDC | Standard auth tables; graph layer tracks collaboration relationships derived from auth events |
| FAIR4RS | Provenance graph captures the full chain from data source to published output, supporting FAIR traceability |
| W3C PROV-O | Provenance edges (used_by, derived_from, generated_by) align with W3C PROV ontology concepts |

---

## Relational Foundation

```sql
-- Standard relational tables for operational CRUD (abbreviated — same core structure
-- as the Hybrid model for workspaces, users, memberships, tokens)

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
    profile JSONB NOT NULL DEFAULT '{}',
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'editor',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_workspace ON projects(workspace_id);

CREATE TABLE notebooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    content JSONB NOT NULL DEFAULT '{"nbformat": 4, "nbformat_minor": 5, "metadata": {}, "cells": []}',
    kernel_name VARCHAR(100) GENERATED ALWAYS AS (content->'metadata'->'kernelspec'->>'name') STORED,
    cell_count INT GENERATED ALWAYS AS (jsonb_array_length(COALESCE(content->'cells', '[]'::jsonb))) STORED,
    created_by UUID NOT NULL REFERENCES users(id),
    last_edited_by UUID REFERENCES users(id),
    last_edited_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notebooks_project ON notebooks(project_id);

CREATE TABLE data_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    connector_type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,
    credentials_encrypted BYTEA,
    schema_cache JSONB,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE execution_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    trigger_type VARCHAR(50) NOT NULL,
    triggered_by UUID REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'queued',
    context JSONB NOT NULL DEFAULT '{}',
    result JSONB,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    duration_ms BIGINT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_execution_runs_notebook ON execution_runs(notebook_id, created_at DESC);

CREATE TABLE scheduled_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    cron_expression VARCHAR(100) NOT NULL,
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT true,
    config JSONB NOT NULL DEFAULT '{}',
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notebook_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    content JSONB NOT NULL,
    message TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (notebook_id, version_number)
);

CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    cell_id VARCHAR(100),
    parent_comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id),
    body TEXT NOT NULL,
    resolved BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE collaboration_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    yjs_document_id VARCHAR(255) NOT NULL UNIQUE,
    yjs_state BYTEA,
    last_update_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE published_dashboards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    config JSONB NOT NULL,
    published_by UUID NOT NULL REFERENCES users(id),
    published_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Graph Layer

```sql
-- Entity type registry for the graph
CREATE TYPE graph_node_type AS ENUM (
    'workspace', 'project', 'notebook', 'cell',
    'user', 'data_connection', 'dataset', 'table', 'column',
    'execution_run', 'dashboard', 'model', 'environment'
);

-- Edge type registry
CREATE TYPE graph_edge_type AS ENUM (
    -- Structural
    'contains',           -- workspace->project, project->notebook, notebook->cell
    'member_of',          -- user->workspace
    
    -- Data lineage (OpenLineage-aligned)
    'reads_from',         -- notebook->data_connection, cell->table
    'writes_to',          -- notebook->table, execution->dataset
    'derived_from',       -- output_table->source_table, notebook->notebook
    'transforms',         -- cell->table (the cell transforms this table's data)
    
    -- Collaboration
    'edited_by',          -- notebook->user (weighted by edit count)
    'reviewed_by',        -- notebook->user (commented on)
    'shared_with',        -- notebook->user
    'collaborates_with',  -- user->user (co-edited same notebooks)
    
    -- Dependency
    'depends_on',         -- notebook->notebook (imports outputs from another)
    'imports',            -- cell->library/package
    'uses_model',         -- notebook->model (ML model reference)
    'scheduled_after',    -- notebook->notebook (execution ordering)
    
    -- Publication
    'published_as',       -- notebook->dashboard
    'feeds',              -- data_connection->notebook->dashboard (end-to-end lineage)
    
    -- AI
    'similar_to',         -- notebook->notebook (content similarity)
    'recommended'         -- user->notebook (AI recommendation)
);

CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id UUID NOT NULL,  -- references the actual entity in its relational table
    node_type graph_node_type NOT NULL,
    workspace_id UUID NOT NULL,  -- tenant scoping
    label VARCHAR(255) NOT NULL,  -- human-readable label
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties example for a 'table' node:
    -- {
    --   "connection_id": "...",
    --   "schema": "public",
    --   "table_name": "sales",
    --   "column_count": 12,
    --   "row_count_approx": 1500000,
    --   "last_queried_at": "2026-05-20T10:00:00Z"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entity_id, node_type)
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type, workspace_id);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_nodes_workspace ON graph_nodes(workspace_id);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN(properties jsonb_path_ops);

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type graph_edge_type NOT NULL,
    workspace_id UUID NOT NULL,  -- denormalised for tenant-scoped queries
    weight FLOAT DEFAULT 1.0,  -- for weighted relationships (e.g., edit frequency)
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties example for 'reads_from':
    -- {
    --   "columns_accessed": ["id", "amount", "created_at"],
    --   "query_pattern": "SELECT",
    --   "first_accessed_at": "2026-03-01T10:00:00Z",
    --   "access_count": 47
    -- }
    -- properties example for 'collaborates_with':
    -- {
    --   "shared_notebooks": 5,
    --   "co_editing_sessions": 12,
    --   "first_collaboration": "2026-01-15T10:00:00Z"
    -- }
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to TIMESTAMPTZ,  -- NULL = currently valid; set to expire temporal edges
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type, workspace_id);
CREATE INDEX idx_graph_edges_workspace ON graph_edges(workspace_id);
CREATE INDEX idx_graph_edges_temporal ON graph_edges(valid_from, valid_to)
    WHERE valid_to IS NOT NULL;
```

## Lineage Tracking (OpenLineage-Compatible)

```sql
-- Materialized lineage view: traces data flow from source to output
-- Updated when notebooks execute and access data connections

CREATE TABLE lineage_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_run_id UUID NOT NULL REFERENCES execution_runs(id) ON DELETE CASCADE,
    notebook_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,  -- 'read', 'write', 'transform'
    source JSONB NOT NULL,
    -- source example:
    -- {
    --   "connection_id": "...",
    --   "type": "table",
    --   "namespace": "postgres://analytics-db",
    --   "name": "public.sales",
    --   "columns": ["id", "amount", "region", "created_at"]
    -- }
    target JSONB,
    -- target example (for writes):
    -- {
    --   "type": "dataframe",
    --   "name": "df_sales_summary",
    --   "columns": ["region", "total_sales", "avg_order_value"]
    -- }
    cell_id VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lineage_events_run ON lineage_events(execution_run_id);
CREATE INDEX idx_lineage_events_notebook ON lineage_events(notebook_id);
CREATE INDEX idx_lineage_events_source ON lineage_events USING GIN(source jsonb_path_ops);
```

## Example Graph Queries

```sql
-- Data lineage: trace the full path from a source table to all outputs
-- "What notebooks read from the sales table, and what dashboards do they feed?"
WITH RECURSIVE lineage AS (
    -- Start: find the graph node for the 'sales' table
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.node_type,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.node_type = 'table'
      AND gn.properties->>'table_name' = 'sales'
      AND gn.workspace_id = 'workspace-uuid'
    
    UNION ALL
    
    -- Traverse: follow edges downstream
    SELECT
        gn2.id,
        gn2.label,
        gn2.node_type,
        l.depth + 1,
        l.path || gn2.id
    FROM lineage l
    JOIN graph_edges ge ON ge.source_node_id = l.node_id
    JOIN graph_nodes gn2 ON gn2.id = ge.target_node_id
    WHERE ge.edge_type IN ('reads_from', 'derived_from', 'transforms', 'published_as', 'feeds')
      AND ge.valid_to IS NULL  -- only current edges
      AND gn2.id != ALL(l.path)  -- prevent cycles
      AND l.depth < 10  -- limit traversal depth
)
SELECT node_id, label, node_type, depth
FROM lineage
ORDER BY depth;

-- Collaboration graph: find users who frequently co-edit with a given user
SELECT
    u.display_name,
    u.email,
    ge.weight AS collaboration_strength,
    ge.properties->>'shared_notebooks' AS shared_notebooks
FROM graph_edges ge
JOIN graph_nodes source_node ON source_node.id = ge.source_node_id
JOIN graph_nodes target_node ON target_node.id = ge.target_node_id
JOIN users u ON u.id = target_node.entity_id
WHERE source_node.entity_id = 'current-user-uuid'
  AND source_node.node_type = 'user'
  AND ge.edge_type = 'collaborates_with'
  AND ge.valid_to IS NULL
ORDER BY ge.weight DESC
LIMIT 10;

-- Impact analysis: what would break if we change this data connection?
-- "Which notebooks, dashboards, and scheduled runs depend on connection X?"
WITH RECURSIVE dependents AS (
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.node_type,
        0 AS depth
    FROM graph_nodes gn
    WHERE gn.entity_id = 'connection-uuid'
      AND gn.node_type = 'data_connection'
    
    UNION ALL
    
    SELECT
        gn2.id,
        gn2.label,
        gn2.node_type,
        d.depth + 1
    FROM dependents d
    JOIN graph_edges ge ON ge.target_node_id = d.node_id
    JOIN graph_nodes gn2 ON gn2.id = ge.source_node_id
    WHERE ge.edge_type IN ('reads_from', 'depends_on', 'published_as')
      AND ge.valid_to IS NULL
      AND d.depth < 5
)
SELECT DISTINCT node_type, label, depth
FROM dependents
WHERE depth > 0
ORDER BY depth, node_type;

-- AI recommendation: notebooks similar to what a user has been working on
SELECT
    target_node.entity_id AS recommended_notebook_id,
    n.name AS notebook_name,
    ge.weight AS similarity_score,
    ge.properties->>'reason' AS recommendation_reason
FROM graph_edges ge
JOIN graph_nodes source_node ON source_node.id = ge.source_node_id
JOIN graph_nodes target_node ON target_node.id = ge.target_node_id
JOIN notebooks n ON n.id = target_node.entity_id
WHERE source_node.entity_id = 'user-uuid'
  AND source_node.node_type = 'user'
  AND ge.edge_type = 'recommended'
  AND ge.valid_to IS NULL
ORDER BY ge.weight DESC
LIMIT 5;

-- Cross-notebook dependency map for a project
SELECT
    src_n.name AS source_notebook,
    tgt_n.name AS target_notebook,
    ge.edge_type,
    ge.properties
FROM graph_edges ge
JOIN graph_nodes src ON src.id = ge.source_node_id
JOIN graph_nodes tgt ON tgt.id = ge.target_node_id
JOIN notebooks src_n ON src_n.id = src.entity_id
JOIN notebooks tgt_n ON tgt_n.id = tgt.entity_id
WHERE src.node_type = 'notebook'
  AND tgt.node_type = 'notebook'
  AND ge.edge_type IN ('depends_on', 'derived_from', 'scheduled_after')
  AND ge.workspace_id = 'workspace-uuid'
  AND ge.valid_to IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | workspaces, users, workspace_memberships |
| Projects & Notebooks | 2 | projects, notebooks (JSONB content) |
| Execution & Scheduling | 2 | execution_runs, scheduled_runs |
| Version Control | 1 | notebook_versions |
| Collaboration | 3 | comments, collaboration_sessions, published_dashboards |
| Data Connections | 1 | data_connections |
| Graph Layer | 3 | graph_nodes, graph_edges, lineage_events |
| **Total** | **15** | 12 relational + 3 graph layer tables |

---

## Key Design Decisions

1. **Graph layer is an overlay, not a replacement** — the relational tables are the source of truth for operational data. The graph layer is a derived, queryable index of relationships. If the graph tables were dropped, the application would still function for basic CRUD; only relationship queries and lineage tracking would be lost. This separation means the graph can be rebuilt from relational data at any time.

2. **Typed nodes and edges with JSONB properties** — using enums for `node_type` and `edge_type` provides schema-level documentation of all relationship types while JSONB properties allow each edge type to carry different metadata. This balances structure (you know what relationship types exist) with flexibility (each type carries different data).

3. **Temporal edges with valid_from / valid_to** — relationships change over time. A user may stop collaborating with another, a notebook may switch data sources, a dashboard may be unpublished. Temporal edges preserve the historical graph for time-travel queries ("who was collaborating on this notebook in March?") while `valid_to IS NULL` filters give the current state.

4. **OpenLineage-compatible lineage events** — the `lineage_events` table captures data access patterns during notebook execution, following the OpenLineage model of job -> dataset -> job. These events are used to build and update the `reads_from`, `writes_to`, and `derived_from` edges in the graph layer, enabling data lineage visualization.

5. **Weighted edges for collaboration strength** — the `weight` column on `graph_edges` enables ranking relationships. For `collaborates_with` edges, weight increases with co-editing frequency. For `similar_to` edges, weight reflects content similarity. For `reads_from` edges, weight reflects access frequency. This enables AI-powered recommendations: "notebooks you might find useful" ranked by graph proximity and edge weight.

6. **Workspace-scoped graph for multi-tenancy** — both `graph_nodes` and `graph_edges` carry a `workspace_id` column (denormalised from the source entity). All graph queries filter by workspace, and PostgreSQL Row-Level Security can enforce this at the database level. Cross-workspace edges are not supported to maintain tenant isolation.

7. **Graph maintained by event-driven updates** — when a notebook is created, edited, executed, or shared, the application emits events that update the graph layer. This decoupled approach means the graph can lag slightly behind the relational state but avoids tight coupling between CRUD operations and graph maintenance. A background worker processes graph update events.

8. **Recursive CTEs for traversal in PostgreSQL** — while a dedicated graph database (Neo4j, Apache AGE) would provide more efficient traversal, PostgreSQL's recursive CTEs handle the expected scale of a notebook platform (thousands of notebooks per workspace, not billions of nodes). This avoids the operational complexity of a second database. If scale demands it, the graph layer can be migrated to a dedicated graph DB without changing the relational foundation.
