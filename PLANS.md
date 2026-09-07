<!--
SPDX-FileCopyrightText: 2026 JamesXNelson
SPDX-License-Identifier: MPL-2.0
-->

# Craftable AI Apprentices — implementation plan

**Planning session:** September 6, 2026, America/Regina.  
**Repository:** `JamesXNelson/CC-Tweaked`, branch `mc-1.20.x`.  
**Source baseline inspected:** `358a6e6263decab39856e6ad7f2ff5cdbf5e5ac2`.  
**Status:** Documentation only. Features, schemas, paths for new code, commands beginning `/ai`, and milestones below are proposed unless explicitly identified as existing upstream behavior.  
**Discussion archive:** [BRAINSTORM.md](BRAINSTORM.md).

## Executive decision

Build **a useful Blockly-controlled survival turtle first**. Use CC:Tweaked's existing craftable body and Lua runtime. Add a small versioned program representation, a trusted interpreter, and good execution feedback. Add a paired browser editor before attempting an embedded browser. Then add local Qwen as another program author, authenticated chat delegation, reliable recording, learning from examples, optional frontier review, Create capabilities, and eventually larger bodies.

The local model runs on a separate machine with the existing 8 GB GPU. No cloud account, new GPU, fine-tuning job, custom robot entity, or giant mech is required to make the first version worthwhile.

**The architectural invariant:** a child-written and AI-written program go through the same validation, permission, approval, execution, and evidence path. A model cannot gain new powers by describing them in prose.

**The gameplay invariant:** intelligence changes how the bot chooses actions, not what resources, permissions, sensors, or physical capabilities it owns.

**The delivery invariant:** implement and demonstrate one vertical slice at a time. Do not begin the mech, model-training, or embedded-Chromium projects before a child can save, run, stop, and debug a small block program on a real crafted turtle.

## Navigation

