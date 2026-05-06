# Standards & API Reference

> Project: Notebook Collaboration Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### Jupyter & Notebook Format Standards

**nbformat — Jupyter Notebook Format Specification**
- URL: https://nbformat.readthedocs.io/en/latest/
- GitHub: https://github.com/jupyter/nbformat
- The de facto standard for representing interactive notebooks. Notebooks are JSON documents containing cells (code, markdown, raw), metadata, and outputs. The current major version is nbformat 4. Any new notebook platform must support this format for ecosystem compatibility. Licensed BSD 3-Clause.

**JupyterLab Server REST API**
- URL: https://jupyterlab-server.readthedocs.io/en/stable/api/rest.html
- OpenAPI YAML spec: https://github.com/jupyterlab/jupyterlab_server/blob/main/jupyterlab_server/rest-api.yml
- Defines the REST endpoints served by the JupyterLab Server, including extension listings and configuration. The spec is documented in OpenAPI 3 format.

**Jupyter Server REST API**
- URL: https://jupyter-server.readthedocs.io/en/latest/developers/rest-api.html
- The reference server implementation used by JupyterLab and compatible platforms. Defines endpoints for kernel management, session management, content (notebook) CRUD, and terminal access. Underpins the interoperability of all Jupyter-compatible clients.

**nbconvert**
- URL: https://nbconvert.readthedocs.io/
- Standard toolchain for converting `.ipynb` notebooks to HTML, PDF, LaTeX, scripts, and slideshows. Any platform offering export functionality should align with or extend nbconvert's template system.

**repo2docker / Binder**
- URL: https://repo2docker.readthedocs.io/
- De facto standard for building reproducible computational environments from a Git repository. Defines a convention for `requirements.txt`, `environment.yml`, `Dockerfile`, and other environment specification files. Directly relevant to reproducibility features.

---

### Real-Time Collaboration Standards

**Yjs — CRDT Library (de facto standard for collaborative notebooks)**
- URL: https://github.com/yjs/yjs
- Documentation: https://docs.yjs.dev/
- Yjs is a JavaScript CRDT (Conflict-free Replicated Data Type) library adopted by JupyterLab as the foundation of its real-time collaboration extension (`jupyter-collaboration`). It implements shared data types (Y.Text, Y.Map, Y.Array) that auto-merge concurrent edits without conflicts. Network-agnostic: works over WebSocket, WebRTC, or custom transports.

**jupyter-collaboration (Yjs-based RTC for JupyterLab)**
- URL: https://github.com/jupyterlab/jupyter-collaboration
- PyPI: https://pypi.org/project/jupyter-collaboration/
- The official Jupyter real-time collaboration extension, built on Yjs. Implements the Y-sync protocol for synchronising notebook documents across multiple clients. Version 4.2.1 released February 2026.

**W3C WebRTC Specification**
- URL: https://www.w3.org/TR/webrtc/
- W3C standard for browser-to-browser real-time communication. Used by Yjs's `y-webrtc` provider to enable peer-to-peer notebook collaboration without a central relay server. Co-specified with IETF.

**IETF RFC 8825 — Overview: Real-Time Protocols for Browser-Based Applications (WebRTC)**
- URL: https://datatracker.ietf.org/doc/html/rfc8825
- The IETF companion to the W3C WebRTC spec, defining the transport-layer protocols (DTLS, SRTP, ICE, STUN, TURN) underlying WebRTC-based collaboration.

---

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The standard for documenting REST APIs. Hex exposes a public OpenAPI-documented API; JupyterLab Server's REST API is described in OpenAPI YAML. Any new notebook platform should publish its API as an OAS 3.1 document.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/draft/2020-12/schema
- Foundational schema language used by OpenAPI 3.1 for request/response validation. nbformat uses JSON Schema to validate notebook structure. Relevant for defining data contracts in notebook APIs and export formats.

**IETF RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines HTTP method semantics (GET, POST, PUT, DELETE) and status codes. Baseline compliance is required for any REST API in the platform.

**IETF RFC 6570 — URI Template**
- URL: https://datatracker.ietf.org/doc/html/rfc6570
- Used by JupyterLab Server and OpenAPI to describe parameterised API endpoints. Relevant for defining notebook and kernel management API routes.

---

### Security & Authentication Standards

**OAuth 2.0 — IETF RFC 6749**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The industry-standard authorisation framework. JupyterHub implements OAuth 2.0 for its authentication flow; Deepnote and Databricks use OAuth 2.0 for SSO and API token issuance. Required for enterprise SSO integration.

