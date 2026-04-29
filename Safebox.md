# Safebox Plugin — Architecture & Implementation Reference

> **Audience**: LLM implementers and developers building the Safebox plugin for Qbix.  
> This document is self-contained. It covers every layer of the architecture, all stream types, the sandbox API, PHP/JS conventions, database tables, and the Safebux economic model.

---

## Table of Contents

1. [Overview & Philosophy](#1-overview--philosophy)
2. [Plugin File Structure](#2-plugin-file-structure)
3. [Stream Types & Ontology](#3-stream-types--ontology)
4. [Workflows, Workloads, Steps, Tasks](#4-workflows-workloads-steps-tasks)
5. [Tools & Capabilities](#5-tools--capabilities)
6. [The Sandbox API (Q.Sandbox)](#6-the-sandbox-api-qsandbox)
7. [Streams.fetch — Lazy Materialization](#7-streamsfetch--lazy-materialization)
8. [Streams.related — Facet Search](#8-streamsrelated--facet-search)
9. [Action.propose — Write Semantics](#9-actionpropose--write-semantics)
10. [Safebox.yield — Intermediate Results](#10-safeboxyyield--intermediate-results)
11. [Q.batcher in Capabilities — Transparent Batching](#11-qbatcher-in-capabilities--transparent-batching)
12. [Protocol Reference](#12-protocol-reference)
13. [API Key Management and the Executor Node](#13-api-key-management-and-the-executor-node)
14. [Capability Auto-Generation](#14-capability-auto-generation)
15. [Governance](#15-governance)
   - 15.1. [The Governance Primitive](#151-the-governance-primitive)
   - 15.2. [Community Policies — Action Governance](#152-community-policies--action-governance)
   - 15.3. [Judgments — Automating Individual Vote Decisions](#153-judgments--automating-individual-vote-decisions)
   - 15.4. [Keys as Attested Claims](#154-keys-as-attested-claims)
16. [Workspaces](#16-workspaces)
17. [Safebux Economics](#17-safebux-economics)
18. [Migration Script](#18-migration-script)
19. [Database Schema](#19-database-schema)
20. [PHP Implementation Guide](#20-php-implementation-guide)
21. [JavaScript Implementation Guide](#21-javascript-implementation-guide)
22. [Node.js Implementation](#22-nodejs-implementation)
23. [Node.js Orchestrator](#23-nodejs-orchestrator)
24. [End-to-End Flow Example](#24-end-to-end-flow-example)
25. [Context Assembly and Enrichment](#25-context-assembly-and-enrichment)
26. [Protocol.System — Container Governance](#26-protocolsystem--container-governance)
25. [Appendix: Configuration Reference](#appendix-configuration-reference)

---

## 1. Overview & Philosophy

Safebox is a Qbix plugin implementing **governed AI workflow infrastructure**. It lets organizations define workflows, run them on attested executors, interact with external providers, and enforce governance policies — all with a full audit trail stored as Streams.

### How Everything Fits Together

```
PUBLIC STREAMS                          PRIVATE / ORG STREAMS
(publisherId = "Safebox" or reverse-domain)
                                        (publisherId = orgExecutorUserId)

Safebox/workflow/{slug}                 Safebox/workload/{id}
  └── [Safebox/step] ──►                  └── [Safebox/workload/task] ──►
      Safebox/step/{id}                       Streams/task/{id}
        Safebox/tool: "Safebox/tool/..."         instructions: { toolStreamName,
        Safebox/inputs: [...]                                    branchContext,
        Safebox/outputs: [...]                                   inputs }
              │                                       │
              │  (kickoff → Node compiles plan.json)  │  (executor runs tool code)
              ▼                                       ▼

Safebox/tool/{sha256}               ┌─────────────────────────────────────┐
  code.js (sandbox, read-only)      │  SANDBOX  (Web Worker / vm2)        │
  M-of-N auditor signatures         │  Streams.get/fetch/related          │
              │                     │  Action.propose                     │
              │                     │  Safebox.yield                      │
              └─────────────────────┘
                    │
      ┌─────────────┘
      ▼
CAPABILITY
Safebox/capability/Websites/post
  runs via Protocol.HTTP/LLM/SMTP/etc.
  credential: {{credentials:key}}
  charges Safebux
```

### Execution Flow

```
1. KICKOFF
   PHP: Safebox_Workflow::kickoff() → creates Safebox/workload, sendToNode compile

2. PLAN COMPILATION (Node.js)
   Fetches all steps + edges in parallel → Kahn's algorithm → writes plan.json
   → sendToPHP Safebox/workload/compiled {roots[]}

3. TASK DISPATCH (PHP)
   Creates Streams/task per root step, writes branchContext into instructions
   → sendToNode Streams/task/run for all roots simultaneously

4. TOOL EXECUTION (Node.js sandbox)
   Streams.fetch() → checks cache → runs Capability if miss → charges Safebux
   Action.propose() → queued, PHP runs cheap policies synchronously
   Safebox.yield() → PHP dispatches successor tasks immediately

5. CAPABILITY RUN (if Streams.fetch miss)
   Finds Safebox/capability for (publisherId, typePrefix)
   Loads + verifies code, substitutes {{credentials:key}} credentials
   Calls Protocol.http / llm / smtp / files
   Writes result stream, charges Safebux

6. ACTION FLUSH (on task success)
   Reads safebox_action_queue for this task
   Runs policy check per action (deterministic sync, or M-of-N async)
   Approved actions execute: Streams CRUD, SMTP, Telegram, etc.

7. SUCCESSOR DISPATCH (PHP, on each yield or completion)
   Merges outputs into branchContext, runs Kahn's over plan.json
   Creates + dispatches all simultaneously-satisfiable successors

8. WORKLOAD COMPLETE
   All tasks done → Safebox/workload/completed message, Safebux settlement
```

### Core Design Principles

**Rails before runtime.** Every tool, capability, and workflow is audited before any execution.

**Everything is a stream.** External data, AI outputs, tasks, workloads, capabilities, actions — all are Qbix Streams.

**Tools only read and propose.** Sandbox code reads streams and proposes actions. It never writes directly.

**Streams.fetch is the single entry point for external data.** All external data enters through `Streams.fetch`, materialized lazily, cached permanently.

**Costs fall as usage grows.** Cache hits cost less. Early materializers earn commissions.

---

## 2. Plugin File Structure

```
Safebox/
├── Safebox.php
├── config/
│   └── plugin.json                      # Stream types, routes, event hooks
├── scripts/Safebox/
│   └── 0.1-Streams.mysql.php
├── classes/
│   ├── Safebox.php
│   └── Safebox/
│       ├── Workflow.php                 # kickoff(), validate()
│       ├── Workload.php                 # onPlanCompiled(), advanceWorkflow(), checkComplete()
│       ├── Task.php                     # create(), onComplete(), onYield()
│       ├── Action.php                   # onActionCreated(), executeNow(), flushQueue()
│       ├── Capability.php               # fetchStream(), findCapability()
│       ├── EmailWebhook.php             # Inbound email materializer (PHP)
│       ├── Webhook.php                  # Provider webhook routing (arrive, find)
│       ├── Credentials.php              # Credential stream model (PHP)
│       ├── Orchestrator.js              # compilePlan(), runTask(), runCapability()
│       ├── Sandbox.js                   # Q.Sandbox tool/capability execution
│       ├── Protocol.js                  # All Protocol.* adapters
│       └── Credentials.js                  # ECIES decrypt (Node.js)
├── handlers/
│   └── Safebox/
│       ├── workflow/, workload/, task/, action/
│       ├── actionQueue/, yield/, stream/
│       ├── executor/, capability/, credential/
│       ├── webhook/response.php         # Provider webhook ingestion (all providers)
│       └── email/webhook/post.php       # Email stream materializer (Node→PHP)
└── web/js/
```

**Handler registration — two mechanisms:**

Direct event handlers are autoloaded by file path (no config needed):
```
Q::event("Streams/create/Safebox/action")  →  handlers/Streams/create/Safebox/action.php
Q::event("Safebox/workload/compiled")      →  handlers/Safebox/workload/compiled.php
```

Before/after hooks must be declared in `plugin.json`:
```json
{
  "Q": {
    "handlersAfterEvent": {
      "Q/Plugin/install":                    ["Safebox/after/Q_Plugin_install"],
      "Db/Row/Streams_Message/saveExecute":  ["Safebox/after/Db_Row_Streams_Message_saveExecute"]
    },
    "handlersBeforeEvent": {
      "Db/Row/Streams_Stream/save":          ["Safebox/before/Db_Row_Streams_Stream_save"]
    }
  }
}
```

---

## 3. Stream Types & Ontology

**Stream names always start with their type.** `publisherId="com.x"` + `streamName="Websites/post/1234"` ✓

A new type is only added when two or more independent providers map to the same shape and no existing type fits. Type is shape only — behavior comes from the Streams substrate.

### Core Streams (universal primitives)

| Type | Shape |
|---|---|
| `Streams/file` | Binary artifact — URL, MIME, size |
| `Streams/category` | Taxonomy node / classification container |
| `Streams/chat` | Conversation container with message history |
| `Streams/task` | Work item with status, assignee, dependencies |
| `Streams/space` | Social container — group, community, organization |
| `Streams/image` | Raster image — URL, dimensions, alt text |
| `Streams/audio` | Audio clip — URL, duration |
| `Streams/video` | Video clip — URL, duration, thumbnail |
| `Streams/pdf` | PDF document |
| `Streams/text` | Body of text; used for LLM completions |
| `Streams/live` | Live audio/video broadcast |
| `Streams/incoming` | Channel for receiving messages from external systems |
| `Streams/outgoing` | Channel for sending messages to external systems |

### Web Content (Websites namespace)

| Type | Maps to |
|---|---|
| `Websites/post` | Twitter, Facebook, Telegram channels, Discourse, LinkedIn |
| `Websites/article` | Medium, Substack, WordPress |
| `Websites/webpage` | Any crawled/static page |
| `Websites/profile` | Twitter, LinkedIn, GitHub identity pages |
| `Websites/feed` | Twitter search, Reddit subreddit, RSS/Atom |

### Commerce (Assets plugin)

| Type | Maps to |
|---|---|
| `Assets/product` | Shopify, Amazon, WooCommerce |
| `Assets/service` | Freelancer listing, SaaS tier |
| `Assets/holding` | Token balance, NFT, equity, customer record |

### Safebox Internal Stream Types

| Type | Shape | publisherId |
|---|---|---|
| `Safebox/workflow` | Template — tree of steps | `"Safebox"` |
| `Safebox/step` | One node — tool ref, inputs/outputs | `"Safebox"` |
| `Safebox/workload` | Running instance — status, plan.json | `communityId` |
| `Safebox/tool` | Sandboxed JS business logic | `"Safebox"` or `communityId` |
| `Safebox/capability` | Sandboxed JS stream materializer | provider publisherId |
| `Safebox/action` | Proposed side effect | `communityId` |
| `Safebox/policy` | Governance gate | `communityId` (for community policies) or `userId` (for user auto-vote policies) |
| `Safebox/keys` | OpenClaim-attested key ring for a user | `communityId` (the community that issued the role attestations) |
| `Safebox/executor` | Registered executor | `communityId` |
| `Safebox/provider` | External system registration | `"Safebox"` |
| `Safebox/credential` | Encrypted API key | `communityId` |

### Email Job Streams (inbound email)

Inbound emails are materialised as `Safebox/job` streams private to the org:

```
publisherId  = communityId
streamName   = Safebox/job/email/{provider}/{messageId}
type         = Safebox/job
content      = plain-text body (truncated 64 KB for DB; full in body.txt)

Attributes:
  Streams/email/messageId   — RFC 2822 Message-ID
  Streams/email/from        — sender address
  Streams/email/to          — recipient(s), comma-separated
  Streams/email/subject     — subject line
  Streams/email/date        — ISO 8601 date
  Streams/email/provider    — 'aws' | 'cloudflare'
  Streams/email/hasHtml     — 'true' | 'false'
  Safebox/status            — 'arrived'
  Safebox/attachmentCount   — number of attachments (string)
```

Attachment sub-streams (type `Streams/file`, related to email stream):
```
streamName = {emailStreamName}/attachments/{index}

Attributes:
  Streams/file/filename   — original filename (sanitised)
  Streams/file/mimeType   — MIME content type
  Streams/file/size       — bytes (string)
  Streams/file/sha256     — SHA-256 hex of file content
  Streams/file/path       — relative path from APP_FILES_DIR
```

File layout on disk (mirrors Qbix `Q/uploads/Streams/{splitId}/{streamName}/` convention):
```
Q/uploads/Streams/{splitPublisherId}/{streamName}/
  body.txt
  body.html
  attachments/
    {sha256}/
      {filename}     ← content-addressed; two identical attachments share one file
```

### Channel Naming Convention

```
{communityId} / Streams/incoming/imap/{emailAddress}       ← IMAP inbound
{communityId} / Streams/outgoing/smtp/{emailAddress}       ← SMTP outbound
{communityId} / Streams/incoming/telegram/{chatId}         ← Telegram inbound
{communityId} / Streams/outgoing/telegram/{chatId}         ← Telegram outbound
{communityId} / Safebox/job/email/{provider}/{messageId}   ← inbound email job stream
```

All channel streams for a customer relate to their `Assets/holding/{customerId}` stream.

---

## 4. Workflows, Workloads, Steps, Tasks

### Step Stream

```
publisherId: "Safebox"
streamName:  Safebox/step/{uniqueId}
attributes:
  Safebox/tool:     "Safebox/tool/{sha256}"
  Safebox/inputs:   ["user","brandColors"]    ← indexed via syncRelations
  Safebox/outputs:  ["twitterProfile"]         ← indexed via syncRelations
  Streams/types:    {"user":"Websites/profile"} ← not indexed — validation metadata
```

Step-to-step edges use relation type `Safebox/step/follows` with `extras`:
```json
{ "Safebox/mapping": { "consumerInputLabel": "producerOutputLabel" } }
```

### plan.json — Frozen Workflow Snapshot

Written by Node at kickoff. PHP reads this file (not live stream queries) on every yield and completion to find satisfiable successors. Mid-flight workflow edits never affect a running workload.

```json
{
  "workflowId": "enrich-contacts",
  "frozenAt": 1711700000,
  "input": { "communityId": "NYU" },
  "steps": [
    { "stepStreamName": "Safebox/step/fetchUsers",
      "toolStreamName":  "Safebox/tool/abc123",
      "inputs": [], "outputs": ["user"] }
  ],
  "edges": [
    { "from": "Safebox/step/fetchUsers",
      "to":   "Safebox/step/fetchProfile",
      "mapping": { "user": "user" } }
  ]
}
```

### Branch Context

Each task's `instructions` blob carries a `branchContext` — the union of all named outputs from ancestor tasks in its branch. PHP reconstructs full execution state from this closure on every callback (PHP is stateless between requests).

### Kickoff Flow

```
PHP: kickoff() → creates workload (status=compiling) → sendToNode Safebox/workload/compile

Node: compilePlan()
  → fetches all Safebox/step streams + follows-edges in parallel (Promise.all)
  → Kahn's: identifies roots, validates no cycles
  → validates tool existence and input wiring
  → writes plan.json → sendToPHP Safebox/workload/compiled {roots[]}

PHP: onPlanCompiled()
  → sets workload status=running
  → creates Streams/task per root with branchContext in instructions
  → sendToNode Streams/task/run for all roots simultaneously
```

---

## 5. Tools & Capabilities

| | Tool | Capability |
|---|---|---|
| Purpose | Business logic, orchestration | Materializing external streams |
| Calls | `Streams.get/fetch/related`, `Action.propose` | `Protocol.*` adapters |
| Triggered by | Orchestrator executing a workflow step | `Streams.fetch` on missing stream |
| Storage | `Safebox/tool/{sha256}` | `Safebox/capability/Websites/post` |

Tool code: `plugins/Safebox/files/tools/{sha256[:3]}/{sha256}.js`. The `Safebox/sha256` attribute must equal `sha256(code.js)` and match the stream name's last segment. Any byte change → new hash → new stream → new audit. `title` and `content` fields are LLM-generated display summaries — auditors review `code.js` directly.

---

## 6. The Sandbox API (Q.Sandbox)

All sandbox method keys are flat dot-separated strings: `"Streams.get"`, `"Action.propose"`, `"Safebox.yield"`. `buildPreamble()` translates these into ergonomic nested objects inside the sandbox.

**Execution hash** covers `code + ctx + input + rpcLog + result` (preamble excluded — derived). Used for idempotent replay and action queue deduplication.

### Complete API

```js
// ── READS ─────────────────────────────────────────────────────────────────

Streams.get(publisherId, streamName)
// Returns stream fields or null. Never blocks for materialization.

Streams.fetch(publisherId, streamName)
// Materializes via capability if missing. Blocks until status="ready". Charges Safebux.

Streams.related(publisherId, streamName, relationType, ascending, options)
// Object shorthand: { hashtag: "bitcoin" } → "attribute/hashtag=bitcoin"
//                   { category: "food" }   → "category/food"
// Multi-key = AND semantics. No materialization triggered.

Streams.refresh(publisherId, streamName)
// Force re-materialization even if ready. Full cost.

Streams.lookup(publisherId, types, titleQuery, orderByTitle)
// Find streams by type + title LIKE pattern. titleQuery is a SQL LIKE pattern,
// e.g. "Invoice%" or "%draft%".

Streams.search(query, options)
// Full-text facet search across the substrate's search index.

Streams.getMessages(publisherId, streamName, options)
// Read posted messages from a stream the caller has read access to.
// Returns an array of plain message objects newest-first by default,
// each carrying { ordinal, type, content, byUserId, sentTime,
// instructions, weight }. Options: limit (default 100, capped by the
// per-type Streams/types/<type>/getMessagesLimit config, default 1000),
// min / max for ordinal range (inclusive), ascending (true for
// oldest-first), type to filter by message type. Access is enforced
// via testReadLevel('messages') inside Stream.getMessages — no read
// access yields an empty array, no error.

Streams.getParticipants(publisherId, streamName, options)
// Read the participation roster. Returns an array of plain participant
// objects, each carrying { userId, state, posted, subscribed, reason,
// role, extra, insertedTime, updatedTime }. Options: limit (default
// 100, capped by Streams/types/<type>/getParticipantsLimit, default 100),
// state ('participating', 'left', 'invited'), ascending (default false).
// Access is enforced via testReadLevel('participants') — no read
// access yields an empty array.

// ── WRITES (all go through Action.propose — see §9) ──────────────────────

var a1 = await Action.propose("Streams/create", {
  publisherId, streamType, fields, attributes
  // follows: []  = root of DAG / independent branch
  // omit        = auto-chain to previous action in this batch (sequential default)
});
// Returns { actionId } immediately — PHP ran cheap deterministic policies
// synchronously. Voting policies resolve async as votes accumulate (§15.1).
// a1.actionId is usable in subsequent follows arrays.

await Action.propose("Streams/update",  { publisherId, streamName, fields, attributes });
await Action.propose("Streams/relate",  { fromPublisherId, fromStreamName,
                                          toPublisherId,   toStreamName, relationType, weight });
await Action.propose("Streams/unrelate", { /* same shape as relate */ });
await Action.propose("Streams/message", { publisherId, streamName, messageType, instructions });
await Action.propose("Streams/updateRelation", { /* atomic unrelate+relate */ });
await Action.propose("Web3/transaction", { /* signed EVM tx — see §12 Protocol.Web3 */ });

// ── YIELDS ────────────────────────────────────────────────────────────────

Safebox.yield({ outputLabel: streamOrScalar, ... })
// Merges into task.yieldedOutputs. Fire-and-forget. PHP dispatches any unblocked successors.

// ── DATA HELPERS (pure functions over blob content) ──────────────────────

Data.slice(content, fromLine, toLine)   // line-range extraction
Data.grep(content, pattern, contextLines) // pattern match with optional context
Data.structure(content, type)            // parse: "json" | "jsonl" | "csv" | "tsv"
Data.jsonl(content, filter)              // filter JSONL by predicate expression

// ── CACHE & RUNTIME ───────────────────────────────────────────────────────

Cache.get(key) / Cache.set(key, value)      // workload-scoped

Crypto.sign(payload)                         // ES256 signature using the workload's
                                             //   communityId-scoped key (OpenClaim format)
Crypto.verify(payload, signature, publicKey) // verify an OpenClaim signature
Crypto.EVM.submitTransaction(tx)             // signed EVM tx — routes through
                                             //   Action.propose("Web3/transaction")

Runtime.log(level, message, extra)           // structured log, appears in rpcLog
Runtime.sleep(ms)                            // yielding sleep — does NOT spin
Runtime.llm(spec)                             // shorthand for Protocol.LLM (§12)
Runtime.fork(stepStreamName, inputs)          // fan out — schedule a sub-step
```

---

## 7. Streams.fetch — Lazy Materialization

```
Streams.fetch(publisherId, streamName)
  ├── exists + status="ready"   → return fields (cache hit)
  │     charge Safebux; 50% commission → original payer
  ├── exists + status="pending" → wait for Streams/changed
  └── doesn't exist →
        create as "pending"
        find capability: (publisherId, typePrefix of streamName)
        run capability → status="ready"
        return fields (full materialization cost)
```

**`Safebox/status` is always an attribute, never a field.**

---

## 8. Streams.related — Facet Search

```js
Streams.related(publisherId, streamName, type, isCategory, opts)
```

Thin wrapper over `Q.Streams::related`. `type` is the relation type string; pass `undefined` to return all relations. `isCategory` defaults to `true` — the query finds streams related *to* `(publisherId, streamName)` as a category hub. `opts` supports `limit`, `offset`, and `ascending`.

Facet search on attributes is expressed by traversing category streams whose names encode the facet: `Safebox/category/tag/{tagName}`, `Safebox/category/lang/{code}`, etc. Multi-facet AND is modeled by intersecting result sets at the tool level.

No materialization is triggered by `Streams.related` — it is a read-only index query. Materialization happens only through `Streams.fetch` (§7).

---

## 9. Action.propose — Write Semantics

All writes go through `Action.propose`. Tools never call `Streams::create`, `Streams::update`, or `Streams::relate` directly. Reasons: atomicity (actions only execute if the tool completes without abort), auditability (entries in rpcLog feed the execution hash), policy governance (every proposal triggers the community's policy runner), replay safety.

**Signature:**

```js
Action.propose(type, payload)  →  Promise<{actionId}>
```

**The eight action types:**

| type | what it does |
|---|---|
| `Streams/create` | Create a new stream. For `Safebox/capability` and `Safebox/tool` streams, `payload.attributes['Safebox/pendingCode']` gets written to disk and sha256-verified at execute time. `Safebox/approved` is not writable from this path — approval comes only through governance (§15.4). |
| `Streams/update` | Mutate whitelisted fields (`title`, `content`, `icon`) and non-governance attributes on an existing stream. Cannot change access levels, stream type, or publisherId. |
| `Streams/relate` | Insert the `Streams_RelatedTo` + `Streams_RelatedFrom` row pair. Payload: `toPublisherId`, `toStreamName`, `relationType`, `fromPublisherId`, `fromStreamName`, optional `weight`. |
| `Streams/unrelate` | Remove the pair. Same payload shape. |
| `Streams/message` | Post a `Streams_Message`. Bumps messageCount, writes the row, fires the notify cascade (online via socket, offline via email/mobile/push per each participant's subscription rules). |
| `Streams/updateRelation` | Atomic unrelate+relate for type change (`changeType` field) or weight-only update in place. Single transaction. |
| `Streams/registerRelations` | **Blocked from the governance pipeline.** Platform-global `syncRelations` index install — runs outside `Action.propose` with a rail-level policy (`railLevelSignaturesRequired`, see §15.4). |
| `Web3/transaction` | Signed EVM transaction via `Users_Web3::execute`. Gated by `Safebox/web3/allowPayloadPrivateKey` config opt-in; appId derives from the community config, not payload. |

**Action streams.** Each proposal creates a `Safebox/action/{actionId}` stream published by `communityId` (NOT the workload publisher). Attributes: `Safebox/actionType`, `Safebox/payload` (JSON), `Safebox/status`, `Safebox/approvals`, `Safebox/rejections`. `syncRelations` on `Safebox/status` keeps the workspace index updated automatically.

**Lifecycle:** `proposed → held | approved | rejected → executed | abandoned | failed`.

**DAG via `follows`.** `payload.follows` is an array of prior actionIds this action waits for. The executor runs Kahn's topological sort over the batch: actions with zero in-batch ancestors execute first, their dependents unblock as each completes. If `follows` is unspecified, the Orchestrator auto-chains to the immediately prior action in the same propose batch — sequential tools get sequential semantics for free. Pass `follows: []` (or `null`) to opt out for root-of-DAG / parallel branches.

**Hold semantics.** When an action's governance is still `held` or `proposed` (awaiting M-of-N threshold), its **transitive in-batch dependents** wait — we BFS-mark them and skip this flush pass. Sibling actions that don't follow the held node run freely. When the threshold eventually resolves, the next `flushQueue` call picks up the held sub-DAG and resumes Kahn from there. This is the specific reason Kahn is worth it over a sequential walk: a threshold policy holding A mid-batch doesn't strand B, which was also unblocked but has nothing to do with A.

**Reject cascade.** A predecessor in `rejected` / `abandoned` / `failed` status cascades abandonment to dependents, unless the dependent carries `payload.blocking = false`.

**Stream-side audit trail.** Each `follows` entry also creates a `Safebox/follows` relation between the action streams, so the DAG is visible through normal Streams search (UI history views, audit reports). The executor does not read these relations during dispatch — it reads the follows-array from the queue table's JSON column (named `dependsOn` at the schema level for compatibility with pre-r29 installs; treated as `follows` everywhere above that column). The relation and the column are kept in sync at propose-time so the relation graph matches execution order.

**Execution timing.** Actions execute in real-time as soon as their governance resolves AND their DAG predecessors complete — not in a batch at task end. `holdUntilFlush: true` in a policy return value defers a specific action until task completion (all-or-nothing semantics).

---

## 10. Safebox.yield — Intermediate Results

```js
Safebox.yield({ outputLabel: streamOrScalar, ... })
```

Plain string input bindings in `plan.json` resolve from `yieldedOutputs` by name before checking final task results. This enables fan-out: each yield may unblock a downstream branch immediately. All yields appear in `rpcLog` → included in execution hash.

---

## 11. Q.batcher in Capabilities — Transparent Batching

```js
var batcher = Q.batcher(function(items, callback) {
  // batch API call, callback(null, results[]) same length as items
}, { max: 100, ms: 50 });

// Tool calls one-by-one — batching transparent
var tweet = await batcher(tweetId);
```

Use for APIs with batch endpoints (Twitter lookup, OpenAI embeddings). Tool code is unaware of batching.

---

## 12. Protocol Reference

Protocols are the **only** way capability or tool code reaches the outside world. All Protocol methods must be declared in the capability manifest; the sandbox exposes only declared methods. Keys are resolved from the Credentials module before the sandbox runs — capability code never sees credentials, only `{{placeholder}}` tokens.

```
Protocol.LLM(spec)           — LLM chat completion, adapter-dispatched
Protocol.SMTP(spec)          — Send email via platform transport
Protocol.Email(spec)         — Receive/process inbound email
Protocol.SMS(spec)           — Send SMS
Protocol.Push(spec)          — Send push notification
Protocol.Telegram(spec)      — Telegram Bot API (outbound)
Protocol.Web3(spec)          — EVM blockchain
Protocol.Payment(spec)       — Payments (Stripe / Web3)
Protocol.Transcription(spec) — Audio/video → text
Protocol.Diffusion(spec)     — Image generation (Stability AI)
Protocol.Image(spec)         — Image generation/editing (multi-provider)
Protocol.HTTP(spec)          — Raw HTTP (declared URLs only)
Protocol.Files               — Read/write workload files
```

**Design invariants:**

1. No ambient credentials — keys resolved before sandbox runs
2. All calls declared in manifest — undeclared methods throw immediately
3. Normalised responses — consistent shape with `usage` object and adapter-specific `raw`
4. Promise-based — all methods return Promises
5. Adapter dispatch via `spec.adapter` or model prefix — always config-overridable

---

### Protocol.LLM

Chat completion, adapter-dispatched by model name prefix.

| Model prefix | Adapter |
|---|---|
| `claude-*` | `Protocol.LLM.Anthropic` (direct API) |
| `@cf/*` | `Protocol.LLM.Cloudflare` (Workers AI hosted) |
| `{provider}/*` e.g. `anthropic/`, `openai/`, `alibaba/` | `Protocol.LLM.Cloudflare` (AI Gateway proxied) |
| anything else | `Protocol.LLM.OpenAI` (OpenAI-compatible) |

**Spec fields:**
```
model          {string}   Model name. Default: config Safebox/llm/models/{adapter}
apiKey         {string}   {{placeholder}} resolved from Credentials
messages       {array}    [{role, content}] OpenAI-style
system         {string}   Prepended as system message (OpenAI) or top-level param (Anthropic)
maxTokens      {number}   Default: 4096
temperature    {number}   Default: 0.7
_systemBlocks  {array}    Anthropic only: content blocks with cache_control for KV caching
metadata       {object}   Cloudflare only: per-request tags for CF cost dashboard
```

**Response:** `{content, usage: {promptTokens, completionTokens, token}, variant}`

**Cloudflare AI Gateway** (`@cf/*` and `{provider}/*` models):

One endpoint, 70+ models, 12+ providers. Routes through `gateway.ai.cloudflare.com/v1/{accountId}/{gatewayId}/{provider}/v1/chat/completions`. Automatic failover. Unified CF billing. Supports both Cloudflare-hosted open models (`@cf/meta/llama-4-scout-17b-16e-instruct`, `@cf/moonshotai/kimi-k2.5`) and proxied external providers (`anthropic/claude-sonnet-4-6`, `openai/gpt-4o`, `alibaba/qwen3-max`).

```js
// Cloudflare-hosted open model
await Protocol.LLM({
    model:    '@cf/meta/llama-4-scout-17b-16e-instruct',
    apiKey:   '{{credentials:cf_api_token}}',
    messages: [{ role: 'user', content: 'Classify this ticket.' }],
    metadata: { workflow: 'support-triage', tier: 'pro' }
});

// Proxied partner model through AI Gateway — billing through CF, not provider directly
await Protocol.LLM({
    model:    'anthropic/claude-sonnet-4-6',
    apiKey:   '{{credentials:cf_api_token}}',
    messages: [{ role: 'user', content: 'Summarise this invoice.' }]
});
```

Config: `Safebox/cloudflare/apiToken`, `Safebox/cloudflare/accountId`, `Safebox/cloudflare/gatewayId` (default `'default'`).

---

### Protocol.SMTP

Send email via the platform's configured transport (sendmail, SMTP relay, or logged-only). Delegates to the Qbix Users plugin. Supports Handlebars templating.

**Send-only. Authoritative. For receiving email see `Protocol.Email`.**

```
to       {string}   Recipient address(es), comma-separated
subject  {string}   Subject line (Handlebars source)
body     {string}   Plain-text body (Handlebars source)
html     {string}   HTML body (optional)
fields   {object}   Template variables
from     {array}    [address, name] — falls back to Users/email/from config
```

---

### Protocol.Email

**Receive and process inbound email.** Modeled after `Protocol.Transcription` — provider pushes, Safebox materialises a stream, capability fetches the result.

**`Protocol.SMTP` = send. `Protocol.Email` = receive. These are distinct namespaces.**

**Why not IMAP?** Agents react to events, not timers. Both supported providers (SES, Cloudflare Email Routing) deliver via push webhook — zero latency, no polling credentials. IMAP would introduce latency proportional to the poll interval and requires a stateful TCP connection. IMAP is explicitly not implemented as a Protocol; if polling from a legacy mailbox is ever needed, a capability can wrap `Q_Utils::sendToPHP` to call a PHP IMAP handler directly.

**Adapters:** `'aws'` (SES receipt rule → S3 → SNS) | `'cloudflare'` (CF Email Routing Worker)

**Actions:**

| Action | Description |
|---|---|
| `webhook` | Parse raw provider payload → normalised EmailMessage. Called by Node after PHP `arrive()` fires. Does NOT write files — PHP does that. |
| `fetch` | Retrieve materialised email from `Safebox/job/email/{provider}/{messageId}` stream. |
| `reply` | Send reply via `Protocol.SMTP` with `In-Reply-To`/`References` headers set for thread continuity. |

**Full inbound flow:**

```
Provider → POST /Safebox/webhook
  webhook/response.php:
    Cloudflare-Email: externalId = body.message_id
    SES-Email:        externalId = SNS Message → mail.messageId
  Webhook::arrive($rawPayload)
    → sets Safebox/status='arrived', posts Streams/changed (wakes Streams.fetch waiters)
    → if workloadId on stream: sendToNode Safebox/workload/resume
    → if platform == 'Cloudflare-Email':
        Safebox_EmailWebhook::process('cloudflare', rawPayload, communityId, streamName)
          → parses CF Worker JSON payload (_parseCFPayload)
          → writes body.txt, body.html to disk
          → writes attachments/{sha256}/{filename}
          → creates/updates Safebox/job/email/cloudflare/{messageId} stream
          → creates Streams/file sub-streams, relates them
          → posts Streams/changed

For AWS SES (raw RFC 2822 in S3 — PHP cannot parse MIME):
  Workload tool code calls:
    Protocol.Email.Aws({action:'webhook', payload:snsPayload, communityId, jobStreamName})
      → parses SNS envelope, extracts S3 object key
      → fetches raw RFC 2822 from S3 via SigV4 GET
      → returns normalised EmailMessage + _rawBase64
      → chains sendToPHP('Safebox/email/webhook', {email, provider:'aws', communityId, jobStreamName})
    PHP Safebox_email_webhook_post():
      → same file-writing and stream-creation as CF path above
```

**Why SES goes through Node but CF doesn't:** CF Workers pre-parse the email before posting. SES stores raw RFC 2822 in S3 — PHP cannot parse MIME. Node fetches the S3 object and delegates parsing to `Safebox_EmailWebhook::_parseSESPayload` via sendToPHP, which then receives `_rawBase64`.

**EmailMessage normalised shape:**
```
{ messageId, from, to, subject, date, text, html,
  attachments: [{filename, mimeType, size, sha256, data}] }
```

```js
// In a tool or capability:

// Fetch a materialised email
var email = await Protocol.Email({
    action:      'fetch',
    adapter:     'cloudflare',
    messageId:   jobStream.getAttribute('Streams/email/messageId'),
    communityId: ctx.communityId
});
// email.text, email.subject, email.from, email.streamName

// Reply via Protocol.SMTP with threading headers set automatically
await Protocol.Email({
    action:    'reply',
    adapter:   'cloudflare',
    messageId: email.messageId,
    reply: {
        from:    'support@example.com',
        to:      email.from,
        subject: 'Re: ' + email.subject,
        text:    'Thanks for reaching out.'
    }
});
```

Config: `Safebox/email/adapter`, `Safebox/email/aws/bucket`, `Safebox/aws/accessKey`, `Safebox/aws/secretKey`, `Safebox/aws/region`, `Safebox/cloudflare/apiToken`, `Safebox/cloudflare/accountId`.

---

### Protocol.SMS

Send SMS via the platform's Twilio config (or carrier-gateway fallback). Delegates to the Qbix Users plugin.

```
to     {string}   E.164 phone number, e.g. '+14155551234'
body   {string}   Message body (Handlebars source)
fields {object}   Template variables
from   {string}   Sender number — falls back to Users/mobile/from
```

---

### Protocol.Push

Send push notification via APNs/FCM. Delegates to the Qbix Users plugin.

```
userId  {string}   Platform user ID
title   {string}
body    {string}
data    {object}   Extra payload delivered to the app
badge   {number}   iOS badge count
sound   {string}
```

---

### Protocol.Telegram

Telegram Bot API — **outbound actions only for capability code.** 37 methods across 6 categories.

**Architecture boundary — critical:**

```
Inbound (webhook, stream creation, user mapping):
  → Telegram plugin PHP: Telegram_after_Telegram_log
    Creates Telegram/chat/{chatId} streams, maps users via futureUser(),
    sets botStatus, chatType, isForum attributes.
  → Safebots plugin: hooks on Streams/chat/message to trigger LLM responses.
  → Nothing here. Protocol.Telegram is outbound-only.

Outbound (capability code sends messages, media, performs moderation):
  → Protocol.Telegram.* methods below
```

Token resolved from `Users/apps/telegram/{appId}/token`. Capability code uses `appId` or a `{{credentials:name}}` placeholder — never sees the token directly.

**Generic call:**
```js
await Protocol.Telegram({ appId, method: 'sendMessage', params: { chat_id: 123, text: 'Hi' } });
```

All named methods below are wrappers. Use named methods in capability code — they validate parameters and provide camelCase spec fields.

**Sending messages:**

`sendMessage` — `chatId`, `text`, `parseMode` ('Markdown'|'MarkdownV2'|'HTML'), `replyMarkup`, `disablePreview`, `replyToMessageId`, `messageThreadId` (forum topics)

`sendPhoto` — `photo` (URL or file_id), `caption`, `parseMode`, `replyMarkup`, `replyToMessageId`, `messageThreadId`

`sendVideo` — `video`, `duration`, `width`, `height`, `caption`, `parseMode`

`sendAudio` — `audio` (MP3/M4A, shown in music player), `title`, `performer`, `duration`, `caption`

`sendVoice` — `voice` (OGG/OPUS, shown as voice note), `duration`, `caption`

`sendAnimation` — `animation` (GIF or H.264/MPEG-4 without sound), `duration`, `width`, `height`, `caption`

`sendDocument` — `document`, `caption`, `parseMode`

`sendSticker` — `sticker` (WEBP/TGS/WEBM file_id or URL)

`sendLocation` — `latitude`, `longitude`, `horizontalAccuracy` (metres, 0–1500), `livePeriod` (seconds, for live location)

`sendVenue` — `latitude`, `longitude`, `title`, `address`, `foursquareId`

`sendMediaGroup` — `media` (array of InputMedia objects, 2–10 items; caption only on first)

`sendChatAction` — `action`: `'typing'`|`'upload_photo'`|`'record_video'`|`'upload_video'`|`'record_voice'`|`'upload_voice'`|`'upload_document'`|`'choose_sticker'`|`'find_location'`

```js
// Show typing indicator, then send result
await Protocol.Telegram.sendChatAction({ appId, chatId, action: 'typing' });
var result = await Protocol.LLM({ ... });
await Protocol.Telegram.sendMessage({ appId, chatId, text: result.content });
```

**Message management:**

`editMessageText` — `chatId`+`messageId` or `inlineMessageId`, `text`, `parseMode`, `replyMarkup`

`editMessageCaption` — edit caption of a photo/video/document message

`editMessageReplyMarkup` — update only the inline keyboard without touching content

`deleteMessage` — `chatId`, `messageId` (bot needs delete_messages admin permission in groups)

`forwardMessage` — `chatId` (destination), `fromChatId`, `messageId`

`copyMessage` — like forward but without the "Forwarded from" header; supports optional new `caption`

`pinChatMessage` — `chatId`, `messageId`, `disableNotification`

`unpinChatMessage` — `chatId`, optional `messageId` (omit to unpin most recent)

**Chat info & moderation:**

`getChatMember` — `chatId`, `userId` → ChatMember with `status` field (`'member'`, `'administrator'`, `'kicked'`, etc.)

`getChatAdministrators` — `chatId` → array of admin ChatMember objects

`banChatMember` — `userId`, `untilDate` (Unix timestamp, 0=permanent), `revokeMessages` (delete all messages)

`unbanChatMember` — `userId`, `onlyIfBanned` (default true — no error if not banned)

`restrictChatMember` — `userId`, `permissions` (ChatPermissions object), `untilDate`

```js
// Mute a user for 1 hour
await Protocol.Telegram.restrictChatMember({
    appId, chatId, userId: spammer.userId,
    permissions: { can_send_messages: false, can_send_media_messages: false },
    untilDate: Math.floor(Date.now() / 1000) + 3600
});
```

`createChatInviteLink` — `name`, `expireDate`, `memberLimit`, `createsJoinRequest`

**Bot configuration:**

`setMyCommands` — `commands` [{command, description}], `scope`, `languageCode`

`deleteMyCommands` / `getMyCommands`

**Interactions:**

`answerCallbackQuery` — `callbackQueryId`, `text` (toast), `showAlert`, `url`, `cacheTime`. Must call within 10 seconds.

`answerInlineQuery` — `inlineQueryId`, `results` (array of InlineQueryResult, max 50), `cacheTime`, `isPersonal`, `nextOffset`

`sendInvoice` — `title`, `description`, `payload`, `providerToken` ({{credentials:name}}), `currency`, `prices` [{label, amount}], `photoUrl`, `needName`, `needEmail`, `needPhoneNumber`, `needShippingAddress`

`answerPreCheckoutQuery` — `preCheckoutQueryId`, `ok` {boolean}. If `ok=false`, `errorMessage` required. Must call within 10 seconds.

**Files:**

`getFile` — `fileId` → File object with `file_path` (valid for 1 hour)

`getFileUrl` — `filePath` (from getFile result) → HTTPS download URL. **Pure helper — no API call.** Constructs URL from resolved token.

```js
// Download a photo from an incoming message
var update = ctx.update; // from Telegram/chat stream instructions
var fileId = update.message.photo.slice(-1)[0].file_id; // largest size
var fileInfo = await Protocol.Telegram.getFile({ appId, fileId });
var url = await Protocol.Telegram.getFileUrl({ appId, filePath: fileInfo.result.file_path });
var bytes = await Protocol.HTTP({ method: 'GET', url, timeout: 30000 });
```

**Helper: `suggestionsToKeyboard(suggestions)`**

Converts Safebots suggestions array to Telegram `InlineKeyboardMarkup`. Pure function, no API call.

Button types: `'reply'` and `'action'` → `callback_data` buttons (bot receives `callback_query` update); `'url'` → url button (`https://` only, other schemes rejected). Buttons in rows of 2. `callback_data` truncated to 64 bytes (Telegram limit).

```js
var keyboard = Protocol.Telegram.suggestionsToKeyboard([
    { type: 'reply', label: 'Approve', payload: 'approve' },
    { type: 'reply', label: 'Reject',  payload: 'reject' },
    { type: 'url',   label: 'View doc', url: 'https://example.com/doc/1' }
]);
await Protocol.Telegram.sendMessage({
    appId, chatId, text: 'Please review this proposal.',
    replyMarkup: keyboard
});
```

Config: `Users/apps/telegram/{appId}/token`

---

### Protocol.Web3

EVM blockchain interaction via the Qbix Users_Web3 PHP layer (SWeb3). Routes through `sendToPHP` because SWeb3 is PHP-only.

**Two modes:**

| Mode | When | Returns |
|---|---|---|
| View call | `privateKey` omitted | Decoded return value of contract method |
| Signed transaction | `privateKey` present | Transaction hash |

```
contractABI      {string|array}  Path to .abi.json view, JSON string, or array
contractAddress  {string}        On-chain contract address
methodName       {string}        Contract function to call
params           {array}         Arguments
appId            {string}        Which Users/apps/web3 chain config to use
caching          {boolean|null}  true=cache all, false=never, null=cache truthy only
privateKey       {string}        {{credentials:name}} — triggers signed transaction
transaction      {object}        from, gas, gasPrice, value, nonce
```

**Response:** `{result?, transactionHash?, usage}`

```js
// Read balance (view call)
var bal = await Protocol.Web3({
    contractABI: 'Intercoin/templates/ERC20',
    contractAddress: '0xABC...',
    methodName: 'balanceOf',
    params: [walletAddress]
});

// Transfer tokens (signed transaction)
var tx = await Protocol.Web3({
    contractABI: 'Intercoin/templates/ERC20',
    contractAddress: '0xABC...',
    methodName: 'transfer',
    params: [recipient, amount],
    privateKey: '{{credentials:org_wallet_key}}'
});
```

---

### Protocol.Payment

Unified payment processing, adapter-dispatched. Both adapters return the same normalised shape.

**Adapters:** `'stripe'` (default) | `'web3'`

**Normalised response:** `{transactionId, status, amount, currency, usage, raw}`

`status`: `'succeeded'` | `'pending'` | `'failed'`

`raw`: full provider response (payment_intent object for Stripe, transactionHash for Web3)

**Protocol.Payment.Stripe** — Stripe Payment Intents create+confirm in one call.

```
amount          {number}   Cents (smallest currency unit)
currency        {string}   ISO 4217 e.g. 'usd'
to              {string}   Stripe customer ID 'cus_...'
paymentMethod   {string}   Stripe payment method ID 'pm_...' (saved card/bank)
description     {string}   Optional statement descriptor
idempotencyKey  {string}   Stripe-Idempotency-Key for safe retries
apiKey          {string}   {{credentials:stripe_secret_key}}
metadata        {object}   Key-value pairs stored on the intent
```

Stripe returns 200 on success, 402 for card declines, 400 for bad requests — all with JSON bodies. Errors surface `error.message` and `error.code`.

**Protocol.Payment.Web3** — thin wrapper over `Protocol.Web3`. Maps `to` → `transaction.to`, `amount` → `transaction.value`. All `Protocol.Web3` fields pass through.

```js
var charge = await Protocol.Payment({
    adapter:        'stripe',
    amount:         4999,       // $49.99
    currency:       'usd',
    to:             customer.stripeId,
    paymentMethod:  customer.defaultPaymentMethod,
    idempotencyKey: invoiceId,
    apiKey:         '{{credentials:stripe_secret_key}}'
});
// charge.transactionId  — payment_intent ID
// charge.status         — 'succeeded' | 'pending' | 'failed'
// charge.raw.next_action — present if 3DS required
```

Config: `Safebox/payment/adapter`, `Safebox/stripe/secretKey`.

---

### Protocol.Transcription

Audio/video → text, adapter-dispatched.

**Adapters:** `'assemblyai'` | `'aws'` | `'openai'`

**Actions:**

| Action | Description |
|---|---|
| `transcribe` | Submit job. With `webhook:true`: PHP submits, provider calls back (non-blocking). Without: Node submits directly (OpenAI synchronous; AssemblyAI/AWS return jobId for polling). |
| `poll` | Submit then poll until done. Blocks up to `maxWaitMs`. Good for short clips. |
| `fetch` | Retrieve previously submitted job by `transcriptId`. |

```
adapter       {string}   'assemblyai'|'aws'|'openai'
source        {string}   Publicly accessible audio/video URL
transcriptId  {string}   For action='fetch'
webhook       {boolean}  true → PHP submission + provider callback to /Safebox/webhook
communityId   {string}   Community for tracking stream
diarization   {object}   {max: N} speaker count hint
language      {string}   Language code hint
maxWaitMs     {number}   Poll timeout ms (default 60000)
```

**Response:** `{id, status, text?, words?, streamName?, error?}`

---

### Protocol.Diffusion

Image generation via Stability AI.

```
prompt, negativePrompt  {string}
model                   {string}   Default: config Safebox/diffusion/model
width, height           {number}
steps                   {number}   Default: 30
cfgScale                {number}   Default: 7
apiKey                  {string}   {{credentials:stability_key}}
```

**Response:** `{data: string, format: string, mimeType: string}` — `data` is base64 PNG, safe to store as stream content.

---

### Protocol.Image

Multi-provider image generation and background removal. Routes to AI plugin adapters.

**Actions:** `'generate'` | `'removeBackground'`

**Adapters:** `'openai'` | `'google'` | `'aws'` | `'ideogram'` | `'hotpotai'` | `'removebg'`

```
action    {string}  'generate' | 'removeBackground'
adapter   {string}  Default: config Safebox/image/adapter
prompt    {string}  Text prompt (generate)
image     {string}  Base64 input image (removeBackground / edit)
format    {string}  'png' | 'jpeg' (default 'png')
width, height {number}
apiKey    {string}  {{placeholder}}
```

**Response:** `{data: string, format: string, mimeType: string}` — base64.

---

### Protocol.HTTP

Raw HTTP. Only URLs matching declared `urlPattern` entries in the capability manifest are permitted. SSRF protection blocks private/loopback/link-local addresses regardless of declared urlPattern. Redirects not followed.

```
method   {string}   'GET'|'POST'|'PUT'|'PATCH'|'DELETE'
url      {string}   Must match a declared urlPattern
headers  {object}   Key-value headers
body     {any}      JSON-serialized if object; string passed as-is
params   {object}   Query string parameters (appended to URL)
timeout  {number}   Milliseconds (default: 30000)
```

**Response:** `{status: number, body: string, headers: object}`

---

### Protocol.Files

Read and write files within the workload's upload folder. No network I/O — filesystem only.

```js
await Protocol.Files.write(workloadData, 'output/report.txt', reportText);
var content = await Protocol.Files.read(workloadData, 'input/data.csv');
```

Files stored under `Q/uploads/Streams/{splitPublisherId}/{workloadStreamName}/`. Paths are relative to that root and must not contain `..`.

---

## 13. API Key Management and the Executor Node

Credentials are stored as **`Safebox/credential/{keyName}` streams** (publisherId = communityId), managed by `Safebox_Credentials`. The ciphertext lives in the `Safebox/ciphertext` attribute as an AES-GCM hex blob. The org key that encrypts it is wrapped by a per-community wrapping key derived by the Requester Safebox from its on-disk master secret.

**Two-box security model:**
- **Requester Safebox** (`CredentialsRequester.js`) holds a 32-byte master secret on disk. Derives `communityWrappingKey = HKDF(masterSecret, communityId)` per request and sends it over mTLS to the executor.
- **Executor Safebox** (`Credentials.js`) holds the wrapped org key in `safebox_key_management`. Unwraps it with the wrapping key, decrypts the credential, uses it ephemerally, zeros all key material.
- Neither box alone is sufficient. PHP never decrypts credentials.

**AES-GCM blob format** (stored as hex in `Safebox/ciphertext`): `IV(12) | ciphertext(n) | tag(16)`

**Placeholder substitution grammar:**

| Placeholder | Resolved from |
|---|---|
| `{{credentials:keyName}}` | AES-GCM credential decrypted ephemerally from `Safebox/credential/{keyName}` stream |
| `{{credentials:key\|base64}}` | Decrypted then base64-encoded |
| `{{credentials:key\|hmac_sha256:payload}}` | HMAC-SHA256 of payload using credential value |
| `{{oauth:platform:field}}` | `users_external_from` OAuth token (auto-refreshed) |
| `{{config:path}}` | `Q_Config::get(...)` non-secret value |

**Three capability types:**

| Type | Keys | Single-payer? |
|---|---|---|
| `execute` | Always BYOK — the org's identity on that platform | Never |
| `import` | BYOK day-0 → foundation single-payer later | Eventually |
| `compute` | BYOK day-0 → foundation single-payer later | Eventually |

**Executor types:** `safebox` (Nitro-attested cloud), `browserExtension` (WebAuthn PRF), `mobileApp` (Secure Enclave), `desktop` (OS credential store).

---

## 14. Capability Auto-Generation

When `Streams.fetch` finds no approved capability for `(publisherId, typePrefix)`, Safebox triggers auto-generation via the Safebots dialog pipeline. See Safebots.md for the full generation workflow.

---

## 15. Governance

Safebox has one governance primitive: **an action stream, a policy stream that gates it, and a threshold of OpenClaim-signed votes that resolves it**. Every write that matters — creating a stream, sending a message, mutating an attribute, deploying a capability — flows through this primitive. There is no other write path.

The sections below build up the model bottom-up:

- §15.1 — The primitive itself: action + policy + threshold + signed votes.
- §15.2 — Community policies: how a community declares which actions need governance.
- §15.3 — Judgments: how individual signers automate their own voting decisions while preserving signer attribution.
- §15.4 — Keys as attested claims (forward-looking; not all of this is in the substrate today — see the note at the bottom of §15.4).

The whole governance surface is intentionally small. Tool auditing, capability deployment, key rotation, judgment registration — each one is a `Streams/create` or `Streams/update` action governed by the appropriate policy. No separate code paths.

### 15.1. The Governance Primitive

Every write goes through `Action.propose` (§9). Proposal creates a `Safebox/action` stream published by the community the action affects. The action sits in status `proposed` until governance resolves it.

The substrate's `Safebox_Action::onActionCreated` runs on stream creation:

1. Look up policies in the community's `Safebox/policies` category, filtered by relation `type = actionType`. Among matching approved policies, the most-restrictive voting policy wins (highest `Safebox/approvalThreshold`); deterministic policies run inline and can short-circuit to rejection.

2. If the chosen policy is a voting policy with `approvalThreshold > 0`, status moves to `'held'`. The substrate dispatches notifications to eligible signers via `Streams/message` of type `Safebox/action/voteRequest`, and dispatches matching judgments asynchronously (§15.3).

3. Each manual vote is a POST to `/Safebox/sign` carrying an OpenClaim signed by the voter's private key (ES256 via WebAuthn / Secure Enclave, or EIP-712 via Ethereum wallet). The handler verifies the signature, attributes the vote to the issuer's userId, and stores the full claim in the action stream's `Safebox/approvals` map (or `Safebox/rejections` for reject votes), keyed by userId.

4. After each vote, the handler counts approvals. When `count(approvals) >= M`, status flips to `'approved'` and execution proceeds. When `count(rejections) >= P`, status flips to `'rejected'`. The thresholds M and P are per-policy attributes (`Safebox/approvalThreshold`, `Safebox/rejectionThreshold`) with platform-default fallbacks from `Safebox/governance/{approvalThreshold,rejectionThreshold}` config.

5. If neither threshold is ever crossed, the action stays in `'held'` indefinitely. No timeout. Manual votes or judgment dispatches resolve it.

**Eligibility** is checked at the policy level: any user with the `Safebox/auditor` label (or the action-type-specific `Safebox/auditor/{actionType}` label) granted by the community is eligible to sign. The community itself is implicitly an eligible signer on its own actions — it can vote on its own actions via judgments without any explicit label. The substrate enforces eligibility on every vote submission, so revoking an auditor's label between dispatch and verdict invalidates votes that haven't yet been recorded.

**Vote claims are independently verifiable.** Each claim stores its own key material in `key[]` and signature in `sig[]`. Anyone with read access to the action stream can re-verify the claims without trusting Safebox's database — an external auditor with the action's approvals map can prove who approved without contacting the substrate.

**Self-promotion guard.** Two compound action types — `Streams/create:Safebox/judgment` and `Streams/create:Safebox/policy` — require at least one human (non-judgment) signature even when the threshold is met by judgments alone. Without this, a community judgment could mint more judgments unattended, becoming a self-replicating bypass of M-of-N. Threshold met by judgments alone for these types puts the action in `'held'` with `Safebox/error` set, waiting for a human to sign; once any human signs, the action proceeds.

### 15.2. Community Policies — Action Governance

A community publishes `Safebox/policy` streams under its `Safebox/policies` category. The category stream is auto-created by the create handler on first policy creation. Each policy carries:

```
Safebox/policySubtype       — "voting" | "deterministic" | "tool-backed"
Safebox/approvalThreshold   — for voting: M in M-of-N (0 means auto-approve)
Safebox/rejectionThreshold  — for voting: P in P-of-Q (default: 1)
Safebox/sha256              — for deterministic: hash of policy.js under
                              files/policies/{sha256}.js
Safebox/delay               — optional seconds after approval before execution
Safebox/approved            — "true" once the policy itself passes its
                              own M-of-N review
Safebox/scope               — "platform-default" only on bootstrap policies
                              shipped by trusted seed code
```

Policies are themselves governed objects. Creating a `Safebox/policy` stream is a `Streams/create` action gated by the community's own `Streams/create:Safebox/policy` policy (typically two-of-three to authorize new governance rules). The `0.5` migration seeds a small set of platform-default policies to bootstrap a fresh installation; everything else goes through normal governance.

A policy's create handler forces `Safebox/approved` to `'false'` on creation, regardless of what the caller passes in — except for streams whose publisher is on the `Safebox/trustedSeedPublishers` allowlist and whose `Safebox/scope` is `'platform-default'`. This blocks a privileged caller from minting pre-approved policies via direct `Streams::create`.

**Activation by relation.** A policy doesn't govern an action type until it's related into the community's `Safebox/policies` category with the relation `type` set to the action type:

```php
Streams::relate(
    $communityId,
    $communityId, 'Safebox/policies',
    'Streams/message',                   // ← action type as relation type
    $communityId, 'Safebox/policy/message-approval',
    array('skipAccess' => true)
);
```

The dispatcher queries `Streams::related(communityId, communityId, 'Safebox/policies', true, ['type' => $actionType])` to find applicable policies. Multiple policies can govern the same action type — the most-restrictive voting policy wins among the matched set.

**Threshold semantics.** `approvalThreshold = M` means M signatures from any eligible signers approve. `approvalThreshold = 0` means no signatures required — the action auto-approves immediately. M=0 from a policy stream (rather than from platform-default config) requires the policy stream itself to be `Safebox/approved = 'true'`, since auto-approve policies are themselves governance-defining and need to have been ratified.

**Scope filters via relation extras.** A policy's relation into `Safebox/policies` can carry `extras.scope` to narrow which proposals it covers — e.g. a `Streams/create` policy that applies only when `payload.streamType` is `Safebox/capability`. The matching predicate is partial-equal: every key in scope must be present and equal in the action's payload (or in the proposing task's provenance attributes for `tool`/`workflow`/`step` keys).

### 15.3. Judgments — Automating Individual Vote Decisions

A judgment is a piece of JS code that votes on a specific user's behalf when matching actions arrive. Judgments don't replace M-of-N thresholds — they let one of the M votes come from automation owned by a real signer instead of from a manual click. A 1-of-1 voting policy with the community's own judgment activated means *the community auto-approves*; a 2-of-3 with the same judgment means *the community auto-approves and two more humans need to sign*.

The mechanism is built from three streams and two relation types:

```
<userId>/Safebox/judgment/<slug>              — the JS code reference
<userId>/Safebox/judgments                    — the user's personal registry
<communityId>/Safebox/policy/<name>           — the policy being delegated to

Registry relation:
    FROM <userId>/Safebox/judgment/<slug>
    INTO <userId>/Safebox/judgments
    type     = action type (e.g. "Email.send")
    extras   = { scope: { ... } }

Activation relation:
    FROM <userId>/Safebox/judgment/<slug>
    INTO <communityId>/Safebox/policy/<name>
    type     = "Safebox/judgment"
    weight   = ordering within this signer's chain (lower runs first)
```

The registry relation is the user's own declaration: *I have authored this judgment, and it applies to actions of this type with this scope*. The activation relation is the explicit opt-in to vote on a specific community policy: *I am putting this judgment on the hook to vote for me when actions land at this policy*. Both must agree — registry without activation means the judgment exists but votes nowhere; activation without registry means the dispatcher refuses to run the judgment (defense-in-depth: stale activations don't accidentally vote on unintended action types).

#### Authoring a judgment

The JS lives at `files/judgments/<sha256>.js`. The function takes an `input` object with the action's data and returns one of three shapes:

```js
module.exports = async function(input) {
    // input.action.actionType    — e.g. "Streams/create"
    // input.action.payload       — the proposed action's payload (parsed)
    // input.action.taskStreamName — provenance: which task proposed this
    // input.action.publisherId   — community whose action this is
    // input.action.streamName    — the Safebox/action stream's name
    // input.signerUserId         — user this vote will be cast as
    // input.communityId          — community the action affects

    // Read APIs (cross-tenant guarded — only the signer's and community's streams):
    var task = await Streams.get(input.signerUserId, input.action.taskStreamName);
    var verdict = await Runtime.llm({ model: 'gpt-4.1-mini', messages: [...] });

    return { approve: 'yes', reason: 'matches my pre-approved template' };
    // or:   { approve: 'no',  reason: 'recipient not in allowlist' };
    // or:   null   /* abstain — user can vote manually later */
};
```

Inside the sandbox the judgment has full read access (`Streams.get/fetch/related/getMessages/getParticipants`, `Runtime.llm`, `Cache.get/set`, `Crypto.sha256`, `Runtime.log`). Write APIs (`Action.propose`, `Safebox.yield`) throw loud errors — a judgment cannot propose new work, only return a verdict. Timeout, crash, or non-shape return is treated as abstain (fail-closed: a broken judgment never approves automatically).

#### Registering and activating

The cleanest path is the `Safebox::judgmentRegister` helper:

```php
Safebox::judgmentRegister(
    $userId,                                     // who owns this judgment
    'email-template-check',                      // slug
    'email-template-check.js',                   // codeFile under files/judgments/
    '5d7f...',                                   // sha256 of the file
    'Streams/update',                            // action type
    array('streamType' => 'Streams/text')        // optional scope filter
);
// Creates Safebox/judgment/email-template-check, relates into the user's
// own Safebox/judgments registry. Stream starts with Safebox/approved='false'
// — judgments cannot vote until they pass the community's review.
```

After community review approves the judgment (an `Streams/update` action setting `Safebox/approved` to `'true'`), the user activates it against the community's policy:

```php
Safebox::judgmentActivate(
    $userId,                                     // judgment owner
    'Safebox/judgment/email-template-check',     // the judgment
    $communityId,                                // community whose policy
    'Safebox/policy/text-updates',               // the policy
    1.0                                          // weight (lower = runs first)
);
```

Once activated, the judgment will be dispatched whenever an action lands at that policy AND the registry-relation type matches AND the registry-relation scope matches the action's payload + provenance.

#### How dispatch works

When an action goes to `'held'` status, `dispatchJudgments` runs:

1. Query the chosen policy's incoming `Safebox/judgment` activation relations.
2. For each activation, the `fromPublisherId` is the signer's userId; check eligibility (community OR Safebox/auditor labeled).
3. Look up the user's registry relation (in `<userId>/Safebox/judgments` filtered by judgment stream name) to read the action-type and scope.
4. Filter: registry's relation type must equal the action type. Registry's `extras.scope` must match the action.
5. Group judgments by signer; within each signer, sort by activation weight ascending.
6. Send one `Safebox/judgment/run` message per signer to Node, carrying the ordered chain.

Node then runs the chain in order. The first judgment that returns a definitive verdict (`approve: 'yes'` or `approve: 'no'`) stops the chain for that signer. Abstaining judgments fall through to the next. If all the user's judgments abstain, no vote is cast for them — they can vote manually later.

When a definitive verdict arrives, Node POSTs a substrate-signed envelope to `/Safebox/sign` with `automated: '1'`, `signerUserId: <user>`, and `claim.kind: 'judgment'`. The PHP handler verifies:
- the request came from the substrate-internal Node socket (`Safebox::isNodeRequest`)
- the claim is well-formed with `kind=judgment`, expiry not yet passed
- the action is still in `'held'` status (no late votes on resolved actions)
- the signer is still eligible (community itself, or auditor-labeled)
- the claim's `iss` matches the signerUserId (no cross-attribution)

If all checks pass, the substrate envelope is recorded in the action's approvals map under `signerUserId` exactly as if the user had clicked approve manually. The threshold counter increments. If M is reached and the self-promotion guard doesn't fire (action type isn't `Streams/create:Safebox/{judgment,policy}` or there's already a human signature), the action moves to `'approved'` and executes.

#### Composition with the existing voting flow

Judgments compose cleanly with manual signing:

| Setup | Behavior |
|---|---|
| 1-of-1 policy, community has activated judgment that approves | Community auto-approves all matching actions |
| 1-of-1 policy, community has activated judgment that abstains | Action stays in `'held'`; human admin signs manually |
| 2-of-3 policy, community + 2 humans labeled auditor; community has activated judgment that approves | Community signs automatically; one human still needs to sign |
| 3-of-N policy, judgment activated only on community side | Community contributes one of three needed votes; two humans still required |
| Threshold-2 policy with judgments that REJECT | Single reject suffices to block (matches default `rejectionThreshold = 1`) |

The community's auditor roster is independent of which signers have judgments. Adding a new auditor doesn't grant them any judgment automatically; they sign manually unless they later author and activate their own judgment.

#### Audit trail

Every action stream's `Safebox/approvals` map records the full claim per signer. For automated judgment votes, the claim's `kind: 'judgment'` and `stm.judgmentStreamName` identify which judgment cast the vote. Reading an approved action shows exactly which judgments voted, in what order, with what reason.

Phase 2 of the system (deferred) replaces the substrate-signed envelope with a license-backed user signature: instead of substrate trust attesting "this judgment voted as user X," the user pre-signs an OpenClaim license authorizing the judgment to vote on their behalf within a scope, and the substrate consumes one use of the license per matching action. Wire shape stays the same; what changes is whose key actually signed the recorded claim.

#### Worked example — community auto-approves any registered bot

The platform-Safebots use case: a community has a list of registered bot publishers, and wants any action proposed by a tool owned by one of those publishers to be auto-approved. The judgment owner is the community itself; the threshold is 1-of-1 (only the community needs to sign).

**Step 1.** Author the judgment JS at `files/judgments/<sha256>.js` (compute the sha256 over the file's UTF-8 content). The function reads the action's `taskStreamName`, follows it to the proposing tool, reads the tool's `Safebox/owner`, and checks membership in the community's registered-bots list:

```js
module.exports = async function approveRegisteredBots(input) {
    var action = input.action || {};
    if (!action.taskStreamName) return null;       // can't determine — abstain

    var task = await Streams.get(input.signerUserId, action.taskStreamName);
    if (!task) return null;
    var taskAttrs = task.attributes
        ? (typeof task.attributes === 'string'
           ? JSON.parse(task.attributes) : task.attributes)
        : {};
    var toolName = taskAttrs['Safebox/tool'];
    if (!toolName) return null;

    var tool = await Streams.get(input.signerUserId, toolName);
    if (!tool) return null;
    var toolAttrs = tool.attributes
        ? (typeof tool.attributes === 'string'
           ? JSON.parse(tool.attributes) : tool.attributes)
        : {};
    var toolOwner = toolAttrs['Safebox/owner'];
    if (!toolOwner) return null;

    // Community's allowlist stream — JSON-encoded array in stream content
    var bots = await Streams.get(input.signerUserId, 'Safebots/registered-bots');
    if (!bots || !bots.content) return null;
    var registered;
    try { registered = JSON.parse(bots.content); } catch (e) { return null; }

    if (registered.indexOf(toolOwner) !== -1) {
        return { approve: 'yes',
            reason: 'tool owner ' + toolOwner + ' is a registered bot' };
    }
    return null;   // not registered — abstain (humans decide)
};
```

**Step 2.** Register the judgment as the community itself:

```php
Safebox::judgmentRegister(
    $communityId,                                  // owner = the community
    'approve-registered-bots',                     // slug
    'approve-registered-bots.js',                  // codeFile
    $sha256OfFile,                                 // computed at install
    'Streams/update'                               // action type to auto-vote
    // no scope filter — applies to all Streams/update actions in this community
);
```

This creates `Safebox/judgment/approve-registered-bots` published by the community, with `Safebox/approved: 'false'`. It also creates the registry relation in `<communityId>/Safebox/judgments` with type `Streams/update`. The judgment exists but votes nowhere yet.

**Step 3.** Run the judgment's creation through the community's normal review process (a `Streams/update` action setting `Safebox/approved: 'true'`). For Safebots's first install, the `Safebox/scope: 'platform-default'` exemption shipped in the create handler lets the seed script set `Safebox/approved: 'true'` directly on the judgment without a manual signing step — see §15.2's note on platform-default scope.

**Step 4.** Define the policy that this judgment will vote on. A 1-of-1 voting policy where the community itself is the only signer:

```php
Streams::create($communityId, $communityId, 'Safebox/policy', array(
    'name'  => 'Safebox/policy/Safebots/bot-updates',
    'title' => 'Auto-approve registered bots updating community streams',
    'attributes' => Q::json_encode(array(
        'Safebox/policySubtype'      => 'voting',
        'Safebox/approvalThreshold'  => 1,
        'Safebox/rejectionThreshold' => 1,
        'Safebox/approved'           => 'true',
        'Safebox/scope'              => 'platform-default',
    )),
), array('skipAccess' => true));

// Relate the policy under Streams/update
Streams::relate(
    $communityId,
    $communityId, 'Safebox/policies',
    'Streams/update',
    $communityId, 'Safebox/policy/Safebots/bot-updates',
    array('skipAccess' => true)
);
```

**Step 5.** Activate the judgment against the policy:

```php
Safebox::judgmentActivate(
    $communityId,                                  // judgment owner = community
    'Safebox/judgment/approve-registered-bots',    // the judgment
    $communityId,                                  // policy publisher = community
    'Safebox/policy/Safebots/bot-updates',         // the policy
    1.0                                            // weight
);
```

**Step 6.** Maintain the `Safebots/registered-bots` content stream as bots are activated and deactivated. Each bot's `publisherId` (the userId of the bot owner) goes in or out of the JSON array:

```php
$bots = Streams_Stream::fetchOrCreate(
    $communityId, $communityId, 'Safebots/registered-bots',
    array('type' => 'Streams/text', 'content' => '[]')
);
$list = json_decode($bots->content, true) ?: array();
if (!in_array($newBotPublisherId, $list)) $list[] = $newBotPublisherId;
$bots->content = Q::json_encode($list);
$bots->changed();
```

**Result.** When a registered bot's tool calls `Action.propose('Streams/update', ...)`, the action lands at the policy. The policy is voting/threshold-1, so the action goes to `'held'`. The dispatcher finds one activation (community's judgment), runs it. The judgment reads the proposing tool's owner, checks the registered-bots list, returns `{approve: 'yes'}`. The substrate posts a substrate-signed envelope to `/Safebox/sign` with `signerUserId: $communityId`. Threshold met; action approved; execution proceeds.

When a non-registered bot's tool tries the same proposal, the judgment abstains. The action stays in `'held'` until a human admin signs manually. No silent auto-approval for unauthorized bots.

#### Anatomy of a 2-of-3 with judgment

Same shape, different threshold. The community creates the policy with `approvalThreshold: 2`, labels two human auditors (`Users_Contact` rows with `label: 'Safebox/auditor'`), and activates the judgment:

```
Setup:
  policy.approvalThreshold = 2
  community is one eligible signer (always — implicit)
  alice has Safebox/auditor label
  bob has Safebox/auditor label
  community's "approve-registered-bots" judgment activated against policy

Action arrives from a registered bot:
  status: held
  dispatchJudgments runs:
    eligible = {community, alice, bob}
    activations under this policy = [community's judgment]
    community's judgment scope matches → fire
    judgment returns approve: 'yes'
  /Safebox/sign records community's vote (kind=judgment)
  approvals: 1 of 2 — still held
  
  Alice clicks approve in the UI:
  /Safebox/sign records alice's vote (human OCP signature)
  approvals: 2 of 2 — threshold met
  status: approved → execute
```

Bob's vote was never needed because community + alice met threshold. If alice had been unavailable, the action would have stayed held until either bob signed or alice came back. Threshold-2 governance with one automated signer is structurally a "human-in-the-loop with one auto-vouch" pattern — useful when you want speed when the bot is well-behaved but real human review for anything unusual.

### 15.4. Keys as Attested Claims

(Forward-looking section — parts of this are aspirational.)

User identity in Safebox is conceptually a collection of OpenClaims attesting that particular public keys are authorized to act for a user in a specific role. Each OCP claim stores its own key material in `key[]` and signature in `sig[]`, so a vote stays verifiable forever — even after the voter rotates their key.

Today the substrate uses Qbix's standard user identity for signing. Per-role key-streams (`Safebox/keys/{userId}` published by the community) are a planned extension that hasn't fully shipped. The `Safebox/governance/railLevelSignaturesRequired` config (default 3) is the visible piece of this work — operations affecting the platform's trust root require higher thresholds that cannot be overridden by community configuration.

What IS in the substrate today:
- OCP-format claims for all votes (verifiable independently of Safebox's database)
- ES256 (WebAuthn / Secure Enclave) and EIP-712 (Ethereum wallet) signature support
- Substrate-signed envelopes for judgment-cast votes (Phase 1 of license delegation)
- The `claim.kind` discriminator that distinguishes human OCP signatures from substrate-issued judgment envelopes

What's planned but not yet shipped:
- Per-role key streams with rotation and revocation through normal governance
- License-backed judgment signing (Phase 2): user pre-signs an OCP license, substrate consumes uses per matching action
- Cross-system OCP discovery via well-known URLs

The `0.5` migration's seeded policies cover the bootstrap case (creating a community with working governance from a fresh database). Beyond that, every key-management action would itself be an `Action.propose` against a community policy — the substrate already has the primitive, the schema and helpers around it just haven't been built yet.


## 16. Workspaces

Post-v0.5, `workloadPublisherId` equals `communityId` directly — no `~` separator, no per-workload publisher identity. Streams produced by a workload are published under the community that owns the workload. This simplifies access control (the community's labels govern who can read/write), simplifies billing (the community's Safebux ledger is debited), and matches how users already think about "my org's data."

```
workloadPublisherId  == communityId          e.g. "NYU"
Safebox/workload/{workloadId}                published by communityId
Safebox/action/{actionId}                     published by communityId
Safebox/keys/{userId}                         published by communityId
```

Workspaces are still tracked for governance history — the `safebox_workspaces` table records workspace lineage (`id`, `communityId`, `parentId`). A cascade query loads the most-derived workspace version first, falling back to the ancestor chain. This supports the v2 feature where workspace publishers may diverge from communityId for experimental or forked community configurations; today they coincide.

The older `{orgUserId}~{workspaceUniqueId}` convention is deprecated and does not appear in current workloads.

---

## 17. Safebux Economics

Safebux is Assets plugin credits with `communityId = "Safebox"`. Balances in `Assets/credits/{userId}` streams.

**Price structure under `Assets/prices/{module}`:**

```
compute:  variable by token/image count, model-specific rates (min 100, max 50000)
import:   per-call, provider-specific rates (min 20, max 5000)
task:     50 Safebux (flat)
tool:     20 Safebux (flat)
capability: 10 Safebux (flat)
```

Cache hit: 50% → original payer (commission), 50% → platform.

BYOK capabilities only charge `capability: 10` (infrastructure) — the user bears real API cost directly.

Protocol `usage` objects carry raw component counts (`{token: 1847}`, `{call: 1}`). PHP computes actual cost via `Assets::computeCost($module, $action, $usage, $variant)`. Protocols never contain pricing logic.

---

## 18. Migration Script

The current installer migration is `scripts/Safebox/0.5-Streams.mysql.php`. Later migrations (0.6–0.9) handle incremental schema adjustments:

- **0.5-Streams.mysql.php** — Creates all Safebox tables (see §19) and seeds platform-level streams: the `Safebox` community's `Safebox/policies` category, the initial policy set for platform publisher, the `Safebox/provider/index`, and the `Safebox/keys/Safebox` trust-root stream with founding-auditor claims (see §15.4 bootstrapping).
- **0.6-rename-orgId.mysql.php** — Renames the legacy `orgId` column to `communityId` across all Safebox tables, reflecting the terminology convergence post-v0.5.
- **0.7-rename-At-to-Time.mysql.php** — Normalizes timestamp column names (`createdAt` → `createdTime`, etc.) to match Qbix conventions.
- **0.8-Safebox.mysql.php** — Adds cross-Safebox federation tables (`safebox_execution`, `safebox_seen_jti`) for the intercloud-attestation workflow.
- **0.9-drop-keychain-table.mysql.php** — Drops the unused `safebox_keychain` table (credentials live in `Safebox/credential` streams, not a DB table).
- **0.10-rename-policy-index.mysql.php** — Renames the community-side policy category from `Safebox/policy/index` to `Safebox/policies` so it matches the other plural-category patterns (`Safebox/credentials`, user-side `Safebox/policies`). Updates `streams`, `streams_relatedTo`, and `streams_relatedFrom` rows.

`syncRelations` templates are registered by 0.5 via `Streams::registerRelation` for the following stream types, so that each type's `Safebox/status` attribute automatically maintains a searchable index relation: `Safebox/action`, `Safebox/tool`, `Safebox/capability`, `Safebox/policy`, `Safebox/workload`. This is a Streams-plugin feature — it is not related to Kahn's topological sort, which is the separate DAG-ordering algorithm used by `flushQueue` and `compilePlan`.

---

## 19. Database Schema

| Table | Purpose |
|---|---|
| `streams` | Core Qbix streams (Safebox extends, does not replace) |
| `streams_task` | Extends `streams` for Streams/task rows: `instructions MEDIUMBLOB`, `errors MEDIUMBLOB` — holds the Safebox closure context |
| `safebox_action_queue` | Proposed actions pending governance + flush, with follows-array tracking |
| `safebox_key_management` | Per-community wrapped key material (org keys; future: Lamport, SPHINCS+, delegation keys) |
| `safebox_workspaces` | Workspace lineage (`id`, `communityId`, `parentId`) |
| `safebox_artifact_ref` | Artifact refcount for chat-timeline pruning |
| `safebox_execution` | Cross-Safebox execution status polling (federation) |
| `safebox_seen_jti` | Replay prevention for cross-Safebox requests (JWT-like `jti` tracking) |

Credentials (API keys) are stored as `Safebox/credential` streams, not a DB table. The wrapped community key lives in `safebox_key_management` (the `safebox_keychain` table referenced in earlier drafts was never created and is dropped by migration 0.9 if found).

`Safebox/credential` stream attributes:

```
publisherId   = communityId
streamName    = 'Safebox/credential/{keyName}'

Attributes:
  Safebox/keyName    — machine name (matches stream name tail)
  Safebox/ciphertext — AES-GCM hex blob: IV(12)|ct(n)|tag(16), encrypted with community key
  Safebox/encFormat  — 'orgkey' | 'ecies' (transit, re-wrapped on first use)
  Safebox/usedBy     — JSON array of capability typePrefix strings
```

All credential streams auto-relate to the `Safebox/credentials` category stream (the index) per the `relate` config in `plugin.json`.

---

## 20. PHP Implementation Guide

Key classes and responsibilities:

**`Safebox_Workflow`** — `kickoff()` creates the workload stream and fires `sendToNode`. `validate()` checks step wiring at save time.

**`Safebox_Workload`** — `onPlanCompiled()` reads plan.json and dispatches root tasks. `advanceWorkflow()` runs Kahn's over plan edges on every yield/completion. `checkComplete()` marks workload done when no tasks remain.

**`Safebox_Task`** — `create()` writes full closure into `instructions`, fires `sendToNode`. `onComplete()` flushes action queue, advances workflow. `onYield()` merges outputs into branchContext, advances workflow.

**`Safebox_Action`** — `onActionCreated()` queries applicable policies for the action's type, runs deterministic policies inline (a `reject` short-circuits), and identifies the voting policy whose threshold governs. Votes arrive as `Streams/vote` messages (§15.1) and trigger tally recomputation on each message-save hook. When a threshold crosses, `Safebox/status` flips. `executeNow()` fires on the `approved` transition via the Streams after-save hook — it does not execute in PHP but forwards to Node via `sendToNode('Safebox/action/scheduleApproved')`; Node consults the full follows-DAG from the action queue and dispatches via Kahn as predecessors complete. `flushQueue()` handles deferred actions at task completion, Kahn over the batch, hold-cascade on transitively-held nodes, reject-cascade on abandoned dependents.

**`Safebox_Capability`** — `fetchStream()` creates a stream in `pending` status and triggers the capability via a Streams hook. `findCapability()` looks up by `(publisherId, typePrefix)`. `onComplete()` sets `status=ready` and posts `Streams/changed` so subscribers (including Node-side `Streams.fetch` waiters) unblock.

**`Safebox_EmailWebhook`** — `process($provider, $rawPayload, $communityId, $jobStreamName)` routes to `_parseCFPayload()` or `_parseSESPayload()`. Writes body files and attachments to disk. Creates job stream and Streams/file sub-streams. Called from `Webhook::arrive()` for Cloudflare-Email only; SES goes through the HTTP handler.

**`Safebox_Webhook`** — `find($platform, $externalId)` looks up routing row. `arrive($rawPayload)` sets stream status, posts Streams/changed, sends resume to Node, calls EmailWebhook for CF.

**PHP execution model:** PHP never blocks waiting for Node. All Node work is fire-and-forget via `Q_Utils::sendToNode`. Node calls back via short HTTP POSTs (`Q_Utils::sendToPHP`).

---

## 21. JavaScript Implementation Guide

Browser bootstrap registers Safebox tools via `Q.Tool.define`. Tool state is managed via `Q.Streams` subscriptions for real-time workload status updates. The `Safebox/workload` tool subscribes to `Streams/task/update` messages on the workload stream and updates task status indicators.

---

## 22. Node.js Implementation

### Q.Utils.sendToPHP

```js
Q.Utils.sendToPHP(path, params, options)
// path:    REST resource e.g. 'Safebox/workload/compiled', 'Streams/stream'
// options.method: 'GET' (default) | 'POST' | 'PUT' | 'DELETE'
// Signed with Users.signature() (ECDSA P-256) — PHP verifies with Users::verify()
// Returns Promise resolving to result slot
```

### Q.Streams Node Methods

All wrap `sendToPHP` with correct REST paths:

```js
Q.Streams.getAsync(publisherId, streamName)
Q.Streams.createAsync(publisherId, fields)
Q.Streams.updateAsync(publisherId, streamName, fields)
Q.Streams.postMessageAsync(publisherId, streamName, type, instructions)
Q.Streams.relateAsync(fromPub, fromName, toPub, toName, type)
Q.Streams.relatedAsync(publisherId, streamName, type, ascending, opts)
```

### Socket Event Handlers

```js
Q.Socket.onEvent('Safebox/workload/compile').set(function(data) {
    Orchestrator.compilePlan(data);
}, 'Safebox');

Q.Socket.onEvent('Safebox/capability/run').set(function(data) {
    Orchestrator.runCapability(data);
}, 'Safebox');

Q.Socket.onEvent('Streams/task/run').set(function(data) {
    Orchestrator.runTask(data);
}, 'Safebox');

Q.Socket.onEvent('Safebox/workload/resume').set(function(data) {
    Orchestrator.onWebhookArrived(data);
}, 'Safebox');
```

---

## 23. Node.js Orchestrator

**`compilePlan(data)`**

Fetches all step streams and follows-edges in parallel via `Promise.all`. Runs Kahn's algorithm (validates no cycles, identifies roots). Validates tool existence and input wiring. Writes `plan.json` to `Q/uploads/Streams/{splitId}/Safebox~workload~{id}/plan.json`. POSTs `{roots[]}` back to PHP via `sendToPHP('Safebox/workload/compiled', ..., {method:'POST'})`.

**`runTask(data)`**

Loads tool code from file, verifies SHA-256 hash matches `Safebox/sha256`. Builds sandbox methods object (`streams.get`, `streams.fetch`, `Action.propose`, `Safebox.yield`, etc.). Runs `Q.Sandbox.Runner`. On yield: `sendToPHP('Safebox/yield', ...)` — PHP runs Kahn's and dispatches successors. On complete: persists action queue via `sendToPHP('Safebox/actionQueue', ..., {method:'POST'})`, flushes via `{method:'PUT'}`, reports completion via `sendToPHP('Safebox/task/complete', ...)`.

**`runCapability(data)`**

Loads capability code, verifies hash. Builds Protocol.* methods object. Runs sandbox. Reports `status='ready'` or `status='failed'` back to PHP.

**`Safebox/workload/resume`**

`Webhook::arrive()` sends this event when a provider webhook fires. The Node handler unblocks any `Streams.fetch()` promise awaiting that `streamName`:

```js
case 'Safebox/workload/resume':
    Orchestrator.onWebhookArrived(data); break;
// data: { workloadPublisherId, workloadId, streamPublisherId, streamName, communityId }
```

`onWebhookArrived` resolves the awaiting fetch and lets the workload advance without waiting for the `Streams/changed` polling subscription.

---

## 24. End-to-End Flow Example

**Scenario:** Restaurant uses "Generate Menu Image" workflow.

```
1. POST /Safebox/workflow { workflowId: "generate-menu-image",
     input: { item: "burger", brandColors: ["#c8410a","#f5f0e8"] } }

   PHP: kickoff() → creates workload (status=compiling) → sendToNode compile

2. Node: compilePlan()
   Fetches all steps + follows-edges in parallel
   Kahn's → root: "Safebox/step/findOrCreateImage"
   Writes plan.json → sendToPHP Safebox/workload/compiled {roots}

3. PHP: onPlanCompiled()
   Sets workload status=running
   Creates Streams/task/abc123 with full closure in instructions
   sendToNode Streams/task/run

4. Sandbox runs tool:
   var existing = await Streams.related("com.stability", "index/Streams/image",
     "Safebox/tag/burger", true, { limit: 5 });
   await Safebox.yield({ candidates: existing }); // PHP dispatches any unblocked tasks

   var image = await Streams.fetch("com.stability", "Streams/image/" + promptHash);
   // → capability runs: Protocol.Diffusion({...}) → charges 4200 Safebux
   // → status="ready", Streams/changed fires

   var a1 = await Action.propose("Streams/relate", { ... }); // PHP runs policies

   return { imageStream: { publisherId: "com.stability", streamName: image.name } };

5. Tool completes:
   Node: persist + flush action queue
   PHP: executes approved Streams::relate(), advances workflow
   PHP: checkComplete() → workload status=done → Streams/changed message
   Browser: notified → shows generated image

6. Next request (same prompt, different restaurant):
   Streams.fetch("com.stability", "Streams/image/{samePromptHash}")
   → cache hit: 2100 Safebux (50% of 4200)
   → 1050 credited to original payer (restaurant A)
   → returns immediately
```
---

## 25. Context Assembly and Enrichment

`Safebox_Context` assembles KV-cache-friendly prompts from streams and exposes a pluggable enrichment layer for resolving references and surfacing interactive clarifications before any big-LLM invocation.

### 25.1. Why sort by updatedTime

A rendered prompt is a concatenation of stream renders. If the prefix bytes are stable across turns, the inference engine's KV cache can reuse the attention states for those prefix tokens rather than recomputing them. The simplest way to get a stable prefix: sort ascending by `updatedTime`, so stale streams go first and the tail is whatever just changed. New activity always lands at the end, where it naturally displaces only the most recent cache entries.

No semantic-updated-time column, no stability hints. The `updatedTime` on the `streams` row is sufficient.

Tiebreak by `(publisherId, streamName)` lexicographic so streams with identical `updatedTime` deterministically sort the same way across calls.

### 25.2. Render format

Each stream renders as:

```
⟨stream pub="..." name="..." type="..."⟩
{title, if any}
{content, if any}
{contentRelevant attributes, if configured}
⟨/stream t="..."⟩
```

Headers and trailers use U+27E8 / U+27E9 (mathematical angle brackets) because they tokenize cleanly and don't collide with `<stream>` literals that might appear inside user content. The trailer carries `updatedTime` rather than the header: for append-only stream types, appending a new entry only mutates the bytes at the stream's tail and the trailer — the prefix of the rendered stream stays byte-identical across turns.

Attribute inclusion is controlled per stream type under config path `Safebox/Context/attributes/{type}` — a JSON array of attribute names. By default, nothing is rendered except `title` + `content`. Cosmetic fields (`lastSeenAt`, view counts, UI state) are excluded.

### 25.3. assemble() return

```php
array(
  'text'          => '…concatenated rendered streams…',
  'streams'       => array(array('publisherId'=>'…','streamName'=>'…'), …),
  'hashes'        => array('sha256…', 'sha256…', …),  // hashes[i] = hash of renders 0..i
  'cacheKey'      => 'sha256…',  // hashes[N-2] — the prefix before the tail stream
  'bundleHash'    => 'sha256…',  // hashes[N-1] — the full bundle
  'tokenEstimate' => 1234,
  'truncated'     => false,
)
```

`hashes` is the full ladder so callers can pick any prefix-depth. The inference layer may check `cacheKey` against its own state to decide whether to reuse the prefix attention.

If `options.tokenBudget` is set and the assembled text exceeds it, streams drop from the END (most recent first) — preserving the cache-stable prefix — and `truncated` is `true`.

### 25.4. The enrich() hook chain

`Safebox_Context::enrich($text, $context)` fires seven `Q::event` hooks in order, each accumulating into a shared `$result` passed as the fifth arg (the native `$result` slot):

| Hook | What listeners do |
|---|---|
| `Safebox/Context/enrich/ner` | Named-entity extraction — find references in the text. |
| `Safebox/Context/enrich/resolve` | Disambiguate ambiguous references against the current graph. |
| `Safebox/Context/enrich/expand` | Pull in k-hop neighborhood streams likely to be relevant. |
| `Safebox/Context/enrich/retrieve` | Candidate retrieval via Streams.related / vector / symbolic search. |
| `Safebox/Context/enrich/rerank` | Bounded top-K pruning of accumulated candidates. |
| `Safebox/Context/enrich/clarify` | Produce clarifying questions for the UI when resolution is ambiguous. |
| `Safebox/Context/enrich/suggest` | Proactive follow-up suggestions for the UI. |

Return shape:

```php
array(
  'references'  => array(…typed reference objects…),
  'streams'     => array(array('publisherId'=>'…','streamName'=>'…'), …),
  'questions'   => array(
     array(
       'id'       => 'ref:a1b2c3d4…',
       'question' => 'Which spec did you mean by "the spec"?',
       'type'     => 'single_select',
       'options'  => array(
          array('value' => 'pub|name', 'label' => '…'),
          …
       ),
       'priority' => 'blocking',  // or 'optional'
     ),
     …
  ),
  'suggestions' => array(…),
  'metadata'    => array(…),
)
```

Nothing in `enrich()` spends big-LLM tokens. Listeners doing heavy work (small-LLM inference, vector search, tree-sitter parsing) dispatch to Node-side services via `Q_Utils::sendToNode`.

### 25.5. Interactive clarification

When `enrich()` returns non-empty `questions`, the caller surfaces them as interactive UI elements — buttons, selection lists, drag-to-rank — BEFORE invoking the big LLM. The user's response gets folded back into `$context['answers'][$id]` on the next call, and the default clarify listener suppresses any question whose id appears in `answers`.

`priority: blocking` means the UI must resolve the question before big-LLM invocation. `priority: optional` means the UI may surface it as a dismissible chip; the LLM can proceed with the best-guess resolution.

This is where a lot of token budget gets reclaimed: small-model output surfaces as UI, not as prompt text, which means zero big-LLM tokens are consumed resolving ambiguity that a human can resolve faster with a tap.

### 25.6. Registering a custom listener

Domain plugins (Grokers, etc.) register listeners on the hook chain without ever modifying `Safebox_Context`. Example — a listener that uses a Node-side small NER model:

```php
// Register in your plugin's config/plugin.json under handlersAfterEvent:
// "Safebox/Context/enrich/ner": ["YourPlugin/Context/enrich/ner"]

function YourPlugin_Context_enrich_ner($params, &$result)
{
    $text = $params['text'];
    $response = Q_Utils::sendToNode(array(
        'Q/method' => 'Safebox/capability/ner',
        'text'     => $text,
        'timeout'  => 500  // ms — stay fast
    ));
    foreach ($response['entities'] as $ent) {
        $result['references'][] = array(
            'type'       => $ent['type'],
            'text'       => $ent['surface'],
            'candidates' => $ent['candidates'],  // later resolve listener uses these
            'span'       => $ent['span'],
            'confidence' => $ent['score'],
        );
    }
}
```

Listeners are free to read what earlier hooks in the chain contributed via `$params['prior']` — e.g. the resolve listener reads NER references, the clarify listener reads resolve candidates.

### 25.7. Defaults shipped with the plugin

- `Safebox/Context/enrich/ner` — regex-only NER: `stream://pub/name` URIs, `@userId` mentions, HTTP(S) URLs. No Node dispatch; pure PHP. Handler at `handlers/Safebox/Context/enrich/ner.php`.
- `Safebox/Context/enrich/clarify` — emits a single-select question for each reference with two or more candidates and no prior answer in `$context['answers']`. Handler at `handlers/Safebox/Context/enrich/clarify.php`.

Domain plugins layer richer listeners on top of these — they run after the defaults in registration order, and can prune or augment the accumulated result freely.

### 25.8. Sandbox API

Tools and capabilities reach both methods through the standard sandbox:

```js
var out = await context.assemble([
    { publisherId: "NYU", streamName: "Streams/article/42" },
    { publisherId: "NYU", streamName: "Streams/chat/thread-17" }
], { tokenBudget: 8000 });
// out.text, out.streams, out.cacheKey, out.bundleHash, out.tokenEstimate, out.truncated

var enriched = await context.enrich(userMessage, {
    currentArtifact: { publisherId: "NYU", streamName: "Streams/article/42" },
    user:            { publisherId: "Safebox", streamName: "Users/alice" },
    answers:         { "ref:a1b2c3d4": "NYU|Streams/article/47" }  // prior UI answer
});
// enriched.references, enriched.streams, enriched.questions, enriched.suggestions
```

Both sandbox calls round-trip through `sendToPHP('Safebox/context/assemble'|'Safebox/context/enrich')` so the PHP event system fires all registered listeners.

---

## 26. Protocol.System — Container Governance

Protocol.System is the privileged channel for Docker container management on the Safebox host. Unlike the other protocols (HTTP, SMTP, LLM, Web3, etc.) which run inside the sandbox over per-community credentials, Protocol.System operates on host-level infrastructure shared by every community on the host. That makes it a fundamentally different governance surface.

### 26.1. What Protocol.System is for

Each Safebox host runs a set of Docker containers — local LLM inference, vector search, MariaDB, Redis, application services. Communities running on the host invoke these via capabilities: a capability that talks to a local LLM container at `http://172.20.0.50:8001` doesn't manage the container, it just expects it to be running. The container lifecycle (start, stop, restart, image upgrade, exec) is governed separately, by Protocol.System.

This separation is intentional. Capabilities are sandboxed user code; container management is host-level operation that needs M-of-N admin authorization, not arbitrary capability invocation.

### 26.2. Trust model

Three trust boundaries, in order:

**Safebox governance gate.** PHP-side `Safebox_System::execute` verifies the OpenClaim: M-of-N signatures from authorized signers, signer membership in `Safebox/keys/Safebox` for the per-container required role, jti uniqueness, claim shape sanity. This is the cryptographic boundary — only properly governance-approved operations reach the dispatch step.

**Safebox process boundary.** PHP→Node communication (within one Safebox process) is trusted. A compromised Node process already has every secret PHP would protect with a token, so an in-process verifiedOpToken adds no security; it just adds complexity. The honest model: PHP did the verification, Node executes.

**Infrastructure peer-UID boundary.** The `system-protocol-api` process (Infrastructure layer, separate from Safebox) accepts connections only from the Safebox process UID. An attacker who runs code as some other unprivileged user on the host cannot reach the Docker socket via Protocol.System, even if they somehow hold the HMAC key.

The HMAC on the localhost channel is **not** a third governance layer — it's defense against process-replacement / port-reuse races on localhost. If `system-protocol-api` crashes and something else binds to port 4000, the HMAC verifies that responses came from the API the operator installed.

### 26.3. Per-container governance via `Safebox/system/registry`

Every container Protocol.System touches must have a registry entry on the `Safebox/system/registry` stream (published by `Safebox`). Registry content is JSON:

```json
{
  "safebox-mariadb": {
    "owningCommunityId": "Safebox",
    "governance": {
      "requiredRole":       "Safebox/system-ops-db",
      "requiredSignatures": 4,
      "allowedActions":     ["start", "stop", "status", "restart"]
    }
  },
  "safebox-app-safebox": {
    "owningCommunityId": "Safebox",
    "governance": {
      "requiredRole":       "Safebox/system-ops",
      "requiredSignatures": 2,
      "allowedActions":     ["start", "stop", "status", "restart",
                             "git", "zfs-snapshot", "zfs-rollback",
                             "nginx-config", "nginx-cert"],
      "actionThresholds": {
        "status":       1,
        "zfs-rollback": 4,
        "exec":         4
      },
      "imagePattern":   "^registry\\.example\\.com/safebox/app:[a-z0-9-]+$"
    }
  },
  "community-x-app-server": {
    "owningCommunityId": "community-x",
    "governance": {
      "requiredRole":       "community-x/system-ops",
      "requiredSignatures": 2,
      "allowedActions":     ["start", "stop", "status", "restart", "exec", "pull"]
    }
  }
}
```

Updates to the registry stream go through standard governance (Streams/update with M-of-N from the platform-operator role). The registry is the only place that says "this container exists for governance purposes" — Infrastructure has its own separate allowlist (`/etc/safebox/managed-containers.json`) controlled by the host operator.

**Registry fields:**

- `requiredRole` — which role's keys (in `Safebox/keys/Safebox`) can authorize ops on this container
- `requiredSignatures` — default M-of-N threshold for actions on this container
- `allowedActions` — array of action strings this community has authorized. Defaults to `["status"]` (read-only) for entries that don't specify, so a freshly-provisioned registry entry is harmless until an explicit Streams/update broadens it
- `actionThresholds` (optional) — per-action override of `requiredSignatures`. Lets `status` be 1-of-N (read-only, low risk) while `zfs-rollback` and `exec` require 4-of-N (destructive or highest-privilege). Without per-action overrides, operators are forced to use the most-privileged action's threshold for every operation
- `imagePattern` (optional) — regex that `pull` action's `imageTag` must match. Belt-and-suspenders against Infrastructure's own pattern check; lets governance refuse a `pull` to an unexpected image tag before it even reaches the API

**Action vocabulary.** The action string is opaque to the Safebox governance layer — Safebox verifies signers and dispatches whatever action name is in the claim. The Infrastructure side defines what actions exist. Operators must coordinate with Infrastructure on the action set; sending `action: "frobnicate"` will pass Safebox's syntactic check but will fail at the Infrastructure boundary.

Currently supported actions on the Infrastructure side:

| Action | Purpose | Notes |
|---|---|---|
| `start`, `stop`, `restart` | Container lifecycle | Standard |
| `status` | Container state inspection | Read-only; usually 1-of-N |
| `exec` | Run a command inside running container | Highest privilege; recommend 4-of-N. Cmd is argv array, never a shell string |
| `pull` | Pull a new image tag | Constrained by `imagePattern` if set |
| `git` | Clone or update a Git repo (with submodules) | `commit` must be SHA-1 or SHA-256 hex; `dest`/`url`/`repo` validated against shell metachars |
| `npm` | Update Node packages | `package`, `version` required |
| `composer` | Update PHP packages | `package`, `version` required |
| `zfs-snapshot` | Create ZFS snapshot | `dataset` and `snapshot` validated against shell metachars |
| `zfs-rollback` | Rollback to ZFS snapshot | Destructive; recommend high threshold |
| `nginx-config` | Configure an Nginx site | `domain` validated as DNS-safe |
| `nginx-cert` | Renew or install SSL cert | |
| `model-load`, `model-update` | Manage AI model containers | |

For unknown actions, Safebox does shape-validation (regex on the action string itself) but no payload validation. Infrastructure must reject unknown actions on its end.

### 26.4. The §15.4 keys-as-claims connection

Authority to execute Protocol.System operations comes from `Safebox/keys/Safebox` — the platform trust root stream. Keys attested with the role named in the registry entry (e.g. `Safebox/system-ops-db`) are eligible to sign operations on that container. Threshold is M-of-N over those keys.

Key rotation, revocation, and bootstrap follow §15.4 directly. There is no separate authorized-keys list; system-ops keys live in the same trust root as every other governed key.

For multi-org tenancy: each community has its own role suffix. `community-x/system-ops` keys are attested in `Safebox/keys/Safebox` with `stm.role = "community-x/system-ops"`. The registry entry for `community-x-app-server` requires that role, so only community-X admins authorize it. A community-Y admin cannot operate on community-X's containers because their keys aren't attested for `community-x/system-ops`.

### 26.5. CLI signing flow

Protocol.System handlers are Node-only. The browser never has access to admin private keys. Admin signing happens offline:

```bash
# Admin generates an OpenClaim using their private key
safebox-admin system start community-x-app-server \
    --jti $(uuidgen) \
    --key ~/.safebox/admin1.key

# Output: a JSON OpenClaim with stm + key + sig
# Multiple admins sign sequentially, accumulating signatures:
safebox-admin system start --add-signature \
    --claim claim.json \
    --key ~/.safebox/admin2.key

# Final M-of-N claim is POSTed to the local Safebox via:
curl -X POST http://localhost/Safebox/system/start \
    -H "Content-Type: application/json" \
    -d "$(cat final-claim.json)"
```

The CLI tool (`safebox-admin`, separate package) handles claim canonicalization, signature accumulation, and the POST. Admin private keys never leave the admin's laptop.

### 26.6. The dispatch path end-to-end

```
1. Admin signs OpenClaim offline (CLI tool)
2. CLI POSTs to /Safebox/system/{action} (Node-only handler)
3. Handler validates request shape, calls Safebox_System::executeFromRequest
4. Safebox_System::execute:
   a. Validates claim shape (ocp, stm, jti, key[], sig[])
   b. Loads registry from Safebox/system/registry stream
   c. Looks up container's per-container governance (role, threshold, allowedActions)
   d. Refuses if action not in allowedActions
   e. Loads Safebox/keys/Safebox, extracts keys attested for the required role
      (filtering revoked keys via revocation claims, expired keys via nbf/exp)
   f. Verifies each signature individually via Q_Crypto_OpenClaim::verify
   g. Checks each signer's key matches an authorized OCP key string
   h. Counts distinct authorized signers, refuses if below threshold
   i. Inserts jti into safebox_seen_jti (replay protection)
   j. Writes safebox_system_log row with status='verified'
   k. Sends Safebox/system/dispatch event to Node
   l. Updates row to status='dispatched'
5. Node-side _onSystemDispatch:
   a. Calls Protocol.System(spec)
   b. Protocol.System POSTs to localhost:4000 with HMAC over body
   c. system-protocol-api verifies peer UID == Safebox UID
   d. system-protocol-api verifies request HMAC
   e. system-protocol-api checks container in /etc/safebox/managed-containers.json
   f. system-protocol-api applies exponential backoff
   g. system-protocol-api invokes Docker via dockerode (no shell)
   h. Response signed with HMAC
   i. Protocol.System verifies response HMAC
6. _onSystemDispatch POSTs result to PHP via PUT /Safebox/system/result
7. Safebox_System::recordResult flips audit row to 'completed' or 'failed'
```

Steps 1-4 are governance (Safebox-side, this plugin). Steps 5d-h are Infrastructure-side (separate process — see system-protocol-api Infrastructure spec). Steps 4k, 5a-c, 5i, and 6-7 are the Safebox/Infrastructure handoff.

### 26.7. Audit table — `safebox_system_log`

Created by migration `0.11-add-system-log.mysql.php`; `details` column added by `0.12-add-system-log-details.mysql.php`. Columns:

| | |
|---|---|
| `id` | Auto-increment primary key |
| `jti` | Unique nonce (also recorded in safebox_seen_jti) |
| `container`, `scope`, `action` | From the claim's stm |
| `details` | One-line operator-facing summary, e.g. `clone github.com/Qbix/Platform → /opt/qbix/platform @a1b2c3d4e5f6` for `git`, or `rollback zpool/app-safebox → @before-update-1745880000` for `zfs-rollback`. Generated by `Safebox_System::_actionDetails()` |
| `signers` | JSON array of OCP key strings that successfully signed |
| `signerCount`, `thresholdM` | Verified count and required threshold |
| `claim` | Full OpenClaim, capped at 64 KB |
| `result` | API response, capped at 64 KB |
| `errorMessage` | Generic error message if failed |
| `status` | `'verified'` → `'dispatched'` → `'completed'`/`'failed'` |
| `insertedTime`, `dispatchedTime`, `completedTime` | Lifecycle timestamps |

The audit table is **separate from `safebox_action_queue`** by design. `action_queue` holds *pending governance decisions* awaiting threshold; `system_log` holds *already-executed privileged operations* whose M-of-N check happened up front. Mixing them would conflate these two states and confuse governance UI.

Common operator queries:
```sql
-- Recent operations on a container
SELECT action, details, signerCount, thresholdM, status, completedTime
FROM safebox_system_log
WHERE container = 'safebox-app-safebox'
ORDER BY insertedTime DESC LIMIT 20;

-- All git updates that touched a particular plugin
SELECT container, details, status, completedTime
FROM safebox_system_log
WHERE action = 'git' AND details LIKE '%plugins/Streams%'
ORDER BY insertedTime DESC;

-- Stuck-in-dispatched rows (Node crashed mid-call)
SELECT id, jti, container, action, details, dispatchedTime
FROM safebox_system_log
WHERE status = 'dispatched'
  AND dispatchedTime < NOW() - INTERVAL 5 MINUTE;
```

Rows in `dispatched` state for more than a few minutes indicate a Node crash mid-dispatch — operator manually inspects Docker state and reconciles.

### 26.8. Why this is single-Safebox-per-host

If a community has `exec` capability inside its own containers, the security boundary between communities ceases to be Safebox's governance layer and becomes Linux container isolation. To keep per-community trust boundaries cryptographic rather than depending entirely on container hardening, the recommended deployment is **one Safebox per organization** that runs containers it controls.

The host's AWS account holder (or whoever physically runs the hardware) is the platform operator. They install Safebox+Qbix once per organization that wants a Safebox, and the organization's admins bootstrap their own `Safebox/keys/Safebox` trust root. Cross-Safebox operations happen via OCP federation (existing protocol — see `ExecutionRequest`).

A single Safebox managing many communities' containers is supported by the registry's `owningCommunityId` field, but it requires trust between those communities and significantly tighter container hardening. The default deployment model is one Safebox per community where independent governance is desired.

### 26.9. Configuration

```json
{
  "Safebox": {
    "system": {
      "hmacKeyPath":  "/etc/safebox/system-api.key",
      "apiBaseUrl":   "http://127.0.0.1:4000",
      "apiTimeoutMs": 30000
    }
  }
}
```

The HMAC key file must be readable by the Safebox process (group `safebox-hmac`, mode 0640). If the key is missing or shorter than 32 chars at module load, Protocol.System rejects every call until it's fixed. Fail-closed by design.

---

## 27. Local Model Runners — Protocol.{LLM,Image,Transcription,Diffusion}.Local

Safebox capabilities call inference protocols (`Protocol.LLM`, `Protocol.Image`, `Protocol.Transcription`, `Protocol.Diffusion`) without caring whether the underlying request goes to a SaaS API or a local model-runner container. Each protocol has a `.Local` sub-adapter selected by `adapter: 'local'` or by a `local:` prefix on the model name. The capability-facing API is unchanged.

This chapter covers the local-runner path: what runners must implement, how Safebox discovers and routes among them, how telemetry flows, and how cloud fallback is configured.

### 27.1. Why local runners

Per-token costs from cloud LLM providers add up fast at platform scale. Operators who run their own inference hardware (or rent dedicated GPU instances) want capabilities authored against `Protocol.LLM(...)` to seamlessly hit local models when configured, without rewriting capabilities. Beyond cost, local inference gives operators control over cache scoping (per-tenant prefix caches), batching policy, queue priority, and data residency.

The local-runner contract is documented in `local-runner-INFRASTRUCTURE-SPEC.md` (a separate doc shipped to the Infrastructure team). What follows is the Safebox-side view.

### 27.2. Trust model and transport

Runner containers sit on the same Docker host as Safebox-app containers. The default and preferred transport is **Unix domain sockets via a shared volume**, with TCP-on-Docker-network as a supported fallback for multi-host or migration scenarios.

**Why Unix sockets are the default.** Three concrete safety properties:

1. **No network surface to misconfigure.** A misconfigured firewall, an accidental `-p` flag, a host-network container — none of these can expose a Unix socket. The socket is a filesystem object; only volume-mount and file permissions grant access.
2. **`SO_PEERCRED` gives kernel-verified caller identity.** When a process connects, the kernel reports the connecting UID/GID. Unforgeable. So a runner can refuse connections from anything other than the configured Safebox UID — a real authentication boundary, not just "anyone on the network."
3. **Filesystem permissions are well-understood.** `chmod 0660`, group ownership. Compared to "this port is bound on this network with these other containers and Docker's iptables rules and..." Unix sockets are easier to reason about and harder to misconfigure.

There's also a small efficiency win (3-4× lower latency, 2× higher throughput than TCP loopback) but for inference workloads the speedup is in the noise. Safety is the actual reason.

**Convention:** the shared volume `safebox-sockets` is mounted at `/run/safebox/services/` in both Safebox-app and every runner container. Runners create sockets at `/run/safebox/services/<service-id>.sock`; Safebox connects via the same path. One mount per cluster, regardless of how many runners exist.

**TCP fallback** is supported for multi-host deployments (runners on a different physical box) or transitional cases. The runner's `/v1/capabilities.transport` declares which transport(s) it exposes; Safebox prefers Unix socket when both are available.

**Trust assumptions:**

- Runners do inference work; they have no privileges to modify the host or other containers
- An attacker reaching a runner gets free inference cycles, not host compromise
- Per-request HMAC is **not required** by default — the socket-permission boundary is sufficient. Optional response-side HMAC mode exists for paranoid operators (mostly meaningful for TCP transport; Unix sockets get peer-UID auth for free)

The contract is intentionally lighter than `Protocol.System`'s mutual-HMAC-over-Unix-socket. Inference traffic volumes make per-request HMAC a measurable latency cost; the threat model doesn't justify it.

The full wire-protocol contract that runner containers implement is in `local-runner-INFRASTRUCTURE-SPEC.md` — the document shipped to the Infrastructure team. What follows is the Safebox-side view.

### 27.3. Two governed registry streams

Two streams in the `Safebox` publisher control routing. Both are **created and maintained by operators**, not by the plugin — there is no auto-bootstrap. An operator running this plugin for the first time has no streams; `Safebox_Inference::route` returns null for every model until the operator signs the first `Streams/update` adding a runner and a model.

This is consistent with how `Safebox/system/registry` works: the registry-as-stream model means M-of-N governance is the gate for adding any new runner or model, not just an afterthought.

`Safebox/inference/runners` content (JSON object, runnerId → entry):

```json
{
  "safebox-model-llm-1": {
    "transport": {
      "unixSocket": "/run/safebox/services/safebox-model-llm-1.sock",
      "tcp":        { "host": "safebox-model-llm-1", "port": 8080 }
    },
    "runnerType":   "vllm-0.6.0",
    "supports":     ["chat", "complete", "embed"],
    "loadedModels": ["llama-3.1-70b"],
    "health":       "healthy",
    "capacityHint": "available",
    "lastSeenTime": 1745880000
  }
}
```

The `transport` block can carry `unixSocket`, `tcp`, or both. `Safebox_Inference::route` prefers `unixSocket` when present. Legacy entries with top-level `host`/`port` are still supported for backward compatibility.

`Safebox/inference/models` content (JSON object, logical name → entry):

```json
{
  "llama-3.1-70b": {
    "task":                "chat",
    "physicalId":          "meta-llama/Llama-3.1-70B-Instruct",
    "approvedRunnerTypes": ["vllm-0.6.0"],
    "loadPolicy":          "on-demand",
    "fallback": {
      "protocol": "LLM",
      "adapter":  "openai",
      "model":    "gpt-4o-mini"
    },
    "approvedBy": ["admin1-key", "admin2-key"]
  },
  "sdxl": {
    "task":     "image-generate",
    "fallback": { "protocol": "Image", "adapter": "openai", "model": "dall-e-3" }
  }
}
```

**Governance.** Updates to either stream go through standard M-of-N (`Streams/update` signed by platform-operator role keys). Approving a model = adding it to the models registry. Approving a runner = adding it to the runners registry. **Per-request governance is not in scope** — inference traffic volumes make it impractical; the model entry's presence in the registry is the standing approval.

**Operator first-time setup**, in order:

1. Deploy the runner container(s) per Infrastructure's package, mounted on the `safebox-sockets` shared volume
2. Confirm runner is up: `curl --unix-socket /run/safebox/services/<service-id>.sock http://localhost/v1/capabilities` returns runner info
3. Sign and submit a `Streams/update` to create `Safebox/inference/runners` with one entry per deployed runner
4. Sign and submit a `Streams/update` to create `Safebox/inference/models` with one entry per approved model
5. From this point, capabilities calling `Protocol.LLM({adapter: 'local', model: '<logical-name>', ...})` route through the registry

### 27.4. Routing flow

A capability calls `Protocol.LLM({adapter: 'local', model: 'llama-3.1-70b', messages: [...]})`. Step by step:

1. `Protocol.LLM.Local` (or Image, Transcription, Diffusion) builds a task-specific request body and calls the shared `_callLocalRunner` helper.
2. `_callLocalRunner` invokes `Q.Utils.sendToPHP('Safebox/inference/route', {model, task, tenantId})`.
3. PHP `Safebox_Inference::route()` reads both registry streams (cached in-process for 30 seconds), finds a healthy runner with the model loaded, prefers the one with the freshest `available` capacity hint over `near-saturated` over `saturated`. Returns `{runner, model, fallback}` where `runner` carries either `socketPath` (preferred) or `host`/`port` from the registry's `transport` block.
4. Node HTTP-POSTs to the runner using the canonical Safebox path (`/v1/chat`, `/v1/image/generate`, `/v1/transcribe`, `/v1/diffuse`, etc.) with `X-Safebox-Tenant`, `X-Safebox-Request-Id`, `X-Safebox-Cache-Mode`, and other headers per spec §5. Transport selection (socket vs TCP) is automatic based on the registry entry shape.
5. Runner serves the request. Response body in spec §6 shape (camelCase fields, flat structure — no nested `choices` array). Response telemetry headers per spec §7: `X-Safebox-Request-Id` echo, `X-Safebox-Runner-Id`, `X-Safebox-Model-Id`, `X-Safebox-Cache-Hit`, `X-Safebox-Cache-Tokens-Reused`, `X-Safebox-Queue-Wait-Ms`, `X-Safebox-Compute-Ms`, `X-Safebox-Capacity-Hint`.
6. **Image-data resolution.** For `Protocol.Image.Local` / `Protocol.Diffusion.Local`, runners may inline image bytes (`data: <base64>`) or return a runner-local URL (`url: "/output/abc.png"`). The latter is common for ComfyUI workflows that write to disk. `_fetchRunnerResource` performs a follow-up `GET` over the same socket transport to retrieve the bytes. Non-relative URLs are rejected — runners can only point Safebox at paths on themselves, never at external URLs.
7. Node fire-and-forget POSTs to `Safebox/inference/telemetry` with parsed telemetry. PHP increments Redis counters and updates the runner's `capacityHint` and `lastSeenTime` in the registry.
8. Result returned to capability in the protocol's standard shape (same as `Protocol.LLM.OpenAI` etc. would return), with `telemetry` field carrying request-correlation and timing data.

**Defensive normalization.** Local adapters tolerate runners that emit OpenAI-shape `usage` (snake_case `prompt_tokens`, `total_tokens`) instead of spec-shape (`promptTokens`). The local adapter accepts either and emits spec shape downstream. Runners-fronting-OpenAI are common; this avoids silently zeroed-out token counts.

**Capacity-aware routing without per-request `/v1/capacity` polls.** The runner sets `X-Safebox-Capacity-Hint: available|near-saturated|saturated` on every response. Step 7 writes that hint back into the registry stream. The next routing decision (step 3) reads it and prefers freshest `available` runners. No separate polling process is needed; capacity-state propagates as a side effect of the inference traffic itself.

### 27.5. Backpressure and cloud fallback

When the local path fails:

- **Runner returns `503 Saturated`** — `_callLocalRunner` sees the status, records the failure with `errorClass: 'http-503'`, invokes the fallback descriptor if configured.
- **Runner returns `404 Model not loaded`** — same: try fallback.
- **Network failure / connection refused** — same.
- **Runner returns `500` or other non-transient error** — fails the call (no fallback). Genuine bugs shouldn't be papered over with cloud retries.
- **No runner has the model** — `runner: null` in route descriptor, fallback fires immediately.

The fallback descriptor is per-model in `Safebox/inference/models`:

```json
{ "fallback": { "protocol": "LLM", "adapter": "openai", "model": "gpt-4o-mini" } }
```

When invoked, the local adapter dispatches to `Protocol.LLM.OpenAI` (or `Anthropic`, `Cloudflare` per `adapter` field) with the caller's original spec but the substituted cloud model. Telemetry records the fallback path.

Operators who want **fail-closed-on-local** semantics (e.g., for data-residency reasons) simply omit `fallback` from the model entry. Without it, the local path's failures propagate to the capability as exceptions.

### 27.6. Telemetry

Each inference call writes one row into `safebox_inference_log` (table created by migration 0.13, Db_Row class `Safebox_InferenceLog`). The handler `Safebox/inference/telemetry` accepts a fire-and-forget POST from Node and inserts the row using the standard Qbix `new Safebox_X(); $row->save()` pattern that mirrors `safebox_system_log`.

**Schema:**

| Column | Type | Purpose |
|---|---|---|
| `id` | BIGINT auto | Surrogate key |
| `insertedTime` | TIMESTAMP | When the row was written |
| `tenantId` | VARCHAR(64) | Community / tenant making the call |
| `model` | VARCHAR(128) | Logical model name from the registry |
| `runnerId` | VARCHAR(128) | Runner that served the call |
| `requestId` | VARCHAR(64) | `X-Safebox-Request-Id` echoed by runner |
| `cacheHit` | TINYINT(1) | Whether prefix cache served part of the call |
| `tokensReused` | INT UNSIGNED NULL | Tokens served from cache (NULL if not reported) |
| `promptTokens` | INT UNSIGNED NULL | Tokens in the prompt |
| `completionTokens` | INT UNSIGNED NULL | Tokens generated |
| `queueWaitMs` | INT UNSIGNED NULL | Time queued before dispatch |
| `computeMs` | INT UNSIGNED NULL | GPU+CPU time |
| `errorClass` | VARCHAR(32) NULL | `http-503`, `network`, `timeout`, etc. — or NULL on success |

Indexes: `(tenantId, insertedTime)`, `(model, insertedTime)`, `(runnerId, insertedTime)`, `(requestId)`.

**Why a real DB table.** Three concrete needs:

- **Billing by token.** `SUM(promptTokens + completionTokens)` per `(tenantId, model)` over a billing period is the basis for Safebux charges. APCu counters don't survive PHP-FPM restart and split across hosts; the table doesn't.
- **Cache-hit / Safebux rebate accounting.** When `cacheHit = 1` and `tokensReused > 0`, part of the response was served from KV cache. The economics specify a discount on the charge plus a rebate to whoever originally warmed the cache. That accounting needs the per-call rows, not aggregate counters.
- **Operator visibility.** SQL queries answer "how busy was llama-3.1 yesterday" cleanly:

  ```sql
  SELECT model, COUNT(*) AS calls, SUM(cacheHit) AS hits,
         SUM(promptTokens + completionTokens) AS total_tokens,
         AVG(computeMs) AS avg_ms
  FROM safebox_inference_log
  WHERE tenantId = 'community-x'
    AND insertedTime > NOW() - INTERVAL 1 DAY
  GROUP BY model;
  ```

**Per-call telemetry returned to capability.** Every call to `Protocol.LLM.Local` etc. returns a `telemetry` field on the result:

```js
{
  content:   "...",
  usage:     { promptTokens, completionTokens, totalTokens },
  variant:   "llama-3.1-70b",
  telemetry: {
    requestId:    "<uuid that capability can put in its own logs>",
    runnerId:     "safebox-model-llm-1",
    modelId:      "meta-llama/Llama-3.1-70B-Instruct",
    cacheHit:     true,
    tokensReused: 1234,
    queueWaitMs:  45,
    computeMs:    1820,
    capacityHint: "available"
  }
}
```

The `requestId` is the same value Safebox sent in `X-Safebox-Request-Id` and the runner echoed back per spec §7 — and is the same value persisted in the `requestId` column of `safebox_inference_log`. Capabilities, runner logs, and audit rows all correlate by this id.

**Capacity hints don't go in the table.** Per-runner capacity state ("available", "near-saturated", "saturated") goes into Q_Cache (APCu-backed) where the next routing decision can read it cheaply. Last-write-wins is the right semantic for this and per-host scope is acceptable. See `Safebox_Inference::updateRunnerState()`.

**Retention.** Operators choose. A nightly aggregator can roll detail rows into a per-day summary table and truncate, or keep raw rows for the legal billing window. Not in v1; the schema doesn't preclude it.

**Why not Streams.** A stream-per-request would be hundreds of thousands of streams per day at modest QPS. Streams in Qbix are heavyweight (DB row + access rows + file directory) and have access semantics that don't apply to internal billing trails. The dedicated table is the right shape — same reasoning as `safebox_system_log` for governance audit.

### 27.7. Implementation footprint

- `classes/Safebox/Inference.php` — registry-aware routing, telemetry writes
- `handlers/Safebox/inference/route/post.php` — Node-only routing endpoint
- `handlers/Safebox/inference/telemetry/post.php` — Node-only telemetry sink
- `classes/Safebox/Protocol.js`:
  - `_resolveLocalRoute`, `_postToLocalRunner`, `_extractTelemetry`, `_recordLocalTelemetry`, `_callLocalRunner` — shared helpers
  - `Protocol.LLM.Local` (chat / complete / embed)
  - `Protocol.Image.Local` (generate / edit / removeBackground)
  - `Protocol.Transcription.Local` (transcribe)
  - `Protocol.Diffusion.Local` (diffuse)
- README §27 (this chapter)
- `local-runner-INFRASTRUCTURE-SPEC.md` — separate doc, the contract Infrastructure conforms to

No DB migrations. The registry streams are created on first registry update by an operator.

### 27.8. Configuration

```json
{
  "Safebox": {
    "inference": {
      "registryCacheTtlSec": 30,
      "defaultTimeoutMs":    120000
    }
  }
}
```

Most behavior is governed by the registry streams, not config — operators sign updates to the streams rather than editing config files. Only timeout and cache-TTL knobs live in config.

### 27.9. What this is not

- **Not a service mesh.** Runners speak HTTP; Safebox makes routing decisions in PHP. There's no sidecar, no mTLS, no envoy. Add those at the Docker/Kubernetes layer if you want them; Safebox doesn't need to.
- **Not a model marketplace.** The registry is the operator's curated set, not a public catalog.
- **Not auto-scaling.** When all runners are saturated, the call falls back to cloud (if configured) or fails. Spinning up new runners is an operator-driven Protocol.System action (`model-load` on a new runner instance), not an automatic response to load.
- **Not a request scheduler.** Cross-runner load balancing is a single-pass "pick the freshest available runner with this model"; there's no global queue or admission controller. Each runner manages its own queue per the local-runner spec.

### 27.10. Integration readiness

The Safebox side is integration-ready against runners conforming to the v1.0 wire spec. Specifically:

| Spec section | Safebox-side implementation | Status |
|---|---|---|
| §3.1 Unix-socket transport | `_postToLocalRunner` uses `runner.socketPath` | ✓ ready |
| §3.2 TCP fallback | `_postToLocalRunner` uses `runner.host`/`runner.port` | ✓ ready |
| §3.4 Transport declared in `/v1/capabilities` | Read by operator, written into `Safebox/inference/runners` | ✓ ready (operator action) |
| §4 Endpoint paths | `/v1/chat`, `/v1/complete`, `/v1/embed`, `/v1/image/{generate,edit,remove-bg}`, `/v1/transcribe`, `/v1/diffuse` | ✓ ready |
| §5 Outgoing headers | All `X-Safebox-*` prefixed, request-id sent on every call | ✓ ready |
| §6 Request body shape | camelCase, flat, per-task | ✓ ready |
| §6 Response body shape | Flat, defensive normalization for `usage` (camelCase or snake_case) | ✓ ready |
| §6 Image data | Inline base64 OR runner-local URL (auto-fetched) | ✓ ready |
| §7 Telemetry headers | All parsed (lowercased per Node convention), surfaced on result | ✓ ready |
| §8 Backpressure (`503`/`404`) | Triggers fallback descriptor if configured, else fails | ✓ ready |
| §9 `/v1/capabilities` polling | Operator writes results into runner registry; no Node-side poller | ✓ ready (operator action) |
| §10 Optional response HMAC | Not yet implemented; tracked as future work | — deferred |
| §13 On-demand model load | Via `loadPolicy: "on-demand"` in models registry | ✓ ready |
| §17 Versioning | Runners declare `version` in `/v1/capabilities` | ✓ ready (read but not yet enforced) |

**Operator first-time setup checklist:**

- [ ] Deploy runner container(s) with the `safebox-sockets` shared volume mounted at `/run/safebox/services/`
- [ ] Verify each runner: `curl --unix-socket /run/safebox/services/<id>.sock http://localhost/v1/capabilities`
- [ ] Sign and submit a `Streams/update` creating `Safebox/inference/runners` with one entry per runner, transport block declaring socket path
- [ ] Sign and submit a `Streams/update` creating `Safebox/inference/models` with one entry per approved logical model, including `physicalId`, `task`, and optional `fallback` descriptor
- [ ] Test from a capability: `Protocol.LLM({adapter: 'local', model: '<logical-name>', messages: [...]})` should hit the runner

**Things deliberately not auto-bootstrapped** (consistent with `Safebox/system/registry`):

- The plugin install does not create the inference streams. They are operator-curated, M-of-N-governed; auto-creation would defeat the governance gate.
- There is no auto-discovery of runners. Operators decide what counts as approved infrastructure; Safebox refuses to route to runners not in the signed registry.

**Known limits:**

- Multi-host: not supported via Unix socket. Use TCP transport for multi-host. (Tested for single-host; multi-host is structurally supported but unproven in deployment.)
- Streaming: spec §9 declares `features.streaming` as an optional capability; current Safebox-side adapters don't yet handle streaming responses (they read full body before parsing). Streaming support is future work.
- LoRA / fine-tune routing: model registry currently maps logical → single physical; LoRA-per-call routing not yet supported. Would require an extension to the request body and the model registry.

---

## Appendix: Configuration Reference

```json
{
  "Assets": {
    "credits": { "exchange": { "credits": 1, "USD": 100 } },
    "prices": {
      "Safebox": {
        "compute": {
          "min": 100, "max": 50000,
          "rates": { "token": 0.002, "image": 4000 },
          "variants": {
            "gpt-4o":              { "token": 0.002  },
            "gpt-4o-mini":         { "token": 0.0004 },
            "stable-diffusion-xl": { "image": 4000   },
            "claude-sonnet-4-6":   { "token": 0.003  }
          }
        },
        "import":     { "min": 20, "max": 5000, "rates": { "call": 20 } },
        "task":       50,
        "tool":       20,
        "propose":    0,
        "capability": 10
      },
      "com.twitter":  { "import": { "min": 5,  "max": 100,    "rates": { "call": 5  } } },
      "com.pixabay":  { "import": { "min": 3,  "max": 50,     "rates": { "call": 3  } } },
      "com.openai": {
        "compute": { "min": 50, "max": 100000, "rates": { "token": 0.002 },
          "variants": { "gpt-4o": { "token": 0.002 }, "gpt-4o-mini": { "token": 0.0004 } } }
      }
    },
    "commissions": { "Safebox": { "computed": 0.5, "imported": 0.5 } }
  },
  "Safebox": {
    "governance": {
      "defaultSignaturesRequired":   2,
      "railLevelSignaturesRequired": 3,
      "priceChangeTimelockDays":     7
    },
    "sandbox": {
      "defaultTimeoutMs":    30000,
      "capabilityTimeoutMs": 60000,
      "maxMemoryMB":         256
    },
    "llm": {
      "maxTokens": 4096,
      "models": { "openai": "gpt-4.1-mini", "anthropic": "claude-sonnet-4-6" }
    },
    "cloudflare": {
      "apiToken":  "{{credentials:cf_api_token}}",
      "accountId": "{{credentials:cf_account_id}}",
      "gatewayId": "default"
    },
    "email":   { "adapter": "cloudflare", "aws": { "bucket": "my-ses-inbound-bucket" } },
    "payment": { "adapter": "stripe" },
    "stripe":  { "secretKey": "{{credentials:stripe_secret_key}}" },
    "aws": {
      "accessKey": "{{credentials:aws_access_key}}",
      "secretKey": "{{credentials:aws_secret_key}}",
      "region":    "us-east-1"
    },
    "protocols": {
      "http": { "maxRedirects": 3, "timeoutMs": 10000 },
      "llm":  { "maxTokens": 4096 },
      "smtp": { "from": "noreply@safebox.io" }
    },
    "Context": {
      "attributes": {
        "Streams/article":      ["Streams/article/abstract"],
        "Safebox/action":       ["Safebox/actionType", "Safebox/status"],
        "Safebox/policy":       ["Safebox/actionType", "Safebox/approvalThreshold"]
      }
    }
  }
}
```

*End of Safebox Architecture Reference. Version: 2026-04.*
