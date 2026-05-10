# Code Plugin — Architecture & Implementation Reference (v0.1)

> **Audience**: LLM implementers and developers building or extending the Code plugin for Qbix.
> Self-contained for the Code plugin; assumes familiarity with Safebox (`Safebox/README.md`) and Grokers (`Grokers/Grokers.md`).

-----

## Table of Contents

1. [Overview & Philosophy](#1-overview--philosophy)
1. [Plugin Dependencies](#2-plugin-dependencies)
1. [Plugin File Structure](#3-plugin-file-structure)
1. [Stream Types — What Code Owns and What It Reads](#4-stream-types--what-code-owns-and-what-it-reads)
1. [Workspace Model](#5-workspace-model)
1. [Attribute Indexes for Search](#6-attribute-indexes-for-search)
1. [The Workflow Family](#7-the-workflow-family)
1. [Tools](#8-tools)
1. [Capabilities](#9-capabilities)
1. [Event Handlers](#10-event-handlers)
1. [Contract Verification (A/B/C)](#11-contract-verification-abc)
1. [Ripple to Callers](#12-ripple-to-callers)
1. [Git Projection](#13-git-projection)
1. [Naming Conventions](#14-naming-conventions)
1. [Installation Script](#15-installation-script)
1. [What Code Doesn’t Invent — Substrate Primitives Used As-Is](#16-what-code-doesnt-invent--substrate-primitives-used-as-is)
1. [Open Questions](#17-open-questions)
1. [Workflow Signaling and Resume Semantics](#18-workflow-signaling-and-resume-semantics)

-----

## 1. Overview & Philosophy

The Code plugin applies Safebox’s governed workflow primitive to source code, using Grokers’ typed knowledge graph as the substrate. It introduces the absolute minimum of new ontology — four Code-owned stream types, two capabilities, twelve tools, three workflows — and otherwise rides entirely on what Grokers, Safebox, and Streams already provide.

**What this plugin does**: takes a Grokers-indexed codebase, clones it into a workspace, proposes structured modifications using the Grokers contract attributes (preconditions, postconditions, sideEffects, invariants — all prose strings written by the Grokers analyzer agent), verifies contract preservation, ripples to callers when contracts change incompatibly, runs covering tests, and projects successful sprints to git as long-lived branches.

**What this plugin does not do**: index codebases (that’s Grokers), provide the workflow primitive (that’s Safebox), provide the substrate (that’s Qbix Streams).

The architectural commitment, lifted from the Safebox arXiv paper: workflows are domain-agnostic; what makes a workflow useful is the right combination of substrate primitives. Code is the worked example for source code as the domain. Other domain plugins (infrastructure, regulatory filings, scientific data) follow the same pattern.

-----

## 2. Plugin Dependencies

```
Qbix Streams (Qbix core)
   ├── Grokers ≥ 0.1
   │     for: Grokers/repo, Grokers/symbol, Grokers/extern,
   │          Grokers/calls, Grokers/covers, Grokers/concept,
   │          Grokers/clue, ontology.json
   └── Safebox ≥ 0.37
         for: workflows, tools, capabilities, action proposals,
              governance, conventions, sandbox API,
              Streams.ontology()
        │
   Code (this plugin) ≥ 0.1
        │
   Safebots, future agent layers
```

Code does NOT extend Grokers’ ontology. It uses Grokers types as-is. The only new types Code defines are bookkeeping: `Code/sprint`, `Code/finding`, `Code/branch`, `Code/testRun`.

-----

## 3. Plugin File Structure

```
Code/
  package.json                  @qbix/code, peer-deps Safebox + Grokers
  README.md                     this document
  config/
    plugin.json                 dependency declaration, event handlers, stream type defaults
    workflows/                  workflow JSON definitions seeded at install
      code-modify.json
      code-rename.json
      code-document.json
      code-cloneRepo.json
      code-audit.json
      code-notify.json
  classes/
    Code.php                    static helpers (gitProjection, branchName, sourceRepoOf)
    Code/
      Code.js                   Node executor RPC entry (cloneWorkspace, destroyWorkspace, pushBranch)
      Workspace.js              ZFS snapshot/clone/destroy via child_process
      Projection.js             git add+commit+push via child_process
  files/
    Safebox/
      tools/                    tool source — codeFile paths relative here
        find/code/callers.js
        find/code/coveringTests.js
        propose/code/rewrite.js
        verify/code/contract.js
        apply/code/rewrite.js
        ripple/code/callers.js
        clone/code/repo.js
        clone/code/upstream.js
        refresh/code/comprehension.js
        run/code/tests.js
        complete/code/sprint.js
        summarize/code/changes.js
        audit/code/finding.js
        scan/code/symbols.js
        notify/code/subscribers.js
      capabilities/
        Code/
          workspace.js          materializes a workspace ZFS clone
          repo.js               materializes a Grokers/repo via git clone + ZFS create
          testRun.js            materializes Code/testRun
          branch.js             materializes Code/branch
  handlers/
    Code/
      after/
        Q_Plugin_install.php    triggers migration
      message/
        Code_workspaceStarted.php   ZFS clone after Code/workspaceStarted message
        Code_sprintCompleted.php    git push after Code/sprintCompleted message
    Streams/
      after/
        Streams_Stream_save_Grokers_repo.php   ZFS destroy on workspace close (closedTime set)
  scripts/
    Code/
      0.1-Code.mysql.php        install: stream type registration, tool/capability/workflow seeding,
                                default convention seeding for PHP/JS/Python
```

The `files/Safebox/{tools,capabilities}/` layout is mandatory — it matches the canonical Safebox plugin convention so the `Safebox/codeFilePlugin: 'Code'` attribute resolves the path correctly.

-----

## 4. Stream Types — What Code Owns and What It Reads

### Stream types Code owns

|Type          |Purpose                                                                                                                           |Materialized by                                  |
|--------------|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
|`Code/sprint` |A code modification sprint — workflow execution producing one workspace repo, one set of modifications, optionally one git branch.|created by workflows; not externally materialized|
|`Code/finding`|An audit observation needing investigation.                                                                                       |created by `audit/code/finding` tool             |
|`Code/branch` |A git branch produced by a successful sprint, with commit sha.                                                                    |`Safebox/capability/Code/branch`                 |
|`Code/testRun`|A test execution record with pass/fail and captured output.                                                                       |`Safebox/capability/Code/testRun`                |

### Stream types Code reads (defined by Grokers)

|Type               |Used for                                                                                                                                                                                                                                                                         |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`Grokers/repo`     |source and workspace repos — the unit operations target                                                                                                                                                                                                                          |
|`Grokers/module`   |plugin/package containment unit; maps to a Qbix plugin or equivalent                                                                                                                                                                                                             |
|`Grokers/class`    |class containment — a bucket of methods                                                                                                                                                                                                                                          |
|`Grokers/method`   |callable named unit (class method or top-level function)                                                                                                                                                                                                                         |
|`Grokers/symbol`   |non-class, non-method symbols — constants, type aliases, traits, decorators, enum members, etc.                                                                                                                                                                                  |
|`Grokers/component`|Qbix tool definition or framework-registered component (Q.Tool.define, React component, etc.)                                                                                                                                                                                    |
|`Grokers/hook`     |event-bridge stream with three-phase semantics (before/primary/after); promoted from `Grokers/extern` category in Grokers update 2                                                                                                                                               |
|`Grokers/extern`   |bridge node for cross-process dispatch — config keys (`config`), text strings (`text`), templates (`template`), HTTP endpoints (`endpoint`), ORM tables (`table`), message types, socket events, env vars, and other categories. Hooks are now their own stream type (see above).|
|`Grokers/pattern`  |per-callee aggregate of arg distributions and suspected bugs across all call sites; read by audit/code/finding                                                                                                                                                                   |
|`Grokers/concept`  |confirmed concepts injected into LLM prompts in `propose/code/rewrite`                                                                                                                                                                                                           |
|`Grokers/clue`     |unresolved investigations — informs audit findings; v0.1+ clue types include possible-dynamic-tool-instantiation, possible-dynamic-dispatch, possible-implicit-handler, possible-metaprogramming, possible-dynamic-event-binding                                                 |
|`Grokers/test`     |tests that cover a symbol — filtered execution via `find/code/coveringTests`                                                                                                                                                                                                     |


> **Tranche dependency.** The split of `Grokers/symbol` into `Grokers/module`, `Grokers/class`, `Grokers/method` plus residual `Grokers/symbol` is requested in the Code-to-Grokers feature request, tranche 1. Until tranche 1 lands on the Grokers side, Code’s tools fall back to filtering `Grokers/symbol` streams by `Grokers/type` attribute (function/method/class/constant). Tools that depend on tranche 2 (HTML visitor, handlers/ walker, plugin.json visitor, JS event property-path resolver) report partial coverage with `note` outputs explaining what’s missing. See §17.

### Grokers contract attributes Code consumes

Per Grokers’ `config/ontology.json`, these are **prose strings** written by the LLM analyzer agent (not arrays, not structured fields):

- `Grokers/preconditions` — what must be true to call this symbol
- `Grokers/postconditions` — what is true after it returns
- `Grokers/sideEffects` — DB writes, stream mutations, events fired, filesystem changes
- `Grokers/invariants` — properties that hold throughout execution
- `Grokers/doc` — generated documentation (markdown)
- `Grokers/qualName` — fully-qualified symbol name
- `Grokers/file` — relative file path
- `Grokers/language` — enum: php / javascript / typescript / python / go / rust / swift / java / c / cpp
- `Grokers/symbolHash`, `Grokers/astHash` — for staleness detection

Contract attributes apply to `Grokers/method` streams (and to `Grokers/class` for class-level invariants). Code’s tools must handle all four contract attributes being absent — comprehension may not yet have run on a symbol, in which case the LLM is instructed to re-derive contract from source.

### Grokers/pattern attributes Code consumes

A `Grokers/pattern` stream is a materialized aggregate per callee — one stream per `Grokers/method` (or `Grokers/symbol`) with at least one caller. Maintained incrementally by Grokers’ indexer aggregator after every regrok pass. Reading the aggregate replaces scanning every `Grokers/calls` relation to compute distributions.

- `Grokers/callee` — stream name of the callee
- `Grokers/callerCount` — distinct caller symbols
- `Grokers/callSiteCount` — total call site edges
- `Grokers/arg0dist` — JSON bucket map of first-arg shapes, e.g. `{"dynamic:this": 31, "literal:true": 8}`
- `Grokers/arg1dist` — JSON bucket map for second-arg position
- `Grokers/suspectedBugs` — count of call sites where hints flagged `requiresPairedRemove=true` but no matching `.remove()` was found in the same symbol — surfaces likely Q.Event leak patterns and similar
- `Grokers/updatedTime` — Unix ms timestamp of the last aggregation pass

### Grokers relations Code traverses

The substrate convention: containment relations are named after the contained-end stream type, walked bidirectionally via `isCategory`. A bucket walked with `isCategory=true` returns its contents; a contained stream walked with `isCategory=false` returns its bucket(s).

**Containment relations:**

|Relation        |Bucket → contents                                             |Contents → bucket           |
|----------------|--------------------------------------------------------------|----------------------------|
|`Grokers/module`|repo → modules in it                                          |module → its repo           |
|`Grokers/class` |module → classes in it (or repo → classes)                    |class → its module          |
|`Grokers/method`|class → its methods, or module → top-level functions          |method → its class or module|
|`Grokers/symbol`|repo or module → other symbols (constants, type aliases, etc.)|symbol → its container      |

**Verb-relations (cross-cutting dependencies, not containment):**

|Relation              |Direction                                 |What Code uses it for                                                                                                                                                                      |
|----------------------|------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`Grokers/calls`       |caller → callee                           |finding callers (reverse via `isCategory=true`); finding callees (forward via `isCategory=false`)                                                                                          |
|`Grokers/invokes`     |symbol → hook (or extern)                 |symbol fires a hook (e.g. `Q.handle("Foo.bar", ...)`, `Q::event(...)` → targets `Grokers/hook/Foo.bar`); extra carries `phase: “before”                                                    |
|`Grokers/subscribes`  |symbol → hook                             |symbol registers to receive a hook event (e.g. `Q.Event.set` calls); pairs with `Grokers/invokes` to form a dispatch bridge                                                                |
|`Grokers/handles`     |handler-method → hook                     |for PHP `handlers/<plugin>/<phase>/<event>.php` convention AND `plugin.json` `handlersAfterEvent` registrations (materialized by Grokers’ config-connector); extra carries `phase: “before”|
|`Grokers/activates`   |config-file symbol → handler method       |for `plugin.json` `handlersAfterEvent`/`handlersBeforeEvent`/`handlersValidateEvent` registrations                                                                                         |
|`Grokers/instantiates`|caller-symbol or markup-symbol → component|a `Q::tool()` call, a `Q.Tool.create()` call, or a `data-tool="..."` attribute each emit one of these to a `Grokers/component`                                                             |
|`Grokers/reads`       |symbol → extern                           |symbol reads from a config/template/registry (e.g. `Q.Template.load`, `Q.Config.get`)                                                                                                      |
|`Grokers/writes`      |symbol → extern                           |symbol writes to a registry (e.g. `Q.Template.set`, `Q.Config.set`)                                                                                                                        |
|`Grokers/covers`      |test → symbol                             |finding tests covering a symbol (reverse)                                                                                                                                                  |
|`Grokers/concept`     |repo → concept                            |injecting confirmed concepts into prompts                                                                                                                                                  |
|`Grokers/pattern`     |callee → pattern stream                   |reaching the per-callee aggregate from a callee symbol                                                                                                                                     |

Per Grokers’ ontology, **`weight` on these relations stores the source line number**, and **`extra` is a JSON-encoded object** carrying:

```
{
    line, certainty, ctrl?, ctrlType?, note?,
    arg0kind, arg0text?, arg0val?,
    arg1kind, arg1text?, arg1val?
}
```

`arg0kind`/`arg1kind` values:

- `"literal"` — static literal extractable at parse time; `arg0val` carries the parsed value
- `"dynamic"` — non-literal expression; `arg0text` carries the raw node text (e.g. `"this"`, `"$var"`, `"fn()"`)
- `"absent"` — argument position not present in the call
- `"array"` — array literal; `arg0val` holds JSON-encoded element list

The arg fields matter especially for `Q.Event.prototype.set` calls in Qbix codebases: `arg0text="this"` means the handler is keyed to a `Q.Tool` instance and gets auto-removed on tool destroy (no manual cleanup needed); `arg0val=true` means page-unload auto-removal; a string-literal key means the call requires a paired `.remove(key)` in the destroy path or it’s a leak. Code’s `ripple/code/callers` consumes these fields to classify caller adaptations more accurately than line numbers alone.

-----

## 5. Workspace Model

A code modification workspace is a copy of the source `Grokers/repo` plus its symbol streams, sitting at a different filesystem path. Symbols modified inside a workspace are independent from the originals.

### Two ways to do this — substrate has both, but only one is wired up today

**(A) `Streams::fork` — the substrate-native primitive.** The Streams plugin ships `Streams::fork($asUserId, $publisherId, $streamName, $ordinal, $toPublisherId, $toStreamName, $options)`. It copies fields, sets up a JSON `fork` chain on the new stream, fires `Streams/fork/{type}` before/after events, and posts a `Streams/forked` message on the new stream. The fork chain enables transparent message-history traversal across the source and fork via `Streams_Message::forkChain` and the workspace cascade in `Streams::related` (the cascade is on the v1.3.1 roadmap; not yet active). The convention is `toPublisherId = "{publisherId}~{workspaceId}"`.

**(B) Plain new stream with back-reference attribute — Code’s transitional approach.** The `clone/code/repo` tool creates a fresh `Grokers/repo` stream under the community publisher, with a `Code/sourceRepoUri` attribute pointing back at the source. No fork chain, no shared message history. Side effects (ZFS clone, attribute population) happen via the `Code/workspaceStarted` message hook.

### Why Code uses (B) in v0.1

`Streams::fork` is a substrate-level PHP primitive. To use it from a Safebox tool, the tool would need an `Action.propose('Streams/fork', ...)` action type — and Safebox v0.37 does not yet expose `Streams/fork` in its `Action.php` switch statement (only `Streams/create`, `Streams/update`, `Streams/relate`, `Streams/post-message`, etc.). Until Safebox adds `Streams/fork` as a governed action type, Code tools cannot use the substrate’s fork primitive directly.

The transitional approach is functionally adequate — Code’s workflows don’t depend on fork-chain message inheritance. When Safebox adds `Streams/fork` as an action type, `clone/code/repo` will switch to using it and gain the fork-chain benefits at no cost to the calling workflow.

### What `clone/code/repo` actually does in v0.1

1. Reads the source repo’s `Grokers/repoHash`.
1. Computes a workspace `repoHash` deterministically from `(sourceRepoHash, workspaceLabel, timestamp)`, slugged to 16 hex chars per Grokers convention.
1. Proposes `Streams/create:Grokers/repo` with the new hash, `Code/sourceRepoUri` and `Code/sourceRepoHash` attributes pointing back, and `Safebox/status='pending'`.
1. Posts `Code/workspaceStarted` on the new workspace repo. The `Code_workspaceStarted` handler runs `zfs snapshot` and `zfs clone`, updates the workspace repo’s `Grokers/repoRoot` and `Grokers/zfsDataset` to point at the clone’s mountpoint, and copies the source’s `Code/testCommand` attribute onto the workspace if not already inherited.

After the handler completes, the workspace looks like a fresh Grokers/repo at a different mountpoint. Symbol modifications inside the workspace become `Streams/update` actions on `Grokers/symbol/{newRepoHash}/...` streams; the originals at `Grokers/symbol/{sourceRepoHash}/...` are untouched. When the workspace is closed (via `Streams::close`), the save-after hook detects `closedTime` was just set on a stream carrying `Code/zfsCloneName` and tears down the ZFS clone.

-----

## 6. Attribute Indexes for Search

Code uses the substrate’s built-in attribute-indexing mechanism — `Streams_Stream::registerRelations` plus `syncRelations`. When a stream’s attribute changes, `syncRelations` automatically creates/destroys relations of the form `attribute/{key}={value}` between the index stream and the source stream. Tools query the index via `Streams.related(idxPub, idxName, 'attribute/{key}={value}', isCategory=true, opts)` to find streams by attribute value.

This is exactly the mechanism Grokers’ search buckets (`Grokers/search/symbols`, etc.) use. Code adopts the same pattern rather than inventing parallel infrastructure.

### Indexes Code registers at install time

|Index stream                |Stream type   |Indexed attributes                                                  |
|----------------------------|--------------|--------------------------------------------------------------------|
|`Safebox/index/Code/finding`|`Code/finding`|`Safebox/status`, `Code/severity`, `Code/findingType`, `Code/status`|
|`Safebox/index/Code/branch` |`Code/branch` |`Safebox/status`                                                    |
|`Safebox/index/Code/testRun`|`Code/testRun`|`Safebox/status`, `Code/passed`                                     |

(v0.3 also registered an index for `Code/sprint`, but no v0.1 tool actually creates streams of that type — the workload itself is `Safebox/workload`, and per-sprint metadata lives on the workspace `Grokers/repo`. The unused index has been removed in v0.4.)

Registration happens via:

```php
Streams_Stream::registerRelations(
    '',                              // any publisher
    'Code/finding',                  // stream type
    $executorId,                     // index publisher
    'Safebox/index/Code/finding',    // index stream
    array('Safebox/status', 'Code/severity', 'Code/findingType', 'Code/status')
);
```

After registration, anywhere a `Code/finding` stream is saved with `Code/severity = 'high'`, `syncRelations` creates an `attribute/Code/severity=high` relation between the index and the finding. Tools find all high-severity findings via:

```js
var rels = await Streams.related(executorId, 'Safebox/index/Code/finding',
    'attribute/Code/severity=high', true, { limit: 100 });
```

Per the substrate convention this is how all attribute-based search works — Grokers uses it for symbols, Safebox uses it for actions, Code uses it for findings/sprints/branches/testRuns. **No bespoke search bucket infrastructure**.

-----

## 7. The Workflow Family

v0.1 ships eight workflows with full step DAGs. The shape mirrors safebots.ai/coding.html: each workflow forks a workspace, reads context, decides, applies, verifies, completes. Composition between workflows happens through messages and handlers — not a central orchestrator.

### `code/modify` — change a symbol’s behavior with bounded ripple

15+ steps including progress messages and conditional terminal paths:

1. `clone-workspace` — create workspace repo (transitional pattern; will use `Streams::fork` when available)
1. `find-covering-tests` — locate tests covering this symbol via `Grokers/covers`
1. `propose-rewrite` — LLM proposal grounded in concepts + conventions + contract
1. `verify-contract` — A/B/C classification
1. `post-rewrite-proposed` — fires `Code/rewriteProposed` progress message
1. `apply-rewrite` — `Streams/update` on workspace symbol; returns line counts and shift parameters
1. `shift-line-numbers` — fires `Code/lineShift` so the PHP handler bulk-shifts subsequent symbols’ position-in-file (`Streams/file` relation weights) and outgoing verb-relations’ weights via SHIFT MODE of `Streams::updateRelations`. Advisory; Grokers’ regrok overwrites with authoritative line numbers.
1. `ripple-callers` — only when class C; bounded by `depthRemaining`; fires `Code/rippleStarted`/`Code/rippleEnded`/`Code/depthExhausted`/`Code/checkpointRequested` per outcome
1. `notify-subscribers` — only when class C; fans out subscriber + hook + extern + associated-peer notifications with phase metadata (the associated walk surfaces LLM-discovered semantic peers per Grokers Update 4)
1. `post-tests-started` — fires `Code/testRunStarted`
1. `run-tests` — execute via `Safebox/capability/Code/testRun`
1. `post-tests-completed` — fires `Code/testRunCompleted`
1. `summarize-changes` — walks `Code/rewriteApplied` messages on workspace repo, produces markdown changelog
1. `refresh-comprehension` — posts `Code/regrokRequested` advisory message and triggers Grokers pattern aggregator
1. `complete-sprint-success` / `complete-sprint-with-findings` / `complete-sprint-aborted` — three terminal variants; see §18

Edge conditions:

- `apply-rewrite → shift-line-numbers` (always)
- `shift-line-numbers → ripple-callers` if class is C
- `shift-line-numbers → post-tests-started` if class is A or B (skip ripple+notify)
- `ripple-callers → notify-subscribers` if `checkpointRequested != true`
- `post-tests-completed → summarize-changes` if `passed === true`; else `→ complete-sprint-aborted`
- `refresh-comprehension → complete-sprint-with-findings` if `depthCapReached == true`; else `→ complete-sprint-success`

A failing test takes the abort path. The workspace stays for inspection. Depth-exhausted cascade takes the with-findings path; the workspace is valid but not all callers were processed.

### `code/rename` — mechanical identifier rename

6 steps. No contract verification (rename preserves contract by construction). Per-caller fan-out is left to follow-on workloads spawned by the orchestrator. Includes summarize + complete-sprint to produce a git commit when projection is configured.

### `code/document` — docstring updates only

5 steps. Reuses `propose/code/rewrite` with a docstring-only change description; the LLM constraint enforces “do not modify the function body.”

### `code/cloneRepo` — bring an upstream codebase into Safebox

1 step (`clone-upstream`) wrapping the `clone/code/upstream` tool, which calls the `Code/repo` capability for `git clone` + ZFS dataset creation, and creates a fresh `Grokers/repo` stream. Used for initial deployment, periodic refresh (clone at a newer commit; old repo stays as historical record), and historical analysis (clone at an older commit). All three uses go through the same workflow.

### `code/audit` — read-only walks producing findings

2 steps:

1. `scan-symbols` — walks symbols in a `Grokers/repo` matching an audit query (`missingContract`, `minComplexity`, `language`, `qualNamePattern`, attribute equality, `minSuspectedBugs`). Read-only.
1. `emit-findings` — fan-out `audit/code/finding` per candidate, creating `Code/finding` streams for human or downstream-agent investigation.

The query is a structured object passed in by the caller. Examples: `{missingContract: true}` finds symbols missing one or more `Grokers/preconditions`/`postconditions`/`sideEffects`/`invariants` attributes; `{minComplexity: 30, language: "javascript"}` finds expensive JS symbols; `{attributeKey: "Grokers/securitySensitive", attributeValue: true}` finds symbols pre-flagged; `{minSuspectedBugs: 1}` finds callees whose `Grokers/pattern` aggregate flagged at least one likely-leak call site (most commonly `Q.Event.set` calls with string-literal keys missing a paired `.remove()`).

### `code/auditClues` — read-only walks of clues, producing findings

2 steps. Parallel to `code/audit` but walks the `Grokers/clue` graph rather than symbol attributes. Used for surfacing static-analysis boundaries (`possible-dynamic-dispatch`, `possible-implicit-handler`, etc.) and Grokers-derived semantic observations (`possible-doc-discrepancy` from tier-2 grok pass — see §18.9). Filters by `clueType` and `minSeverity`.

### `code/auditCss` — read-only walks of CSS selectors, producing findings

2 steps. Walks `Grokers/selector` streams (per Grokers Update 4 — populated by the css.js, html.js, handlebars.js, jsx.js visitors during indexing) for CSS quality issues. Three `scanType` values:

- `cssConflicts` — surface selectors with `Grokers/hasConflict=true` and at least N (default 2) contributing rule blocks. Multiple files defining competing properties for the same selector means cascade order determines outcome — brittle to refactor. Severity scales with rule count.
- `unusedSelectors` — selectors defined in CSS (`Grokers/ruleCount > 0`) but never referenced in HTML or templates (`Grokers/usageCount === 0`). Dead style rules. Low severity.
- `undefinedSelectors` — selectors referenced in HTML or templates (`Grokers/usageCount > 0`) but no CSS rule defines them (`Grokers/ruleCount === 0`). Likely broken or stale styling. High severity.

Findings emit with `findingType: "css-{scanType}"`. The reason text in each finding includes the selector text, specificity tuple, and counts.

### `code/notify` — standalone contract-change notification

1 step wrapping `notify/code/subscribers`. Fans out subscriber + hook-bridge + extern-bridge + associated-peer notifications. The hook walks include phase metadata so recipients understand whether the contract change affects a before/primary/after handler relationship. The associated walk (per Grokers Update 4) traverses `Grokers/associated` edges in both directions, surfacing LLM-discovered semantic peers — pairs the analyzer flagged as `semantic-pair`, `always-used-together`, `precondition`, `postcondition`, `alternative`, or `replaces`. Same machinery `code/modify` uses inline; exposed as a workflow so external parties can trigger it directly (e.g., when an upstream contract changes outside a Code workflow’s scope).

### Deferred workflows (not yet shipped)

- `code/split` — decompose a function into helpers
- `code/inline` — eliminate a helper, propagate body
- `code/test` — generate tests for under-covered symbols
- `code/migrate` — fan out parallel `code/modify` sub-workloads
- `code/createSymbol` — for new-code authoring scenarios (anticipated v0.5). When implemented, will walk `Grokers/example` (a *relation*, not attribute) from the relevant `Grokers/pattern` stream — typically the one with `patternType=component-file-structure` for the target module — to find the top-weighted reference implementations, then read each example method’s file at its `startLine`/`endLine` extras for RAG injection into the generation prompt. Per Grokers Update 4 plus correction: `Grokers/example` carries `score` (also the relation weight), `reason`, `startLine`, `endLine` extras.

-----

## 8. Tools

All 20 tools follow the Safebox plugin-shipped tool convention:

- File path: `files/Safebox/tools/<verb>/code/<noun>.js`
- Stream name: `Safebox/tool/<verb>/code/<noun>`
- `Safebox/codeFile`: `<verb>/code/<noun>.js` (relative to `files/Safebox/tools/`)
- `Safebox/codeFilePlugin`: `'Code'`
- `Safebox/sha256`: SHA-256 of the file at install time

### Action types declared per tool

Per Safebox’s runtime contract enforcement, every Action.propose call’s type must be in the tool’s declared `Safebox/actionTypes`. v0.1 uses **bare action verbs only** (e.g., `Streams/update`) — no colon-suffixed forms — because Safebox v0.37’s action-type regex `^[A-Za-z][A-Za-z0-9_./-]{0,127}$` rejects colons. The stream type the action targets goes inside the payload as `streamType`, not in the action type string.

|Tool                        |actionTypes                                                                                                                                                                            |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`find/code/callers`         |`[]`                                                                                                                                                                                   |
|`find/code/coveringTests`   |`[]`                                                                                                                                                                                   |
|`propose/code/rewrite`      |`[]`                                                                                                                                                                                   |
|`verify/code/contract`      |`[]`                                                                                                                                                                                   |
|`apply/code/rewrite`        |`[ Streams/update, Streams/message ]`                                                                                                                                                  |
|`ripple/code/callers`       |`[ Streams/create, Streams/message, Streams/update ]` (sub-workloads + progress messages + checkpoint state)                                                                           |
|`clone/code/repo`           |`[ Streams/create, Streams/message ]` (workspace repo + workspaceStarted)                                                                                                              |
|`clone/code/upstream`       |`[ Streams/create ]` (fresh `Grokers/repo`)                                                                                                                                            |
|`refresh/code/comprehension`|`[ Streams/message ]`                                                                                                                                                                  |
|`run/code/tests`            |`[ Streams/message ]` (test result; capability creates the testRun stream)                                                                                                             |
|`complete/code/sprint`      |`[ Streams/message, Streams/update ]`                                                                                                                                                  |
|`summarize/code/changes`    |`[]`                                                                                                                                                                                   |
|`audit/code/finding`        |`[ Streams/create ]`                                                                                                                                                                   |
|`scan/code/symbols`         |`[]`                                                                                                                                                                                   |
|`notify/code/subscribers`   |`[ Streams/message ]`                                                                                                                                                                  |
|`post/code/progress`        |`[ Streams/message ]` (generic progress-message proxy used by workflow JSON at step boundaries)                                                                                        |
|`request/code/checkpoint`   |`[ Streams/message, Streams/update ]` (pause-for-review at cascade boundary)                                                                                                           |
|`shift/code/lineNumbers`    |`[ Streams/message ]` (posts `Code/lineShift` for the PHP after-handler to pick up; PHP-side does the bulk relation+attribute updates via `Streams::updateRelation` and `setAttribute`)|
|`scan/code/clues`           |`[]` (read-only walk of `Grokers/clue` streams, filtered by `clueType` and `minSeverity`)                                                                                              |
|`scan/code/selectors`       |`[]` (read-only walk of `Grokers/selector` streams; supports cssConflicts / unusedSelectors / undefinedSelectors scan types per Grokers Update 4)                                      |

### Sandbox API used

Tools use only the canonical Safebox sandbox API:

- `Streams.get(publisherId, streamName)` — single stream fetch (cached batch)
- `Streams.related(publisherId, streamName, type, isCategory, opts)` — relation traversal
- `Streams.fetch(publisherId, streamName)` — capability-triggered materialization
- `Streams.getMessages(publisherId, streamName, opts)` — message log read
- `Streams.ontology(pluginName)` — fetch and parse `config/ontology.json` (for grounding LLM prompts)
- `Action.propose(type, payload)` — governed write
- `Runtime.llm(spec)` — LLM call with `{model, system, messages, maxTokens, temperature}`
- `Crypto.sha256(text)` — for deterministic stream name slugging (always `await`ed)

Stream attributes returned to the sandbox arrive as JSON strings — every tool inlines `parseAttrs(s)` to unmarshal:

```js
function parseAttrs(s) {
    if (!s) return {};
    var raw = s.attributes || (s.fields && s.fields.attributes) || '{}';
    try { return JSON.parse(raw); } catch (e) { return {}; }
}
```

### Cross-posting Code/rewriteApplied

`apply/code/rewrite` posts `Code/rewriteApplied` on **two** streams: the symbol itself (audit trail) and the workspace repo (so `summarize/code/changes` can find the modification list with one `getMessages` call instead of walking every symbol). This is the v0.1 pattern for surfacing per-sprint modifications without bespoke indexing.

The workspace-repo cross-post and other progress messages also flow to the **workload stream** when workflow JSON wires it through. See §18 for the full progress-message vocabulary.

-----

## 9. Capabilities

Four capabilities, each materializing one stream type:

### `Safebox/capability/Code/workspace`

Materializes a `Code/workspace` (or workspace `Grokers/repo`) by taking a ZFS snapshot of the source repo’s dataset and creating a writable clone. Returns `Grokers/repoRoot` (mountpoint), `Grokers/zfsDataset` (clone name), `Code/zfsCloneName`, `Code/zfsSnapshot`. Two trigger paths:

- **(a) Substrate-fork path** (when Safebox v0.38+ exposes `Streams/fork` as a governed action type): the `Streams/after/Streams/fork/Grokers/repo` hook invokes this capability. The capability code is the same; only the trigger differs.
- **(b) Message-driven path** (Safebox v0.37 transitional, what v0.1 ships): the `Code/workspaceStarted` message handler invokes the equivalent ZFS work via `Q_Utils::sendToNode → Code/cloneWorkspace`.

### `Safebox/capability/Code/repo`

Materializes a fresh `Grokers/repo` by cloning an upstream git repository at a specific commit into a new ZFS dataset. Used by `clone/code/upstream` (and therefore by the `code/cloneRepo` workflow). Reads `Code/upstreamUrl`, `Code/clonedAtCommit`, `Code/zfsDatasetName` from pending stream attributes; writes back `Grokers/repoRoot`, `Grokers/zfsDataset`, `Grokers/clonedAtCommit`, `Grokers/upstreamUrl`, `Code/clonedAt`.

### `Safebox/capability/Code/testRun`

Materializes a `Code/testRun` stream by running the workspace repo’s `Code/testCommand` in the cloned filesystem (`Grokers/repoRoot`). Returns pass/fail, exit code, duration, captured stdout/stderr. The test command is read from the workspace repo’s `Code/testCommand` attribute, inherited from the source repo at clone time:

```json
{
    "command": "npm",
    "args": ["test"],
    "timeout": 600000
}
```

### `Safebox/capability/Code/branch`

Materializes a `Code/branch` stream by running git add + commit + push against the workspace’s `Grokers/repoRoot`, targeting the `Grokers/gitProjection.remote` configured on the workspace repo (inherited from source).

### Sandbox availability

All four capabilities use `Protocol.System.exec` for shell-out. Safebox’s standard executor build does NOT expose `Protocol.System` — it’s available only on Code-enabled hardened executor builds. When unavailable, the capabilities return `Safebox/status='pending-host-action'` with the input attributes forwarded; the PHP-side handlers (`Code_workspaceStarted` for ZFS, `Code_sprintCompleted` for git) run the equivalent operation in the privileged Node executor process via `Q_Utils::sendToNode`. Production deployments use the PHP-handler path.

-----

## 10. Event Handlers

Five handlers, registered in `config/plugin.json` under `handlersAfterEvent`. The hook keys match Qbix Streams’ event-naming convention exactly (the `Streams/after/` prefix is required for save-after hooks scoped by stream type; the `Streams/message/` prefix dispatches by message type).

### `Q/Plugin/install` → `Code/after/Q_Plugin_install`

Fires once when the plugin is installed. Wraps the install routine in `scripts/Code/0.1-Code.mysql.php` — registers stream types, seeds built-in tools/capabilities/workflows, and seeds the default `Safebox/convention/code/*` streams (PHP, JS, Python).

### `Streams/message/Code/workspaceStarted` → `Code/message/Code_workspaceStarted`

Fires when `clone/code/repo` posts `Code/workspaceStarted` on a new workspace repo. Reads the source dataset from the source repo’s `Code/zfsDataset`, RPCs to the Node executor (`Code/cloneWorkspace`), takes the ZFS snapshot, creates the writable clone, then writes back to the workspace stream:

- `Grokers/repoRoot` ← clone mountpoint
- `Grokers/zfsDataset` ← clone dataset name
- `Code/zfsCloneName` ← same as zfsDataset
- `Code/zfsSnapshot` ← snapshot name
- `Code/testCommand` ← copied from source repo (if not already set)
- `Safebox/status` ← `'ready'`

If the source has no `Code/zfsDataset`, the handler marks the workspace `'ready-substrate-only'` and skips ZFS work — useful for testing workflow logic without filesystem operations. ZFS failure marks `Safebox/status='failed'` with `Code/error` set; the handler does not throw, since the message-after hook runs after the message commits and substrate rollback isn’t possible at that point.

### `Streams/message/Code/sprintCompleted` → `Code/message/Code_sprintCompleted`

Fires when `complete/code/sprint` posts `Code/sprintCompleted`. Reads the workspace’s `Grokers/gitProjection`; if absent, no-op (substrate-only sprint). If present:

1. Resolves the verb (`modify`, `rename`, etc.) by parsing the workload’s `Safebox/workflow` attribute
1. Renders the branch name from `Grokers/gitProjection.branchTemplate` or the plugin’s `Code.branchTemplateDefault`
1. Composes a commit message including the sprint summary (cross-posted as `Code/sprintCompleted`’s `changeDescription` field by `complete/code/sprint`) plus workload provenance
1. RPCs to `Code/pushBranch` which runs `git add -A && git checkout -B <branch> && git commit -m <msg> && git push --force-with-lease`
1. Posts `Code/sprintPushed` (success) or `Code/sprintPushFailed` (failure) on the workspace as audit trail

### `Streams/message/Code/lineShift` → `Code/after/Streams_message_Code_lineShift`

Fires when `shift/code/lineNumbers` posts `Code/lineShift` on a workspace repo (typically as the step right after `apply/code/rewrite` in the modify workflow). The handler reads the message instructions (`filePublisherId`, `fileStreamName`, `shiftStartLine`, `shiftDelta`, `modifiedSymbolPublisherId`, `modifiedSymbolStreamName`) and calls `Code::shiftLineNumbers`, which uses the SHIFT MODE of `Streams::updateRelations` (string weight values `"+N"` / `"-N"` triggering a single SQL UPDATE per relation type):

1. Walks `Streams/file` from the file stream once (sorted by weight) to enumerate symbols-in-file. Files are indexed as `Streams/file` streams; each symbol relates TO its containing file via the `Streams/file` relation type with `weight` = symbol start line.
1. For symbols at line ≥ `shiftStartLine` (excluding the modified symbol itself), bulk-shifts the `Streams/file` relation’s weight via one `Streams::updateRelations` call with `array('Streams/file' => '+N')` and `minWeight` filter.
1. For each shifted symbol, walks its outgoing verb-relations (`Grokers/calls`, `Grokers/reads`, `Grokers/writes`, `Grokers/invokes`, `Grokers/subscribes`, `Grokers/handles`, `Grokers/instantiates`, `Grokers/activates`) once to collect target coordinates, then bulk-shifts via one `Streams::updateRelations` per family with the same SHIFT MODE syntax.
1. For the **modified symbol itself**, doesn’t shift its position-in-file (start line typically doesn’t move when its body grows or shrinks). Instead, clears all its outgoing verb-relations via `Streams::unrelate` so regrok emits fresh ones from the new AST.

Posts `Code/lineShifted` ack with counts (`shiftedSymbols`, `shiftedRelations`, `clearedRelations`, `errorCount`). Best-effort and advisory — Grokers’ regrok of the file is authoritative. Known limitation: SHIFT MODE updates `weight` only, not `extra.line` (relation extras are JSON-encoded and not per-row rewritable in pure SQL); see §18.8 for details.

### `Streams/after/Streams/Stream/save/Grokers/repo` → `Streams/after/Streams_Stream_save_Grokers_repo`

Fires after any save on a `Grokers/repo` stream. The handler checks whether `closedTime` was just set (transition from null to non-null in `$params['changes']`) AND the stream has a `Code/zfsCloneName` attribute. If both hold, it’s a workspace fork being closed — RPCs to `Code/destroyWorkspace` to tear down the ZFS clone. Otherwise no-op.

This is the substrate-native rollback path. A workflow that fails verification calls `Streams::close()` on its workspace; the substrate fires the save event with `closedTime` in changes; this handler tears down ZFS state.

(v0.3 had a separate `Streams/Stream/close/Grokers/repo` hook key, but Qbix Streams doesn’t fire that event — close is just a save with `closedTime` set. v0.1 ships the correct save-after pattern.)

### Future hook — `Streams/after/Streams/fork/Grokers/repo`

When Safebox adds `Streams/fork` as a governed action type, `clone/code/repo` switches to using it; the substrate fires `Streams/after/Streams/fork/Grokers/repo` and the after-hook replaces `Code_workspaceStarted`. The hook’s job is the same — set up ZFS state and copy attributes — but the trigger becomes substrate-native rather than message-driven.

-----

## 11. Contract Verification (A/B/C)

The `verify/code/contract` tool re-derives a contract from new source and compares to the existing prose-string contract attributes. Classification:

- **A — preserved**: equivalent contract (modulo wording).
- **B — upward-compatible**: preconditions relaxed OR postconditions strengthened OR side effects removed. Existing callers still satisfy.
- **C — incompatible**: preconditions strengthened OR postconditions weakened OR new side effects added. Some callers will break.

The LLM is instructed with a **conservative bias**: when uncertain between B and C, prefer C. Misclassifying B as C costs one extra ripple pass that finds all callers still satisfy; misclassifying C as B causes silent breakage.

Short-circuit: byte-equivalent source after trim → trivially A, no LLM call.

-----

## 12. Ripple to Callers

Triggered by `code/modify` only when verify returns C. The `ripple/code/callers` tool walks `Grokers/calls` reverse from the modified symbol, classifies each caller against the new contract, and spawns `code/modify` sub-workloads for callers needing semantic changes.

Per-edge `extra` carries `{line, certainty, ctrl?, ctrlType?, arg0kind, arg0text?, arg0val?, arg1kind, arg1text?, arg1val?}`. The classifier uses the arg fields heavily: `arg0text="this"` on a `Q.Event.set` call site means the handler is keyed to a tool instance and gets auto-removed on tool destroy — no manual cleanup work needed regardless of the contract change. A string-literal key without a matching `.remove()` in the same caller is already flagged via `Grokers/pattern.suspectedBugs` and gets escalated rather than auto-rippled.

Recursion is bounded by `depthRemaining` (default 5, configurable). Most ripples terminate within 2-3 levels.

Per-caller verdicts are LLM judgments and can be wrong. Mitigations: (a) tests in the next workflow step catch breakage; (b) the `Grokers/pattern.suspectedBugs` count gives the LLM prior knowledge of likely-broken sites before it judges; (c) future versions can add a `Safebox/judgment` gate for high-stakes symbols.

-----

## 13. Git Projection

Each `Grokers/repo` stream may carry an optional `Grokers/gitProjection` attribute:

```json
{
    "remote":         "git@github.com:org/repo.git",
    "branchTemplate": "safebox/code-{verb}/{workloadShortId}",
    "credentialRef":  "Safebox/credentials/git-org-repo"
}
```

If absent, the repo is substrate-only — no git activity. If present, successful sprints push to the remote on a templated branch.

The plugin never pulls. When upstream has moved on, the operator runs a fresh clone (via `clone/code/repo` against the new commit) which produces a new `Grokers/repo` stream coexisting with the old as historical record.

Branches are long-lived. The plugin does not delete or merge them — that’s the upstream side’s prerogative.

-----

## 14. Naming Conventions

Per Safebox’s existing usage (verified against `scripts/Safebox/0.5-Streams.mysql.php`):

|Layer                    |Convention                                            |Example                                                                                                           |
|-------------------------|------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
|Workflows                |`Safebox/workflow/<namespace>/<noun>` lowercase       |`Safebox/workflow/code/modify`                                                                                    |
|Tools                    |`Safebox/tool/<verb>/<namespace>/<noun>` lowercase    |`Safebox/tool/find/code/callers`                                                                                  |
|Capabilities             |`Safebox/capability/<TargetStreamType>` mirrors target|`Safebox/capability/Code/testRun`                                                                                 |
|Code-owned stream types  |`Code/<type>` (capitalized first segment)             |`Code/sprint`, `Code/finding`                                                                                     |
|Code-owned message types |`Code/<event>` or `Code/<event>/<sub>`                |`Code/sprintCompleted`, `Code/workspaceStarted`, `Code/rewriteApplied`, `Code/testRunPassed`, `Code/testRunFailed`|
|Code-owned attribute keys|`Code/<key>`                                          |`Code/sourceRepoHash`, `Code/zfsDataset`                                                                          |

-----

## 15. Installation Script

`scripts/Code/0.1-Code.mysql.php` runs at install time. It:

1. Verifies Safebox ≥ 0.37 and Grokers ≥ 0.1 are installed.
1. Registers attribute-index templates via `Streams_Stream::registerRelations` for each `Code/*` type — see §6.
1. For each tool file in `files/Safebox/tools/`, computes sha256 and creates a `Safebox/tool/<verb>/code/<noun>` stream with `Safebox/codeFile`, `Safebox/codeFilePlugin: 'Code'`, declared `Safebox/inputs`, `Safebox/outputs`, `Safebox/actionTypes`. Sets `Safebox/approved: 'true'` per the Safebox install convention for built-in tools.
1. For each capability file, same pattern — creates `Safebox/capability/Code/<noun>` streams.
1. Loads each workflow JSON file from `config/workflows/`, creates a `Safebox/workflow/code/<noun>` stream with the steps and edges as `Code/steps` and `Code/edges` attributes.
1. Seeds default `Safebox/convention/code/*` streams (PHP, JavaScript, Python) with `Safebox/conventionDomain: 'code'`, `Safebox/severity: 'recommended'`, `Safebox/approved: 'true'`, and `appliesTo: { language }`. The `Safebox/convention` stream type is owned by the Safebox plugin (substrate-level shared namespace); Code seeds only the code-domain subtree. See §18.11 for the layered context model and the cross-domain note.

The script is idempotent. Re-running updates sha256 hashes if files changed; doesn’t duplicate streams.

-----

## 16. What Code Doesn’t Invent — Substrate Primitives Used As-Is

This section is a catalog of substrate features Code uses without modification. Anyone extending Code should look here first before adding new infrastructure.

|Substrate primitive                                                                         |What Code uses it for                                                                                                   |
|--------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
|`Streams_Stream::registerRelations` + `syncRelations`                                       |All attribute-based search (severity, status, etc.). No bespoke indexing.                                               |
|`Streams::fork` (substrate)                                                                 |Long-term workspace primitive; Code v0.1 ships transitional pattern until `Streams/fork` becomes a governed action type.|
|`Streams_Message::forkChain`                                                                |Message-history traversal across forks; relevant once Code switches to `Streams::fork`.                                 |
|`Streams.ontology(pluginName)`                                                              |Grounding LLM prompts in Grokers’ actual schema, not hand-written prose.                                                |
|Grokers’ `Grokers/calls` relation with line-number `weight` and `extra={ctrl,certainty,...}`|Caller traversal, ripple classification — line numbers and certainty fed into LLM prompts.                              |
|Grokers’ `Grokers/covers` relation with `orthogonalityScore` in extra                       |Test selection for `code/modify` to avoid full-suite runs.                                                              |
|Grokers’ confirmed-concepts pattern (`Grokers/conceptStatus = 'confirmed'`)                 |LLM prompt grounding in `propose/code/rewrite`.                                                                         |
|Safebox’s tool sandbox API                                                                  |All tool execution. No Protocol.* in tools.                                                                             |
|Safebox’s `Action.propose` action surface                                                   |All writes. The colon-suffixed `actionTypes` allowlist gates each tool’s authority.                                     |
|Safebox’s capability dispatcher                                                             |`Code/testRun` and `Code/branch` materialization via `Streams.fetch` on stream type matching.                           |
|Safebox’s `Streams/category` for index streams                                              |All four `Safebox/index/Code/*` streams are plain category streams.                                                     |

-----

## 17. Open Questions and Known Limitations

### Known v0.1 limitations on the Safebox side

1. **Safebox v0.37 `Streams/create` allowlist**. Safebox v0.37’s `Action.php` validates that `Streams/create` actions target a type from a known whitelist. The Code plugin creates four types not yet in that list: `Code/finding`, `Code/testRun`, `Grokers/repo`, `Safebox/workload`. Action proposals targeting these will fail until Safebox v0.38 expands the allowlist (or Safebox is patched in-place by the operator). The tools that would otherwise fail are `audit/code/finding`, `run/code/tests`, `clone/code/upstream`, and `ripple/code/callers`’s sub-workload spawn — i.e., most of v0.1.
1. **`$_materializableTypes` registry**. Safebox v0.37’s capability dispatcher dispatches `Streams.fetch` to a capability when the requested stream type is in `$_materializableTypes`. v0.1 needs `Code/testRun`, `Code/branch`, `Code/workspace`, and `Grokers/repo` registered there. The capabilities exist and self-declare their `Safebox/streamTypes`; the substrate-side dispatcher needs the matching whitelist update.
1. **Workflow JSON template syntax**. The workflow JSON files use placeholder syntax like `{steps.X.Y}`, `{input.X|default:n}`, `{workload.shortId}`, and `forEach: "{steps.X.candidates}"`. This syntax is what the Safebox orchestrator should consume but hasn’t been verified against the orchestrator’s actual implementation. The DAG structure (steps, edges with conditions) is correct; the placeholder rendering may need adjustment.

### Known limitations gating on Grokers feature request tranches

The Code-to-Grokers feature request specifies four tranches of work that progressively widen what Code can correctly resolve. v0.1 ships with code that gracefully degrades when each tranche is absent.

1. **Tranche 1 — ontology promotion** (`Grokers/module`, `Grokers/class`, `Grokers/method` stream types with same-named bucket relations; rename `listensTo` → `subscribes`; new clue types). Until tranche 1 lands, Code’s tools fall back to filtering `Grokers/symbol` streams by `Grokers/type` attribute. Tools that would walk `Grokers/method` from a class to find sibling methods instead use qualName-prefix matching against the class’s qualName. Functionally correct but slower and less precise.
1. **Tranche 2 — Qbix coverage visitors** (toposort type tracking; HTML/template visitor for `data-tool`; PHP `handlers/` directory walker; `plugin.json` config visitor; Qbix hint files). Partial: per Grokers update 2, the config-connector (the plugin.json visitor) has shipped, materializing handlers from `handlersAfterEvent`/`handlersBeforeEvent`/`handlersValidateEvent` registrations as `Grokers/handles` and `Grokers/activates` edges with phase metadata. The remaining tranche-2 items are still pending. Until they land:
- “Find all activations of a tool” misses `data-tool` HTML attributes — typically 30-50% of activation sites in a UI-heavy plugin
- “Update all callers” silently misses dynamic dispatch sites (`call_user_func`, `array_map`, JS bracket calls)
- JS `Q.Event` subscriptions on tool instances may not resolve to canonical hook paths
   
   Tools that would consume these emit a `note` field in their output explaining what’s missing; consumers should surface this to operators.
1. **Tranche 3 — JS event resolution and audit-quality aggregators** (JS event property-path resolver; multi-pass cross-statement reasoning; `moduleAggregate` and `eventGraph` aggregators). Until tranche 3 lands, JS `Q.Event` subscriptions on tool instances may not resolve to canonical extern paths; the `code/audit` workflow has no `moduleAggregate` to query for “riskiest module” findings.
1. **Tranche 4 — markdown visitor**. `code/document/repo` workflow (not in v0.1 anyway) is gated on this.

### Other substrate evolution items

1. **`Streams::updateRelations` SHIFT MODE patch.** Code’s `shiftLineNumbers` PHP method (called by the `Code/lineShift` after-handler) uses string weight values `"+N"` / `"-N"` on `Streams::updateRelations` to issue bulk SQL UPDATEs instead of per-row updates. This requires the Streams plugin to merge the patch shipped alongside Code as `Streams-updateRelations-shift-patch.php`. The patch is backwards-compatible (literal numeric weights still produce the original Cartesian-product insert-or-overwrite). Until the Streams plugin merges it, `Code::shiftLineNumbers` throws on the first `updateRelations` call and the workflow’s terminal step records the failure in the ack message — the workflow itself continues (since shift is advisory) but with a longer staleness window until regrok arrives.
1. **`Streams/fork` as a governed action type.** Once Safebox adds it, `clone/code/repo` switches to use it for proper fork-chain semantics. v0.1’s transitional message-driven approach works in the meantime.
1. **Workspace cascade in `Streams::related`.** The Streams plugin’s 1.3.1 roadmap includes workspace-aware relation cascading. When this lands, Code’s tools that walk Grokers/calls etc. on a workspace repo automatically see edges from the source repo without re-creating them.
1. **Grokers reanalysis as a workload.** `refresh/code/comprehension` posts an advisory `Code/regrokRequested` message and runs the Grokers pattern aggregator (no LLM, no API cost — pure DB aggregation from the existing `Grokers/calls` relation extras). Full per-symbol re-comprehension is still advisory until Grokers ships its workload-based reanalysis API.
1. **`Protocol.System` in the standard executor build.** Today, capabilities fall back to PHP-side handlers via `Q_Utils::sendToNode`. If `Protocol.System` lands in the standard sandbox, the capabilities run inline with full audit visibility through the sandbox’s invocation hash.
1. **Per-symbol governance gates.** For high-stakes symbols (security paths, public APIs), the workflow should require a `Safebox/judgment` vote on the contract classification before ripple proceeds. The infrastructure exists in Safebox but Code doesn’t yet ship a finding-class judgment.
1. **`Grokers::contextSiblings` from inside the Q.Sandbox.** Per Grokers Update 4, `contextSiblings` is exposed as a Node export (`Grokers/classes/Grokers/harness/tools`) and a PHP static (`Grokers::contextSiblings`). Both surfaces are outside the Safebox sandbox, which doesn’t have arbitrary `require()`. Until a sandbox-callable primitive exists (a Safebox capability returning the JSON, or a new sandbox API method `Grokers.contextSiblings`), Code’s `propose/code/rewrite` walks the underlying relations (class siblings via `Grokers/method`, pattern co-members via `Grokers/pattern`, `Grokers/associated` peers) directly. This duplicates Grokers’ definition of “what counts as a relevant sibling”; will swap to the public call when the sandbox-callable surface lands.

### Architectural separation: graph context belongs in Safebots

A note for downstream consumers (Safebots team, future tooling): Code v0.1’s tools assemble their own graph context — `propose/code/rewrite` fetches conventions, concepts, ontology, framework patterns, and Grokers-detected codebase patterns directly. The full layered model is documented in §18.11 (`Safebox/convention` for human-authored prose, `Grokers/pattern` for machine-derived conventions, `Grokers/concept` for confirmed framework concepts). This is appropriate for already-resolved-coordinates actions (the user already specified which symbol to modify; the graph walk is mechanical follow-through). It is NOT appropriate for intent-resolution work like “find what events a new tool should subscribe to given this description.” That class of work — interpreting freeform user intent into specific symbol coordinates and assembling LLM context across multiple workflows — belongs to Safebots, not Code. Code provides action verbs and structured tools; Safebots calls them with already-resolved arguments. See `Code-for-Safebots.md` (the parallel handoff document) for the integration contract.

These are the natural integration points as the substrate evolves. Items 1–3 are required for v0.1 to actually run end-to-end on Safebox; items 4–7 are graceful-degradation cases that progressively widen as Grokers tranches land; item 8 is required for full bulk-shift performance (the workflow degrades gracefully without it); items 9–14 are quality-of-life improvements that don’t block correctness.

-----

## 18. Workflow Signaling and Resume Semantics

This section answers six questions raised by the Safebots team on how to coordinate during long-running workflows and ripple cascades. The behavior described is what Code v0.1 implements; the message vocabulary lets Safebots translate workflow execution into real-time chat UX.

### 18.1 Cascade depth and recursion semantics

The `code/modify` workflow accepts `depthRemaining` as input. The default is `3`; the hard ceiling enforced in `config/plugin.json` is `5`. Behavior at each step:

- When `code/modify` runs with `depthRemaining = N` and verify-contract returns class C, ripple-callers spawns sub-workloads, each receiving `depthRemaining = N - 1`.
- When a sub-workload’s `depthRemaining` reaches `0`, the next ripple invocation does not spawn further sub-workloads. Instead it posts a `Code/depthExhausted` message listing the unprocessed callers and returns `depthCapReached: true`.
- The terminal step branches: when `depthCapReached == true`, the workflow ends with `Code/sprintCompletedWithFindings` (the findings list includes the depth-exhausted note); otherwise `Code/sprintCompleted` (full success).

There is no per-invocation symbol-count cap distinct from depth in v0.1. Depth is the only cascade limit. If a single function has 100 callers and it’s modified at depth 3, all 100 are processed at depth 3 (then their callers at depth 2, etc.). This is a deliberate v0.1 simplification — production usage will surface whether a symbol-count cap is needed.

### 18.2 Progress message vocabulary

Code workflows post the following message types on the workload stream during execution. Safebots subscribes to `Streams/message` on the workload (or to `Code/*` patterns specifically) and translates each into chat output.

|Message type                      |Fired by                                                          |Instructions schema                                                                                                                                                                  |
|----------------------------------|------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`Code/rewriteProposed`            |post-rewrite-proposed step (after verify)                         |`{ symbolStreamName, verifyClass, depthRemaining, changeNotes }`                                                                                                                     |
|`Code/rewriteApplied`             |apply/code/rewrite tool                                           |`{ symbolPublisherId, symbolStreamName, sourceLength, docstringLength, verifyClass, modifiedLines?, testsPassedSoFar?, depthRemaining? }` (cross-posted on symbol and workspace repo)|
|`Code/rippleStarted`              |ripple/code/callers tool                                          |`{ parentSymbol, callerSymbols, callerCount, depthRemaining }`                                                                                                                       |
|`Code/rippleEnded`                |ripple/code/callers tool                                          |`{ parentSymbol, modifiedCallers, skippedCallers, failedCallers, depthRemaining, spawnedSubWorkloadCount }`                                                                          |
|`Code/depthExhausted`             |ripple/code/callers tool when depth hits zero                     |`{ parentSymbol, unprocessedCallers, depthRemaining: 0, suggestedFollowupAction }`                                                                                                   |
|`Code/checkpointRequested`        |request/code/checkpoint tool, or ripple when `pauseAtClassCRipple`|`{ parentSymbol, plannedCallers, callerCount, depthRemaining, note, requestedTime }`                                                                                                 |
|`Code/testRunStarted`             |post-tests-started step                                           |`{ testCount, repoStreamName }`                                                                                                                                                      |
|`Code/testRunCompleted`           |post-tests-completed step                                         |`{ passed, failureSummary, testRunStreamName }`                                                                                                                                      |
|`Code/sprintCompleted`            |complete/code/sprint with `outcome:"success"`                     |`{ workloadName, changeDescription, completedAt }`                                                                                                                                   |
|`Code/sprintCompletedWithFindings`|complete/code/sprint with `outcome:"partial"`                     |`{ workloadName, changeDescription, completedAt, findings, findingCount }`                                                                                                           |
|`Code/sprintAborted`              |complete/code/sprint with `outcome:"aborted"`                     |`{ workloadName, changeDescription, completedAt, abortReason }`                                                                                                                      |

All progress messages are posted on the **workload stream**, whose coordinates are passed to tools as `workloadPublisherId` and `workloadStreamName` from the orchestrator’s `{workload.publisherId}` / `{workload.streamName}` template variables. Workspace-repo cross-posts of `Code/rewriteApplied` are additional, to support summarize-changes without walking every symbol.

The `Safebox/tool/post/code/progress` tool is the mechanism workflow JSONs use to fire any progress message at a step boundary. It’s a transparent proxy — workflow JSON specifies the messageType and instructions; the tool does the propose-message action. Workflows can add new progress messages by adding a step that calls `post/code/progress`.

### 18.3 Workflow status states and recoverability

The workload stream’s `Safebox/status` follows the substrate’s standard lifecycle: `pending` → `running` → terminal. The terminal state for Code workflows is signaled by which sprint-completion message fires:

|Terminal message                  |Workspace state                               |Recoverability                                                                                              |
|----------------------------------|----------------------------------------------|------------------------------------------------------------------------------------------------------------|
|`Code/sprintCompleted`            |Valid; gitPush handler triggers               |Workflow done; nothing to recover                                                                           |
|`Code/sprintCompletedWithFindings`|Valid; gitPush does NOT auto-trigger          |Safebots reviews findings, optionally schedules followup workflows; current workspace can be merged manually|
|`Code/sprintAborted`              |Possibly invalid (test failure left mid-state)|Safebots should not merge; user should inspect workspace; failed tests visible via `Code/testRun` stream    |

Failures during ripple processing — when individual caller classifications return `class: 'unknown'` because a stream couldn’t be retrieved or the LLM verdict was unparseable — do NOT abort the cascade. Each failed caller is captured in `failedCallers` of the `Code/rippleEnded` message. The workflow continues; Safebots surfaces these as items needing review.

If 8 of 12 caller rewrites succeed and the 9th sub-workload aborts (its tests fail), the parent workflow’s state is determined by:

- The 8 successful rewrites are present in the workspace (already committed via apply/code/rewrite).
- The 9th sub-workload’s terminal message is `Code/sprintAborted` on its own workload, but it doesn’t propagate to abort the parent.
- The parent workflow’s tests run on the workspace state (containing 8 successful rewrites). If those pass, the parent terminates with `Code/sprintCompletedWithFindings` and the failed sub-workload is referenced as a finding.
- Safebots can resume by scheduling a fresh `code/modify` against the failed caller (with revised changeDescription) using the same workspace.

### 18.4 Checkpoint mechanism for interactive review

The `code/modify` workflow accepts an optional `pauseAtClassCRipple: true` input. When set, `ripple/code/callers` posts `Code/checkpointRequested` instead of spawning sub-workloads, sets `Safebox/awaitsCheckpoint: true` on the workload, and returns `checkpointRequested: true`.

The workflow’s edge from `ripple-callers` to `notify-subscribers` is gated on `checkpointRequested != true`. When checkpoint is requested, the workflow halts at the ripple step.

Safebots resume protocol:

1. Detects the awaiting state via `Safebox/awaitsCheckpoint` attribute on the workload.
1. Reads the checkpoint details from the `Code/checkpointRequested` message (planned callers, depth, etc.).
1. Surfaces to the user via chat.
1. On approval: schedules a new `code/modify` workflow per caller (or one parent that handles them), with adjusted depth budget and `pauseAtClassCRipple: false` to let the cascade run unchecked once approved.
1. On rejection: closes the parent workload as cancelled (sets `Safebox/status` to `aborted`).

For non-checkpoint flows (`pauseAtClassCRipple: false` or unset), the cascade proceeds without pause and Safebots surfaces a simpler “rippling N callers, depth M” status without interactive gates.

### 18.5 notify-subscribers timing

`notify/code/subscribers` runs synchronously within the workflow when verify-contract returns class C. The workflow’s edge from `notify-subscribers` to `post-tests-started` waits for the notify step to complete (i.e., for all `Streams/message` actions to be proposed).

However, the notification messages are themselves async: the proposed `Streams/message` actions get queued, the substrate delivers them, and downstream subscribers pick them up at their own pace. The workflow does not wait for downstream acknowledgment.

The `notificationActionIds` output contains the action IDs of each proposed notification. Safebots can poll their state independently via `Streams_Action::fetchByActionIds` if it needs to confirm downstream propagation before signaling final completion to the user.

### 18.6 Resume after disconnect

The workload stream’s `Streams/message` log contains every Code/* progress message fired during execution. After a Safebots disconnect:

1. Safebots fetches the workload stream and reads its `Safebox/status` attribute to determine if execution is in-progress, paused (awaiting checkpoint), or terminal.
1. If not terminal, Safebots fetches the workload’s message log via `Streams_Message::fetchFor()` to enumerate progress messages already fired.
1. Safebots reconstructs which messages have been shown to the user (it tracks this in its own chat-side state) and replays the unshown ones.
1. Safebots subscribes to the workload’s message stream for new messages (via the standard Streams/message subscription mechanism).

All `Code/*` progress message instructions are stable on replay — they reference stream coordinates and counts, not transient state. Replay is idempotent.

If Safebots was tracking a workflow that hit checkpoint while disconnected, the `Safebox/awaitsCheckpoint` attribute survives — the workflow is still paused, ready for Safebots to surface the checkpoint when it reconnects.

### 18.7 What this enables

With this signaling vocabulary in place, Safebots can:

- Stream progress as a chat thread: “Rewrote `Streams_Stream::beforeSave` (Class C, 12 callers identified). Processing callers… Rewrote 8 of 12. Verifying… Tests passing… Depth limit reached at depth 3; 4 callers in deeper graph saved as findings. Sprint complete with findings — review?”
- Offer interactive checkpoints: “About to ripple 12 callers, depth 3. Proceed? [Approve | Reject | Review]”
- Recover from disconnect cleanly via the message log replay
- Surface partial-success states distinctly from full success and from failure
- Translate failed caller classifications into explicit findings the user can act on

### 18.8 Line-number bulk-shift after edits

When `apply/code/rewrite` updates a `Grokers/symbol`’s content and the new source has a different line count than the old, every symbol *defined later in the same file* now sits at a stale position-in-file, and every `Grokers/calls`/`Grokers/reads`/`Grokers/writes`/etc. relation originating in those later symbols carries stale `weight` (= line number) and `extra.line`.

The substrate convention used here: **files are indexed as `Streams/file` streams** (the existing Streams plugin type), and each symbol relates TO its containing file via the `Streams/file` relation type with `weight` = the symbol’s start line in the file. Walking `Streams/file` from a file stream with `isCategory=true` returns the symbols inside, ordered by line.

Code’s modify workflow inserts a `shift-line-numbers` step immediately after `apply-rewrite`. The step posts a `Code/lineShift` message which the PHP after-handler `Code/after/Streams_message_Code_lineShift` picks up and uses to invoke `Code::shiftLineNumbers`. The shift implementation uses the **SHIFT MODE of `Streams::updateRelations`** — string weight values of form `"+N"` or `"-N"` trigger a single SQL UPDATE per relation type that adds the signed delta to every matching row’s weight, with `minWeight` restricting the update to relations at or after the shift point.

Concretely:

1. Walk `Streams/file` from the file stream (one query, sorted by weight) to enumerate symbols-in-file.
1. Partition: the modified symbol gets handled separately (verb-relations cleared, not shifted); other symbols at line ≥ `shiftStartLine` go into the shift batch.
1. Bulk-shift `Streams/file` relation weights via one `Streams::updateRelations` call with `array('Streams/file' => '+30')` and the from-symbols as IN-list filter.
1. For each shifted symbol, walk its outgoing verb-relations (`Grokers/calls`, `Grokers/reads`, etc.) once to collect target stream coordinates, then call `Streams::updateRelations` per verb-type to bulk-shift the matching subset.
1. For the **modified symbol itself**, clear all outgoing verb-relations via `Streams::unrelate` (not shift) — relations from the old AST aren’t recoverable by mechanical shift; regrok will emit fresh ones.

After completion, posts a `Code/lineShifted` ack message with counts (`shiftedSymbols`, `shiftedRelations`, `clearedRelations`, `errorCount`).

**This is best-effort, advisory.** Authoritative line numbers come from Grokers’ regrok of the file (triggered later via `refresh/code/comprehension`). The bulk-shift narrows the staleness window — typically seconds — between `apply-rewrite` and regrok during which other workflow steps (especially `ripple/code/callers`, which reads relation extras for call-site context) would otherwise see stale line numbers.

**Known limitation: `extra.line` is not updated by SHIFT MODE.** The bulk SQL UPDATE operates on the `weight` column directly; it cannot easily rewrite the JSON-encoded `extra` field per-row in pure SQL. For consumers that read line number from `weight` (the substrate-canonical place), this is a non-issue. For consumers that read `extra.line` (which is redundant with `weight` in the normal case), values are stale until Grokers’ regrok runs and re-emits both consistently. `ripple/code/callers` reads `extra.line` for context — its values may be off-by-N briefly between shift and regrok.

**Multi-symbol concurrent edits in the same file** are NOT handled in v0.1: if two `code/modify` workflows run in parallel on different symbols in the same file, their shifts can race. v0.1 expects Safebots to serialize multi-symbol same-file modifications.

This implementation requires `Streams::updateRelations` to support SHIFT MODE (string weight values). The patch ships as `Streams-updateRelations-shift-patch.php` for the Streams plugin team to merge. Until merged, `Code::shiftLineNumbers` will throw on the first `updateRelations` call and the workflow’s terminal step will record the failure in the ack message — the workflow itself continues (since shift is advisory) but with a longer staleness window until regrok arrives.

### 18.9 Docblock discrepancy detection (forthcoming)

Per Code-to-Grokers Update 1 Item 2, Grokers’ tier-2 grok pass will emit `Grokers/clue` streams of type `possible-doc-discrepancy` when a method’s docblock semantically disagrees with the LLM-derived contract (preconditions, postconditions, sideEffects, invariants).

Code consumes these via the `code/auditClues` workflow:

```js
// Find all docblock discrepancies of medium+ severity
{
  workflow: "Safebox/workflow/code/auditClues",
  inputs: {
    repoPublisherId, repoStreamName, communityId,
    clueType: "possible-doc-discrepancy",
    minSeverity: "medium",
    findingType: "docDiscrepancy",
    descriptionPrefix: "Docblock disagrees with derived contract:"
  }
}
```

Each surfaced finding can be fixed by the existing `code/document` workflow, passing the discrepancy’s description as `docInstructions`. Safebots can offer a one-click “fix all docblock discrepancies” that schedules these in batch.

The `scan/code/clues` tool that powers `auditClues` also accepts other `clueType` values (`possible-dynamic-tool-instantiation`, `possible-dynamic-dispatch`, etc.) for parallel use cases.

### 18.10 Pattern injection in propose/code/rewrite

Per Grokers Update 4, the pattern aggregator runs automatically after indexing and emits `Grokers/pattern` streams with structural pattern types: `naming-convention`, `naming-convention-php`, `naming-convention-js`, `indentation`, `css-namespace-prefix`, `css-var-naming`, `component-file-structure`, `semantic-pattern`. Each carries `Grokers/patternScope` (e.g. `repo` or `module:Streams`), `Grokers/patternValue` (the convention as a string), `Grokers/patternTemplate` (JSON for structural patterns), and `Grokers/patternCoverage` (fraction of similar code that conforms, 0.0–1.0).

`propose/code/rewrite` walks the repo’s `Grokers/pattern` streams, filters to relevant types and scopes, and injects each as a one-line constraint into the LLM system prompt. Filtering rules:

- `naming-convention-php` only included when the symbol’s language is PHP; `naming-convention-js` only when JavaScript.
- Module-scoped patterns only included when the symbol’s `Grokers/module` matches the pattern’s scope (or is a sub-module).
- Patterns with coverage below 50% are dropped — they’re not actually conventions, just noise.
- `call-arg-distribution` patterns are excluded entirely — too granular for a propose prompt; relevant for verify/audit instead.

Each included pattern surfaces in the prompt as:

```
- **naming-convention-php** (module:Streams, 87% coverage): Class names follow Streams_Pascal style; method names are camelCase
- **indentation** (repo, 96% coverage): tabs, width 1
```

The LLM gets these alongside concepts and conventions, treating them as constraints on the rewrite. Coverage % gives the model a soft signal: 96% means follow it; 60% means there’s a real exception class.

Patterns are read fresh per workflow run, not cached. The aggregator runs after every regrok per Update 4, so patterns stay current.

### 18.11 Conventions, patterns, and concepts: layered context for code authoring

When `propose/code/rewrite` builds the LLM prompt, it pulls from three different layers of codebase knowledge. Each layer has different authorship, currency, and authority. Knowing which is which matters for picking the right one when extending Code’s tools.

**Layer 1: `Safebox/convention`** — human-authored prose, prescriptive.

Stream type owned and registered by the Safebox plugin (substrate-level). Multiple plugins write to it, scoped by domain via the `Safebox/conventionDomain` attribute. Code seeds three baseline streams during install (`Safebox/convention/code/php`, `Safebox/convention/code/js`, `Safebox/convention/code/python`) carrying `Safebox/conventionDomain: 'code'`. Safebots will eventually seed chat-domain conventions (`Safebox/convention/chat/voice`, etc.) carrying `Safebox/conventionDomain: 'chat'`.

Each stream’s `content` is markdown prose authored or approved by a human. Attributes:

- `Safebox/conventionDomain` — `'code' | 'chat' | 'governance' | 'cross-domain'`
- `Safebox/approved` — `'true' | 'false'` (governance signoff)
- `Safebox/severity` — `'mandatory' | 'recommended' | 'preferred'` (how strictly to enforce)
- `appliesTo` — domain-specific JSON: `{ language?, framework? }` for code, `{ channelType?, role? }` for chat
- `kind` — `'convention'` (allows mixing with other configurable streams in the same bucket)
- `source` — provenance: `'Code-builtin'`, `'operator-authored'`, `'agent-proposed'`, etc.

`propose/code/rewrite` filters by `Safebox/conventionDomain === 'code'` (implicit — it walks `Safebox/convention` related to the repo and reads `appliesTo.language` to match the symbol) and only injects conventions where `Safebox/approved === 'true'`. The prose goes into the system prompt under `## Conventions`.

Use Layer 1 for: rules that aren’t yet baked into enough code for frequency analysis to detect (“never use eval”), security/architectural rules (“controller → service → repository”), team norms with rationale, anything where human judgment about the “should” is the source of truth.

**Layer 2: `Grokers/pattern`** — machine-derived, descriptive.

Stream type owned by Grokers. Populated automatically by Grokers’ pattern-aggregator after each repo index. Each stream represents an observed pattern in the codebase, not an authored rule. Attributes carry `Grokers/patternType` (`naming-convention`, `indentation`, `css-namespace-prefix`, etc.), `Grokers/patternScope` (`'repo'` or `'module:Streams'`), `Grokers/patternValue` (the convention as a string), `Grokers/patternCoverage` (fraction of similar code that conforms, 0.0–1.0), and `Grokers/patternTemplate` (JSON for structural patterns).

`propose/code/rewrite` walks repo-scoped and module-scoped patterns, filters out low-coverage ones (< 50% — noise rather than convention), and injects each as a one-line constraint under `## Detected codebase patterns`. The coverage fraction is included so the LLM has a soft signal about how strict to be.

Use Layer 2 for: anything observable from frequency analysis — naming, indentation, file structure, CSS namespace prefixes, argument distributions. Patterns stay current automatically as the codebase evolves; conventions don’t.

**Layer 3: `Grokers/concept`** — confirmed framework concepts, definitions.

Stream type owned by Grokers. Populated by Grokers’ concept-discovery LLM pass. Each stream describes a confirmed framework concept (e.g., “Q.Tool — a registered DOM component with Qbix’s lifecycle integration”). Attributes carry `Grokers/conceptStatus` (only `'confirmed'` concepts are eligible for prompt injection), `Grokers/conceptDescription`, examples, and so on.

`propose/code/rewrite` walks the repo’s confirmed concepts and injects descriptions under `## Confirmed concepts in this codebase`. This gives the LLM glossary-level grounding so it knows what `Q.Tool` or `Streams.fetch` means without re-deriving it from source.

Use Layer 3 for: definitional context that should anchor the LLM’s understanding of the codebase’s vocabulary, distinct from rules about how to write code.

**Decision rule when extending Code’s tools**

|If you want to express…                                             |Use this layer                                                                                                                                               |
|--------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
|“Don’t ever use eval”                                               |`Safebox/convention` (prescriptive, security)                                                                                                                |
|“This codebase uses 4-space indents”                                |`Grokers/pattern` (machine-derived)                                                                                                                          |
|“Streams_Stream is Qbix’s substrate stream object”                  |`Grokers/concept` (definitional)                                                                                                                             |
|“Class methods should follow Streams_Pascal”                        |Both work — convention if it’s a hard rule, pattern if it’s an observation. Prefer pattern (auto-updates as codebase evolves) unless human authority matters.|
|“All security-sensitive functions must validate inputs”             |`Safebox/convention` (rule with rationale; not derivable from frequency)                                                                                     |
|“94% of methods in Streams plugin use camelCase”                    |`Grokers/pattern` (this is exactly what it’s for)                                                                                                            |
|“When firing Q::event, prefer the third-arg phase form for new code”|`Safebox/convention` (recommended practice; codebase mixes both forms)                                                                                       |

**Why three layers rather than one**

The naive question is whether all three could fold into a single “code-context” stream type. The answer is no — they have different writers, different lifecycles, and different correctness criteria.

Conventions are slow-moving and human-authored — they last as long as the rule does, regardless of code churn. Patterns are fast-moving and code-derived — they update on every regrok. Concepts are mid-frequency and LLM-derived — they update when an analyzer pass confirms a new pattern as a named concept.

Mixing them would mean the layer with the slowest update cadence (conventions) would either get stale-stamped on every regrok, or would silently fall behind the others. Keeping them separate lets each layer maintain currency on its own schedule, and lets propose/code/rewrite combine them with appropriate weights at prompt time.

**Cross-domain note for Safebots**

When Safebots reads `Safebox/convention` for chat behavior (brand voice, persona, internal-process), it filters by `Safebox/conventionDomain === 'chat'`. Code’s `Safebox/conventionDomain === 'code'` streams don’t appear in that filter, and vice versa. Both share the namespace; neither sees the other’s defaults. The Safebox plugin owns the type registration and the schema; consumers (Code, Safebots, future plugins) seed and read their own subtree by domain.
