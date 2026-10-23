# ProjectOS — Technical Decisions

> **Document scope:** Architectural decision record for the public ProjectOS reference design, based on the [README](../README.md). These are documented design choices and tradeoffs, not a retrospective claim about undisclosed internal implementation details.

## Decision summary

| ID | Decision | Choice | Primary reason |
| --- | --- | --- | --- |
| ADR-001 | Backend structure | Modular FastAPI service | Simpler to build, test and deploy than microservices |
| ADR-002 | Primary store | Azure Database for PostgreSQL Flexible Server | Relational task/project structure and flexible metadata |
| ADR-003 | AI orchestration | PydanticAI + Azure OpenAI | Typed tool calling and structured responses |
| ADR-004 | AI data access | Allowlisted read-only service tools | Permission control and grounded responses |
| ADR-005 | Source connectivity | Separate Graph, Dataverse, file adapters | Accurate support for differing Microsoft product APIs |
| ADR-006 | Synchronization | Manual refresh / polling first | Reliability without unsupported webhook assumptions |
| ADR-007 | Shared task representation | Canonical schema | Source-agnostic dashboard, filters and AI |
| ADR-008 | Frontend | React + TypeScript + Tailwind | Reusable interactive UI with type-safe integration |
| ADR-009 | Exports | Server-generated filtered CSV | Easy interoperability and permission-aware reporting |

## ADR-001 — Use a modular FastAPI backend

**Context.** ProjectOS needs source ingestion, task queries, exports, and AI interactions. These operations share access control, task normalization and a database.

**Decision.** Use one Python FastAPI service organized into modules (`api`, `services`, `connectors`, `models`, `ai`, `db`). Background imports can be run by a separate worker process when required, without splitting into independent network services.

**Why.** One deployable backend minimizes infrastructure and simplifies debugging, testing and reuse of task-query logic by the REST API and AI tools.

**Tradeoff.** Long-running imports should not block HTTP workers. As demand grows, move sync jobs to an explicit job runner/queue; only split services where scaling or ownership justify it.

## ADR-002 — Use PostgreSQL instead of SQLite, Cosmos DB or Azure SQL

**Context.** Projects, tasks, users, assignments and dependencies are relational; source-specific metadata varies.

**Decision.** Use **Azure Database for PostgreSQL (Flexible Server)**, SQLAlchemy, and Alembic. Use relational tables for core fields plus a `JSONB` metadata field for source-specific attributes.

**Why.** PostgreSQL supports joins, constraints, indexes, concurrent access and flexible `JSONB` in one system. Local Docker-based PostgreSQL also closely matches the hosted database.

**Alternatives considered.**

| Alternative | Why not the default |
| --- | --- |
| SQLite | Excellent for a local prototype; less appropriate for shared hosted concurrency and managed operations |
| Azure SQL Database | Strong Microsoft-native alternative, but PostgreSQL fits this Python/SQLAlchemy design well |
| Azure Cosmos DB | A document model is less natural for many-to-many assignments and related project tasks; extra complexity is unnecessary |

**Tradeoff.** A managed PostgreSQL deployment incurs cost and operations overhead compared with SQLite. Do not imply that a relational database alone resolves cross-tool semantic duplicates.

## ADR-003 — Use PydanticAI instead of Semantic Kernel

**Context.** The assistant needs to answer operational questions over structured project data, not run complex autonomous multi-agent workflows.

**Decision.** Use **PydanticAI** to define typed tools and structured outputs, with **Azure OpenAI** providing the model. Maintain the tool implementations in the existing FastAPI service layer.

**Why.** PydanticAI works naturally with Python and Pydantic validation; it keeps tool contracts explicit and makes it straightforward to return an explanation together with matching task IDs or recommended dashboard filters.

**Alternatives considered.**

- **Semantic Kernel / Microsoft Agent Framework:** Valuable for Microsoft-centric and more complex agent workflows, but not required for a small read-only query assistant.
- **Direct Azure OpenAI SDK:** Fewer dependencies, but the app would implement more tool dispatch and response validation itself.
- **LangGraph:** Better suited to explicit multi-step stateful workflows than this initial scope requires.

**Tradeoff.** Agent frameworks introduce a dependency and model-call overhead. Reassess the choice if requests remain simple enough for direct function calling.

## ADR-004 — Read-only tools, not arbitrary AI-generated SQL

