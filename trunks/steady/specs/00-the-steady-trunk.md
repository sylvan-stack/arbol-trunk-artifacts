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

| Phase | Responsibility | Initial implementation |
|---|---|---|
| **Intake** | Preserve the submitted message, gather the initial context needed to understand it, and classify the request. | Pass through to Planning. |
| **Planning** | Use the request, context and labels to decide how to proceed. A simple question may go straight to Execution, while research or implementation may warrant a short plan. | Pass through to Execution. |
| **Execution** | Answer the request or carry out the plan. Steady may supply a workflow or let the agent decide how to proceed. Agent tool interactions happen here. | Run the existing single Driver inference and retain its outcome. |
| **Finalization** | Inspect or classify the outcome when needed and run any wrap-up operations before the Exchange ends. | Pass the Execution outcome through to the shared end handling. |

For example, an agent can request a tool during Execution and resume in another Turn when it gets the result. Both Turns belong to the same Execution phase and the same Exchange. A Driver run can contain multiple Turns.

#### Elma Chat phase preview

The Elma Chat preview runs a fictional Exchange on demand. It chooses a fresh random delay of 5 to 20 seconds for each phase and shows one sample message when the phase starts and another when it finishes. The preview keeps the Exchange's current phase and each message's originating phase in local state. A readable status names the phase in progress. The transcript keeps each phase's label and background tint as the Exchange advances. At completion the transcript remains available, and Replay starts a fresh simulation. This is a design preview with a local lifecycle. Production phase storage and Elma's Bridge interface remain open design work.

## Requirements

STE-0.1 through STE-0.14 use the corresponding Trivial requirements as their starting point. The checks describe what would qualify Steady's own Build.

| ID | The Steady Trunk must… | Origin | Verified by |
|---|---|---|---|
| STE-0.1 | make one Driver run on the handed route when an Exchange reaches inference, cancel without launching when a stop arrives first, and end with the Driver's outcome or the Exchange failure that forced the run to stop | TRV-0.1, ARC-2, ARC-3 | Exchange outcome and stop checks |
| STE-0.2 | finish after the first run outcome or route refusal, including `rate_limited`, with no retry or alternate route | TRV-0.2 | Driver failure and route refusal checks |
| STE-0.3 | resolve `arbol-<driver>-driver` on the search path using the handed Driver or the `driver` key in `~/.arbol/arbol-trunk/steady.json`, and start it as `run --control --request` with the private Exchange's `run-1.json` and no version flag. When neither input names a Driver, refuse with `invalid_input` and name the settings file and key | TRV-0.3 | Launcher and missing Driver checks |
| STE-0.4 | check launcher and Subscription capabilities before inference and leave validation of the handed model and effort to the Driver. Preserve the Driver's error when it refuses the route. Fail with `driver_failed` when the launcher is missing or the Driver name is invalid, naming the launcher, or when capability preparation fails, giving the reason | TRV-0.4 and Trivial's Driver capability checks | Missing launcher, capability and Driver refusal checks |
| STE-0.5 | write the DRV-2 request with `runId` `<exchange>.1`, `workingDirectory`, `tools: "all"`, `history`, and the message as user `input`. Include `subscriptionId`, `model` and `effort` only when the route names them, and use the chat identity in STE-0.14. Limit the request to these fields | TRV-0.5, ARC-4.7 | Driver request checks |
| STE-0.6 | send every earlier Exchange of the chat that has not been replaced, regardless of its outcome, in Exchange order. Within each Exchange, send the user message, every assistant text block exactly as saved and every top-level tool call with its recorded result or error, in position order and in DRV-2 form | TRV-0.6 | History checks for completed, partial and replaced Exchanges |
| STE-0.7 | read that history in the acceptance transaction, after marking any Exchange replaced by the resend | TRV-0.7 | Replacement and concurrent acceptance checks |
| STE-0.8 | use the Driver for the main inference only, even when handed a small-call route, with no classification or title generation | TRV-0.8 | Exchange check with a small-call route |
| STE-0.9 | send the user message and full saved history with the route and installation chat metadata described here. Add no system prompt or file contents. Read only Steady's own settings file for its default Driver | TRV-0.9, ARC-4 | Prompt, history and settings checks |
| STE-0.10 | stop a silent run and fail with `driver_silent` after 16 minutes without a Driver event while no tool call or question is open, including while a command left in the background runs. Measure elapsed time with `CLOCK_UPTIME_RAW` | TRV-0.10, ARC-3.4, ARC-3.10 | Silence and awake-clock checks |
| STE-0.11 | send a stop line to the Driver's control input when stopping a run. Check the deadline each loop and send `SIGKILL` if the process is still alive 16 seconds after the stop line or its `finished` event, whichever came first. Use `CLOCK_UPTIME_RAW` and end the Exchange after the Driver process has exited | TRV-0.11, ARC-3.10 | Stop, exit and awake-clock checks |
| STE-0.12 | use typed Python 3.14 on the interpreter packaged in its Build, depend only on the standard library and pinned psycopg 3, and share no implementation code with another Trunk | TRV-0.12 | Build policy and packaged interpreter checks |
| STE-0.13 | build `arbol-trunk/steady` from `trunks/steady/` with entry `bin/arbol-trunk-steady`. Declare `schemaCompatibility` and `installationLayout` in the Build description and record. Qualify a working Build with its unit checks, shared lifecycle checks on a single installation for each declared schema compatibility version, and one real message through the Claude Driver | TRV-0.13, ARC-4.3, ARC-5 | Unit checks, installation lifecycle checks and live Claude message |
| STE-0.14 | use the accepted chat's canonical lowercase, hyphenated UUID as `chatSessionId`, including for first and replacement messages. Add `arbolChat: true` for Claude and Codex. Send the continuity field alone for Pi and MiMo | TRV-0.14, ARC-4.7 | Driver request and live chat identity checks |
| STE-0.15 | run the phases in the order Intake, Planning, Execution and Finalization after acknowledging acceptance and before saving the end. Keep settlement on the shared recovery path | STE-D3, ARC-4.6 | Accepted Exchange and settlement checks |
| STE-0.16 | provide pass-through placeholders for Intake, Planning and Finalization, and run the single Driver inference in Execution as specified by STE-0.1 through STE-0.14 | STE-D3 | Phase order and baseline Exchange checks |
| STE-0.17 | demonstrate the four phases in an Elma Chat preview using a fictional Exchange, an independent random delay of 5 to 20 seconds per phase, and a message at each phase start and finish. Show the current phase separately from the stable phase labels and background tints on transcript content, and allow replay after completion | Owner direction | Elma phase preview |