**OpenID Connect (OIDC) 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Used by JupyterHub's pluggable authenticators (GitHub, Google, LDAP) and Databricks for enterprise identity federation. Required for SAML-free enterprise deployments.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- The authoritative reference for web application security risks. Directly relevant for: notebook execution sandboxing (code injection), credential storage (secrets in notebooks), session management, and access control for shared notebooks.

**NIST SP 800-63 — Digital Identity Guidelines**
- URL: https://pages.nist.gov/800-63-3/
- US federal standard for identity and authentication assurance levels. Relevant for enterprise and government deployments where compliance with identity assurance tiers is required.

**SCIM 2.0 — System for Cross-domain Identity Management (IETF RFC 7643/7644)**
- URL: https://datatracker.ietf.org/doc/html/rfc7643
- Standard for provisioning and de-provisioning user identities across enterprise systems. Databricks implements SCIM for enterprise user management. Required for any enterprise-tier notebook platform selling to large organisations.

---

### Data Science & Reproducibility Standards

**FAIR Data Principles**
- URL: https://www.go-fair.org/fair-principles/
- Publication: Wilkinson et al. (2016). *Scientific Data*. https://doi.org/10.1038/sdata.2016.18
- Findable, Accessible, Interoperable, Reusable — the guiding framework for scientific data management. Directly relevant to notebook platforms targeting research institutions. A notebook platform that implements FAIR-compliant data provenance and metadata tracking is differentiated in the academic and public-sector market.

**FAIR Principles for Research Software (FAIR4RS)**
- URL: https://www.nature.com/articles/s41597-022-01710-x
- Extension of FAIR principles to research software and computational notebooks. Defines expectations for how notebooks should be cited, versioned, and made interoperable.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Specification: https://spec.modelcontextprotocol.io/
- Anthropic's open standard for connecting AI models to tools and data sources. Relevant for exposing notebook state (cell outputs, variable values, schema) to AI coding agents within the platform. An MCP server wrapping the Jupyter Server REST API would allow any MCP-compatible LLM to read notebook context and write cells.

---

## Similar Products — Developer Documentation & APIs

### Deepnote

- **Description:** Cloud-based collaborative data science notebook platform with real-time multiplayer editing, AI agent, and data app publishing. Drop-in Jupyter replacement.
- **API Documentation:** https://deepnote.com/docs/deepnote-api
- **API Tracker:** https://apitracker.io/a/deepnote
- **GitHub (open-source components):** https://github.com/deepnote/deepnote
- **Developer Guide:** https://deepnote.com/docs/getting-started
- **Standards:** REST/JSON; Bearer token authentication
- **Authentication:** API key (bearer token); OAuth for SSO in enterprise tier
- **Key Capability:** The API primarily enables programmatic notebook execution; additional endpoints manage projects and webhooks.

---

### Hex

- **Description:** AI analytics platform combining Python + SQL notebooks with one-click data app publishing. Features a Notebook Agent powered by Claude Sonnet 4.
- **API Documentation:** https://learn.hex.tech/docs/api/api-overview
- **Public API Reference:** https://learn.hex.tech/docs/api/api-reference
- **API Tracker:** https://apitracker.io/a/hex-tech
- **Standards:** REST/JSON; OpenAPI 3 spec available for download
- **Authentication:** API key; OAuth for workspace authentication
- **Base URL:** `https://app.hex.tech/api/v1`
- **Key Capability:** API supports triggering project runs, managing projects, and workspace administration. OpenAPI spec downloadable from docs.

---

### JupyterHub

- **Description:** Open-source multi-user Jupyter server for teams, classrooms, and research institutions. Foundation for self-hosted notebook collaboration.
- **API Documentation:** https://jupyterhub.readthedocs.io/en/latest/howto/rest.html
- **REST API Reference:** https://jupyterhub.readthedocs.io/en/5.2.1/reference/rest-api.html
- **GitHub:** https://github.com/jupyterhub/jupyterhub
- **Developer Guide:** https://jupyterhub.readthedocs.io/
- **Standards:** REST/JSON; OpenAPI-describable endpoints; OAuth 2.0 for token scopes
- **Authentication:** API tokens (bearer); OAuth 2.0 scope-based authorisation (JupyterHub 2.0+)
- **Key Capability:** Full programmatic control over users, groups, servers, and services. RBAC via scopes introduced in v2.0.

