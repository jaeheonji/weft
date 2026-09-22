# Weft Product Architecture Design

**Date:** 2026-09-22
**Status:** Approved design
**Audience:** Weft maintainers and contributors

## 1. Purpose

Weft is a terminal-based Kubernetes operations workbench for platform and SRE engineers. It is optimized for making a change and observing its effects across related resources, rather than browsing one Kubernetes kind at a time.

Weft differs from a standalone Kubernetes TUI in three ways:

1. A stateful server watches clusters, resolves resource relationships, owns workspaces, and executes operations. Multiple lightweight terminal clients connect to that server.
2. A workspace can contain several independent views. Terminal multiplexers such as tmux, zellij, and Herdr can place one client per pane, while a standalone client switches quickly among the same server-side views.
3. Resources can be explored as logical services. A service may span Ingresses, Services, workloads, Pods, autoscalers, declared observability references, and multiple clusters.

The first release targets individual platform or SRE engineers operating several clusters. It starts as a local daemon but uses a protocol and security boundary that also support a centrally deployed server. Cloud resources and production-grade multi-user operation are later extensions.

## 2. Product Principles

- **Change and observe in one workspace.** A user can act on a resource and immediately follow rollout state, events, logs, and related resources.
- **Composition over terminal layout ownership.** A client renders one view. A terminal multiplexer owns panes, resizing, and process layout.
- **The standalone experience remains useful.** Without a multiplexer, one client switches among the workspace's server-side views. It does not implement an internal split-pane system.
- **Service context complements resource truth.** Weft retains the exact Kubernetes identity of every object while also presenting logical groups and relationships.
- **Explicit definitions are first-class.** Automatic discovery and Git-managed declarations are equal graph inputs. Every relationship exposes its provenance.
- **Writes are fast but bounded.** The server applies risk-aware policy and never lets a client bypass required review.
- **AI remains external.** Weft produces structured context and exposes capabilities to external agents. It does not embed an AI decision-maker or assume a particular agent approval model.
- **Credentials stay outside Weft persistence.** Weft stores credential references, never credential material.

## 3. Terminology

- **Target:** One configured Kubernetes cluster connection.
- **Target Set:** The clusters available to a workspace. A view may select one or several members of the set.
- **Workspace:** A durable server-side operating context containing a Target Set, views, Link Groups, and pinned state.
- **View:** A query, presentation, and context binding rendered by one client at a time.
- **Link Group:** A named synchronization channel that shares selection, target and namespace scope, filters, and time range among linked views.
- **Resource Graph:** The normalized resources and typed relationships known to the server.
- **Resource Group:** A logical service or other named grouping resolved from discovered and declared graph inputs.
- **Context Bundle:** A bounded, redacted, reproducible snapshot of selected workspace context for JSON export or an external agent.
- **Change Set:** A reviewed operation with an immutable resolved target list, preflight results, and per-target execution status.

## 4. System Architecture

### 4.1 Processes and transport

Weft is implemented in Go as separate server and client processes that communicate through versioned gRPC services and Protobuf messages. Server-streaming RPCs carry view projections and operation progress; unary RPCs handle bounded commands and metadata operations.

- Local clients connect through a Unix domain socket.
- Remote clients connect through TLS. The v1 remote mode uses mutual TLS inside a single operator trust boundary; multi-tenant identity and organization-level RBAC are not v1 features.
- The protocol handshake exchanges protocol versions, capabilities, and caller identity before a client attaches to a workspace and view.
- A protocol major-version mismatch rejects the connection. A capability mismatch disables only the unsupported feature.

Local and central deployment modes use the same application protocol and server implementation. They differ only in listeners, authentication, credential sources, and operational configuration.

### 4.2 Server components

The server is the source of truth for active operating state:

- **Session Gateway:** Authenticates connections, negotiates capabilities, attaches clients, and supports reconnection.
- **Cluster Manager:** Resolves credential references, performs Kubernetes discovery, and owns list/watch clients for every target.
- **Resource Store:** Holds normalized object snapshots and indexes needed by queries. It is an observed-state cache, not a second Kubernetes database.
- **Graph Engine:** Resolves discovered and declared nodes, memberships, and typed edges.
- **Workspace Manager:** Owns workspaces, views, Target Sets, Link Groups, and pinned context.
- **Query and Projection Engine:** Evaluates view queries and streams only the snapshot and deltas needed by each attached client.
- **Action Engine:** Resolves targets, evaluates policy, performs preflight checks, executes Kubernetes operations, and reports progress.
- **Context Gateway:** Creates bounded context bundles used by JSON export and the MCP adapter.
- **Persistence:** Stores durable metadata in SQLite and records operations in an append-only audit journal.

