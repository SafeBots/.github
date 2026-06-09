# Safebots

Open-source AI infrastructure with a different bet at the foundation: actions are proposed, not executed. Every side effect — sending an email, charging a card, deleting a row, calling a paid API — goes through M-of-N approval before it leaves the box. Meant to be run by both people and organizations, including regulated institutions, Safebots continuously compound and refine information in a [graph database](https://community.qbix.com/t/qbix-streams-as-a-graph-database/769) powered by [Qbix Platform](https://github.com/Qbix), which has attracted 7M users across 100+ countries since 2011. There are five main repositories:

* **Infrastructure** — an auditable, locked-down Linux environment
* **Safebox** — standardized substrate for AI orchestration
* **Safebots** — AI-powered collaboration for teams and organizations
* **Grokers** — intelligently ingests existing code bases
* **Code** — safely generates code and runs experiments

## Contents

- [🧱 Predictable enough to build on](#-predictable-enough-to-build-on)
- [🎯 What this is](#-what-this-is)
- [🧩 Components](#-components)
- [🔭 Field convergence](#-field-convergence)
- [🪨 Substrate primitives](#-substrate-primitives)
- [🔏 OpenClaiming Protocol](#-openclaiming-protocol)
- [🧭 Architecture](#-architecture)
- [📚 Papers](#-papers)
- [📝 Long-form essays](#-long-form-essays)
- [🚦 Status](#-status)
- [👋 Contact](#-contact)
- [🤝 Join the movement](#-join-the-movement)

---

> **Questions?**
>
> [![Meet the founders at the weekly AMA](https://img.shields.io/badge/%E2%86%92%20Meet%20the%20founders%20at%20the%20weekly%20AMA-4CAF50?style=for-the-badge&logoColor=white)](https://calendly.com/safebotsai/weekly)

---

<p align="center">
  <img src="https://safebots.ai/img/safebots/safebots.webp" alt="Safebots — sovereign AI infrastructure" width="70%">
</p>

---

## 🧱 Predictable enough to build on

Every infrastructure layer that eventually disappeared into the background got there the same way: it became *uneventful*. Predictable enough that the layer above could stop worrying about it and start building. Four cases, each showing what changed once the threshold was crossed.

| Layer | The question it removed | What got built on top |
|---|---|---|
| **Shipping container** (1956) | "Will this cargo survive the trip and arrive in a form the next vessel can load?" | Global supply chains. Logistics planners stopped designing around per-shipment variance and started designing around predictable boxes. |
| **Transistor** (1947→) | "Does this individual switch reliably flip when I apply voltage?" | The entire stack from microprocessors to phones. Circuit designers stopped verifying each switch and started composing them by the billion. |
| **IEEE 754** (1985) | "What did my compiler and FPU do to this floating-point number?" | Portable scientific software. Numerical programmers stopped checking the math and started trusting that addition meant the same thing on every machine. |
| **HTTPS padlock** (mid-1990s→) | "Is this TLS handshake correctly implemented end-to-end?" | Open-web commerce. A user could see the lock and enter a credit card without inspecting code. |

None of these layers won by being cleverer. They won by becoming **boring and predictable**. The artisan-to-industrial transition is the same shape every time: instances stop being unique, failure modes stop being surprising, the layer below stops being a source of arbitrariness. The layer above can compose without re-verification. *Trustworthy by construction rather than by inspection.*

AI agents are still at the artisanal stage. Every deployment is bespoke:

- **Approval shapes** vary by framework — LangChain has one, AutoGPT has another, Anthropic's harness a third, every enterprise integration its own.
- **Execution boundaries** vary by runtime — some sandboxes, some bare processes, some hardware-isolated, most "trust the process."
- **Audit formats** vary by vendor — log lines here, traces there, structured records elsewhere, often none.
- **Trust signals** vary by deployment — a system prompt here, a moderation API there, a human review queue somewhere else.

Every integration is a one-off that has to be re-verified each time. There is no equivalent of "see the padlock and proceed."

The substrate this repo builds is one attempt at the uneventful version:

- **One signed-claim envelope** (OpenClaiming Protocol) — every approval, authorization, attestation across every component verifies with the same code.
- **One execution-hash recipe** — server sandboxes and browser sandboxes produce identical hashes from identical inputs.
- **One approval primitive** (M-of-N OpenClaim) — humans, smart contracts, and hardware attesters all sign the same shape, and the substrate gates on the same threshold logic.
- **One unit of work** — every action is a structured proposal evaluated before any side effect lands.

The layer above can then stop asking *"did this action actually happen the way the model claims it did"* and start building on the assumption that it did.

---

## 🎯 What this is

Five components — Safebox, Safebots, Grokers, Code, Infrastructure — composed against one substrate (Qbix Streams). Each handles a distinct concern, none reinvents what the layer below provides.

| Component | Role | Status |
|---|---|---|
| **Safebox** | Workflow + governance + economics. Tools propose writes via `Action.propose`; substrate decides via M-of-N OpenClaim. Sandboxed execution with deterministic execution hash. | Production core; governance and Protocol adapters in current sprint |
| **Safebots** | Agent behavior layer. Behaviors declared on streams, not as separate bot users. Inline suggestions, dialog sessions, contract-enforcement judgments. | Developer preview |
| **Grokers** | Knowledge layer. Bottom-up code comprehension and stream observation, persisted as queryable typed graph. Cross-language extern graph for ten languages. | Integration in progress |
| **Code** | Conversational development. ZFS-backed workspace forks; push-only git as PR projection. Multiple teams parallel against one upstream. | Workflows specified; parallel-team semantics validated |
| **Infrastructure** | Secure host. Unix-domain-socket transport, `SO_PEERCRED`, HMAC mutual auth, container allowlist, systemd hardening, JTI replay protection. | Production-ready |

The architectural property that ties these together: **the model that proposes a write is decoupled from the authority that approves it.** Run an unaligned open-source model as the proposer and warrant-governed execution still holds, because the proposer cannot write anything the quorum has not signed. The decoupling is what the substrate is for; everything else is implementation of that property.

---

## 🧩 Components

<img src="https://safebots.ai/img/safebots/safebot1.webp" alt="A safebot" align="right" width="180">

Each of the five components is documented in its own reference file.

### [Safebox](./Safebox.md) — Governed AI Workflow Infrastructure

The workflow and economics layer. Defines `Safebox/workflow`, `Safebox/step`, `Safebox/workload`, `Safebox/task`, `Safebox/tool`, `Safebox/capability`, `Safebox/action`, `Safebox/provider`, `Safebox/protocol` as stream types. Workflows are trees of Steps that produce Workloads, themselves trees of Tasks. Tools read streams and propose writes via `Action.propose` only. They never write directly. Capabilities materialize external data via `publisherId` + stream-name prefix matching (`com.x` + `Websites/post/{tweet_id}`, `com.openai` + `Streams/text/{prompt_hash}`, etc).

Each run produces a deterministic execution hash. `Streams.fetch()` lazily materializes external data and charges Safebux on the way through, with cache hits priced cheaper and 50% routed to the original payer. Content-addressed artifact economy expressed as ordinary stream operations.

### [Safebots](./Safebots.md) — Behavior Layer for Communities and Users

The agent layer. A community enabling inline suggestions in its own chat *is* the bot. There is no separate bot user and no parallel identity. Behaviors are declared on streams via `Safebots/bot` configuration; the universal dispatch hook fires on relevant events; goal streams hold system prompts, patterns, and field specifications; governance is whatever Safebox policy the community already runs.

Writes go through `Action.propose` exactly like every other Safebox tool, which means the agent inherits all governance for free. A prompt injection in user input cannot induce actions outside the handler's declared contract because the action machinery does not trust the model.

### [Grokers](./Grokers.md) — Observation and Knowledge Indexing

The knowledge layer. Indexes two distinct kinds of content with the same machinery: stream content (for context enrichment) and source code (for codebase comprehension).

For source code: parses files with tree-sitter across ten languages (PHP, JavaScript, TypeScript, Python, Java, C, C++, C#, Swift, Rust, Go), builds a typed extern graph, runs LLM agents bottom-up through the call graph in topological order. A symbol is never analyzed before its callees, enforced by the scheduler. When a callee's contract changes, callers via `Grokers/invokes` re-queue automatically.

For stream content: observes posts at ingest and in background, building a graph of summaries, entity extractions, and semantic links the Safebox context-enrichment pipeline traverses without re-running inference on every turn.

Paper: [Grokers (arXiv:2606.00050)](https://arxiv.org/abs/2606.00050) — three formal theorems on byte-identity, accumulation monotonicity, and dual-traversal ordering.

### [Code](./Code.md) — Conversational Coding via Workspaces

The development surface. Three workflows on top of Safebox: `code/cloneRepo` produces a `Grokers/repo` stream grounded in an upstream commit; `code/modify` runs a sprint inside a workspace fork (filesystem-level ZFS clone, parallel-to-existing-team execution); `code/regrok` refreshes comprehension when upstream changes.

The cascade of workspaces under each `Grokers/repo` is forward-only — no merge primitive, no pull, no resync. New baselines are new clones. Multiple teams run in parallel against the same upstream because each team's safebox is independent.

### [Infrastructure](./Infrastructure.md) — Docker Execution Layer

The host layer. Implements `system-protocol-api`, a Node.js service sitting between Safebox's `Protocol.System` and the Docker daemon. M-of-N governance happens in Safebox before any request reaches Infrastructure; this layer provides defense in depth across eight further layers (peer UID via `SO_PEERCRED`, HMAC mutual auth, container allowlist, per-action permissions, image-pattern enforcement, exponential backoff with churn detection, JTI replay protection, systemd hardening).

Reachable only from the qbix process via Unix domain socket, never from the network. Compromised Node cannot bypass governance because it cannot forge a signed verifiedOpToken; compromised API cannot forge responses because Safebox verifies HMAC on the way back.

---

## 🔭 Field convergence

The architectural pattern these five components implement has independent confirmation in 2026 publications from IBM Research, Oracle, Microsoft, and Stanford CRFM. They arrived at structurally equivalent conclusions from different starting points.

| Publication | Org | Date | What they call it / what we call it |
|---|---|---|---|
| [Governance by Construction for Generalist Agents](https://arxiv.org/abs/2605.20874) | IBM Research Haifa | May 2026 | Five-checkpoint policy interception, policy-as-code, no model fine-tuning ↔ `Action.propose` + Safebox policy machinery |
| [Agent Logic and Scalable AI Adoption](https://huggingface.co/blog/ibm-research/agent-logic-and-scalable-ai-adoption) | IBM Research (Nicholas Fuller, VP) | June 2026 | "Pre-indexed structured representation... bounding LLM to local reasoning" with 30× / 15× / 4× token reductions ↔ Grokers + Safebox |
| [From Model Safety to Runtime Governance](https://blogs.oracle.com/ai-and-datascience/runtime-governance-enterprise-agentic-ai) | Oracle OCI (Kishore Pusukuri, Dr.) | April 2026 | "Agent Runtime Controller evaluates schema-valid proposed actions against active policy pack" ↔ Safebox warrant objects |
| [Agent Governance Toolkit](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents/) | Microsoft Open Source | April 2026 | "Actions the AGT kernel denies are not 'unlikely.' They are structurally impossible." ↔ same property, hardware-attested |
| [The Shift from Models to Compound AI Systems](https://bair.berkeley.edu/blog/2024/02/18/compound-ai-systems/) | UC Berkeley / Databricks (Zaharia, Khattab) | Feb 2024 | Compound systems beat monolithic models — Grokers and Safebox are formal instances |
| [DSPy](https://arxiv.org/abs/2310.03714) | Stanford CRFM (Omar Khattab) | 2023→ | Compiling LM programs into optimized prompts ↔ Grokers' Wisdom Library compiles them into cached deterministic programs |

Convergence across IBM, Oracle, Microsoft, Stanford, and Berkeley in one calendar year on the same architectural conclusion. The labels vary by organization, but the primitive is the same one: **structured action representation as the unit of governance, enforced at runtime by deterministic policy, separated from the model that proposed it.**

What this repo adds beyond any of the above: hardware-attested execution (TPM-attested deterministic AMI, no remote-access vectors), cross-tenant trust properties via the OpenClaiming Protocol, and an open-source implementation that doesn't lock the user into a single cloud vendor.

---

## 🪨 Substrate primitives

Underneath every component is **Qbix Streams** — a directed property graph stored as ordinary MySQL tables. Each node is a typed stream with attributes, a participant list with read/write/admin levels, and an append-only message timeline. Each edge is a row in `streams_related_to` and `streams_related_from`, indexed in both directions.

Three properties matter for everything above:

- **Relations are first-class typed edges.** Faceted search and neighborhood traversal are native operations. Most graph databases require recursive query workarounds for the same patterns.
- **Workspaces and forks add copy-on-write.** `Streams::fork()` snapshots a relation neighborhood at a point in time; the workspace cascade resolves reads transparently; tombstones suppress deletes without rewriting history.
- **Per-stream access control.** Every node has its own participant list and four-tier inheritance (public, contact-label, participant role, direct grant). Sharing one artifact or one conversation is a routine operation.

Production deployment since 2011. Powers the [Groups](https://groups.app) app (~7M users across 100+ countries). Source: [github.com/Qbix](https://github.com/Qbix).

---

## 🔏 OpenClaiming Protocol

Every signed assertion in the stack — every approval, every authorization, every cross-system attestation — is an OpenClaim. The protocol is a single canonical envelope for signed claims, with RFC 8785 JCS canonicalization inlined for deterministic serialization and EIP-712 typed-data signatures so the same claim verifies under any EVM wallet, any browser, and any backend with no glue code.

The shape stays the same regardless of who is signing or what is being claimed:

| Where it's used | What gets signed | Who signs |
|---|---|---|
| **Safebox** approvals | A proposed action (its manifest, recipient set, capability set, execution window) | M-of-N human approvers or policy contracts |
| **SafeCloud** delegation | An access grant for a chunk range (derived from the HKDF subtree key) | The owner of the data |
| **SafeCloud** payment | A balance commitment for materialization fees | The payer; verified on-chain by Jets |
| **Infrastructure** verifiedOpToken | A request authorized to cross the Unix socket into the Docker daemon | Safebox itself, on behalf of an approved action |
| **Intercoin** governance | A vote, a transfer authorization, a policy update | DAO members, signers, multisig participants |
| **Hardware attestation** | A TEE attestation report committing to the running AMI cascade hash | The hardware manufacturer's root key |
| **Cross-substrate** federation | An identity assertion from one homeserver to another | The asserting server's long-lived key |

Because the envelope is identical across these cases, **one verifier verifies all of them**. The same JavaScript that checks a corporate signer's payment authorization checks a community member's vote, checks a hardware attester's claim about an AMI, checks an SDK's delegation proof to a Jet routing server. New kinds of claims arrive as new claim *types* inside the existing envelope; no new signature scheme is required.

OCP is the substrate-level standard that makes "the model proposes, the substrate decides" work in practice. Without a single signature standard, every component would invent its own, and the policies that compose across components would have to translate between them. With OCP, policies are written once and apply everywhere.

Specification and reference implementations (PHP, JavaScript, Solidity) at [github.com/OpenClaiming](https://github.com/OpenClaiming).

---

## 🧭 Architecture

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

Grokers and Code sit alongside Safebox rather than below Infrastructure because they don't depend on container execution. Grokers depends only on `{Q, Streams, Users}`; Code depends on Grokers plus the Safebox workflow runtime, with ZFS as the materialization layer for filesystem forks.

---

## 📚 Papers

Six arXiv preprints across two research programs — AI inference and applications, and distributed systems. The implementations referenced from this repo correspond to the architectural claims in these papers.

### AI inference and applications

| Paper | Category | What it does | Link |
|---|---|---|---|
| **Grokers: Bottom-Up Inductive Comprehension and Write-Time Intelligence over Typed Knowledge Graphs** | cs.AI | Three formal theorems on bottom-up code/knowledge comprehension: byte-identity (100% KV-cache hit rate via deterministic context assembly), accumulation monotonicity (the deterministic-coverage fraction of a wisdom library is non-decreasing), dual-traversal ordering (generation and comprehension are uniquely-correct reverses on any DAG). | [arXiv:2606.00050](https://arxiv.org/abs/2606.00050) · [PDF](https://safebots.ai/papers/Grokers.pdf) |
| **Context: Proactive Goal-Directed Intelligence via Composable Sandboxed Programs, Declarative Wiring, and Structured Interaction** | cs.AI | Stable-prefix-first prompt assembly turning the KV cache into a shared resource across sessions. System prompt as cache checkpoint 1, summaries 2-4, scratch and tail volatile. The architectural argument behind the Safebox context-enrichment pipeline. | [arXiv:2605.23928](https://arxiv.org/abs/2605.23928) · [PDF](https://safebots.ai/papers/Context.pdf) |
| **LAWS: Learning from Actual Workloads Symbolically — A Self-Certifying Parametrized Cache Architecture for Neural Inference, Robotics, and Edge Deployment** | cs.LG | Certified-expert library with Lipschitz-bounded correctness; programs synthesized from observed inference traces; deterministic fast-path that grows monotonically. Applies across neural inference, robotics control, and edge workloads. | [arXiv:2605.04069](https://arxiv.org/abs/2605.04069) · [PDF](https://safebots.ai/papers/LAWS.pdf) |
| **Sequential KV Cache Compression via Probabilistic Language Tries: Beyond the Per-Vector Shannon Limit** | cs.LG | Per-token surprisal bound on per-vector entropy proven hundreds of times below the per-vector Shannon limit that TurboQuant and per-vector quantization approaches operate against. The gap widens as context grows. | [arXiv:2604.15356](https://arxiv.org/abs/2604.15356) · [PDF](https://safebots.ai/papers/KV.pdf) |
| **Probabilistic Language Tries: A Unified Framework for Compression, Decision Policies, and Execution Reuse** | cs.LG | The data structure underneath the inference-time experts in LAWS and the sequential bound in the KV paper. One representation that unifies compression, policy learning, and cached execution. | [arXiv:2604.06228](https://arxiv.org/abs/2604.06228) · [PDF](https://safebots.ai/papers/PLT.pdf) |

### Distributed systems

| Paper | Category | What it does | Link |
|---|---|---|---|
| **Intercloud: Eventual Consistency for Decentralised Economies via Chilling-Effect Consensus** | cs.DC | Submitted to DISC 2026. Bidirectional blockchain interoperability without trusted oracles; chilling-effect consensus replaces voting-with-stake as the deterrent mechanism. | [arXiv:2605.22830](https://arxiv.org/abs/2605.22830) · [PDF](https://safebots.ai/papers/IC.pdf) |
| **Magarshak Machine / SPACER** | — | The execution model and consensus property underneath the Safebox sandbox: pristine environment, declared I/O surface, deterministic replay-hash, with formal results on what can and cannot be proven about the execution after the fact. | [PDF](https://safebots.ai/papers/MM.pdf) |

A patent portfolio of seven provisional filings covers the corresponding implementation work.

---

## 📝 Long-form essays

Architectural reference material on the substrate, the governance model, and what they make possible. Each links to a deeper write-up on safebots.ai.

### Architecture

| Essay | What it covers |
|---|---|
| [The ETHOS Stack](https://safebots.ai/ethos.html) | Extensible Trusted Homeserver Operating System. Seven components, one cryptographic discipline. The unified architectural document. |
| [Safebox](https://safebots.ai/stack.html) | The five structural walls (workflow, workload, tool, capability, protocol) that make wrong behavior structurally impossible rather than discouraged by policy. |
| [Safebots](https://safebots.ai/safebots.html) | How organizations collaborate via bots, goals, tools, judgments, and policies. The argument for moving things forward while preserving governance. |
| [Infrastructure](https://safebots.ai/infrastructure.html) | Multi-tenant AI hosting where compliance becomes a consequence of how the system runs, not a separate layer bolted on. HIPAA, GDPR, SOC2, and PCI fall out of attested hardware, sealed key custody, and per-stream access control. |

### Why the substrate beats the agent on every dimension that matters

| Essay | What it covers |
|---|---|
| [Why agent swarms parallelize the wrong unit](https://safebots.ai/kimi.html) | Kimi K2.6's swarm parallelizes LLM calls; Safebox parallelizes tools and treats the LLM as a callable resource within them. Same deliverable, ~95% cheaper per run, unbounded parallelism width vs the ~300-agent ceiling that the coordinator's context window imposes, with full audit and M-of-N governance. |
| [Why Safebots beat OpenClaw and Hermes on every dimension that matters](https://safebots.ai/openclaw.html) | OpenClaw hit 365K stars and 138 CVEs in 63 days. Hermes followed with 165K stars and the same architectural exposure. Both inherit the user's credentials, run prompt-based governance, and trust community skill marketplaces. Safebox does what they do, on any device, with structural enforcement instead of prompt-level promises, and a cost curve that improves with network scale. |
| [99% of agent capability with none of the unsupervised side effects](https://safebots.ai/agents.html) | Workflows of declarative tools with manifests, reputation, and M-of-N governance cover 99% of what open-ended agents do. The remaining 1% is precisely the unsupervised-side-effect category that should not exist. Includes the architectural mechanism (toposort, on-demand tool generation, capability sandboxing) and the failure modes it structurally prevents. |
| [Grokers vs Cursor / Claude Code](https://safebots.ai/grokers.html) | Pre-computed knowledge graphs beat on-demand inference. Cursor and Claude Code re-infer the dependency graph every turn; Grokers materializes it once and queries it in <10ms. 100–1000× cheaper on common operations, structurally correct rather than guess-based, with topological parallel refactoring at O(depth) instead of O(N). |
| [Skills as filesystem vs streams](https://safebots.ai/skills.html) | Anthropic's Skills primitive is the right answer for a filesystem substrate. On a graph substrate that already organizes, audits, and federates community knowledge, the same primitive becomes duplicative — the substrate handles what Skills tries to layer on top. |

### Direction and history

| Essay | What it covers |
|---|---|
| [Singularity](https://safebots.ai/singularity.html) | Why this layer of infrastructure decides whether AI capability deployment goes well. Hardware-attested execution, structural governance, and capability partitioning are the primitives that make superhuman capability deployable without ceding control of side effects. |
| [The Compromise Problem](https://safebots.ai/compromise.html) | Five real production incidents where agents did exactly what they were not asked to do — deleting databases, fabricating records, lying about recovery — and the seven structural defenses that would have made each one impossible. Empirical evidence that prompt-level guardrails fail and that substrate-level governance is what survives contact with adversarial inputs and overconfident models. |
| [Origins](https://safebots.ai/origins.html) | How Qbix, Intercoin, and Safebox got built before AI made them obviously needed. Fifteen years of substrate work across three layers: the federated stream substrate, the on-chain economic layer, and the governance plugin that ties them together. |
| [Wisdom](https://safebots.ai/wisdom.html) | Grokers, Safebots, and Safebox mapped onto three faculties of how the mind moves from insight to deliberation to durable knowledge. |
| [Journey](https://safebots.ai/journey.html) | The founder's path. |

---

## 🚦 Status

Active development across all components.

- **Safebox** — production-ready core; governance, Protocol adapters, context assembly, and local-model runners in current sprint.
- **Safebots** — developer-preview readiness; chat orchestration, storefront greeting, and contract-enforcement judgment approaching smoke-test.
- **Grokers** — integration in progress against the Safebox context-enrichment pipeline; symbol indexing and extern graph construction complete for PHP, JavaScript, TypeScript, Python, Java, C, C++, C#, Swift, Rust, Go.
- **Code** — workflows specified (`cloneRepo`, `modify`, `regrok`); ZFS dataset provisioning integrated with Infrastructure; parallel-team semantics validated against the workspace cascade.
- **Infrastructure** — production-ready; systemd service, HMAC mutual auth, allowlist, backoff, and JTI replay protection all implemented.

---

## 👋 Contact

[**Greg Magarshak**](https://linkedin.com/in/magarshak) · Qbix · Intercoin · Safebots Inc

Explore the source:

[![Qbix](https://img.shields.io/badge/github.com%2FQbix-substrate%20%26%20Safebots-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Qbix)
[![Intercoin](https://img.shields.io/badge/github.com%2FIntercoin-smart%20contracts-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Intercoin)
[![OpenClaiming](https://img.shields.io/badge/github.com%2FOpenClaiming-OCP%20spec%20%26%20libs-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/OpenClaiming)

---

## 🤝 Join the movement

Every Sunday at 5PM ET we run a live AMA. Bring a problem you want to solve, a workflow you want to build, a question about the architecture, or just curiosity about what's actually possible. First-time users get set up in the first ten minutes; builders pair-build new workflows live; organizations sketch real solutions to real problems on the call. No pitch deck. No fee. The library grows every week and so does the network.

[![Save your spot at the weekly AMA](https://img.shields.io/badge/%E2%86%92%20Save%20your%20spot%20at%20the%20weekly%20AMA-4CAF50?style=for-the-badge&logoColor=white)](https://calendly.com/safebotsai/weekly)

---

*Qbix · Intercoin · Safebox · Safebots · Grokers · Code · Infrastructure*
