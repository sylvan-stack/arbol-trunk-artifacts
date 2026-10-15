---
role: authored
---
# Trivial Trunk specifications

What the Trivial Trunk does: for each [Chat Exchange](../../../../arbol/GLOSSARY.md#chat-exchange) it makes one run of a Driver with the route it is handed, and nothing else. Its code name is `trivial`. It is an [Arbol Trunk](../../../../arbol/GLOSSARY.md#arbol-trunk), the default Trunk of every Root, and the most stable of all Trunks, there so that Arbol keeps working when every other Trunk fails. For example, `arbol-trunk run "What does build.sh do?"` starts it in the current Root, because the command names no other Trunk.

It implements version 1 of the [Root Contract](../../../../arbol-root/contract/README.md) entirely, with `"rootContracts": [1]`. That contract says what every Trunk does and what the Root gives it, and it is not repeated here. The specification here says only what the contract leaves to each Trunk: the one run, the conversation the agent is sent, the request to the Driver, what the Trivial Trunk never does, and its program, Build and checks.

## Specifications

| Spec | Title | Status |
|---|---|---|
| [TRV-0](00-the-trivial-trunk.md) | The Trivial Trunk | review |

## Conventions

Conventions (shape of a specification, requirement identifiers, origin, checks, status, decisions) are defined once in the [Driver Contract's README](../../../../arbol/driver-contract/README.md#conventions), with the additions of the [Root Contract's README](../../../../arbol-root/contract/README.md#conventions): the checks *journey* and *drill*, and *contract*, which means the Root Contract's conformance suite ([RCT-4](../../../../arbol-root/contract/04-conformance.md)). Identifiers here start with `TRV`: `TRV-0.3` is requirement 3 of specification 0. Decisions are numbered once for this folder, `TRV-D1` onwards.
