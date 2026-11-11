---
role: authored
---
# Justifications

## <before program_policy>

Reason: TRV-D12: the Build's live check runs, asserts and reports an explicitly selected Subscription, defaulting to test-anthropic

Origins: TRV-D12, TRV-0.13, BLD-0.5 · Intention i-f96f0464 · 2026-11-11

## live_fixture

Reason: TRV-0.13: the Build's live check uses the shared arbol-root checkout, since the readonly clone is retired

Origins: TRV-0.13, BLD-0.5 · Intention i-a59a8efd · 2026-11-11

## live

Reason: TRV-D12: the Build's live check runs, asserts and reports an explicitly selected Subscription, defaulting to test-anthropic

Origins: TRV-D12, TRV-0.13, BLD-0.5 · Intention i-f96f0464 · 2026-11-11

## driver_bridge

Reason: TRV-0.13: the live check's Driver bridge passes USER, which claude needs to find its Keychain login

Origins: TRV-0.13, BLD-0.5 · Intention i-aa61dfe9 · 2026-11-11

