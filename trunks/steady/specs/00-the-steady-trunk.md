---
role: authored
spec: STE-0
status: draft
---
# Steady Trunk · 00 The communication strategy

The **Steady Trunk**, code name `steady`, is the owner's chosen name for a new [Arbol Trunk](../../../../arbol/GLOSSARY.md#arbol-trunk). Its communication strategy is not yet decided, so this draft is not ready for acceptance or implementation.

This will be the standalone specification of the Steady Trunk. It will state the communication strategy the Trunk implements and the behavior the [Root Contract](../../../../arbol-root/contract/README.md) leaves to each Trunk. It follows the conventions of the [Driver Contract](../../../../arbol/driver-contract/README.md#conventions) and the additions used by the existing [Trivial Trunk specification](../../trivial/specs/00-the-trivial-trunk.md).

## Scope

Covered when written: the Trunk's communication strategy, its work inside one Chat Exchange, its Driver calls and conversation handling, its program and Build, its Root-specific choices, and its checks.

Not covered: the shared Root Contract, the common `arbol-trunk` command and Exchange lifecycle, Driver internals, Root selection, and Builds and their registry. Those remain in their existing specifications and are not duplicated here.

## Open decisions

### What communication strategy does the Steady Trunk implement?

No strategy has been selected. The specification must first say what changes for the owner and what the Steady Trunk does inside an Exchange. It must then settle the number and purpose of Driver calls, conversation and history handling, stopping and settling behavior beyond the shared contract, and which existing components it reuses. Requirements and implementation Beats wait for those decisions.

## Requirements

Not yet written. This section will contain testable requirements only after the communication strategy is decided.

## Interface

Not yet written. It will contain the Steady Trunk's request, event, state and Build interfaces without repeating the shared Root Contract.

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| STE-D1 | What are the display name and code name of the new Trunk? | Steady Trunk, code name `steady`; requirement prefix `STE` | Decided by the owner on 11 November 2026: “name it "Steady Trunk"”; the lowercase code name follows the Trunk naming convention |
