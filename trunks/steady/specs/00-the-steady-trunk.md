---
role: authored
spec: STE-0
status: draft
---
# Steady Trunk · 00 The Steady Trunk

The **Steady Trunk**, code name `steady`, is intended to be Arbol's primary Trunk for everyday chat work. Its starting behaviour comes from the [Trivial Trunk specification](../../trivial/specs/00-the-trivial-trunk.md#requirements). The requirements below are a draft baseline for discussing what Steady should add or change.

Steady runs in the single Arbol installation described by [ARC-4](../../../../arbol/architecture/04-single-database.md) and uses the file layout in [ARC-5](../../../../arbol/architecture/05-where-arbol-keeps-its-files.md). Arbol can select it by its own code name. ARC-4 defines how the installation handles a start that omits the Trunk code name.

## Scope

This specification covers Steady's work inside a [Chat Exchange](../../../../arbol/GLOSSARY.md#chat-exchange), its Driver calls and conversation handling, and its program, Build and checks. [ARC-2](../../../../arbol/architecture/02-chat-exchange-across-components.md), [ARC-3](../../../../arbol/architecture/03-stuck-trunks.md) and ARC-4 govern the shared Exchange lifecycle. [DRV-2](../../../../arbol/driver-contract/02-inference.md) governs Driver inference, and [BLD-0](../../../../arbol/builds/00-builds-and-versions.md) governs Builds and the registry.

## Behaviour

At the baseline, Steady accepts a message, reads the chat history in the acceptance transaction and makes one Driver run on Arbol's chosen route. It streams events to the Exchange folder, saves completed messages and tool calls, and forwards stop and answer controls to the Driver. It ends the Exchange with the outcome of that run or the failure that forced it to stop.