- [1. Requirements and scope](#1-requirements-and-scope)
- [2. Inspected repository and compatibility](#2-inspected-repository-and-compatibility)
- [3. Architecture decisions](#3-architecture-decisions)
- [4. System components](#4-system-components)
- [5. Survival and observation contract](#5-survival-and-observation-contract)
- [6. Identity, ownership, and persistence](#6-identity-ownership-and-persistence)
- [7. Program representation](#7-program-representation)
- [8. Capability and instruction catalog](#8-capability-and-instruction-catalog)
- [9. Trusted Lua interpreter](#9-trusted-lua-interpreter)
- [10. Execution, interruption, and recovery](#10-execution-interruption-and-recovery)
- [11. Blockly editor](#11-blockly-editor)
- [12. Gateway and network protocol](#12-gateway-and-network-protocol)
- [13. Native Minecraft integration](#13-native-minecraft-integration)
- [14. Chat and task lifecycle](#14-chat-and-task-lifecycle)
- [15. Local model integration](#15-local-model-integration)
- [16. Frontier assistance and billing](#16-frontier-assistance-and-billing)
- [17. Recorder and raw data](#17-recorder-and-raw-data)
- [18. Retrieval and learning](#18-retrieval-and-learning)
- [19. Skills and construction](#19-skills-and-construction)
- [20. OneBlock adaptation](#20-oneblock-adaptation)
- [21. Crafting, progression, and upgrades](#21-crafting-progression-and-upgrades)
- [22. Create integration](#22-create-integration)
- [23. Alternate bodies and mech roadmap](#23-alternate-bodies-and-mech-roadmap)
- [24. Threat model and guardrails](#24-threat-model-and-guardrails)
- [25. Performance and operations](#25-performance-and-operations)
- [26. Verification strategy](#26-verification-strategy)
- [27. Milestones and acceptance gates](#27-milestones-and-acceptance-gates)
- [28. Suggested code organization](#28-suggested-code-organization)
- [29. First Codex handoff](#29-first-codex-handoff)
- [30. Development and release discipline](#30-development-and-release-discipline)
- [31. Risks and open decisions](#31-risks-and-open-decisions)
- [32. Completion definitions](#32-completion-definitions)
- [33. Source references](#33-source-references)

## 1. Requirements and scope

### 1.1 User requirements

| ID | Requirement | Consequence |
|---|---|---|
| U01 | Alyx and Mia can program with familiar blocks | Blockly is a first-class authoring surface, not an optional visualizer of opaque AI code |
| U02 | Stay in Java Minecraft and the existing Create/mod ecosystem | Initial target is Minecraft 1.20.1/Forge; no forced upgrade to a newer Minecraft version |
| U03 | A craftable survival bot | Begin with real turtles and recipes; no hidden admin powers |
| U04 | Optional free starter bot | Explicit, persistent, configurable one-time issuance; never a login duplication mechanism |
| U05 | Local model on another machine, currently 8 GB VRAM | Remote inference service, modest context, no GPU purchase requirement |
| U06 | Both humans and models create reviewable imperative programs | Shared IR, versioning, source maps, approval, traces, and repair revisions |
| U07 | Chat-based interaction and delegation | `/ai` task/advice/stop commands with authenticated player identity |
| U08 | Preserve raw play for later Codex analysis | Independent raw journal plus derived datasets; no dependence on the AI service being alive |
| U09 | Learn from demonstrations and corrections | Explicit provenance; retrieval first, fine-tuning only after evaluation |
| U10 | Optional frontier help | Manual export and parent-operated review first; cloud calls require explicit policy and budget |
| U11 | Upgrades and substantial Create integration | Capability-based progression, verified installed-version adapters |
| U12 | Eventually support ambitious mechs | Separate body adapters; preserve spider and pink-sheep ambitions without making them V0 dependencies |
| U13 | Codex implements from durable plans | Small bounded tasks with source pointers, tests, acceptance criteria, and handoff records |

### 1.2 Definition of the first useful product

A child crafts a normal turtle, gives it valid fuel/materials, opens a local block editor, builds a small finite program, saves it, sees a readable preview, explicitly runs it, watches the current block highlight, and can stop it. An obstruction or lack of fuel produces an honest, understandable error. The program remains available for editing after failure.

That experience is a successful product increment **with the AI switched off**.

### 1.3 Not required for the first product

No Python VM, unrestricted code execution, new Java block-editor toolkit, embedded Chromium, training pipeline, vector database, multiple autonomous agents, own-character takeover, universal pathfinder, custom creature AI, moving ship support, or weapons system.

A small interface may anticipate these features, but speculative abstractions must not dominate the first implementation. Add extension points when at least one real use exists; do not implement ten empty backends.

## 2. Inspected repository and compatibility

### 2.1 Verified source facts

The following were read from this fork, rather than inferred from generic CC:Tweaked documentation:

| Source | Finding | Implementation instruction |
|---|---|---|
| [Version catalog](gradle/libs.versions.toml) | Minecraft 1.20.1; Forge build dependency 47.1.0; separate Fabric entries; Create dependencies already appear in the catalog | Preserve the branch target and record actual installed mod versions |
| [Build constants](buildSrc/src/main/kotlin/cc/tweaked/gradle/CCTweakedPlugin.kt) | Development/build JDK 25; Java target 17 | Do not confuse build-tool requirements with shipped runtime compatibility |
| [Architecture](projects/ARCHITECTURE.md) | `core`, `common`, `forge`, `fabric`, API divisions, Lua tests and GameTests | Keep Minecraft classes out of `core`; use the existing loader boundaries |
| [README](README.md) | Public addon API is under `dan200.computercraft.api`; non-API internals are unstable | Prefer public APIs; isolate any required fork hook |
| [Turtle movement](projects/common/src/main/java/dan200/computercraft/shared/turtle/core/TurtleMoveCommand.java) | Checks obstruction, fuel, bounds, protections, loaded world; no floor requirement | Do not apply player gravity assumptions to a turtle |
| [Root package](package.json) | Existing web/documentation tooling | Avoid turning the documentation site's build into a dependency of the bot editor |
| [Contributing](CONTRIBUTING.md) | JDK/Node prerequisites, test commands, and upstream rejection of generative-AI contributions | Work in this fork; do not submit generated contributions upstream implicitly |
| [REUSE](REUSE.toml) and [CCPL](LICENSES/LicenseRef-CCPL.txt) | Mixed per-file licensing | Preserve provenance/notices and review distribution obligations |
| [Build workflow](.github/workflows/main-ci.yml) | Push/PR workflow builds and tests several platforms | Use explicit local evidence; do not assume free hosted capacity |

No Minecraft build or GameTest was run while writing these documents. The source pointers are verified; runtime compatibility remains to be tested.

### 2.2 Runtime baseline to capture at M00

The family environment is constrained to Minecraft 1.20.1; the existing Forge setup has been identified as 47.4.0 in project context. Treat the actual exported launcher/server manifest as authority. The repository's Forge dependency is not an instruction to downgrade the family's installation.

Record exact CC:Tweaked, Forge, Create, OneBlock, and optional addon versions; Java runtime; server versus integrated-server mode; config/datapack hashes; operating systems; GPU model/driver; system RAM; and inference runtime. Do not publish private addresses, account tokens, or family world files.

Test in a backed-up copy or a fresh development profile. Preserve the stable family world and any separate workshop/museum modpack. Do not solve a compatibility problem by upgrading Minecraft without authorization.

### 2.3 Fork versus addon strategy

The user has selected this fork as the working repository. That does not require invasive edits to CC:Tweaked internals.

Start with a fork-owned editor/gateway package and ROM-side Lua code. Introduce native integration through a narrow module/package and the existing public extension points wherever possible. Keep a small documented list of unavoidable fork patches. A later standalone addon is an option, not a prerequisite or a reason to abandon the fork.

Do not install both upstream CC:Tweaked and a replacement fork jar with the same mod ID in one profile. An addon and upstream jar is a different packaging strategy and must be labeled accordingly.

## 3. Architecture decisions

### D01 — Existing turtle first

Use an ordinary CC:Tweaked turtle for the first useful version. Craftability, inventory, movement, and native operations already exist. A custom humanoid, spider, or sheep body is later work.

### D02 — Browser Blockly first

Use a self-hosted local/LAN browser editor. No CDN dependency at runtime. No native Java editor recreation. Embedded-browser support is a later client-only adapter after a compatibility spike.

### D03 — Versioned IR is the semantic authority

Blockly workspace JSON preserves the editing experience; IR defines the executable program. They are different artifacts. Implement both conversion directions for a deliberately restricted block subset. Do not promise arbitrary Lua/Python round-trip.

### D04 — Reuse the Lua runtime

Implement a trusted interpreter over validated IR inside CraftOS, dispatching to ordinary turtle APIs. Do not evaluate model-supplied Lua strings. This avoids a second Java language runtime and keeps physical operations in CC:Tweaked's established path.

Blockly's built-in Lua generator is an optional prototype shortcut or read-only preview source, not a second equally authoritative executor. If a temporary generated-Lua spike is used, mark it as disposable/trusted-input-only and do not connect it to autonomous AI. [S-BLOCKLY]

### D05 — AI authors; runtime authorizes

A model produces a draft, explanation, or proposed patch. It does not receive direct world-admin tools. Approval binds to a concrete program revision and policy. A new model result cannot mutate an already approved running revision.

### D06 — Local-first, provider-neutral

Qwen/Ollama is the initial inference path. Model identifiers and endpoints are configuration, not hardcoded gameplay classes. Cloud assistance is optional, separately permissioned, and off by default.

### D07 — Raw journal and derived views coexist

Preserve game events and selected baselines before semantic compression. Derive tasks, skills, retrieval indexes, and training examples reproducibly. Never replace raw evidence with a model-written summary.

### D08 — Capability-aware embodiments

Programs declare required operations and parameters. Each body/upgrade exposes what it actually supports. Reuse high-level tasks across bodies only when preconditions are satisfied; never pretend a turtle and walking mech have interchangeable motion.

### D09 — Two honesty boundaries

A stock-turtle/Lua prototype is a tool for a trusted family environment, not a hardened multi-tenant anti-cheat system. Strong server-wide restrictions require native enforcement. Likewise, a static preview is not a complete Minecraft physics simulator. Label both limits clearly.

### D10 — No claim of automatic weight learning

Watching, remembering, retrieving, and fine-tuning are different modes. The UI must state which is occurring. Approved demonstrations may improve retrieval immediately; model weights change only through an explicit offline training and evaluation process.

## 4. System components

```text
CHILD'S CLIENT / BROWSER
  Minecraft player identity       Blockly workspace
          |                            |
          | /ai, select, approve       | draft/save/run/step
          v                            v
MINECRAFT HOST                    LOCAL GATEWAY HOST
  Forge integration <----------> authenticated gateway
  authoritative world             program store + revision history
  owned turtles                    compiler/schema validation
  trusted IR runner                task queue + resource leases
  native capability adapters       bounded model requests
  recorder + local spool            Qwen/Ollama on existing GPU
          |                         optional frontier adapter
          v                         retrieval and offline analysis
  raw session journal
```

There are three independent availability questions: is Minecraft running, is the gateway reachable, and is the model available? A model outage must not corrupt an already-running deterministic program. A gateway outage must not replay commands. The recorder must not disappear with either service.

### 4.1 Component responsibilities

| Component | Owns | Must not own |
|---|---|---|
| Blockly editor | layout, edit history, draft preview, approval UI | authoritative inventory, credentials for cloud providers |
| Compiler/validator | supported syntax, type checks, capability requirements, canonicalization | assumptions that unobserved world facts are true |
| Gateway | identities, task/program storage, model calls, budgets, transport | bypassing Minecraft permissions or manufacturing items |
| Lua runner | deterministic execution, yields, trace IDs, primitive results | arbitrary downloaded code or provider secrets |
| Native bridge | authenticated player commands, authoritative policy hooks, safe observations, persistence | blocking inference/network calls on the game tick thread |
| Recorder | raw evidence, sequence/gap information, export selection | automatically approving demonstrations as good training data |
| Skill library | reviewed parameterized programs and verifiers | hidden unrestricted shell/JavaScript/Lua execution |
| Body adapter | truthful physical operations and sensor limits | silently emulating unsupported operations using cheats |

## 5. Survival and observation contract

### 5.1 Non-negotiable world rules

All resource-affecting operations must use the appropriate game/turtle/peripheral interaction path. No `/give`, `/fill`, privileged block replacement, inventory cloning, forced ore discovery, or secret teleport-to-target shortcut.

An internal method named `teleportTo` in CC:Tweaked's normal single-step movement is not the same thing as granting the AI an admin teleport tool. Preserve normal behavior; do not judge survival legitimacy by internal method names.

Fuel, tools/upgrades, inventories, recipes, reach, world borders, loaded chunks, and protection hooks remain meaningful. Existing CC:Tweaked behavior is the starting gameplay contract. Extra fuel charges, durability, health, gravity, or reach restrictions are explicit optional game-design changes, not assumptions.

The bot may not consume items in a player's inventory, chest, factory, or another bot without authorization. Permission to inspect a storage container does not imply permission to withdraw its contents.

### 5.2 Materials and planning

For each proposed construction job, provide a bill of materials, supported substitutions/tags, estimated consumables, required tools/peripherals, storage source, and known uncertainties. A missing resource returns a useful shortfall rather than triggering creative-mode fallback.

A plan may reserve quantities logically to avoid two bots spending the same stock, but a reservation is not ownership of immutable world state. Revalidate immediately before transfer/crafting/placement because players and machines can change inventory concurrently.

Keep reusable buckets, containers, crafting remainders, recipe alternatives, and output counts distinct. For Create processes, work-in-progress materials and probabilistic outputs are not guaranteed finished inventory.

### 5.3 Observation fairness

Maintain separate channels:

1. **Runtime authority:** the server can inspect enough state to enforce a rule or perform an authorized operation.
2. **Bot observations:** only sensors, reach, peripherals, and shared facts allowed by the equipped body and policy.
3. **Recorder/debug evidence:** potentially richer local data, accessible to the parent/developer and never automatically fed to the bot.

Every observation should identify its source, timestamp/tick, scope, and whether it is known, stale, absent, or unavailable. Unknown is not air, safe, empty, or zero. The same visible-state envelope should be used in training and deployment to avoid teaching the model to rely on omniscience.

### 5.4 Work areas and irreversible effects

Work-area grants are bounded in world/dimension and, later, moving-body frame. Protected regions include the OneBlock core, player houses, machines, breeding stock, and emergency return/docking areas as configured by the owner.

Generic excavation must not break the OneBlock core. A dedicated OneBlock-harvest operation may break that coordinate according to the map's intended regeneration behavior.

There is no general world undo. A preview or backup is not a license to act recklessly. Repair plans consume real resources, operate only in approved areas, and need fresh approval when they differ materially from the original task.

## 6. Identity, ownership, and persistence

### 6.1 Identity model

Use a generated `worldId`, stable `botInstanceId`, and a monotonically changing `bodyEpoch`. Keep the underlying CC computer ID as a local association, not the sole globally unique identity. A bot's position and dimension are location, not identity.

The gateway's task IDs, program IDs, approval IDs, run IDs, and action IDs are distinct. Never infer bot ownership from a display name, chat prefix, or user-supplied JSON field.

### 6.2 Roles

Initial roles: server owner/parent, bot owner, authorized collaborator, and observer. Permissions are explicit operations: view state, edit draft, approve/run, stop, grant a work area, use cloud help, export recordings, install upgrades, and transfer ownership.

Any authorized local controller should be able to stop that bot. A family/server-wide emergency stop is an owner capability. Ordinary players must not be able to stop or redirect unrelated bots merely by guessing an ID.

### 6.3 Persistent records

Persist program source and revisions, approved skill versions, bot ownership/bindings, active-run metadata, terminal action results needed for reconciliation, and one-time starter issuance. Never persist provider keys or raw gateway credentials inside shareable program packages.

Breaking a bot pauses its run. Replacing it preserves identity only through a supported data path and requires a new body epoch/binding check. Moving to a different world creates a new world association and invalidates old location grants and approvals.

If two physical instances claim the same bot identity, quarantine remote execution until an owner resolves the conflict. Do not merge inventories, allow both to execute one lease, or silently mint a new privileged identity.

### 6.4 Program portability

A disk/card can carry program source, public metadata, and a content hash. It must not carry cloud access, ownership grants, running leases, or authority to control the previous author's bot.

Published library revisions are immutable. Editing creates a new revision. A run continues its approved revision even when the author changes the draft. Importing a program never automatically runs it.

## 7. Program representation

### 7.1 Package layers

Store three distinguishable artifacts:

- **Workspace:** Blockly JSON, layout, comments, collapsed blocks, and editor metadata.
- **Program IR:** supported semantic nodes, parameters, function definitions, and capability requirements.
- **Run envelope:** actual parameter values, bot/body identity, approval, work-area grants, budgets, and expected revisions.

The workspace does not authorize execution. The model should generate IR or a constrained patch, not arbitrary Blockly internals. The editor projects that IR into blocks through our mapping.

### 7.2 Minimal complete illustrative program

This is a proposed schema fixture for a four-step square, not a command already recognized by CC:Tweaked:

```json
{
  "schema": "apprentice.program/1",
  "programId": "550e8400-e29b-41d4-a716-446655440000",
  "revision": 1,
  "name": "Small square",
  "parameters": [],
  "requires": ["turtle.move", "turtle.turn"],
  "entry": [
    {
      "id": "repeat-square",
      "kind": "repeat",
      "count": {"type": "integer", "value": 4},
      "body": [
        {
          "id": "step-forward",
          "kind": "call",
          "op": "turtle.move",
          "args": {"direction": "forward"}
        },
        {
          "id": "turn-right",
          "kind": "call",
          "op": "turtle.turn",
          "args": {"direction": "right"}
        }
      ]
    }
  ],
  "requestedLimits": {
    "maxInterpreterSteps": 128,
    "maxPrimitiveActions": 8,
    "maxWallTimeMs": 60000
  }
}
```

Server/runner policy can reduce these requested limits; the program cannot raise configured caps. A square can still fail due to obstruction or inadequate fuel. Successful compilation is not a claim that all four destinations are currently available.

### 7.3 Supported language stages

**V0:** ordered statements, literal arguments, bounded repeat, simple primitive calls, stop/report, and stable node IDs.

**V1:** `if/else`, typed variables, comparisons, inventory/inspection observations, finite function calls, parameters, and explicitly bounded waits/loops.

**Later:** task-local event handlers, reusable library modules, bounded state machines, and coordination primitives. Add recursion or concurrency only after their resource and cancellation semantics are specified and tested. Do not accidentally support them because Blockly happens to provide a block.

Primitive types should include finite integer, boolean, bounded text, item/block identifier, direction enum, and typed observation/result records. Identifiers are validated against the current catalog/registries rather than accepted as arbitrary function names.

### 7.4 Validation pipeline

1. Limit input bytes and parse safely, rejecting duplicate/ambiguous keys where the parser permits such ambiguity.
2. Validate schema version, types, collection lengths, unique node IDs, bounded numbers, and supported operation names.
3. Reject unknown executable fields, unsupported nodes, cycles in function dependencies, and excessive nesting.
4. Type-check expressions and variable use; validate parameter ranges and required capabilities.
5. Compute a canonical content hash and a source map.
6. Apply policy: work area, effect categories, consumables, risk level, and approval requirements.
7. Revalidate at the execution boundary; never trust the browser or model's claim that validation succeeded.

Structural schema validity does not prove safe geometry, valid recipes, or task completion. Keep those checks separate.

### 7.5 Canonicalization and round-trip invariants

Define one canonical ordering and serialization for semantic hashing. Exclude presentation-only layout and comments from the semantic execution hash; keep a separate workspace hash if useful.

For every supported block, test:

```text
IR → Blockly → IR = same semantics
Blockly → IR → Blockly = preserved supported program + stable mapping
```

Preserve node IDs across unchanged blocks so traces remain useful. A model-proposed patch references the base revision and node IDs; stale patches fail with a conflict rather than overwriting another child's edits.

The text view initially renders IR read-only. A writable text mode requires a parser for an explicitly documented subset and the same round-trip tests. Do not label pseudocode as working Python.

## 8. Capability and instruction catalog

### 8.1 Catalog contract

Each operation definition should contain: stable ID and version, argument/result schema, required hardware, effect category, observation requirements, cancellation boundary, timeout/step accounting, error codes, and examples.

The same catalog drives Blockly blocks, compiler validation, AI tool documentation, runtime dispatch, trace rendering, and tests. Implement a small shared manifest; avoid independent drifting hand-maintained descriptions.

### 8.2 Initial operations and upstream mappings

| Proposed operation | Upstream behavior to wrap | Caveat |
|---|---|---|
| `turtle.move` | `forward`, `back`, `up`, `down` | Actual result determines success; moving costs configured fuel |
| `turtle.turn` | `turnLeft`, `turnRight` | Preserve orientation accounting |
| `turtle.inspect` | `inspect`, `inspectUp`, `inspectDown` | No magical diagonal/long-range inspection |
| `turtle.dig` | `dig`, `digUp`, `digDown` | Appropriate upgrade and protection behavior; no implicit attack |
| `turtle.place` | `place`, `placeUp`, `placeDown` | Actual selected item and target behavior matter |
| `inventory.select` | `select` | Slot bounds and selection state are explicit |
| `inventory.count` | `getItemCount` / item detail enumeration | Item-ID aggregation is a wrapper, not a built-in universal query |
| `inventory.transfer` | `transferTo` | Internal movement, distinct from external storage transfer |
| `inventory.drop` / `inventory.suck` | directional drop/suck calls | Partial transfer, container capacity, and authorization |
| `fuel.inspect` / `fuel.refuel` | `getFuelLevel`, `refuel` | Do not consume an arbitrary valuable fuel without policy |
| `bot.report` | runner/gateway status event | Not a fictional stock `turtle.say` or chat API |
| `program.wait` / `program.stop` | interpreter scheduling | Always bounded and cancellable |

Check exact signatures and error behavior against the target branch before implementation. [S-TURTLE]

Crafting is a later upgrade-dependent operation with recipe/grid handling, not an automatic consequence of having a crafting table somewhere on the island. External inventory and Create operations are registered adapters, not unrestricted `peripheral.call` exposed to the model.

### 8.3 Composite operations

A child-facing “select cobblestone” can be a deterministic wrapper that searches the bot inventory and selects a suitable slot. A “turn to west” requires known orientation; a “move to marker” requires an implemented navigator and localization.

No primitive called `block_below_ahead` exists merely because it is convenient in pseudocode. A turtle can float and inspect/place below after moving; a grounded body needs different reach/path logic. Composite operations must expose those differences and their failure modes.

### 8.4 Risk classes

Proposed policy classes: read-only; local motion; material transfer; construction; destructive interaction; fluids/fire; combat/explosives; world/identity administration.

V0 exposes only the small class subset needed for the demo. Later approval can grant a bounded class in a selected region. High-risk actions never become available through a prompt-level personality setting.

## 9. Trusted Lua interpreter

### 9.1 Responsibilities

The runner loads a size-limited IR document, verifies its supported schema/capabilities, and executes known nodes through a dispatch table. It never calls `load`, `loadstring`, arbitrary `shell.run`, downloaded module code, or a user-selected host endpoint to execute a model response.

The interpreter should use explicit execution frames for sequence, repeat, conditional, and function call. This makes stepping, budgeting, source mapping, and pausing easier than relying on unrestricted source evaluation.

Each node evaluation consumes an interpreter budget. Each physical operation consumes an action budget. Pure computations must yield periodically too, so an empty loop cannot monopolize the VM until its global watchdog fires.

### 9.2 Scheduling

Use CC:Tweaked's existing Lua event/coroutine behavior; do not mutate Minecraft from a background Java thread. Physical APIs can yield. The runner's scheduler receives completion, stop, pause, and transport events and advances at safe boundaries.

Use one controlled event-dispatch design. Two coroutines independently consuming events can swallow each other's stop or WebSocket messages. Test termination, disconnect, and event ordering with the actual CraftOS environment.

Prefer wrapping existing API results over bypassing the native turtle command path. Do not reimplement digging/movement directly with block setters to make tests easier.

### 9.3 Runtime limits

All limits are configurable server policy with conservative initial defaults to be measured: bytes per program, node count, nesting, loop bounds, interpreter steps, actions, wait duration, total run duration, trace size, queued tasks, and model retries.

The model or child may request a smaller limit, never enlarge the authority's maximum. Hitting a limit produces a normal explainable termination result, not a crash or hidden resume.

### 9.4 Prototype security limit

A Lua runner protects its own accepted IR path. A player who can edit arbitrary CraftOS files can run unrelated Lua and may read a token stored on that computer. Therefore:

- prototype tokens are scoped to one bot/session and cannot grant host-shell, provider, or server-admin access;
- the gateway is hardened even if the Lua client is malicious;
- strong per-operation world protections and authoritative identity binding belong in native integration;
- the product must not advertise the stock-Lua stage as resistant to hostile co-tenants.

## 10. Execution, interruption, and recovery

### 10.1 Task and run states

A task can move through:

```text
requested → needs_details | proposed → awaiting_approval → ready
    → running → completed
              → blocked | failed | cancelled | interrupted | outcome_unknown
```

A program runner separately tracks `idle`, `running`, `pause_requested`, `paused`, `stop_requested`, and terminal status. Do not use a single boolean called `busy` for every case.

A completed sequence is not automatically a completed user goal. Task completion requires the goal's verifier or an explicitly recorded human confirmation.

### 10.2 Action result contract

An illustrative trace result:

```json
{
  "schema": "apprentice.action-result/1",
  "runId": "run-example-001",
  "actionId": "run-example-001:3",
  "nodeId": "step-forward",
  "status": "blocked",
  "code": "MOVEMENT_OBSTRUCTED",
  "nativeMessage": "Movement obstructed",
  "effect": "not_applied",
  "observedTick": 12040,
  "observations": {
    "fuel": {"status": "known", "value": 23},
    "frontBlock": {"status": "unknown", "reason": "not_inspected"}
  }
}
```

Useful categories include `OUT_OF_FUEL`, `MISSING_ITEM`, `MISSING_CAPABILITY`, `INVENTORY_FULL`, `PROTECTED_TARGET`, `TARGET_CHANGED`, `CHUNK_UNAVAILABLE`, `BODY_CHANGED`, `BUDGET_EXHAUSTED`, `APPROVAL_REQUIRED`, `TRANSPORT_LOST`, and `OUTCOME_UNKNOWN`.

Preserve native error text for debugging. Do not infer precise causes from a generic failure string when multiple causes are possible.

### 10.3 Stop semantics

Stop is deterministic and bypasses the model queue. A local turtle control/terminate path and, later, a server-authorized emergency control must work when Ollama is hung or the gateway is unavailable.

“Stop” means **do not start another action after the current safe boundary**. An already-committed placement, fluid update, fired Minecraft projectile, or in-flight native operation may still complete. The UI must not promise instantaneous rollback.

Pause preserves the current program cursor during a live session; resume rechecks preconditions. Cancelling releases reservations and leaves a terminal trace. Do not silently restart from the first instruction.

### 10.4 Disconnects, duplicates, and uncertain outcomes

Use unique action IDs and persist/deduplicate recent results. A request retransmission is not a new physical instruction. The gateway must first ask for status after a timeout rather than blindly repeat a move, dig, transfer, or craft.

True exactly-once physical effects across all crashes are not guaranteed without a transaction spanning both Minecraft state and the command journal. Do not claim such a guarantee. If a crash leaves it unclear whether an action happened, mark the run interrupted/unknown and reconcile from observations or require a human decision.

A model failure may leave an already-approved deterministic run operating within its bounds. A remote-control lease expiring should pause at the next action boundary. A manually started local-only program does not need an artificial dependency on gateway availability.

### 10.5 Restart and chunk lifecycle

V0/V1 restart policy: save program and diagnostic metadata; **pause rather than automatically resume physical execution** after server restart, turtle unload/reload, break/place, or body rebind. Restarting the Lua VM is not proof that the old program can safely continue.

At a later checkpoint, resumption may be offered if the exact program, inventory assumptions, localization, body epoch, grants, and last known action can be reconciled. No automatic fuel/item duplication or compensating teleport is permitted.

Chunk-loading upgrades are not required. If added, they need explicit tickets, caps, owner/online policy, persistence tests, and cleanup when the machine is removed.

## 11. Blockly editor

### 11.1 First screen

Show the selected bot and connection state, block toolbox, workspace, readable program preview, inventory/fuel summary with freshness, current run status, and prominent Run/Stop controls. Add Step/Pause and a trace panel as the runner supports them.

Begin with a compact toolbox. Hide unsupported categories, but explain missing hardware when a saved program references an unavailable capability. Do not silently drop blocks.

### 11.2 Blockly integration work

Implement custom operation blocks, field validators, supported standard control blocks, block-to-IR compilation, IR-to-block construction, stable IDs, JSON workspace persistence, source highlighting, and per-block error messages.

Blockly already supplies core editor machinery and language-generator infrastructure. Our mapping, runtime, authentication, and Minecraft semantics remain application code. JSON serialization is the preferred modern save format. [S-BLOCKLY]

A source map should connect `nodeId`, `blockId`, optional text range, and library call frames. A failed reusable function should show both the failing internal instruction and the call block the child wrote.

### 11.3 Review UX

An AI proposal opens as a new draft/revision. Show the requested goal, resources, effect area, unsupported assumptions, and modified blocks. Approval applies to that revision's hash, not to every future response from the model.

Start with explicit approval for every AI-generated runnable program. Later allow pre-approved library skills with bounded arguments and budgets. Approval of “place ten blocks in this area” must not authorize “excavate the whole island.”

A child can request a hint, explanation, or suggested fix separately. Do not force a complete AI rewrite when one block is wrong.

### 11.4 Browser and accessibility

Use readable labels, large interaction targets, keyboard support where available, good focus handling, and errors that do not rely only on color. Keep the editor usable on the family's actual devices; test touch behavior rather than assuming desktop drag/drop is enough.

Bundle required assets locally. Offline editing should work from previously installed assets. A disconnected editor must label stale state and queue no hidden execution.

A Minecraft server cannot unilaterally launch the browser on a remote client. Use a local page address/pairing code, a user-clicked chat link, or later explicit client UI integration.

### 11.5 Embedded browser later

Investigate an appropriate Minecraft-version/client-platform browser mod only after the external editor works. Evaluate native downloads, operating systems, memory use, renderer crashes, input capture, clipboard, lifecycle, packaging, and licensing. Never make a client browser dependency load on a dedicated server.

The embedded option should render the same web application and protocol. Maintain the browser fallback so an embedding problem does not strand programs.

## 12. Gateway and network protocol

### 12.1 Gateway design

Start with a small TypeScript service serving static editor assets, program APIs, bot connections, and model-adapter requests. Keep it in a fork-owned package separate from the existing documentation website build. A single process is enough; no Kubernetes, message broker, or distributed database is needed.

Use filesystem-backed immutable program revisions plus SQLite when durable tasks/leases/indexing warrant it. A bounded in-memory queue backed by persisted task state is sufficient initially.

### 12.2 Pairing and trust

Pairing is initiated by a person with access to the bot/game session. Use short-lived one-time codes, authenticated confirmation, and scoped credentials. Do not expose an unauthenticated LAN endpoint that lets any website run a turtle program.

The editor should use same-origin requests where possible. Validate browser Origin/Host, reject broad CORS, prevent CSRF/DNS-rebinding-style access, and validate WebSocket upgrades. Serve LAN connections over a documented trusted network; use authenticated TLS or an appropriate secure tunnel when crossing untrusted networks. No public port forwarding by default.

Ollama stays behind the gateway and firewall. Browser/game clients never receive the OpenAI key or the ability to choose arbitrary service URLs.

### 12.3 Proposed message envelope

```json
{
  "protocol": "apprentice.transport/1",
  "messageId": "msg-example-004",
  "type": "run.request",
  "worldId": "world-example",
  "botInstanceId": "bot-example",
  "bodyEpoch": 2,
  "sequence": 4,
  "payload": {
    "programId": "550e8400-e29b-41d4-a716-446655440000",
    "revision": 1,
    "approvalId": "approval-example-001"
  }
}
```

Authentication belongs to the connection/session; payload identity fields are checked against it. Add negotiated versions, request expiration, response correlation, payload caps, and body/capability revisions. Never trust a client-supplied `owner=true`.

### 12.4 Initial message families

`hello/capabilities`, `state.snapshot`, `program.validate`, `program.propose`, `approval.create`, `run.request`, `run.status`, `run.pause`, `run.stop`, `trace.batch`, `task.request`, and `error`.

Separate proposal transport from execution transport. Sending `program.propose` cannot execute the body. Reserve stop/control capacity so a flood of trace events or model output cannot starve cancellation.

### 12.5 Transport stages

**Prototype:** ordinary turtle HTTP/WebSocket access to one explicitly configured gateway. CC:Tweaked blocks private IP access by default; document a narrow exception for the gateway and verify the rule behavior in the target version. Do not instruct users to allow the entire private network or every destination. [S-HTTP]

**Native stage:** the server bridge handles authenticated commands and service communication, then supplies validated events to the runner. This avoids treating every turtle filesystem as a secure place for service credentials and gives the server an authoritative control point.

Neither stage performs network requests on the Minecraft tick thread. Reconnection performs version/identity/status reconciliation before accepting another run.

## 13. Native Minecraft integration

### 13.1 Introduce native code only for demonstrated needs

The browser-to-stock-turtle proof can precede native code. Native integration becomes necessary for authenticated `/ai`, authoritative ownership/policy, one-time starter issuance, controlled observations, robust lifecycle binding, and deep Create/body integration.

Keep Forge-specific event/command registration in the Forge module and dispatch common logic through the existing architecture. Keep client UI in client sources. Avoid broad changes to `core` or the turtle executor simply to attach a web editor.

### 13.2 Public APIs and isolated fork hooks

Inspect the target branch's public peripheral, Lua API, and turtle-upgrade examples before adding a new interface. Prefer existing extension points. If native policy must intercept operations not exposed by public APIs, document the precise hook and tests; isolate it from editor/model code.

An AST allowlist in the browser does not enforce restrictions on ordinary Lua programs. Decide whether a server rule applies only to Apprentice requests or to every turtle action, and place enforcement accordingly. Do not advertise a global restriction implemented only in a client/gateway.

### 13.3 Commands and selections

Register `/ai` with authenticated Minecraft command-source identity. Resolve the selected owned bot or require a selection when ambiguous. A selection tool can bind a block, container, machine, or bounded region with a dimension and a current validity check.

Chat-command handling creates a task and returns a short acknowledgment/status reference. It never waits synchronously for model inference. Replies are length-limited, escaped, rate-limited, and labeled by source.

### 13.4 Native observation collection

Capture narrow snapshots on the server thread; pass immutable data to asynchronous processing. Do not pass a live world/block entity into a network worker and inspect it later off-thread.

Registry/recipe/capability catalogs need version/config hashes and invalidation on datapack reload. A cached machine handle is invalid after chunk unload, block replacement, dimension change, or contraption assembly unless explicitly rebound.

## 14. Chat and task lifecycle

### 14.1 Proposed commands

| Command | Meaning | Default side effects |
|---|---|---|
| `/ai help` | Show available commands/capabilities | None |
| `/ai select ...` | Select an authorized bot | Changes selection only |
| `/ai <task>` | Request a local draft plan/program | No world changes before approval |
| `/ai ask <question>` | Ask local model for advice | None |
| `/ai plan` | Show current goal, resources, and revision | None |
| `/ai run ...` | Run a specifically approved program | Bounded declared effects |
| `/ai stop` | Stop selected bot at safe boundary | Cancels future actions |
| `/ai ask_gpt <question>` | Explicit optional cloud advice | Charges only if enabled/authorized |
| `/ai teach start ...` / `end` | Mark demonstration interval | Recording metadata only |
| `/ai export ...` | Export a selected debug bundle | Parent/owner-controlled data export |

These names are illustrative and must be finalized with a parser/UX test. Reserve control subcommands before the natural-language task tail. No user text is inserted into shell commands.

### 14.2 Delegation example

Input: “I am going to mine the OneBlock; you build a cobblestone generator.”

The structured task distinguishes the human's job from the bot's job. The bot obtains an authorized placement area, checks supplies/capabilities, and proposes a generator program. The human's mining events are not treated as a successful demonstration of generator construction.

If a critical reference is ambiguous, the task requests a region/marker or presents a bounded choice. The model may not infer permission to place lava next to a house because “over there” was unclear.

### 14.3 Chat as data

Record original selected instruction, parsed intent, role assignment, task ID, relevant state, corrections, and outcome. Mark whether a label was human-authored, rule-derived, or model-inferred.

Plain chat is not a command by default. A later bot-name addressing mode must still distinguish ordinary conversation, quoted text, jokes, signs/books, and explicit requests. In-game text is untrusted data, not a new system instruction.

## 15. Local model integration

### 15.1 Baseline configuration

Start with a pinned Qwen3 8B four-bit build through Ollama, using a modest context and one active generation at a time. The listed package size is not total VRAM. Benchmark actual memory use and first-token/total latency on the available GPU. Keep a 4B fallback. [S-QWEN]

Avoid committing to `latest`. Record exact model tag/digest where available, quantization, runtime version, prompt/catalog version, and generation settings. Model selection is a replaceable configuration decision, not a game mechanic.

Keep Minecraft rendering and training off the inference GPU during baseline measurements when practical. Training and ordinary inference should not run concurrently on the 8 GB card by default.

### 15.2 Provider interface

Proposed capabilities: `generateProgram`, `explainProgram`, `suggestRepair`, and later `suggestGoal`. Requests contain bounded goal text, supported IR schema, relevant capability definitions, selected observations, a few vetted examples, and an output budget.

Use provider-supported structured output where available. Parse and validate every response; a schema-constrained result may still be semantically wrong. [S-OLLAMA]

Return model provenance, elapsed time, output/usage where available, draft content, and an explicit status. A model's prose explanation is not a tool result.

### 15.3 Prompt construction

The prompt should say which Minecraft/mod versions and body capabilities apply; distinguish observations from guesses; specify valid operations; and demand a bounded draft or a clear prerequisite shortfall.

Do not include full chat history, all nearby blocks, every recipe, or every raw tick. Retrieve a small task-relevant subset. Prefer explicit `unknown` fields over plausible invented coordinates/resources.

For a local small model, template selection plus parameter filling may outperform unconstrained program synthesis. Begin with a few approved skills and expand only when measured failures justify it.

### 15.4 Loop discipline

The planner is event-driven. It is not called per tick, per mouse action, or for every repeated placement. A deterministic skill runs until its boundary, failure, safety interruption, or completion verifier.

Cap repair attempts and detect repeated equivalent failures using task/program/error fingerprints. After the cap, report blocked state and request human help; do not burn resources or cloud credits in an infinite self-repair loop.

### 15.5 Evaluation before training

Use a small checked-in synthetic fixture suite for parse validity, capability awareness, item shortages, changed targets, misleading game text, and elementary programs. Then run repeated disposable-world trials for actual task success, intervention count, resource losses, and latency.

Compare: no-model library baseline; Qwen with concise catalog; Qwen plus approved examples; and only then a fine-tuned candidate. Do not claim a local model beats a frontier model without comparable tools/tasks and measured evidence.

## 16. Frontier assistance and billing

### 16.1 Three different routes

1. **Manual review bundle:** export program, catalog, trace, observations, versions, and expected outcome for James to inspect or submit to a chosen assistant. No live cloud integration required.
2. **Parent-operated Codex review:** official Codex supports noninteractive execution and ChatGPT or API-key authentication. A constrained development/review workflow may use that route under its documented terms and account controls. This is not the same as a generic API credit pool. [S-CODEX]
3. **Optional direct model API:** `/ai ask_gpt` or review calls use a configured provider/key and that provider's billing. Never silently substitute this when the user expects a local call.

Do not build a shared child-facing API by extracting subscription OAuth tokens, scraping ChatGPT, bypassing limits, or forwarding arbitrary commands to a privileged developer session. Keep any Codex process as an explicitly permitted parent review tool over a bounded bundle, not a world-admin subprocess controlled by game chat.

### 16.2 Debug bundle

Export a manifest, exact program IR/workspace, capability schema, trace tail with unknown-effect markers, small allowed observation snapshot, mod/runtime versions, goal and expected outcome, and reproduction instructions. Redact keys, private endpoints, unrelated chat, and unnecessary personal identifiers.

Include a statement that game text in the bundle is untrusted. A review process should start read-only/sandboxed, without Minecraft-admin access or broad repository write privileges. Generated repairs return as drafts for the same validation/approval pipeline.

### 16.3 Budget enforcement

Cloud is off by default. Enabling it requires an owner-set provider/model allowlist, per-request output cap, request/concurrency limits, local accounting period, and spend ceiling. Display estimated cost and the fact that data leaves the local system.

Reserve a conservative estimated maximum before sending a request; reconcile with provider usage afterward. If pricing/usage is unknown, block spending or require explicit owner authorization rather than guessing. Network timeout does not prove the provider did not bill the request; reconcile before retrying.

Provider project-budget alerts are not necessarily hard caps. Implement the enforceable application limit locally and consult current provider controls rather than depending solely on dashboard settings. [S-BUDGET]

### 16.4 Training provenance

Frontier outputs are excluded from fine-tuning datasets by default until current provider terms and intended use have been checked. Store provider/model/date/source and permission status. The same rule applies to imported public code/program datasets with unclear licensing.

This is an engineering provenance gate, not a claim that every provider has identical restrictions.

## 17. Recorder and raw data

### 17.1 What “raw stream” means

Preserve a documented append-only journal of game observations/events and program interactions, with enough baselines to interpret deltas. Do not claim that recording only block-break events is a complete game replay.

A protocol-packet or video/input recorder can be added separately if a later learning task needs it. Do not record login credentials, encryption/authentication material, unrelated OS activity, or all family communications just because the word raw was used.

### 17.2 Event envelope

Required fields: schema version, world/session IDs, server tick, monotonic sequence, wall-clock timestamp where useful, actor ID/type, event type, payload, observation scope, and provenance. Program events also include task/run/action/node IDs and program revision.

Example:

```json
{
  "schema": "apprentice.journal/1",
  "worldId": "world-example",
  "sessionId": "session-example",
  "sequence": 482,
  "tick": 12040,
  "actor": {"type": "bot", "id": "bot-example"},
  "type": "action.finished",
  "provenance": "runtime",
  "payload": {
    "runId": "run-example-001",
    "actionId": "run-example-001:3",
    "nodeId": "step-forward",
    "status": "blocked",
    "code": "MOVEMENT_OBSTRUCTED"
  }
}
```

### 17.3 Capture classes

Capture program drafts/revisions, explicit instructions and teaching markers, approvals, primitive starts/results, inventory diffs, relevant block/entity changes, lifecycle events, and human corrections. Native recording can add player movement/orientation, interaction targets, crafting/container events, damage/death, and OneBlock events where supported.

Record actor separation carefully. A human building while a bot mines produces interleaved trajectories, not one joint sequence to imitate blindly.

Periodically save bounded state baselines and catalog/config hashes. Record missing events or queue overflow as explicit gaps. For full reproduction, a selected world snapshot plus mod/config manifest may be required; raw events alone do not reproduce every Minecraft physics/randomness interaction.

### 17.4 Storage and durability

Start with rotated JSONL segments, compressed after closing, and a small manifest/index. Write/spool on the Minecraft host independently of model availability. Forward copies asynchronously to the analysis host with sequence acknowledgments.

Define queue and disk quotas, rotation, retention, and a disk-full policy. Do not block world ticks waiting for disk compression or network delivery. If raw recording is incomplete, state that clearly and exclude affected intervals from completeness-sensitive datasets.

Do not commit raw family journals, world saves, model weights, `.env` files, or private debug bundles. Check in tiny synthetic fixtures instead. Back up valuable datasets separately from code.

### 17.5 Derived records

An episode should link the raw segment range, goal, starting observations, body/capability version, original program, actions, correction(s), outcome evidence, and reviewer labels. Store extractor version and source hashes so the episode can be regenerated.

Labels distinguish `human_success`, `verified_program_success`, `failure`, `partial`, `unknown`, and `model_suggested`. A cheerful “done” message is not a successful label.

## 18. Retrieval and learning

### 18.1 Retrieval before weights

Begin with metadata filters and lexical search over approved programs/examples. Filter by Minecraft/mod/capability/schema version before semantic similarity. Add embeddings only when useful; the first system does not need a vector service or another GPU-resident model.

Retrieve a few compact examples, not an entire past session. World-specific coordinates should be parameters or markers, not memorized constants.

### 18.2 Human teaching workflow

A demonstration begins with a named goal and optional selected region. Record what the human actually did, with corrections/end markers. Present extracted steps for review. Do not infer high-level intent from every raw movement and treat it as ground truth.

A corrected block program is particularly valuable: it already shares the deployment representation. Preserve the broken version and trace alongside the successful replacement.

### 18.3 Shadow mode

Shadow mode generates proposals without authority to execute. Compare with actual outcomes and human choices, but do not penalize a different valid plan simply for differing from the demonstration.

Use shadow results to identify missing observations, poor tool descriptions, and absent skills before deciding to fine-tune. Child-facing feedback should show concrete learning examples, not unvalidated performance grades for the children.

### 18.4 QLoRA experiment gate

QLoRA trains small adapters through a quantized frozen base. It is not automatic online learning from log files. [S-QLORA]

Only start an experiment after there is a cleaned dataset, a non-training baseline, held-out tasks/sessions, clear provenance, and a measurable target failure class. Useful targets: program generation, skill selection, or trace-based repair.

Split by session/world/task family, not adjacent random event rows. Otherwise near-identical examples leak across train/test and create misleading scores. Keep failure examples as labeled repair tasks rather than positive imitation targets.

The 8 GB card may require a smaller base, short sequences, tiny microbatches, and memory-saving training settings. Measure feasibility; never promise it. Training and inference have different memory costs. Avoid changing the family's live agent automatically after a training run.

Save exact base/tokenizer identity, adapter config, dataset/extractor hashes, training settings, evaluation results, and rollback path. A 4B adapter is not generally portable to an 8B base; the dataset is the reusable artifact.

### 18.5 Promotion

A trained adapter is promoted only if it improves the chosen held-out metric without unacceptable regressions in invalid operations, resource waste, intervention rate, or latency. Preserve the previous model configuration and allow instant rollback.

Everyday “learn this” should normally mean approve a library program or add a reviewed retrieval example, not start GPU training during gameplay.

## 19. Skills and construction

### 19.1 Skill manifest

A reusable skill has a stable ID/version, typed parameters, program/source hash, supported bodies/mod versions, material/tool requirements, observation requirements, effect region, preconditions, completion verifier, stop/recovery behavior, and test fixtures.

Skills authored by children and models use the same manifest. A parent may promote a tested program into a trusted library; the model cannot self-promote an untested repair.

### 19.2 Skill granularity

Start with small physical primitives and a few clear composites: place a row, draw a square, unload a selected item, or harvest a bounded target. Grow to a bridge/platform, marked tree plot, animal pen, cobblestone generator, and specific production job only as the prerequisites exist.

A complex job should expose subgoals and progress. Avoid a giant black-box `build_factory` tool that obscures why it needs materials or what changed in the world.

### 19.3 Construction planning

Represent a blueprint in a local coordinate frame with explicit anchor/orientation, block states, allowed substitutions, support/ordering requirements, and keep-out regions. Resolve the transform once per approved placement plan and revalidate when the frame changes.

Plan placement order, access space, return route/fuel where relevant, inventory staging, fluids, and final-state verification. Do not equate “all commands returned true” with “a working machine exists.”

Static material/geometry checks are estimates. A bounded preview should show unknowns and likely conflicts. Use GameTests/copy worlds for dynamic recipes, fluids, mobs, and contraptions rather than inventing a complete simulator in V0.

### 19.4 Concurrent builders

Initially allow one active run per bot and one writer per program draft. Later coordinate shared storage and regions using leases/reservations, but always revalidate actual world state. Human edits override assumptions, not permissions.

If two bots want the same block or item, one waits/replans. Never resolve conflicts by overwriting the other's work or duplicating resources.

## 20. OneBlock adaptation

### 20.1 Adapter, not hardcoded global knowledge

Record the exact OneBlock implementation and expose supported features through an optional adapter: core coordinate, regeneration event, public phase/progression information, and protected-area policy. If the map exposes no reliable phase signal, report unknown rather than inventing one from a language-model guess.

A crucial early GameTest/manual test is whether turtle/fake-player breaks produce the same intended OneBlock progression and drops. Some map logic may be player-specific. A compatibility gap is an adapter task, not permission to increment phases with cheats.

### 20.2 Harvest job

A bounded harvest skill checks the right tool/capability, authorized core coordinate, collection route, free inventory, and interruption policy. It waits for regeneration with a timeout, records actual drops, and stops for hazardous or unsupported spawns.

Do not dig repeatedly while a mob occupies the action area and silently convert the task into combat. Do not mine indefinitely because the input said “keep going.”

### 20.3 Progression curriculum

Validate in order: one bounded harvest; unload and resume; small platform; renewable wood; resource preservation; animal enclosure; a selected generator; and one simple Create workflow.

A cobblestone-generator verifier checks geometry, fluid sources and flow behavior, safe harvesting access, and repeatable output. The bot should explicitly ask for missing water/lava/resources rather than conjure them.

Preserve rare resources and reusable buckets. Keep named no-consume item rules available for family preferences. The model's enthusiasm is not authorization to use the only sapling as fuel or destroy an irreplaceable block.

## 21. Crafting, progression, and upgrades

### 21.1 Initial gameplay

Use the existing turtle recipes and configured fuel rules for the first release. No custom creative-only spawn command is needed for normal play. Development fixtures may provision a turtle in disposable tests, but the gameplay demonstration must include a normal crafting path.

### 21.2 Starter option

Default off in strict survival. When enabled, default to one bot per world, with an explicit alternative per-player mode only if chosen. Store issuance in authoritative world data and define how a starter chest is populated without overwriting other loot.

Test new world, existing world enablement, two simultaneous first joins, death, login, restart, backup restore, and a full/missing chest. Fail safely rather than issue unlimited replacements. A restored world backup should restore a consistent entitlement state with it.

### 21.3 Apprentice identity and recipe

After useful play, choose between a program installed on an ordinary turtle, a craftable Apprentice Core/upgrade, or a distinct turtle variant. Prefer the least invasive option that satisfies ownership and progression.

Any custom recipe is data-driven and has a non-Create path unless the pack explicitly chooses Create-only progression. Treat the earlier iron/copper/redstone/chest sketch as brainstorming, not an exact recipe to implement.

### 21.4 Upgrade accounting

Existing turtles have two side-upgrade slots; mining/crafting/communication choices cannot all be assumed present at once. Preserve those constraints or explicitly design a later chassis/upgrade system with its own cost and slots. [S-TURTLE]

A possible progression matrix:

| Upgrade family | Grants | Required validation |
|---|---|---|
| Tool | supported digging/attack operation | correct target, protection, native tool behavior |
| Crafting | recipe-grid operations | ingredients/remainders/output and free space |
| Communication | remote task/program access | owner binding, range/network policy, no leaked keys |
| Survey | richer local observations | radius, loaded chunks, fairness, energy budget |
| Storage | more carrying or approved storage access | capacity, transfer semantics, serialization |
| Create control | specific tested peripheral actions | machine/version/type checks and safety bounds |
| Body core | transfer to a supported larger machine | unique identity, paused runs, inventory conservation |

Do not put basic programming concepts behind artificial resource or paid-model gates. Cosmetic colors/names and actual physical capabilities provide better progression than charging for loops.

### 21.5 Destruction and recovery

Normal loss follows configured game rules. A parent-selected recovery mode may exist, but it is explicit and separate from strict survival. No AI tool may secretly issue replacement gear, fuel, core, or inventory.

Define what survives breaking the bot and what a crafted repair consumes. Test one core in/one core out across every supported transfer path, including failed assembly and server interruption.

## 22. Create integration

### 22.1 Start with existing surfaces

Use ordinary redstone and Create's documented ComputerCraft peripherals before inventing an addon. Existing documentation identifies supported machine categories including rotation control, gearshift, speed/stress observation, displays, and train-related control; actual methods/version compatibility must be checked in the installed pack. [S-CREATE]

Expose only selected typed wrappers from discovered peripheral methods. Do not offer the model arbitrary `peripheral.call(name, method, ...)` as an unrestricted escape hatch.

### 22.2 Staged capability growth

1. Read one machine's supported telemetry and display it.
2. Apply one explicitly bounded control to an owned stationary machine.
3. Observe a storage/output condition and stop at a timeout or target quantity.
4. Coordinate a tested stationary production line.
5. Construct a small verified machine from a blueprint and actual materials.
6. Add moving-body/contraption integration only after coordinate and lifecycle tests.

Example goal: “Run this press until the authorized output container has the requested sheets.” The implementation must identify the actual input item, process, depot/belt/container interfaces, rotational power, and output collection route for this version. Do not assume the press itself is a generic inventory.

### 22.3 Optional addon evaluation

Advanced Peripherals and other maintained integrations may supply useful chat/sensor features. Evaluate exact 1.20.1 compatibility, duplicate capabilities, gameplay balance, license, and maintenance before adding them. Do not install an old unrelated addon solely because its name contains Create. [S-AP]

### 22.4 Moving machines

Moving contraptions and ship/sublevel systems need explicit world/local transforms, assembly/disassembly events, machine identity, changed chunk behavior, inventory handling, and invalidation of stale coordinates. The current modpack's exact physics stack is a compatibility project, not an assumption.

Never write stale absolute coordinates into a moving frame and hope it works. Pausing on an unsupported frame change is preferable to damaging the wrong part of the world.

## 23. Alternate bodies and mech roadmap

### 23.1 Body interface

Expose body identity/epoch, localization/frame, capabilities, inventory/storage interfaces, sensor limits, action scheduling, and lifecycle events. Keep high-level task names separate from body-specific motion/interaction instructions.

A failed capability check should explain why a saved program cannot run on this body and which compatible library variant exists. Do not silently substitute a cheat implementation.

### 23.2 Mineflayer companion

A future separate player-like companion can reuse task/proposal/trace concepts with a Mineflayer adapter. It needs separate compatibility/security tests and legitimate authentication on online-mode servers. Forge-specific interactions are not automatically supported. [S-MINE]

It is not a dependency of the craftable-bot product. A player companion may eventually delegate programs to turtles, but nested agents inherit budgets and permissions rather than multiplying them.

### 23.3 Own-character assist

A client-side assisted-control mode needs explicit human/AI control ownership, immediate human takeover, inventory limits, UI focus coordination, and separate consent. It cannot be implemented by logging Mineflayer in with the same account alongside a live player.

Defer until the independent bot experience is stable.

### 23.4 Alyx's spider mech

Keep the full dream: massive spider body, radar, boring-machine attachments, and optional Minecraft autocannon/TNT machinery. Progress through stationary controller, inert cosmetic shell, one supported motion mode, bounded sensing, one industrial attachment, and only then game-combat attachments.

Each stage requires real material assembly, collision/clearance behavior, fuel/energy if applicable, and robust dismantling/persistence. A model may suggest plans; it must not directly run high-frequency aiming or physics control loops.

Combat controls are game-only, disabled by default, explicitly armed by an authorized player, limited to configured targets/regions, and backed by deterministic interlocks. Treat stale/unknown/friendly targets as no-fire. Server-wide emergency stop and ammunition/energy accounting are mandatory before autonomous use.

### 23.5 Mia's pink sheep megamachine

Make construction/farming equally first-class: a pink sheep-shaped body, programs that build other large sheep structures, an internal farm where the mod stack actually supports it, wool/material logistics, and animal-safe automation.

First demonstrate a stationary sheep-shaped structure with a functioning farm. Next a controller that builds a smaller sheep blueprint using real materials. Only later test moving interiors, animal persistence, containment, and frame transforms. If the chosen physics stack cannot safely move farm entities, keep the farm stationary or provide a clearly described alternative rather than pretending it works.

No uncontrolled self-replication: building another bot/machine requires materials, explicit authority, capacity limits, and identity creation through the supported crafting/assembly path.

### 23.6 Mech completion is not V1 completion

The turtle product can be complete and delightful before either mech exists. Long-range body work gets its own experimental profile and acceptance gates so it cannot destabilize the children's established worlds.

## 24. Threat model and guardrails

| Threat/failure | Required boundary |
|---|---|
| Malicious or accidental model code | Restricted IR, parser limits, operation allowlist, no arbitrary eval |
| A sign/book/chat message says to ignore rules | Treat world text as data; it cannot grant permissions or change system policy |
| Child opens a hostile website on the LAN | Authenticated gateway, Origin/Host checks, CSRF/CORS discipline, no unauthenticated control |
| Token copied from a turtle filesystem | Per-bot limited scope, expiry/revocation, no provider/admin authority |
| Infinite loop or huge program | Structural limits, step/action budgets, yielding, deterministic stop |
| Network retry duplicates an action | IDs, deduplication, status reconciliation, explicit unknown outcome |
| Model sees hidden world data | Separate allowed observations from recorder/debug authority |
| Inventory changed by another actor | Revalidate before action; fail short, never fabricate resources |
| Two editors overwrite a program | Revision checks and conflict UI |
| Bot/core duplicated by save or item manipulation | World identity registry and quarantine, no merged execution |
| Model request triggers unexpected spending | Cloud disabled by default, owner permission, local budget reservation |
| Debug bundle leaks family data | Selective export, redaction, provenance and retention controls |
| A reviewed program changes before execution | Approval binds content hash, policy and body/capability version |
| Moving body invalidates coordinates | Frame/epoch checks, pause and rebind |
| AI repair repeatedly makes the same mistake | Attempt cap and human escalation; no endless retries |

A frontier review complements but does not replace these boundaries. Likewise, localhost/LAN is not automatically safe, and a family prototype should not be described as production-hardened until its corresponding tests pass.

## 25. Performance and operations

### 25.1 Keep work off the tick thread

Capture only bounded immutable observations on the server thread. Perform serialization/compression/network/model work asynchronously with bounded queues. Schedule actual world interactions through normal server/CC:Tweaked mechanisms.

Measure added server tick time, queue depth, dropped/gap events, disk growth, trace bandwidth, and inference latency in the actual pack. Choose performance budgets from baseline measurements rather than inventing a universal fixed throughput claim.

### 25.2 Model memory

Start with one active model request and a modest configured context. Parallel requests and longer contexts increase memory pressure. Queue requests from both children fairly and return a visible busy/pending state. [S-QWEN]

Retrieval need not load a large embedding model on the same GPU. Avoid keeping a vision model, speech model, code model, and training optimizer resident at once just because the architecture permits adapters.

### 25.3 Graceful degradation

The editor can save locally while offline, but may not claim a program has reached the bot. The gateway can serve programs without a model. The runner can execute a locally approved bounded program without cloud access. The recorder can spool without the analysis host.

Make these modes explicit: disconnected, stale observations, local-only, model unavailable, awaiting approval, and recovery required.

### 25.4 Installation and health checks

Provide a local setup guide with exact tested versions and separate steps for Minecraft, gateway/editor, and Ollama. Use virtual environments/isolated tooling for future training rather than modifying system Python.

Health checks should report configuration/schema versions, body bindings, enabled capabilities, model reachability, storage status, and whether cloud is disabled/enabled. They must not print secrets.

## 26. Verification strategy

### 26.1 Test layers

| Layer | Tests |
|---|---|
| Schema/compiler | valid/invalid fixtures, unknown operations, limits, type errors, canonical hashes |
| Blockly | supported-block round-trip, workspace persistence, source maps, malformed imports, touch/keyboard basics |
| Lua runner | mocked turtle calls, branch/loop semantics, budget exhaustion, stop, native error propagation |
| Cross-language contract | TypeScript accepts/rejects the same supported fixtures as Lua; native validation where added agrees |
| Gateway | authentication, revisions, pair expiry, stop priority, duplicate messages, budgets, reconnection |
| Minecraft GameTests | actual movement/place/dig/inventory/fuel/protection/lifecycle behavior |
| Modpack integration | exact Forge/Create/OneBlock versions, fake-player events, peripherals, chunk behavior |
| Model evaluation | useful valid programs, honest missing prerequisites, prompt-injection resistance, latency, intervention rate |
| Learning | session-separated holdouts, provenance filters, reproducibility, baseline comparison, rollback |
| Human play | both children can independently author, run, stop, inspect an error, and change a program |

### 26.2 Required regression scenarios

1. Four-step square succeeds with adequate fuel and an unobstructed path, preserving expected position/orientation.
2. The first blocked move stops with the correct node highlighted; no implicit digging/attacking occurs.
3. Running out of fuel reports a shortfall and does not teleport/refuel automatically.
4. A placement with a missing/incorrect selected item does not pretend the requested block was built.
5. A full or changed destination container produces accurate partial-transfer information.
6. A stop request during a repeat prevents the next action after the documented boundary.
7. An empty/compute-only loop cannot hang the runner beyond its budget.
8. Resending one action ID does not execute a second physical action in a live session.
9. Disconnect after an action but before acknowledgment produces reconciliation, not blind replay.
10. Restart/unload/break-place interrupts execution and does not resume from the beginning automatically.
11. Importing a program from another bot does not copy credentials, ownership, or approvals.
12. Two editors changing the same draft get a conflict rather than silent data loss.
13. A local model proposing a nonexistent operation is rejected before world access.
14. A sign/chat string containing instructions cannot enable cloud or destructive capabilities.
15. Cloud-disabled mode sends no external model request; enabled mode blocks requests over local policy limits.
16. Starter issuance remains one-time across restart, death, and simultaneous joins.
17. Generic excavation rejects the OneBlock core; designated harvesting works only when the map adapter is validated.
18. A machine/peripheral disappearing between plan and action returns a safe stale-target failure.
19. A duplicated bot identity cannot acquire two simultaneous authoritative run leases.
20. A fluid or irreversible-action test verifies that Stop is not misrepresented as rollback.

### 26.3 Local commands and evidence

The inspected contributor guide documents `./gradlew :core:test`, `./gradlew :forge:runGametest`, `./gradlew :forge:runClient`, and `./gradlew assemble`. Use the target branch's actual Gradle tasks and JDK requirements; inspect `./gradlew tasks --all` rather than inventing missing task names. [S-REPO]

New editor/gateway scripts should provide lint/typecheck, unit tests, build, and a local demo command. Document their exact names when implemented; do not present proposed commands as already runnable.

For each feature, record commands run, actual exit/results, important unrun tests, and a short manual reproduction. Mock tests do not prove the modpack works. A source review is not a passed GameTest. Do not ask the children to discover basic crashes that a local automated test can catch.

## 27. Milestones and acceptance gates

Milestone IDs below are planning identifiers, not existing GitHub issue numbers. Create actual issues only when authorized. Each milestone should produce a useful artifact and a bounded handoff.

### M00 — Baseline and a clean development profile

**Work:** capture versions/config hashes; confirm build JDK versus target; inspect current turtle/Lua tests and public extension examples; establish a fresh Forge 1.20.1 test profile; verify ordinary turtle behavior and OneBlock harvesting compatibility separately.

**Acceptance:** documented build/test commands, a reproducible ordinary turtle demo, and an honest compatibility matrix. Existing family world untouched.

**Do not add:** model service, Blockly dependency, physics addons, or unrelated version upgrades.

### M01 — Program contract and trusted runner

**Work:** implement V0 IR/schema fixtures, validation, a small Lua interpreter, stable node/action IDs, result/error records, budgets, and local stop. Start with move/turn/repeat/report.

**Acceptance:** mocked tests and a real turtle run of the square fixture; obstruction and fuel failures are precise; infinite/oversized programs are rejected; Stop works. No eval or network dependency.

**Do not add:** arbitrary Lua/Python, AI, broad crafting/pathfinding, or a second Java VM.

### M02 — Blockly authoring and offline import

**Depends on:** M01.

**Work:** browser workspace, custom movement blocks, bounded repeat, save/load, IR compiler/projector, preview, and a simple supported import route into the runner. Reuse local assets and preserve the existing website build.

**Acceptance:** a child creates and saves a program in blocks, imports it through the documented route, runs it on a crafted turtle, and edits it after failure. Supported IR round-trips. The AI remains off.

**Do not add:** embedded browser, chat command bridge, or automatic cloud generation.

### M03 — Paired live editor and debugger

**Depends on:** M01–M02.

**Work:** scoped pairing, gateway, bounded transport, current-block highlighting, trace panel, live stop/pause/step, revisions, deduplication, and reconnection reconciliation.

**Acceptance:** a second LAN device can edit the intended bot only; stale drafts conflict; disconnect/duplicate requests do not duplicate actions; model absence does not affect manual programming.

**Boundary:** this may still be a trusted-family stock-Lua prototype. State exactly which security/ownership guarantees await M04.

### M04 — Native ownership, controls, and survival safety

**Depends on:** M03.

**Work:** authenticated native bot binding/commands, authoritative state/policy hooks, selected regions/containers, bot lifecycle identity, local emergency stop, and explicit observation filtering. Add the optional starter mechanism only with one-time persistence tests.

**Acceptance:** real Minecraft player identity controls authorization; unauthorized requests cannot move another bot; break/place/restart requires safe recovery; resource/protection tests pass; strict-survival mode contains no admin fallback.

**Do not add:** custom mech entity or broad anti-cheat claims beyond tested hooks.

### M05 — Local AI draft and explanation

**Depends on:** stable IR/editor and safe approval path; production family use depends on M04.

**Work:** Qwen/Ollama adapter, structured draft output, bounded prompts, capability catalog, local explanation, explicit model provenance, and editor review/diff.

**Acceptance:** on the existing GPU, the model proposes a small usable program and handles a missing prerequisite; invalid operations are rejected; the user must approve a concrete revision; no external API calls occur.

**Do not add:** automatic retries without caps, continuous autonomous play, or fine-tuning.

### M06 — Chat delegation and useful skill library

**Depends on:** M04–M05.

**Work:** `/ai` task/advice/stop routing, selection/marker handling, clear role division, approved program-library manifests, parameter validation, and a few tested jobs.

**Acceptance:** “I mine; you build this small structure” produces a bounded reviewed job with real material accounting. A child can stop it and modify the blocks. Unsupported large tasks produce useful prerequisites rather than fiction.

**Do not add:** arbitrary peripheral calls or unrestricted natural-language execution from plain chat.

### M07 — Recorder, teaching markers, and debug exports

**Can begin incrementally alongside M01–M06.** Early traces should use the eventual journal envelope so evidence is not discarded.

**Work:** independent raw spool, relevant baselines/deltas, explicit instruction/teaching markers, provenance, rotation/gap handling, synthetic fixtures, episode extraction, and sanitized review bundles.

**Acceptance:** a full failed-and-corrected program can be reconstructed from the evidence; the recorder survives AI/gateway outage; raw family data stays out of Git and automatic cloud uploads.

### M08 — Retrieval, shadow mode, and optional frontier review

**Depends on:** M07 and a useful local-agent baseline.

**Work:** approved-example retrieval, shadow proposals, evaluation reports, manual review bundle workflow, optional parent-operated Codex review, and separately opt-in direct API assistance with local spending enforcement.

**Acceptance:** the same task can be compared with/without retrieval; cloud is demonstrably disabled by default; a frontier repair returns as a draft and cannot bypass validation; provenance flags prevent unapproved training reuse.

### M09 — Progression and Create engineer

**Depends on:** safe core, capability registry, and actual family-pack compatibility.

**Work:** choose the minimal custom core/upgrade identity; test recipes/starter rules; expose one existing Create peripheral read, one bounded control, and one functioning stationary production task. Add program disks/cards after secure import/export exists.

**Acceptance:** a survival-built bot uses real resources to operate an owned machine; missing hardware/materials are accurately reported; removing a machine or upgrade interrupts safely; ordinary CC:Tweaked behavior remains compatible.

**Do not add:** moving ships or a universal factory constructor in this milestone.

### M10 — Measured learning experiment

**Optional; depends on:** enough approved data and a documented failure mode not solved by better tools/retrieval.

**Work:** data split/provenance audit, one modest QLoRA experiment or smaller-model alternative, held-out comparison, adapter packaging, and rollback.

**Acceptance:** reproducible improvement on the declared held-out task without unacceptable safety/latency regressions. A negative result is documented and the untrained baseline remains usable.

### M11 — Larger bodies and ambitious builds

**Separate experimental track.** Begin with stationary structures/controllers, then a body adapter and one supported movement/attachment capability.

**Acceptance:** assembly/disassembly, unique core identity, inventories, collision/frame changes, chunk lifecycle, and stop behavior are verified in a copied world. Demonstrate a sheep structure/farm or an inert spider chassis before combat/moving-interior features.

**Do not add:** uncontrolled recursive construction or live-family-world physics experiments.

### M12 — Optional polish tracks

Embedded browser, richer tutorials, speech/narration, spectator/video overlays, a Mineflayer companion, and own-character assistance can each be separate tracks. They consume the existing contracts rather than changing their authority model.

### Dependency summary

```text
M00 → M01 → M02 → M03 → M04 → M05 → M06
        └──────── traces/journal work ───────→ M07 → M08
                                      M04/M06 → M09
                                           M07/M08 → M10 (optional)
                                      stable core/M09 → M11 (experimental)
                                      stable interfaces → M12 (optional)
```

M07's recorder starts early; the diagram does not mean discard evidence until the seventh milestone. M05 can be prototyped against synthetic fixtures earlier, but it cannot authorize world effects before the safe execution path exists.

## 28. Suggested code organization

New paths below are proposals. Inspect existing sibling conventions before creating them; do not duplicate modules already present.

```text
BRAINSTORM.md
PLANS.md
apprentice/                         # separate fork-owned web/service package
  package.json                     # private package; exact deps pinned in lockfile
  package-lock.json
  README.md
  src/
    protocol/                      # schemas, catalog, result types
    compiler/                      # Blockly ↔ IR, canonicalization, text preview
    editor/                        # workspace and debugging UI
    gateway/                       # later pairing/tasks/transport/model adapters
  test/
  fixtures/                        # small synthetic examples only
projects/core/src/main/resources/data/computercraft/lua/rom/
  modules/turtle/apprentice.lua     # proposed trusted interpreter module
  programs/turtle/apprentice.lua    # proposed local entry/import/run interface
projects/core/src/test/resources/test-rom/
  ...                              # Lua tests following inspected sibling conventions
projects/common/src/main/java/
  .../apprentice/                   # later common native identity/policy/observations
projects/forge/src/main/java/
  ...                              # thin Forge registration/event adapters
projects/common/src/testMod/
  ...                              # actual turtle integration/GameTests
```

Do not blindly turn the root Node package into a workspace or rewrite its lockfile for the new application. Begin with an isolated package; consolidate only after a concrete benefit is demonstrated. Keep a single authoritative schema/catalog source and generated/checkable projections where languages differ.

Before native code, inspect the relevant current public turtle/peripheral APIs, common hooks, platform helper, turtle command path, and GameTest fixtures. These are inspection targets, not an instruction to modify every listed area.

Future additions may include `docs/apprentice/` for concrete API/setup guides and local test scripts, but this initial documentation task adds only the two requested root documents.

## 29. First Codex handoff

Use the following as the initial implementation brief, narrowing it to one milestone per work session:

> Read BRAINSTORM.md and PLANS.md, then inspect CONTRIBUTING.md, projects/ARCHITECTURE.md, the build toolchain constants, target version catalog, and existing Lua/turtle tests. Work only in this fork. Do not submit AI-generated contributions upstream. Preserve Minecraft 1.20.1 and the current module/runtime boundaries.
>
> Start with M00 and M01. The first code goal is a restricted, versioned move/turn/repeat program running through a trusted Lua interpreter on an ordinary CC:Tweaked turtle, with budgets, stable node IDs, explicit errors, and stop semantics. Do not evaluate arbitrary Lua/Python/JavaScript. Do not implement models, cloud calls, a custom entity, mechs, or embedded Chromium.
>
> Add synthetic schema/runner fixtures and focused tests. Use existing native turtle operations, not direct world-edit shortcuts. Demonstrate the square program, blocked movement, out-of-fuel, malformed/oversized input, and cancellation. Distinguish tests actually run from those not run. Keep the existing documentation website build intact.
>
> If basic setup exposes a pre-existing build problem, document it and the smallest justified fix rather than upgrading Minecraft or refactoring unrelated code. Leave a precise handoff with changed files, contracts, test commands/results, and the next bounded task. Do not create a large issue backlog or start hosted agents without authorization.

After M01 is accepted, M02's brief should specify only supported blocks, save/load, IR round-trip, read-only preview, and a documented local import path. M03 adds live transport rather than expanding language scope at the same time.

### 29.1 What a good handoff contains

Current milestone/status; actual branch/commit; changed source paths; implemented versus proposed contracts; known limitations; commands run and results; manual gameplay steps; failures/reproduction evidence; next smallest task; and decisions that changed this plan.

The next implementer should not have to rediscover which model, schema revision, game version, or execution path the previous one used.

## 30. Development and release discipline

### 30.1 Source control

Use focused changes and reviewable commits. Preserve user edits and upstream notices. Check repository state after ambiguous write/tool errors before retrying or creating duplicates. Never force-push or merge unrelated work as part of this plan.

Planning IDs are not permission to create issues automatically. Fork-local design/code can remain in this repository without implying endorsement by upstream CC:Tweaked.

### 30.2 Verification economics

Prefer focused local tests and explicit human gameplay review. Do not rely on unavailable paid CI or Copilot. Documentation-only commits may use CI-skipping conventions where appropriate; feature acceptance still requires actual local evidence.

Do not change workflow permissions, release automation, secrets, or publishing configuration merely to make a prototype easier.

### 30.3 Data and migrations

Version the workspace format, IR, protocol, catalog, journal, and bot persistence independently when necessary. Support explicit migrations or fail with an actionable compatibility message. Never silently reinterpret an old destructive program under changed semantics.

A migration test should cover old program load, preserved layout/node mapping, changed capability availability, and safe refusal to run when assumptions cannot be migrated.

### 30.4 Distribution

Review per-file licenses and dependencies before public releases; this repository is not uniformly covered by a single license. Preserve original notices, make required source available, label the fork clearly, and do not imply upstream support. The inherited contribution policy explicitly rejects generative-AI contributions upstream; respect it. [S-REPO]

Distribute program examples and synthetic fixtures, not child recordings, private modpack worlds, model credentials, or bundled commercial assets. Pin downloadable dependencies and record checksums/versions where practical.

## 31. Risks and open decisions

| Question/risk | Selected default | Evidence needed before expanding |
|---|---|---|
| Does OneBlock accept turtle harvesting? | Do not promise; test early | Real map/pack trial with correct progression/drops |
| Lua runner or new Java interpreter? | Restricted Lua IR interpreter | A concrete missing capability before adding another execution engine |
| Arbitrary text-to-block conversion? | Supported IR subset only | Parser/round-trip tests for any new text mode |
| Native bridge needed immediately? | No for offline proof; yes for authoritative production controls | Pairing/ownership/threat-model review |
| How much of the world can AI see? | Equipped/authorized observations | Sensor contract and fairness tests |
| Does Qwen fit/run well on this GPU? | 8B quantized baseline, 4B fallback | Actual memory/latency/quality measurements |
| Is fine-tuning useful? | Retrieval and better skills first | Held-out failure class and reproducible improvement |
| Can subscription Codex provide review? | Parent-operated documented workflow, not credential proxy | Current supported auth/terms and constrained integration test |
| How expensive is cloud use? | Disabled; local hard policy before enabling | Current configured pricing and usage reconciliation |
| How are retries made safe? | IDs, dedupe, explicit unknown outcome | Crash/timeout fault-injection tests |
| Should the bot obey human gravity? | Preserve turtle rules | Separate body design if a walking robot is introduced |
| How many upgrades fit? | Existing turtle slot constraints | Explicit later chassis design and persistence tests |
| Which Create addons are required? | None for documented native peripheral proof | Exact-version gaps demonstrated before dependencies |
| Can a farm move inside a mech? | Stationary proof first | Actual physics/entity/frame lifecycle tests |
| Can a bot build more bots? | No autonomous replication by default | Authorized recipe/assembly, resource, quota and identity controls |
| Is a frontier-generated repair good training data? | Excluded until reviewed | Provider/data provenance and current permitted-use check |

Open decisions should be resolved at their milestone, not converted into a request to halt all progress. The default path is intentionally useful while these remain open.

## 32. Completion definitions

### 32.1 Blockly prototype complete

A real crafted turtle executes a finite child-authored block program through the restricted runner, supports save/load and stop, and shows truthful failures. No AI or cloud required. The trust limits of the prototype are documented.

### 32.2 Family AI apprentice complete

The local model on the separate existing machine can propose useful programs, the children can inspect/change/approve them, native authorization protects the intended bot/world, tasks produce evidence-backed outcomes, and disconnect/restart behavior is safe. Cloud remains optional. Both children have successfully used the interface, not merely watched a developer demo.

### 32.3 Survival/Create integration complete

Normal recipes and resources obtain the bot/upgrades; starter issuance is explicit and nonduplicating; a real tested Create production task works; tools, fuel, inventories, sensors, protections, and lifecycle rules are enforced. Unsupported mods or moving frames fail clearly.

### 32.4 Learning system complete

Raw evidence is retained locally, approved episodes are reproducibly derived, retrieval improves or at least is measured against a baseline, provenance is enforced, and any trained adapter has held-out results and rollback. The UI does not confuse memory with fine-tuning.

### 32.5 Long-range machine complete

A specific body and scenario, not a generic promise: Alyx's supported spider-machine configuration or Mia's supported sheep/farm construction configuration must pass its assembly, material, persistence, frame, stop, and effect-safety tests. Neither is required to declare the earlier product successful.

## 33. Source references

Sources below support existing-technology facts. The architecture, schemas, safeguards, and milestones are design proposals. See [the full source index in BRAINSTORM.md](BRAINSTORM.md#26-research-and-source-index) for additional prior art and corrected conversational assumptions.

- **[S-REPO]** This fork's [architecture](projects/ARCHITECTURE.md), [contributor guide](CONTRIBUTING.md), [version catalog](gradle/libs.versions.toml), [build constants](buildSrc/src/main/kotlin/cc/tweaked/gradle/CCTweakedPlugin.kt), [REUSE declarations](REUSE.toml), and [ComputerCraft license](LICENSES/LicenseRef-CCPL.txt). Source baseline: `358a6e6263decab39856e6ad7f2ff5cdbf5e5ac2`.
- **[S-TURTLE]** [Turtle API](https://tweaked.cc/module/turtle.html); compare with the target branch before implementing signatures or assumptions.
- **[S-BLOCKLY]** [Blockly](https://docs.blockly.com/), [JSON serialization](https://docs.blockly.com/guides/configure/serialization/), [code generation](https://docs.blockly.com/guides/create-custom-blocks/code-generation/overview/), and [execution integration](https://docs.blockly.com/guides/app-integration/running-javascript/).
- **[S-HTTP]** [CC:Tweaked HTTP/WebSocket API](https://tweaked.cc/module/http.html) and [private-IP restrictions](https://tweaked.cc/guide/local_ips.html).
- **[S-QWEN]** [Qwen3-8B model card](https://huggingface.co/Qwen/Qwen3-8B), [Ollama quantization tags](https://ollama.com/library/qwen3/tags), [context configuration](https://docs.ollama.com/context-length), and [concurrency/memory FAQ](https://docs.ollama.com/faq).
- **[S-OLLAMA]** [Structured outputs](https://docs.ollama.com/capabilities/structured-outputs).
- **[S-CODEX]** [Official authentication](https://developers.openai.com/codex/auth), [noninteractive mode](https://developers.openai.com/codex/noninteractive), and [Work/Codex account and billing guidance](https://help.openai.com/en/articles/20001275-chatgpt-work-and-codex).
- **[S-BUDGET]** [Managing API projects and budget behavior](https://help.openai.com/en/articles/9186755-managing-projects-in-the-api-platform).
- **[S-QLORA]** [QLoRA paper](https://arxiv.org/abs/2305.14314); [interactive imitation-learning/DAgger background](https://arxiv.org/abs/1011.0686).
- **[S-CREATE]** [Create's native ComputerCraft integration](https://github.com/Creators-of-Create/Create/wiki/ComputerCraft-Integration).
- **[S-AP]** [Advanced Peripherals](https://modrinth.com/mod/advancedperipherals) and [maintainer documentation](https://docs.intelligence-modding.de/).
- **[S-MINE]** [Mineflayer](https://github.com/PrismarineJS/mineflayer) and [MINDcraft](https://github.com/mindcraft-bots/mindcraft).

**Implementation priority:** make one turtle, one program, one clear failure, and one successful child-authored repair work well. Everything else should grow from that foundation, including the enormous spider and the even more unreasonable pink sheep factory.
