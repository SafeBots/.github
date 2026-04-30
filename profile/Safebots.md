# Safebots Plugin — Architecture & Implementation Reference

> **Audience**: LLM implementers and developers building or extending the Safebots
> plugin for Qbix.
> Safebots adds *configurable behaviors* to existing users — a community's chat
> publisher gets inline suggestion generation, artifact authorship, and deliberative
> coordination as features of the publisher, not as a separate bot entity.
> This document assumes familiarity with Safebox.md.

---

## Table of Contents

1. [Overview & Philosophy](#1-overview--philosophy)
2. [Plugin File Structure](#2-plugin-file-structure)
3. [Stream Types](#3-stream-types)
4. [Bots — The Core Abstraction](#4-bots--the-core-abstraction)
5. [The Publisher-Augments Model](#5-the-publisher-augments-model)
6. [Attribution — `from @source via @agent`](#6-attribution--from-source-via-agent)
7. [Safebots/suggest — Inline Suggestions](#7-safebotssuggest--inline-suggestions)
8. [Safebots/goal — Goal Streams](#8-safebotsgoal--goal-streams)
9. [Safebots/dialog — Dialog Streams](#9-safebotsdialog--dialog-streams)
10. [Storefront Greeting](#10-storefront-greeting)
11. [Chat Orchestration — Replies, Edits, and Working Context](#11-chat-orchestration--replies-edits-and-working-context)
12. [Relations as Emissions — Channels](#12-relations-as-emissions--channels)
13. [Artifact Lifetime & Refcount Model](#13-artifact-lifetime--refcount-model)
14. [Voting, Versions & Promotion](#14-voting-versions--promotion)
15. [Safebots/goal/chat — JS Chat Extension](#15-safebotsgoalchat--js-chat-extension)
16. [Capability & Tool Generation from Chat](#16-capability--tool-generation-from-chat)
17. [PHP Implementation Guide](#17-php-implementation-guide)
18. [JavaScript Implementation Guide](#18-javascript-implementation-guide)
19. [Node.js Implementation](#19-nodejs-implementation)
20. [UI Reference](#20-ui-reference)
21. [Install & Seed](#21-install--seed)
22. [Appendix: Configuration Reference](#22-appendix-configuration-reference)

---

## 1. Overview & Philosophy

Safebots is the **behavior layer** on top of Safebox. Where Safebox handles
infrastructure — stream materialization, workflow execution, action governance,
economics — Safebots handles the LLM-guided behaviors that communities and users
attach to their own streams: inline suggestions in chats, artifact authorship,
deliberative coordination, policy-summarization, auto-voting.

### The core insight — composition, not inheritance

Earlier drafts of this plugin treated bots as a separate kind of user, with
their own identity, access rows, subscriptions, and governance. That model
introduced duplication at every layer — duplicate access rows, duplicate
identity keys, duplicate billing contexts, duplicate attribution fields.
Each layer kept asking "who is this?" and the answer kept wanting to be
"the community."

The current design uses **composition**. A user (community or individual) can
publish `Safebots/bot/*` streams declaring behaviors they want active on
their own streams. When events happen on a user's streams, the plugin's hooks
consult the publisher's bot configuration and dispatch accordingly.
There is no bot user. The community *is* its own bot, in the sense that
bot-assisted activity on the community's streams runs under the community's
identity.

### The publisher-augments model

When a message is posted in a chat, Safebots dispatches to the **chat's
publisher's** behaviors. Not the poster's behaviors. Not the reader's
behaviors. The publisher's. This matches the Telegram inline-bot pattern:
inline bots are a property of the chat, enabled by the chat's owner; they
augment what participants can produce. The same pattern here. A community
that enables the suggest behavior on its chats has inline suggestions
available in those chats. A community that disables it doesn't.

Individual users who want personal bots publish their own bot streams:

```
{communityId} / Safebots/bot/suggest          ← community's inline-suggest config
{communityId} / Safebots/bot/greet            ← community's onboarding-message config
{alice}       / Safebots/bot/daily-summary    ← Alice's personal daily recap
```

The shape is identical; the difference is only the publisher.

### What Safebots is, in two paragraphs

Safebots doesn't create bot users. Safebots is a mechanism for existing users
to declare automated behaviors on their own streams. A chat's behaviors are
the chat publisher's behaviors. A user's behaviors are the user's behaviors.
There is no bot user to provision. There are no Safebots-specific access
grants in the installer, because the publisher already has full access to
its own streams.

When an event happens on a stream — a message is posted, a stream is created,
a relation is proposed — the Safebots dispatcher looks up the stream's
publisher, finds the publisher's active behaviors matching the event, and
invokes each matching behavior's referenced `Safebox/tool`. The tool runs
inside the Safebox sandbox with the publisher's billing context, makes
whatever LLM or capability calls it needs, and either writes directly (if
the publisher's access permits) or emits `Actions.propose(...)` for
governance. The result flows back to the triggering caller or appears as
normal chat messages, attributed to the publisher.

### Bots as streams — the substrate does the work

The composition model is structural, not cosmetic. A bot, in this system,
*is* a stream — or more precisely, a small composition of them. The
bot configuration is a `Safebots/bot/*` stream. The policies
governing when the behavior writes autonomously versus proposes for review
are `Safebox/policy/*` streams. The tools the behavior references are
`Safebox/tool/*` streams. The keys authorizing amendments to any of these
are entries in `Safebox/keys/*` streams. Every structural element of what a
community informally calls "its bot" is, formally, a stream in the Qbix
substrate.

This matters because Qbix Streams — described in detail as a
[graph database hiding in plain sight](https://community.qbix.com/t/qbix-streams-as-a-graph-database/769)
— already handles most of what bot governance would otherwise need to
reinvent:

- **Access control** comes from `Streams_Access` rows, same as any other
  stream. Who can read, edit, or admin a bot configuration is a
  normal Qbix permissions question.
- **Collaboration** comes from the same discussion-then-decision flow
  Qbix runs for documents. Multiple community members contribute to a
  behavior's development; amendments go through `Actions.propose`.
- **Relationship voting** comes from the substrate's built-in vote
  machinery on relations. Two candidate versions of a behavior can run
  in shadow; the community votes on which produces better output; the
  winner gets promoted through the same curatorial process used for any
  other artifact.
- **Forking with provenance** comes free. A community forks another
  community's audited behavior, re-audits only the diff, and adopts the
  customized version under its own governance.
- **History and audit** come free. Every change to a behavior is a
  message on the behavior's stream, replayable, attributable.
- **Federation** comes free. A bot in Community A can reference a
  tool published by Community B, each side under its own governance.

None of this required building bot-specific infrastructure. Fifteen
years of substrate maturity does the work that would otherwise require
fifteen years of AI-specific infrastructure reinvention. The category
error most AI-agent architectures make is assuming bots need their own
identity system, their own memory store, their own governance model,
their own tool-use protocols. They don't. Make the bot a stream, attach
behavior to it, let the substrate handle the rest.

For the longer-form take on why this architecture works — and how it
maps to the three Chabad faculties of Chochmah, Binah, and Da'at
(Grokers, Safebots, Safebox) — see the
[wisdom essay](https://safebots.ai/wisdom.html).

### How it composes with Safebox

|Safebox does                                    |Safebots does                              |
|------------------------------------------------|-------------------------------------------|
|Workflow/step/task orchestration                |Define bot streams and dispatch hooks |
|Stream materialization, caching, commissions    |Invoke bot tools via Safebox sandbox  |
|`Actions.propose` governance pipeline           |Compose behavior output into `Actions.propose` payloads when governance is needed |
|`Streams.fetch` / `Streams.related` sandbox API |Use these in behavior tool code            |
|Protocol.* (LLM, SMTP, Telegram, etc.)          |Behavior tools call these for their work   |
|Sandboxed tool execution with sha256 attestation|Reference audited tools from bot streams|
|Billing via community's Safebux balance         |Inherit — the community is its own payer   |

Safebots is the small plugin. Most of the architectural weight is in Safebox;
Safebots is the application layer that configures how a publisher's streams
get augmented with LLM-guided behaviors.

### The composition at runtime

What actually happens when a community's suggest behavior fires on a chat:

```
        event on Streams/chat stream (e.g. suggest-request)
                           │
                           ▼
        ┌─────────────────────────────────────────┐
        │  Safebots dispatcher (in PHP)           │
        │  looks up stream.publisherId's          │
        │  Safebots/bot/suggest              │
        └─────────────────────────────────────────┘
                           │
                           ▼  (sendToNode reply-mode IPC)
        ┌─────────────────────────────────────────┐
        │  Safebox sandbox (in Node)              │
        │  loads behavior.tool (sha256-verified)  │
        │  runs tool with event payload           │
        └─────────────────────────────────────────┘
                           │
                ┌──────────┴──────────┐
                ▼          ▼          ▼
         Streams.fetch  Runtime.llm   Action.propose
         (read)         (think)       (write, if governance approves)
                │          │          │
                └──────────┴──────────┘
                           │
                           ▼
        ┌─────────────────────────────────────────┐
        │  Result returns to PHP → client,        │
        │  OR appears as a new chat message       │
        │  authored under publisher identity      │
        └─────────────────────────────────────────┘
```

The pattern is the same for any behavior: event hits a stream, dispatcher
looks up the publisher's matching behavior, tool runs in the sandbox,
result flows back. There is no bot user in the flow. There is no special
access path. There is no Safebots-specific governance. The substrate's
existing primitives do the work.

---

## 2. Plugin File Structure

```
Safebots/
├── README.md                                    # this file
├── config/
│   └── plugin.json                              # stream types, routes, event hooks
├── scripts/Safebots/
│   └── 0.1-Streams.mysql.php                    # seed: platform goals, default bots,
│                                                #       Safebox activation policy
├── classes/
│   ├── Safebots.php                             # displayNameOf, modelFor,
│   │                                            # ensureActivation, register/unregisterBotPublisher,
│   │                                            # defaultActionTypes
│   ├── Safebots.js                              # Node-side addMethod handlers — Safebots/invokeTool dispatches through Orchestrator.runTask
│   └── Safebots/
│       ├── Bot.php                              # bot enumeration + dispatch + seedDefaults
│       ├── Goal.php                             # goal-prompt interpolation
│       └── Dialog.php                           # dialog stream creation + completion
├── handlers/
│   └── Safebots/
│       ├── after/
│       │   ├── Q_Plugin_install.php             # bootstrap seed on install
│       │   └── Streams_postMessages.php         # universal dispatcher hook
│       ├── suggest/response.php                 # thin PHP dispatcher → Node (reply mode)
│       ├── dialog/response.php                  # create dialog stream
│       ├── dialog/complete.php                  # mark dialog done
│       ├── goal/respond/response.php            # user posted chat → trigger LLM response
│       └── goal/message/response.php            # Node calls back with LLM output
├── text/Safebots/content/en.json
└── web/
    ├── css/Safebots.css
    └── js/
        ├── Safebots.js                          # browser bootstrap + chat extension registry
        └── tools/
            ├── goal/chat.js                     # goal-directed chat extension
            └── safebots/chat.js                 # inline-suggest extension + storefront
```

**Handler registration.** Two event hooks registered via `plugin.json`, no
direct per-stream-type file autoload required:

```json
{
  "Q": {
    "handlersAfterEvent": {
      "Q/Plugin/install":     ["Safebots/after/Q_Plugin_install"],
      "Streams/postMessages": ["Safebots/after/Streams_postMessages"]
    }
  }
}
```

`Streams/postMessages` is the universal dispatcher subscription — fires
once per substrate message batch, filters in-memory against each
publisher's active bots and their handler maps (see §4 "How dispatch
works"). Previous iterations registered per-event handlers for every
message type any bot might care about; collapsing to the single
`postMessages` hook removes that coupling and lets communities author
bots reacting to new message types without touching plugin configuration.

---

## 3. Stream Types

|Type                |Shape                                                           |Publisher                       |
|--------------------|----------------------------------------------------------------|--------------------------------|
|`Safebots/bot`      |Declares a bot: one or more handlers, each a `(message type → tool)` mapping|user (community or individual)  |
|`Safebots/goal`     |Interpolated system-prompt template for goal-directed chats     |user (or `Users::communityId()` for platform-default goals)|
|`Safebots/dialog`   |In-progress guided-conversation session                         |community                       |
|`Safebots/artifact` |Content being produced or revised in a chat                     |community                       |

Platform-default goals are published under `Users::communityId()` (the hosting
app's community) so that all communities on the deployment can reference them
via the community-fork-then-platform-fallback pattern in
`Safebots_Goal::findGoalStream`.

### `Safebots/bot` attributes

A bot is a stream whose attributes carry its whole configuration. Two
attributes define everything:

```
Safebots/active   : true | false                      ← whole-bot on/off
Safebots/handlers : {                                  ← map keyed by message type
  "Streams/created": {
     "streamType": "Streams/chat",
     "tool":       "Safebox/tool/greet/chat",
     "active":     true
  },
  "Streams/relatedTo": {
     "streamType": "Streams/chat",
     "fromType":   "Safebots/goal",
     "tool":       "Safebox/tool/greet/goal",
     "active":     true
  }
}
```

The handlers map is keyed by a substrate stream-message type (the `type`
column of `streams_message`, e.g. `Streams/created`, `Streams/joined`,
`Streams/chat/message`, `Streams/relatedTo`, `Safebox/action/approved`).
Each entry declares a single handler with these fields:

| Field        | Purpose |
|--------------|---------|
| `tool`       | `Safebox/tool/{path}` stream reference; audited code the sandbox runs |
| `active`     | Per-handler on/off — lets a bot keep a handler defined but quiesced |
| `streamType` | Filter: only fire when the affected stream matches this type |
| `fromType`   | Filter for relate-messages: the type of the OTHER stream involved |
| `suppressSelfAuthored` | Set to `true` to prevent this handler from firing on messages authored by the stream's publisher itself — needed only when the handler's tool writes the same message type it reacts to (to break infinite dispatch loops) |

Both the bot's `Safebots/active` AND the matching handler's `active` must
be true for a handler to fire — the two flags represent different
concerns. Whole-bot disable lets a community pause all of a bot's
behaviors without editing the handlers out. Per-handler disable lets a
bot author keep a handler defined and readable while turning off just
that one reaction.

Booleans are stored as actual booleans (`true`/`false`), not strings.
Qbix's attribute system round-trips native JSON types; `getAttribute`
returns the type it was set with.

### Bots index

Bots are related to `Safebots/bots` (the index category) for enumeration.
The relation type is uniformly `Safebots/bot`:

```php
Streams::relate(
    $publisherId,
    $publisherId, 'Safebots/bots',     // to: the index
    'Safebots/bot',                     // relation type
    $publisherId, $botStreamName,       // from: the bot
    array('skipAccess' => true, 'weight' => time())
);
```

The dispatcher fetches all bots for a publisher in one query and filters
in-memory against each bot's `Safebots/handlers` attribute. Using a
per-bot relation (rather than per-handler) matches the mental model: a
bot is a bot, not a collection of handler declarations. For most
publishers the bot count is small (5–20), so the in-memory filter is
cheap. If ever a publisher carries hundreds of bots, we can add a
secondary index keyed by event type.

---

## 4. Bots — The Core Abstraction

A bot is a declaration that says: *when one or more events happen on my
streams, run these tools.* The events are substrate stream-messages; the
tools are audited Safebox tool streams. One bot can listen to multiple
event types — each forms a coherent *role* the community has configured
as a unit.

### Minimal bot example

```php
$handlers = array(
    'Safebots/suggest/request' => array(
        'streamType' => 'Streams/chat',
        'tool'       => 'Safebox/tool/suggest/messages',
        'active'     => true,
    ),
);

Streams::create($communityId, $communityId, 'Safebots/bot', array(
    'name'       => 'Safebots/bot/suggest',
    'title'      => 'Inline suggestions',
    'attributes' => Q::json_encode(array(
        'Safebots/active'   => true,
        'Safebots/handlers' => $handlers,
    )),
), array('skipAccess' => true));

// Related to the publisher's bots index with the uniform relation type.
Streams::relate(
    $communityId,
    $communityId, 'Safebots/bots',
    'Safebots/bot',
    $communityId, 'Safebots/bot/suggest',
    array('skipAccess' => true, 'weight' => time())
);
```

`Safebots_Bot::seedDefaults($publisherId)` is the idiomatic way to seed
the two default bots (`Safebots/bot/suggest`, `Safebots/bot/greet`)
under a publisher — it handles the index category creation, streams, and
relations in one call.

### How dispatch works

Safebots registers ONE hook, on the substrate event `Streams/postMessages`,
which fires once per batch of messages posted anywhere in the substrate.
The hook is in `handlers/Safebots/after/Streams_postMessages.php` and
calls `Safebots_Bot::dispatch($streams, $posted, $skipAccess)`.

For each `(publisherId, streamName)` pair in the batch:

1. Enumerate the publisher's active bots via
   `Safebots_Bot::activeBotsForPublisher($publisherId)` — a single
   `Streams::related` call on the `Safebots/bots` category. Memoized
   per dispatch so multiple messages under the same publisher don't
   re-query.
2. For each posted message under that stream, walk each bot's
   `Safebots/handlers` map:
   - Skip if the bot's `Safebots/active` is not true.
   - Skip if `handlers[message.type]` is missing.
   - Skip if the handler's `active` is false.
   - Skip if `streamType` filter doesn't match the affected stream's type.
   - Skip if `fromType` filter (for relate-messages) doesn't match the
     counterpart stream's type.
   - Skip if the handler declared `suppressSelfAuthored: true` AND
     the message was authored by the publisher itself. This opt-in
     loop-guard is needed only for handlers whose tools write the same
     message type they react to (so they'd otherwise re-dispatch their
     own writes). Most handlers leave it off — a greet bot reacting to
     `Streams/created` wants to fire even when a community admin
     creates a chat under the community's own publisherId.
3. Surviving handlers dispatch fire-and-forget to Node via
   `Q_Utils::sendToNode` with `Q/method: 'Safebots/invokeTool'`. The
   Node handler loads the tool, runs it in the Safebox sandbox, and
   the tool's outputs flow back through normal substrate writes.

The dispatcher does not itself make LLM calls, fetch streams, or propose
actions. Those all happen inside the tool, in the Safebox sandbox, where
they're billed, cached, and governed by the standard Safebox machinery.

### Why there's no per-event registration

Previous iterations of this plugin registered per-event
`handlersAfterEvent` entries in `plugin.json` (e.g.
`Streams/create/Streams/joined`). That worked but required the plugin to
pre-declare every event any bot might care about — adding a new bot
meant editing `plugin.json`.

Subscribing to `Streams/postMessages` once and filtering in-memory by each
bot's `Safebots/handlers` attribute removes that coupling entirely.
Communities can author new bots reacting to new message types without
ever touching plugin configuration. The hot-path cost — early-return
when a publisher has zero active bots — is a single `Streams::related`
hit, cheap enough that every substrate message posting can pay it.

### Why bots aren't themselves specially governed

A bot is a configuration stream. Creating or updating a bot stream goes
through the normal Streams update pipeline, which — if the community has
a policy on `Safebox/policies` covering `Streams/update` on
`Safebots/bot` — triggers `Actions.propose` like any other stream edit.
Communities that want strict governance of their bot config set such a
policy; communities that are happy with direct edits leave it off.

The *tool* a handler references must be audited (per Safebox.md §15.4) —
that's where the code-signing gate lives. A community can reference any
audited tool in a handler; they cannot reference an un-audited one because
the Safebox sandbox rejects loading unaudited tool code.

### Activation is the governance decision

When a community relates a bot stream to its `Safebots/bots` category,
that act of enabling IS the standing governance decision. The bot's
tools propose `Safebox/action` streams whenever they write, and a
Safebots-published **judgment** owned by the community votes approve
on each proposal where the proposing tool's owner is on the community's
`Safebots/registered-bots` list. The judgment vote is recorded as the
community's own signature against per-action-type voting policies
seeded at install (threshold 1, so the community-as-signer alone
clears the bar). See "How activation maps to a judgment + policy"
below for the full structural picture.

The "1 of 1 by the community" pattern is expressed exactly: the
community pre-casts its approval by activating the bot; every subsequent
firing of the bot's tools rides on that pre-cast approval. No per-firing
vote, no per-firing prompt to the community, no special
bot-exempt-from-governance mode. The bot's writes look to Safebox
exactly like any other proposed action — just with a judgment-driven
auto-vote that completes the policy's quorum on its own.

Communities wanting stricter governance on specific bot behaviors raise
the policy threshold for that action type (e.g. 2-of-3 instead of 1-of-1)
or activate additional, stricter judgments at higher weight to vote
reject before the catch-all approve-bounded-bot-actions judgment can
vote yes. The substrate runs judgments in weight order; first
definitive verdict wins per signer. The audit trail records exactly
which judgment voted and why.

`Safebots::ensureActivation($publisherId, $actionTypes)` wires up
all of this for the default action types the seeded bots' tools
produce. The installer calls it for the hosting community; other
communities onboarding Safebots call it under their own publisher with
`Safebots::defaultActionTypes()` (extensible for community-authored
bots that propose additional action types).

### How activation maps to a judgment + policy

Safebox's newer versions (v0.21+ in the team's working substrate;
not yet present in the `Safebox_15` reference Safebots audits against)
ship default-deny in `Action::propose` and a **judgment** mechanism:
a piece of JS owned by a signer (a user or a community) that votes
on behalf of that signer. Safebots adopts the judgment mechanism
for activation.

This is the key substrate dependency for the round. Where the
judgment system has shipped, Safebots's contract enforcement runs
end-to-end: greet bots' proposed `Streams/update` actions hit the
seeded policy, the activated judgment evaluates them against the
firing handler's `actions` declaration, votes approve as the
community-as-signer at threshold 1, and the action commits.
Where the judgment system has not shipped (Safebox_15), the seeded
policies still exist but no judgment runs against them; bot
proposals sit in `'held'` status waiting for a human signature
through Safebox's existing `Safebox/policy/vote` IPC path.
Safebots ships the judgment file, the activation relations, and
the contract — but the run-time integration depends on the substrate
having the dispatch machinery. See the handoff doc for substrate-
version coordination.

Concretely, when a community installs Safebots and `ensureActivation`
runs, it seeds:

  - `Safebots/registered-bots` — a content stream holding a JSON
    array of userIds whose tools the activation judgment trusts.
    Initially `[$communityId]` since the default bots are owned by
    the community itself. The list grows as custom bots from other
    publishers are added via `Safebots::registerBotPublisher`.

  - `Safebots/sensitive-categories` — a content stream listing
    category names whose entries trigger side effects (email send,
    payment, public feed, etc.). Empty at install. Communities
    populate it as integrations are wired up. The judgment refuses
    any `Streams/relate` whose destination matches an entry,
    regardless of declarations.

  - Three `Safebox/tool/*` streams — the starter tools (greet/chat,
    greet/goal, suggest/messages). Each carries
    `Safebox/codeFilePlugin: 'Safebots'`, `Safebox/codeFile`,
    `Safebox/sha256` (computed from the on-disk file at install),
    and `Safebox/actionTypes` declaring what the tool may propose
    (`["Streams/update"]` for greet, `[]` for suggest). Where Safebox's
    runtime ships the declared-vs-proposed actionTypes check, the
    type-level bound runs before the proposal even reaches the policy
    (defense in depth alongside the judgment); where it doesn't, the
    judgment is the sole enforcement of the declaration. Either way,
    the declaration on the tool stream is what the contract refers to.

  - `Safebox/judgment/Safebots/approve-bounded-bot-actions` — the
    Safebots-authored judgment owned by the community. The JS lives
    at `files/judgments/approve-bounded-bot-actions.js`; the substrate
    verifies its sha256 at run time. The judgment reads the proposing
    tool's chain (action → task → tool → owner), finds which bot's
    handler invoked the tool from the task's attribution attributes,
    reads that handler's declared `actions` array off the bot stream,
    and matches the proposed action against allowed shapes — type,
    target stream, attribute names (allowlist for Streams/update),
    message type, etc.

  - One `Safebox/policy/Safebots/<action-type-slug>` stream per action
    type (`Streams/update`, `Streams/message`, `Streams/relate` by
    default). Each is a voting policy with `approvalThreshold: 1`,
    related to the community's `Safebox/policies` index under its
    action type.

  - An activation relation from the judgment INTO each per-type
    policy. When a registered bot proposes an action, Safebox finds
    the matching policy, dispatches to the activated judgment, the
    judgment evaluates the contract, and the threshold-1 policy
    approves the action immediately on a yes vote.

All of these streams carry `Safebox/scope: 'platform-default'` so
where Safebox ships a self-promotion guard on judgment/policy/tool
creates (newer substrate versions; not present in `Safebox_15`), the
guard treats them as install-time platform infrastructure rather than
requiring an M-of-N audit. The seeding itself happens under
`skipAccess: true` which only the platform publisher has during seed,
so the gate that *actually enforces* who can mint these streams is
the access flag at install time. The `Safebox/scope` attribute is
the metadata the substrate's guard reads where the guard is present.
Documented in the handoff doc.

### What the contract enforces

The judgment approves an action proposal IFF all of these hold:

  1. The action's task carries non-empty `Safebots/bot` and
     `Safebots/handlerKey` attributes — the proposing tool was invoked
     through Safebots's dispatcher with proper attribution.

  2. The proposing tool's owner equals the signer's community. Tools
     owned by other publishers go through different judgments (none
     shipped by default).

  3. The bot stream exists, is owned by the community, and has a
     `Safebots/handlers` entry under the firing handlerKey.

  4. That handler entry's `actions` array contains an entry that
     matches the proposed action by type, target stream
     (`onStream: "self"` resolves to the trigger stream), and per-
     action-type fields:
       - For `Streams/update`: every key in `payload.attributes` must
         appear in the handler's declared `attributes` array
         (allowlist). The substrate's blocklist filter is coarser —
         Safebots narrows it to specific names. Proposals carrying
         `payload.fields` fail the contract regardless.
       - For `Streams/message`: declared `messageType` matches the
         proposed `payload.messageType`.
       - For `Streams/create`: declared `streamType` matches the
         proposed `payload.type`.
       - For `Streams/relate`: declared `relationType` and `fromType`
         match; both endpoints must be community-owned.

  5. The action's target stream is NOT a governance-shaped stream:
     `Safebox/policy/*`, `Safebox/judgment/*`, `Safebox/tool/*`,
     `Safebots/bot/*`, `Safebots/chatPolicy`,
     `Safebots/sensitive-categories`, `Safebots/registered-bots`.
     These remain governed by the substrate's own policy machinery.

  6. For `Streams/relate`: both endpoints are community-owned (cross-
     tenant relations always require human signing), AND the
     destination category is NOT on `Safebots/sensitive-categories`.

Anything failing 1–4 abstains (next judgment in chain runs; if all
abstain, action waits for human signing). Anything failing 5–6 rejects
(positively out-of-bounds, no further chain). Approve-by-default never
happens — the judgment must affirmatively match the contract.

### End-to-end: from tool body to executed write

Every gate, in order, that a tool-proposed action passes through before
the underlying write actually happens:

**Gate 0 — Tool registration (install time).** Each `Safebox/tool/*`
stream carries a `Safebox/sha256` computed from the on-disk file at
install. SandboxRunner refuses to load tool code whose live hash
doesn't match the registered hash. Compromised tool source on disk
fails the load; the tool can't run at all.

**Gate 1 — Tool's declared `Safebox/actionTypes` (Safebox runtime, where shipped).**
When the tool body calls `Action.propose(type, payload)`, newer
Safebox versions check `type` against the tool's registered
`actionTypes` array at the propose call site (`Orchestrator.js`
Action.propose entry). If the array is non-empty and the type isn't
in it, the call rejects synchronously inside the sandbox — the action
stream never gets created. Where this runtime check is shipped, it's
a coarse-grained type-level bound running before the proposal even
reaches the policy machinery. Where it isn't (older substrate
versions), the activation judgment alone enforces the type-vs-
declaration relationship — see Gate 3. Either way, suggest's tool
declaring `actionTypes: []` means any `Action.propose` call from
suggest fails the contract; only the gate at which it fails differs.

**Gate 2 — Action stream's policy lookup (Safebox).** The runtime
creates a `Safebox/action` stream. `Action::onActionCreated` runs.
It queries the action's community for policies whose relation type
matches the action type. If there are no matching policies, the
action's status becomes `'approved'` immediately (no governance
applies). If there are matching policies, the most-restrictive
voting policy wins; the action's status becomes `'held'` pending votes.

**Gate 3 — Voting (Safebox).** `Action.php` reads two thresholds from
the chosen policy:
  - `Safebox/approvalThreshold` (M) — number of approval signatures
    required for the action to commit.
  - `Safebox/rejectionThreshold` (P) — number of rejection signatures
    required for the action to die permanently.

Defaults from `Safebox/config/plugin.json` are M=2, P=1 — but the
policy's own attribute overrides the default. Whichever count reaches
its threshold first decides: M approvals → status `'approved'`;
P rejections → status `'rejected'`.

Approval signatures come from two sources, mixed:
  - Human OCP signatures from users on the policy's labeled-auditor
    set, or from the community-as-itself.
  - Judgment-cast signatures: each activated judgment evaluates the
    action; on `{approve: 'yes'}` Safebox records the signer's signature
    via the substrate's claim-envelope mechanism with `claim.kind:
    'judgment'`. Judgments run in weight order; first definitive
    verdict per signer wins.

The Safebots judgment (`approve-bounded-bot-actions`) casts the
community's vote on actions matching the firing handler's `actions`
contract. With the seeded policies' threshold M=1, that single vote
is sufficient. With P=1, a single rejection from anywhere also kills
the action.

**Gate 4 — Action execution (Safebox).** Once status is `'approved'`,
`onActionApproved` fires. The substrate executes the action by
calling the underlying Streams operation as the action's publisher
(privileged — the policy approval IS the access decision at this
point). For `Streams/update` actions, this writes to allowed fields
(title/content/icon) and sets non-blocklisted attributes; for
`Streams/message`, the message-post path; for `Streams/relate`, the
relate path. **The write itself happens here.**

**Gate 5 — Substrate access on execution (Streams).** The substrate
operation runs as the publisher. The publisher always has full write
access to its own streams (no access check needed for self-writes).
For relations, both endpoints' access is checked — but again, the
publisher has full access to streams it owns. For cross-publisher
operations the substrate's normal access machinery applies.

The chain end-to-end:

```
tool body
  ↓ Action.propose(type, payload)
Gate 1: actionTypes runtime check
  ↓ creates Safebox/action stream
Gate 2: policy lookup (relation-type matches action type)
  ↓ status = 'held' pending votes
Gate 3: voting (M approvals OR P rejections)
  ↓ status = 'approved'
Gate 4: onActionApproved → execute as publisher
  ↓ underlying Streams operation
Gate 5: substrate access (publisher writes to own streams: pass)
  ↓
write committed
```

### Where the default bots clear gates without humans

For the three default bots' actions:

  - **Gate 0** clears at install — `ensureActivation` registers the
    tool with `Safebox/sha256` set; the file on disk matches.
  - **Gate 1** clears in the tool body — greet/* tools' declared
    actionTypes is `["Streams/update"]`, and they only call
    `Action.propose("Streams/update", ...)`.
  - **Gate 2** clears via the per-action-type policy seeded by
    `ensureActivation` — `Safebox/policy/Safebots/update` is
    related under `Streams/update` in the community's
    `Safebox/policies` index.
  - **Gate 3** clears via the contract judgment voting `{approve:'yes'}`
    on behalf of the community-as-signer. M=1 means that vote alone
    clears the bar. P=1 means a single rejection from any human
    auditor would still kill the action — humans aren't excluded from
    the reject path, just not required for the approve path.
  - **Gate 4** runs as the community publisher.
  - **Gate 5** clears because the community is writing to its own stream.

So default bots write without any human voting because **every
governance-time decision** that approves their writes was already made
when an admin installed the plugin (gate 0), audited the handler
declarations (gate 2's per-handler scope), and approved the
contract-enforcement judgment (gate 3's signer behavior). The runtime
chain just cashes those upstream decisions. A bot whose tool tries to
propose something outside its handler's declared `actions` contract
fails at gate 3 (judgment abstains), the action sits in `'held'`
status, and humans must sign manually for it to ever approve.

### What user-authored writes look like (today)

A user manually calling `Streams::relate` to add a stream to a category
takes a different path entirely: **none of gates 0–4 apply.** The
substrate just checks the user's writeLevel against the category
stream's `writeLevel` requirement (`relate` level by default, ~23 of
40). If the user has it, the relation commits immediately. No
`Safebox/action` is created; no policy fires; no judgment runs.

The voting machinery exists only for actions that flow through
`Action.propose` — which today happens only from tool execution. User-
initiated writes are gated purely by Streams' own access-level system.

This asymmetry is a real architectural choice and a discussion topic
for future rounds (see Safebots.md entry 63 for context). The current
shape: tools propose, the substrate votes; users with sufficient
writeLevel just write.

### The structural safety story

Eight properties together make Safebots structurally distinct from "an
agent running with community privileges":

  1. **Install-time integrity.** Tool registrations, judgment, policies,
     and activation relations are created at install with `skipAccess`
     by the platform publisher. Post-install modifications go through
     Safebox's governance machinery. The install state is the trusted
     root.

  2. **Type-level bound (runtime, version-dependent).** Where Safebox's
     runtime ships the declared-vs-proposed actionTypes check, the
     runtime rejects any `Action.propose` whose type isn't in the
     tool's declared `Safebox/actionTypes` array. Coarse but mandatory.
     Suggest declares `[]` and is structurally read-only at the runtime
     gate. Where the runtime check isn't shipped, the judgment is the
     sole enforcement of declaration matching — the `actions` array
     bounds proposals at the judgment layer rather than at the runtime
     layer. Either way, the declaration on the tool stream is what the
     contract refers to.

  3. **Per-handler bound (judgment).** A judgment vote requires the
     proposed action to match the firing handler's declared `actions`
     array exactly. Fine-grained.

  4. **Cross-tenant bound (judgment).** `Streams/relate` requires both
     endpoints community-owned. Cross-community relate always requires
     human signing.

  5. **Sensitive-category bound (judgment).** Destinations on the
     community's sensitive list always require human signing
     regardless of declarations.

  6. **Governance-stream bound (judgment).** Actions targeting policy,
     judgment, tool, bot, or governance-config streams always reject.

  7. **Code-execution integrity (substrate).** Tool code is sha256-
     verified at run time. Judgment code likewise. Bot declarations
     live on the bot stream, which is itself governed.

  8. **Vote attribution (substrate).** Every approved action carries
     the substrate-signed envelope of the approving judgment with
     `claim.kind: 'judgment'` and `claim.stm.judgmentStreamName`. The
     audit trail shows exactly which judgment voted yes and why.

The blast radius of any Safebots bot is determined at governance time,
not at run time. A prompt injection in user chat input cannot induce
actions outside the handler's declared `actions`. A compromised LLM
cannot propose unaudited writes. A bug in a bot's tool body cannot
escape the type-level and per-handler bounds. A community can only
weaken the bounds by going through governance to modify its
declarations, sensitive list, or judgment — and that weakening is
visible in the audit trail.

### Judgments are the user-authoring surface

What we ship in `files/judgments/approve-bounded-bot-actions.js` is one
example of a judgment, not the only kind. The mechanism is the
substrate's automation primitive for human-in-the-loop displacement:
any user (or community) can author a judgment that automates a specific
kind of approval they'd grant if asked. "I bulk approve emails using
this template, sent to anyone on my Friends label." "I auto-approve
status updates within this project as long as the assignee posts them."
"I approve relate actions whose destination is in my Personal category."

Each is a JS file. Reading the Safebots judgment shows the pattern:
walk the action's provenance, match against a predicate, return a
verdict. Communities and users will write many more. The framework is
the seam.

### Compaction is supported but rarely needed

Chat-context size in this design is bounded by user authorship: the
chat log carries what users actually typed. Large pastes become
artifact streams (`Streams/text` or similar) related to the chat;
the chat message references them rather than embedding them. Assistant
turns are persisted as `Safebots/digest` streams, and the chat log
references digests rather than embedding their full content.

So a typical chat — even a long-running one — fits comfortably in any
modern LLM context window. Compaction (turning old portions of a chat
into a summary digest) is supported via the `Safebots/digest` stream
type but rarely needed in practice. If a chat does grow beyond useful
context, the orchestrator can mint a digest covering the older portion
and replace those messages in the assembled context with a reference
to the digest. One compaction event is a one-time KV-cache invalidation;
subsequent prompts cache against the new shape. Future round.

---

## 5. The Publisher-Augments Model

In Telegram, when you add an inline bot to a chat, that bot is available to
anyone typing in the chat. The chat's administrator decided to enable it; the
participants then use it. Safebots works the same way.

### The dispatch rule

For a given event on a stream:

- **The stream's publisher** is the user whose behaviors are consulted.
- **The event's subject** (the stream, the message, the relation) is the input
  the behavior's tool operates on.
- **The event's triggerer** (whoever posted the message, whoever proposed
  the action) is captured as context for the tool but isn't the one whose
  behaviors run.

A user posting in a community's chat does not trigger their own behaviors
against the community's stream; they trigger the *community's* behaviors.
That's the right model — it's the community's chat, the community decides
what bot-assistance is available, participants get the benefit.

### Same user in two roles

A community user and an individual user can both publish bot streams.
Alice publishes behaviors that fire on Alice's streams (her personal timeline,
her DMs, her notes). Her community publishes behaviors that fire on the
community's streams. They don't interfere. If Alice is participating in the
community's chat, only the community's behaviors fire — because the chat's
publisher is the community.

### Cross-community content flow

If a chat has a single publisher (the common case), only that publisher's
behaviors fire. A chat with co-publishers (not currently a Qbix primitive,
but if it were) would invoke each publisher's behaviors — each under its own
billing context.

The more common cross-community scenario is *content flow*: a behavior on
Community A's chat fetches data from Community B's shared streams to inform a
suggestion. This composes cleanly with the attribution strip (§6) —
source = B (data origin), agent = A (who produced the message), strip shows
"from @B via @A."

---

## 6. Attribution — `from @source via @agent`

Every augmented message carries attribution metadata in
`instructions['Safebots/via']`:

```json
{
  "source": { "userId": "...", "displayName": "..." },
  "agent":  { "userId": "...", "displayName": "..." },
  "promptRef": "...",
  "artifactKind": null
}
```

- **source** is the community whose data / policies / knowledge shaped the
  content. Usually this equals the stream's publisher.
- **agent** is the community (or user) that produced the augmentation —
  invoked the LLM, ran the tool, authored the output. Usually this also
  equals the stream's publisher.

Both are recorded so history stays reconstructable; which parts render on
screen are decided at render time by the four-case collapse rule.

### The four-case collapse rule

Render time has two userIds available from the message itself:

- `author` = `message.byUserId` — whoever posted.
- `publisher` = `message.publisherId` — the stream's owner.

Combined with source and agent from the instructions:

| author ?= agent | source ?= publisher | Render                          |
|-----------------|---------------------|---------------------------------|
| equal           | equal               | no strip (fully redundant)      |
| equal           | different           | `from @source`                  |
| different       | equal               | `via @agent`                    |
| different       | different           | `from @source via @agent`       |

When a community's own behavior generates a message that the community posts
under its own identity, all four are the same userId and the strip hides —
the message looks like any other community-authored post. When a user
accepts an inline suggestion and sends it, `author` is the human but `agent`
is the community, so the strip shows "via @community" to indicate AI
involvement. Cross-community content flow naturally produces the full
`from @source via @agent` rendering.

### Implementation

The collapse rule lives in `web/js/tools/safebots/chat.js`
`_renderViaStrip()`. It reads the `Safebots/via` instructions blob from each
message and applies the rule to decide what (if anything) to render above
the message content.

---

## 7. Safebots/suggest — Inline Suggestions

The `Safebots/suggest` endpoint powers Telegram-style inline completions in
any chat whose publisher has a `Safebots/bot/suggest` behavior active.

### Request

```
POST /Safebots/suggest
  publisherId      chat stream's publisher (the community)
  streamName       chat stream's name
  composerText     what the user has typed so far (may be empty)
  maxSuggestions   upper bound on returned suggestions (1–5, default 3)
```

### PHP handler: thin dispatcher

The PHP handler does auth, access, request shaping, and dispatches to Node
via `sendToNode(reply: true)` — the synchronous round-trip primitive. No
LLM work happens in PHP. All heavy lifting — prompt assembly, Protocol.LLM
call, billing, cache consultation — happens in Node, inside the Safebox
sandbox, as the tool code referenced by the community's suggest behavior.

```
Safebots/suggest(publisherId, streamName, composerText, maxSuggestions):
    requester = Users::loggedInUser(true)                      # auth gate
    stream    = Streams_Stream::fetch(null, publisherId, name) # access check
    stream->testReadLevel('content') or throw

    behavior = Safebots_Bot::find(publisherId, 'suggest-request')
    if !behavior or !behavior.active:
        return []                                              # no suggestions available

    source = { userId: publisherId, displayName: Safebots::displayNameOf(publisherId) }
    agent  = source                                            # community is its own bot
    goalPrompt = Safebots_Goal::buildSystemPrompt(publisherId, streamName)
    composerText = truncate(composerText, maxComposerLength)

    nodeResult = sendToNode({
        'Q/method':        'Safebots/invokeTool',
        'behavior':        behavior.name,
        'tool':            behavior.getAttribute('Safebots/tool'),
        'publisherId':     publisherId,
        'streamName':      streamName,
        'composerText':    composerText,
        'maxSuggestions':  maxSuggestions,
        'goalPrompt':      goalPrompt,
        'communityId':     publisherId,
        'requesterUserId': requester.id
    }, {reply: true, timeout: 10000})

    # Node returns { suggestions: [{ text, artifactKind }, ...] }
    # PHP attaches source/agent before slotting
    suggestions = []
    for item in nodeResult.suggestions:
        suggestions.append({
            text:         item.text,
            source:       source,
            agent:        agent,
            promptRef:    null,
            artifactKind: item.artifactKind
        })
    Q_Response::setSlot('suggestions', suggestions)
```

### Node handler: runs the tool

```js
server.addMethod('Safebots/invokeTool', function (parsed, req, res, ctx) {
    // Load the referenced Safebox/tool, verify sha256
    // Build sandbox API, invoke tool with event payload
    // Tool returns suggestions array → res.send({ slots: { suggestions: [...] } })
});
```

The Node handler uses the reply-mode IPC form — `sendToNode(reply: true)`
from PHP blocks until Node calls `res.send(...)`, returning the decoded JSON
body synchronously. This is the right primitive for suggestion generation:
users expect a 1–2 second response, and async-job patterns (webhooks, polling)
are wrong-shaped for interactive UI.

### Client — chat extension

The `Safebots/safebots/chat` tool (in `web/js/tools/safebots/chat.js`) is a
Streams/chat extension that adds a "Suggest" menu item and a hidden
suggestions panel. Typing in the composer, or tapping "Suggest," calls the
endpoint; tapping a suggestion fills the composer with its text and records
the attribution for the next `beforePost` hook to attach to the outgoing
message. See §15 for the full chat-extension reference.

---

## 8. Safebots/goal — Goal Streams

A goal stream is a reusable system-prompt template, with placeholders for
interpolation at chat-attachment time. Goals are how Safebots chats know
what they're trying to accomplish beyond freeform conversation.

### Platform vs community goals

Platform-default goals ship in the install seed script, published by
`Users::communityId()` (the hosting app's community). Communities that want
to customize a goal create their own version under their own publisher:

```
Users::communityId() / Safebots/goal/generate/capability   ← platform default
{acmeCommunityId}    / Safebots/goal/generate/capability   ← Acme's customization
```

`Safebots_Goal::findGoalStream($goalStreamName, $communityId)` tries the
community's own version first, falls back to the platform default. This
lets any community override any goal without touching the install script.

### Goal attributes

```
Safebots/fields     field-name: value pairs for template interpolation
Safebots/context    context blob — urls, background info, etc.
Safebots/pattern    "fields" | "artifact" | "process" | "codeGen"
Safebots/model      override model for this goal
```

### Prompt interpolation

`Safebots_Goal::buildSystemPrompt($chatPublisherId, $chatStreamName)`:

1. Follows the chat stream's `Safebots/goal` relation to find its goal stream
   (a chat has at most one goal at a time).
2. Reads the goal's content (the template).
3. Merges `Safebots/fields` + `Safebots/context.urls` into a single
   interpolation dict.
4. Runs `Q::interpolate($content, $dict)` and returns the resulting system
   prompt.

If the chat has no goal relation, returns an empty string — callers (like
`Safebots/suggest`) append context for their own use.

---

## 9. Safebots/dialog — Dialog Streams

A dialog is a time-boxed conversation instance focused on producing a specific
artifact or collecting a specific set of fields. Dialogs are created from
templates that reference a goal.

### `Safebots_Dialog::create`

```php
Safebots_Dialog::create($communityId, $dialogType, $draftName, $goalFields);
```

- `$communityId`: the community publishing this dialog (also the chat's
  publisher, so behaviors fire against this community).
- `$dialogType`: `"capability"`, `"tool"`, `"workflow"`, etc. — selects the
  goal to relate.
- `$draftName`: slug used in the stream name.
- `$goalFields`: field values for interpolation.

Creates a `Safebots/dialog/{draftName}` stream under the community, sets
initial attributes, relates it to the platform (or community-fork) goal.
The community itself is implicitly the publisher — behaviors attached to
the community run on this dialog the same way they'd run on any other
chat the community publishes.

### `Safebots_Dialog::complete`

Marks a dialog `dialogStatus = done`, records completion time, and posts a
`Safebots/dialog/complete` message to the dialog stream. Because the dialog
is published by the community, the completion message is authored by the
community under its own identity — no bot user needed.

---

## 10. Storefront Greeting

Chats can carry a `Safebots/greeting` attribute — a structured value the
chat extension renders as a sticky strip above the message stream. It's
the chat's "storefront": always-present content every viewer sees
identically, like a pinned message, updateable over time by whatever
mechanism the community has configured.

### Why storefront, not reactive

Earlier iterations of this plugin used a reactive pattern: listen for
`Streams/joined`, check an idempotency flag, post a greeting as the
first non-publisher user joins. That worked but conflated display with
authorship. Users who never formally "joined" (arrived via a link, read
over a shoulder, viewed an export) would never see a greeting. Races on
simultaneous first joins produced duplicate posts. The flag itself was
state that wanted to live in display code, not in the substrate.

The storefront model separates cleanly. **What** is displayed is the
current value of `Safebots/greeting`, pure function of the attribute.
**When** that value is authored is a bot's concern — the greet bot
declares which events trigger a refresh and writes the attribute when
they fire. The chat extension reads the attribute on render and any
time the stream refreshes. No idempotency bookkeeping, no per-user
state, no join-dependent authorship.

### The `Safebots/greeting` attribute

```
Safebots/greeting: {
    text:     "Welcome. Feel free to introduce yourself...",
    options:  [
        { label: "What can I do here?",  payload: "help" },
        { label: "Start a conversation", payload: "open" }
    ],
    byUserId: "{communityId}"
}
```

`text` is the greeting prose; `options` is an array of tappable buttons
the chat extension renders below it. Tapping an option fills the
composer with the option's label so the user can edit and post normally —
the user still authors their message, the storefront merely suggests
starting points.

`byUserId` is metadata for audit (which userId authored the greeting —
typically the community itself). The chat extension doesn't use it
directly but it's available for UI that wants to show "storefront
authored by @community." Recency information comes from the substrate's
own stream-modification timestamp, not from a duplicate field inside
the value.

### The greet bot

`Safebots/bot/greet` is the canonical greet bot that
`Safebots_Bot::seedDefaults` installs. Its handlers map declares:

```
Safebots/handlers: {
  "Streams/created": {
    "streamType": "Streams/chat",
    "tool":       "Safebox/tool/greet/chat",
    "active":     true
  },
  "Streams/relatedTo": {
    "streamType": "Streams/chat",
    "fromType":   "Safebots/goal",
    "tool":       "Safebox/tool/greet/goal",
    "active":     true
  }
}
```

The first handler fires when a chat stream is created — the tool writes
the initial `Safebots/greeting` attribute so the storefront is present
from birth. The second handler fires when a goal is attached to the
chat — the tool regenerates the greeting to reflect the new goal
context. Communities can extend this bot with further handlers (e.g.
`Safebox/action/approved` when the community promotes a new version, to
refresh the storefront's "currently featured" message) without editing
any plugin code.

The starter greet tools ship pre-audited with the plugin at
`files/Safebots/tools/greet/chat.js` and
`files/Safebots/tools/greet/goal.js`, and the seed script copies them
into Safebox's `files/tools/greet/` directory at install time. The
corresponding `Safebox/tool/greet/chat` and `Safebox/tool/greet/goal`
streams are created with `Safebox/sha256` computed from the file
contents, `Safebox/codeFilePlugin: 'Safebots'` (so SandboxRunner
loads the file from the Safebots plugin's own files dir), and
`Safebox/scope: 'platform-default'` (so Safebox's runtime signature
verification accepts them at install time without an explicit
M-of-N audit). The greet bot's handlers reference these tool streams
by name; when a triggering event hits, Safebots's dispatcher mints a
task stream with the four `Safebots/*` attribution attributes and
calls `Q.Safebox.Orchestrator.runTask`, which loads the tool, verifies
the sha256 against the file on disk, runs it in the sandbox, and
routes any `Action.propose` calls through the contract-enforcement
judgment.

### The chat extension's storefront render

`web/js/tools/safebots/chat.js::_renderGreeting` reads
`chatTool.state.stream.getAttribute('Safebots/greeting')`, renders the
text and options into a `.Safebots_greeting` element, and prepends it to
the chat's message container. The element is sticky so it stays visible
as users scroll through recent messages.

`_hookGreetingRefresh` subscribes to the existing `chatTool.state.onRefresh`
event (the same hook that re-hydrates the chat tool on attribute
changes) with a distinct tool-id suffix so the greeting re-renders
whenever the attribute changes — e.g. when the greet bot refreshes the
storefront after a goal is attached.

CSS for the storefront strip lives in `web/css/Safebots.css` under the
`.Safebots_greeting` selector family.

---
## 11. Chat Orchestration — Replies, Edits, and Working Context

The artifact IS the chat surface. There is no separate "chat stream"
that points at an artifact via attribute. A stream whose type config
includes `messages/Streams/chat/message/post: true` accepts chat
messages directly. Out of the box this is `Streams/chat` and
`Streams/text/small`; a deployment can opt other artifact types in
(`Streams/image`, `Streams/pdf`, `Streams/video`, `Streams/audio`,
`Streams/file`, `Streams/question`, `Streams/text`, etc.) by adding
the same key under those types' `messages` config in the host's
`Streams/config/plugin.json` overlay.

The artifact's own access rows decide visibility and posting:
`readLevel >= 'messages'` (numeric 35) lets a participant see chat
messages on the stream; `writeLevel >= 'post'` (numeric 20) lets them
post `Streams/chat/message`, `Streams/chat/edit`, and
`Streams/chat/remove`. The artifact's `title`, `icon`, and `content`
are visible to anyone with `readLevel >= 'see'` (numeric 10), so users
who can't read the conversation can still see what the artifact is.
The bot's `Safebots/active` + `Safebots/handlers` machinery from §4
decides which (messageType, streamType, fromType) tuples fire which
tools.

This collapses what older systems split into two streams (the artifact
and the conversation about it) into one. The artifact's own ordinal
sequence carries its content edits, its chat messages, and any edits
to those chat messages — all interleaved, all replayable from the
stream alone. A `Streams/pdf` artifact has the conversation about it
in its own message log; a `Streams/image` artifact does the same; a
`Streams/question` answers itself in place. By convention every
stream's name starts with its `$type/`, so a URI like
`{publisherId}/Streams/pdf/{id}` carries enough information for tools
and renderers to pick the right preview without an extra fetch.

### What's substrate vs what's Safebots

`Streams/chat/message`, `Streams/chat/edit`, and `Streams/chat/remove`
are substrate-level message types — registered under
`Streams/types/*/messages` in `Streams/config/plugin.json`, not invented
here. The substrate handles delivery, autosubscribe, notification
filtering, and the relation-to-artifact allowlist
(`Streams/chat/allowedRelatedStreams`).

`Safebox_Context::assemble()` is the substrate's prompt builder. It
takes a set of streams, sorts them by `updatedTime` ascending, fires
a hook chain so listeners can mutate the set, renders each stream
into a byte-stable XML-bracketed block (header / title + content +
attributes / trailer-with-time), concatenates them, and returns the
text together with incremental sha256 hashes that callers use as
KV-cache keys. That's the byte-stable foundation. The seven enrich
phases (`ner` → `resolve` → `expand` → `retrieve` → `rerank` →
`clarify` → `suggest`) compose on top — each is callable as a
standalone via the per-phase methods added in entry 56
(`Safebox_Context::enrichResolve`, etc.).

The Safebox sandbox already exposes a complete API surface to tools:
`Streams.fetch`, `Streams.related`, `Streams.search`, `Streams.lookup`
for reads; `Action.propose` for writes through governance; `Runtime.llm`
for inference calls; `Data.slice`, `Data.grep`, `Data.structure`,
`Data.jsonl` for content handling; `Crypto.sign`, `Crypto.verify`,
`Crypto.sha256` for cryptography; the full `Safebox.protocol.*`
namespace (HTTP, SMTP, Email, Payment, Web3, Telegram, etc.) for
real-world side effects. Tools are JS code that runs in the sandbox
and calls these methods. They're imperative — they can branch, loop,
retry, recur, call the LLM, decide based on the response. ReAct is
one *pattern* a tool can implement; it's not the architecture.

**Naming convention.** The substrate exposes these namespaces as
top-level Pascal-case bindings in tool scope (`Streams`, `Action`,
`Runtime`, `Cache`, `Crypto`, `Data`, `Safebox`). The verified pattern
in `Safebox_15/files/tools/verify/safebox/tool.js` is the canonical
reference: tools call `await Streams.fetch(...)`, `await Action.propose(...)`,
`Runtime.llm(...)`, etc. Lowercase forms (`streams.fetch`, `action.propose`)
appear only in JSDoc descriptive text. Plugin-contributed sandbox
methods (added via the `Safebox/Orchestrator/buildMethods` event hook)
follow the *opposite* convention: namespaced under their owning
plugin's lowercase name (`safebots.assemble`, `safebots.transcript`).
The capitalization signals what the binding is — substrate primitive
or plugin contribution. Tools authored under Safebots conventions
should follow this rule consistently.

What Safebots adds:

- **A chat-orchestration layer** — `Safebots_Context` (PHP,
  `classes/Safebots/Context.php`) and `Safebots.Context` (Node,
  `classes/Safebots/Context.js`). Both read DB and never write. They
  expose: transcript walking with edit replay (apply `Streams/chat/edit`
  and `Streams/chat/remove` messages to their target ordinals at read
  time), `isMultiParticipant` detection, `resolveChatPolicy` priority
  resolution (stream attribute → bot attribute → config default →
  hardcoded fallback), the bounded `FOCUS_VOCABULARY` for observation
  streams, and the `buildInstructions` / `buildEditInstructions`
  schema normalizers. PHP-side flows call the PHP class directly;
  Node-side IPC handlers call `Q.Safebots.Context`. The Node module
  HTTPs back to `/Q/plugins/Safebots/internal/context` for the few
  phases that need PHP-only DB access (transcript, participants,
  policy); cheaper phases (preview, related-summary) run Node-side
  via `Q.Streams`.

- **An instructions schema** for chat messages — `version`, `summary`,
  `refs`, `toolCalls`, `mentions`, `visibility`. Both `Safebots_Context`
  and `Safebots.Context` enforce it via their `buildInstructions`
  normalizer; the working-context assembler reads it.

- **A dispatch policy** — `Safebots_Bot::shouldRespondTo` decides
  whether a bot should react to a chat-typed message on a given stream.
  Solo streams pass through, multi-participant streams require mention
  / name-prefix / goal-trigger. Wired into `_dispatchMessage` as a gate
  before tool invocation; opt-out via handler `bypassChatGate: true`
  for moderation/logging bots.

- **An edit-message authoring helper** — `Safebots_Bot::postEdit`
  posts a `Streams/chat/edit` referencing a target ordinal and
  carrying a `changes` dict in instructions. Round 18's orchestrator
  uses this to amend its initial reply with artifact attachments,
  visibility flips, and progressive content updates. The Node-side
  counterpart is `Q.Utils.sendToPHP('Safebots/internal/postEdit', ...)`,
  exposed as `Safebots.postEdit(payload)` in `classes/Safebots.js`.

- **Round 18 will add the chat workflow and its tools** — implemented
  as a Safebox workflow definition (`Safebots/workflow/chat`) that
  composes three kinds of Safebots-authored tools: context tools that
  read the artifact's neighborhood and may call `Runtime.llm` (for
  summarization or extraction) but cannot propose actions; a decide
  tool that calls `Runtime.llm` against the assembled context and the
  user's message and emits structured intent (text reply plus optional
  action dispatches); action tools with typed inputs that propose chat
  messages and edits but do not call `Runtime.llm`. The two halves —
  LLM-calling and action-proposing — stay disjoint by registration-time
  rule; `verify/safebox/tool` rejects any tool declaring both
  `Runtime.llm` and `Action.propose` in its method set. Composition
  happens at the workflow layer, where policies, logging, and other
  hooks slot between steps. The chat workflow is itself a governance-
  approved artifact with its own sha256, just like a tool.

  Safebots contributes three read-only primitives to the sandbox method
  map under the `safebots.*` namespace:

  - **`safebots.assemble(streams, opts)`** — wraps
    `Safebox_Context::assemble` with chat-aware preprocessing
    (digest-stream injection, future specializations). Returns
    `{text, hashes, cacheKey, ...}` — byte-stable text plus
    incremental sha256 hashes for KV-cache reuse across turns.
  - **`safebots.renderStream(pub, name, opts)`** — wraps
    `Safebox_Context::renderStream` with type-specific specialization.
    Pure pass-through for most types; adds "(earlier conversation
    summary)" framing for `Safebots/digest` streams; suppresses any
    inline message list for `Streams/chat` streams (the transcript
    renders separately).
  - **`safebots.applyEditReplay(messages)`** — pure transform.
    Walks chat messages in ordinal order, applies `Streams/chat/edit`
    and `Streams/chat/remove` to their targets, returns post-replay
    state. No DB access, runs locally inside the sandbox.

  This follows the Qbix layering convention: Safebox provides
  generic substrate primitives (`Safebox_Context::assemble`,
  `Safebox_Context::renderStream`); Safebots wraps them with
  chat-specific machinery and exposes the wrappers under its own
  namespace. Tools authored under Safebots prefer the Safebots
  primitives; tools that want raw substrate access still call
  `streams.*` and the rest. Other primitives the chat decide tool
  uses — `Streams.fetch`, `Streams.related`, `Streams.search`,
  `Streams.getMessages`, `Streams.getParticipants`, `Runtime.llm`,
  `data.*` — are Safebox's contribution to the methods map.

  The contribution mechanism is one event hook in `Safebox.Orchestrator`:
  `Safebox/Orchestrator/buildMethods`, fired after the methods object is
  assembled and before it reaches `SandboxRunner`. Safebots's listener
  in `classes/Safebots.js` calls `Safebots.Context.exposeForSandbox(ctx)`
  and merges the returned closures. Identity (community, requesting
  user, wrapping key) propagates through the closure over ctx so
  substrate-level access checks scope reads correctly when the
  primitives reach back to PHP via `Safebots/internal/context`.

  Two of the three primitives (`assemble`, `renderStream`) HTTP back
  to PHP for the actual rendering — single source of truth lives in
  `Safebox_Context` PHP, and re-implementing the byte-stable algorithm
  in Node would risk drift across language boundaries that matters
  for KV-cache reuse. The third (`applyEditReplay`) is pure transform
  and runs locally with no HTTP round-trip.

  Nothing in this design requires `Runtime.invokeTool` (tool-to-tool
  dispatch) or a tool-call-aware LLM helper in the runtime. Tools use
  substrate primitives. Composition happens in the workflow. The
  parsing of LLM output into structured intent is the decide tool's
  job, written in JS like any other tool body.

### The chat stream is canonical

Everything the model sees on any turn is reconstructable from the
stream plus tool definitions plus system prompt. No hidden state, no
shadow logs, no parallel context store. Drop the stream and rerun, and
the model produces equivalent output. This is the property that makes
the system replayable, auditable, and KV-cache-friendly all at once.

### Message instructions schema

Chat messages on a stream carry `instructions` for assembly metadata.
The shape is the same regardless of who's posting — human, bot, or
automated process — though some fields are usually empty for human
turns.

```json
{
  "version": 1,
  "summary": "load-bearing facts of this turn, 50–300 tokens",
  "refs": ["stream URIs referenced by this message"],
  "toolCalls": [
    {
      "tool": "Safebox/tool/fetch/stream",
      "shortParams": { "publisherId": "com.x", "name": "Websites/post/abc" },
      "hash": "0x... canonicalized RFC 8785 hash",
      "result": "stream URI of result, or null",
      "error": null
    }
  ],
  "mentions": ["userId of bot or person addressed, if any"],
  "visibility": "visible"
}
```

`version` is required; assembly handles multiple versions. `summary`
is filled by the model at response time for bot turns, or by the
ingest pipeline for user turns with attachments. `refs` carries URIs
for any large content the message references — content lives in
separate addressable streams, never embedded inline. `toolCalls`
records the *fact* of each call (tool name, short params, canonicalized
hash, result ref) — not the response body, which stays cached and
re-fetchable. `mentions` is how a bot in a multi-participant stream
recognises it's being addressed. `visibility` is `visible` (default),
`hidden_until_response`, or `redacted`.

### Working-context assembly order

When a bot is invoked on a turn, the chat decide tool — running in the
Safebox sandbox — assembles the prompt by composing primitive calls.
The order is fixed; the byte-stable rendering of stream blocks is what
makes KV-cache reuse work across turns:

1. **System prompt** (static, cache-friendly).
2. **Tool definitions** (static for the session, cache-friendly).
3. **Identity stub** — who the user is, what bot this is, what
   community context applies. Drawn from user memory plus the bot's
   `Safebots/bot` stream attributes, passed in as workflow inputs.
4. **Stream prefix** — the artifact stream and any related streams
   the tool wants in context, rendered by `safebots.assemble`.
   The substrate sorts by `updatedTime` ascending, renders each via
   `safebots.renderStream` (which delegates to
   `Safebox_Context::renderStream` for most types and specializes
   for digest/chat types), concatenates, and returns text plus
   incremental sha256 hashes. The hash chain is what enables
   prefix-stable KV-cache reuse: a turn that adds one new stream
   to the set produces a prompt prefix that's byte-identical for
   the first N streams, so the model's KV cache from the prior turn
   stays valid up to the new addition.
5. **Related-stream selection.** The tool calls `Streams.related` on
   the artifact, decides which relations to include based on its own
   policy (proximity, type filter, recency), and adds those streams
   to the set passed to `safebots.assemble`. Tools deepen
   any single relation via `Streams.fetch` if the model asks for it
   in a follow-up turn; the prefix only includes the relations the
   tool selected for this turn.
6. **Chat-message transcript walked oldest → newest** with edit
   replay applied. The tool calls `Streams.getMessages` to fetch
   raw messages, then `safebots.applyEditReplay` to compute
   the post-replay state. The transcript is appended after the
   stream prefix; messages older than the verbatim window (default
   K=10, configurable) carry only `(ordinal, role, summary, refs)`
   while the last K carry full content plus structured
   `instructions`. The original messages are never modified;
   replay is computed at read time.
7. **Latest user turn** — appended last as the prompt's final
   message, full content with attachment summaries inlined.

This ordering is deliberate. Stable content sits at the prefix where
the KV cache hits across turns; only the tail of new content needs
prefilling. Steps 1-3 are turn-invariant. Step 4's text is byte-
identical across turns when the underlying streams haven't changed,
plus a strict suffix when streams have been added. Step 6 grows by
one ordinal per turn at the tail. The KV cache stays warm for the
shared prefix; only the tail (new transcript messages, new user turn)
requires fresh inference.

Note that step 4 plus step 5 plus step 6 together replace what older
designs called "pinned artifact preview" — the preview, the related
graph, and the conversation are all on the same stream, all reachable
through the standard substrate calls.

### KV-cache discipline — Lindy ordering, breakpoints, infrastructure tier

The cache reuse described above is real but conditional. Three
disciplines together make it actually pay off, and they're worth
calling out because they're easy to break by accident.

**Lindy ordering for stream blocks.** Step 4 sorts by `updatedTime`
ascending. The Lindy framing is the reason: a stream that hasn't been
edited in six months is overwhelmingly likely to stay that way; a
stream edited yesterday is much more likely to be edited again
tomorrow. Sorting oldest-first puts the most stable content at the
prefix, where cache reuse is most valuable. Newer, more volatile
streams sit at the tail where cache invalidation costs the least.
This isn't an optimization the substrate has to apply — it's the
default ordering — but tool authors must not fight it by re-sorting
or interleaving timestamps.

**Breakpoint convention.** Cache breakpoints can be set at chosen
`updatedTime` boundaries. Instead of one large stream block, structure
the assembly into several blocks separated at semantically meaningful
times: "everything older than 7 days," "everything between 7 days and
1 day ago," "everything from today." When a stream in the most recent
block changes, only that block's cache is invalidated; the older
blocks' KV-cache prefixes remain valid. The exact breakpoint placement
is a tool-author decision — the substrate exposes the ordering and
hashes, the tool decides where to cut. For most tools, three
breakpoints (older-than-N-days / recent / current-turn) suffice.

**Byte-stability requirements.** For cache reuse to actually fire,
the prompt prefix must be byte-identical across requests. Things that
break this:

  - **Timestamps in the prompt head.** Never write the current time
    into the system prompt or any pre-tail block. The substrate's
    `Safebox_Context::renderStream` puts each stream's timestamp at
    the trailer end of its rendering specifically to keep the head
    cache-friendly. Tool authors who add their own per-call
    timestamping into block [1] or [2] break this.
  - **Variable content before the tail.** The "small variable tail"
    pattern — composer text, latest user turn, suggestion count —
    only works as a tail. Inserting variable content earlier in the
    prompt invalidates everything from that point forward.
  - **On-the-fly summarization in the prefix.** Computing summaries
    in the tool body and embedding them in pre-tail blocks means the
    summary's bytes shift on every call (different LLM output). The
    substrate's `Safebots/digest` streams are explicitly designed
    around this — digests are content-addressed by sha256 and only
    written once per arc, so when they appear in step 4 they're
    byte-stable across turns.
  - **Non-deterministic ordering tiebreaks.** When two streams have
    identical `updatedTime`, the tiebreak must be stable
    (`publisherId+streamName` is what the substrate uses). A
    randomized tiebreak makes every assembly different by chance.

**Infrastructure tier — many KV caches simultaneously.** With our own
inference stack (vLLM or equivalent on org-controlled GPUs), prefix
caching can be juggled across thousands of conversations and dozens
of communities concurrently. Eviction policy, cache size, and prefix-
match strategy are infrastructure-team concerns. Safebots's role at
this layer is to author tool prompts that compose well with whatever
cache strategy the deployment uses — which mostly means following
the three disciplines above and not assuming a particular cache
window. The cost saving shows up as reduced GPU prefill compute (the
expensive part of inference); same throughput on fewer GPUs, same
latency on cheaper hardware. Per-tenant cache scoping
(`X-Cache-Tag: safebots/{publisherId}/{streamName}` on the inference
call) is the sandbox-side hook that tells the inference layer which
prefix bucket to consult; the tool author opts in by passing the tag
through to `Runtime.llm`.

### Three-layer output model

Tool responses live at three layers, each with a different role in
the chat surface and the LLM's working context.

**Layer 1 — visible chat content.** What the user sees in the
transcript. This is the `content` field of a `Streams/chat/message`
the bot posts (or a `Streams/chat/edit` if it's amending an earlier
message). Plain readable text, no structured metadata. The chat
extension renders this directly into the message row.

**Layer 2 — structured metadata accompanying the message.** Lives in
the message's `instructions` field as JSON. Includes:

  - **Entity references.** A list of proper nouns / projects /
    streams the response refers to, each with `{name, kind, uri?}`.
    `uri` is present when the entity is a known stream (the LLM
    recognized it as having a stream-name shape); the server can
    proactively fetch by exact match when assembling future turns.
    Entities without `uri` are inert metadata for future search.
  - **Tool-call ledger.** What tools the response invoked, with
    inputHash + executionHash for each. Lets the LLM (and human
    auditors) see what was computed without re-running it.
  - **Stream references.** For streams the response cited: full
    `(publisherId, streamName, title)` triples. Titles are short by
    convention — including them avoids re-fetching when only the
    title is needed for context.

The instructions field is sent to the LLM on subsequent turns as
part of the transcript-replay rendering. It's also surfaceable in
the UI as a click-to-expand affordance ("more info") rather than
shown raw. **A `Streams/chat/edit` message that amends an earlier
message MAY add to or replace the instructions on the message it
edits** — the substrate's edit-replay logic merges the instructions
forward, so a second-pass entity extraction or a corrected tool-call
ledger lands on the message it belongs to without a separate stream.

In addition to per-message instructions, an arc of conversation
accumulates a `Safebots/digest` stream that summarizes the broader
trajectory — pinned context for future turns when the verbatim
transcript drops out of the window. The digest is layer-2-ish too,
but at a coarser granularity.

**Layer 3 — full content + attributes + files on demand.** When a
tool needs more than what layer 1 + layer 2 carry, it fetches
streams (`Streams.fetch`), walks relations (`Streams.related`), and
in the file-bearing cases reads on-disk content directly. Grokers
produce relations that pre-index the substrate's content, so for
queries the Grokers framework already handles, relation-based
lookup is the right primitive. Fall through to `Data.grep` /
`Data.slice` / `ripgrep` only when no relevant relations exist —
relations are precomputed and cheap, byte-grep is on-demand and
expensive.

The layering pays off on each successive turn: the LLM sees layer 1
in the transcript and layer 2 as machine-readable metadata, so it
can decide which layer-3 fetches are needed for the current
response. Most turns don't fetch layer 3 at all; the metadata is
sufficient.

### Canonical `instructions` schema for layer-2 metadata

Tools that post chat messages with structured metadata follow this
shape on the message's `instructions` field. The shape is open at
the top level (tools may add their own keys for domain-specific
state) but the four fields below are reserved with the documented
semantics; tools that read messages should treat unknown keys as
opaque.

```json
{
  "entities": [
    { "name": "Q3 launch plan", "kind": "project" },
    { "name": "Streams/chat/announcements",
      "kind": "stream",
      "uri":  "communityId/Streams/chat/announcements" }
  ],
  "refs": [
    { "publisherId": "alice",
      "streamName":  "Streams/text/q3-plan-draft",
      "title":       "Q3 launch plan — draft" }
  ],
  "toolCalls": [
    { "tool":          "Safebox/tool/graph/explore",
      "inputHash":     "sha256:8f3a...",
      "executionHash": "sha256:b1c2..." }
  ],
  "summary": "Pulled the Q3 plan draft and listed three blockers."
}
```

**`entities`** — an array of `{name, kind, uri?}` records. `name` is
the canonical display name. `kind` classifies the entity (`person`,
`place`, `project`, `stream`, `document`, `concept`, `code-symbol`,
etc. — bounded vocabulary, extensible per round). `uri` is present
when the entity is a known stream and absent for entities that are
just proper nouns. Resolution under the requesting user's identity
correctly fails for streams they shouldn't see; the entity stays as
metadata, not as a leak.

**`refs`** — an array of `{publisherId, streamName, title}` triples
identifying streams the response substantively cited. Titles are
short by convention (≤80 chars in v1); including them avoids
re-fetching when only the title is needed for next-turn context.
Distinct from `entities` because refs are streams the response
already used, while entities are things the response talked about
that may or may not have streams.

**`toolCalls`** — ledger of sandboxed tool invocations the response
made, each `{tool, inputHash, executionHash}`. Lets the LLM (and
human auditors) see what was computed without re-running it. The
hashes are the canonical Safebox `Safebox/inputHash` /
`Safebox/executionHash` from the underlying task streams.

**`summary`** — optional. A one-sentence summary of the message's
substantive content. Distinct from layer 1 (which is the visible
content itself); when layer 1 is long, the summary is what gets
included in older-message rendering once the verbatim window drops
the message off the tail.

A `Streams/chat/edit` message that amends an earlier message may
provide a partial `instructions` object — the substrate's
edit-replay merges keys forward, so a follow-up entity extraction
pass can post an edit message containing only `{entities: [...]}`
and leave the rest of the original instructions intact. To replace
rather than merge a top-level key, set its value to `null` in the
edit (the merge logic treats null as a deletion).

Tools that post chat messages SHOULD populate `entities` and
`toolCalls` when they have the information; `refs` when they cited
specific streams. Tools that don't have the information leave the
field absent, not present-but-empty — absence is meaningful (no
extraction was attempted), empty array is meaningful (extraction
ran and found none).

### Prompts as files — substrate convention

Tool prompts live in JSON files alongside the tool's JS body, per
the substrate's `toolPromptPath` convention (`Orchestrator.js`
line 85). For a tool at `files/Safebots/tools/<base>.js`, the
substrate auto-loads `files/Safebots/tools/<base>.prompts.json` if
present, exposes its keys as the top-level `prompts` binding in tool
scope (parallel to `Streams`, `Action`, `Runtime`), and tracks
`Safebox/promptSha256` separately from `Safebox/sha256` on the tool
stream. Prompt edits show up as a different prompt-hash without
invalidating the code-hash; the audit chain has two layers.

This separation lets prompt content be edited by people who aren't
JavaScript engineers, kept in a single language-neutral file the LLM
itself can read and propose edits to, and verified independently from
code at install time. The `_starterTools` registration computes both
hashes when present; `_ensureToolStream` writes both attributes onto
the seeded `Safebox/tool/*` stream.

The tool body reads prompts as `prompts.<key>` and applies any
template-variable substitution itself. A future round may add
community-level prompt overrides (where a community can replace the
default prompt with one of their own without forking the tool); the
mechanism — prompts living as streams that the substrate's loader
prefers over the file-system fallback — is a Safebox-side concern,
recorded in `handoff/Safebox-extensions-for-Safebots.md`.

### Eager summarization is the substrate

Every uploaded artifact gets observed at ingest. Every bot response
includes a self-summary written in the same generation call as the
response. This is a deliberate cost trade: output tokens dominate
inference cost, summary outputs are small, and eager processing is
cheap relative to the cost it saves on every downstream turn that
would otherwise re-include unsummarized content.

Observation streams are content-addressed:
`Safebox/observation/{focus}/{base64-source-uri}`. `focus` is bounded
vocabulary — `general`, `text`, `layout`, `code`, `data` — not
open-ended. Open `focus` was a DoS vector; a confused model could
spawn unbounded distinct observation streams.

### Replies happen through edit messages

The flow for a bot reply:

1. User posts a `Streams/chat/message` at ordinal N. It commits and
   other participants on the stream see it (or see a placeholder, if
   the chat extension is configured to redact prompts until completion
   in this stream).
2. The bot starts working. It broadcasts an ephemeral working
   indicator — `Safebots/working` — through the existing socket layer.
   Ephemeral, no ordinal consumed.
3. The bot's reply commits as a single `Streams/chat/message` at the
   next ordinal. The reply contains the user-visible response text
   plus `instructions.summary`, `instructions.refs`,
   `instructions.toolCalls`.
4. As artifacts are produced or attached during the turn, **edit
   messages amend the reply in place**. An edit message at a later
   ordinal references the reply's ordinal and lists what to change.

`Streams/chat/edit` is modeled exactly on `Streams/changed` (see
`Streams_Stream::changed()` in the Streams plugin). `Streams/changed`
records "stream fields changed — here's the diff."
`Streams/chat/edit` records "message at ordinal N changed — here's
the diff."

```json
{
  "type": "Streams/chat/edit",
  "instructions": {
    "targetOrdinal": 42,
    "changes": {
      "content": "the new visible text",
      "instructions": {
        "summary": "the new self-summary",
        "refs": ["Streams/image/...", "Streams/file/..."],
        "toolCalls": [...]
      }
    }
  }
}
```

The original message at ordinal 42 is **never modified**. The edit at
ordinal 50 supersedes its rendered state. The chat extension applies
edits client-side at render time: walk messages in ordinal order, for
each `Streams/chat/edit` apply the diff to the in-memory render of its
target. Working-context assembly does the same — for each ordinal it
computes the latest state from the base message plus any subsequent
edits that target it. Causal history sits in the ordinal sequence:
the original prompt, the working indicator, the reply, the edits
attaching artifacts — every event has an ordinal and a timestamp,
none of them rewrite the past.

`Streams/chat/remove` follows the same pattern: an ordinal records
that another ordinal was withdrawn. The original is preserved; the
extension renders it as removed.

### Edits that attach artifacts

The most common edit pattern is "attach an artifact to a reply that's
already committed." The bot generates an image or fetches a file or
produces an observation; instead of posting a separate message, it
posts an edit that adds the artifact's URI to `instructions.refs` of
the reply, and (if appropriate) calls `Streams::relate` to attach the
artifact stream to the conversation stream as a typed relation. The
chat extension renders the new ref inline beneath the reply, using
the `{streamType}/preview` tool family.

Multiple edits to the same target compose: edit A adds ref X, edit B
changes the summary, edit C adds ref Y. The renderer applies them in
ordinal order. If two edits change the same field, last-write-wins by
ordinal — the natural causal interpretation.

The substrate gates which artifact types can be related to a chattable
stream via `Streams/chat/allowedRelatedStreams` config; the default
allowlist covers `Streams/video`, `Streams/audio`, `Streams/pdf`,
`Streams/image`, `Streams/question`, expandable per-deployment.

### Redacting the prompt

For some workflows the user wants the prompt hidden until the reply is
ready — typically in multi-participant streams where half-formed
prompts read as noise. Two modes:

- **`visibility: visible`** (default). Prompt shown immediately. Other
  participants see it at ordinal N. Reply arrives at N+1.
- **`visibility: hidden_until_response`**. Prompt commits at ordinal N
  with `visibility` marking it hidden. Other participants see a
  placeholder. When the reply commits at N+1, an edit message at N+2
  changes the prompt's `instructions.visibility` to `visible`. The
  rendered transcript then shows both.

The ordinals exist either way. Causal history is intact whether the UI
renders the prompt immediately, blurs it, or hides it entirely.
Auditors and replays see the same sequence regardless of rendering
choice.

### Solo vs multi-participant streams

The mechanism is identical; the visibility defaults differ.

A **solo artifact** — one human plus a bot, both with `post`
writeLevel — runs the orchestrator aggressively. The user is the only
audience for their own in-flight prompt. Default `visibility: visible`,
working indicators inline, edits attach artifacts as they arrive. The
user sees the turn evolve live.

A **multi-participant artifact** — multiple humans plus one or more
bots all with `post` writeLevel, typically because the artifact lives
under a community publisher and the community has invited members —
defaults to `hidden_until_response` for prompts addressed to bots.
Other participants don't see the prompt until the reply is ready. A
half-formed prompt sitting visible while a bot works for ten seconds
reads as noise to other humans; a redacted-then-revealed prompt
produces a cleaner conversation flow. Configurable per-stream via
`Safebots/chatPolicy` attribute or per-community via config default.

A bot in a multi-participant stream only responds when (a) the
message's `instructions.mentions` includes the bot's userId, or
(b) the message text starts with the bot's display name, or (c) a
goal stream is active and the message matches the goal's trigger
pattern. Random chatter between humans is not a prompt. The
`Safebots/active` + `Safebots/handlers` machinery from §4 governs
which (messageType, streamType, fromType) tuples fire which tools;
the dispatcher already filters in-memory per batch.

### Tools, dedup, and the tool-call ledger

Within a turn, the model deepens on demand through tools. Stream
navigation (`fetch/stream`, `list/related`, `search/streams`) lets it
pull more of the artifact's neighborhood — siblings under the same
category, prior versions, related questions. Artifact slicing
(`range`, `toc`, `diff`) handles deep dives into one
artifact. Focused observation (`vision.observe`, `text.summarize`)
and writes through `Action.propose` complete the surface.

Tool calls are hashed with canonicalized arguments via the existing
RFC 8785 utility. Identical calls within a turn dedupe to one cache
entry. The transcript's `instructions.toolCalls` records the hash and
a resultRef — not the raw response — so future turns that need the
same content re-issue the call and hit the cache, rather than
re-embedding the response in the prompt. Long, deeply tool-augmented
sessions stay small.

### Compaction

When working context approaches the budget cap (default 32K tokens,
trigger at 80%), a compaction pass runs over the oldest contiguous
span of verbatim chat messages on the stream and produces a
`Safebots/digest` stream consolidating them, related to the artifact
via `Safebots/digest`. Future assemblies use the digest in place of
the individual messages.

The compaction pass is **append-only and ref-aware**. Original
messages are never modified; the digest is a derived layer that
assembly prefers. Critically, before digesting span [a, b], the pass
checks whether any message in the still-verbatim window references
any ordinal in [a, b] via `instructions.refs` or `replyTo`. If so,
the referenced messages are preserved and only the unreferenced
subset is digested. Otherwise a recent "remember what I said earlier
about the pricing model?" loses its referent.

### What's implemented today vs Round 18

This section's design is partly shipped and partly pending. What
exists in code right now (Round 18-prep):

- `Safebots_Context` (PHP) and `Safebots.Context` (Node) — the
  chat-orchestration helpers described above. Transcript walking
  with edit replay, multi-participant detection, chat-policy
  resolution, instruction-blob normalizers, the bounded
  `FOCUS_VOCABULARY`, and the canonical observation-stream
  naming (`Safebox/observation/{focus}/{base64-uri}`) all work
  identically across both languages.
- `Safebots_Bot::shouldRespondTo` and the chat gate inside
  `_dispatchMessage` — bots in multi-participant streams require
  mention / name-prefix / goal-trigger before firing. Solo streams
  bypass the gate; handlers can opt out via `bypassChatGate: true`.
- `Safebots_Bot::postEdit` plus `Safebots/internal/postEdit` HTTP
  endpoint plus `Safebots.postEdit` Node helper — the round trip
  for amending a prior reply with content / refs / visibility
  changes.
- `Safebots/internal/context` HTTP endpoint — the Node-only read
  channel for transcript / participant / policy phases that need
  PHP-side DB access.
- The chat extension (`web/js/tools/safebots/chat.js`) renders edits
  in place, hides edit/remove rows, and applies the
  `hidden_until_response` and `redacted` placeholder for non-author
  viewers. Once a visibility flip arrives, the placeholder dissolves.
- `Safebots/digest` stream type registered for compaction outputs;
  no compaction tool yet writes them.
- `Streams/graph` SVG visualization tool (Round 65) — node-link
  graph rendering for a publisher's stream-and-relation neighborhood,
  mobile-friendly with touch-pan/pinch-zoom, click-to-expand
  via `Streams.related` walks. Lives in the **Streams plugin** as
  `Streams/web/js/tools/graph.js` alongside `Streams/tree`; both
  visualize the same graph data, tree as a hierarchical
  Q/expandable-driven view and graph as a force-directed view.
  The tool is purely a viewer — it reads via the standard substrate
  access cascade and never writes.
- `Safebox/tool/graph/explore` (Round 65) — chat-invoked tool that
  walks a stream's neighborhood, calls `Runtime.llm` for a
  natural-language summary, returns layer-2 metadata in the
  canonical `instructions` schema (entities, refs, summary), and
  optionally proposes `Streams/relate` / `Streams/unrelate`
  mutations when the user's query implies a graph edit. Registered
  in `_starterTools` with declared action types
  `[Streams/relate, Streams/unrelate]`; the `Safebots/bot/explore`
  bot's handler entry declares the per-handler bounds the
  contract judgment enforces. This tool predates the
  workflow-three-categories split (§22): until
  `verify/safebox/tool` ships the disjunction-rejection rule,
  explore runs in the simpler single-tool form (it both calls
  `Runtime.llm` AND proposes actions), with the contract judgment
  as the safety boundary. The tool body is structured in
  clearly-marked phases so the eventual decomposition is mechanical.

What awaits Round 18:

- The chat workflow — `Safebots/workflow/chat`, a Safebox workflow
  definition that sequences three steps: context expansion (calls
  Safebots-authored context tools), decide (calls a Safebots-authored
  decide tool that uses `Runtime.llm`), and act (dispatches a
  Safebots-authored action tool to post the reply). The workflow is
  itself a governance-approved artifact with its own sha256.
- The Safebots-authored tools the workflow composes:
  `Safebots/tool/context/*` (transcript, related, preview, possibly
  per-phase wrappers around Safebox's seven `enrich*` methods);
  `Safebots/tool/decide/chat` (LLM-driven, returns structured intent;
  forbidden from proposing actions); `Safebots/tool/act/postMessage`
  and `Safebots/tool/act/postEdit` (typed inputs, narrow action types,
  no `Runtime.llm` call). Plus the eventual decomposition of
  `Safebox/tool/graph/explore` into `context/graph-walk` +
  `decide/graph` + `act/relate` once the disjunction rule lands.
- The compaction tool — runs over the oldest contiguous span of
  verbatim chat messages, produces a `Safebots/digest` stream,
  preserves messages referenced by the still-verbatim window.
- Eager-summarization tools — observe-at-ingest for uploads and
  self-summary at generation time.
- Per-tenant cache scoping at the inference call (`X-Cache-Tag:
  safebots/{publisherId}/{streamName}`) — passed in the `Runtime.llm`
  spec; relies on Safebox's vLLM adapter which supports this tag
  natively.

Round 18 depends on one Safebox-side substrate change: an event hook
`Safebox/Orchestrator/buildMethods` fired in `Orchestrator.runTask`
after the sandbox methods object is assembled, letting plugins augment
the map with their own namespaced primitives. Safebots's listener
in `classes/Safebots.js` is already wired (defensively — registers
even if Safebox has not shipped the hook yet) and contributes
`safebots.*` methods via `Safebots.Context.exposeForSandbox(ctx)`.
The companion handoff doc to the Safebox team specifies the change.

### Replayability

Three properties hold by construction:

1. **The artifact stream is sufficient.** Given the stream, system
   prompt, and tool definitions, any turn's working context is
   reconstructable. Cache state is irrelevant to reconstruction.
2. **Bounded prompt size.** Working-context assembly never exceeds
   the configured budget regardless of conversation length or
   artifact-tree size. Compaction handles the long tail; eager
   summarization handles the wide one.
3. **No hidden state.** Every fact the model uses on a turn is in the
   stream or fetchable through tools. Removing the stream and rerunning
   produces equivalent behavior.

The cost of these properties is paid up front, in eager summarization
at ingest and self-summary at generation. The benefit compounds —
every downstream turn assembles from summaries instead of raw content,
every edit composes by ordinal-order replay, every audit reproduces
from the stream alone, and the same machinery handles solo
working-with-the-bot and a community discussion of the same artifact
without any structural change.

---

## 12. Relations as Emissions — Channels

`Streams::relate()` is the universal emission primitive. Sending an email,
posting a tweet, emitting an AI summary — all modeled as relating the
content stream to a channel stream. The `Streams/relatedTo` message IS the
record, delivered via the existing notification cascade (see Safebox.md §9
and the `Streams_Stream::notifyParticipants` pipeline).

### Channel naming

```
{communityId} / Streams/incoming/imap/{emailAddress}
{communityId} / Streams/outgoing/smtp/{emailAddress}
{communityId} / Streams/incoming/telegram/{chatId}
{communityId} / Streams/outgoing/telegram/{chatId}
{communityId} / Streams/incoming/com.twitter/{handle}
{communityId} / Streams/outgoing/com.twitter/{handle}
```

All channel streams for a customer relate to the customer's
`Assets/holding/{customerId}` stream.

### Inbound email

Incoming emails are materialized as `Safebox/job/email/{provider}/{messageId}`
streams published by the community (see Safebox.md §3 for the attachment
sub-stream layout). Once created, the community's `Safebots/bot/*`
configs can trigger on `stream-create` events matching `Safebox/job` to
route incoming mail through reviewers, route to tickets, generate
auto-replies, etc. — all as normal behavior dispatch.

---

## 13. Artifact Lifetime & Refcount Model

An artifact (a draft proposal, a generated image, a spec document) has a
lifetime within a chat: `[firstRef, lastRef]`. Messages in the chat can
reference the artifact via `instructions['Safebots/artifact']` — the
refcount is the count of messages whose instructions cite the artifact.

When the refcount drops to zero AND the artifact isn't related-out to any
persistent category (publication, archive, external channel), it's eligible
for pruning. The `safebox_artifact_ref` table (Safebox.md §19) tracks
refcounts.

Artifacts that are promoted (via votes — see §14) get a durable relation
to the community's publication category, which holds a refcount above zero
permanently until explicit demotion.

---

## 14. Voting, Versions & Promotion

Multiple versions of an artifact (forks of a draft, alternative proposals)
can coexist. Promotion from "candidate" to "published" is governed by
`Users_Vote` rows on the candidate artifact streams, tallied by a community
policy.

### Proposal flow

1. Behavior produces artifact version V1 → behavior's tool calls
   `Actions.propose('Streams/create', {...})` with `Safebots/artifact` as
   the stream type.
2. Chat message references V1 via `instructions['Safebots/artifact']`.
3. If V1 is a candidate for publication, a parallel
   `Actions.propose('Streams/relate', {...})` with relation type
   `Safebots/candidate` from the publication category to V1.
4. Community members vote on V1 via `Users_Vote` rows.
5. When a vote causes V1 to become the top candidate, a proposal to
   promote it — `Actions.propose('Streams/relate', {...})` with relation
   type `Safebots/promoted` — is evaluated. The community's promotion
   policy checks the current vote tally; if V1 is the winner, the
   promotion relation commits.
6. The previous promoted relation (if any) is superseded by a matching
   `Streams/unrelate` proposal, evaluated under the same policy.

### Reversible vs irreversible

Some actions cannot be unwound — an email is sent, a tweet is posted, a
payment is made. Those live in Safebox as executor capabilities with
appropriately strict policies (often higher thresholds than content
governance). Chat-internal operations (add a message, relate a stream,
update an attribute) are reversible and can run under looser policies.

---

## 15. Safebots/goal/chat — JS Chat Extension

The goal-directed chat extension is a Streams/chat extension that reads the
chat's goal relation, interpolates the system prompt, and hooks message
rendering + posting.

### Key implementation notes

**Prerequisite declaration.** The tool declares `["Streams/chat"]` as
prerequisite in `Q.Tool.define`, so Qbix guarantees the chat tool loads
first. The tool reads `chatTool.state.stream` and defers via
`onRefresh.add(...)` if the stream isn't yet loaded.

**`onMessageRender` and `beforePost` hooks are tool-keyed.** Multiple chat
extensions can coexist on the same element; each registers its handler
with its own tool key (`.set(handler, tool)`) and they don't collide. The
attribution-strip extension (`Safebots/safebots/chat`) and the
goal-directed extension (`Safebots/goal/chat`) run on the same chat and
read different `instructions` keys.

**Posting as the publisher vs as the user.** Autonomous bot posts — the
greeting, goal-respond callbacks — use the chat publisher as `byUserId`.
Suggestion-accepted posts use the human user as `byUserId` and rely on
the attribution strip to show AI involvement. The behavior's governance
mode determines which path is taken: `autonomous` = community posts
directly; `propose` = proposal queues for review.

**`beforePost` attribution tagging.** When the user accepts an inline
suggestion, the chat extension stashes the attribution metadata on
`tool._pendingVia`. The `beforePost` hook reads this and adds
`instructions['Safebots/via']` to the outgoing message. The attribution
travels with the message forever.

### Dispatch from the chat extension to Safebots/suggest

The chat extension calls `Q.req('Safebots/suggest', ['suggestions'], ...)`
when the user taps the menu item or triggers the trigger character. Results
come back as `data.slots.suggestions`, each entry carrying
`{text, source, agent, promptRef, artifactKind}`. Clicking a suggestion
fills the composer and stashes the attribution.

---

## 16. Capability & Tool Generation from Chat

Communities can grow their own capabilities and tools by having Safebots
chats guide the design. A dialog of type `capability` or `tool` uses a
platform goal whose prompt walks the community through spec, then hands
the spec to a generator tool that produces the code and proposes
`Actions.propose('Streams/create', ...)` with the appropriate stream type.

The resulting stream carries `Safebox/pendingCode` (per Safebox.md §9) —
the code file is written to disk at execute time and sha256-verified.
`Safebox/approved` remains `false` until a `Streams/update` proposal (the
audit) sets it true, gated by the community's `Safebox/tool` or
`Safebox/capability` update policy — typically requiring M-of-N approvals
from a `Safebox/auditors`-style label (per Safebox.md §15.4).

This is exactly the same flow community policies themselves go through
(§15.4 of Safebox.md). A community that wants new bot behavior writes the
tool in chat, proposes its creation, proposes its audit, and once both
proposals have crossed their governance thresholds the behavior is
available to reference.

---

## 17. PHP Implementation Guide

### `Safebots_Bot`

```php
class Safebots_Bot {

    /**
     * Enumerate all active bots on a publisher.
     * One Streams::related call, memoized per-request by the dispatcher
     * so multiple messages under the same publisher in a batch don't
     * re-query. Filters bots whose Safebots/active attribute is boolean true.
     */
    static function activeBotsForPublisher($publisherId);

    /**
     * Universal dispatch entry. Called by the Streams/postMessages
     * after-hook with the full posted batch. For each (publisherId,
     * streamName) pair, enumerates active bots and fires every matching
     * handler. Exceptions are caught and logged — a bot's failure must
     * never take down the calling write.
     *
     * @param array $streams  [$publisherId][$streamName] => Streams_Stream
     * @param array $posted   [$publisherId][$streamName] => [Streams_Message,...]
     * @param bool  $skipAccess
     */
    static function dispatch($streams, $posted, $skipAccess = false);

    /**
     * Seed the default bots for a publisher. Creates Safebots/bots
     * index category, Safebots/bot/suggest, Safebots/bot/greet, and
     * relates each to the index with relation type Safebots/bot.
     * Idempotent.
     */
    static function seedDefaults($publisherId);
}
```

Matching filter logic inside `dispatch`:

```
for each posted message M on (publisherId P, stream S):
    for each active bot B on P:
        H = B.Safebots/handlers[M.type]
        if !H or !H.active: skip
        if H.streamType and H.streamType != S.type: skip
        if H.fromType and counterpartType(M) != H.fromType: skip
        if M.byUserId == P and H.suppressSelfAuthored: skip
        sendToNode('Safebots/invokeTool', { tool: H.tool, ...context })
```

### `Safebots::ensureActivation`

```php
static function ensureActivation($publisherId, array $actionTypes = []);
```

Seeds the full activation set for a community in one idempotent call:

- `Safebox/judgments` and `Safebox/policies` registry categories
- `Safebots/registered-bots` content stream (JSON array of trusted
  bot publishers)
- `Safebots/sensitive-categories` content stream (JSON array of
  category names whose entries trigger side effects)
- Three `Safebox/tool/*` streams for the starter tools (greet/chat,
  greet/goal, suggest/messages), with `Safebox/codeFilePlugin: 'Safebots'`
  and `Safebox/sha256` computed from the on-disk file contents
- `Safebox/judgment/Safebots/approve-bounded-bot-actions` — the
  contract-enforcement judgment owned by the community
- Per-action-type `Safebox/policy/Safebots/<slug>` voting policies
  (threshold 1)
- Registry relations from the judgment to `Safebox/judgments` (one
  per action type)
- Activation relations from the judgment INTO each per-type policy

All seeded streams carry `Safebox/scope: 'platform-default'` — metadata
for the substrate's self-promotion guard where it ships. The actual
enforcement of who can create these streams at seed time is
`skipAccess: true`, which only the platform publisher has during the
seed call.

Idempotent: existing streams are detected and skipped; relations catch
duplicate-key exceptions quietly. Re-running is safe.

Called from:
- `scripts/Safebots/0.1-Streams.mysql.php` at install (for the hosting
  community).
- Any community onboarding Safebots — typically when
  `Safebots_Bot::seedDefaults` is called under a new publisher.

### `Safebots::registerBotPublisher` / `unregisterBotPublisher`

```php
static function registerBotPublisher($communityId, $botPublisherId);
static function unregisterBotPublisher($communityId, $botPublisherId);
```

Maintain the community's `Safebots/registered-bots` JSON list. The
activation judgment reads this list to decide whether a proposing
tool's owner is authorized. Idempotent — adding an already-listed
userId is a no-op, removing an absent one likewise.

`registerBotPublisher` is called automatically from `Bot::seedDefaults`
for the community itself. Communities authoring custom bots whose
publisher differs from the community call `registerBotPublisher`
explicitly during their onboarding.

### `Safebots::defaultActionTypes`

```php
static function defaultActionTypes(): array;
```

Returns the set of Safebox action types the seeded default bots' tools
produce. Currently `['Streams/update', 'Streams/message',
'Streams/relate']`. Communities authoring custom bots extend this with
the types their bots' tools will propose, and call
`ensureActivation` under their own publisher.

### `Safebots_Goal::buildSystemPrompt`

Follows the chat's `Safebots/goal` relation, interpolates the template,
returns the resulting prompt.

```php
static function buildSystemPrompt($chatPublisherId, $chatStreamName) {
    // Use platform-default community as asUserId so cross-community reads
    // of platform goals succeed without community-specific grants.
    $botId = Users::communityId();
    list($relations, $goalStreams) = Streams::related(
        $botId, $chatPublisherId, $chatStreamName,
        false,  // what THIS stream relates TO
        array('type' => 'Safebots/goal', 'limit' => 1, 'skipAccess' => true)
    );
    if (!$goalStreams) return '';

    $goal    = reset($goalStreams);
    $fields  = $goal->getAttribute('Safebots/fields',  array());
    $context = $goal->getAttribute('Safebots/context', array());
    $urls    = Q::ifset($context, 'urls', array());

    $all = array_merge((array)$fields, (array)$urls);
    return $all
        ? Q::interpolate($goal->content, $all)
        : $goal->content;
}
```

`skipAccess => true` is legitimate here — it's an infrastructure read from
the community's own code against a stream the community owns or a
platform-default goal stream. It is NOT a governance bypass for bot writes
(which go through `Actions.propose` and ride the activation policy).

### `Safebots_Dialog`

```php
static function create($communityId, $dialogType, $draftName, $goalFields)
```

Creates the dialog stream under `$communityId`, relates to goal, sets
attributes. Idempotent.

```php
static function complete($publisherId, $streamName, array $result)
```

Marks status done, posts completion message under the community's identity
(the publisher of the dialog stream).

### Handlers

- `handlers/Safebots/suggest/response.php` — thin dispatcher (§7)
- `handlers/Safebots/dialog/response.php` — create dialog from client request
- `handlers/Safebots/dialog/complete.php` — mark dialog done
- `handlers/Safebots/goal/respond/response.php` — user posted chat → fire LLM
- `handlers/Safebots/goal/message/response.php` — Node callback with LLM output
- `handlers/Safebots/after/Streams_postMessages.php` — universal dispatcher hook
  that routes every substrate message batch through `Safebots_Bot::dispatch`
- `handlers/Safebots/after/Q_Plugin_install.php` — installer entry point,
  triggers seed script + activation
- `handlers/Safebots/internal/postEdit/response.php` — node-internal
  endpoint for posting `Streams/chat/edit` messages
- `handlers/Safebots/internal/context/response.php` — node-internal
  endpoint for context-assembly primitives (`assemble`, `renderStream`,
  `transcript`)

### `Safebots::displayNameOf($userId)`

Resolves a userId to a display name via `Streams_Avatar::fetch`, with a
defensive fallback to the raw userId on lookup failure. Used by the
attribution builder.

```php
static function displayNameOf($userId) {
    if (!$userId) return '';
    try {
        if (class_exists('Streams_Avatar')) {
            $avatar = Streams_Avatar::fetch($userId, $userId, false);
            if ($avatar && method_exists($avatar, 'displayName')) {
                $dn = $avatar->displayName(array('short' => true));
                if ($dn) return $dn;
            }
        }
    } catch (Exception $e) {
        Q::log('Safebots::displayNameOf: avatar lookup failed: '
            . $e->getMessage());
    }
    return $userId;
}
```

---

## 18. JavaScript Implementation Guide

### `web/js/Safebots.js` — Browser Bootstrap

Registers all Safebots tools via `Q.Tool.define` and maintains the
`Q.Safebots.Chat.extensions` array listing chat extensions that activate
automatically on any `Streams/chat` tool.

```js
Q.Tool.define({
    "Safebots/goal/chat":     { js: "...", css: "...", text: [...] },
    "Safebots/safebots/chat": { js: "...", css: "...", text: [...] },
    "Safebots/stream/viewer": { js: "...", css: "...", text: [...] }
});

Q.Safebots.Chat = {
    extensions: ['Safebots/goal/chat', 'Safebots/safebots/chat']
};
```

Platform code that renders chats iterates this registry and activates each
extension on the chat element, so adding a new extension doesn't require
touching chat-rendering call sites.

---

## 19. Node.js Implementation

The Node side listens for IPC dispatched by PHP's `Q_Utils::sendToNode`.
Methods are registered against the IPC server returned by `Q.listen()`:

```js
var server = Q.listen();

server.addMethod('Safebots/goal/respond', function (parsed) {
    // fire-and-forget; Protocol.LLM then sendToPHP back
});

server.addMethod('Safebots/invokeTool', function (parsed, req, res, ctx) {
    // reply-mode: Safebox sandbox runs the behavior's tool, res.send(result)
});
```

**Note on IPC APIs.** `Q.Socket.onEvent(...)` is the *browser-side* socket
API; it does not route `sendToNode` traffic. Node IPC goes through the HTTP
dispatcher registered by `Q.listen`, which exposes `server.addMethod(name,
handler)`. Use `addMethod`. Reply-mode handlers take the four-arg form
`(parsed, req, res, ctx)` and call `res.send(JSON.stringify({slots: {...}}))`.

### Cross-plugin imports

Qbix merges every installed plugin's `classes/` directory into the Node
module resolution path. Use bare names for cross-plugin imports:

```js
var Protocol = Q.require('Safebox/Protocol');    // ← bare name
// NOT:
// var Protocol = require('./Safebox/Protocol'); // ← fails with MODULE_NOT_FOUND
```

### `asUserId = null` semantics

On PHP, omitting the first arg of `Streams_Stream::fetch()` resolves to
`Users::loggedInUser()` — there's a session. On Node, there is no PHP
session; `null` resolves to the public viewer. Any Node-side fetch that
needs authoritative access must pass an explicit identity. Callbacks to
PHP carry the identity explicitly in their payload for exactly this reason.

### API keys

Protocol.LLM substitutes credential placeholders at call time. Credentials
live in `Safebox/credential/{keyName}` streams published by the community
(see Safebox.md §13). Use the `{{credentials:keyName}}` placeholder in
config:

```js
var apiKey = Q.Config.get(['Safebots', 'llmApiKey'], '{{credentials:openai_api_key}}');
Protocol.LLM({ model: model, apiKey: apiKey, messages: messages, ... });
```

---

## 20. UI Reference

### Layout — Q/columns

Safebots UI lives inside Q/columns where provided by the host app. The chat
is one column; overlays (artifact detail, history, branching) open as
adjacent columns, not modals. The chat extension's `_openArtifactChat`
uses `Q.invoke` to open a new column containing a chat scoped to the
artifact stream, with the same extensions active, so iteration recurses.

### Status badges

Dialog and artifact streams render with status badges reflecting
`Safebots/dialogStatus` or `Safebots/status` attributes:

- `drafting` — gray
- `specReady` / `candidate` — blue
- `done` / `promoted` — green
- `rejected` / `abandoned` — red

### Via-strip styling

The attribution strip is a small, low-weight pill above the message content.
Font is 11px, background is a low-opacity gray, text color is the secondary
text color — intentionally unobtrusive. Attribution is inspectable but
shouldn't pull attention away from message content.

---

## 21. Install & Seed

### What the installer does

```php
function Safebots_after_Q_Plugin_install($params)
{
    if (!isset($params['plugin']) || $params['plugin'] !== 'Safebots') return;

    // Seed default bots, platform goals, and the Safebox activation policy
    // under the hosting community. No bot-user provisioning — the community
    // is its own bot. No access grants — the community has max access on
    // its own streams.
    require_once SAFEBOTS_PLUGIN_DIR . DS . 'scripts' . DS . 'Safebots'
        . DS . '0.1-Streams.mysql.php';
}
```

### What the seed script does

Resolves the hosting app's community once at the top:

```php
$botId = Users::communityId();
```

Creates, all under `$botId`:

- **`Safebots/goals`** — the platform goal index category.
- **Platform-default goal streams** (`Safebots/goal/collect/profile`,
  `Safebots/goal/generate/capability`, etc.) related to the goals index.
- **Three starter tool streams** following the verb-first convention,
  authored from pre-audited source files that ship with the plugin at
  `files/Safebots/tools/`:
  - `Safebox/tool/suggest/messages` — inline-suggestions tool (sync reply mode).
  - `Safebox/tool/greet/chat` — writes the storefront greeting when a
    chat is created.
  - `Safebox/tool/greet/goal` — refreshes the storefront greeting when
    a goal stream is attached to a chat.
  Each tool's source file is copied into Safebox's `files/tools/`
  directory at the path declared in the tool's `Safebox/codeFile`
  attribute (e.g. `suggest/messages.js`, `greet/chat.js`); the sandbox
  loader resolves it from there. `Safebox/sha256` is computed and
  `Safebox/approved` is set to `'true'`. Pre-approval is legitimate
  here because the hosting community's decision to install Safebots
  IS the approval for the starter set. Community-authored tools that
  arrive later flow through the standard M-of-N audit pipeline.
  Each starter tool stream is auto-related to the canonical
  `Safebox/tools` index by Safebox's
  `handlers/Streams/create/Safebox/tool.php`; no Safebots-specific
  sub-index is created.
- **`Safebots/bots`** — the bots index category.
- **`Safebots/bot/suggest`** — default inline-suggest bot with a handler
  for `Safebots/suggest/request` referencing
  `Safebox/tool/suggest/messages`.
- **`Safebots/bot/greet`** — default storefront greeting bot with handlers
  for `Streams/created` and `Streams/relatedTo` referencing the two
  starter greet tools (`Safebox/tool/greet/chat` and
  `Safebox/tool/greet/goal`).
- **`Safebox/policies`** — the Safebox policy index for this community.
- **`Safebox/judgments`** — the Safebox judgments registry for this
  community, used by Safebox's dispatcher to find judgments owned by
  the community-as-signer.
- **`Safebots/registered-bots`** — content stream holding a JSON
  array of userIds whose tools the activation judgment trusts.
  Initialized to `[$communityId]` since default bots are owned by
  the community itself. `Safebots::registerBotPublisher` extends it
  when custom bots from other publishers are added.
- **`Safebots/sensitive-categories`** — content stream listing
  category names whose entries trigger side effects (email send,
  payment, public feed, etc.). Empty at install. The judgment
  refuses any `Streams/relate` whose destination is on this list,
  regardless of declarations.
- **Three `Safebox/tool/*` streams** — `greet/chat`, `greet/goal`,
  `suggest/messages`. Each carries `Safebox/codeFile`,
  `Safebox/codeFilePlugin: 'Safebots'` (so SandboxRunner loads from
  the Safebots plugin's files dir), `Safebox/sha256` (computed from
  the on-disk file at install), `Safebox/actionTypes` declaring what
  the tool may propose (`["Streams/update"]` for greet, `[]` for
  suggest), and `Safebox/scope: 'platform-default'`.
- **`Safebox/judgment/Safebots/approve-bounded-bot-actions`** —
  contract-enforcement judgment owned by the community. Carries
  `Safebox/codeFile: 'approve-bounded-bot-actions.js'`,
  `Safebox/sha256` (computed from the on-disk file at install time),
  `Safebox/owner: $communityId`, `Safebox/approved: 'true'`,
  `Safebox/scope: 'platform-default'`. The platform-default scope
  is metadata for the substrate's self-promotion guard where the
  guard ships; seeding itself happens under `skipAccess: true`.
- **`Safebox/policy/Safebots/<slug>`** streams — one per default
  action type, `Streams/update` → `update`, `Streams/message` →
  `message`, `Streams/relate` → `relate`. Each is a voting policy
  with `approvalThreshold: 1`, also carrying
  `Safebox/scope: 'platform-default'`.
- **Relations from each policy into `Safebox/policies`** — relation
  type = action type. This is how Safebox's dispatcher finds the
  policy when an action of that type is proposed.
- **Registry relations from the judgment into `Safebox/judgments`** —
  one per action type, relation type = action type. This is how
  Safebox's dispatcher finds the judgment when an action arrives
  under one of the matching policies.
- **Activation relations from the judgment into each policy** —
  relation type `'Safebox/judgment'`. This is what tells Safebox's
  dispatcher to actually run the judgment against actions landing
  on that policy.

All creates are idempotent via `Streams_Stream::fetch(..., throwIfMissing:
false)` + skip-if-present; all relates catch duplicate-key exceptions
quietly. Re-running the seed is safe.

Other communities that want to opt into Safebots run an analogous seed
under their own publisher — either via an admin UI action or directly:

```php
Safebots_Bot::seedDefaults($theirCommunityId);
Safebots::ensureActivation(
    $theirCommunityId,
    Safebots::defaultActionTypes()
);
```

Communities authoring custom bots that propose additional action types
call `ensureActivation` again with those types appended; existing
artifacts are detected and skipped, new policies + activations get
added.

For custom bots owned by a publisher OTHER than the community itself,
also call `Safebots::registerBotPublisher($communityId, $botPublisherId)`
so the judgment recognizes that publisher's tools as authorized.

### What the installer does NOT do

- Does not create a bot user (there isn't one).
- Does not write access rows (the community has its own streams covered
  by publisher privilege).
- Does not rename anything from prior versions. Deployments from before
  the composition-model conversion that had streams published under a
  literal `'Safebox'` user string or a provisioned `'Safebots'` bot user
  need a manual re-seed under the hosting community — the target community
  id depends on the deployment and can't be inferred by a migration
  script.
- Does not author community-bespoke tools. The three starter tools
  (`Safebox/tool/suggest/messages`, `Safebox/tool/greet/chat`,
  `Safebox/tool/greet/goal`) ship pre-audited and handle the
  default suggest + storefront-greeting flows; communities wanting
  customized suggest voice, richer greeting templates, or entirely new
  bot behaviors go through Safebox's capability/tool generation
  pipeline (Safebox.md §15.4) — Safebots dogfoods that pipeline to
  generate its own community-specific tools.

---

## 22. Appendix: Configuration Reference

```json
{
  "Safebots": {
    "llmApiKey":            "{{credentials:openai_api_key}}",
    "defaultModel":         "gpt-4o-mini",
    "maxUserMessageLength": 4000,
    "maxComposerLength":    2000,
    "suggestTimeoutMs":     10000,
    "models": {
      "fields":   "gpt-4o-mini",
      "artifact": "gpt-4o",
      "process":  "gpt-4o",
      "codeGen":  "claude-sonnet-4-6"
    }
  }
}
```

Notes:
- The platform bot user field (`Safebots/safebotUserId`) that older drafts
  documented has been removed. There is no bot user.
- The bot-access migration keys (`Safebots/defaultBotWriteLevel`,
  `Safebots/botAccessiblePrefixes`) are likewise removed — there is no
  install-time access provisioning.
- Credentials use the `{{credentials:name}}` syntax and resolve against
  `Safebox/credential/{name}` streams (Safebox.md §13).

---

*End of Safebots Plugin Architecture Reference.*
*Version: 2026-04 · companion to Safebox.md*
