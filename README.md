# Notebook Collaboration Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source collaborative notebook platform that is Jupyter-compatible and combines real-time co-editing, version control, and scheduled execution.

The Notebook Collaboration Platform is a cloud-hostable and self-hostable workspace for data science and ML teams who need more than a vanilla JupyterHub deployment but do not want to be locked into a proprietary SaaS. It targets the gap between bare-metal Jupyter (powerful but DevOps-heavy) and polished commercial tools (Deepnote, Hex, Databricks) by combining a `.ipynb`-compatible multiplayer notebook with AI capabilities that understand the full notebook state.

---

## Why Notebook Collaboration Platform?

- JupyterHub gives full control but ships no built-in collaboration UX, no scheduling, and no AI assistance — teams pay for that polish in DevOps time.
- Deepnote, Hex, and Databricks deliver strong collaboration but are proprietary SaaS with no self-hosting path; pricing in the contested mid-market sits at roughly $24–$40/editor/month.
- Google Colab is free but throttles compute, has no team management, and offers Python only.
- Briefer is the only fully open-source tool with native multiplayer plus dashboards, but its community, integrations, and AI features are still early-stage.
- Across every incumbent, raw cell diffs, manual scheduling, and isolated single-notebook collaboration leave clear room for an AI-native open alternative.

---

## Key Features

### Real-Time Collaborative Notebook

- Multiplayer editing with presence indicators using Yjs/CRDT (BSD-compatible)
- `.ipynb` import and export for portability across the Jupyter ecosystem
- Mixed Python and SQL canvas in a single notebook
- Cloud-hosted persistent storage with share-by-link
- Role-based access control (viewer, editor, admin)

### Versioning and Git Integration

- Automatic version history with per-change audit trail
- Git integration with commit, branch, and visual diff viewer
- Semantic version diffing: plain-language summaries of what changed between runs
- Compatibility with existing Git-based review workflows

### Scheduling and Orchestration

- Scheduled notebook execution with email or webhook notifications
- REST API and webhooks for triggering runs from external systems
- Foundation for future cross-notebook pipeline orchestration

### AI-Native Assistance

- Code completion that reads full notebook state — variables, schemas, prior outputs
- Automated narrative generation: prose suggestions between code blocks
- Semantic diff summaries describing what changed and why it matters
- Output anomaly detection alerting collaborators when results deviate from expected ranges
- Intelligent scheduling that predicts runtime and right-sizes compute before the job starts

### Publishing and Extensibility

- Notebook-to-dashboard publishing for non-technical stakeholders
- Planned R and Julia kernel support beyond the Python and SQL MVP
- Self-hostable open-source edition for air-gapped or on-premise deployments

---

## AI-Native Advantage

Unlike incumbents that bolt generic autocomplete onto a notebook, this platform treats the notebook's live state — variable values, dataframe schemas, and prior cell outputs — as first-class context for the model. That same context unlocks features no current tool offers: plain-language semantic diffs between versions, narrative-gap detection that suggests explanatory prose between code blocks, anomaly detection on cell outputs, and AI-aware scheduling that predicts runtime from historical execution data before provisioning compute.

---

## Tech Stack & Deployment

The platform is designed for both managed cloud and self-hosted deployment, with parity between the two so teams can migrate without rewriting workflows. The notebook layer builds on the Jupyter ecosystem: `nbformat` for the document model, `.ipynb` for portability, and Yjs CRDTs for real-time collaboration (the same approach used by JupyterLab's `jupyter-collaboration` extension). Authentication uses OIDC / OAuth 2.0 for enterprise SSO. Reproducible environments follow the Binder / repo2docker pattern via Docker images. Integration surface includes a REST API and webhooks for external orchestration.

---

## Market Context

The broader data science platform market is valued in the tens of billions, with the notebook-specific segment estimated at several hundred million dollars and growing as data teams scale. Funding in the space is significant: Deepnote raised $23.8M (Index Ventures, Accel, Y Combinator) and Databricks is valued at over $62B as of 2024. Primary buyers are data science and ML teams at mid-size to large companies, university research groups, data engineers building reproducible pipelines, and analysts who want notebooks without the infrastructure burden.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
