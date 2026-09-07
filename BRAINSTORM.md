<!--
SPDX-FileCopyrightText: 2026 JamesXNelson
SPDX-License-Identifier: MPL-2.0
-->

# Craftable AI Apprentices — BRAINSTORM

**Planning session:** September 6, 2026, America/Regina.  
**Repository:** `JamesXNelson/CC-Tweaked`, initially inspected at `358a6e6263decab39856e6ad7f2ff5cdbf5e5ac2` on `mc-1.20.x`.  
**Status:** Discussion archive and design space, not a claim that any new feature is implemented.  
**Companion document:** [PLANS.md](PLANS.md) is the implementation authority when an exploratory idea here conflicts with a selected approach there.

This document preserves the Minecraft/AI/coding-project ideas discussed by James and the assistant, including the changes of direction, attractive alternatives, caveats, and long-range dreams. It deliberately does not include unrelated family conversations or shopping details. Labels distinguish **James's requirements**, **assistant proposals**, **research findings**, and **open possibilities**. New engineering conclusions from inspecting this fork are identified rather than retroactively attributed to the conversation.

The working name **Apprentice** is a placeholder. **Cobbler** was suggested as a friendly individual bot name, not a decided product name.

## Contents

1. [The motivation](#1-the-motivation)
2. [The original AI-OneBlock idea](#2-the-original-ai-oneblock-idea)
3. [Prior art and the change in direction](#3-prior-art-and-the-change-in-direction)
4. [The central shared-program idea](#4-the-central-shared-program-idea)
5. [A real survival-game object](#5-a-real-survival-game-object)
6. [The children's eventual machines](#6-the-childrens-eventual-machines)
7. [Blockly, text, and the coding experience](#7-blockly-text-and-the-coding-experience)
8. [The bot's body](#8-the-bots-body)
9. [What the LLM should and should not do](#9-what-the-llm-should-and-should-not-do)
10. [Local models and hardware](#10-local-models-and-hardware)
11. [Chat as interaction and teaching](#11-chat-as-interaction-and-teaching)
12. [Recording play without throwing information away](#12-recording-play-without-throwing-information-away)
13. [Learning in stages](#13-learning-in-stages)
14. [The code-review and repair loop](#14-the-code-review-and-repair-loop)
15. [Remote deployment](#15-remote-deployment)
16. [OneBlock capabilities](#16-oneblock-capabilities)
17. [Create and modded automation](#17-create-and-modded-automation)
18. [Progression and upgrades](#18-progression-and-upgrades)
19. [Teaching, collaboration, and personality](#19-teaching-collaboration-and-personality)
20. [Recording entertaining videos](#20-recording-entertaining-videos)
21. [Safety, fairness, and trust](#21-safety-fairness-and-trust)
22. [Development workflow](#22-development-workflow)
23. [Corrections and unresolved assumptions](#23-corrections-and-unresolved-assumptions)
24. [A map from ideas to implementation](#24-a-map-from-ideas-to-implementation)
25. [Questions to resolve by experiments](#25-questions-to-resolve-by-experiments)
26. [Research and source index](#26-research-and-source-index)

## 1. The motivation

**James's starting point:** Alyx really enjoys videos of AI playing Minecraft OneBlock. Many of the videos appear to be made by people who understand Minecraft much better than they understand LLM orchestration. James wants to explore a more competent, inspectable approach, not just reproduce an entertainingly confused chatbot pressing buttons.

Both Alyx and Mia have used block coding at Code Ninjas and are already reasonably capable with it. They miss Minecraft Education's programmable Agent and its block/Python learning experience, but they increasingly want Java Edition, Create, and Java mods. The desired project combines those interests instead of requiring a choice between them. Microsoft's Agent/MakeCode material is inspiration for the interaction pattern, not an implementation to copy wholesale. [EDU]

The project grew from **watching AI play** into **building and teaching a craftable programmable companion**. It should still be fun without an LLM, without a cloud subscription, and before the long-term learning system exists.

The strongest short-term outcome James identified was simple: **even getting Blockly working with CC:Tweaked would be amazing**.

## 2. The original AI-OneBlock idea

The first concept was a Java Minecraft player controlled through Mineflayer. OneBlock seemed a useful restricted environment: mine the regenerating block, collect materials, expand the island, craft, protect useful animals and saplings, survive phase changes, and build renewable production.

The assistant proposed avoiding a stream of screenshots or individual keypress decisions. Instead, normal code would execute reliable skills while an LLM selected goals and jobs. Example skill ideas included:

- `mine_oneblock(count)`, `craft(item, count)`, `make_pickaxe(material)`;
- `expand_platform(...)`, `build_pen(...)`, `store_excess_items()`;
- `build_tree_farm()`, `build_cobblestone_generator()`, `fight_nearest_hostile()`;
- collecting drops, monitoring inventory capacity, replacing tools, and recovering from interruptions.

These names were **illustrative capabilities**, not existing Mineflayer or CC:Tweaked functions. Some are substantial programs, and some require equipment or integrations not present on an ordinary turtle.

A compact state report would contain health where relevant, fuel, inventory, tools/upgrades, nearby observations, known island layout, OneBlock information where legitimately available, the current goal, the last action, and new events. The model would choose the next bounded job rather than decide at Minecraft tick frequency.

Potential replanning events included task completion, a full inventory, an unavailable resource, a new phase, a mob appearing, a tool/capability becoming unavailable, a changed target, and repeated lack of progress. Immediate stop and hazard behavior belong in deterministic code; waiting for model inference is not a safety mechanism.

Three separate memory categories were proposed: **world facts**, **current/long-term goals**, and **lessons from experience**. These should not become an ever-growing chat transcript. Facts need timestamps, provenance, and invalidation when the world changes.

## 3. Prior art and the change in direction

### Mineflayer and MINDcraft

The likely YouTuber name identified during the conversation was **Emergent Garden**, and the central software prior art was **MINDcraft**. The exact video James saw was not supplied, so the attribution should remain a likely identification rather than a verified claim about every creator's credits. The primary repository is `mindcraft-bots/mindcraft`; it combines LLMs and Mineflayer and documents a local Ollama backend. [MIND]

Mineflayer is a programmable Java Edition client: software logs into a server as a player and exposes world, inventory, movement, and interaction APIs to JavaScript. It is useful prior art for a separate AI player. It is not an embedded LLM and is not, by itself, a Java mod that transparently understands every Forge/Create integration. [MINE]

The initial suggestion became **adapt MINDcraft rather than build an AI player from nothing**. After the coding-robot requirement emerged, the preferred first implementation changed again: **build on CC:Tweaked's craftable turtles instead of making Mineflayer the central dependency**.

The MINDcraft line remains valuable for planner organization, model adapters, demonstrations, and a future player-like companion. It should not pull arbitrary generated JavaScript execution into the turtle product.

### CC:Tweaked and Blockly

CC:Tweaked already provides programmable computers and mobile turtles. Its documented turtle API includes movement, inspection, item handling, digging, placing, and upgrade-dependent operations. [CCT] [TURTLE]

Blockly provides the mature visual editor. It supports custom blocks and JSON workspace serialization, and already has a Lua generator. It is an editor/code-generation library, not our runtime, authorization layer, Minecraft integration, or AI system. [BLOCKLY] [BLOCKLY-SAVE] [BLOCKLY-GEN]

This is the practical combination: reuse CC:Tweaked's body and Lua environment; reuse Blockly's editor; build the child-friendly program model, debugging experience, permissions, and optional AI around them.

## 4. The central shared-program idea

**James's proposal:** both the children and the model can generate imperative instructions, which James or a frontier model can review to fix bugs.

**Selected design direction:** the child, local model, optional frontier model, and eventually a text editor all author the same program representation.

```text
Child edits blocks ───────┐
                         │
Local model drafts ──────┼──> Versioned program IR ──> validation/review ──> execution
                         │             │
Human/frontier repair ───┘             └──> blocks, readable text, traces, examples
```

IR means **intermediate representation**: a small structured program, not free-form code with host-machine privileges. The earlier term AST, or abstract syntax tree, described the tree-shaped part of this representation. The product also needs identifiers, versions, capability requirements, and source mapping.

A good loop is:

```text
request → proposed program → inspect blocks → approve a version → execute
   ↑                                                   ↓
   └──────────── correction and evidence ← trace/outcome
```

A program is more valuable than an opaque action because the kids can read it, change it, reuse it, and understand a repair. A library function should be able to start as a child's own working block program and later become a named reusable skill.

The AI is not the owner of the world. It is one possible program author.

## 5. A real survival-game object

**James's requirement:** the bot should be craftable. The optional starter chest could contain one free bot. Otherwise the player obtains it through legitimate progression.

Alyx cares about not using cheats in a survival world. James contrasted that with Dadland's occasional replacement of legitimately obtained enchanted netherite gear after void accidents. The useful design distinction is **an explicit family/server recovery rule versus hidden bot privileges**. A recovery feature must never quietly become normal execution behavior.

The appeal is being able to beat the game using a robot that was crafted and operated under the world's rules. That does not necessarily mean a turtle must obey identical physics to a human player; it means the mod's documented rules are visible, consistently enforced, and paid for with real resources.

The core bot should use real inventory, actual placement/breaking operations, loaded chunks, owned equipment, and configured fuel. It must report missing materials instead of creating them. `/ai build an iron farm` should produce a proposal and a material shortfall, not invoke `/fill`.

The first playable milestone should use the **existing turtle recipe and existing items**, avoiding a custom entity and progression rebalance before the programming experience works. A distinctive Apprentice Core, shell, program card, and revised recipe are later possibilities.

The starter option needs an explicit policy: one per world by default, or one per player only when selected. It must not issue another free bot on every login, death, or restart.

## 6. The children's eventual machines

**James's long-range image for Alyx:** a massive spider mech with radar, autocannons, boring machines, and TNT launchers.

**James's long-range image for Mia:** an equally ambitious machine that builds giant pink sheep mechs with pink sheep farms inside them.

These are Minecraft game features and creative ambitions, not real-world robotics or weapons designs. Both are important product inspiration. Mia's build-oriented automation should not be treated as merely a pink cosmetic variant of Alyx's combat progression.

The architectural insight is that **the brain, the programs, and the body should be separable**. A trusted controller/program library could eventually operate a turtle, stationary factory computer, construction machine, or supported large vehicle. However, the body must advertise real capabilities: a floating one-block turtle, walking mech, train, and contraption do not share movement semantics.

Possible progression:

```text
small programmable turtle
    → better tools and useful programs
    → factory supervisor / construction assistant
    → a physically assembled large machine
    → coordinated specialist machines
```

A mech cannot be implemented merely by scaling a turtle model. Multiblock assembly, collision, inventories, moving coordinate frames, attachments, persistence, and chunk behavior are separate engineering problems. Keep this ambition alive without making it a dependency of Blockly.

Minecraft combat machinery needs an additional server-authorized capability layer: explicit arming, limited targeting, protected players/pets/structures, real ammunition and energy where required, and deterministic shutdown. World repair is not a substitute for those controls. Sheep-farm interiors similarly need animal containment, space, valid habitats, and tested moving-entity behavior rather than magically teleporting livestock into a model.

## 7. Blockly, text, and the coding experience

Three UI routes were discussed:

| Route | Attraction | Main cost |
|---|---|---|
| A local/LAN browser editor | Mature Blockly, easy debugging, usable on a second screen | Pairing, transport, and switching between browser and game |
| An embedded browser in Minecraft | Education-like integrated feel | Browser runtime, platform support, input/focus, native dependencies, packaging |
| A native Java block-editor recreation | No embedded browser dependency | Rebuilding drag/drop, layout, connections, undo, accessibility, and procedures |

The recommendation remains **browser first**. A program link or pairing code on the turtle's screen can lead to the editor. The server does not automatically have the right to launch a browser on a remote player's machine; a clickable link or client integration requires the user's action.

The original suggestion was not to execute general Python inside the server. That remains sensible. Python-like readable examples were proposed as a teaching surface, not a promise of Python compatibility. Arbitrary Python or JavaScript would bring execution and sandboxing problems unrelated to learning Minecraft programming.

A correction discovered during planning is that CC:Tweaked already has a Lua environment: we do not need a second general language VM in Java. A small trusted Lua interpreter can execute our restricted IR using turtle APIs. Blockly's existing Lua generator is also useful for an initial experiment or a read-only generated-code view. The full product still needs its own supported-subset mapping and cannot automatically round-trip arbitrary Lua/Python into blocks.

Early blocks should be small and understandable: movement, turning, inspecting, digging, selecting an item slot, placing, counting inventory, saying something, repeating, conditions, and stopping. Variables and functions follow. Arbitrary recursion, unbounded parallelism, shell execution, and generic HTTP blocks are not early features.

Desired editor controls: **Run, Pause, Step, Stop, Save, Open, Explain, Suggest Fix, Review**. Highlight the actual executing block and the actual failing block. Preserve the child's workspace when an AI suggests a replacement. Show a diff and offer a named revision, not a silent rewrite.

Program Cards or existing ComputerCraft disks could become physical ways to exchange a named program, such as `Safe Bridge v3`. Program portability must not copy owner credentials, bot identities, or cloud keys.

## 8. The bot's body

### Craftable native turtle: the preferred starting point

The bot exists independently of the child, has its own inventory, accepts delegated tasks, and does not need an additional authenticated Minecraft player account. It can become familiar and personal without being omnipotent.

CC:Tweaked is not a promise that every native interaction automatically works with every mod. Normal block interactions, recipes, peripherals, fake-player protections, and loader/version compatibility must be tested.

### Mineflayer player: still attractive

A separate player-like AI could follow, trade, converse, share expeditions, and participate in ordinary survival. The bot would need the appropriate authenticated account on an online-mode server; do not weaken server authentication to avoid that requirement. MINDcraft documents the separate-account consideration. [MIND]

This route remains optional, especially for videos or a companion that uses player mechanics. It should share task/program concepts where useful, not dictate the initial implementation.

### Automating the child's own character

James also proposed letting the AI automate the child's own user. The assistant noted ownership conflicts: human and AI moving at once, inventory being consumed unexpectedly, or one closing a screen the other opened.

A future client-side assist/takeover mode would need explicit start/stop, immediate input takeover, inventory/resource boundaries, and control ownership. Mineflayer cannot simply log in as the same player alongside an existing session to accomplish this. Prefer the separate craftable bot first.

### A bot that programs another bot

A playful later possibility is an AI player writing programs for coding robots: one companion helps the child while delegating harvesting or factory jobs to other craftable machines. This is a future orchestration feature, not permission for nested unrestricted agents.

## 9. What the LLM should and should not do

The model should interpret an instruction, suggest a goal, choose a known skill, explain a program, or propose a repair. Deterministic code should validate inventory, consult recipes, resolve geometry, enforce permissions, execute primitives, and check results.

Earlier illustrative questions included whether to keep mining, secure a cow, create renewable wood, or replace a missing tool. High-level decisions can be useful model work; repetitive movement and block placement should not incur another inference for every action.

The assistant proposed a hierarchy: a local inexpensive model for most requests, with optional frontier assistance when stuck. Specific percentages of local/cloud usage were aspirations, not measured forecasts.

A verifier can often be ordinary code. If a proposed craft needs three ingots and only two are present, return an exact resource shortfall. Do not ask another model to invent the inventory arithmetic.

High-level skills must expand into inspectable, bounded programs or explicitly registered native operations. A name such as `build_tree_farm` is not an excuse to hide arbitrary generated host code.

The planning vocabulary must reflect the currently installed robot and peripherals. A language model should not see `radar.scan` until radar exists, is equipped, authorized, and implemented.

## 10. Local models and hardware

**James's constraints:** an existing GPU with 8 GB VRAM; the model will run on a different machine from Minecraft; a GPU upgrade is not a prerequisite.

The proposed baseline is a modest quantized Qwen model served locally, initially through Ollama. A pinned `qwen3:8b-q4_K_M` package is listed at about 5.2 GB, but that is model-package size, not total runtime VRAM. Context/KV cache, execution buffers, other applications, and concurrency also use memory. [QWEN] [OLLAMA-MODEL] [OLLAMA-MEM]

A 4B model is a fallback for more headroom. Benchmark the actual GPU rather than promising tokens per second or declaring the named model permanently optimal. Model choice should be configurable, with exact model identifiers, quantization, prompt version, and runtime version recorded for reproducibility.

Kimi was discussed because James is open to non-OpenAI local models. That is welcome; provider neutrality is a requirement. The earlier discussion misstated Kimi K2's size. Moonshot's own card specifies **one trillion total parameters and 32 billion activated parameters**. Active parameters are not the same as all weights that must be stored. Full Kimi is not an 8 GB deployment target. The project does not depend on claims about later Kimi releases. [KIMI]

Other candidates mentioned included gpt-oss-20b and larger Qwen models after a hardware upgrade. They are evaluation candidates, not commitments. A 16 GB NVIDIA card and an AMD alternative were discussed, but earlier Canadian retail price ranges are not a durable planning input. Do not buy hardware before the existing card proves the workflow. No training-speed or inference-cost claim from the conversation should become an acceptance criterion.

Local inference is not literally free: it uses electricity, hardware, storage, and maintenance. Its attraction here is controllable cost, privacy, offline operation, and freedom to experiment.

## 11. Chat as interaction and teaching

James proposed explicit collaborative instructions, for example:

> I am going to mine the OneBlock; you build a cobblestone generator.

This communicates role division, not just a sequence of observed actions. Preserve the original instruction as well as any structured interpretation. A model should not be trained to believe that the human's mining behavior demonstrates how the bot should build a generator.

Proposed command family:

```text
/ai <task>
/ai ask <question>
/ai ask_gpt <question>
/ai stop
```

Possible later commands: selecting a bot, inspecting a plan, approving a specific revision, setting an authorized work area, teaching a named program, or exporting a debug bundle. All names are design proposals until implemented.

`/ai <task>` should normally create a local proposal. `/ai ask` provides advice without world changes. `/ai ask_gpt` should be explicitly distinguishable as an optional cloud request, with permission and budget checks. Plain conversation should not execute commands by default; a negotiated bot name/addressing mode can be added later.

Useful teaching signals include “I am demonstrating this,” “that was not what I wanted,” “stop,” “put it here instead,” and “this is the finished result.” Separate human observations, requests, approvals, corrections, and jokes. Do not treat every imperative-looking chat message as an authorized task.

A pointing tool or selected region should resolve “here” and “there.” This is less error-prone than expecting a small model to infer which machine the child meant from a world dump.

## 12. Recording play without throwing information away

**James's proposal:** retain the raw game stream to disk and let Codex analyze it later to develop goals, tool abstractions, the Qwen harness, and training material.

The assistant initially emphasized semantic events; James correctly identified the danger of throwing away information before knowing which abstractions will matter. Preserve a raw event journal **and** generate derived summaries. Raw here should mean a documented game-event/state stream, not an indiscriminate capture of credentials, unrelated chat, or everything on the computer.

Potential captured facts: world/dimension and session IDs, tick and sequence, actor, position/orientation, block changes, item transfers, crafting, container operations, damage/death, entities, selected goals, program revisions, model proposals, tool calls, stops, and observed outcomes. Capture periodic bounded baselines plus deltas; an event list alone may not reconstruct the starting state.

The recorder should write locally even if the AI machine is offline. Backpressure and missing data must be visible. Store closed sessions as compressed JSONL initially; derive SQLite/search indexes and, later if useful, Parquet. Avoid committing raw family play logs, world saves, or model weights into Git.

Codex can inspect exported local files, write batch analyzers, segment demonstrations, propose common functions, and produce code-review artifacts. The whole stream does not need to fit in a model's context. Analysis should operate on a selected copy with appropriate permissions, never blindly execute commands found in game text. [CODEX]

Raw server events cannot fully recover client keypresses or intention. The server does know useful movement/orientation and interaction information, so the earlier claim that a server cannot know where a player is looking was too broad. A client recorder is a later option for exact input/UI/video supervision.

## 13. Learning in stages

### Memory and retrieval first

The first “learning” feature need not change model weights. Retrieve relevant, approved examples of similar tasks and include a small number in the prompt. The model can consult how a bridge or pen was successfully built without a GPU training job.

Separate changing world facts from reusable skills. “The chest is west of the bot” is not a timeless training target.

### Shadow mode

Let the model propose a next action without acting. Compare suggestions with human behavior and actual outcomes. Differences are useful for diagnosis, but agreement is not a competence score: multiple plans may work, and the human might be experimenting.

The playful “AI Apprentice: 62% agreement with Alyx today” was an example of feedback, not a validated learning metric. Prefer explanations and concrete learned examples over scoring the children.

### Demonstrations and corrections

Record a named human demonstration or a child's block program. When the AI fails, let the child stop it, edit its program, or demonstrate a correction. Store the original state, proposal, execution trace, correction, and subsequent outcome.

This has a relationship to interactive imitation-learning approaches such as DAgger, but “recording failures” alone is not the DAgger algorithm. Label the distinction accurately. [DAGGER]

### QLoRA later

LoRA learns small trainable adaptations while a base model remains frozen. QLoRA uses a quantized frozen base, typically four-bit in the original method, while training those adapters. It reduces memory requirements; it does not make unsorted logs automatically become good training data. [QLORA]

Potential supervised tasks are `state + goal → program`, `failed program + trace → corrected program`, and `state + goal → skill choice`. Train only on appropriately reviewed examples. Fine-tuning an 8B model on 8 GB is an experiment with strict memory constraints, not a promise. Start smaller if necessary.

The **dataset** can be reused when moving from a 4B to an 8B model; a LoRA adapter cannot generally be moved between different base architectures/shapes. Keep base-model and tokenizer identity with every adapter.

Human demonstrations and permitted local outputs are the cleanest initial data source. Frontier-generated repairs need provider provenance and a rights/terms check before being used to train a local model. Default them to excluded from training until that review occurs. Do not assume all purchased API output is automatically eligible for distillation.

## 14. The code-review and repair loop

This is the project's strongest combined educational and AI feature:

```text
intent + actual capabilities + program
                 ↓
          execution evidence
                 ↓
       failure classification
                 ↓
  human/local/frontier proposed repair
                 ↓
       reviewed new program revision
                 ↓
           re-test and verify
```

Desired failure information: operation, stable block/node ID, bot identity, instruction arguments, a small relevant observation snapshot, native error text, normalized error category, resource changes, and whether an action may already have happened.

An assistant can explain a failure; it must not invent success. A frontier model's review is advice, not a security proof. The runtime remains authoritative.

Earlier examples described a player walking off an island. For a CC:Tweaked turtle, examples must use its real behavior instead: obstruction, running out of fuel, wrong item selection, changed block, no inventory space, an unavailable upgrade, or an unloaded destination. Optional floor/build-area safeguards should be identified as our rules, not existing turtle gravity.

A useful debug export includes the precise program version, supported instruction definitions, trace tail, observations and unknowns, Minecraft/mod versions, expected outcome, actual outcome, and reproduction steps. That is much better review context than a gigantic unstructured world dump.

## 15. Remote deployment

The intended arrangement separates Minecraft from inference:

```text
Minecraft host                         AI/service host
-------------                          ---------------
Forge 1.20.1 world                     authenticated local gateway
CC:Tweaked + runner    <── LAN ──>     Blockly editor and project store
optional native bridge                local Qwen/Ollama
raw event journal                     retrieval and offline analysis
                                      optional cloud adapter
```

The browser editor may run on the child's gaming computer or another LAN device. Ollama should not be exposed directly to every game client. The gateway enforces roles, request size, rate limits, and cloud policy.

A WebSocket was suggested for bidirectional state, commands, and status. CC:Tweaked already supports HTTP/WebSocket APIs, but private-network destinations are blocked by default. A development route must narrowly allow the configured gateway rather than disable all private-IP protections. [CCT-HTTP] [CCT-LAN]

A production native bridge can avoid requiring ordinary turtle scripts to hold host-service secrets. Network code must not block Minecraft's tick thread. The physical bot should continue a previously approved bounded program during ordinary model latency; losing the gateway must not start a new task, replay a destructive command, or leave an armed machine operating indefinitely.

## 16. OneBlock capabilities

The OneBlock world gives an initial curriculum, not a universal fixed API. The actual map/datapack/mod version must be inventoried. Phase progression, generated blocks, special drops, and mob spawning may differ.

Useful jobs discussed include mining in batches, collecting drops, retaining rare resources, expanding a platform, crafting tools, establishing renewable wood and food, storing materials, securing animals, building a cobblestone generator, managing hostile interruptions, and improving the island aesthetically.

A staged curriculum could be:

```text
move/turn → place a row → inspect/dig → harvest a bounded OneBlock batch
    → unload a chest → bridge/platform → tree plot → animal pen
    → cobblestone generator → a working production line
```

Each advanced job needs explicit prerequisites and a completion check. A generator needs correct fluid behavior and valid water/lava supplies; a pen is not complete merely because a program placed some fences; a tree farm needs a renewable cycle, not only a planted sapling.

The central regenerating block needs special protection: general excavation/building cannot overwrite it, while a designated harvesting job may break it as the map intends. Test whether the chosen OneBlock implementation counts fake-player/turtle breaks correctly before promising progression.

## 17. Create and modded automation

The early discussion imagined direct machine inspection, inserting materials, setting a rotation speed, waiting for output, controlling redstone links, connecting shafts, and reusable tree/iron/cobblestone factory programs.

These are goals, not a list of APIs already present. The model must use actual item IDs, block states, recipe types, and machine capabilities from the installed pack. A press does not receive finished sheets as its ingredient just because an earlier illustrative snippet used that name.

A significant research correction: **Create already documents native ComputerCraft integration**, including supported peripheral categories such as speed/stress measurements, rotation control, sequenced gearshift, display link, and train station. An addon is not required for those documented integrations. Verify methods against the installed version. [CREATE-CC]

Advanced Peripherals is also relevant prior art for chat, sensors, and other computer extensions; evaluate its actual 1.20.1 compatibility and gameplay balance before adding a dependency. The earlier-mentioned Create: Computing is an older 1.18.2 addon, not the preferred starting point here. [AP] [OLD-COMPUTING]

The progression should begin with redstone and supported peripherals, then stationary automation, then construction assistants, then moving contraptions/ships. A change of coordinate system on an assembled machine is not just a different block position. No promise of Valkyrien/Clockwork/Sable or cannon integration should be made until exact versions are tested.

## 18. Progression and upgrades

The conversation proposed a craftable basic robot and later upgrades for tools, carrying, sensors, fuel/energy, communications, Create interaction, and eventually larger bodies.

An illustrative iron/copper/redstone/chest recipe was floated. It was not balanced or validated and is not an implementation requirement. Start with CC:Tweaked's real recipes and choose custom progression only after gameplay.

Possible paths:

| Path | Examples |
|---|---|
| Builder | placement assistance, surveys, blueprint checks, authorized material access |
| Miner | valid digging tools, tunnel programs, boring-machine control |
| Farmer | planting/harvesting, safe animal handling, crop and wool production |
| Engineer | redstone, machine telemetry, verified Create peripherals |
| Companion | chat, local planning, named memory, help/explanation |
| Large machine | docking/core transfer, multiblock controls, supported moving bodies |

An upgrade should grant a real supported capability, not make the prompt claim the bot has hardware. Existing turtles have limited upgrade slots; installing tools, a crafting table, or communication hardware creates real tradeoffs. Do not assume all upgrades fit simultaneously.

There was also an idea of limiting program slots or memory as late-game progression. Treat that carefully: artificial limits that obstruct learning or force a paid cloud model would be counterproductive. Cosmetic shells, program organization, and physical abilities are better levers than paywalling basic loops or explanations.

## 19. Teaching, collaboration, and personality

The desired progression is **blocks → readable programs → functions → debugging → AI-assisted programming → Java modding**. The robot is a tool for learning, not merely a task-completion service.

The AI could have selectable presentation styles: cautious survivalist, bold explorer, obsessive engineer, comically overambitious builder, or a companion who treats Alyx/Mia as the people giving it advice. A deliberately silly voice must not change safety policy or pretend the robot has real emotions.

Narration should be separate from control and grounded in events. One example joke was about having nineteen logs and the architectural ambitions of an empire. Such narration can be templated; it need not incur an inference every few seconds.

Potential interactions: explain only the next problem, show a hint before a full solution, highlight which blocks changed in a repair, let the child predict the result, and celebrate a reusable function they authored. Do not silently replace the child's work with an opaque perfect program.

Multiple children need separate workspaces, explicit sharing, ownership/roles, and arbitration when they send conflicting commands to one bot. A child should be able to say “stop” without racing a remote model.

## 20. Recording entertaining videos

The original AI-player idea included a normal Minecraft client as spectator/camera and OBS for capture, while Mineflayer ran headlessly. A craftable turtle can also be filmed from an ordinary client; it does not need its own renderer service.

Potential later features: task labels, a visible current program, amusing but truthful narration, failure/repair highlights, before/after builds, and time-indexed recordings linked to the event journal. Do not require video/audio capture for the core product. Child voice, account details, and family chat should not automatically be uploaded or included in public demos.

## 21. Safety, fairness, and trust

Safety here includes **protecting the survival world**, **protecting the host computers and API account**, and **making the behavior understandable to children**.

Programs need bounded execution, cancellation, explicit effects, valid capabilities, inventory accounting, and protected-area rules. AI-generated code must never receive arbitrary OS shell, filesystem, Java reflection, or unconstrained network access.

The important distinction is that a restricted Apprentice program protects this execution path; it does not magically prevent someone from writing an unrelated unrestricted CraftOS program. Strong server-wide rules must be enforced at the server operation boundary, not only in our UI.

The model-visible world should be limited to equipped sensors and authorized observations. Full debug/recording access is not permission to give the bot omniscient ore/entity information.

Cloud access is opt-in. API keys stay outside Minecraft saves and program cards. Any app-enforced spending cap must live in the gateway; provider dashboard budgets must not be assumed to be hard cutoffs. [BUDGET]

## 22. Development workflow

James wants the heavy design work documented now so Codex can implement later. The fork is the working repository. Documents should give executable slices, source locations, test requirements, and decisions, not merely a motivational roadmap.

The first deliverable of this session is documentation only. No game-code implementation, model training, world alteration, or hardware purchase is implied.

The source inspection found:

- `mc-1.20.x` targets Minecraft 1.20.1; the build catalog currently lists Forge 47.1.0. The family's existing Forge installation must be tested rather than casually replaced. [REPO-VERSIONS]
- The fork's build uses JDK 25 tooling but targets Java 17. Do not confuse the development JDK with Minecraft's target or import Java 8 restrictions from an unrelated project. [REPO-JDK]
- Core/Lua, common Minecraft code, and Forge/Fabric adapters already have deliberate module boundaries. [REPO-ARCH]
- Upstream's current contribution instructions explicitly reject AI-generated contributions. This is a fork-local project; do not silently submit generated code upstream or misrepresent its provenance. [REPO-CONTRIB]
- Licensing is per-file and mixed, including MPL-2.0 and original ComputerCraft-licensed material. Preserve notices and review redistribution obligations instead of calling the whole fork one permissive license. [REPO-LICENSE]

Prefer small, focused local verification and human gameplay testing. Do not rely on paid hosted CI/Copilot as the proof that a feature works. Creating these planning files is not authorization to open dozens of issues or launch coding agents.

## 23. Corrections and unresolved assumptions

This archive preserves the ideas but does not canonize conversational mistakes.

| Earlier idea or statement | Corrected planning position |
|---|---|
| All GPT integration necessarily requires direct API billing | Direct API calls are separately billed. Codex also documents subscription authentication and noninteractive execution. A parent-operated review job is a separate supported workflow to investigate, not permission to repurpose credentials as an unrestricted shared chat API. [CODEX-AUTH] [CODEX-EXEC] |
| A dashboard monthly budget is a hard cap | Enforce an application-side cap; current project-budget documentation describes soft alerts. [BUDGET] |
| Kimi K2 had 671B parameters | Moonshot's model card says 1T total / 32B active. Do not preserve the erroneous memory arithmetic. [KIMI] |
| Local 8B QLoRA will definitely fit in 8 GB | Feasibility depends on exact hardware, sequence length, implementation, and optimizer. It is an experiment after retrieval and evaluation. |
| Move a 4B adapter to 8B later | Reuse datasets; train an adapter compatible with the exact new base. |
| The bot must fall if it walks over air | The inspected turtle movement code has no floor requirement. Player-like falling belongs to a different embodiment. [REPO-MOVE] |
| A turtle has player-like health/tool wear | Do not assume player mechanics; verify and preserve the selected CC:Tweaked behavior. |
| A server knows nothing about the player's look direction | Server observations include useful orientation; exact physical input still needs client instrumentation. |
| Blockly automatically renders our AST | We must implement and test IR-to-blocks and blocks-to-IR mappings. Arbitrary source-code round-trip is out of scope. |
| CC:Tweaked needs a custom Java interpreter for every program | A restricted interpreter in its existing Lua runtime is the default approach. |
| Only an old addon offers Create integration | Create's own documented ComputerCraft peripherals are a better first stop. [CREATE-CC] |
| The LLM can directly call a universal `build_tree_farm` | It can only call a registered, tested skill with declared requirements and real execution. |
| Raw play automatically labels intentions | Preserve explicit teaching/task markers and distinguish actors. Infer only with provenance and review. |
| A browser guard makes all turtle programming secure | Raw CraftOS is a separate control path; enforce strong policies at the appropriate server boundary. |
| A frontier repair is safe and automatically eligible for training | Validate the repair like any other proposal, and separately check data provenance/usage terms. |

Earlier specific model-price tables, GPU prices, difficulty scores, and broad comparative claims about YouTube agents were provisional discussion, not benchmarked findings. This project should collect its own measurements.

## 24. A map from ideas to implementation

| Conversation idea | Plan destination |
|---|---|
| Blockly on a real turtle, immediately useful | First vertical slice and editor/runtime milestones |
| Craftable bot; optional single starter | Survival contract and progression milestone |
| Kids and models write the same program | Canonical IR, block mapping, revision approval |
| Chat delegation and `/ai` | Authenticated command routing and task model |
| Local Qwen on separate 8 GB machine | Remote gateway, model adapter, inference benchmarks |
| Optional GPT advice/review | Cloud policy, debug export, parent-operated review workflow |
| Raw stream to disk | Journal, snapshots, provenance, offline extraction |
| Learning from play | Retrieval, shadow mode, corrections, later QLoRA |
| Good goals/tools/harness from Codex analysis | Debug bundles, skill promotion, implementation handoffs |
| Mineflayer player or own-character automation | Deferred alternate embodiments, not V0 dependencies |
| Create factories | Existing peripherals first, tested capability adapters later |
| Spider mech and pink sheep megaproject | Long-range body adapters and scenario gates |
| Funny narration and videos | Optional presentation layer and capture tooling |
| Programs as shareable objects | Named revisions, library functions, disks/cards |
| Legitimate survival and no hidden cheating | Resource, observation, permissions, and persistence invariants |

## 25. Questions to resolve by experiments

The following are research tasks, not reasons to delay the first coding milestone:

1. Which exact CC:Tweaked build and OneBlock implementation run correctly with the family's unchanged 1.20.1 modpack? Does turtle harvesting trigger OneBlock progression?
2. Does a browser plus trusted Lua IR runner provide enough polish before any Java changes? Which integration actually requires a native server bridge?
3. Which fixed block subset is the smallest that the kids find useful, and which explanation/step controls matter most?
4. Does Qwen produce valid useful programs from a concise capability catalog on the available GPU? Would a smaller model plus better examples outperform a larger one here?
5. Which observation facts are actually missing when a program fails? Add sensors based on evidence, not omniscient dumps.
6. Which protections must apply to all turtle operations, and which are optional Apprentice guardrails for trusted family play?
7. Which Create integrations work through existing peripherals, and which genuinely require an addon or fork hook?
8. Does retrieval plus verified libraries already achieve the desired experience without fine-tuning?
9. How should a bot core preserve identity across break/place, upgrade, repair, and eventual mech assembly without duplication?
10. Which long-range mech features fit the current modpack, and which need a separate experimental profile?

## 26. Research and source index

External references were checked during preparation of this document. Prefer pinned source and installed-version behavior over a moving website's latest examples. Repository references below refer to the initial inspected source; these files are evidence for integration decisions, not promises that future upstream revisions are identical.

- **[EDU]** [Minecraft Education: basic Agent movement](https://education.minecraft.net/en-us/lessons/basic-moves-the-agent), [build with the Agent](https://education.minecraft.net/en-us/lessons/build-with-the-agent), and [MakeCode](https://minecraft.makecode.com/). Educational interaction inspiration.
- **[MINE]** [PrismarineJS/Mineflayer](https://github.com/PrismarineJS/mineflayer). Separate programmable-player prior art.
- **[MIND]** [MINDcraft](https://github.com/mindcraft-bots/mindcraft). LLM/Mineflayer integration, local provider support, account and generated-code cautions. [Emergent Garden's channel](https://www.youtube.com/@EmergentGarden) is the likely remembered creator; an exact source video remains unidentified.
- **[CCT]** [CC:Tweaked documentation](https://tweaked.cc/) and [this fork's README](README.md).
- **[TURTLE]** [Turtle API](https://tweaked.cc/module/turtle.html). Validate details against the pinned 1.20.1 branch.
- **[CCT-HTTP]** [HTTP and WebSocket API](https://tweaked.cc/module/http.html).
- **[CCT-LAN]** [Local-IP restrictions](https://tweaked.cc/guide/local_ips.html). Prefer narrow endpoint permission over broadly disabling private-network restrictions.
- **[BLOCKLY]** [Blockly documentation](https://docs.blockly.com/).
- **[BLOCKLY-SAVE]** [JSON workspace serialization](https://docs.blockly.com/guides/configure/serialization/).
- **[BLOCKLY-GEN]** [Code generation and built-in language generators](https://docs.blockly.com/guides/create-custom-blocks/code-generation/overview/).
- **[BLOCKLY-RUN]** [Execution integration and JavaScript execution considerations](https://docs.blockly.com/guides/app-integration/running-javascript/). Blockly does not provide our game runtime.
- **[CREATE-CC]** [Create's ComputerCraft integration](https://github.com/Creators-of-Create/Create/wiki/ComputerCraft-Integration).
- **[AP]** [Advanced Peripherals](https://modrinth.com/mod/advancedperipherals), [maintainer documentation](https://docs.intelligence-modding.de/). Optional prior art; compatibility must be pinned.
- **[OLD-COMPUTING]** [Create: Computing](https://modrinth.com/mod/create-computing). Historical 1.18.2 addon mentioned in the conversation, not a recommended dependency.
- **[QWEN]** [Qwen3-8B model card](https://huggingface.co/Qwen/Qwen3-8B). A baseline candidate, not a claim of latest/best model.
- **[OLLAMA-MODEL]** [Qwen3 tags and quantizations](https://ollama.com/library/qwen3/tags).
- **[OLLAMA-STRUCT]** [Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs). Schema-constrained output still needs semantic validation.
- **[OLLAMA-MEM]** [Context length](https://docs.ollama.com/context-length) and [Ollama FAQ](https://docs.ollama.com/faq). Runtime memory and concurrency considerations.
- **[KIMI]** [Moonshot's Kimi K2 model card](https://huggingface.co/moonshotai/Kimi-K2-Instruct). Corrects earlier size claims.
- **[QLORA]** [QLoRA paper](https://arxiv.org/abs/2305.14314). Quantized-base adapter training.
- **[DAGGER]** [A Reduction of Imitation Learning and Structured Prediction to No-Regret Online Learning](https://arxiv.org/abs/1011.0686). Interactive imitation-learning background.
- **[CODEX]** [ChatGPT Work and Codex](https://help.openai.com/en/articles/20001275-chatgpt-work-and-codex). Local development/review workflow and billing distinction.
- **[CODEX-AUTH]** [Official Codex authentication documentation](https://developers.openai.com/codex/auth).
- **[CODEX-EXEC]** [Official Codex noninteractive execution](https://developers.openai.com/codex/noninteractive).
- **[BUDGET]** [API project management and budget behavior](https://help.openai.com/en/articles/9186755-managing-projects-in-the-api-platform). Recheck current provider controls when implementing billing.
- **[REPO-VERSIONS]** [Version catalog](gradle/libs.versions.toml).
- **[REPO-JDK]** [Build toolchain constants](buildSrc/src/main/kotlin/cc/tweaked/gradle/CCTweakedPlugin.kt) and [Java conventions](buildSrc/src/main/kotlin/cc-tweaked.java-convention.gradle.kts).
- **[REPO-ARCH]** [Repository architecture](projects/ARCHITECTURE.md).
- **[REPO-MOVE]** [TurtleMoveCommand](projects/common/src/main/java/dan200/computercraft/shared/turtle/core/TurtleMoveCommand.java).
- **[REPO-CONTRIB]** [Contribution instructions](CONTRIBUTING.md).
- **[REPO-LICENSE]** [REUSE declarations](REUSE.toml) and [original ComputerCraft license](LICENSES/LicenseRef-CCPL.txt).

**Bottom line:** start with a child-authored Blockly program controlling a genuinely crafted turtle. Preserve the same inspectable program and evidence model as the project grows toward an AI apprentice, Create engineer, and eventually the spider mech or self-contained pink sheep megamachine.