---

### Jupyter Server (core REST API)

- **Description:** The reference server implementation underpinning JupyterLab, Jupyter Notebook, and compatible clients. Defines the REST API for kernel and session management.
- **API Documentation:** https://jupyter-server.readthedocs.io/en/latest/developers/rest-api.html
- **JupyterLab Server API (OpenAPI):** https://jupyterlab-server.readthedocs.io/en/stable/api/rest.html
- **OpenAPI YAML:** https://github.com/jupyterlab/jupyterlab_server/blob/main/jupyterlab_server/rest-api.yml
- **Standards:** REST/JSON; OpenAPI 3
- **Authentication:** Token-based; integrates with JupyterHub OAuth flow
- **Key Capability:** Endpoints for kernels, sessions, contents (CRUD for notebooks), terminals, and configuration. This is the API any Jupyter-compatible client must implement against.

---

### Databricks

- **Description:** Enterprise data lakehouse platform with collaborative multilingual notebooks, Spark integration, MLflow, and Delta Lake.
- **API Documentation:** https://docs.databricks.com/aws/en/reference/api
- **REST API Reference:** https://docs.databricks.com/api/workspace/introduction
- **Python SDK:** https://docs.databricks.com/aws/en/dev-tools/sdk-python
- **Python SDK GitHub:** https://github.com/databricks/databricks-sdk-py
- **Python SDK Docs:** https://databricks-sdk-py.readthedocs.io/
- **Developer Guide:** https://docs.databricks.com/aws/en/notebooks/notebooks-collaborate
- **Standards:** REST/JSON; SDKs available for Python, Java, Go, R
- **Authentication:** Personal access tokens; OAuth 2.0 M2M; Azure AD / AWS IAM integration
- **Key Capability:** Comprehensive APIs covering workspace, clusters, jobs, notebooks, Git folders, Unity Catalog, and MLflow. Databricks SDK for Python ships pre-installed on Runtime 13.3 LTS+.

---

### Google Colab Enterprise

- **Description:** Managed, security-hardened notebook environment on Google Cloud (Vertex AI). Inherits Google Cloud IAM, security, and compliance capabilities.
- **API Documentation:** https://docs.cloud.google.com/colab/docs
- **REST API Resources:** https://docs.cloud.google.com/colab/docs/reference/rest
- **Developer Guide:** https://developers.google.com/colab
- **Standards:** REST/JSON and gRPC; Google API Client Library conventions
- **Authentication:** Google OAuth 2.0; Application Default Credentials (ADC); Cloud IAM
- **Key Capability:** Runtime template management via `v1.projects.locations.notebookRuntimeTemplates`. Full Vertex AI API surface available within notebooks via ADC.

---

### Briefer (Open Source)

- **Description:** Open-source collaborative data platform combining multiplayer Python + SQL notebooks with built-in dashboard publishing and AI query generation.
- **GitHub:** https://github.com/briefercloud/briefer
- **Product:** https://briefer.cloud/
- **Standards:** REST/JSON (limited public API surface; primarily internal)
- **Authentication:** Self-hosted; configurable per deployment
- **Key Capability:** Open-source reference implementation for multiplayer notebooks with dashboard publishing. Codebase is inspectable for implementation patterns. Apache 2.0 licence.

---

## Notes

**Emerging / Evolving Areas:**

- **MCP as notebook integration layer**: The Model Context Protocol (MCP) is rapidly becoming the standard for connecting LLMs to tools. A notebook platform that exposes an MCP server wrapping the Jupyter Server REST API would allow any MCP-compatible AI agent to read notebook context, inspect variable states, and author cells — enabling a new class of autonomous data science agents.

- **Y-sync / Yjs as the de facto CRDT standard for notebooks**: While no formal W3C or IETF standard governs notebook-specific real-time sync, Yjs and the Y-sync protocol have become the practical standard through adoption by JupyterLab. New platforms should treat Yjs compatibility as interoperability infrastructure, not a vendor choice.

- **FAIR4RS compliance gap**: No commercial notebook platform currently markets explicit FAIR4RS compliance despite the academic and public-sector appetite for it. This represents both a positioning opportunity and a standards gap.

- **Security standards for notebook execution**: OWASP guidance does not have a notebook-specific profile. The sandboxing of arbitrary user code (kernel isolation, secrets management, output sanitisation) is entirely platform-defined. This is a risk area for any multi-tenant deployment.
