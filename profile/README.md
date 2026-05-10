# Safebots Platform

**Safebots** is a sovereign AI infrastructure stack — sealed computation, governed autonomy, community-owned intelligence. Five interlocking plugins built on the Qbix Streams substrate, each handling a distinct layer of the architecture, none of them reinventing what the layer below already provides.

-----

## How this improves on the current state of the art

**Verifiable safety, not promised safety.** Every tool, capability, and workflow is sha256-attested at install. The model that proposes a write is structurally separated from the authority that approves it via M-of-N OpenClaim verification. A jailbroken or unaligned model still cannot write anything the quorum has not signed. Compromised hosts cannot forge responses because the layer above verifies HMAC signatures on the way back. Each defensive layer fails closed without unlocking the next, instead of one trusted boundary protecting everything.

**A real graph database underneath, not a vector store retrofitted into one.** Streams is a property graph stored as ordinary MySQL tables, with bidirectional indexed typed edges, attribute-derived relations via `syncRelations`, and O(log N) traversal in either direction. Faceted search and neighborhood walks are native operations, not recursive query workarounds. The fork and workspace layer adds copy-on-write semantics — `Streams::fork()` snapshots a relation neighborhood at a point in time, the workspace cascade resolves reads transparently, tombstones suppress deletes without rewriting history. Forking a workflow forks the ZFS state, the MySQL state, and the relation graph in lockstep. No existing system does this as one operation.

**Per-stream access control, not per-tenant.** Every node in the graph carries its own participant list with read/write/admin levels, four-tier inheritance (public, contact-label, participant role, direct grant), and append-only audit history. Sharing a single artifact, a single conversation, a single agent’s output is a routine operation, not a workaround over coarser permissions.

**Policy as substrate, not bolt-on.** Writes go through `Action.propose` and are gated by community policy machinery before execution — the same machinery that handles everything else, not a parallel review system grafted onto an agent framework. Sandbox boundaries are typed: tools can read streams, propose writes, materialize external streams via Protocol adapters, and yield artifacts; they cannot write directly, escape governance, or bypass quotas.

**Inference economics that improve with scale, not degrade.** Stable-prefix-first prompt assembly turns the KV cache into a shared resource — system prompt as cache checkpoint 1, summaries 2-4, scratch and tail volatile. Combined with PLT-based sequential KV compression (which proves the per-token surprisal bound on per-vector entropy is hundreds of times below the per-vector Shannon limit that TurboQuant operates against, with the gap widening as context grows), cache reuse extends across sessions, not just within one. `Streams.fetch` charges Safebux on materialization with cache hits priced cheaper and 50% routed to the original payer. Every certified expert call becomes a content-addressed artifact in the economy.

**Replayability as a guarantee.** Given a stream, a system prompt, and the tool definitions, any turn’s working context is reconstructable. No hidden state, no shadow logs, no parallel context store. The deterministic execution hash is structural, not aspirational.

**Sovereignty as a property of the architecture, not a marketing claim.** No layer stores plaintext on behalf of users, holds keys for them, or requires trusting a centralized party. The architecture makes surveillance expensive to add, not cheap to remove.

-----

## The substrate: Qbix Streams

Underneath every component is a directed property graph stored as ordinary MySQL tables. Each node is a typed stream with attributes, a participant list with read/write/admin levels, and an append-only message timeline. Each edge is a row in `streams_related_to` and `streams_related_from`, indexed in both directions. `syncRelations` projects designated attributes into typed relation rows automatically.

Three properties matter for everything above. Relations are first-class typed edges, so faceted search and neighborhood traversal are native. The workspace and fork layer adds copy-on-write semantics — fork-on-mutate, transparent cascade resolution, tombstone-based deletion, no merge primitive needed because new baselines are just new clones. And every node has its own access control, audit trail, and federation surface, so the layers above don’t reinvent any of it.

-----

## Components

### [Safebox.md](./Safebox.md) — Governed AI Workflow Infrastructure

The workflow and economics layer. Safebox defines `Safebox/workflow`, `Safebox/step`, `Safebox/workload`, `Safebox/task`, `Safebox/tool`, `Safebox/capability`, `Safebox/action`, `Safebox/provider`, and `Safebox/protocol` as stream types. Workflows are trees of Steps that produce Workloads, themselves trees of Tasks. Tools read streams and propose writes via `Action.propose` only — never write directly. Capabilities materialize external data by mapping `publisherId` plus stream-name prefix: a tweet becomes `com.x` + `Websites/post/{tweet_id}`, an OpenAI completion becomes `com.openai` + `Streams/text/{prompt_hash}`. Capability dispatch is provider plus type-prefix matching.

Each run produces a deterministic execution hash. `Streams.fetch()` lazily materializes external data and charges Safebux on the way through — content-addressed artifact economy with proper attribution, expressed as ordinary stream operations.