## Interface

Use `arbol-<driver>-driver run --control --request <exchange-folder>/run-1.json`. The request file has mode `0600` and contains the fields in STE-0.5 and STE-0.14. The Driver uses the supplied `chatSessionId` and history to resume or rebuild its conversation under [DRV-2](../../../../arbol/driver-contract/02-inference.md#the-conversation).

For capability preparation, query `info`, `subscriptions families` and `subscriptions list`. Require the selected Driver identity, contract version 1 or 2, and available Subscription families. Bound each query to five seconds and 256 KiB, and cancel preparation on stop. When `~/.arbol/pi-cutover.json` exists, check it before the queries and again before inference. In a nonterminal phase, block a route when the Driver matches a `blockedRoutes` entry and the Subscription matches or is unspecified on either side. A malformed or unreadable marker fails closed. See [Trivial's Driver capability and migration checks](../../trivial/specs/00-the-trivial-trunk.md#driver-capability-and-migration-checks).

## Open decisions

What data does each phase need to receive and produce? Decide which information Elma Chat should show while an Exchange is active, which belongs in saved chat history, and which stays private to Steady. The initial phase hooks are internal. Phase payloads, persistence and Bridge interfaces are open for discussion, along with the work each placeholder will take on.

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| STE-D1 | What are the display name and code name of the new Trunk? | Steady Trunk, code name `steady`; requirement prefix `STE` | Decided by the owner on 11 November 2026: “name it "Steady Trunk"”; the lowercase code name follows the Trunk naming convention |
| STE-D2 | What role and starting behaviour should Steady have? | Use Trivial's requirements as the draft baseline for Arbol's primary Trunk | Owner direction, 19 November 2026 |
| STE-D3 | How should Steady organise an Exchange? | Intake → Planning → Execution → Finalization, following durable acceptance | The owner agreed to the phase names and requested placeholders on 19 November 2026 |
