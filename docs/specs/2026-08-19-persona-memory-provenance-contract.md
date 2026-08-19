# Persona and Memory Provenance Contract

**Status:** Proposed contribution for [#4253](https://github.com/tinyhumansai/openhuman/issues/4253)

**Scope:** Documentation and design contract for the first vertical slice of the guided persona builder and memory dashboard.

## 1. Purpose

OpenHuman already exposes memory and agent-profile capabilities to the runtime, but a non-technical user needs a clear answer to three questions before trusting those capabilities: what the system believes, where each statement came from, and how to correct or retire it. This document defines a small, implementation-neutral contract for presenting persona and memory state without creating a second UI-only source of truth.

The contract is intentionally narrower than the full #4253 feature. It does not prescribe a complete dashboard, template catalog, or storage migration. It defines the state and provenance semantics that those surfaces should share.

## 2. Source-of-truth rule

The runtime profile and memory stores remain authoritative. The React UI may edit and display these records, but it must not persist a parallel persona or memory model in browser-only storage. A successful UI mutation is complete only after the runtime source of truth acknowledges it and the UI refreshes from that source.

The flow is therefore:

```text
User action → UI validation → runtime RPC → authoritative store → refreshed view
```

A failed or unconfirmed runtime mutation must not be presented as saved. Optimistic UI may show a pending state, but it must be discarded when the RPC fails or the refreshed record does not contain the requested change.

## 3. Memory record contract

The dashboard should render each memory record with the following semantic fields, whether they are returned directly by the current backend or introduced through an adapter at the UI boundary:

| Field | Meaning | User-facing treatment |
| --- | --- | --- |
| `statement` | The retained fact, preference, goal, or observation. | Primary readable content. |
| `source` | How the record entered the system. | Always visible in a compact provenance label. |
| `confidence` | A bounded estimate of record reliability. | Display as qualitative bands, not false precision. |
| `state` | Lifecycle or epistemic status. | Distinguish active, provisional, needs-confirmation, and retired. |
| `projectContext` | The work or personal context to which the record belongs. | Show the active context and prevent silent cross-context retrieval. |
| `createdAt` / `lastConfirmedAt` | Record age and most recent confirmation. | Use for freshness and review prompts. |
| `evidenceRefs` | References to the interaction, import, or approved output that supports the record. | Link or disclose when available; never expose secrets. |

The initial source vocabulary should preserve the distinction between explicit user statements, approved outputs, interaction inferences, and imported documents. If the runtime has a different enum, the adapter should map it losslessly and keep an unknown value visible as `Other` rather than silently classifying it.

## 4. State transitions

Memory lifecycle and epistemic status should be explicit. The minimum supported transitions are:

```text
provisional → active
provisional → needs-confirmation
active → needs-confirmation
active → retired
needs-confirmation → active
needs-confirmation → retired
```

A user correction must create an auditable replacement or correction event rather than silently mutating the displayed text. Retiring a record must remove it from default active retrieval while preserving its history for contradiction review and debugging. A newer explicit user statement outranks an older inference, but the older inference should remain traceable as superseded rather than disappearing without explanation.

## 5. Persona sections

The guided persona editor should map to structured sections that the runtime already understands or can adopt incrementally:

| Section | Examples |
| --- | --- |
| Professional identity | Name, roles, industries, languages, locations, public positioning. |
| Communication preferences | Tone, structure, audience, verbosity, languages, anti-preferences. |
| Active projects | Project name, context, goals, constraints, relevant tools, current status. |
| Working principles | Approval requirements, safety boundaries, evidence standards, review gates. |
| Learning profile | Provisional preferences and interaction patterns awaiting confirmation. |

Symbolic astrology or numerology inputs, if supported by a future profile layer, should be stored as reflective context with explicit provenance and uncertainty. They must not be written as deterministic personality facts or allowed to outrank explicit user instructions and real business constraints.

## 6. Project isolation

Every memory search and persona view should carry an explicit context selector. The default must be the current project or workspace context, not a global union of all memories. A global search should require an intentional user action and should disclose that it crosses contexts.

At minimum, the UI should make the following states distinguishable:

1. The record belongs to the active project.
2. The record belongs to another known project and is excluded by default.
3. The record has no project context and is available only to explicitly global flows.
4. The record is restricted and must not be rendered in a broader context.

The implementation should reuse the runtime’s existing project or source-scope mechanisms rather than inventing a browser-only filter. A project selector that changes only the visible list but not the RPC query is not sufficient.

## 7. Minimum vertical slice

A low-risk first implementation can deliver the following without attempting the entire epic:

1. A read-only memory list that renders statement, provenance, state, confidence band, freshness, and project context.
2. A project-context filter that is passed through to the authoritative memory query.
3. A confirmation action for provisional or needs-confirmation records.
4. A correction action that writes through the runtime source of truth and refreshes the record.
5. Tests covering explicit user statements, approved outputs, interaction inferences, imported documents, cross-project exclusion, correction failure, and retirement.

The persona editor can then reuse the same provenance and confirmation components for structured profile sections.

## 8. Safety and privacy requirements

The dashboard must not expose passwords, access tokens, authentication codes, payment information, or unnecessary personal data about third parties. Logs and analytics must not include user-authored memory text, evidence contents, credentials, or raw provider envelopes. Restricted records should be redacted or omitted according to the runtime policy, not hidden only with CSS.

External write actions remain approval-controlled. A correction or retirement initiated by the user is a direct user-approved mutation; an agent-generated suggestion to correct or retire a record must stop at a review state until the user confirms it.

## 9. Verification plan

The implementation should add focused tests at the smallest affected layers. Runtime tests should verify that project scope is applied to the query and that correction or retirement failures do not produce a false success. UI tests should verify provenance labels, state transitions, context filtering, and the error state after a rejected mutation. A manual smoke path should create a provisional record, confirm it, correct it, switch project context, and verify that unrelated records do not appear by default.

This contract should be revised when the runtime RPC schemas are finalized. Until then, adapters should prefer explicit `Other` or `Unknown` values over lossy inference.