Key concepts: workflows · workloads · steps · tasks · capabilities · the sandbox API · `Streams.fetch` · `Action.propose` · `Safebox.yield` · Protocol.* adapters (LLM, SMTP, Telegram, Web3, Payment, HTTP) · M-of-N governance with judgments · Safebux economics · context assembly and enrichment.

-----

### [Safebots.md](./Safebots.md) — Behavior Layer for Communities and Users

The agent layer. A community enabling inline suggestions in its own chat *is* the bot — there is no separate bot user, no parallel identity. Behaviors are declared on streams via `Safebots/bot` configuration; the universal dispatch hook fires on relevant events; goal streams hold system prompts, patterns (`fields`, `process`, `artifact`), and field specifications; governance is whatever Safebox policy the community already runs.

The turn pipeline is structured for KV-cache reuse from the ground up. Every section of the prompt is wrapped in XML trust labels so the model can distinguish authoritative from user input. Writes go through `Action.propose` exactly like every other Safebox tool, which means the agent inherits all governance for free — and a prompt injection cannot induce actions outside the handler’s declared contract because the action machinery doesn’t trust the model.

Key concepts: publisher-augments model · `Safebots/bot` stream configuration · the universal dispatch hook · the `Safebots/suggest` inline-completion endpoint · goal streams and prompt interpolation · dialog sessions · storefront greeting · chat orchestration · artifact lifetime and voting · the contract-enforcement judgment · KV-cache discipline.

-----

### [Grokers.md](./Grokers.md) — Observation and Knowledge Indexing

The knowledge layer. Grokers sits alongside Safebox on the Streams substrate, indexing two distinct kinds of content with the same machinery: stream content (for context enrichment) and source code (for codebase comprehension).

For stream content, Grokers observes posts at ingest and in the background, building a graph of summaries, entity extractions, and semantic links that the Safebox context enrichment pipeline (`ner → resolve → expand → retrieve → rerank → clarify → suggest`) traverses without re-running expensive inference on every turn. For source code, Grokers parses files with tree-sitter across ten languages, builds a typed extern graph (a `Grokers/extern` stream is the slot connecting any caller to any callee — a Q::Event hook, a config key, a DB table, a template name, a message type, an HTTP endpoint, an npm module), and runs LLM agents bottom-up through the call graph in topological order, leaves first. A symbol is never analyzed before its callees, enforced by the scheduler. When a callee’s contract changes, callers via `Grokers/invokes` re-queue automatically.

Bottom-up semantic comprehension persisted as a queryable graph that survives across sessions, rather than top-down RAG over file contents reconstructed each turn.

Key concepts: observation streams · focus vocabulary · the typed extern graph · bottom-up topological scheduling · relation-based retrieval · eager summarization · the `Safebox/Context/enrich/*` hook chain · update cascades through `Grokers/invokes`.

-----

### [Code.md](./Code.md) — Conversational Coding via Workspaces

The development surface. The architectural realization behind Code is that coding is workspaces too — the substrate that handles user data handles code identically, with the same forks, the same governance, the same audit. ZFS provides the filesystem-level fork; git push provides one-way projection to the human-facing review surface; Grokers provides the queryable model of what’s already there.

Code defines three workflows on top of Safebox. `code/cloneRepo` produces a new `Grokers/repo` stream grounded in a specific upstream commit (fresh git clone into its own ZFS dataset, then full grok pass). `code/modify` runs a sprint inside a workspace fork of a repo (filesystem-level ZFS clone, parallel-to-existing-team execution, no contention). `code/regrok` refreshes comprehension when the underlying source changes. The cascade of workspaces under each `Grokers/repo` is forward-only — no merge primitive, no pull, no resync. New baselines are new clones. Multiple teams run in parallel against the same upstream because each team’s safebox is independent.

A developer asking “add a token refresh method to JWTManager” doesn’t get RAG over file contents. The agent queries the extern graph for what already exists (signature, callers, config keys, error text keys, related endpoints), proposes patches as Action proposals through Safebox, runs tests in the sandbox, and pushes a branch only on success. GitHub Actions and CI become composable substrate operations rather than separate infrastructure.

Key concepts: `code/cloneRepo`, `code/modify`, `code/regrok` workflows · ZFS-backed filesystem forks · push-only git as PR projection · forward-only cascade (no merge primitive) · parallel teams sharing upstream · structured plan composition from `conventions.json` · extern-graph queries instead of similarity search.

-----

### [Infrastructure.md](./Infrastructure.md) — Docker Execution Layer

The host layer. Infrastructure implements `system-protocol-api`, a Node.js service sitting between Safebox’s `Protocol.System` and the Docker daemon. M-of-N governance happens in Safebox before any request reaches Infrastructure; this layer provides defense in depth.

