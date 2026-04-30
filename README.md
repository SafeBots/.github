# Safebots Platform

**Safebots** is a sovereign AI infrastructure stack — sealed computation, governed autonomy, community-owned intelligence. Three interlocking plugins, each handling a distinct layer of the architecture.

---

## Components

### [Safebox.md](./Safebox.md) — Governed AI Workflow Infrastructure

The substrate. Safebox handles stream materialization, sandboxed tool execution, capability dispatch, cryptographic governance, and the Safebux economic layer. Every write to the system flows through `Action.propose` and is gated by community policy; every external data fetch is cached, attributed, and priced. Workflows are trees of audited tools; capabilities are the bridge to the outside world.

Key concepts: workflows · workloads · steps · tasks · capabilities · the sandbox API · `Streams.fetch` · `Action.propose` · `Safebox.yield` · Protocol.* adapters (LLM, SMTP, Telegram, Web3, Payment, HTTP, and more) · M-of-N governance with judgments · Safebux economics · context assembly and enrichment.

---

### [Safebots.md](./Safebots.md) — Behavior Layer for Communities and Users

The application layer. Safebots lets any user — community or individual — declare automated behaviors on their own streams, without provisioning a separate bot identity. A community enabling inline suggestions in its chat *is* the bot. The behaviors are streams; the tools they reference are audited Safebox tools; the governance is the community's own Safebox policy machinery.

Key concepts: publisher-augments model · `Safebots/bot` stream configuration · the universal dispatch hook · the `Safebots/suggest` inline-completion endpoint · goal streams and prompt interpolation · dialog sessions · storefront greeting · chat orchestration (transcript replay, edit messages, working context assembly) · artifact lifetime and voting · the contract-enforcement judgment · KV-cache discipline.

---

### [Grokers.md](./Grokers.md) — Observation and Knowledge Indexing

The knowledge layer. Grokers observe streams at ingest and in the background, building a rich graph of typed relations — summaries, entity extractions, semantic links — that tools and the context assembler can traverse without re-running expensive inference on every turn. The Safebox context enrichment pipeline (`ner → resolve → expand → retrieve → rerank → clarify → suggest`) is where Grokers' outputs pay off.

Key concepts: observation streams · focus vocabulary · relation-based retrieval · eager summarization · the `Safebox/Context/enrich/*` hook chain.

---

### [Infrastructure.md](./Infrastructure.md) — Docker Execution Layer

The host layer. Infrastructure implements the `system-protocol-api` service that sits between Safebox's `Protocol.System` and the Docker daemon. All M-of-N governance happens in Safebox; this layer provides secure execution with defense-in-depth: peer UID verification via `SO_PEERCRED`, HMAC request/response signing, an operator-controlled container allowlist, per-action permissions, exponential backoff to prevent churn, and JTI replay protection. The service runs as a hardened systemd unit and is never reachable except from the Safebox process itself.

Key concepts: Unix domain socket transport · `SO_PEERCRED` peer UID verification · `managed-containers.json` allowlist · per-action allowlisting (`exec`, `pull` opt-in) · exponential backoff with churn detection · JTI replay protection · SIGHUP config reload · systemd hardening · structured JSON logging · supply chain integrity via `system-registry.json`.

---

## Architecture in Brief

```
  Communities & Users
         │
         ▼
  Safebots — behaviors declared on streams
         │  publisher-augments dispatch
         │  chat orchestration, goals, suggestions
         │
         ▼
  Safebox — governed infrastructure
         │  workflows, tools, capabilities
         │  Action.propose → M-of-N policy
         │  Streams.fetch → lazy materialization
         │  Protocol.* → external world
         │  Protocol.System → container governance
         │
         ▼
  Infrastructure — secure host execution layer
         │  system-protocol-api (separate process)
         │  SO_PEERCRED · HMAC · allowlist · backoff
         │  Docker API via /var/run/docker.sock
         │
         ▼
  Grokers — knowledge indexing
         │  observation at ingest
         │  relation graph over substrate
         │
         ▼
  Qbix Streams — the substrate
         streams, relations, messages,
         access, notifications, federation
```

Everything is a stream. Tools read and propose; they never write directly. The community is its own bot. Code is sha256-attested at install; governance happens before runtime, not at it.

---

## Design Principles

**Sovereignty is structural.** No layer stores plaintext on behalf of users, holds keys for them, or requires trusting a centralized party. The architecture makes surveillance expensive to add, not cheap to remove.

**Composition, not inheritance.** Bots are not special users. Behaviors are declared on streams. Governance is the standard policy machinery, not a bot-specific bypass. The substrate handles access control, collaboration, audit, and federation — Safebots and Grokers configure it, not reinvent it.

**Rails before runtime.** Every tool, capability, and workflow is audited before execution. A prompt injection in user input cannot induce actions outside the handler's declared contract. A compromised LLM cannot propose unaudited writes.

**Everything is replayable.** Given a stream, the system prompt, and the tool definitions, any turn's working context is reconstructable. No hidden state, no shadow logs, no parallel context store.

**Costs fall as usage grows.** Cache hits cost less. Early materializers earn commissions. The economics reward contributing to the commons.

---

## Status

Active development. See each component's reference document for implementation status and what's pending in the current round.

- Safebox: production-ready core; governance, Protocol adapters, context assembly, and local model runners in current sprint.
- Safebots: developer-preview readiness; chat orchestration, storefront greeting, and contract-enforcement judgment approaching smoke-test.
- Infrastructure: production-ready; systemd service, HMAC mutual auth, allowlist, backoff, and JTI replay protection all implemented.
- Grokers: integration in progress against the Safebox context enrichment pipeline.

---

*Qbix · Intercoin · Safebox · Safebots · Grokers*