**Context.** The model needs to search tasks and summarize work, but should not acquire broad access to the database or other users' projects.

**Decision.** Expose a limited set of service-backed tools such as `get_overdue_tasks`, `get_project_summary`, `get_tasks_by_assignee`, and `get_resource_workload`. Validate input types and enforce tenant/project authorization within every tool execution.

**Why.** Results are deterministic and testable, sensitive data access is bounded, and the model cannot issue unrestricted SQL or write to Microsoft 365.

**Tradeoff.** New question types may require implementing additional tools. Project-risk explanations should use explicit criteria or be labeled as AI interpretation.

## ADR-005 — Separate Microsoft source adapters

**Context.** “Microsoft Project” and “Planner” do not share one universal task API.

**Decision.** Integrate **Planner basic through Microsoft Graph**, **Project for the web / Planner Premium through Dataverse where available**, and **Excel / CSV via an import adapter**. Other Project product variants require a specifically supported connector or export.

**Why.** Each adapter owns source authentication, pagination, API fields, rate-limit handling and mapping into the canonical task schema. The app avoids coupling the unified model to any one external tool.

**Tradeoff.** Connector coverage depends on tenants, licenses, permissions and source-specific APIs. Dataverse access is more involved than a simple Graph call. Excel/CSV imports may be manual and cannot promise automatic cloud updates.

## ADR-006 — Prefer polling/manual refresh to universal webhooks

**Context.** Near-real-time updates are useful, but Graph change-notification support is resource-specific and cannot be assumed for all Planner, Project and Excel tasks.

**Decision.** Start with user-triggered sync and/or bounded scheduled polling. Store per-source sync timestamps, expose freshness in the UI, and use idempotent upserts. Add event-triggered refresh only when a particular API/resource supports it.

**Why.** It is transparent, debuggable, reliable for a hackathon-scale solution and avoids unsupported “real-time Graph webhooks for every source” claims.

**Tradeoff.** Changes are eventually consistent and polling consumes API quota. Respect throttling, backoff and job concurrency limits.

## ADR-007 — Normalize all sources to one schema

**Context.** A Gantt chart, calendar, task list, and AI assistant need consistent fields regardless of the originating system.

**Decision.** Store common fields (`source`, `external_id`, `project_id`, `title`, `start`, `due`, `status`, `progress`) in a unified model. Keep assignees and dependencies in relational join tables; retain source-only attributes in `JSONB`.

**Why.** One task service can implement filters and project reporting once. Source adapters remain replaceable.

**Tradeoff.** Source features do not map perfectly. Preserve provenance and represent unavailable fields as missing rather than inventing defaults that appear authoritative. Prevent duplicate ingestion using `(tenant_id, source, external_id)`.

## ADR-008 — Use React, TypeScript and Tailwind for the frontend

**Context.** The interface needs responsive, interactive timeline/calendar views, fast filtering and a conversational panel.

**Decision.** Use React with TypeScript for typed components and Tailwind CSS for consistent styling. Fetch backend-authorized data through REST.

**Why.** Component reuse across the task list, Gantt, calendar and chat results reduces implementation effort. The frontend is independent of the Microsoft API and storage choices.

**Tradeoff.** Timeline and calendar libraries can impose data-shape constraints; adapters should transform canonical tasks at the UI boundary, not force source schemas through the application.

## ADR-009 — Export filtered results as CSV

**Context.** Teams need to share concise status snapshots with stakeholders using different tools.

**Decision.** Generate CSV exports from the **same authorized task query** used by the dashboard, applying the user's filters on the backend.

**Why.** CSV is easy to implement and compatible with Excel, while server-side generation keeps permissions consistent.

**Tradeoff.** CSV does not preserve chart formatting or dependency visualization. Escape formula-like values to reduce spreadsheet formula-injection risk.

## Non-goals for the reference design

- Autonomous task modification or writing back to Microsoft Planner/Project.
- Universal instantaneous synchronization across Microsoft 365 products.
- Complex multi-agent orchestration or vector search for structured task fields.
- Reproducing proprietary source code, configuration, tenant data or screenshots.

## Future options (not prerequisites)

Add durable background job processing, selective supported webhook/event triggers, server-sent events for live UI feedback, richer resource utilization metrics, and document retrieval only if users need to search unstructured project files.

See [architecture.md](architecture.md) for the corresponding component and data-flow diagrams.
