# Grokers Plugin — Architecture & Implementation Reference

> **Audience**: LLM implementers and developers building or extending the Grokers
> plugin for Qbix.
> Grokers adds *autonomous codebase comprehension* to any repository — parsing
> source files with tree-sitter, building a dependency graph, and dispatching AI
> agents to analyze each function bottom-up, storing everything as queryable
> Streams and relations.
> This document assumes familiarity with Safebox.md.

---

## Table of Contents

1. [Overview & Philosophy](#1-overview--philosophy)
2. [Plugin File Structure](#2-plugin-file-structure)
3. [Stream Types & Schema](#3-stream-types--schema)
4. [The Index Pass](#4-the-index-pass)
5. [The Symbol Graph](#5-the-symbol-graph)
6. [AST Structural Hash](#6-ast-structural-hash)
7. [The Work Queue — Bottom-Up Scheduling](#7-the-work-queue--bottom-up-scheduling)
8. [Agent Architecture](#8-agent-architecture)
9. [Harness & Tools](#9-harness--tools)
10. [Streams Client](#10-streams-client)
11. [Incremental Re-index — regrok](#11-incremental-re-index--regrok)
12. [Convention Discovery](#12-convention-discovery)
13. [Extern Nodes & Relation Types](#13-extern-nodes--relation-types)
14. [Tester & Orthogonality Validator](#14-tester--orthogonality-validator)
15. [Investigator Agent — Concept Formation](#15-investigator-agent--concept-formation)
16. [Documentation Generator](#16-documentation-generator)
17. [Chatbot — RAG over the Graph](#17-chatbot--rag-over-the-graph)
18. [Parallel Refactoring via Topological Scheduling](#18-parallel-refactoring-via-topological-scheduling)
19. [PHP Plugin Handlers](#19-php-plugin-handlers)
20. [CLI Reference](#20-cli-reference)
21. [Install & Seed](#21-install--seed)
22. [Appendix: Configuration Reference](#22-appendix-configuration-reference)

---

## 1. Overview & Philosophy

Grokers is the **knowledge layer** for the Qbix platform's AI stack. Where
Safebox handles sealed infrastructure — workflow execution, action governance,
stream materialization, billing — Grokers handles the pre-computation of
codebase understanding: parsing every symbol, building the dependency graph,
dispatching LLM agents bottom-up, and storing the results as queryable Streams
relations so every subsequent query is answered in milliseconds, not minutes.

### The core insight — amortize inference, don't repeat it

Today's AI coding tools (Cursor, Claude Code) re-infer the dependency graph on
every query. They read files, extract imports, reason about which symbols
reference which, build a mental model — and discard it at the end of the turn.
Next turn, the same work happens again.

Grokers pays the comprehension cost once. The result is materialized as Streams
and relations. At query time — "what depends on this symbol?", "what config keys
does this module read?" — the answer comes from a single indexed DB query. No
file reads, no LLM inference, no re-traversal of the graph.

For any codebase touched more than a few times, this model is orders of
magnitude cheaper. For systematic changes spanning hundreds of files, it is the
difference between feasible and infeasible.

### Bottom-up guarantee

A symbol is never analyzed until all symbols it calls have already been
comprehended. The Kahn-sort work queue enforces this. When an agent analyzes
`JWTManager::validate`, the summaries, preconditions, and side effects of every
callee are already in the graph. The agent reasons about known knowledge, not
raw source.

### Bots as streams — the substrate does the work

Grokers is a plugin. Every symbol, extern, clue, concept, and doc page is a
Qbix stream. Dependency edges are `streams_related_to` rows. Work queue state
is a `Safebox/status` attribute. Faceted search is `registerRelations` +
`syncRelations`. Real-time push, access control, versioned audit trail —
Qbix Streams handles all of it. Grokers provides the parsing, the AI loop,
and the domain logic on top.

### How it composes with Safebox and Safebots

| Safebox does | Grokers does |
|---|---|
| Workflow/step/task orchestration | Define symbol/extern streams, dispatch analysis |
| `Action.propose` governance pipeline | Submit comprehension writes as proposals when governance required |
| `Streams.fetch` / `Streams.related` sandbox API | Use these in every agent tool call |
| Sandboxed tool execution, sha256 attestation | Reference audited analysis tools from agent runners |
| Billing via community's Safebux balance | Inherit — analysis runs under the community's billing context |

Safebots uses Grokers as its knowledge layer: when a Safebot workflow needs to
understand a codebase before making changes, it calls `Streams.related()` on
the pre-computed Grokers graph rather than reading raw files.

---

## 2. Plugin File Structure

```
grokers/
├── Grokers.php                          # Plugin entry: Q::$Plugin->autoload
├── config/
│   └── plugin.json                      # Stream types, routes, event hooks
├── scripts/Grokers/
│   ├── 0.1-Streams.mysql.php            # Installer: registerRelations, seed streams
│   └── Grokers.js                       # CLI: index, regrok, analyze, test,
│                                        #       conventions, docs, ask, history, status
├── classes/
│   ├── Grokers.js                       # Node entry; Q.Grokers.PUBLISHER getter
│   └── Grokers/
│       ├── agents/
│       │   ├── base.js                  # Tool-use loop, context trim, safeStringify
│       │   ├── analyzer.js              # Comprehension agent
│       │   ├── chatbot.js               # RAG chatbot; exports makeReadOnlyHandlers
│       │   ├── investigator.js          # Clue investigation → concept formation
│       │   └── tester.js                # Test generation agent
│       ├── conventions/
│       │   └── discovery.js             # 4-phase convention auto-discovery
│       ├── docs/
│       │   └── generator.js             # Documentation page generation
│       ├── expander/
│       │   └── index.js                 # File region extraction for agent context
│       ├── harness/
│       │   └── tools.js                 # Tool defs + handlers; makeReadOnlyHandlers;
│       │                                # makeAnalyzerHandlers; EXEC_TOOL_DEFS
│       ├── hints/
│       │   └── loader.js                # Load hints/*.json framework hints files
│       ├── indexer/
│       │   ├── builder.js               # Batch upsert streams + relations; upsertRecords
│       │   ├── graph.js                 # Tarjan SCC + Kahn topological sort
│       │   └── watcher.js               # regrok: mtime/hash incremental re-index
│       ├── parser/
│       │   ├── index.js                 # parseFile, parseRepo (batched parallel)
│       │   ├── record.js                # makeRecord, computeAstHash, ControlTracker
│       │   ├── externs.js               # Extern extraction helpers
│       │   └── languages/               # 11 tree-sitter language visitors:
│       │       ├── php.js               #   php, javascript, typescript, python,
│       │       ├── javascript.js        #   java, go, rust, csharp, swift, c, cpp
│       │       └── ...
│       ├── sandbox/
│       │   └── runner.js                # Q.Sandbox execution wrapper
│       ├── streams/
│       │   └── client.js                # Streams API: CRUD, relate, updateRelationExtra,
│       │                                # logAgentStep, getReadySymbols, BFS traversal
│       ├── swarm/
│       │   └── scheduler.js             # Parallel worker dispatch; analyzerLoop;
│       │                                # testerLoop; investigatorLoop
│       └── validator/
│           └── orthogonality.js         # Mutation testing for test quality scoring
└── handlers/Grokers/
    ├── node_request_helper.php
    ├── after/Q_Plugin_install.php
    └── queue/response.php               # SQL work-queue claim endpoint
```

**Handler registration** via `plugin.json` — one event hook:

```json
{
  "Q": {
    "handlersAfterEvent": {
      "Q/Plugin/install": [["Grokers/after/Q_Plugin_install"]]
    }
  }
}
```

---

## 3. Stream Types & Schema

### Stream type overview

| Type | Publisher | Purpose |
|---|---|---|
| `Grokers/repo` | ownerUserId | One per indexed repository; root of the search index |
| `Grokers/symbol` | ownerUserId | One per function/method/class/arrow; holds comprehension |
| `Grokers/component` | ownerUserId | One per frontend tool (Q.Tool.define etc.); holds defaults JSON |
| `Grokers/extern` | ownerUserId | One per external dependency node (config key, table, endpoint…) |
| `Grokers/test` | ownerUserId | One per generated test; linked to its symbol via `Grokers/covers` |
| `Grokers/clue` | ownerUserId | Uncertain observation accumulating evidence across agents |
| `Grokers/concept` | ownerUserId | Confirmed abstract pattern (hook-system, ORM lifecycle…) |
| `Grokers/doc` | ownerUserId | Generated documentation page for a symbol |
| `Streams/category` | ownerUserId | Search bucket categories (`Grokers/search/symbols` etc.) |

### Attribute namespace discipline

**Every attribute set by Grokers must be prefixed `Grokers/`.** The only
exceptions are Safebox's own attributes (`Safebox/status`, `Safebox/scope`)
and Streams' built-in columns (`title`, `icon`, `content`, `readLevel`,
`writeLevel`, `adminLevel`).

### Platform field taxonomy

| Field | Row type | Column | Indexed? | Purpose |
|---|---|---|---|---|
| `attributes` | `streams_stream` | TEXT | Yes (via syncRelations) | Stream metadata |
| `extra` | `streams_related_to` | VARCHAR(1023) | No | Per-relation metadata |
| `instructions` | `streams_message` | TEXT | No | Per-message payload |

**Extras** use flat camelCase keys (no `Grokers/` prefix) — the relation row
is already scoped to a specific Grokers relation type.

### Key attributes — `Grokers/symbol`

```
# Set during index pass
Grokers/file             relative path from repo root
Grokers/qualName         fully-qualified name e.g. "Users_Vote::beforeSave"
Grokers/fromLine         1-indexed start line (string)
Grokers/toLine           1-indexed end line (string)
Grokers/language         "php"|"javascript"|"typescript"|"python"|"java"|...
Grokers/params           JSON: [[name, default, type, confidence], ...]
Grokers/returnType       inferred or declared return type
Grokers/isLeaf           "1"|"0" — no outgoing calls to internal symbols
Grokers/symbolHash       sha256(source lines for this symbol).slice(0,16)
Grokers/sourceFileHash   sha256(entire file content).slice(0,16)
Grokers/astHash          structural hash — insensitive to identifier names
Grokers/sourceHash       sha256 of trimmed source (legacy compat)
Grokers/control          JSON control-flow map {"a":1,"ca":3}
Grokers/fileRole         role from hints file (e.g. "httpEndpoint", "model")
Grokers/minLevel         minimum grokking level for this file role

# Set by analyzer agent
Grokers/summary          one-sentence plain-language description
Grokers/preconditions    JSON array of strings
Grokers/postconditions   JSON array of strings
Grokers/invariants       JSON array of strings
Grokers/sideEffects      JSON array of strings
Grokers/comprehendedTime Unix ms as string
Grokers/category         optional free-form domain tag

# Rename/move tracking (set by watcher.js)
Grokers/previousQualName previous qualified name before rename
Grokers/previousFile     previous file path before move
Grokers/renameHint       JSON: [{qualName, relFile, score}, ...] candidates

# Work queue
Safebox/status           null=pending, "running", "done", "failed"
Grokers/attempts         integer as string

# Removal tracking
Grokers/status           "removed" when symbol no longer in codebase
Grokers/removedTime      Unix ms as string
```

### Relation types — complete reference

| Relation type | From | To | Meaning |
|---|---|---|---|
| `Grokers/symbol` | symbol or component | repo | Membership — belongs to this repo |
| `Grokers/calls` | symbol | symbol | Direct function call |
| `Grokers/reads` | symbol or component | extern | Reads from this extern |
| `Grokers/writes` | symbol or component | extern | Writes to this extern |
| `Grokers/invokes` | symbol | extern/endpoint | HTTP call to this endpoint |
| `Grokers/implements` | symbol | extern or concept | Implements this extern or concept |
| `Grokers/loadedFrom` | symbol | extern/methodFile | Stub points to implementation file |
| `Grokers/requires` | component | component | Won't activate without this sibling |
| `Grokers/composes` | component | component | Renders this child component |
| `Grokers/listensTo` | symbol or component | extern/event or concept | Subscribes to this event |
| `Grokers/emits` | component | extern/event | Fires this event |
| `Grokers/uses` | component | extern/template | Renders this template |
| `Grokers/covers` | test | symbol | Test exercises this symbol |
| `Grokers/observedIn` | clue | symbol | Clue was noticed while analyzing this symbol |
| `Grokers/corroborates` | clue | clue | Independent observation of same pattern |

**Relation extras** (per-edge metadata, camelCase keys):

| Relation | Extra keys |
|---|---|
| `Grokers/calls` | `line: number` |
| `Grokers/reads` | `line: number`, `certainty: "certain"\|"likely"\|"possible"`, `dynamic: boolean` |
| `Grokers/writes` | `line: number`, `certainty: string` |
| `Grokers/invokes` | `line: number`, `method: string` |
| `Grokers/implements` | `resolvedBy: "static"\|"convention"\|"agent"` |
| `Grokers/covers` | `passingTime: string`, `orthogonalityScore: number` |
| `Grokers/observedIn` | `agentStep: number` |

### Message types

#### On `Grokers/symbol` streams

| Message type | Instructions keys | Meaning |
|---|---|---|
| `Grokers/agentStep` | `step`, `harnessToolName`, `input`, `output` | One agent tool-use step |
| `Grokers/symbol/comprehended` | `summary`, `comprehendedTime` | Comprehension written |
| `Grokers/symbol/invalidated` | `relFile`, `oldHash`, `newHash` | Source changed; re-queued |
| `Grokers/symbol/moved` | `fromFile`, `toFile` | Symbol relocated to new file |
| `Grokers/symbol/renamed` | `fromQualName`, `toQualName`, `astHash`, `confidence` | Automatic rename detected |
| `Grokers/symbol/removed` | `relFile`, `renameCandidates` | Symbol no longer present |
| `Grokers/symbol/staleCall` | `removedQualName`, `renameHint` | This symbol calls a removed symbol |

#### On `Grokers/repo` streams

| Message type | Meaning |
|---|---|
| `Grokers/repo/indexed` | Full index pass completed |
| `Grokers/repo/analyzed` | Analysis pass completed |
| `Grokers/repo/regroked` | Incremental re-index completed |
| `Grokers/file/changed` | One or more symbols changed in a file |

### `registerRelations` — the faceted search index

Called once at install to create `streams_related_from` template rows.
`syncRelations()` then auto-maintains `streams_related_to` attribute-index rows
whenever a stream's attributes change:

```php
// Installer script
Streams_Stream::registerRelations('', 'Grokers/symbol', $app, 'Grokers/search/symbols',
    ['Grokers/language', 'Grokers/type', 'Grokers/isLeaf',
     'Grokers/fileRole', 'Grokers/minLevel']);

Streams_Stream::registerRelations('', 'Grokers/component', $app, 'Grokers/search/components',
    ['Grokers/language', 'Grokers/componentFramework', 'Grokers/isPlugin']);

Streams_Stream::registerRelations('', 'Grokers/extern', $app, 'Grokers/search/externs',
    ['Grokers/language', 'Grokers/externCategory']);

Streams_Stream::registerRelations('', 'Grokers/clue', $app, 'Grokers/search/clues',
    ['Grokers/clueType', 'Grokers/clueStatus', 'Grokers/clueConfidence']);

Streams_Stream::registerRelations('', 'Grokers/concept', $app, 'Grokers/search/concepts',
    ['Grokers/conceptStatus', 'Grokers/conceptDomain']);
```

This makes queries like "find all PHP leaf functions" a single indexed SQL
lookup rather than a table scan.

---

## 4. The Index Pass

`grokers index <repoPath>` performs three things: parse, graph, upsert.

### Parse

`parser/index.js` walks the repo with `fast-glob`, filters binary files (null
byte check), and calls `parseFile(absPath, repoRoot, repoHash)` for each.
`parseFile` dispatches to the correct tree-sitter language visitor, then
immediately computes:

- `sourceFileHash` — sha256 of the entire file content, sliced to 16 hex chars
- `symbolHash` — sha256 of the symbol's source lines (fromLine..toLine)
- `astHash` — structural hash from `computeAstHash` (see §6)

All three hashes are stored on every record and written to the symbol stream by
`builder.js`. They form the baseline for incremental re-indexing.

`parseRepo` parallelizes with batched `Promise.all` in groups of 50 with
`setImmediate` between groups to keep the event loop responsive on large repos.

### Graph

`graph.js` builds the call graph from all records:

1. **Tarjan's SCC** (iterative, explicit work stack — no recursion overflow)
   to detect and mark cycles.
2. **Kahn topological sort** with a `queueSet` for deduplication.
   SCC members whose only unresolved callees are within the same SCC are seeded
   into the queue to break the deadlock.
3. Assigns `cycleId` to all SCC members.

Output: topologically ordered `records[]` ready for bottom-up analysis.

### Upsert

`builder.js` writes to Qbix Streams in batches:

- `batchUpsertStreams` — creates/updates `Grokers/symbol` streams with all
  metadata attributes. Writes `Grokers/symbolHash`, `Grokers/sourceFileHash`,
  `Grokers/astHash`, `Grokers/control`, `Grokers/params`, `Grokers/returnType`,
  `Grokers/indexedTime`.
- `batchUpsertComponents` — creates/updates `Grokers/component` streams.
- `upsertExterns` — creates extern streams from extracted extern references.
- `writeRelations` — writes `Grokers/calls`, `Grokers/reads`, `Grokers/writes`
  etc. in parallel batches of 20 for throughput.

`upsertRecords(records, repoStreamName)` is exported for use by `watcher.js`
when new symbols appear in changed files.

---

## 5. The Symbol Graph

### Grokking levels

| Level | Method | Cost | Purpose |
|---|---|---|---|
| 0 | Static only | Free | Params from type hints, call edges, control flow, all three hashes |
| 1 | Signature LLM (batched) | Cheap | Infer missing types, param names |
| 2 | Full body comprehension | Medium | Summary, preconditions, postconditions, side effects |
| 3 | Cross-symbol enrichment | Medium | Annotate call sites with callee parameter names |
| 4 | Pattern investigation | Variable | Investigate accumulated clues → concept formation |

`Grokers/fileRole` and `Grokers/minLevel` from the hints file determine which
level each symbol needs. Leaf utility functions stay at level 2. Core
orchestration symbols are analyzed at level 3 or 4.

### The work queue SQL

The SQL query in `queue/response.php` IS the work queue — no separate table:

```sql
SELECT rt.fromStreamName
FROM streams_related_to rt
JOIN streams_stream s ON s.publisherId = rt.fromPublisherId
  AND s.name = rt.fromStreamName
WHERE rt.type = 'Grokers/symbol'
  AND rt.toPublisherId = ?
  AND rt.toStreamName = ?    -- the repo stream
  AND JSON_EXTRACT(s.attributes, '$.\"Grokers/comprehendedTime\"') IS NULL
  AND (JSON_EXTRACT(s.attributes, '$.\"Safebox/status\"') IS NULL
       OR JSON_EXTRACT(s.attributes, '$.\"Safebox/status\"') = 'failed')
  AND NOT EXISTS (
    SELECT 1 FROM streams_related_to calls
    JOIN streams_stream cs ON cs.publisherId = calls.toPublisherId
      AND cs.name = calls.toStreamName
    WHERE calls.fromPublisherId = rt.fromPublisherId
      AND calls.fromStreamName = rt.fromStreamName
      AND calls.type = 'Grokers/calls'
      AND JSON_EXTRACT(cs.attributes, '$.\"Grokers/comprehendedTime\"') IS NULL
  )
LIMIT ?
```

`Safebox/status` is the execution lock: set to `"running"` before dispatch,
`"done"` or `"failed"` on completion. The claim uses `rawQuery` with a `NOT
EXISTS` guard so concurrent workers don't double-claim the same symbol.

### Control flow tracking

`record.js` exports `ControlTracker` and `traverseWithControl`. Control type
codes: 1=if, 2=else, 3=for, 4=while, 5=do-while, 6=try/.then, 7=catch/.catch,
8=finally/.finally, 9=closure. JavaScript closures use a `handledClosures` Set
to prevent ctrlMap mutation bugs. Results stored as `Grokers/control` JSON on
the symbol stream.

---

## 6. AST Structural Hash

`computeAstHash(funcNode, paramNames, externalNames)` in `record.js` produces a
16-char hex hash that is:

- **Insensitive to**: identifier names (function name, local variables,
  parameter names)
- **Sensitive to**: control flow structure, external callee names, literal
  values (numbers, operators), parameter count and types

**Algorithm**: walk AST, serialize node types and semantic values into a
canonical string. Local identifiers are replaced with positional placeholders
(`p0`, `p1` for params; `v0`, `v1` for locals). External callee names are
preserved because they are dependencies, not local names.

**How it powers rename detection in `regrok`**:

| Situation | Action |
|---|---|
| Same AST hash, different qualName | **Automatic rename** — comprehension preserved, stream updated in place |
| Same qualName, different file | **Moved** — comprehension cleared (different context), stream updated |
| Not found by name or AST hash | **Removed** — heuristic name-similarity candidates, staleCall on callers |

Only identical AST hash triggers automatic rename. Similarity scoring (method
name prefix match, same class) produces rename *hints* for human review but
never automatic graph updates.

All 11 language parsers call `R.computeAstHash(node, paramNames, calleeNames)`
after `gatherEdges` so every record carries `rec.astHash` from the first index
pass onward.

---

## 7. The Work Queue — Bottom-Up Scheduling

`swarm/scheduler.js` runs three parallel loops:

### analyzerLoop

Polls `queue/response.php` every `POLL_MS` (default 2000ms) for symbols
ready to comprehend. "Ready" means: all callees have `Grokers/comprehendedTime`
set, and the symbol itself is unclaimed.

Dispatches up to `MAX_WORKERS` (default 8) concurrent analysis workers. Each
worker is an independent `AnalyzerAgent` instance with its own Anthropic client.
Workers share no state. The work queue SQL enforces the bottom-up ordering.

When all workers are busy, the loop waits. When the queue is empty and all
workers are idle, the loop signals completion.

### testerLoop

After analysis is ≥80% complete, begins dispatching tester workers for
comprehended symbols that haven't been tested (`Grokers/testRunTime` not set).
Fetches candidate streams in pages of 50; filters in memory.

### investigatorLoop

Background lower-priority loop. Polls `Grokers/search/clues` for high-confidence
open clues. Dispatches investigation agents when the main analysis queue is below
50% utilization.

### Hints loading

The scheduler loads the hints directory once at startup and passes `opts.hints`
to each `analyzeSymbol` call. The hints file provides `fileRole` mappings and
`minLevel` thresholds, so the analyzer knows how deeply to investigate each
symbol without configuration per-symbol.

---

## 8. Agent Architecture

### base.js — the tool-use loop

All agents inherit from a shared loop in `base.js`:

```
1. Build system prompt + initial user message
2. Call Anthropic API (claude-opus-4-6)
3. For each tool_use block in the response:
   a. Dispatch to the handler
   b. Serialize result with safeStringify (catches circular refs)
   c. Log as Grokers/agentStep message on the symbol stream
   d. Push tool_result into messages
4. If stop_reason = end_turn: done
5. If stop_reason = tool_use: loop
6. If maxSteps reached (default 20): mark failed, throw
```

**Context window trim** at turn boundaries only. The trim keeps the initial
user message plus the most recent N complete turn *pairs* (assistant + user),
always sliced at an even count, with a role check to avoid orphaning
tool_use/tool_result pairs. The Anthropic API rejects conversations with
orphaned tool blocks.

**`safeStringify`**: wraps `JSON.stringify` in a try/catch. Returns
`{"error": "not serializable: ..."}` on failure rather than crashing the
entire agent loop.

### analyzer.js

Receives: symbol source hunk, callee one-line summaries, known extern relations,
relevant concepts from the ontology.

Must call `memory_set` before finishing. After `runLoop` returns, the agent
verifies `Grokers/comprehendedTime` was set — if not, marks the symbol `failed`
and throws.

Callee summaries are capped at 10 callees × 150 chars each to bound prompt
size.

### tester.js

Uses `Tools.makeReadOnlyHandlers(repoRoot, repoStreamName)` — the full set of
read-only tools — plus `sandbox_run` and `store_test`. Explicitly does **not**
use `makeAnalyzerHandlers` to prevent the tester from accidentally calling
`memory_set` and overwriting a symbol's comprehension.

Language fallback is `'unknown'` (not `'php'`) so non-PHP symbols get the
correct test framework.

### investigator.js

Dispatched when a `Grokers/clue` accumulates ≥3 evidence entries. Reads all
evidence via `streams_related`, fetches source of 2–3 representative symbols,
then calls `streams_create` to produce a `Grokers/concept` stream, `streams_relate`
to wire implementing symbols, `ontology_propose` to post to the ontology stream,
and `investigation_conclude` to close the clue.

`applyVerdict`: verdict `'explained'` → sets clueStatus `'explained'`;
`'promoted'` → creates concept stream, posts `Grokers/ontology/confirmed`;
`'dismissed'` → sets clueStatus `'dismissed'` with reason.

### chatbot.js

Read-only tool-use loop with no write tools. Exports `makeReadOnlyHandlers`
so tester.js and other read-only agents can use it. The `streams_related`
handler uses `fetchRelated(streamName, undefined, ...)` when no `relationType`
is specified, which omits the type field from the PHP request and returns all
relation types.

---

## 9. Harness & Tools

`harness/tools.js` defines three exported tool sets:

```
READ_TOOL_DEFS     — context_expand, memory_get, externs_get, concepts_get,
                     streams_fetch, streams_related
WRITE_TOOL_DEFS    — memory_set, streams_create, streams_relate, streams_unrelate,
                     clue_write, ontology_propose, investigation_conclude
EXEC_TOOL_DEFS     — sandbox_run, sandbox_run_tests
```

All three arrays are exported. `EXEC_TOOL_DEFS` was missing from exports in an
early version, causing `tester.js` tool setup to silently fail.

### makeAnalyzerHandlers(symbolStreamName, repoRoot, repoStreamName)

Full handler set for analysis agents: all READ + all WRITE. Includes `memory_set`
which writes comprehension to the named symbol stream.

### makeReadOnlyHandlers(repoRoot, repoStreamName)

Defined in `tools.js` (canonical) and exported from both `tools.js` and
`chatbot.js`. Built by filtering `makeAnalyzerHandlers('')` down to only
`READ_TOOL_DEFS` entries. The `symbolStreamName=''` argument is harmless for
read tools.

Tester and chatbot agents use this to guarantee they cannot overwrite
comprehension.

### Key handler notes

**`context_expand`**: Path traversal guard — `path.resolve()` check ensures
the requested file is within `repoRoot`. Returns `{error: 'Path traversal
rejected'}` on violation rather than reading `/etc/passwd`.

**`memory_set`**: Guards empty summary. After writing, `applyComprehensionOutput`
writes all comprehension attributes, creates ExternAnnotation relations, emits
clues for uncertain annotations, posts `Grokers/symbol/comprehended`.

**`externs_get`**: Fetches 5 extern relation types in parallel via `Promise.all`
(reads, writes, invokes, implements, loadedFrom).

**`streams_related`**: Omits `type` field from PHP request when `relationType`
is undefined, returning all relation types.

**`clue_write`**: Idempotent — fetches existing clue by stream name (deterministic
sha256 of repoHash+patternText), appends evidence if found, creates new stream
if not. Escalates confidence at 3 and 5 evidence entries.

**`logAgentStep`**: Truncates `output` to 800 chars preview to stay within the
1023-byte `instructions` column limit.

---

## 10. Streams Client

`streams/client.js` wraps `Q.Utils.sendToPHP` with Grokers-domain methods:

```
createStream(type, streamName, attrs, content)
updateStream(streamName, attrs)         — null value = delete attribute
fetchStream(streamName)
batchFetchStreams(streamNames[])
relate(from, to, type, weight, extra)
unrelate(from, to, type)
updateRelation(from, to, type, weight?, extra?)
updateRelationExtra(from, to, type, extra)   — partial merge via setExtra()
fetchRelated(streamName, relationType?, isCategory, limit, offset)
fetchRelatedByTypePrefix(streamName, prefix, isCategory, limit)
postMessage(streamName, type, content, instructions)
logAgentStep(symbolStreamName, step, toolName, input, output)
getReadySymbols(repoStreamName, limit)
claimSymbol(symbolStreamName)
getHighConfidenceClues(repoStreamName)
getTransitiveDependents(symbolStreamName, maxDepth, maxNodes)
```

### `updateRelationExtra` — no unrelate+relate needed

`Streams_RelatedTo` has PHP ORM methods `setExtra($key, $value)` and
`clearExtra($key)` that do a read-modify-write on the `extra` JSON blob and
save in one `UPDATE`. The `Streams/related` PUT handler accepts an `extra` field
for this in-place partial merge.

`updateRelationExtra` sends: `{ ..., extra: JSON.stringify(extraObject) }` via
PUT. This avoids the orphan risk of unrelate+relate (if relate fails after
unrelate, the relation is permanently gone).

`orthogonality.js` uses `updateRelationExtra` to update `passingTime` and
`orthogonalityScore` on `Grokers/covers` relations.

### `getTransitiveDependents`

BFS over `Grokers/calls` relations in reverse (isCategory=true). Bounded by
`maxDepth=5` and `maxNodes=200`. Returns the full set of symbols that depend
on a given symbol, in topological order, for use by the parallel refactoring
scheduler.

---

## 11. Incremental Re-index — regrok

`grokers regrok <repoPath> [--dry-run]`

The `regrok` command detects changed files by content hash and reconciles the
symbol graph without a full re-index. Key design decisions:

### Algorithm

1. **Scan filesystem**: sha256(fileContent) for each source file.
2. **Load known symbols from DB**: reads `Grokers/sourceFileHash` (file-level)
   and `Grokers/symbolHash` (symbol-level) from all symbol streams for this repo.
   Files whose hash matches the stored hash are skipped entirely — zero parse cost.
3. **Re-parse all changed/added files upfront**: builds `allNewByQual` and
   `allNewByAstHash` across ALL changed files before any reconciliation starts.
   This is what makes cross-file move and rename detection work — symbols moved
   from file A to file B appear in `allNewByAstHash` even when processing file A.
4. **Per changed file — three-tier symbol reconciliation**:
   - **Priority 1** (same qualName, different file): move — update stream, post moved
   - **Priority 2** (same astHash, different qualName): automatic rename —
     keep `comprehendedTime`, update qualName, post renamed
   - **Priority 3** (not found): removed — heuristic name-similarity candidates,
     post staleCall on all callers
5. **New symbols** (in new parse but not in old): `builder.upsertRecords()`
6. **Post messages**: `Grokers/file/changed` only when something actually changed;
   `Grokers/repo/regroked` with stats.

### Why no Graph.build for new symbols

`Graph.build(toUpsert)` only resolves calls against records in the current batch.
New symbols calling existing repo symbols — not in `toUpsert` — would have their
edges silently dropped. Instead, new symbols are upserted as bare streams.
The work queue schedules them for LLM analysis; the analyzer discovers externs
dynamically.

### Stale doc marking

When a symbol's content changes (invalidation branch), `watcher.js` computes
the corresponding doc stream name and calls `updateStream(docStreamName,
{'Grokers/isStale': '1'})`. This makes `grokers docs --stale-only` actually
work — it regenerates only docs whose symbols changed.

### File content cache

`_fileContentCache` in `watcher.js` avoids re-reading files for symbol hash
computation. One read per file per regrok pass.

---

## 12. Convention Discovery

`grokers conventions <repoPath>` populates `.grokers/conventions.json` in four
phases. Everything in the 11 language visitors (config externs, hook patterns,
ORM class-to-table mappings, component registration) was itself originally
discovered by running this pipeline and confirming the results.

### Phase 1 — File structure → fileRoles

Directory tree → LLM → `fileRole` entries with glob patterns. Catches
`handlers/`, `views/`, `migrations/`, `pages/api/`, `models/`, `tests/`.

### Phase 2 — High-frequency literal string first-args → protocol candidates

Tree-sitter scans every call expression. Any `(function, firstStringArg)` pair
appearing 5+ times is a protocol candidate. The LLM is shown representative
call sites and infers what the function and argument represent.

This reliably finds `Q.Tool.define`, `Q.page`, `Q.Template.set`, `Q.req`,
`Q_Config::get`, `add_action`, `do_action`, `@app.route`, etc.

### Phase 3 — Hierarchical array first-args → extern candidates

Same scan, filtered to calls where the first argument is an array of string
literals. Detects `Q_Config::get(['A','B','C'])` patterns. Tracks
post-assignment subscript chains.

### Phase 4 — Author review and promotion

All entries start as `confirmed: false`. Authors confirm, fix, add missed
patterns, write `promptFragments`. Subsequent re-runs add new inferred candidates
without touching confirmed entries.

### Visitor generation

The language visitors' extraction rules are generated dynamically from confirmed
`conventions.json` entries. The Qbix bootstrap conventions ship as a
pre-confirmed `conventions.json` fragment with the plugin. Any other framework's
conventions are produced by running this pipeline.

---

## 13. Extern Nodes & Relation Types

Externs represent cross-cutting dependencies: config values, database tables,
HTTP endpoints, i18n strings, event channels, templates. Each is a
`Grokers/extern` stream. Externs participate in the same dependency graph as
symbols.

### Extern stream naming

```
Grokers/extern/{category}/{normalized-path}
```

Examples:
```
Grokers/extern/config/Streams.types.*.edit
Grokers/extern/text/Streams.content.newMessage.body
Grokers/extern/table/stream
Grokers/extern/endpoint/Streams/stream/put
Grokers/extern/hook/Streams.Stream.save.Streams_chat
Grokers/extern/event/Streams.chat.onRefresh
Grokers/extern/methodFile/Q.Data.verify
Grokers/extern/env/APP_ENV
```

### Extern categories

`config`, `text`, `env`, `table`, `cache`, `endpoint`, `socketEvent`, `event`,
`eventFactory`, `template`, `messageType`, `pageUri`, `hook`, `filter`, `signal`,
`toolFile`, `methodFile`, `componentMethod`, `componentState`, `helper`,
`ormRelation`, `module`, `stylesheet`, `syncRelationsTemplate`

### The two-sided bridge pattern

The HTTP endpoint bridge — and every similar cross-language or lazy-load bridge —
works the same way: one side of the codebase says "I will call X" (`Grokers/invokes`
or `Grokers/loadedFrom`), the other side says "I implement X" (`Grokers/implements`).
Both point at the same extern stream. The bridge closes without runtime execution.

```
JS:   Q.req('Streams/stream', ..., {method:'put'})
      → Grokers/invokes → Grokers/extern/endpoint/Streams/stream/put

PHP:  handlers/Streams/stream/put.php
      → Grokers/implements → Grokers/extern/endpoint/Streams/stream/put
```

The `Q.Method` pattern works identically:
```
Stub: Q.Method.define({verify: new Q.Method()}, "{{Q}}/js/methods/Q/Data", ...)
      → Grokers/loadedFrom → Grokers/extern/methodFile/Q.Data.verify

Impl: plugins/Q/js/methods/Q/Data/verify.js → Q.exports(fn)
      → Grokers/implements → Grokers/extern/methodFile/Q.Data.verify
```

### Relation certainty

```
"certain"  — literal string in AST, fully static
"likely"   — one-step dataflow (e.g. $var = fn(); $var['key'])
"possible" — LLM inference, dynamic dispatch, or indirect path
```

Only `certain` relations drive code generation. `likely` and `possible` are
shown in analysis and seed clue streams for investigation.

---

## 14. Tester & Orthogonality Validator

### Tester agent

1. Reads symbol comprehension via `memory_get`.
2. Generates test code targeting each precondition and postcondition.
3. Runs each test via `sandbox_run` to confirm it passes against real code.
4. Calls `streams_create` to create `Grokers/test` streams.
5. Calls `streams_relate` to create `Grokers/covers` relations.

Only uses `makeReadOnlyHandlers` + `sandbox_run` + `store_test` — never
`makeAnalyzerHandlers`. This prevents the tester from calling `memory_set`
and corrupting the symbol's comprehension.

### Orthogonality validator

After tests are stored and confirmed passing:

1. Load source lines for the symbol.
2. Generate mutations: flip comparisons, negate returns, remove conditionals,
   change numeric literals.
3. For each mutation: run all tests against the mutant.
4. Score: 0.7 if the target test fails + 0.3 if all others pass.
5. `updateRelationExtra(testName, symbolStreamName, 'Grokers/covers',
   { passingTime: ..., orthogonalityScore: score })` — in-place update via
   `setExtra`, no orphan risk.
6. Tests with score < 0.5 get `Grokers/test/lowOrthogonality` message and
   are re-queued.

**`Q.Sandbox` guard**: if `Q.Sandbox` is unavailable (Safebox not loaded),
returns `{ pass: true, skipped: true }` — neutral, doesn't penalize tests
that can't run.

---

## 15. Investigator Agent — Concept Formation

Dispatched when a `Grokers/clue` reaches `Grokers/clueConfidence = 'high'`
(≥5 evidence entries). Mission: examine accumulated evidence and either promote
a concept or dismiss the clue.

### Tool set (read-only + promote/dismiss)

`streams_fetch`, `streams_related`, `context_expand`, `memory_get`,
`streams_create`, `streams_relate`, `clue_write`, `ontology_propose`,
`investigation_conclude`

### Verdicts

| Verdict | Action |
|---|---|
| `"promoted"` | Create `Grokers/concept` stream, wire implementing symbols, post `Grokers/ontology/confirmed`, set clue status `"promoted"` |
| `"explained"` | Point to existing concept that explains the pattern; set clue status `"explained"` |
| `"convention"` | Pattern belongs in `conventions.json`; post to ontology stream for author review |
| `"dismissed"` | False positive; set clue status `"dismissed"` with reason |

### How concepts accumulate

Once a concept exists (e.g. `Grokers/concept/{hash}/hook-system`), subsequent
analysis agents see it in their system prompt under "CONFIRMED CONCEPTS". When
they encounter the pattern again (`Q::event(...)`, `add_action(...)`) they
immediately annotate with `Grokers/listensTo` → concept stream, without needing
to re-derive the pattern from scratch. The graph vocabulary grows with every
investigation.

---

## 16. Documentation Generator

`grokers docs <repoPath> [--stale-only]`

### generateDocs

Paginates all `Grokers/symbol` streams for the repo (500 per page) — not limited
to the first 500. For each symbol, calls `assemblePage`.

### assemblePage

Fetches 6 relation types in parallel via `Promise.all`:

```javascript
var fetches = await Promise.all([
    Streams.fetchRelated(symbolStreamName, 'Grokers/reads',      false, 30),
    Streams.fetchRelated(symbolStreamName, 'Grokers/writes',     false, 30),
    Streams.fetchRelated(symbolStreamName, 'Grokers/calls',      false, 30),
    Streams.fetchRelated(symbolStreamName, 'Grokers/calls',      true,  30),  // calledBy
    Streams.fetchRelated(symbolStreamName, 'Grokers/covers',     true,  20),
    Streams.fetchRelated(symbolStreamName, 'Grokers/implements', false, 10)
]);
```

Assembles a Markdown page with: summary, contracts (pre/post/invariants), side
effects, reads/writes, callers/callees, tests with orthogonality scores.

### Stale-only mode

`--stale-only` skips symbols where the doc stream has `Grokers/isStale !== '1'`.
`watcher.js` sets `Grokers/isStale = '1'` on doc streams when their symbol's
content changes. Without this, `--stale-only` would always skip everything.

### Doc stream

```
type:       Grokers/doc
streamName: Grokers/doc/{sha256(repoStreamName + ':' + symbolStreamName).slice(0,16)}
attributes: Grokers/symbolStreamName, Grokers/generatedTime, Grokers/isStale
content:    full Markdown page
```

---

## 17. Chatbot — RAG over the Graph

`grokers ask <repoPath> "<question>"`

The chatbot is a read-only tool-use agent. It has access to all READ tools but
no write tools. It plans queries, executes graph traversals, assembles precise
context, and answers.

### Why structured retrieval beats embeddings for code

Embedding-based RAG retrieves semantically similar chunks. For code questions it
cannot answer: "what config keys does this module read?" or "what would break if
I rename this function?" Both require traversing `Grokers/reads` and `Grokers/calls`
relations — explicit graph edges, not cosine similarity.

### Query pipeline

For "What happens when a Streams message is posted?":

1. Find symbol streams matching "postMessages" via `streams_related`
2. Fetch comprehensions via `memory_get`
3. Fetch extern relations via `externs_get`
4. For each hook extern: `streams_fetch` to get title and description
5. Fetch concept membership via `streams_related` (Grokers/listensTo)
6. Assemble structured context and answer

### Chatbot system prompt

```
You are a code assistant with access to a structured knowledge graph.
Do NOT guess. Query the graph for precise answers.
Before answering:
  1. Identify which symbols, externs, or concepts are relevant
  2. Query the graph using the available tools
  3. Assemble a precise answer from the graph data

If the graph does not have enough information, say so.
Do NOT synthesize from general knowledge.
```

---

## 18. Parallel Refactoring via Topological Scheduling

The pre-computed call graph enables a fundamentally different refactoring model:
changing code in parallel at a speed that is structurally impossible for
sequential AI tools.

### The algorithm

Given a change to symbol X:

```
1. Query:  getTransitiveDependents(X)        → full set of affected symbols
2. Sort:   kahnSort(dependents)               → callees first, callers last
3. Group:  groupByDepth(topoOrder)            → [{A,B}, {C,D}, {E}, {F}]

4. For each level in order:
   dispatch all symbols in this level in parallel
   each worker receives:
     • the original change description
     • the symbol's source
     • comprehended summaries of its already-updated callees
     • the proposed changes already written to those callees
   each worker proposes: Action.propose(edit for this symbol)

5. All proposals → governance review → batch apply
```

### Complexity

- Sequential (Cursor/Claude Code): **O(N)** LLM calls
- Topological (Grokers): **O(depth)** waves, typically **O(log N)**

For 200 affected symbols across 6 dependency levels: 200 sequential calls vs.
6 waves of ~33 parallel workers. Wall-clock time drops proportionally.

### Why each worker stays small and accurate

Each worker is scoped to a single symbol. It receives only its symbol's source
and the already-updated summaries of its direct callees. Context windows stay
small regardless of the total refactor size. A worker at depth 2 reads the *new*
comprehension of its depth-1 callees — it reasons about the updated API, not
the old one.

### Governance at refactoring scale

Each worker produces an `Action.propose` rather than a direct edit. The full set
of proposals from a wave can be reviewed as a coherent batch. If one proposal is
wrong, that symbol's wave can be re-run without touching others. ZFS snapshots
provide rollback for the entire plan.

---

## 19. PHP Plugin Handlers

### `handlers/Grokers/after/Q_Plugin_install.php`

Bootstrap entry point. Calls the installer script which runs `registerRelations`
and seeds the platform's search bucket category streams.

### `handlers/Grokers/queue/response.php`

Handles `POST /Grokers/queue`. The work queue claim endpoint — returns up to
`$limit` ready symbol stream names and atomically sets `Safebox/status =
'running'` on each to prevent double-dispatch.

Uses `$db->rawQuery($sql, $params)->execute()` throughout (not `$db->query()`).
Uses `array_column()` for single-column SELECT results. Uses `->rowCount()` for
UPDATE claim verification. Both outer query and `NOT EXISTS` subquery include
`fromPublisherId = ?` to scope results to the correct publisher.

### `handlers/Grokers/node_request_helper.php`

Validates that incoming requests come from the local Node.js process via
`Q_Utils::signature()` HMAC verification. Used for internal callbacks.

---

## 20. CLI Reference

```
grokers index     <repoPath>              Parse + build graph + upsert to Streams
grokers regrok    <repoPath> [--dry-run]  Incremental re-index by content hash
grokers analyze   <repoPath>              Run analyzer agents bottom-up
grokers test      <repoPath>              Generate tests + orthogonality validation
grokers watch     <repoPath>              Watch for changes → incremental re-analysis
grokers docs      <repoPath> [--stale-only]  Generate documentation pages
grokers ask       <repoPath> "<question>" Natural language query against the graph
grokers history   <streamName>            Print message timeline for a symbol
grokers status    <repoPath>              Live symbol counts and comprehension progress
grokers conventions <repoPath>           Auto-discover framework conventions
```

**`grokers status`** reads live counts from the DB by paginating all
`Grokers/symbol` relations and checking which have `Grokers/comprehendedTime`
set. It does not rely on pre-aggregated attributes (which were never written).

### Configuration — `grokers.json` in repo root

```json
{
  "baseUrl":      "https://myapp.com",
  "publisherId":  "abc123def456",
  "communityId":  "acme",
  "sessionToken": "...",
  "mysql": {
    "host": "localhost", "user": "qbix",
    "password": "...", "database": "qbix_app"
  }
}
```

---

## 21. Install & Seed

### What the installer does

```php
function Grokers_after_Q_Plugin_install($params) {
    if (!isset($params['plugin']) || $params['plugin'] !== 'Grokers') return;
    require_once GROKERS_PLUGIN_DIR . DS . 'scripts' . DS . 'Grokers'
        . DS . '0.1-Streams.mysql.php';
}
```

### What the seed script does

All creates use `Q::app()` (not a hardcoded string) as the publisher. Everything
is idempotent — existing streams are detected and skipped; relations catch
duplicate-key exceptions.

1. **Search bucket categories**: `Grokers/search/symbols`, `/components`,
   `/externs`, `/clues`, `/concepts`
2. **`registerRelations`** for all Grokers stream types (see §3)
3. **Repo stream** for the hosting app's codebase (optional seed)
4. **`Streams/category`** is registered in `plugin.json` so Qbix accepts
   creation of bucket streams

### PUBLISHER timing

`Q.Grokers.PUBLISHER` is defined as an `Object.defineProperty` getter in
`classes/Grokers.js` — never captured at module load time. The same lazy
pattern applies in every file that needs the publisher at Node startup:
`base.js`, `discovery.js`, and all stream client calls use `getPublisher()`
functions or `Object.defineProperty` getters. `Q.Config` is not initialized
at `require()` time.

---

## 22. Appendix: Configuration Reference

```json
{
  "Grokers": {
    "model":            "claude-opus-4-6",
    "maxWorkers":       8,
    "pollMs":           2000,
    "maxSteps":         20,
    "hintsDir":         null,
    "conventionsDir":   ".grokers",
    "batchSize":        50
  }
}
```

Environment variables:

```
ANTHROPIC_API_KEY      Required for all LLM calls
GROKERS_MODEL          Override model (default: claude-opus-4-6)
GROKERS_MAX_STEPS      Override maxSteps per agent (default: 20)
GROKERS_MAX_WORKERS    Override parallel worker count (default: 8)
```

Notes:

- All Anthropic clients are lazy-initialized: `_client = null; function getClient() { if (!_client) _client = new Anthropic.Anthropic({...}); return _client; }`. This applies in `base.js`, `discovery.js`, and any file that instantiates an Anthropic client. Eager initialization at `require()` time silently captures `undefined` as the API key if the environment variable isn't set yet.
- `Grokers/comprehendedTime`, `Grokers/indexedTime`, `Grokers/removedTime`, `Grokers/passingTime`, `Grokers/testRunTime`, `Grokers/generatedTime` are the canonical timestamp attribute names. The `*At` suffix was removed throughout.
- `updateRelationExtra` is the correct way to update relation extras — not unrelate+relate. The Qbix `Streams/related` PUT handler accepts an `extra` field that calls `setExtra()` on the PHP ORM row.
- The `Grokers/test/lowOrthogonality` message type (not `lowOrthogonalityScore`) is used when a test's score drops below the threshold.

---

*End of Grokers Plugin Architecture Reference.*
*Version: 2026-04 · companion to Safebox.md and Safebots.md*
