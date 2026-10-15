---
role: authored
spec: TRV-0
status: review
---
# Trivial Trunk · 00 The Trivial Trunk

The *Trivial Trunk*, code name `trivial`, is the smallest [Arbol Trunk](../../../../arbol/GLOSSARY.md#arbol-trunk). For each [Chat Exchange](../../../../arbol/GLOSSARY.md#chat-exchange) it makes exactly one run of a Driver, the program that starts a vendor tool such as `claude` ([DRV-2](../../../../arbol/driver-contract/02-inference.md)), with the route it is handed: the Driver, the [Subscription](../../../../arbol/GLOSSARY.md#subscription), the model and the effort that Arbol chose for the message. It sends the agent the chat so far. For example, `cd ~/repos/shop && arbol-trunk run "What does build.sh do?"` starts it in the current Root: it starts `arbol-claude-driver` once ([TRV-D7](#decisions)), lets the agent work in `~/repos/shop`, and prints the lines of the answer, as JSON, as they arrive. This specification states what it does inside an Exchange, which the Root Contract leaves to each Trunk: the one run, the history it sends the agent, the request it writes for the Driver, what it uses when it is handed no route, what it never does, how it stops a run, how its work is laid out, its program, Build and checks, and what may be added to it later. It matters because it is the Trunk that still works when every other Trunk fails: it is the default Trunk of every Root, and no Root is marked working without it ([RCT-0](../../../../arbol-root/contract/00-overview.md#trunks-and-their-code-names)).

It follows the conventions of the [Driver Contract](../../../../arbol/driver-contract/README.md#conventions) and the additions in this folder's [README](README.md#conventions). It implements version 1 of the [Root Contract](../../../../arbol-root/contract/README.md) entirely and repeats none of it: the command `arbol-trunk`, the Exchange folder, accepting, stopping, ending and settling, the chat tables and the conformance suite are there ([RCT-0](../../../../arbol-root/contract/00-overview.md) to [RCT-4](../../../../arbol-root/contract/04-conformance.md)). The owner asked for it on 15 October 2026 (SYL-168) and decided what it is the same day ([TRV-D1](#decisions), TRV-D10, TRV-D11). Which Driver it uses when it is started by hand without one (TRV-D7), and whether a Build needs one real message (TRV-D12), wait for the owner. Where the Root Contract still waits for the owner (RCT-D16, RCT-D28, RCT-D29, RCT-D30), this text follows its proposed answers.

## Scope

Covered: what it is, one run for each Exchange, the history it sends, the agent's own conversation, the request to the Driver, a start without a route, what it never does, stopping the run, one loop and one watch, its program and Build, its checks, what may be added later.

Not covered: everything the Root Contract states for every Trunk, such as the command and what is handed to a Trunk ([RCT-0](../../../../arbol-root/contract/00-overview.md)), the Exchange folder ([RCT-1](../../../../arbol-root/contract/01-the-exchange-folder.md)), accepting, stopping, ending and settling ([RCT-2](../../../../arbol-root/contract/02-starting-stopping-and-settling.md)), the chat tables ([RCT-3](../../../../arbol-root/contract/03-the-chat-tables.md)) and the conformance suite ([RCT-4](../../../../arbol-root/contract/04-conformance.md)); the Driver's run ([DRV-2](../../../../arbol/driver-contract/02-inference.md)); Builds and the registry ([BLD-0](../../../../arbol/builds/00-builds-and-versions.md)); titles and other *small calls*, the short model calls beside the agent's work (ARC-0), and classifications, which it does not make (TRV-D1).

## Behaviour

### What it is

| | |
|---|---|
| Its code name | `trivial`, the Trunk that a command gets when it names none, in every Root (RCT-0) |
| What it does | One Driver run for each Exchange, with the route it is handed, and nothing else (TRV-D1) |
| What it implements | Version 1 of the Root Contract, all of it: `"rootContracts": [1]` |
| Where it lives | The folder `trunks/trivial/` of the `arbol-trunk` repository, built as the component `arbol-trunk/trivial` |
| Why it is small | It is the fallback when every other Trunk fails, so it carries as little logic as it can, and what may be added to it is limited ([below](#what-may-be-added-later)) |

### One Exchange, one run

The *route* is what Arbol chose for the message: the Driver, the Subscription, the model and the effort (RCT-0). The Trivial Trunk works through an Exchange in five steps:

1. It accepts the message as RCT-2 says. In the same transaction it reads the history of the chat ([below](#the-history-it-sends)), so once it has accepted, it needs the database only to save ([TRV-D3](#decisions)).
2. It writes the request of the run, `run-1.json`, into the Exchange's folder, where every file has the mode 0600 (RCT-1).
3. It starts the Driver through its launcher, the program on the search path that starts the Driver's newest working Build (BLD-0), as `run` with a control input and the request file (RCT-0). The *control input* is the run's standard input, which stays open for stop and for answers to the agent's questions (DRV-2). Just before, it reads `control`: after a stop line it starts no run and ends the Exchange as cancelled (RCT-2).
4. It hands every event of the run to `events.jsonl` and to the chat tables (RCT-1, RCT-3), and every `stop` and `answer` line of `control` to the run's control input (RCT-1).
5. When the run is over ([below](#stopping-the-run)), it ends the Exchange with the run's outcome (RCT-2).

```sh
arbol-trunk --root golden trivial -v 1 run --detach --exchange 2d4f6a8c-… --request /…/start.json   # arbol-root starts the Trivial Trunk
arbol-claude-driver run --control --request /…/exchanges/2d4f6a8c-…/run-1.json                      # which starts the one run
```

There is exactly one run, and the Exchange ends as the run ends. For example, a run that ends with `{"code":"rate_limited","retryable":true,"retryAfterSeconds":1800,…}` ends the Exchange as failed with that error at once. The Trivial Trunk waits for nothing and tries no other route; Arbol offers to send the message again with what its [Brain Recipe](../../../../arbol/GLOSSARY.md#brain-recipe) picks next (RCT-2, ARC-0).

Observed in version 1: one message could take up to three runs. After transport trouble it was repeated twice while nothing had been stored, and after a rate limit it moved to another route of the same vendor, with the waits held in memory and the failures sorted by their wording.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Failures and what the Leather Bag sees`

### The history it sends

A Driver keeps no conversation of its own, so every run is sent the *history*: the earlier conversation of the chat, oldest first ([DRV-2](../../../../arbol/driver-contract/02-inference.md#the-conversation)). The Trivial Trunk sends the chat as the [Leather Bag](../../../../arbol/GLOSSARY.md#leather-bag) sees it ([RCT-3](../../../../arbol-root/contract/03-the-chat-tables.md#how-a-chat-is-read)): every earlier Exchange of the chat that was not replaced, whatever its end, in the order of their numbers, and within each the rows in the order of their positions ([TRV-D2](#decisions)).

| Row of the chat tables | Item of the history (DRV-2) |
|---|---|
| The Leather Bag's message, `messages` at position 0 | `{"type": "message", "role": "user", "text"}` |
| A block of the answer, `messages` with the role `assistant` | `{"type": "message", "role": "assistant", "text"}`, one for each block, with the text exactly as it was saved |
| A tool call the agent made itself, `tool_calls` without `parent_call_id` | `{"type": "tool_call", "name", "input", "ok", "result", "error"}`, with `result` and `error` left out where they are NULL |
| A tool call of a sub-agent, `tool_calls` with `parent_call_id` | Left out, as the chat leaves it out |

An Exchange that failed, was cancelled or was interrupted is sent with what it produced, so the agent knows what it already did, also when it stopped half-way. A replaced Exchange is never sent (RCT-3). For a message sent again, the Trunk reads the history after it has marked the Exchange it replaces, in the same transaction (RCT-2), so that Exchange is left out of the new run's own history too. The text of an answer goes back as it was saved, which is exactly as the Driver's `text` events carried it, apart from a NUL character and half of an emoji split in two, which the database cannot store and RCT-3 saves as the replacement character �. A Driver recognises a conversation by that text (DRV-6); where a replaced character makes it differ, the Driver builds the conversation anew, and nothing else follows.

The history is read within the 10 seconds that the acceptance may spend on the database (RCT-2). A chat so long that its history cannot be read in that time has its message refused with `database_unavailable`, and the Draft stays.

The request file of the second message of a chat, for the route of RCT-0's example:

```json
{
  "runId": "2d4f6a8c-1b3e-4c5d-8e7f-9a0b1c2d3e4f.1",
  "subscriptionId": "anthropic",
  "model": "sonnet",
  "effort": "high",
  "workingDirectory": "/Users/sam/repos/shop",
  "history": [
    { "type": "message", "role": "user", "text": "What does build.sh do?" },
    { "type": "message", "role": "assistant", "text": "Let me read it." },
    { "type": "tool_call", "name": "shell", "input": { "command": "cat build.sh" },
      "ok": true, "result": { "output": "#!/bin/sh\nset -e\n…", "exitCode": 0 } },
    { "type": "message", "role": "assistant", "text": "It compiles the three packages and copies them to dist/." }
  ],
  "input": { "role": "user", "text": "Add a --clean option to it." },
  "chatSessionId": "c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b",
  "tools": "all"
}
```

Observed in version 1: the agent was sent the messages and the text of the earlier answers, but no tool call and no result, so it knew what it had said and not what it had read or done. A failed message sent again gave the agent nothing of the failed run.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/07-agent-context.md#What the agent sees of the conversation`
- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Retry and Continue from failure point`

### The agent's own conversation

The vendor tools keep a saved conversation of their own, which still holds what the agent read and did. A Driver continues it when the caller names a *chat session*, an identifier of the caller's own for the conversation, and the history still matches what the saved conversation covers; otherwise it builds a new one from the history (DRV-2, [DRV-6](../../../../arbol/driver-contract/06-state-and-simultaneous-calls.md#the-map-of-chat-sessions)). The Trivial Trunk names the chat's identifier as the chat session, `chatSessionId` ([TRV-D4](#decisions)).

| The next run of the chat… | The Driver… |
|---|---|
| has the Subscription and the working folder of the chat's last completed run, and a history that matches | continues the saved conversation: `started` says `resumed` |
| has another Subscription or working folder, or follows a run that did not complete | builds a new one from the history: `started` says `rebuilt` |
| runs a message sent again | finds that the history no longer holds the replaced answer and builds a new one, so the replaced answer does not reach the agent this way either |

For example, the second message above, sent on the same route as the first, starts with `"continuity": "resumed"`: the vendor tool goes on with its own record of the conversation, the agent's thinking included, which the history does not carry.

### The request to the Driver

| Field of the request (DRV-2) | What the Trivial Trunk writes |
|---|---|
| `runId` | `<exchange>.1`: the Exchange's first and only run |
| `subscriptionId`, `model`, `effort` | From the route, each only when the route names it |
| `workingDirectory` | The working folder (RCT-0) |
| `history` | As above; `[]` for the first message of a chat |
| `input` | The Leather Bag's message, `{"role": "user", "text"}` |
| `chatSessionId` | The chat's identifier |
| `tools` | `all`: every tool the vendor tool offers (DRV-3) |
| Anything else | Nothing: no system prompt, no limits, no labels, no further folders, no search path ([TRV-D5](#decisions)) |

With no limits in the request, the run has the Driver's own: an idle limit of 15 minutes and no limit on the whole run ([DRV-4](../../../../arbol/driver-contract/04-errors-and-limits.md#time-limits-of-a-run)).

The route's `driver` names the launcher: `claude` starts `arbol-claude-driver`, and in general `<driver>` starts `arbol-<driver>-driver`. The Trivial Trunk speaks version 1 of the Driver Contract. It names no `-v`, so the Driver's newest working Build runs, and the run's `started` event says which (DRV-2); this is right only while every working Build of a Driver implements version 1 ([ARC-1](../../../../arbol/architecture/01-roots.md#the-states-of-a-root-and-the-current-root)). It checks nothing of the route itself ([TRV-D6](#decisions)): a Driver refuses a Subscription, a model or an effort it cannot use before anything is sent, with a `finished` event that is the only event of its run (DRV-2), and the Exchange fails with that error. When the launcher refuses because the Driver has no working Build (BLD-0), or no folder of the search path holds the launcher, for example for a mistyped `--driver cluade`, no `finished` event comes, and the Exchange fails with `driver_failed`, whose message carries the launcher's refusal or names the launcher it looked for (RCT-2). The launcher is looked for only in those folders, so a `driver` with a `/` in it, such as `../x`, is never found and fails the same way.

### A start without a route

Arbol always hands the Driver, the Subscription and the model (ARC-0). Started by hand, the Trivial Trunk may be handed part of a route or none:

| Not handed | It uses |
|---|---|
| The Driver | `claude`: it starts `arbol-claude-driver` (TRV-D7) |
| The Subscription | None named: the Driver takes the one its settings file names, and else its own default, `anthropic` for the Claude Driver ([DRV-0](../../../../arbol/driver-contract/00-overview-and-boundary.md#the-settings-file)) |
| The model, the effort | None named: the Driver takes those its settings file names for that Subscription, and else the vendor tool's own (DRV-0) |

The record and the row of the Exchange keep the route as it was handed, `{}` when none was (RCT-1, RCT-3). The run's `started` event, kept in `runs.started`, names the Subscription, the model and the effort that ran. For example, with the settings file of DRV-0's example:

```sh
cd ~/repos/shop && arbol-trunk run "What does build.sh do?"
# {"accepted":true,"duplicate":false,"root":"golden",…,"trunk":"trivial","build":"1"}
# {"n":1,"event":{…,"type":"started","continuity":"fresh","subscriptionId":"anthropic","model":"opus","effort":"high",…}}
```

### What it never does

1. **No small call and no title.** A *small call* is a short model call beside the agent's work, such as one that writes a title (ARC-0). The Trivial Trunk makes none, also when it is handed a small-call route, which stays in the record as it was handed (RCT-1). It writes no title: a chat it starts keeps `chats.title` NULL until the Leather Bag gives it one in Arbol (RCT-3).
2. **No classification.** It never starts `arbol-classifier`.
3. **No second run.** It never repeats a run and never takes another route, whatever the run ended with, a rate limit and a refusal included.
4. **No context of its own.** It sends no system prompt, reads no file for the agent and shortens no history. The agent gets the history, the message, and what the vendor tool brings itself, such as the instructions it finds in the working folder, which a Driver does not switch off ([DRV-D26](../../../../arbol/driver-contract/03-tools-and-permissions.md#decisions)). A long chat can keep working while the Driver continues the vendor tool's own record, and then fail with `context_exceeded` (DRV-4) on every message once the conversation is built anew from the whole history, for example after a stop, a failure or a change of Subscription. The Leather Bag then goes on in a new chat.
5. **No settings of its own.** It reads no settings file. What it uses is handed to it, or is the Driver's default.

Observed in version 1: a system prompt was put together from files for every message, a missing template failed the message, and the files contradicted each other. An optional addition to the message, Cues, sat on the way from Send to the agent and grew to some 7,500 lines of code around a useful core of 600. A title was made beside the first message of each chat.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/07-agent-context.md#System instructions`
- `arbol_deprecated_v1:product-spec/07-agent-context.md#Known problems and lessons`
- `arbol_deprecated_v1:product-spec/02-chat-sessions.md#Titles`

### Stopping the run

The termination signal ends the Trunk's whole process group, as RCT-2 says. Apart from that, the Trivial Trunk stops its run in one way, whatever the reason: a stop line in `control` ([RCT-2](../../../../arbol-root/contract/02-starting-stopping-and-settling.md#stop)), a run that has fallen silent (below), a full `events.jsonl` (RCT-1), or a folder it cannot write (RCT-2) ([TRV-D8](#decisions)):

1. It writes `{"type":"stop"}` to the run's control input.
2. It gives the Driver's process 16 seconds from that line to end, one second more than a Driver has for a stop (DRV-2). It checks this time in each round of its loop ([below](#one-loop-and-one-watch)) and never waits for it in one piece.
3. When the process still lives then, it ends it with SIGKILL.

The run is over when the Driver's process has ended and its output has been read to the end, for at most one second after the process ended; only then does the Exchange end (RCT-2). For a stop line in `control`, that makes at most 1 second to hand the line on, 16 seconds for the Driver and 1 second to read the rest: the end is in the folder within the 18 seconds of RCT-2. The same 16 seconds count from a run's `finished` event when that comes first: a Driver still alive 16 seconds after its `finished` is ended with SIGKILL too. These times are counted with `CLOCK_UPTIME_RAW`, the clock that stands still while the Mac sleeps (RCT-1), and divided by `ARBOL_TEST_TIME_SCALE`, as the 18 seconds of a stop are (RCT-4). What a Driver ended with SIGKILL leaves running, such as the vendor tool, its next call ends (DRV-6).

A run *falls silent* when it gives no event for one minute longer than its idle limit, while no tool call and no question is open and the run has made no `shell` call, a sub-agent's included ([RCT-2](../../../../arbol-root/contract/02-starting-stopping-and-settling.md#a-driver-that-falls-silent), RCT-D30). Because the request sets no limit, the idle limit is the Driver's 15 minutes, and the Trivial Trunk counts 16 minutes from the run's last event, with `CLOCK_UPTIME_RAW`, so a Mac that sleeps for an hour does not make a run silent. For example, a run that has written half its answer and then gives nothing for 16 minutes gets a stop line; its Driver has hung and does not end, so 16 seconds later the Trivial Trunk ends it with SIGKILL, and the Exchange fails with `driver_silent`.

### One loop and one watch

The Trivial Trunk does its work in one loop in one thread, an event loop of Python's `asyncio` ([TRV-D9](#decisions)). Each round of the loop waits at most one second for output of the Driver, a wait that `ARBOL_TEST_TIME_SCALE` divides as it divides the other waits of the loop (RCT-4), and never waits past the next deadline of [Stopping the run](#stopping-the-run); then it appends what arrived to `events.jsonl`, reads the new lines of `control` and hands them on, checks those deadlines, and raises the heartbeat (RCT-1). So `control` is read and the heartbeat rises at least once a second, whatever the agent and the database do.

The loop never waits for the database. It saves the rows that are due in a task beside it, one at a time and in order, on psycopg's asynchronous connection. Each call has a limit of 5 seconds, to connect and for each statement (RCT-2). A call that passes it is given up with its connection, and the rows wait for the next try, 5 seconds later, on a new connection. The *watch* of [RCT-1](../../../../arbol-root/contract/01-the-exchange-folder.md#the-trunk-watches-itself), which ends the Trunk when the heartbeat has stopped rising for 60 seconds, is a second thread that does nothing else.

| The Driver's… | The Trivial Trunk… |
|---|---|
| Control input | writes each line without waiting; what the Driver has not taken yet is kept and written, in order, in a later round |
| Output | reads it line by line, however long a line is, since a `tool_finished` can carry a result of 200 KiB (DRV-2); a line that is not an event is not appended to `events.jsonl` |
| Standard error and environment | leaves them the Trunk's own, so standard error goes to `/dev/null` when the Trunk runs in the background (RCT-2) |

For example, while the agent runs `make` for two hours, every round finds no output, raises the heartbeat, and comes round again within a second. While the database server restarts in the middle of an answer, a stop line still reaches the Driver within a second.

### Its program and its Build

| | |
|---|---|
| The program | Typed Python 3.14, checked with `mypy` in its strict form and with `ruff` when it is built, on the interpreter the Build carries in `runtime/python/`, as the Drivers and `arbol-classifier` do ([ARC-0.3](../../../../arbol/architecture/00-components-and-boundaries.md#languages)) |
| What it uses | The standard library, and psycopg 3 at a version pinned in its `uv.lock`, in its `binary` form, which brings its own PostgreSQL client library, so the Build needs nothing installed beside it ([TRV-D10](#decisions)). `scripts/package_build.py` copies `psycopg` and `psycopg-binary`, as `uv sync --locked` installed them, into the site-packages of the interpreter the Build carries, where the isolated interpreter of the entry finds them. On 15 October 2026 the newest version on PyPI was 3.3.6, with a wheel for Python 3.14 on Apple silicon and no further dependency on Python 3.14 (read on pypi.org that day; nothing was installed) |
| Its folder | `trunks/trivial/` in the `arbol-trunk` repository, with its own `package.json`, which names the version that `arbol-builds` reads (BLD-0), `pyproject.toml`, `uv.lock`, `arbol-build.json`, `bin/`, `src/`, `tests/` and `scripts/`. It shares no code with another Trunk (RCT-0) |
| Its component | `arbol-trunk/trivial`, whose Builds are `arbol-trunk/trivial@1`, `@2` and so on |
| Its entry | `bin/arbol-trunk-trivial`: a few lines of shell that `exec` the Build's own interpreter, isolated from the caller's Python settings, on `bin/entry.py`, as the entry of `arbol-classifier` does, so that the Trunk keeps the process identifier the launcher started (RCT-2) |
| Who starts it | The launcher `arbol-trunk` alone (RCT-0), and the checks below. `arbol-trunk-trivial` is never on the search path |

```json
{
  "component": "arbol-trunk/trivial",
  "build": [
    "uv sync --locked --group dev",
    "uv run --no-sync ruff check src tests scripts bin/entry.py",
    "uv run --no-sync mypy",
    "uv run --no-sync python scripts/package_build.py"
  ],
  "entry": "bin/arbol-trunk-trivial",
  "rootContracts": [1],
  "checks": [
    { "name": "unit", "kind": "unit",
      "command": "\"$ARBOL_BUILD_DIR/runtime/python/bin/python3.14\" -I -B \"$ARBOL_BUILD_DIR/scripts/check_build.py\" unit" },
    { "name": "contract-1", "kind": "contract",
      "command": "\"$ARBOL_BUILD_DIR/runtime/python/bin/python3.14\" -I -B ~/repos/sylvan-stack/arbol-root/contract/conformance/bin/rct-conformance.py --trunk \"$ARBOL_BUILD_DIR/bin/arbol-trunk-trivial\" --root-contract 1" },
    { "name": "live", "kind": "live",
      "command": "\"$ARBOL_BUILD_DIR/runtime/python/bin/python3.14\" -I -B \"$ARBOL_BUILD_DIR/scripts/check_build.py\" live" }
  ]
}
```

### Its checks

A Build counts as working only when all three checks have passed (BLD-0); the third, `live`, waits for the owner's answer to [TRV-D12](#decisions).

| Check | What runs | Costs |
|---|---|---|
| unit | Its parts, without a database server, a Driver or a network: the history made from rows, also for a message sent again, the request, the name of the launcher and the message that names it, the 16 minutes of silence on a clock the test moves, stopping a run whose Driver does not end, that a slow database holds up no round, and that it imports nothing but the standard library and psycopg, from the Build itself and in psycopg's `binary` form; and the Root Contract's requirements of a Trunk that name `unit`: RCT-0.1, RCT-0.2, RCT-0.12, RCT-0.15 (a Keychain read that fails or gives no answer within 2 seconds is refused with `database_unavailable`) and RCT-3.11 | Nothing |
| contract | The Root Contract's conformance suite, every scenario, once for each version in `rootContracts`: `contract-1` for version 1 ([RCT-4](../../../../arbol-root/contract/04-conformance.md#how-a-trunk-builds-check-runs-it)) | Nothing |
| live | One real message through the Claude Driver (below) | About 18,500 input tokens from the test account's usage window, which the vendor prices at about 4 US cents |

The live check works in five steps:

1. It sets up a home folder, a database server and a Root record `rct` of its own, with the `layout.sql` of the same `arbol-root` checkout as `contract-1`, as the suite does, and gives the Trunk only `HOME`, `PATH` and `LANG` (RCT-4, RCT-D29).
2. First on the search path it puts a launcher of its own named `arbol-claude-driver`. It runs the installed `~/.local/bin/arbol-claude-driver` with the owner's home folder and with `--state-dir` set to the Driver's test state directory, or to `ARBOL_BUILD_TEST_STATE_DIR` when that is set, where the test Subscription `test-anthropic` is signed in ([DRV-D41](../../../../arbol/driver-contract/05-conformance.md#decisions)).
3. In an empty folder, it runs:

   ```sh
   "$ARBOL_BUILD_DIR/bin/arbol-trunk-trivial" --root rct run --driver claude --subscription test-anthropic --model haiku \
     "Answer with the single word: ready"
   ```

4. It passes when the exit status is 0, the Exchange completed, `messages` holds a row with the role `assistant`, and `runs` holds one run of the kind `agent` with its usage, whose `started` names `test-anthropic`.
5. It removes its home folder, its database server and the empty folder.

The token count is that of a message with tools on through the Claude Driver on Haiku, recorded on 6 October 2026: five times the 3,635 input tokens of a Driver's own build message, which runs without tools, and more when the owner's skills and MCP servers come with the run (DRV-D27). The live check leaves one small saved conversation in the test Subscription's home. A build while that Subscription is signed out or its allowance is used up fails, like a Driver's (BLD-0), and the last working Build stays in use.

### What may be added later

The Trivial Trunk changes only for one of three reasons ([TRV-D11](#decisions)):

1. **A version of the Root Contract that a Root uses.** It works in every Root, so before a Root of a new version can be marked working, the Trivial Trunk is built for that version; `arbol-roots ready` marks no Root working without it (RCT-0).
2. **A change of the Driver Contract** that its request or its reading of the events needs, such as a working Driver Build that implements another version of it (ARC-1).
3. **A defect.**

Anything else, however small, goes into another Trunk: a title, a system prompt, a retry, a second run, a setting of its own, another library. A new Driver needs no change, because the route's `driver` names the launcher.

## Requirements

| ID | The Trivial Trunk must… | Origin | Verified by |
|---|---|---|---|
| TRV-0.1 | make one Driver run for each Exchange, on the route it was handed, and none when a stop line came before the run started, and end the Exchange with that run's outcome, or with the cause of RCT-2 for which it stopped the run | Changed | contract |
| TRV-0.2 | start no second run after any outcome, `rate_limited` and a refused route included, and take no route other than the one it was handed | Dropped | contract |
| TRV-0.3 | start the run through the launcher `arbol-<driver>-driver` found in a folder of the search path, for the route's `driver` and else for `claude` (TRV-D7), without `-v`, as `run --control --request` with the request file `run-1.json` in the Exchange's folder | New | unit, contract |
| TRV-0.4 | check nothing of the route itself, and end an Exchange whose Driver refuses the route as failed with the Driver's error, and one whose launcher is not found in a folder of the search path, a `driver` with a `/` in it included, as failed with `driver_failed`, naming the launcher | New | unit, contract |
| TRV-0.5 | write the run's request with `runId` `<exchange>.1`, `subscriptionId`, `model` and `effort` only as far as the route names them, the working folder as `workingDirectory`, `tools` `all`, the history, the message as `input` and the chat's identifier as `chatSessionId`, and with no other field | New | unit, live |
| TRV-0.6 | send as history every earlier Exchange of the chat that was not replaced, whatever its end, in the order of their numbers, each with its message, each block of its answer exactly as saved and each tool call the agent made itself with its result, in the order of their positions, in the form of DRV-2, leaving out the calls of sub-agents | Changed | unit, contract |
| TRV-0.7 | read that history in the transaction that accepts the message, after it has marked the Exchange that a message sent again replaces | New | unit |
| TRV-0.8 | make no small call and no classification, also when it is handed a small-call route, and write no title | Dropped | unit |
| TRV-0.9 | add nothing of its own to what the agent gets: no system prompt, no file read for the agent, no shortened history, and read no settings file of its own | Dropped | unit |
| TRV-0.10 | count the silence of RCT-2.21 as 16 minutes, the Driver's default idle limit of 15 minutes and one minute more, with `CLOCK_UPTIME_RAW` | New | unit, contract |
| TRV-0.11 | stop its run, whatever the reason, with a stop line on the run's control input, end the Driver's process with SIGKILL when it has not ended 16 seconds after that line or after its `finished`, whichever came first, counted with `CLOCK_UPTIME_RAW` and checked in each round of its loop, and end the Exchange only once the Driver's process has ended | New | unit |
| TRV-0.12 | be typed Python 3.14 on the interpreter its Build carries, use only the standard library and psycopg 3 at a pinned version, and share no code with another Trunk | New | unit |
| TRV-0.13 | be built as the component `arbol-trunk/trivial` from `trunks/trivial/` with the entry `bin/arbol-trunk-trivial` and `"rootContracts": [1]`, and count as working only when its unit checks, the Root Contract's suite for each version it lists, and one real message through the Claude Driver (TRV-D12) have passed | New | unit, live |

Beyond the Root Contract, these requirements carry what [ARC-0](../../../../arbol/architecture/00-components-and-boundaries.md#what-any-trunk-must-offer) leaves to each Trunk's own specification, "how a Trunk prepares context, whether it makes titles and how it retries": the context in TRV-0.5, TRV-0.6 and TRV-0.9, titles in TRV-0.8, retries in TRV-0.2. They also carry the parts of five accepted architecture requirements, which wait for these specifications, that the Root Contract leaves to the Trunk: of ARC-0.18, "with the route it was handed" (TRV-0.1, TRV-0.2); of ARC-0.24, what a Trunk does with a title (TRV-0.8); of ARC-2.10, that no other route is taken in any case (TRV-0.2); of ARC-3.4, which idle limit counts and how the run is ended (TRV-0.10, TRV-0.11); and of ARC-3.10, that time while the Mac sleeps counts towards none of these limits (TRV-0.10, TRV-0.11). ARC-0.3 is met by TRV-0.12.

## Interface

Proposed. The command, the files of the Exchange folder and the chat tables are the Root Contract's (RCT-0 to RCT-3). What the Trivial Trunk adds is the request of its one run:

```ts
// arbol-<driver>-driver run --control --request <the Exchange's folder>/run-1.json
interface TrivialRunRequest {          // a RunRequest of DRV-2, as the Trivial Trunk fills it; no other field
  runId: string;                       // "<exchange>.1", "2d4f6a8c-1b3e-4c5d-8e7f-9a0b1c2d3e4f.1"
  subscriptionId?: string;             // the route's subscription, only when it was handed, "anthropic"
  model?: string;                      // the route's model, only when it was handed, "sonnet"
  effort?: string;                     // the route's effort, only when it was handed, "high"
  workingDirectory: string;            // the working folder (RCT-0), "/Users/sam/repos/shop"
  history: HistoryItem[];              // DRV-2: the earlier Exchanges not replaced; [] for a first message
  input: { role: "user"; text: string };   // the Leather Bag's message
  chatSessionId: string;               // the chat's identifier, "c3d4e5f6-0718-4293-a4b5-c6d7e8f90a1b"
  tools: "all";
}
```

## Known problems in version 1

Not to be reproduced.

- **Several runs for one message.** Automatic repeats and a move to another route ran inside the turn, with waits held in memory and failures sorted by their wording.
- **A history without the tool trail.** The agent knew what it had said between messages, not what it had read or done.
- **Instructions and additions on the way to the agent.** A system prompt put together from files for every message failed the message when a template was missing, and an optional addition on the path from Send to the agent grew into thousands of lines.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Failures and what the Leather Bag sees`
- `arbol_deprecated_v1:product-spec/07-agent-context.md#What the agent sees of the conversation`
- `arbol_deprecated_v1:product-spec/07-agent-context.md#Known problems and lessons`

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| TRV-D1 | What does the Trivial Trunk do inside an Exchange? | One Driver run with the route it is handed, and nothing else: no small call, no title, no classification, no second run and no other route. It is the Trunk that still works when every other Trunk fails, so it carries as little logic as it can | Decided 15 October 2026 by the owner: "only simple driver call with provided parameters and that's it". That it makes no small call, title or classification and starts no second run is the technical consequence drawn the same day |
| TRV-D2 | Which earlier Exchanges does the history hold, and what of them? | Every earlier Exchange of the chat that was not replaced, whatever its end: its message, each block of its answer exactly as saved, and each tool call the agent made itself, with its result. It is the chat the Leather Bag sees, and the agent knows what it already did, also in an Exchange that failed half-way. The calls of sub-agents are left out, as the chat leaves them out. The alternative: only completed Exchanges, which hides from the agent files it has already changed | Decided 15 October 2026, a technical choice: the proposed default |
| TRV-D3 | When does the Trivial Trunk read the history? | In the transaction that accepts the message, after a message sent again has marked the Exchange it replaces. Once it has accepted, it needs the database only to save, so a database that goes away right after the acceptance fails nothing. The cost: the read falls within the 10 seconds of the acceptance, so a chat whose history takes longer to read has every message refused with `database_unavailable`, and the Draft stays. The alternative: read it after the acceptance, trying for 60 seconds (RCT-2), and fail the Exchange with `database_unavailable` when that does not work | Decided 15 October 2026, a technical choice: the proposed default |
| TRV-D4 | Does it let the Driver continue the vendor tool's own conversation, and under which name? | Yes, under the chat's identifier as `chatSessionId`. While the Subscription and the working folder stay the same, the vendor tool goes on with its own record of the conversation, the agent's thinking included. Whenever the history no longer matches, as after a message sent again, the Driver builds the conversation anew, so a replaced answer does not reach the agent (DRV-6). A chat migrated into another Root with the same identifier goes on with the same saved conversation only while its history matches. The alternative: no chat session, so that every run is built anew from the history, and the agent loses between messages what only the vendor tool's record holds, such as its thinking | Decided 15 October 2026, a technical choice: the proposed default |
| TRV-D5 | What does the request hold beyond the route, the working folder, the history and the message? | Tools on (`all`) and the run identifier `<exchange>.1`. No system prompt, no limits, no labels, no further folders and no search path: the vendor tool's own prompt and the instructions it finds in the working folder reach the agent anyway, and the Driver's idle limit of 15 minutes applies. The alternative: a system prompt of the Trunk's own, such as one that names the Root and the folder, which is text to keep and change, and which makes the Codex Driver build its conversation anew whenever it changes (DRV-6) | Decided 15 October 2026, a technical choice: the proposed default |
| TRV-D6 | Does the Trivial Trunk check the route before it accepts the message? | No. It hands the route to the Driver, which refuses what it cannot use before anything is sent (DRV-2), and the Exchange fails with the Driver's error. The route's `driver` names the launcher `arbol-<driver>-driver`, so a new Driver needs no change of the Trivial Trunk; a launcher that is not on the search path fails the Exchange with `driver_failed`. The alternative: check the route at acceptance and refuse it, keeping the Draft, which needs a list of Drivers, Subscriptions and models inside the Trunk | Decided 15 October 2026, a technical choice: the proposed default |
| TRV-D7 | Started by hand without a Driver, as in `arbol-trunk run "What does build.sh do?"`, which Driver does the Trivial Trunk use? | The Claude Driver, with no Subscription, model or effort named, so that the Driver takes its own defaults: the Subscription its settings file names, else `anthropic`, and the model its settings file names, else the vendor tool's own (DRV-0). It is the Driver of the live check (TRV-D12) and of the Root Contract's examples. Arbol always hands a whole route, so this choice matters only by hand. Its agent may ask the Leather Bag a question (DRV-D25); by hand, the question is answered in Arbol's window when Arbol shows the chat, and otherwise the run waits until Ctrl-C, because a question has no time limit (DRV-3). The alternative: the Codex Driver, whose default Subscription is `chatgpt` (CDX-1), and whose agent never asks, so a run by hand never waits for the person | Open |
| TRV-D8 | How does the Trivial Trunk stop its run, and what if the Driver does not end? | In one way for every reason: a stop line on the run's control input. A Driver that has not ended 16 seconds later, one second more than DRV-2 gives a stop, so that a Driver using its whole time is not killed just as it finishes, is ended with SIGKILL, and what it left running its next call ends (DRV-6). The Exchange ends once the Driver's process has ended, still within the 18 seconds of RCT-2. The alternative: SIGTERM first and SIGKILL 5 seconds later, which asks a Driver that did not stop for the same stop again and waits 5 seconds more | Decided 15 October 2026, a technical choice: the proposed default |
| TRV-D9 | How is the Trivial Trunk's work laid out, so that a slow database holds up neither a stop nor the heartbeat? | Two threads. One `asyncio` loop does all the work: the Driver's output and control input, `events.jsonl`, `control`, the times of a stop and the heartbeat. It saves rows in a task beside it, on psycopg's asynchronous connection with a limit of 5 seconds for each call, and never waits for them, so `control` is read every second as RCT-1 asks, also while the database is slow. The watch of RCT-1 is the second thread and does nothing else. The Claude Driver is built on `asyncio` too. The alternatives: a loop that waits for each call to the database, in which a stop waits up to 5 seconds behind a call and can then take some 22 seconds, past the 18 of RCT-2 and the 20 after which `arbol-root stop` ends the Exchange by force and it is shown as interrupted (ARC-D60); or a third thread for the database, whose calls psycopg's ordinary connection cannot cut off when the server stops answering | Decided 15 October 2026, a technical choice: the proposed default |
| TRV-D10 | What may its program use? | Typed Python 3.14 on the interpreter the Build carries (ARC-0.3), the standard library, and psycopg 3 at a pinned version, in the form that brings its own PostgreSQL client library; nothing else. Every further library is one more part that can break the Trunk that is there when the others break. The alternative: libraries as they are useful, such as one for the command line | Decided 15 October 2026 by the owner as to the principle: each program writes its own plain SQL (RCT-D22), and the Trivial Trunk is minimal. Psycopg 3, pinned, is the technical choice of RCT-D22, and the details are the proposed default, a technical choice made the same day |
| TRV-D11 | What may be added to the Trivial Trunk later? | Only what a version of the Root Contract that a Root uses, a change of the Driver Contract or a defect requires. Anything else, however small, such as a title, a system prompt, a retry, a second run, a setting of its own or another library, goes into another Trunk. The alternative: features added when they are useful, which in time makes the fallback as complex as what it stands in for | Decided 15 October 2026 by the owner as to the principle: the Trivial Trunk is the most stable of all Trunks, there so that Arbol keeps working when every other Trunk fails, minimal and with little logic. The rule is the technical consequence drawn the same day |
| TRV-D12 | Does a Build of the Trivial Trunk count as working only after one real message has gone through it? | Yes. One message, "Answer with the single word: ready", with tools on, through the Claude Driver on its test Subscription `test-anthropic` and the model `haiku`. It costs on every build about 18,500 input tokens from the test account's usage window, which the vendor prices at about 4 US cents, five times a Driver's own build message without tools (BLD-0), and more when the owner's skills and MCP servers come with the run. It is the only proof that a real Driver takes the request the Trivial Trunk writes: the stand-in of RCT-4 takes whatever the Driver Contract allows. A build while the test Subscription is signed out or used up fails, and the last working Build stays in use. The alternative: the unit checks and the suite only, and a real message sent by hand when wanted | Open |
