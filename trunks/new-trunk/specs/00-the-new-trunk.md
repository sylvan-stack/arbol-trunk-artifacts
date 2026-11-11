---
role: authored
status: draft
---
# New Trunk · 00 The communication strategy

**Temporary placeholder.** `new-trunk` is a working directory and title, not a proposed code name. The owner has not yet chosen this Trunk's name or communication strategy. Rename this chapter and assign its requirement prefix after those choices; this draft is not ready for acceptance or implementation.

This will be the standalone specification of one [Arbol Trunk](../../../../arbol/GLOSSARY.md#arbol-trunk). It will state the communication strategy the Trunk implements and the behavior the [Root Contract](../../../../arbol-root/contract/README.md) leaves to each Trunk. It follows the conventions of the [Driver Contract](../../../../arbol/driver-contract/README.md#conventions) and the additions used by the existing [Trivial Trunk specification](../../trivial/specs/00-the-trivial-trunk.md).

## Scope

Covered when written: the Trunk's communication strategy, its work inside one Chat Exchange, its Driver calls and conversation handling, its program and Build, its Root-specific choices, and its checks.

Not covered: the shared Root Contract, the common `arbol-trunk` command and Exchange lifecycle, Driver internals, Root selection, and Builds and their registry. Those remain in their existing specifications and are not duplicated here.

## Open decisions

### What communication strategy does the Trunk implement?

No strategy has been selected. The specification must first say what changes for the owner and what the Trunk does inside an Exchange. It must then settle the number and purpose of Driver calls, conversation and history handling, stopping and settling behavior beyond the shared contract, and which existing components it reuses. Requirements and implementation Beats wait for those decisions.

### What is its code name?

Naming should describe the communication strategy rather than create another Root or a tree of environments. These are options, not assigned names:

| Option | Fits a Trunk that… |
|---|---|
| `dialogue` | conducts a multi-turn conversational Exchange; recommended for an ordinary conversational Trunk |
| `research` | gathers and evaluates sources before synthesizing an answer |
| `review` | inspects work and returns structured findings |
| `relay` | performs bounded handoffs between stages or participants |
| `banyan` | uses a broad or branching strategy and prefers a botanical theme over a functional description |

The final code name, display name and requirement prefix remain open. The owner may choose one of these or another name.

## Requirements

Not yet written. This section will contain testable requirements only after the communication strategy is decided.

## Interface

Not yet written. It will contain the Trunk-specific request, event, state and Build interfaces without repeating the shared Root Contract.

## Decisions

The owner's communication-strategy and naming decisions will be recorded here with the decisions that follow from them.