Clients handle terminal input, local interaction state such as an open palette, and rendering. They do not connect directly to Kubernetes or duplicate the entire server resource cache.

### 4.3 Credential boundary

SQLite contains cluster IDs, display names, API endpoints, and logical credential references. It must not contain kubeconfig documents, bearer tokens, client private keys, cloud access keys, or temporary tokens returned by credential plugins.

A local server resolves references using the user's kubeconfig and Kubernetes `exec` credential flow. A central server receives credentials through Kubernetes ServiceAccounts, read-only mounted files, or an external secret provider. Credential material may exist transiently in server memory as required by the Kubernetes client, but Weft does not copy it into its database, journal, workspace state, or context bundles.

An encrypted built-in credential vault is explicitly outside v1.

## 5. Resource Identity and Graph

### 5.1 Resource identity

A stable resource address consists of:

```text
provider / target / API group / kind / namespace / name
```

The current Kubernetes UID and resource version describe the observed incarnation and revision; they are not the stable address. This distinction lets Weft recognize that a named object was recreated while preventing a stale observation from being treated as the current instance.

The provider field is present in v1 even though Kubernetes is the only implemented provider. This avoids baking Kubernetes identity into workspace, graph, and context contracts without prematurely creating a public plugin SDK.

### 5.2 Discovered relationships

The graph engine derives structural edges from reliable Kubernetes evidence, including:

- owner references;
- workload and Service selectors;
- EndpointSlices;
- Ingress backends;
- workload-to-Pod ownership;
- HPA scale targets; and
- volume claims and mounts where the relationship is unambiguous.

Name similarity is not enabled as a default source in v1. It is too easy to mistake convention for topology.

Every edge records its type, source, observation time, and relevant evidence. The TUI and exported context can therefore explain why two resources are connected.

### 5.3 Declared groups and relationships

Team-owned definitions live in Git as `.weft/*.yaml` manifests. A `ResourceGroup` can declare:

- stable group identity and display metadata;
- applicable targets;
- direct resource references;
- label- and namespace-based membership selectors;
- additional typed relationships, including external references; and
- explicit exclusions.

Resolution precedence is deterministic:

1. An explicit exclusion removes a matching member or edge.
2. Declared membership and relationships add or correct topology.
3. Structural discovery fills remaining relationships.

Server overlays support personal aliases, temporary investigation groups, and additive local relationships. They cannot silently replace a Git-managed group with the same identity. An identity conflict is surfaced as an error unless the user creates a separately named overlay group.

One logical group can apply to several targets. Its resources form separate per-target instance graphs so users can compare environments without losing the exact target of an operation.

## 6. Workspace and View UX

### 6.1 Entry model

The default home is Workspace-first. It lists recent workspaces and templates such as a service-change or incident workspace. A new workspace commonly opens with a logical service list, while a command palette remains available from every view for direct navigation.

A workspace persists:

- its Target Set;
- view specifications and attachment identities;
- Link Group membership;
- linked or pinned state; and
- the last meaningful selection and filters.

### 6.2 Client and multiplexer behavior

Each multiplexer pane runs a client attached to one server-side view. Layout adapters generate pane-launch commands but do not take ownership of multiplexer sessions, resize behavior, or pane lifetimes.

Without a multiplexer, one client exposes a fast view switcher. Inactive views remain server-side and retain their query and context. Standalone mode sacrifices simultaneous rendering, not functionality or workspace compatibility.

### 6.3 Link Groups

Views in a Link Group share:

- selected resource or logical group;
- target and namespace scope;
- search and label filters; and
- log, event, or metric time range when applicable.

A linked view follows updates from its Link Group. A pinned view retains its current context until it is explicitly relinked. Multiple Link Groups can coexist in one workspace, allowing a rollout investigation to change while a cluster-health view remains fixed.

### 6.4 General view model

A view is not tied to a Kubernetes kind:

```text
View = Query × Presentation × Context Binding
```

- A query selects targets, arbitrary GVKs, labels, Resource Groups, or graph traversals.
- A presentation renders a table, tree, detail, YAML document, stream, diff, timeline, graph, or operation state.
- A context binding links the view to a Link Group or pins it to explicit context.

