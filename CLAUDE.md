# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository nature

Documentation-only repo. There is no source code, no build system, no test runner. All artifacts are Chinese-language Markdown design specs. "Tasks" here mean editing specs and keeping them internally consistent.

Working language is **Chinese** — match the existing tone and terminology when editing or adding docs. Code blocks, schema field names, and protocol identifiers stay in English.

## Two parallel specifications

This repo defines two layered specs that must stay aligned:

```
FAP-1   (Future Agent Protocol 1)        →  设计文档-初版/01-协议设计/docs/fap/
FAP-ME  (FAP Memory Engine, FME-1)       →  设计文档-初版/02-记忆系统/docs/fme/
```

FAP-ME is the **official memory extension of FAP-1**. It does not redefine FAP-1's control plane, Mandate, or Receipt — it binds to them. When something looks like it belongs to both layers, FAP-1 owns the protocol contract; FAP-ME owns cognition (layering / retrieval / forgetting / dream).

Each spec directory has a `README.md` index and a numbered chapter series (`00-overview.md`, `01-…`, …) plus topical specs (`kernel-contract.md`, `fap1-binding.md`, etc.). Start from the README to find the right file.

## Two principles that override everything

These come from the user's memory and are non-negotiable:

1. **Pluggable architecture (FAP-1)** — every extension point must be a plugin; Core only defines contracts.
2. **Kernel Contract (FAP-1)** — Core must define the Kernel Contract; plugins only provide algorithms or adapters, **never** participate in security decisions.

The Kernel Contract is enumerated in two places that must stay in sync:
- `设计文档-初版/01-协议设计/docs/fap/kernel-contract.md` (FAP-1 side)
- `设计文档-初版/02-记忆系统/docs/fme/kernel-contract.md` (FAP-ME side)

Concretely, this means edits MUST NOT:

- Move security judgement (Policy / Audit / Tenant isolation / Mandate verification / ContentSafety / Replay) into a plugin trait.
- Let plugins generate Receipts, write hash-chain tips, or arbitrate their own permissions.
- Reorder the fixed Kernel decision sequence (`Transport → Envelope → Header → Replay → Session → AuthN → AuthZ → Mandate → Capability → Risk → Plugin sandbox → Receipt`).

When unsure whether something is "Kernel" or "Plugin", default to Kernel and flag it.

## Currently active revision

`task_plan.md`, `progress.md`, and `findings.md` track an in-flight revision of the FAP-ME spec to align it with FAP-1. Read these before touching `docs/fme/` — they record decisions you must respect:

- FME does **not** define its own Mandate schema. It reuses FAP-1 Mandate and expresses memory-specific rules via signed `constraints` (`memory.purpose`, `memory.allowed_triples`, `memory.sbu_access`).
- Purpose authorization uses a structured `CapabilityTriple` (`capability + layer + action`) with **exact match only**. Do not reintroduce `allowed_ops` strings, contains/prefix/suffix matching.
- HardForget is split: local SQLite/PG/index/CAS in one transaction (strong consistency) + outbox + idempotent `mutation_ledger` + reconciler for S3/Kafka/Qdrant/remote sinks (eventual). `ForgetReceipt` must distinguish `COMMITTED_LOCALLY` / `GLOBALLY_RECONCILED` / `RECONCILE_FAILED`.
- `RedactionReport` proves "executed under declared policy + classifier_version", not "no SBU residue".
- Cross-tenant / cross-org sharing requires `sbu_manifest`. Within-tenant low-risk sharing is policy-optional.
- `CoreMemory` is **not** a separate layer — it is an L2 protected retention tier.
- Third-party plugins default to WASM/sidecar. `rust_native` is restricted to `builtin/trusted`.

The two anchors for cross-cutting consistency in FAP-ME are:
- `fap1-binding.md` — protocol binding (capability / Mandate constraints / Receipt)
- `outbox-reconciliation.md` — cross-system consistency (HardForget outbox / mutation ledger / reconciler)
- `signing-canonical.md` — single source of truth for canonical encoding of every signed object (AuditEvent / ForgetReceipt / RedactionReport / ContextGrant / HandoffPacket / DreamProposal). Field order, hash inputs, enum integers, and `capability_id ↔ CapabilityTriple` mappings live here. Any other document showing a different field order is wrong.
- `chain-state-concurrency.md` — `chain_state` CAS model, isolation-mode placement, and the `chain_state_registry` used by `ChainIntegrityScheduler`.

## Working conventions

- Existing chapter numbering (`NN-name.md`) and the README indices are load-bearing. Don't renumber or rename without updating every cross-reference.
- Cross-spec references use relative paths (e.g. `../../../01-协议设计/docs/fap/kernel-contract.md`). Verify links when moving files.
- Top-level files outside `设计文档-初版/` (e.g. `future-agent协议设计.md`, `下一代vcp协议.md`, the numbered `01.…md` / `02.…md`) are **research / earlier drafts**. The authoritative specs are under `设计文档-初版/01-协议设计/docs/fap/` and `设计文档-初版/02-记忆系统/docs/fme/`. Don't propagate edits from drafts into the spec directories without checking they're still desired.
- Before claiming a doc revision is done, run a ripgrep scan for residual old field names / old schema (the prior task tracker did this — see `progress.md`).

## Git

The repo uses Chinese commit messages (e.g. `添加记忆系统设计文档`, `feat: 添加FAP设计文档`). Match the existing style when committing.