Layer 1 (M-of-N OpenClaim verification with verifiedOpToken) lives in the Safebox plugin. Layers 2 through 9 — peer UID verification via `SO_PEERCRED`, HMAC-signed requests and responses with a shared key at `/etc/safebox/system-api.key`, container allowlist, per-action permissions, image-pattern enforcement, exponential backoff with churn detection, JTI replay protection, systemd hardening — live here. Reachable only from the qbix process (UID 33) via Unix domain socket, never from the network. Compromised Node cannot bypass governance because it cannot forge a signed verifiedOpToken; compromised API cannot forge responses because Safebox verifies the HMAC on the way back.

Key concepts: Unix domain socket transport · `SO_PEERCRED` peer UID verification · `managed-containers.json` allowlist · per-action allowlisting (`exec`, `pull` opt-in) · exponential backoff with churn detection · JTI replay protection · SIGHUP config reload · systemd hardening · structured JSON logging · supply chain integrity via `system-registry.json`.

-----

## Architecture in Brief

```
  Communities & Users
         │
         ▼
  Safebots — behaviors declared on streams         Code — conversational
         │  publisher-augments dispatch                  development
         │  chat orchestration, goals, suggestions       │
         │  KV-cache-disciplined prompt assembly         │  cloneRepo · modify · regrok
         │                                               │  ZFS forks · push-only git
         ▼                                               ▼
  Safebox — governed infrastructure         ◄────┐
         │  workflows, tools, capabilities       │
         │  Action.propose → M-of-N policy       │  Grokers — knowledge indexing
         │  Streams.fetch → lazy materialization │  observation graph (streams)
         │  Protocol.* → external world          │  extern graph (code)
         │  Protocol.System → container gov.     │  bottom-up comprehension
         ▼                                       │
  Infrastructure — secure host execution         │
         │  system-protocol-api (separate proc)  │
         │  SO_PEERCRED · HMAC · allowlist       │
         │  Docker API via /var/run/docker.sock  │
         ▼                                       ▼
  ────────────────  Qbix Streams (the substrate)  ─────────────────
            typed nodes · bidirectional indexed relations ·
            messages · access control · notifications ·
            workspaces · copy-on-write forks · federation
```

Grokers and Code are drawn alongside Safebox rather than below Infrastructure because they don’t depend on container execution. Grokers depends only on `{Q, Streams, Users}`; Code depends on Grokers plus the Safebox workflow runtime, with ZFS as the materialization layer for filesystem forks.

-----

## Design Principles

**Sovereignty is structural.** No layer stores plaintext on behalf of users, holds keys for them, or requires trusting a centralized party. The architecture makes surveillance expensive to add, not cheap to remove.

**Composition, not inheritance.** Bots are not special users. Behaviors are declared on streams. Governance is the standard policy machinery, not a bot-specific bypass. The substrate handles access control, collaboration, audit, and federation — Safebots, Grokers, Code, and Safebox configure it, not reinvent it.

**Authority separated from intelligence.** The model that proposes a write is fully decoupled from the authority that approves it. Run an unaligned open-source model as the proposer and warrant-governed execution still holds, because the proposer cannot write anything the M-of-N quorum has not signed. This is the property that makes “rails before runtime” survive contact with adversarial inputs and adversarial models alike.

**Rails before runtime.** Every tool, capability, and workflow is audited before execution. A prompt injection in user input cannot induce actions outside the handler’s declared contract. A compromised LLM cannot propose unaudited writes.

**Workspaces all the way down.** The substrate’s primitives — streams, relations, forks, workspaces, governance, audit — are domain-agnostic. They coordinate state, not a particular kind of state. User data, agent behaviors, observations, source code: all the same machinery. New domains add new stream types and Protocol adapters; they don’t invent new primitives.

**Everything is replayable.** Given a stream, the system prompt, and the tool definitions, any turn’s working context is reconstructable. No hidden state, no shadow logs, no parallel context store. The deterministic execution hash is a guarantee, not a marketing claim.

**Costs fall as usage grows.** Cache hits cost less. Early materializers earn commissions. The economics reward contributing to the commons rather than walling off contributions to it. Stable-prefix-first prompt assembly turns this from an aspiration into accounting: when summaries are stable across sessions, the KV cache makes the same prefix cheap for everyone who follows.

-----

## Status

Active development. See each component’s reference document for implementation status and what’s pending in the current round.

Safebox: production-ready core; governance, Protocol adapters, context assembly, and local model runners in current sprint. Safebots: developer-preview readiness; chat orchestration, storefront greeting, and contract-enforcement judgment approaching smoke-test. Infrastructure: production-ready; systemd service, HMAC mutual auth, allowlist, backoff, and JTI replay protection all implemented. Grokers: integration in progress against the Safebox context enrichment pipeline; symbol indexing and extern graph construction complete for PHP, JavaScript, TypeScript, Python, Java, C, C++, C#, Swift, Rust, and Go. Code: workflows specified (`cloneRepo`, `modify`, `regrok`), ZFS dataset provisioning integrated with Infrastructure, parallel-team semantics validated against the workspace cascade.

-----

*Qbix · Intercoin · Safebox · Safebots · Grokers · Code*