Kubernetes discovery makes every GVK and CRD available through generic resource renderers. A declarative `ViewSpec` can define queries, columns, sorting, presentation, and Link Group binding in Git or a server overlay.

The initial built-in presentations include:

- Workspace Home;
- generic Resource Table, Tree, Detail, and YAML;
- Resource Graph;
- Logs and Events;
- Diff and rollout/timeline views;
- Change Set, operation progress, and audit history; and
- Context Bundle preview.

This list is a minimum distribution, not a closed set. Internally, presentation implementations use a registry boundary. A public arbitrary-code view plugin SDK is deferred until a second real provider or extension supplies requirements for that API.

### 6.5 Consistent screen grammar

Every operational view uses the same basic chrome:

- The header shows view name, target scope, logical group, and linked or pinned state.
- A multi-target resource list always exposes target identity; it cannot be hidden into ambiguity.
- The footer shows immediately relevant actions, Link Group identity, and background operation status.
- The command palette searches resources, groups, views, workspaces, and actions.
- A dangerous action selected from the palette opens review; it never executes from search alone.

Opening a contextual view from a selected resource links it to the current Link Group by default. The user can immediately pin it or move it to another group.

## 7. Operations and Safety

Weft directly supports Kubernetes changes, including edit/apply/delete and specialized scale and restart actions. All callers, including MCP tools, use the same Action Engine.

The operation flow is:

1. Accept an intent expressed against current selection or an explicit query.
2. Resolve it immediately before review into an immutable list of exact ResourceRefs.
3. Re-read relevant resource versions and perform authorization and policy preflight.
4. Assign the server-enforced mode: `direct`, `confirm`, or `change-set`.
5. Display the actual targets, proposed mutation, diff where available, and policy result.
6. Execute and expose per-target progress as an operation stream.
7. Observe resulting resource changes in linked views and append the outcome to the audit journal.

Policies may depend on action type, environment, target count, caller identity, and resource scope. Production, multi-target, bulk, delete, and generic apply operations can have a non-bypassable minimum review mode. Clients may request more review but not less than server policy permits.

Kubernetes does not provide a cross-cluster transaction, so Weft never describes a multi-target operation as atomic. Partial success is preserved and reported per target. Successful operations are not automatically undone after a later failure. Only actions with a defined, reviewable compensating operation offer rollback.

If the audit journal cannot durably accept the planned operation record, the Action Engine fails closed before mutating Kubernetes.

## 8. State Synchronization and Failure Semantics

When a client attaches to a view, the server returns a snapshot and monotonically ordered projection revision. Subsequent changes arrive as deltas.

- Intermediate changes may be coalesced for a slow client when doing so preserves the latest state.
- Per-view queues are bounded and cannot block cluster watches or other projections.
- A queue overflow, missed revision, or unavailable retained history triggers a fresh snapshot.
- Reconnection resumes from a retained revision when possible and otherwise performs a full resynchronization.

Failures remain visible:

- A disconnected cluster retains its last snapshot marked `stale`, with the last successful observation time.
- An object hidden by insufficient RBAC is represented as inaccessible when its absence is knowable; Weft does not present incomplete visibility as health.
- A stale snapshot cannot authorize a write. The Action Engine revalidates target identity, resource version, and policy.
- Multi-target operations retain every success and failure independently.

## 9. External Agent Boundary

Weft does not contain an AI assistant. The Context Gateway creates a common context model consumed by:

- JSON export from the client or CLI; and
- an MCP adapter exposing resources and tools to an external agent.

The v1 local MCP adapter uses stdio transport and connects to the same server application API as the TUI. Remote agent transport and product-specific integrations are later concerns.

A Context Bundle contains:

- its creation time and source revisions;
- the workspace, Link Group, selection, and query that produced it;
- a bounded slice of resources and graph relationships;
- selected events, logs, diffs, or operation history when requested;
- provenance for relationships and excerpts; and
- explicit truncation and redaction metadata.

Bundle creation enforces configurable size and time-range budgets. Kubernetes Secret payloads, credential fields, and configured sensitive keys are redacted before serialization. Redaction applies equally to JSON and MCP output.

MCP write tools do not bypass Weft policy. They carry the caller identity into the Action Engine. Whether an external agent asks its user for approval is outside Weft, but the server still applies its configured `direct`, `confirm`, or `change-set` floor.

## 10. Persistence

SQLite stores:

- target connection metadata and credential references;
- workspaces, views, Link Groups, and overlays;
- policy metadata; and
- append-only audit and operation records.