Both detached and direct starts go through installation writer admission. Before acknowledging acceptance, Steady publishes the private Exchange folder and records the exact Trunk Build with the chat, message and open Exchange. If the database is unavailable when it has a pending end, Steady retains the folder. Settlement uses the recorded assignment, saves the result idempotently and performs only recovery work. See [ARC-4, Exchange lifecycle and persistence](../../../../arbol/architecture/04-single-database.md#exchange-lifecycle-and-persistence).

### Exchange Phases

An **Exchange Phase** is a stage of a Chat Exchange with a specific responsibility. It may do local work or contain one or more agent [Turns](../../../../arbol/GLOSSARY.md#turn). Durable acceptance and acknowledgement come before Intake. Steady writes and saves the Exchange end after Finalization. Recovery settles the recorded Exchange through the shared lifecycle.

#### Phase transitions

Steady starts Intake after acknowledgement. Each phase makes its hand-off by calling the next phase explicitly. Intake calls Planning when the request needs a plan, or calls Execution directly when the request can be handled immediately. Planning calls Execution with the plan available in the shared Exchange data. Execution calls Finalization with the outcome. Finalization hands that outcome back to the shared Exchange end handling.

| Phase | Responsibility | Initial implementation |
|---|---|---|
| **Intake** | Preserve the submitted message, gather initial context, classify the request and decide whether it needs Planning. | Prepare Cue Cards and call Planning. |
| **Planning** | Use the request, context and labels to decide how to approach work that needs a plan. | Pass through to Execution. |
| **Execution** | Answer the request or carry out the plan. Steady may supply a workflow or let the agent decide how to proceed. Agent tool interactions happen here. | Run the existing single Driver inference and retain its outcome. |
| **Finalization** | Inspect or classify the outcome when needed and run any wrap-up operations before the Exchange ends. | Pass the Execution outcome through to the shared end handling. |

For example, an agent can request a tool during Execution and resume in another Turn when it gets the result. Both Turns belong to the same Execution phase and the same Exchange. A Driver run can contain multiple Turns.

#### Phase data

Each phase owns its own **private phase data** for intermediate work. All phases can read and write the same **public Exchange data** for that Exchange. Before calling its successor, a phase writes any outputs the successor needs to the public data and passes it the same shared object. The successor starts with its own private data. Here public means shared across phases. The Elma Chat and persistence interfaces will define which fields they carry.

For example, Intake can put the request labels in the Exchange data and call Planning. Planning reads those labels, keeps its working notes private, writes the selected plan to the Exchange data and calls Execution.

#### Cue Cards in Intake

After acknowledgement, Intake matches the original accepted message once per Exchange using Forester's non-hook `cues prepare` operation. This reads Mycel's compiled catalog and reuses Forester's card ordering, source checks and 8,000-character inline budget. It treats changed sources as required reads and omits missing ones. Overflow goes into `cues.md` in the private Exchange folder and the context tells the agent to read it. Intake makes the prepared text available to Execution through public Exchange data. Steady keeps the original message, saved history and system prompt intact.

`forester cues prepare --overflow-file <absolute path>` reads the message as UTF-8 from standard input and returns one JSON object with `context`, a string or null. In `on` mode it returns prepared text, or an empty string when no card needs delivery. `off` also returns an empty string. `shadow` returns null so the normal hook can record its shadow result. The operation keeps the native-session delivery ledger for the standalone hooks. If preparation is unavailable or fails, Steady omits the context and the usual Driver and hook path applies.

Drivers that support this extension advertise `capabilities.cueContext: 1` in `info`. On those Drivers, the optional `cueContext` field carries up to 8,000 characters of context for the current native turn. The Driver adds nonempty context at native submission as a labelled Cue Card block. It preserves the canonical `input`, history and continuity fingerprints. The field's presence, even with empty text, means Steady owns delivery on this run. For Claude and Codex it requires the validated Arbol chat marker. Those Drivers set `ARBOL_DISABLED_HOOK_FEATURES=cue-card-injection` in that run's native environment using [Forester's list of optional hook features](../../../../forester/specs/02-hooks.md#optional-hook-features). Forester skips selecting and injecting Cue Cards for the marked run, while the ordinary hook processing continues. Without the field, the Drivers omit the variable. Steady omits the field for older Drivers.

#### Elma Chat phase preview

The Elma Chat preview runs a fictional Exchange on demand. It chooses a fresh random delay of 5 to 20 seconds for each phase and shows one sample message when the phase starts and another when it finishes. The preview keeps the Exchange's current phase and each message's originating phase in local state. Messages follow the regular chat layout. A faint background tint behind each group of messages is the only visible phase cue. Groups retain their tints as the Exchange advances. At completion the transcript remains available, and Replay starts a fresh simulation. This is a design preview with a local lifecycle. Production phase storage and Elma's Bridge interface remain open design work.

Cmd-click anywhere in a tinted transcript region to inspect the Exchange and the phase that produced that region. Cmd+Enter on the focused region opens the same dialog. The compact dialog shows the Exchange identity and progress, the selected phase, the Exchange's public data and that phase's private data in a JSON tree. The data and close control are visible without a title bar. The top-level fields are visible on opening. Each nonempty object or array can expand or collapse independently, including nested objects inside arrays. Nested collections start collapsed. The preview uses fictional phase data while the fields Elma will receive from Steady are being defined.

## Requirements

STE-0.1 through STE-0.14 use the corresponding Trivial requirements as their starting point. The checks describe what would qualify Steady's own Build.

| ID | The Steady Trunk must… | Origin | Verified by |
|---|---|---|---|
| STE-0.1 | make one Driver run on the handed route when an Exchange reaches inference, cancel without launching when a stop arrives first, and end with the Driver's outcome or the Exchange failure that forced the run to stop | TRV-0.1, ARC-2, ARC-3 | Exchange outcome and stop checks |
| STE-0.2 | finish after the first run outcome or route refusal, including `rate_limited`, with no retry or alternate route | TRV-0.2 | Driver failure and route refusal checks |
| STE-0.3 | resolve `arbol-<driver>-driver` on the search path using the handed Driver or the `driver` key in `~/.arbol/arbol-trunk/steady.json`, and start it as `run --control --request` with the private Exchange's `run-1.json` and no version flag. When neither input names a Driver, refuse with `invalid_input` and name the settings file and key | TRV-0.3 | Launcher and missing Driver checks |
| STE-0.4 | check launcher and Subscription capabilities before inference and leave validation of the handed model and effort to the Driver. Preserve the Driver's error when it refuses the route. Fail with `driver_failed` when the launcher is missing or the Driver name is invalid, naming the launcher, or when capability preparation fails, giving the reason | TRV-0.4 and Trivial's Driver capability checks | Missing launcher, capability and Driver refusal checks |
| STE-0.5 | write the DRV-2 request with `runId` `<exchange>.1`, `workingDirectory`, `tools: "all"`, `history`, and the message as user `input`. Include `subscriptionId`, `model` and `effort` only when the route names them, use the chat identity in STE-0.14 and include `cueContext` as specified in STE-0.21. Limit the request to these fields | TRV-0.5, ARC-4.7, STE-0.21 | Driver request checks |
| STE-0.6 | send every earlier Exchange of the chat that has not been replaced, regardless of its outcome, in Exchange order. Within each Exchange, send the user message, every assistant text block exactly as saved and every top-level tool call with its recorded result or error, in position order and in DRV-2 form | TRV-0.6 | History checks for completed, partial and replaced Exchanges |
| STE-0.7 | read that history in the acceptance transaction, after marking any Exchange replaced by the resend | TRV-0.7 | Replacement and concurrent acceptance checks |
| STE-0.8 | use the Driver for the main inference only, even when handed a small-call route, with no classification or title generation | TRV-0.8 | Exchange check with a small-call route |
| STE-0.9 | send the user message and full saved history with the route and installation chat metadata described here. Deliver prepared Cue Cards as turn context under STE-0.21. Read only Steady's own settings file for its default Driver | TRV-0.9, ARC-4, STE-0.21 | Prompt, history and settings checks |
| STE-0.10 | stop a silent run and fail with `driver_silent` after 16 minutes without a Driver event while no tool call or question is open, including while a command left in the background runs. Measure elapsed time with `CLOCK_UPTIME_RAW` | TRV-0.10, ARC-3.4, ARC-3.10 | Silence and awake-clock checks |
| STE-0.11 | send a stop line to the Driver's control input when stopping a run. Check the deadline each loop and send `SIGKILL` if the process is still alive 16 seconds after the stop line or its `finished` event, whichever came first. Use `CLOCK_UPTIME_RAW` and end the Exchange after the Driver process has exited | TRV-0.11, ARC-3.10 | Stop, exit and awake-clock checks |
| STE-0.12 | use typed Python 3.14 on the interpreter packaged in its Build, depend only on the standard library and pinned psycopg 3, and share no implementation code with another Trunk | TRV-0.12 | Build policy and packaged interpreter checks |
| STE-0.13 | build `arbol-trunk/steady` from `trunks/steady/` with entry `bin/arbol-trunk-steady`. Declare `schemaCompatibility` and `installationLayout` in the Build description and record. Qualify a working Build with its unit checks, shared lifecycle checks on a single installation for each declared schema compatibility version, and one real message through the Claude Driver | TRV-0.13, ARC-4.3, ARC-5 | Unit checks, installation lifecycle checks and live Claude message |
| STE-0.14 | use the accepted chat's canonical lowercase, hyphenated UUID as `chatSessionId`, including for first and replacement messages. Add `arbolChat: true` for Claude and Codex. Send the continuity field alone for Pi and MiMo | TRV-0.14, ARC-4.7 | Driver request and live chat identity checks |
| STE-0.15 | start Intake after acknowledging acceptance. Have Intake explicitly call Planning or Execution, have Planning call Execution, and have Execution call Finalization before shared end handling. Keep settlement on the shared recovery path | STE-D3, STE-D4, ARC-4.6 | Direct and planned Exchange paths and settlement checks |
| STE-0.16 | prepare Cue Cards in Intake, provide pass-through placeholders for Planning and Finalization, and run the single Driver inference in Execution | STE-D3, STE-0.20 | Phase order and baseline Exchange checks |
| STE-0.17 | demonstrate the four phases in an Elma Chat preview using a fictional Exchange, an independent random delay of 5 to 20 seconds per phase, and a message at each phase start and finish. Track current phase separately from each message's origin. Use regular chat layout with a faint, stable background tint as the only visible phase cue, and allow replay after completion | Owner direction | Elma phase preview |
| STE-0.18 | give each phase its own private data and read/write access to the same public Exchange data. Carry shared outputs to the next phase through the public Exchange data | STE-D4 | Phase data ownership and hand-off checks |
| STE-0.19 | let the Elma phase preview open a compact, read-only inspector for a tinted transcript region with Cmd-click or Cmd+Enter. Show the Exchange identity and progress, selected phase, public Exchange data and private phase data in a JSON tree. Keep top-level fields visible on opening and allow independent expansion and collapse of individual objects and arrays | Owner direction | Elma phase inspection preview |
| STE-0.20 | match and prepare Cue Cards once in Intake from the accepted user message using the compiled catalog, existing source checks and the owner's delivery mode. Keep overflow in the private Exchange folder and pass prepared context to Execution | Owner direction | Intake and catalog preparation checks |
| STE-0.21 | deliver prepared `cueContext` through a supporting Driver for the current native turn. Preserve original input, history and system prompt. For Claude and Codex, disable the `cue-card-injection` hook feature only for the run that carries `cueContext`. Continue the rest of Forester's hook processing | Owner direction | Driver delivery, continuity and hook checks |

## Interface

Use `arbol-<driver>-driver run --control --request <exchange-folder>/run-1.json`. The request file has mode `0600` and contains the fields in STE-0.5 and STE-0.14. The Driver uses the supplied `chatSessionId` and history to resume or rebuild its conversation under [DRV-2](../../../../arbol/driver-contract/02-inference.md#the-conversation).

For capability preparation, query `info`, `subscriptions families` and `subscriptions list`. Require the selected Driver identity, contract version 1 or 2, and available Subscription families. Bound each query to five seconds and 256 KiB, and cancel preparation on stop. When `~/.arbol/pi-cutover.json` exists, check it before the queries and again before inference. In a nonterminal phase, block a route when the Driver matches a `blockedRoutes` entry and the Subscription matches or is unspecified on either side. A malformed or unreadable marker fails closed. See [Trivial's Driver capability and migration checks](../../trivial/specs/00-the-trivial-trunk.md#driver-capability-and-migration-checks).

## Open decisions

Which fields belong in each phase's private data and in the public Exchange data? Decide which of the shared fields Elma Chat should receive live, which belong in saved chat history, and which are needed only while the Exchange runs. The field definitions, persistence and Bridge interfaces remain open, along with the work each placeholder will take on.

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| STE-D1 | What are the display name and code name of the new Trunk? | Steady Trunk, code name `steady`; requirement prefix `STE` | Decided by the owner on 11 November 2026: “name it "Steady Trunk"”; the lowercase code name follows the Trunk naming convention |
| STE-D2 | What role and starting behaviour should Steady have? | Use Trivial's requirements as the draft baseline for Arbol's primary Trunk | Owner direction, 19 November 2026 |
| STE-D3 | How should Steady organise an Exchange? | Intake → Planning → Execution → Finalization, following durable acceptance | The owner agreed to the phase names and requested placeholders on 19 November 2026 |
| STE-D4 | How do phases hand off work? | Each phase calls its successor explicitly. Intake can skip Planning. Each phase owns private data and can read and write the public Exchange data | Owner direction, 19 November 2026 |