Observed Kubernetes objects remain in the in-memory Resource Store and are rebuilt through list/watch after restart. Weft does not persist a shadow copy of every cluster object in v1.

Git-managed ResourceGroup and ViewSpec manifests remain the portable source of team configuration. SQLite records loaded source identity and validation status, not an opaque replacement for the files.

## 11. V1 Scope and Non-goals

V1 includes:

- a local-first server with Unix socket and mTLS remote connectivity;
- multi-cluster Target Sets;
- arbitrary Kubernetes GVK and CRD exploration;
- generic and specialized built-in presentations;
- declarative ResourceGroup and ViewSpec manifests;
- Resource Graph discovery and provenance;
- Workspace, Link Group, pinning, and standalone view switching;
- direct Kubernetes operations with risk-aware review;
- JSON Context Bundles and a local MCP adapter;
- tmux and zellij layout adapters plus a generic attach-command format usable by other multiplexers; and
- SQLite metadata with an append-only audit journal.

V1 does not include:

- AWS, GCP, or Azure resource providers;
- an arbitrary-code public plugin SDK;
- embedded AI or a product-specific agent integration;
- a credential vault;
- high availability, organization-level multi-tenancy, or collaborative editing;
- a fully event-sourced architecture;
- a client-owned split-pane layout engine; or
- dedicated Prometheus, Loki, or commercial observability query implementations.

External observability resources can still be declared as graph references and included as links or structured context.

## 12. Delivery Milestones

The implementation is divided into independently demonstrable milestones:

1. **Server-client spine:** Protocol, handshake, connection and reconnection, workspace/view attachment, and a minimal rendered projection.
2. **Resource core:** Multi-cluster discovery and watches, Resource Store, graph resolution, ResourceGroup manifests, and generic resource queries.
3. **UX core:** ViewSpec, built-in resource presentations, Link Groups, pinning, standalone switching, and multiplexer adapters.
4. **Operations:** Action policy, immutable target resolution, preflight, Change Sets, operation streaming, and audit journal.
5. **Agent boundary and hardening:** Context Bundle, redaction, JSON export, MCP adapter, fault injection, security checks, and performance validation.

These milestones are sequencing boundaries, not separately deployed services. The first implementation plan should preserve these boundaries and define a runnable acceptance demonstration for each.

## 13. Verification Strategy

### 13.1 Automated tests

- Unit tests cover resource identity, graph precedence, group resolution, queries, Link Group transitions, policy evaluation, context budgets, and redaction.
- Contract tests cover Protobuf compatibility, capability negotiation, snapshot/delta ordering, reconnection, and resynchronization.
- Integration tests cover multiple Kubernetes API servers, dynamic and CRD resources, RBAC restrictions, watch restarts, stale writes, and partial operation failure.
- End-to-end tests use at least two disposable Kubernetes clusters to exercise multi-target discovery, views, and mutations.
- TUI tests use fixed terminal sizes and golden output for rendering, navigation, and action-review flows.
- Fault tests cover slow clients, cluster disconnection, queue overflow, journal failure, and mixed success across targets.
- Security tests assert that credential material and Kubernetes Secret payloads do not enter SQLite, audit records, logs, or Context Bundles.

### 13.2 Initial performance envelope

The v1 reference workload is:

- five clusters;
- 50,000 total observed resources;
- eight concurrent clients in one workspace;
- first usable view within three seconds when the server cache is warm;
- p95 observed-change-to-view latency below one second under steady state; and
- isolation such that a slow client does not measurably delay another projection or a cluster watch.

Benchmarks report results against this envelope. They are not permission to drop correctness, provenance, redaction, or action safety when the envelope is exceeded; overload must degrade explicitly through coalescing, resynchronization, and visible stale state.

## 14. Success Criteria

The design succeeds when a platform engineer can:

1. Connect several clusters without distributing Kubernetes credentials to terminal clients.
2. Open or resume a workspace and see different linked views in several multiplexer panes or switch among the same views in a plain terminal.
3. Find any built-in resource or CRD and inspect its exact target and Kubernetes identity.
4. Treat related resources as a logical service while inspecting the evidence behind every inferred relationship.
5. Perform a reviewed change across one or more targets and follow its per-target effects without losing context.
6. Export the same bounded, redacted context through JSON or MCP for use by an external agent.
7. Understand partial visibility, stale state, and partial operation failure without Weft presenting them as success.
