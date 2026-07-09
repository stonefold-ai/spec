# Stele — Specification v0.6

*The policy language for the SIF gateway: the declarative file that decides, deterministically, what an AI agent is permitted to do, and what is recorded when it tries.*

> **Layering.** Stele is the upper layer. The lower layer — **what the agent can express** (the five action kinds and the intent shape) — is defined in the **SIF RFC** ([`00-RFC-sif-intent-format.md`](00-RFC-sif-intent-format.md)). Stele references SIF for the kinds and the operation shape; it does not redefine them. SIF = *what can be said*; Stele = *what is allowed*; the v0.6 `requireMatch` gate adds the third clause — *what is owed* (§7.16).

**Status:** v0.6 — change set v0.5 → v0.6 **ACCEPTED** 2026-07-09 ([`docs/RFC-changeset-v0.5-to-v0.6.md`](RFC-changeset-v0.5-to-v0.6.md)): the reference implements every item (CS-026…CS-039) and the TCK certifies the four new profiles (`hold-precondition`, `feedback`, `match`, `consume`), in-process and over the wire binding. **Authors:** the agent-platform team.
**Audience:** engineers implementing or writing policies, and reviewers (security, compliance) who must read and certify them.

> **Compatibility:** v0.6 is **additive**: no existing key changes meaning, and existing `apiVersion: stele/v0.1` policy files remain valid as-is. It contains **one deliberate, argued amendment to the frozen shape** — the gate catalog grows from fourteen to fifteen (`requireMatch`, CS-032) — and deliberately does **not** touch the condition grammar (§8): the tolerance comparison is a structured gate field (CS-033), not a new operator. `schema/stele.schema.json` gains optional keys only. Three behavioural deltas an upgrading deployment sees (all intended): the agent-facing feedback default becomes `code+fields` (CS-030), held rows expire actively (CS-028), and duplicate holds collapse (CS-031). v0.5 was additive declarations, semantic completions, and text fixes (digest pins CS-020, batch semantics CS-023, classification order CS-024). Deltas: v0.1 → v0.2 is `docs/RFC-changeset-v0.1-to-v0.2.md`; v0.2 → v0.3 is `docs/RFC-changeset-v0.2-to-v0.3.md`; v0.3 → v0.4 is `docs/RFC-changeset-v0.3-to-v0.4.md`; v0.4 → v0.5 is `docs/RFC-changeset-v0.4-to-v0.5.md`; v0.5 → v0.6 is `docs/RFC-changeset-v0.5-to-v0.6.md`.

## Changelog — v0.5 → v0.6

| ID | Type | §  | Summary |
|----|------|----|---------|
| CS-026 | ADDED | §7.6, §12 | **Preconditions may resolve `hold`.** A named check resolves pass / fail / hold; hold reuses the held lifecycle (no new state). Outages/crashes MUST fail, never hold — hold is only for successfully-read, judgment-shaped ambiguity; a hold without a machine-readable reason code is a check implementation error (resolves fail, logged). Opt-in per check (declared `holdCapable`, registry §5); two-valued checks stay valid. |
| CS-027 | ADDED | §12, §4.4 | **Multi-hold release.** A held row carries the release contract of **every** holding gate; promotion requires each satisfied. Non-approval holds name a `resolvers:` role (identity seam) or fall back to the deployment's default resolver role; a hold with no resolvable contract is refused fail-closed (`hold-unresolvable`), never staged open. Also binds composed approval gates (approval + dual-auth holding together each bind). |
| CS-028 | ADDED | §7.8, §12 | **Held-row expiry enforced.** A held row expires at the earlier of the holding gate's `timeout` and the staging TTL (CS-017), via an active sweep: settles `CANCELLED`/`expired-hold:<gate>` (normative reason), original hold reason code preserved. `onTimeout: allow` satisfies its own contract only. Retires the previously-unenforced `timeout`/`onTimeout` declarations. |
| CS-029 | ADDED | §7, §11 | **Reason codes carry a retry class** — `retryable` \| `terminal` \| `escalate` — returned to the agent with every deny/hold. Undeclared class defaults `terminal`. Registry precondition declarations gain an object form (`name`, `holdCapable`, `reasonCodes`); built-in gate reasons get normative classes (§11). |
| CS-030 | ADDED | §11 | **Agent feedback visibility.** Per gate/check: `code` \| `code+fields` (default) \| `code+evidence` governs what the agent receives on deny/hold; the audit record always carries everything (redact on return, never on write). Deny-rate + reason-code distribution per principal SHOULD be surfaced (probing detection). |
| CS-031 | ADDED | §12 | **Hold dedupe.** Same (agent, action, reason code, candidate refs) within the deployment's dedupe window collapses into one held row with an attempt count; every attempt still audited. Per-principal open-hold budget deferred to v0.6.1. |
| CS-032 | ADDED | §7.16 | **`requireMatch` gate** (gate 15 — the argued frozen-shape amendment). An intent MUST correspond to exactly one open obligation in a declared obligation registry, within declared tolerance; reserved at staging, consumed at settlement. Three-valued: match / no-match / ambiguous; ambiguous never auto-resolves (`onAmbiguous: allow` is illegal). |
| CS-033 | ADDED | §7.16 | **Tolerance clauses are structured fields**, legal only inside `requireMatch.match` (`{ field, matches, within }`); the §8 condition grammar is unchanged. |
| CS-034 | ADDED | registry (docs/06 §5b) | **Obligation registries.** A named, typed, digest-pinnable (CS-020) declaration class; adapter contract `query`/`reserve`/`consume`/`release`, idempotent per (ref, intent id); declared consistency capability `transactional` \| `window` (CS-018 semantics). TCB member; the governed agent MUST NOT hold write access to a registry it matches against. |
| CS-035 | ADDED | §12 | **Reservation lifecycle.** Reserve inside the staging commit; liveness-check inside the dispatch claim; consume with the settle (transactional: same transaction as effect + CS-006 audit write; window: post-confirm, residual surfaced); release on any terminal non-success. Reservations carry their own TTL ≥ the row's decision TTL (orphan recovery); batch reservations are all-or-nothing (CS-023). |
| CS-036 | ADDED | §7.16, §12 | **Agent-independent read path.** `obligation.*` operands resolve from the registry's response only; an intent MAY carry an obligation pointer, which narrows the query but never substitutes for it. |
| CS-037 | ADDED | §11 | **Audit extension.** `obligationRefs` (entitlement-side lineage, complementing `resultRefs`) and `consumption` (`reserved` \| `consumed` \| `released` + receipt id, + residual for window registries). |
| CS-038 | ADDED | §13 | **Linter rules 14–18** (registry/path/tolerance typing; creation/execution separation; `consume: none` on irreversible ⇒ warn; `onAmbiguous: allow` ⇒ error; hold-capability declarations). Rule 4 amended: `requireMatch` counts as a satisfying guard. |
| CS-039 | DOCS | §14, §1 | Worked examples: §14.4 `pay` matches and consumes a purchase-order line; §14.2 `administer` matches a prescription schedule slot. Non-goals block added to §1 (not a system of record; entitlement ≠ judgment; the human is the registry where none can exist). |

## Changelog — v0.4 → v0.5

| ID | Type | §  | Summary |
|----|------|----|---------|
| CS-019 | ADDED | §1 | **Trust boundary stated.** The gateway proves *intents conform to policy*; it does not prove the executing code does what it declares. Connectors, hooks, and the gateway are the trusted computing base; their integrity is a supply-chain property. Text only; non-normative discussion in docs/13. |
| CS-020 | ADDED | registry §5; §10 | **Connector digest pinning.** A connector declaration MAY pin its implementing artifact by `sha256` digest; when declared, the gateway MUST verify at policy load and at dispatch — mismatch is a dependency failure under §10 (fail closed, audited). Additive; existing registries unaffected. Reference implementation shipped (`stonefold_core.digest`). |
| CS-021 | ADDED | arch. decision 11 | **Identity-provider seam.** The session's authenticated `actor:`/`agent:` identities come from an `IdentityProvider` protocol ahead of the pipeline; built-in default is the existing session/transport auth (no behavioural change). No credential scheme integrated or endorsed. Invariant 3 binds every provider. Reference implementation shipped (`stonefold_gateway.identity`). |
| CS-022 | FIXED | §9 | **Kill wording reconciled with the two axes.** The operator hard-kill is unconditional — a policy cannot opt out; `killable` is a *manner-of-stopping* declaration that guards automated halts and informs, but never blocks, the operator. Replaces §9's opening and retires its UNDER-REVIEW note. Text only; the graceful-halt wiring stays deferred (docs/03). |
| CS-023 | ADDED | §12; SIF §5 | **Batch decision semantics.** A SIF batch is decided atomically: every operation is decided first (each with its own audit record); any DENY/HALT refuses the whole batch before anything commits or stages; a HOLD stages per §4.4 and does not refuse the batch (committed `record` ops are not rolled back by a later reject/expiry). Reference implementation shipped (`enforce_batch`); TCK `batch` profile certifies it. |
| CS-024 | CLARIFIED | §7.12; registry §4 | **Classification ordering.** `disclosure.maxClassification` compares by the classification set's **declared order**; the built-in `resultSensitivity` values are ordered `public < internal < confidential < restricted`; a domain substituting its own labels MUST declare them as an ordered value set in the registry. A value missing from the declared order fails closed (§8). |
| CS-025 | DOCS | §6.2, §6.3, §7, §13 | Editorial/clarification batch: §6.2 rule 4 reworded (gates bind by their §7 keys; all matching gates AND — allow-match specificity does not select gates); CS-018's capability home named (gateway-code connector metadata, not registry YAML); `spendLimit` unit defined as gateway configuration; approver `role:` names resolve at the identity seam (§13 rule 1 exemption); `window` absolute `from`/`to` form documented; gate-table row 13 corrected to pass/fail/hold; catalog approval examples re-keyed on stakes per the §5 note; CS-020/021 changelog status corrected to shipped. |

## Changelog — v0.3 → v0.4

| ID | Type | §  | Summary |
|----|------|----|---------|
| CS-017 | ADDED | §12, §4.4 | **Decision freshness.** Every staged effect carries a finite decision **TTL** stamped at staging from gateway configuration (never policy syntax; short for `irreversible` effects); a row claimed past it settles `CANCELLED`/`stale-decision`, and a late approval does not resurrect it. Inside the dispatch claim — after the §9 kill re-check, before the connector call — the gateway re-validates the **volatile** gates (`allowlist`/`denylist`, `window`, `precondition`, `emissionControl`); a failure settles `CANCELLED`/`stale-guard:<gate>`, audited, never a partial dispatch. Non-volatile gates (counters, `valueLimit`/`contentCheck`, approvals) are **not** re-run. |
| CS-018 | ADDED | §6.3 | **Scope no-race.** Connectors declare a scope-reassertion capability, `transactional` \| `window`. A transactional connector re-asserts the scope predicate **inside the effect's own transaction** (zero rows affected ⇒ `FAILED`/`scope-lost`, audited — the write lands on authorized state or not at all). A window connector's target is re-resolved under scope immediately before dispatch, and its declared residual window is surfaced in the audit record. |

## Changelog — v0.2 → v0.3

| ID | Type | §  | Summary |
|----|------|----|---------|
| CS-010 | FIXED | §7.15, §14.3, §13 | `standing` cannot re-enable an explicit `deny` — deny always wins (§6.2). §14.3 wrongly listed `engage` under `deny` while a standing rule enabled it; corrected to rely on **default**-deny. New lint rule 11: an action in both `deny` and a `standing` rule's `enables` ⇒ **error** (the grant is unsatisfiable). |
| CS-011 | FIXED | §7.13 | `emissionControl` example syntax corrected to `{ checks: [...] }` — the previous `{ precondition: [...] }` spelling did not validate against `schema/stele.schema.json` (the fixtures already used `checks`). Also clarified when the gate resolves `hold` vs `fail`. |
| CS-012 | CLARIFIED | §6.1, §13 | Bare-name grant resolution defined: a bare token under a kind matches the **resource** of that name (all of that kind's actions on it) or **any declared action of that kind with that name**, on every resource that declares it. New lint rule 12: a bare action name in `allow` that resolves on more than one resource ⇒ **warn** (use the map form). A bare-name `deny` deliberately matches them all. |
| CS-013 | CHANGED | §8 | Grammar amendment: the right side of `in` / `not in` MAY be a function (e.g. `context.time in window("08:00-18:00")`), and string literals may be single- or double-quoted — legalising the form §7.15's example already used. No other operator/function change. |
| CS-014 | ADDED | §13 | New lint rule 13: `dualAuthorization` with an explicit `quorum` < 2 ⇒ **error** (contradicts the gate's definition, §7.9). |
| CS-015 | DOCS | — | Editorial: section numbering repaired (file structure is §3, kinds are §4; the `standing` row now points at §7.15); §4.3 lists all five `record` built-ins; §7 names the `Resource.action` gate-key form; §7.1's window note fixed; §14.1 gains `quota`/`spendLimit` and §14.5 `window`, so the worked examples now genuinely cover the full gate catalog; §14.3 aligned with its fixture. |
| CS-016 | CLARIFIED | §13 | Rule 1 (every referenced name exists) applies to **`deny` too** — the Registry spec's former "exception for `deny`" (doc 06 §8, undeclared names allowed in `deny`) is removed. A deny of an unknown name is a security no-op (default-deny already refuses unknowns) and almost always a typo; to pre-forbid a capability, **declare** the action in the registry and deny it. |

## Changelog — v0.1 → v0.2

| ID | Type | §  | Summary |
|----|------|----|---------|
| CS-001 | CLARIFIED | §6.3 | Scope means two things: a **filter** for reads/writes, a **pre-resolution authorization check** for effects. |
| CS-002 | CLARIFIED | §7.12 | `disclosure` has a **pre-check** (block before execute, sensitivity known from registry) and a **post-check** (withhold on return, row-dependent) form. |
| CS-003 | CHANGED | §4.4 | Effects are **staged (accepted/pending) by default**; inline execution is opt-in for cancellable effects only. Staging is the substrate for approvals and kill. (`SHOULD` → `MUST`.) |
| CS-004 | ADDED | §9 | Kill **no-race guarantee**: dispatch-time kill check and the `pending → dispatching` transition occur in one serialised transaction; three check points; idempotency key; explicit guarantee scope (prevents new/un-dispatched, cancels cancellable in-flight, compensates declared irreversibles, does **not** reverse committed effects). |
| CS-005 | ADDED | §8 | A condition path that is **absent/null at runtime** makes its gate **fail closed**, distinct from evaluating `false`. |
| CS-006 | ADDED | §11 | The audit write for an executed/settled effect **MUST share the transaction** with the state change (no effect-without-record, no record-without-effect). |
| CS-007 | ADDED | §9 | Kill **propagation** across gateway instances MUST be prompt and **self-healing** (fast notify + authoritative re-read); kill store unreachable ⇒ fail closed for irreversible effects. |
| CS-008 | ADDED | §13 | Linter MUST reject a `compensable` action whose registry entry declares no resolvable `compensation` — enforcing the §5 definition ("`compensable` = a declared undo exists"). Any declared `compensation` MUST name a resource+action that exists in the registry. |
| CS-009 | ADDED | §11 | Audit record gains `resultRefs` (a **list**): the downstream identifier(s) of an executed/settled effect's result (connector-returned id(s) of the created/changed record(s); plural because one action may fan out). The lineage/correlation key that makes an audited effect *locatable* so an external system can reconcile or compensate it. |

*One review item (interception-mode coverage: unmapped tools deny, free-form-string tools flagged) is a transport/architecture concern and lives in `docs/03-architecture-decisions.md`, not in this policy spec.*

### Conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are used as in RFC 2119. A *gateway* is the deterministic enforcement point; a *policy* is one Stele document governing one agent (or a reusable fragment). The *model registry* is the declared catalogue of resources and actions the gateway knows about; the policy references names from it.

---

## 1. Overview and design principles

A Stele document binds an **agent identity** to a set of **permissions** (what it may attempt) and **gates** (deterministic conditions every attempt must satisfy). The gateway evaluates the policy on every action; **no language model runs inside the evaluation**.

Five principles constrain the whole format. They are the reason the language stays small and certifiable.

1. **Default deny.** Anything not explicitly allowed is refused.
2. **Deny wins.** An explicit `deny` overrides any `allow`, always.
3. **Enforcement below the model.** Scope and gates are applied by the gateway after the agent acts; the agent never sees or supplies them.
4. **Deterministic gates only.** Every gate resolves to pass / fail / hold by code or a typed hook — never by model judgement. (A hook MAY call out, e.g. a DLP service, but returns a deterministic verdict.)
5. **Frozen shape.** The vocabulary of *kinds*, *gate types*, and *condition operators* is fixed. Growth happens by adding resources, actions, and named hooks — never new language constructs. (See §13.)

**Trust boundary.** The gateway proves that *intents conform to policy*; it does not prove that the code executing them does what it declares. Connectors, registered hooks, and the gateway itself are the trusted computing base: their integrity is a supply-chain property, mitigated by declaration (connector digest pinning, registry §5 of docs/06; a mismatch is a dependency failure under §10) and by deployment discipline — it is not established by this policy language. The non-normative discussion is in docs/13.

**What v0.6 does not claim (non-goals, CS-039).** Stonefold is not a system of record: it persists match decisions and consumption receipts in the audit log, never obligations. `requireMatch` (§7.16) verifies **entitlement, not judgment** — it establishes that an intent corresponds to a registered obligation within tolerance; it cannot establish that the obligation itself is legitimate. A fraudulent purchase order or an erroneous prescription passes. That risk lives **upstream, at obligation creation**, where existing organisational controls (separation of duties, approval workflows, prescriber/pharmacist verification) already sit — and §13 rule 15 exists precisely so the governed agent cannot become its own upstream. Where no obligation registry can exist (the §14.3 engagement decision has no automatable ground truth), **the human is the registry**: `requireApproval`/`dualAuthorization` remain the honest floor; `requireMatch` does not replace them, and policies SHOULD say so.

---

## 2. Core concepts

| Concept | Meaning |
|---|---|
| **Agent** | The identity the policy governs (e.g. `support-assistant`). One policy per agent, possibly composed from fragments (§3.2). |
| **Actor** | The end principal on whose behalf the agent acts (the human user / session identity). Drives `scope` and approvals. |
| **Resource** | A thing the agent can act on or about — a record type, file, device, channel, sensor (e.g. `Customer`, `Vehicle`, `Email`). Declared in the model registry. |
| **Action** | A named operation of a given **kind** over a resource (e.g. `sendEmail`, `administer`, `engage`). Declared in the registry with its **governance attributes**. |
| **Kind** | One of five fixed categories every action belongs to (§4). |
| **Governance attributes** | Fixed, declared facts about an action that policies reason over: reversibility, emission, operative force, result sensitivity, explainability (§5). |
| **Gate** | A deterministic condition an attempted action must pass (§7). |
| **Decision** | The gateway's verdict: `allow`, `hold` (held for human release, §12), `deny`, or `halt`. |

---

## 3. File structure — top-level keys

### 3.1 Top-level keys

A policy document is YAML. Top-level keys:

| Key | Required | Purpose | Section |
|---|---|---|---|
| `apiVersion` | SHOULD | Spec version, e.g. `stele/v0.1`. | — |
| `agent` | **MUST** | The agent identity this policy governs. | §2 |
| `extends` | MAY | List of fragment policies to compose/inherit. | §3.2 |
| `defaults` | MAY | Document-wide defaults (`failureMode`, `audit`, `killable`). | §9–§11 |
| `allow` | **MUST** | Permissions: actions the agent MAY attempt, by kind. | §6 |
| `deny` | MAY | Explicit prohibitions; override `allow`. | §6 |
| `scope` | MAY | Per-resource scope predicates injected below the model. | §6.3 |
| `gates` | MAY | Deterministic conditions per action / kind / `'*'`. | §7 |
| `standing` | MAY | Time/quantity-conditioned authorizations (ROE, PRN). | §7.15 |
| `killable` | SHOULD | Manner-of-stopping declaration for automated halts (the operator hard-kill is unconditional). | §9 |
| `audit` | SHOULD | Audit level: `none` \| `basic` \| `full`. | §11 |

### 3.2 Composition (`extends`)
A policy MAY list fragments in `extends`; the gateway merges them in order, then applies this document last. Merge rules: `allow`/`deny`/`gates`/`scope` are **unioned**; on conflict, **`deny` always wins** and the **more restrictive** gate value wins (lower limit, narrower allowlist). Composition MUST NOT be able to *widen* a permission a fragment denied.

---

## 4. Action kinds — full enumeration

The five kinds are defined canonically in the **SIF RFC** ([`00-RFC-sif-intent-format.md`](00-RFC-sif-intent-format.md) §2); this section describes their **policy relevance** (which gates matter, where severity comes from). Every action belongs to **exactly one** of these five kinds. The kind is declared in the registry, not chosen by the policy or the agent. The kind shapes which gates are meaningful; it does **not**, by itself, determine severity (that comes from attributes, §5).

### 4.1 `observe` — acquire information, no change to the world
Reading a record, querying data, **passive** sensing, fetching a document. Returns data; changes nothing externally.
- **Primary risk:** disclosure / exfiltration. Reads can leak across tenants or classification levels.
- **Most relevant gates:** `scope`, `disclosure` (result sink), `allowlist`/`denylist`, `rate`, `requireApproval` (e.g. break-glass).
- **Note:** "just reading" is not automatically low-stakes (e.g. accessing a sealed medical record). And **active** sensing that *emits* (radar, sonar, a network probe) is **not** `observe` — it is `effect` (§4.4).

### 4.2 `assess` — produce a consequential judgement
Computing a decision, score, classification, or derived claim others rely on: a triage level, a risk score, a combat identification, a credit decision.
- **Primary risk:** a wrong/biased/unexplained decision that downstream actions trust.
- **Mandatory:** an `assess` action **MUST** declare its inputs and method; high-stakes `assess` **SHOULD** require explanation and/or human confirmation before any `effect` may rely on it.
- **Most relevant gates:** `requireExplanation`, `requireApproval` (`mode: confirm`), `dualAuthorization`, `disclosure`.

### 4.3 `record` — change facts the system owns
Create / update / delete / link / unlink stored data (the classic CRUD; the five built-ins of SIF §2), expressed as named actions.
- **Primary risk:** a record with **operative force** (a DNR, a target designation, a signed diagnosis) is mechanically a `record` but governs real consequences — gate it by its `operativeForce` attribute, not by the kind.
- **Most relevant gates:** `scope`, `precondition`, `valueLimit`, `requireApproval` (when `operativeForce == high`), `rate`/`quota`.

### 4.4 `effect` — cause a change in the external world
Send, dispatch, actuate, pay, drive, transmit — anything reaching beyond the system, **including emitting sensing** (radar/sonar/probe).
- **Primary risk:** irreversibility and blast radius. This is the kind the product exists to govern.
- **Most relevant gates:** all of them; especially `valueLimit`, `spendLimit`, `allowlist`, `precondition`, `contentCheck`, `requireApproval`, `dualAuthorization`, `window`, `quantityCap`, `emissionControl`.
- **Durability rule (CS-003):** because an `effect` cannot be transactionally rolled back, effects are **staged by default**. The gateway **MUST** record the intent and commit it (atomically with any `record` ops in the same batch), return an *accepted/pending* result, then dispatch asynchronously and represent the outcome as a `transition` (`pending → done / failed`) with a declared compensation where one exists. Inline (synchronous) execution is an explicit opt-in permitted **only** for cancellable effects. Staging is also the substrate for approvals (§7.8) and the kill-switch (§9).
- **Freshness rule (CS-017):** staging opens a decide→dispatch gap, so every staged action carries an **expiry (`expires_at`)** stamped at staging from gateway configuration — a decision TTL bounding how stale its decision may get before dispatch (§12). The default MUST be finite; for `irreversible` effects it SHOULD be short (minutes–hours, not days).

### 4.5 `transition` — advance a resource through its declared lifecycle
Move a thing from one declared state to another (`draft → signed`, `conflict_check → active`, `identified → designated`).
- **Primary risk:** performing a step out of order. The legal **from-states** are the institution's permitted process, declared once.
- **Mandatory:** a `transition` action **MUST** declare its legal `from` states; the gateway **MUST** refuse a transition whose current state is not in that set (this is a built-in `precondition`, not optional policy).
- **Most relevant gates:** `precondition` (from-states, built-in), `requireApproval`, `dualAuthorization`, `window`.

> **All five kinds appear, with gates, in the worked examples of §12.**

---

## 5. Governance attributes — full enumeration

Attributes are declared on each action in the registry and are **read-only** to the policy; conditions reference them (e.g. `when action.reversibility == irreversible`). They are how a policy applies severity uniformly without naming every action.

| Attribute | Allowed values | Meaning / typical use |
|---|---|---|
| `reversibility` | `reversible`, `compensable`, `irreversible` | The action's **terminal** recoverability — classify by its most-committed state; a pre-commit *cancellable* window is a runtime/connector property (§8.5, §9), not this attribute. Drives **recovery** controls only: the compensation mandate (`compensable` ⇒ a declared undo MUST exist, §13 rule 10), the fail-closed floor for `irreversible` (§10), and blast-radius warnings (§13.4). `compensable` = a *declared, in-system, gateway-routable* undo **action** exists (refund, discontinue, closeBreaker), distinct from the original — **not** an out-of-band procedure (backup-restore, clinical antidote); `reversible` = self-undo / inverse-data on the same action. **Not** the approval trigger — see the note below. |
| `emission` | `none`, `emits` | Whether the act reveals/transmits into the world even while "just looking." `emits` forces `observe`-looking sensing into `effect` handling. |
| `operativeForce` | `none`, `low`, `high` | Whether parties treat the result as authoritative and act on it (a DNR, a designation). |
| `resultSensitivity` | `public`, `internal`, `confidential`, `restricted`, or a domain classification label | Classification of data an `observe`/`assess` returns. Drives `disclosure`. |
| `explainability` | `none`, `required` | Whether the action (typically `assess`) must carry a recorded rationale. |

Domains MAY extend the *value sets* (e.g. add classification labels) but MUST NOT add new attribute *names*.

> **Note — reversibility ≠ stakes (orthogonal axes).** Reversibility is *recoverability*; it is **not** a proxy for "needs a human." Whether to **hold for approval** is a *stakes* decision and is per-instance: an internal "ticket updated" email and an email leaking PII to an outsider are equally `irreversible`, but only one needs a supervisor. Key approval on **stakes** — `operativeForce`, `resultSensitivity`, and conditions over `data.*` — not on `reversibility`. Gating approval on `action.reversibility == irreversible` *alone* over-gates low-stakes irreversibles (a sent email, a page) and under-thinks high-stakes *reversibles*; the worked example in §14.1 keys its approval on `operativeForce` for this reason. Two cautions: **reversible ≠ safe** — a reversible action can have irreversible *consequences* (a re-closed breaker doesn't restore the blackout; revoking a grant doesn't undo what leaked during the access window), so gate on the consequence's blast radius, not the action's recoverability; and **orthogonal ≠ uncorrelated** — high-stakes irreversibles are common (administer, purge, e-file), so the two axes are *determined separately*, not mutually exclusive. (Design rationale + the cross-domain refinements behind this note: `docs/03` → "Reversibility ≠ stakes".)

---

## 6. Authorization: `allow`, `deny`, `scope`

### 6.1 Syntax
`allow` and `deny` are maps keyed by **kind**, valued by a list of resources or named actions, or `'*'`.

```yaml
allow:
  - observe:    [Customer, Order, Invoice]     # any read of these resources
  - record:     [Note]                          # may create/update Notes
  - effect:     [sendEmail]                      # a specific named effect
  - transition: { Order: [confirm, ship] }       # named transitions on Order
deny:
  - effect:     [refund, exportData]             # never, regardless of allow
  - transition: { Order: [cancel] }
```

- A bare list under `observe` / `record` names **entities** — granting reads/writes of those entities (these kinds are implicit per entity; see the Registry spec §4).
- A bare list under `assess` / `effect` / `transition` names **declared actions** (each bound to an entity in the registry), e.g. `effect: [pay]`.
- A `{ Entity: [names] }` map grants only the **named** actions on that entity (works for any kind), e.g. `transition: { Invoice: [markPaid] }`.
- `'*'` as the value grants the whole kind (use sparingly; the linter warns).

**Bare-name resolution (CS-012).** A bare token under a kind matches either the **resource** of that name — granting *all* of that kind's actions on it, including explicitly declared ones (`observe: [Patient]` grants both the implicit `read` and a declared `readSealed`) — or **any declared action of that kind with that name**, on whichever resource declares it. Action names SHOULD therefore be unique per kind across the registry. If a name is declared by more than one resource, a bare-name `allow` grants it **everywhere it is declared** — the linter warns (§13 rule 12); use the `{ Entity: [names] }` map form to disambiguate. A bare-name `deny` deliberately matches every same-kind action with that name: a broad deny is the safe direction.

### 6.2 Precedence and defaults
The gateway MUST evaluate authorization as:

1. **Default `deny`.** No match ⇒ refused.
2. If any `deny` rule matches the action ⇒ **DENY** (deny always wins).
3. Else if any `allow` rule matches ⇒ proceed to scope and gates.
4. Multiple matching `allow` rules do not compete: any match admits the action, and the gates that then apply are selected by the **`gates` keys alone** (§7) — every gate whose key matches the action (named action, kind, or `'*'`) applies, combined with AND, regardless of which `allow` rule admitted it (CS-025).

### 6.3 `scope`
`scope` maps a resource to a **named scope predicate** resolved by the gateway from the actor's identity and **injected after the model**. The agent cannot read or set it.

```yaml
scope:
  Customer: assignedToCurrentUser     # only rows owned by the actor
  Matter:   clientOf(actor)
  Patient:  inWard(actor.ward)
  Track:    inCompartment(actor.clearance)
```

Scope predicates are declared/registered in the gateway (not free expressions). A scope on a resource applies to **every** kind touching it (an `observe`, a `record`, a `transition`). If a resource has a scope and the actor resolves to an empty set, matching actions return empty / are refused — never widened.

**Reads vs effects (CS-001).** For `observe`/`record`/`transition` that read or write owned data, the predicate is realised as a **filter** (e.g. an injected `WHERE` clause) applied by the connector below the gateway. For an `effect` — where there is nothing to "filter" — the same predicate is enforced as a **pre-resolution authorization check**: the gateway resolves the effect's target first, and if the target is not in the actor's scoped set the action is **DENIED before dispatch**. Either way the agent never supplies or sees its own scope.

**Scope no-race (CS-018).** The scope-on-effect check runs at decision time, and staging (§4.4) widens the check→commit window — so the authorizing state can change in between (an account reassigned to another tenant) and the effect would land on un-authorized state: a TOCTOU race. v0.4 closes it where it can be closed and prices it where it can't, keyed on a capability **each connector declares once** (connector metadata declared in gateway code alongside the connector implementation, like the scope-predicate bindings — not a registry-YAML field, never policy syntax; docs/06 §5):

- **`transactional`** (SQL-class): the gateway MUST re-assert the scope predicate **inside the effect's own transaction** — mechanically, the predicate's constraint is ANDed into the effect's write (`UPDATE … WHERE id = :target AND tenant_id = :actor_tenant`). Zero rows affected ⇒ the effect settles `FAILED` with reason `scope-lost` (audited); the write commits against authorized state **or not at all**. This is the same shape as the kill no-race (§9): the check and the commit share one transaction.
- **`window`** (HTTP, email, device): the predicate cannot ride into the upstream's transaction, so the decision-time pre-check remains the guarantee. The gateway SHOULD re-resolve the target under scope **immediately before dispatch** (shrinking the window to connector latency; a vanished target settles `FAILED`/`scope-lost` with nothing sent), and the connector's **declared** residual window MUST be surfaced in the audit record — the residual risk is priced, not hidden.

This is not dispatch-time re-authorization: `allow`/`deny` and the scope *decision* are not re-derived; only the already-decided predicate is re-asserted against current state. Separately, **pure read staleness is out of scope**: the gateway guarantees scope/disclosure correctness *at read time*, not that the data stays current — and because every effect is re-authorized independently, a stale read cannot itself cause an unauthorized effect.

---

## 7. Gate catalog — full enumeration

Gates attach under `gates`, keyed by a **named action** (bare — `sendEmail` — or resource-qualified — `Order.confirm`), a **kind**, or `'*'` (all actions). All gates that match an action are combined with **AND** — every one MUST pass. Each gate resolves to `pass`, `fail` (⇒ DENY), or `hold` (⇒ held for human release under the holding gate's contract, §12). Any gate value MAY be made conditional with `when:` (§8).

```yaml
gates:
  sendEmail:           # by named action
    rate: 20/hour
  effect:              # by kind (applies to all effects)
    spendLimit: 50/session
  '*':                 # global
    requireApproval: { when: "action.operativeForce == high" }   # key on stakes, not reversibility (§5 note)
```

The complete gate set (rows 1–14 unchanged since v0.1; row 15 is v0.6's one argued amendment to the frozen shape, CS-032; row 6 gained `hold` in v0.6, CS-026):

| # | Gate | Resolves | Purpose |
|---|---|---|---|
| 1 | `rate` | pass/fail | Frequency ceiling per window. |
| 2 | `quota` | pass/fail | Cumulative cap over window/session. |
| 3 | `valueLimit` | pass/fail | Numeric bound on a parameter/field. |
| 4 | `spendLimit` | pass/fail | Cost/$ ceiling per task/session. |
| 5 | `allowlist` / `denylist` | pass/fail | Membership constraint on a field. |
| 6 | `precondition` | pass/fail/hold | Named deterministic check / lifecycle from-states. |
| 7 | `contentCheck` | pass/fail | Typed hook (DLP, PII, classification scan). |
| 8 | `requireApproval` | pass/hold | Human sign-off, optionally conditional. |
| 9 | `dualAuthorization` | pass/hold | Two distinct identities required. |
| 10 | `window` | pass/fail | Temporal allow (hours, date range). |
| 11 | `quantityCap` | pass/fail | Per-subject cumulative cap (e.g. per patient). |
| 12 | `disclosure` | pass/fail | Result classification ↔ allowed recipients/sinks (reads). |
| 13 | `emissionControl` | pass/fail/hold | Deconfliction/authorization for emitting effects. |
| 14 | `requireExplanation` | pass/fail | Action must carry a recorded rationale (assess). |
| 15 | `requireMatch` | pass/fail/hold | Intent must match exactly one open obligation in a declared registry; reserved at staging, consumed at settlement (§7.16). |

### 7.1 `rate`
`N/window` where window ∈ `second|minute|hour|day`. (Duration-*valued* fields elsewhere — `requireApproval.timeout`, `quantityCap.window` — use the `Ns/Nm/Nh/Nd` shorthand instead; the two forms are not interchangeable.) Optional `per:` to scope the count.
```yaml
sendEmail:
  rate: 20/hour
  # or per-target:
charge:
  rate: { limit: 3/day, per: resource.customerId }
```

### 7.2 `quota`
Cumulative cap over a longer horizon or per session.
```yaml
exportReport:
  quota: 100/day
```

### 7.3 `valueLimit`
Bounds a numeric parameter. Supports `max`, `min`, or both.
```yaml
pay:
  valueLimit: { field: data.amount, max: 10000, currency: USD }
setSpeed:
  valueLimit: { field: data.kph, max: 130, min: 0 }
```

### 7.4 `spendLimit`
Cost ceiling for the agent's run; stops retry storms.
```yaml
effect:
  spendLimit: 25/session        # $ or token-cost units, gateway-configured
```
The **unit** (currency or token-cost) and each action's cost assignment are **gateway configuration** (deployment config, like the decision TTLs of §12 — never policy syntax). A policy's number is denominated in the deployment's configured unit, so the figure is not portable across deployments, and a conformance claim does not compare it across gateways (CS-025).

### 7.5 `allowlist` / `denylist`
Membership on a field. Lists MAY reference named sets (`allowlist:corporate-domains`).
```yaml
sendEmail:
  allowlist: { field: data.recipientDomain, set: corporate-domains }
eFile:
  allowlist: { field: data.court, set: approved-court-systems }
transferFunds:
  denylist:  { field: data.destinationCountry, set: sanctioned-list }
```

### 7.6 `precondition`
A named, registered deterministic check, or — for a `transition` — the legal `from` states (the latter is built in and MUST always hold).
```yaml
administer:
  precondition: [fiveRightsVerified, notDiscontinued]
Order.ship:
  precondition: { from: [packed] }          # transition from-states
engage:
  precondition: [positiveIdentification]
issueRefund:                                # a hold-capable check (CS-026) names its resolver
  precondition:
    checks: [matchesOpenCase]
    resolvers: role:support-supervisor
```

**Three results (CS-026).** A named check resolves **pass / fail / hold**. `hold` suspends the intent for human resolution through the held lifecycle (§4.4, §12) — there is no new state. Its use is bounded by three rules:

1. **Outages fail; only readable ambiguity holds.** A check that cannot reach its source system MUST resolve `fail`; a check that raises is a dependency failure under §10 — the gateway MUST map a crash to `fail`, never to `hold`. `hold` is legal only when the data was read successfully AND the result is judgment-shaped (multiple matching records, a near-tolerance value, a field that means "ask a human"). If outages held, every registry blip would become a human interruption and fail-closed would silently degrade into fail-to-a-queue that gets rubber-stamped; scarce holds get read, constant holds get ignored.
2. **A hold carries a machine-readable reason code** (§11) or it is invalid: the gateway MUST treat a code-less hold as a check implementation error — resolve `fail`, log loudly. The human resolving the hold must see *why* it paused without reading check code.
3. **Opt-in per check.** A check declares hold capability in its registry declaration (`holdCapable`, docs/06 §5); two-valued checks remain valid indefinitely.

Who may release a precondition hold is the gate's `resolvers:` field, or the deployment's default resolver role — see the release-contract rules in §12 (CS-027). At dispatch-time re-validation (CS-017) nothing changes: `precondition` is volatile, and any non-PASS at the claim — including a fresh `hold` — settles `CANCELLED`/`stale-guard:precondition`; a claimed row is never re-suspended.

### 7.7 `contentCheck`
A typed hook returning pass/block. Hooks are registered code; the policy names one.
```yaml
sendEmail:
  contentCheck: dlp.basic
publish:
  contentCheck: [pii.scan, classification.scan]
```

### 7.8 `requireApproval`
Holds the action for a human. Fields: `when` (condition; default always), `approvers` (role or set), `quorum` (default 1), `timeout`, `onTimeout` (`deny` default | `allow`), `mode` (`approve` default | `confirm`).
```yaml
'*':
  requireApproval:
    when: "action.operativeForce == high"    # key on stakes, not reversibility (§5 note)
    approvers: role:supervisor
    timeout: 30m
    onTimeout: deny
refund:
  requireApproval: { approvers: role:finance-manager }
```
`approvers` names (`role:…`) resolve against the **identity layer** — the session/`IdentityProvider` seam (architecture decision 11) — not the registry. They are the one referenced namespace §13 rule 1 does not lint: the registry declares the world the agent acts on; who may approve is an organisational fact the deployment owns (CS-025). The same rule covers the `resolvers:` field on hold-capable gates (§7.6, §7.16).

**`timeout` is enforced (CS-028).** A held row expires at the earlier of the holding gate's `timeout` and the staging TTL (CS-017), via an active sweep (§12): an expired hold settles `CANCELLED`/`expired-hold:<gate>` (a normative settle reason, like `stale-decision`), preserving the original hold reason code in the audit record. `onTimeout: deny` (default) cancels; `onTimeout: allow` satisfies **this gate's contract only** — the row promotes iff every other release contract is also satisfied (§12, CS-027). A late release never resurrects an expired row (CS-017 rule 1 unchanged).

### 7.9 `dualAuthorization`
Two **distinct** identities must approve (the actor cannot self-approve). Fields: `approvers`, `quorum: 2` implied, `distinctFrom: actor`.
```yaml
engage:
  dualAuthorization: { approvers: role:weapons-release-authority }
wireTransfer:
  dualAuthorization: { when: "data.amount > 50000", approvers: role:treasury }
```

### 7.10 `window`
Temporal allow. A match outside the window ⇒ fail. Two forms, combinable: **recurring** (`days` / `hours` / `tz`) and **absolute** (`from` / `to` dates — the catalog row's "date range"; CS-025).
```yaml
deploy:
  window: { days: [Mon,Tue,Wed,Thu], hours: "09:00-16:00", tz: "Europe/Bratislava" }
migrationWrite:
  window: { from: "2026-07-01", to: "2026-07-31" }      # absolute date range
```

### 7.11 `quantityCap`
Per-subject cumulative cap over a window — the PRN/standing-order pattern.
```yaml
administer:
  quantityCap: { per: resource.patientId, limit: 3, window: 24h, of: data.drug }
```

### 7.12 `disclosure`
Binds the **result classification** of a read to permitted recipients/sinks. Prevents the exfiltration/spillage leg.
```yaml
observe:
  disclosure:
    when: "action.resultSensitivity == restricted"
    allowSink: [careTeam]                 # named sinks; default-deny others
readIntel:
  disclosure: { maxClassification: actor.clearance }
```
**Two forms (CS-002).** `disclosure` is enforced in whichever form the data allows: a **pre-check** when the result's sensitivity is known from the registry (the read is **blocked before execution**), and a **post-check** when sensitivity is row-dependent (the read executes, but a disallowed result is **withheld on return** and the decision recorded as `deny` with "result withheld"). The gateway MUST use the pre-check form whenever it can determine sensitivity without executing.

**Classification ordering (CS-024).** `maxClassification` compares by the classification set's **declared order**: the built-in `resultSensitivity` values are ordered `public < internal < confidential < restricted`. A domain substituting its own labels (§5) MUST declare them as an **ordered** value set in the registry (docs/06 §4 — order is list position, lowest first). A classification value missing from the declared order makes the gate **fail closed** (the §8 runtime-resolution rule).

### 7.13 `emissionControl`
For `effect` actions with `emission == emits`: require deconfliction/authorization before the emission. Its value takes the same shape as `precondition` (`checks:` / `when:`).
```yaml
radarSweep:
  emissionControl: { checks: [emconAuthorized, deconflicted] }
```
A failed check resolves **fail** (⇒ DENY); the gate resolves **hold** only when the required authorization is a pending human/deconfliction decision rather than a failed deterministic check.

### 7.14 `requireExplanation`
For `assess`: the action MUST produce a recorded rationale (and SHOULD pair with `requireApproval: {mode: confirm}` when high-stakes).
```yaml
triage:
  requireExplanation: true
combatId:
  requireExplanation: true
  requireApproval: { mode: confirm, approvers: role:tactical-officer }
```

### 7.15 `standing` (top-level conditional authorizations)
`standing` declares grants that are *off by default* and switched on by context — ROE states, shift windows, PRN orders. They are evaluated as additional `allow` + gate conditions.

**Standing never overrides `deny` (CS-010).** A standing grant is a *conditional allow* and is subject to the §6.2 precedence unchanged: an explicit `deny` beats it, always. To make an action available *only* under a standing rule, leave it **out of `allow`** (default-deny covers the off state) and do **not** list it in `deny` — a policy that lists the same action in both `deny` and a `standing` rule's `enables` is unsatisfiable and MUST be rejected by the linter (§13 rule 11).
```yaml
standing:
  - name: weapons-free
    when: "context.roeState == 'weapons_free'"
    enables: { effect: [engage] }
  - name: clinic-hours-orders
    when: "context.time in window('08:00-18:00')"
    enables: { transition: { Order: [sign] } }
```

### 7.16 `requireMatch` (CS-032 — v0.6)

An intent MUST correspond to **exactly one open obligation** in a declared obligation registry (docs/06 §5b), within declared tolerance; the obligation is **reserved at staging and consumed at settlement** (§12, CS-035). Applies to `effect`, `transition`, and `record` actions; resolves pass / fail / hold. This is the gate for the *in-bounds wrong action*: every other gate compares the intent against constants the policy carries; `requireMatch` checks the relation between the intent and a record created earlier, by a different authority, in a system the agent cannot write to — an invoice is payable only against an open order, a dose is administrable only against an active prescription. (Rationale, including why this could not be a `precondition` hook or connector logic, is argued in the change set, CS-032.)

```yaml
gates:
  pay:
    requireMatch:
      registry: erp.purchase_orders            # declared obligation registry (docs/06 §5b)
      match:
        - "obligation.vendorId == data.vendorId"
        - "obligation.state == 'open'"
        - "obligation.line.state == 'unconsumed'"
        - { field: obligation.line.amount, matches: data.amount, within: "10%" }   # tolerance clause (CS-033)
      provenance:
        - "obligation.vendor.domain == data.sourceDomain"
      consume: obligation.line
      onNoMatch: deny              # deny (default) | hold
      onAmbiguous: hold            # hold (default); `allow` is not a legal value
      resolvers: role:ap-clerk     # who releases a hold this gate raises (§12, CS-027)
```

**Fields.**

| Field | Required | Meaning |
|---|---|---|
| `registry` | MUST | A declared obligation registry (docs/06 §5b). Unknown ⇒ load error (§13 rule 14). |
| `match` | MUST | A conjunction over the matched obligation: each entry is either a §8-grammar comparison between `obligation.*` and the §8 namespaces (`data.*`, `resource.*`, `actor.*`, `context.*`), or a **tolerance clause** (below). |
| `provenance` | MAY | Additional conjunction binding the matched obligation's counterparty fields to the intent's declared source evidence. Kills the valid-but-wrong-pointer class: a real identifier resolving to a different counterparty's record. Requires the intent type to carry that evidence as typed fields. |
| `consume` | MUST for `effect`/`transition`; MAY be `none` for pure verification on `record` | The obligation path marked spent at settlement (§12). `consume: none` on an irreversible effect lints as a warning (§13 rule 16). |
| `onNoMatch` | MAY | Verdict when zero candidates match: `deny` (default) or `hold`. |
| `onAmbiguous` | MAY | Verdict when more than one candidate matches: `hold` (default). `allow` is not a legal value (§13 rule 17); the gateway MUST NOT auto-select among candidates. |
| `resolvers` | MAY | The `role:` name (identity seam, §7.8) that may release a hold this gate raises; falls back to the deployment default resolver role (§12, CS-027). |

**Tolerance clauses (CS-033).** Approximate equality under declared tolerance is a structured match entry — deliberately **not** a condition-grammar operator (§8 is unchanged):

```yaml
- { field: obligation.line.amount, matches: data.amount, within: "10%" }   # relative, of the obligation-side value
- { field: obligation.dose,        matches: data.dose,   within: 0 }       # exact, stated
```

`field` is an `obligation.*` path; `matches` is a §8 namespace path; `within` is `"N%"` (relative to the obligation-side value) or an absolute number in the field's unit; `0` means exact. Tolerance applies to numeric/money fields only (§13 rule 14).

**Semantics.**

1. **Deterministic evaluation at decision time** (§12 step 4). The gateway queries the registry with the typed selector derived from `match`. Candidate count 1 ⇒ the remaining clauses evaluate against that obligation; 0 ⇒ `onNoMatch`; >1 ⇒ `onAmbiguous`. No model output participates.
2. **Agent-independent read path (CS-036).** Every `obligation.*` operand resolves from the registry's response; agent-supplied copies of obligation data are never match inputs. The intent MAY carry an obligation **pointer** (e.g. `data.poId`); the gateway then queries by id and still evaluates the full `match` conjunction against the re-read record — a pointer narrows the query, it never substitutes for it.
3. **Volatile gate, refined.** `requireMatch` joins the CS-017 dispatch-claim re-validation set — with one refinement: because the obligation is reserved at staging (§12), the claim checks **reservation liveness** (still held, registry reachable) rather than re-running the query. Reservation lost ⇒ `CANCELLED`/`stale-guard:requireMatch`.
4. **Runtime resolution (CS-005 applies).** An `obligation.*` path absent or null on the matched record fails the gate closed. Registry unreachable ⇒ `failureMode` (§10); for `irreversible` effects this MUST resolve closed.
5. **Composition unchanged** (§7): AND with all other matching gates. A matched obligation never relaxes a `valueLimit`, a list, an approval, or any other gate.

**When a plain `precondition` is enough (adoption path).** `requireMatch` is the obligation-checking pattern *promoted into policy syntax* — it is not the only way to run the pattern, and it is deliberately not the place to start. A registered precondition check that queries the system of record and compares typed fields gives the same protection on any gateway, with no registry declaration and roughly fifty lines of code:

```yaml
gates:
  pay:
    precondition: [matchesOpenPurchaseOrder]   # the whole feature, v0.5-style
```

The check follows five rules (they are `requireMatch`'s semantics, hand-carried): read every compared field from the source, never from the intent (the intent may carry an id to narrow the query); compare typed fields only, never free text; fail closed with a reason code on no-match, several-matches, and source-down alike; be deterministic; write the tolerance down as a visible constant. Spending is left to the record system — a real ERP consumes the order line when the invoice posts, an EMR fills the slot when the dose is charted — so the double-spend window closes in the system that owns the record. Preconditions are already volatile (CS-017), so dispatch-time re-checking comes for free.

Reach for `requireMatch` only when you need what a named check cannot give: the match rule **in the policy file** (reviewers, the linter, and the TCK see "payment requires an open order" instead of a function name); `ambiguous ⇒ hold` routed to a named resolver instead of a bare fail; **gateway-managed reservation and consumption** — which matters exactly when the record system does *not* enforce spending itself (a spreadsheet, a homegrown list, a slow-posting upstream); and the standardized `obligationRefs`/`consumption` audit lineage. If your system of record already consumes transactionally and your reviewers accept a named check, the precondition *is* the right-sized tool — using it is not a lesser deployment, and a policy that outgrows it can move to `requireMatch` later without changing the check's semantics.

---

## 8. Condition expression language

Conditions (`when:`) are a **small, frozen** boolean language. No loops, no assignment, no arithmetic beyond comparison. Grammar (EBNF):

```
condition  := orExpr
orExpr     := andExpr ("or" andExpr)*
andExpr    := unary ("and" unary)*
unary      := "not" unary | comparison | "(" condition ")"
comparison := operand op operand
            | operand ("in" | "not in") (list | function)
            | "exists" path
op         := "==" | "!=" | "<" | "<=" | ">" | ">=" | "matches"
operand    := path | literal | function
path       := ident ("." ident)*
function   := ident "(" [ literal ("," literal)* ] ")"
literal    := string | number | boolean | duration | list
duration   := number ("s" | "m" | "h" | "d")
list       := "[" [ literal ("," literal)* ] "]"
```

**Reference namespaces** (read-only):

| Namespace | Examples | Meaning |
|---|---|---|
| `action.*` | `action.kind`, `action.name`, `action.resource`, `action.reversibility`, `action.emission`, `action.operativeForce`, `action.resultSensitivity`, `action.explainability` | The attempted action and its declared attributes. |
| `data.*` | `data.amount`, `data.recipientDomain`, `data.kph` | Parameters the agent supplied. |
| `resource.*` | `resource.patientId`, `resource.currentState`, `resource.ownerId` | Properties of the resolved target. |
| `actor.*` | `actor.id`, `actor.role`, `actor.clearance`, `actor.ward` | The principal the agent acts for. |
| `context.*` | `context.now`, `context.time`, `context.roeState`, `context.sessionSpend` | Ambient state. |

**Functions** (the complete set): `count(scope, window)`, `now()`, `window("HH:MM-HH:MM")`, `spend(window)`. No others.

String literals may be single- or double-quoted. The right side of `in` / `not in` is a list literal or a function returning a collection/range — `context.time in window("08:00-18:00")` (CS-013; this legalises the form §7.15's example already used).

**This grammar is unchanged in v0.6.** The `requireMatch` tolerance comparison is deliberately a structured gate field (§7.16, CS-033), not an operator — CS-013's widening of `in` remains the only grammar change since v0.1. Inside `requireMatch.match`/`provenance`, string entries use this grammar with one additional read-only namespace, `obligation.*`, resolved from the registry's response (CS-036); `obligation.*` is not available anywhere else a condition appears.

**Runtime resolution (CS-005).** Unknown paths are rejected at policy load (§13.9). If a referenced path is **absent or null at runtime** (e.g. `resource.foo` is missing on the resolved target), the gate whose condition referenced it **fails closed** (resolves DENY) — this is distinct from the condition evaluating to `false`. A condition error MUST NOT silently pass a gate.

---

## 9. Kill-switch (`killable`)
Two independent axes (CS-022). **The operator hard-kill is unconditional:** a human operator can always issue a `halt` for a scope — a policy cannot opt out, and `killable` never gates it (otherwise a compromised agent could declare itself un-killable; an operator can pull the process/credentials/network regardless, so an opt-out would be a false guarantee). A `halt`:
- stops in-flight actions for an **action class**, a **session**, or the **agent**;
- causes subsequent matched attempts to resolve `halt` (not `deny`) — a distinct, audited terminal state;
- is itself an audited operator action (who halted, when, scope) and is reversible (the order can be lifted).

**`killable`** (default SHOULD be true for non-trivial agents) is a separate, action-level declaration of the *manner of stopping under normal/automated operation*: `killable: false` means "a generic mid-flight freeze is unsafe for this action — stop it via its declared safe-stop/compensation." It guards **automated** halts and *informs* the operator hard-kill (a warning/confirmation when a hard-kill scope covers non-killable actions) — it never blocks it. `killable` is also distinct from `reversibility`: `killable` = *may a generic live-halt stop this at all?*; `reversibility` = *how much a kill can claw back once in motion* (the "scope of the guarantee" below). Granularity and the graceful-halt mechanism are design work in progress — see `docs/03` → "Kill is two axes".

**No-race guarantee (CS-004).** A `halt` MUST take effect before the connector dispatch of any pending `effect`. The gateway MUST evaluate the kill at three points — entry (whole-agent/session short-circuit), per-action (pipeline step 5), and **at dispatch**, where the kill re-check and the staged action's `pending → dispatching` transition MUST occur in **one serialised transaction** (e.g. a row-locked update) so there is no window in which an action has both passed the kill check and remains un-dispatched. Each staged action carries an **idempotency key** so a cancelled action can never later dispatch.

**Scope of the guarantee (CS-004).** Kill prevents any *new* or *not-yet-dispatched* action; cancels in-flight actions whose connector is cancellable; and triggers declared **compensation** for irreversible effects already dispatched. It does **not** reverse an external effect that has already committed — nor does it reach effects that a committed effect *triggers downstream* (world→world cascades, §11 scope boundary): kill bounds the agent's actions, not the world's reactions.

**Propagation (CS-007).** A kill MUST take effect across all gateway instances **promptly and reliably** — by fast notification (e.g. pub/sub) plus a self-healing authoritative re-read (e.g. an epoch counter) so a dropped notification cannot leave an instance unaware. If the kill store is unreachable, the gateway MUST **fail closed** for irreversible effects.

*Mechanism detail (state stores, the locked-transition transaction, in-flight cancellation) is in the implementation design §8.*

> *The former UNDER-REVIEW note reconciling `killable` with the operator hard-kill is retired: its content is now the section opening above (CS-022). Still open in `docs/03` → "Kill is two axes": graceful-halt as feature vs seam, per-action vs per-agent granularity, whether `killable: false` requires a declared safe-stop, and the one-bool-vs-split question.*

---

## 10. Failure mode (`defaults.failureMode`)
If the gateway, a `contentCheck` hook, or a scope/approval dependency is **unavailable or errors** — or a connector fails its declared-digest check (docs/06 §5) — behavior is governed by `failureMode`:

```yaml
defaults:
  failureMode: closed        # closed (default) | open
```
- `closed` — the action is **denied** (regulated/safety default). MUST be the default.
- `open` — the action is allowed (only for low-stakes deployments).
`failureMode` MAY be overridden per kind/action; an `open` override on an `irreversible` action MUST be a linter error unless explicitly acknowledged.

---

## 11. Audit (`audit`)
Levels: `none` | `basic` (decisions only) | `full` (decisions + parameters + gate results). Regulated deployments SHOULD use `full`. Every evaluated action — **allowed, held, denied, or halted** — produces one append-only record. Required fields at `full`:

| Field | Description |
|---|---|
| `id`, `timestamp` | Unique id and time. |
| `agent`, `actor` | Governing agent and the principal it acted for. |
| `kind`, `resource`, `action` | The attempted action. |
| `parameters` | Typed parameters supplied (subject to redaction policy). |
| `scopeApplied` | Scope predicate(s) injected. For a settled effect, also **which scope-reassertion form ran** (CS-018): `transactional`, or `window` with the connector's declared residual window. |
| `gates` | Each gate evaluated and its result (pass/fail/hold). |
| `decision` | `allow` \| `hold` \| `deny` \| `halt`, with the deciding rule/gate. |
| `approval` | Approver(s), quorum, outcome, timing — if applicable. |
| `outcome` | Connector result: `success` \| `failure` (+ reason) \| `not_executed`. |
| `resultRefs` | Stable downstream identifier(s) of the effect's result — the connector-returned id(s) of the created/changed record(s) (ledger entry, payment, message id, …). A **list**: one action may fan out to several records (a payment *and* its ledger entry), so it is the lineage/correlation key, not a single id. Populated for executed/settled effects; empty otherwise. The handle(s) an external system uses to locate, reconcile, or compensate the effect; the gateway records them but does **not** itself perform the reversal. |
| `obligationRefs` | (v0.6, CS-037) Registry + obligation/line id(s) a `requireMatch` decision matched, with the candidate count (`1`, `0`, or `n>1` for a held-ambiguous decision). The **entitlement-side** lineage key, complementing `resultRefs`: `resultRefs` locate what the effect *produced*; `obligationRefs` locate what *entitled* it. |
| `consumption` | (v0.6, CS-037) `reserved` \| `consumed` \| `released`, with the consume receipt id; for a `window`-capability registry, the declared residual window (the CS-018 pattern: priced, not hidden). |
| `correlationId` | Session/transaction id for replay. |

**Transactional audit (CS-006).** For an executed or settled `effect`, the audit record **MUST** be written in the **same transaction** as the state change it records (the outbox settle), so there can be neither an effect that occurred without a record nor a record of an effect that did not occur. Refusals and holds are recorded **before** the result is returned to the agent. Best-effort side-channel logging is **not** sufficient for the audit log.

**Remediation is downstream (boundary note).** The gateway's role in undoing a wrong-but-allowed effect is to make it *findable and actionable* — a complete, attributable record carrying `resultRefs` (CS-009) — **not** to perform the reversal. The compensating action is executed by the system of record, or as a gated operator action (§9), never reconstructed inside the gateway.

**Scope boundary — the gateway governs agent→world, not world→world.** The unit of enforcement is **one resolved action**; a compound/batch intent is decomposed into N actions, each independently authorized, audited, killed, and carrying its own `resultRefs` (bulk-as-one-effect is out of scope). The gateway records the *direct* effects of an agent action; it does **not** see or govern the **cascade** those effects trigger in downstream systems (a posted payment that fires a webhook → a journal entry → a covenant alert). Therefore an action's `reversibility`, `compensation`, `resultRefs`, and the kill guarantee all describe the **direct** effect only — never the world's reactions to it. Cascade reconciliation is the downstream systems' responsibility, joined back via `resultRefs`/`correlationId`; multi-step transactional consistency across several agent intents (sagas) is out of scope (the audit trail makes them reconstructable and externally unwindable, but Stele guarantees no atomicity across intents). Design analysis: `docs/03` → "Multi-effect & cascade".

**Reason codes and retry classes (CS-029 — v0.6).** Every deny/hold reason a gate or registered check produces is a **machine-readable code** with a declared **retry class**, returned to the agent alongside the decision:

- `retryable` — the defect is in the intent; fix it and resubmit (amount outside tolerance, missing field, busy rate window).
- `terminal` — nothing about this intent family is fixable by the agent; do not resubmit (no obligation exists, denylisted destination, unknown action).
- `escalate` — stop and surface to a human on the **agent's** side (distinct from a gateway hold, which queues on the operator's side).

A code with no declared class defaults to `terminal` — the safe direction is to stop retrying. Registered checks declare their codes and classes in the registry (docs/06 §5); built-in gate reasons carry normative classes: `valueLimit`, `rate`, `quota`, `quantityCap`, `spendLimit`, `window`, `contentCheck`, `requireExplanation` ⇒ `retryable`; `allowlist`/`denylist`, `disclosure`, unknown-action, scope refusals ⇒ `terminal`; `stale-decision` and `stale-guard:<gate>` ⇒ `retryable` (resubmit for a fresh decision); `expired-hold` ⇒ `escalate`; `hold-unresolvable` ⇒ `escalate` (a configuration error a human must fix). An approval-shaped hold with no check code carries no class — the agent's move there is to wait, which none of the three classes means. The audit record carries code and class for every refusal (this section's table, `decision`/`outcome`).

**Agent feedback visibility (CS-030 — v0.6).** What the *agent* receives on a deny or hold is a declared choice, per gate config or per check declaration, via an optional `feedback:` key:

- `code` — reason code + retry class only.
- `code+fields` — **the default**: code, class, and *which* intent fields failed the comparison — never the record-side values they were compared against.
- `code+evidence` — the full comparison including record-side values; for trusted internal loops only.

Visibility governs the agent-facing result only; the audit record always carries everything — **redact on return, never on write**. Reason codes remain an oracle even at `code` (each probe maps one policy wall), so the countermeasure is detection, not blunting: audit tooling SHOULD surface deny-rate and reason-code distribution per agent principal — a converging loop and a mapping loop look different in that data. Repeated `ambiguous` outcomes on one obligation registry SHOULD likewise be surfaced: they signal near-duplicate injection into the obligation store (CS-037).

---

## 12. Evaluation order (the pipeline)
For each attempted action the gateway MUST proceed strictly in this order, stopping at the first terminal verdict:

1. **Resolve** the action's kind, resource, name, and attributes from the registry. Unknown ⇒ DENY.
2. **Authorize** (§6.2): default deny → deny-wins → allow-match.
3. **Inject scope** (§6.3).
4. **Evaluate gates** (§7), cheapest/deterministic first; `requireApproval`/`dualAuthorization` last. Any `fail` ⇒ DENY; else any `hold`(s) ⇒ HOLD — the intent is staged carrying the release contract of **every** holding gate (CS-027, below), and re-enters at step 5 once all are satisfied.
5. **Check kill-switch** (§9). Active ⇒ HALT.
6. **Execute** via the connector as one transaction (effects staged per §4.4 durability rule).
7. **Record** the audit entry (§11) — for every outcome, including refusals.

On any dependency error, apply `failureMode` (§10).

**Batch decision semantics (CS-023).** A SIF batch (SIF §5) is decided **atomically** and executed per operation. The gateway runs steps 1–5 for **every** operation in the batch first (each operation gets its own audit record, §11); any DENY or HALT on any operation refuses the **whole batch** before anything commits or stages — no `record`/`transition` applies, no `effect` stages, and the structured error identifies the failing operation (SIF §6 pointer). A batch is a request for atomicity: an agent that wants independent outcomes submits independent intents. A HOLD does **not** refuse the batch: the batch commits with the held effect staged `PENDING_APPROVAL`, and per §4.4 any `record` ops in the batch commit atomically with that staging. A later rejection or TTL expiry (CS-017) of the held effect does not roll those committed ops back — each was independently authorized, and `correlationId` ties them together for downstream reconciliation (§11).

**Decision freshness (CS-017).** This evaluation runs at **decision time**; for a staged effect (§4.4) the gateway MUST bound how stale that decision can get before dispatch, two ways:

1. **Decision TTL.** Every staged action carries an expiry, set at staging from gateway configuration (deployment config, **not** policy syntax — the language stays frozen). The default MUST be finite; for `irreversible` effects it SHOULD be short (minutes–hours, not days). A row claimed at or after its TTL settles `CANCELLED` with reason `stale-decision` (audited; the agent's ticket resolves to a recoverable refusal). An approval that arrives after expiry does not resurrect the row — the intent must be re-submitted and re-decided.
2. **Volatile-gate re-validation at dispatch.** Inside the dispatch claim — after the §9 kill re-check, before the connector call (order: **kill → TTL → volatile gates → connector**) — the gateway re-evaluates the action's **volatile** gates: `allowlist`/`denylist` (set membership changes), `window` (time has passed), `precondition`/`emissionControl` (world state changes), including registry-intrinsic preconditions; for `requireMatch` the claim checks **reservation liveness** rather than re-running the query (§7.16 rule 3, CS-035). It MUST do so for `irreversible` effects and SHOULD for all staged effects. A dispatch-time failure settles `CANCELLED` with reason `stale-guard:<gate>` (audited), never a partial dispatch.

**Non-volatile gates are NOT re-run**, by definition: `valueLimit` and `contentCheck` judge the staged payload, which is frozen; the counters (`rate`/`quota`/`quantityCap`/`spendLimit`) were consumed at decision time — re-running them double-counts; and a `requireApproval`/`dualAuthorization` grant *is* the release — its freshness is bounded by the TTL (rule 1), not by re-asking. This is **not dispatch-time re-authorization**: `allow`/`deny` and scope *decisions* are not re-derived, approvals are not re-requested; the TTL bounds how stale any decision may get, and re-validation covers only the gate classes whose facts move independently of the agent. The kill switch remains the authoritative dispatch check (§9); CS-017's checks run inside the same claimed transaction, after it.

**Hold release, expiry, and dedupe (CS-027, CS-028, CS-031 — v0.6).** When more than one gate resolves `hold` for the same intent, the staged row carries the **release contract of every holding gate**, and promotion to dispatch requires **each contract satisfied** — satisfying one never satisfies another, and a resolved hold is not a bypass of anything (the gates all evaluated at decision time; dispatch re-validates the volatile ones as always). A release contract is what the holding gate demands: for `requireApproval`, its approvers/quorum/timeout; for `dualAuthorization`, two distinct non-actor identities; for a holding `precondition` or `requireMatch`, the gate's `resolvers:` role (identity seam, §7.8) or, when absent, the deployment's **default resolver role** (gateway configuration, like decision TTLs — never policy syntax). A hold whose contract names no approver/resolver and has no deployment default MUST be refused fail-closed at decision time, audited `hold-unresolvable` — never staged as a row anyone could release. Each release is recorded like an approval (who, when, which contract).

Held rows **expire actively** (CS-028): the gateway MUST sweep held rows (the dispatch worker's loop is sufficient) and settle an expired hold `CANCELLED`/`expired-hold:<gate>`, preserving the original hold reason code — see §7.8 for the `timeout`/`onTimeout` rules. Both deadlines — the gate's `timeout` and the staging TTL (CS-017) — are measured **from staging, on the same clock that stamped the staging TTL** (the decision-time clock): anchoring them on different clocks makes the earlier-of comparison meaningless under skew. And holds **dedupe** (CS-031): a new hold whose (agent, action, reason code, matched-candidate refs) equal an already-open held row's within the deployment's dedupe window collapses into that row — attempt count incremented, latest intent recorded, an audit record still written per attempt. Denies are cheap (pre-commit, no side effect); holds spend human attention, the scarcest resource in the system — ten variants of the same unmatched invoice are one question wearing ten disguises, and the operator sees one queue item. (A per-principal ceiling on open holds is deferred to v0.6.1.)

**Obligation reservation (CS-035 — v0.6).** For an intent that passed a `requireMatch` gate, obligation state tracks the staged effect's lifecycle exactly — otherwise the gate re-opens the races v0.4/v0.5 closed:

- **Decision (step 4):** query + match (§7.16); on pass, the matched candidate's ref enters the decision record (`obligationRefs`, §11).
- **Staging (§4.4 commit):** the gateway calls `reserve(ref, intent_id)`, and the reservation MUST be in place before the staging commit returns *accepted/pending* — reservation is what closes the double-spend window in the decide→dispatch gap that the decision TTL alone does not. `AlreadyReserved`/`AlreadyConsumed` here ⇒ the intent settles refused `no-match`, audited.
- **Dispatch claim:** order per CS-017, extended — kill → TTL → volatile gates (including reservation liveness) → connector.
- **Settlement:** `consume(ref, intent_id)` executes with the settle, per the registry's declared capability (docs/06 §5b): **`transactional`** ⇒ inside the same transaction as the effect's commit and the CS-006 audit write — no consumed-without-effect, no effect-without-consumed; **`window`** ⇒ immediately after connector confirmation, with the declared residual window surfaced in the audit record (the CS-018 pattern: priced, not hidden).
- **Release:** any terminal non-success — `CANCELLED` (kill, `stale-decision`, `stale-guard`, `expired-hold`, rejection) or `FAILED` — MUST release the reservation. A TTL expiry therefore frees the obligation for a re-submitted intent.
- **Orphan recovery:** a crash between `reserve` and the staging commit leaves a reservation with no row. Every reservation therefore carries a TTL of its own, agreed with the adapter and **at least** the staged row's decision TTL; the adapter MUST expire orphaned reservations, the gateway's release path is idempotent (`NotHeld` is a no-op), and a deployment SHOULD reconcile reservations against staged rows on gateway restart. The reservation TTL runs on the **adapter's** clock, not the gateway's — the gateway MUST tolerate the skew: a reservation lost to expiry is caught by the dispatch liveness check, an expired-but-unclaimed reservation MAY be re-acquired by the same intent at that check, and releasing an adapter-expired reservation is the `NotHeld` no-op.
- **Batches (CS-023):** reservations for all operations in a batch are taken with the batch's atomic staging; any refused operation refuses the batch and releases every reservation taken for it.
- **Idempotency:** `reserve`/`consume`/`release` are idempotent per (obligation ref, intent id); a retry never double-consumes; a second **distinct** intent against a consumed line resolves `no-match`.

---

## 13. Validation rules (what the linter MUST check)
1. Every resource/action/scope/hook name referenced exists in the registry — **including names in `deny`** (CS-016). A deny of an undeclared name adds no protection (default-deny already refuses unknowns) and is almost always a typo that would otherwise silently arm itself as a no-op. To pre-forbid a capability, declare the action in the registry and deny it (the pattern the worked registries use for `prescribe`/`discontinue`). *(Approver `role:` names are exempt — they resolve at the identity seam, not the registry; §7.8, CS-025.)*
2. No `allow` and `deny` that *only* a human could disambiguate — `deny` always wins, but overlapping intent SHOULD warn.
3. Every `transition` action referenced has declared `from` states.
4. Actions with `reversibility == irreversible` and no `requireApproval`/`dualAuthorization`/`precondition`/`requireMatch` ⇒ **warn** (CS-038: `requireMatch` counts as a satisfying guard).
5. `failureMode: open` on an `irreversible` action ⇒ **error** unless explicitly acknowledged.
6. `'*'` grants ⇒ **warn** (encourage explicit enumeration).
7. `assess` actions with `explainability: required` but no `requireExplanation` gate ⇒ **error**.
8. Reads of `resultSensitivity > internal` with no `disclosure` gate ⇒ **warn**.
9. Condition expressions parse against the grammar (§8) and reference only known namespaces/functions.
10. A `compensable` action whose registry entry declares **no** `compensation` ⇒ **error** (the attribute value's definition, §5, is "a declared undo exists"); and any declared `compensation` that does **not** name a resource+action present in the registry ⇒ **error**. `irreversible` actions MAY declare a `compensation` but are not required to.
11. An action listed in both `deny` and a `standing` rule's `enables` ⇒ **error** — deny always wins (§6.2), so the standing grant is unsatisfiable (§7.15, CS-010).
12. A bare action name in `allow` that resolves to actions on more than one resource ⇒ **warn** — the grant applies everywhere the name is declared; use the `{ Entity: [names] }` map form to disambiguate (§6.1, CS-012).
13. `dualAuthorization` with an explicit `quorum` < 2 ⇒ **error** (contradicts the gate's definition, §7.9).
14. (v0.6, CS-038) `requireMatch.registry` names a declared obligation registry; every `obligation.*` path in `match`/`provenance`/`consume` exists in that registry's declared schema; a tolerance clause (`within`, §7.16) applies to a numeric/money field ⇒ else **error**.
15. (v0.6, CS-038) The policy grants the same agent `record`/`effect`/`transition` on the resource backing an obligation registry it matches against ⇒ **error** — creation/execution separation: the agent must not author its own obligations (§1 non-goals). Where the registry is external and the overlap is not statically visible, emit **info** pointing at the deployment check (docs/06 §5b).
16. (v0.6, CS-038) `requireMatch` with `consume: none` on an `irreversible` effect ⇒ **warn** (verification without consumption leaves the double-spend window open, §12).
17. (v0.6, CS-038) `onAmbiguous: allow` ⇒ **error** (illegal value; §7.16 — the gateway never auto-selects among candidate obligations).
18. (v0.6, CS-038) A check declared `holdCapable: true` with no declared `reasonCodes` ⇒ **error** (every hold it returns would be code-less and resolve fail, §7.6 rule 2); a hold-capable check gated with no `resolvers` and no visible deployment default resolver role ⇒ **warn** (§12, CS-027 — the warning names the deployment fallback).

---

## 14. Worked examples (non-trivial, all kinds, multiple domains)

Each example exercises several kinds and gates. Together they cover all five kinds and the full gate catalog.

### 14.1 Customer support assistant (data / business)
*All reads scoped to the user's own customers; may email within corporate domains under rate, daily-quota, and DLP limits, with a session spend ceiling on all effects; may never refund or export; anything **high-impact** needs a supervisor (approval keys on stakes — `operativeForce` — not reversibility; see §5 note).*
```yaml
apiVersion: stele/v0.1
agent: support-assistant
defaults: { failureMode: closed, audit: full }
killable: true

allow:
  - observe:    [Customer, Order, Invoice]
  - record:     [Note]
  - effect:     [sendEmail]
  - transition: { Order: [confirm] }
deny:
  - effect:     [refund, exportData]
  - transition: { Order: [cancel] }

scope:
  Customer: assignedToCurrentUser
  Order:    customerAssignedToCurrentUser

gates:
  sendEmail:
    rate: 20/hour
    quota: 200/day
    allowlist:    { field: data.recipientDomain, set: corporate-domains }
    contentCheck: dlp.basic
  effect:
    spendLimit: 25/session          # cost ceiling on all effects; stops retry storms
  Order.confirm:
    precondition: { from: [pending_confirmation] }
  '*':
    requireApproval:
      when: "action.operativeForce == high"   # stakes, not reversibility (§5 note)
      approvers: role:support-supervisor
      timeout: 30m
      onTimeout: deny
```

### 14.2 Ward nurse assistant (healthcare — observe, assess, record, effect, transition)
*Reads scoped to the nurse's ward; sealed records need break-glass; triage is an explained, confirmed assessment; administration is irreversible, enforces five-rights, and (v0.6) must match an active prescription's unconsumed schedule slot — the per-patient dose cap remains as defence-in-depth behind the prescription's own frequency; signing an order is a gated transition; prescribing is forbidden.*
```yaml
apiVersion: stele/v0.1
agent: ward-nurse-assistant
defaults: { failureMode: closed, audit: full }
killable: true

allow:
  - observe:    [Patient, Medication, Observation, Order]
  - assess:     [triage]
  - record:     [Observation]                      # e.g. vitals
  - effect:     [administer, pageOnCall]
  - transition: { Order: [sign], Encounter: [discharge] }
deny:
  - effect:     [prescribe, discontinue]
  - transition: { Medication: [prescribe] }

scope:
  Patient:     inWard(actor.ward)
  Observation: forPatientInWard(actor.ward)

gates:
  observe:
    disclosure:
      when: "action.resultSensitivity == restricted"     # e.g. psych/HIV records
      allowSink: [careTeam]
    requireApproval:
      when: "action.resultSensitivity == restricted and not exists context.breakGlass"
      approvers: role:charge-nurse
  triage:                                    # assess
    requireExplanation: true
    requireApproval: { when: "data.acuity <= 2", mode: confirm, approvers: role:clinician }
  administer:                                # effect, irreversible
    precondition: [fiveRightsVerified, notDiscontinued]
    requireMatch:                             # v0.6: the dose is OWED, not merely in-bounds (§7.16)
      registry: emr.prescriptions
      match:
        - "obligation.patientId == resource.patientId"
        - "obligation.state == 'active'"
        - "obligation.drug == data.drug"
        - "obligation.route == data.route"
        - "obligation.schedule.slotState == 'unconsumed'"
        - { field: obligation.dose, matches: data.dose, within: 0 }   # exact, stated
      consume: obligation.schedule.slot
      onNoMatch: deny
      resolvers: role:pharmacist              # two active orders -> held, pharmacist paged
    quantityCap:  { per: resource.patientId, limit: 3, window: 24h, of: data.drug }
    requireApproval: { when: "action.operativeForce == high", approvers: role:clinician }
  Order.sign:                               # transition, operative
    precondition: { from: [draft] }
    requireApproval: { approvers: role:clinician }
```

### 14.3 Air/maritime track operator (defence — observe vs emitting effect, assess, transition, gated kinetic effect)
*Passive reads are clearance-scoped with disclosure control; an active radar sweep is an emitting `effect` needing deconfliction; combat-ID is an explained, dual-confirmed assessment; engagement is enabled only under a standing ROE state and requires positive ID, a collateral ceiling, and dual authorization — outside that state it falls to default-deny (deliberately **not** an explicit `deny`, which would beat the standing grant; §7.15, §13 rule 11).*
```yaml
apiVersion: stele/v0.1
agent: track-operator-assistant
defaults: { failureMode: closed, audit: full }
killable: true

allow:
  - observe:    [Track, IntelRecord]            # passive
  - assess:     [combatId, collateralEstimate]
  - record:     [TrackAnnotation]
  - effect:     [radarSweep]                     # emits
  - transition: { Track: [identify, designate] }
# 'engage' is deliberately absent from allow AND deny: default-deny covers the
# off state, and the 'weapons-free' standing rule below is its only way in.
# (An explicit deny would beat the standing grant — §7.15, §13 rule 11.)

standing:
  - name: weapons-free
    when: "context.roeState == 'weapons_free'"
    enables: { effect: [engage] }

scope:
  Track:       inCompartment(actor.clearance)
  IntelRecord: inCompartment(actor.clearance)

gates:
  observe:
    disclosure: { maxClassification: actor.clearance }
  radarSweep:                                   # emitting effect
    emissionControl: { checks: [emconAuthorized, deconflicted] }
  combatId:                                     # assess
    requireExplanation: true
    requireApproval: { mode: confirm, approvers: role:tactical-officer }
  Track.designate:
    precondition: { from: [identified] }
  engage:                                       # effect, irreversible, kinetic
    precondition:       [positiveIdentification]
    valueLimit:         { field: data.collateralEstimate, max: 1 }   # CDE threshold
    dualAuthorization:  { approvers: role:weapons-release-authority }
```

### 14.4 Payments operations agent (finance — tiered effects, dual-auth, sanctions, transition)
*Reads tenant-scoped; small payments auto-clear, mid-size need approval, large need dual authorization and a new-payee hold; sanctioned destinations are denied; export is forbidden; (v0.6) every payment must match an open purchase-order line, which it consumes — an unmatched invoice is held for the AP clerk, not silently refused.*
```yaml
apiVersion: stele/v0.1
agent: payments-ops-agent
defaults: { failureMode: closed, audit: full }
killable: true

allow:
  - observe:    [Account, Payment, Payee]
  - record:     [LedgerEntry]
  - effect:     [pay]
  - transition: { Invoice: [send, markPaid] }
deny:
  - effect:     [exportData]

scope:
  Account: tenantOf(actor)
  Payment: tenantOf(actor)

gates:
  pay:
    denylist:   { field: data.destinationCountry, set: sanctioned-list }
    valueLimit: { field: data.amount, max: 1000000, currency: USD }
    rate:       { limit: 50/hour, per: resource.payeeId }
    requireMatch:                            # v0.6: no open order, no payment (§7.16)
      registry: erp.purchase_orders
      match:
        - "obligation.vendorId == data.vendorId"
        - "obligation.state == 'open'"
        - "obligation.line.state == 'unconsumed'"
        - { field: obligation.line.amount, matches: data.amount, within: "10%" }
      provenance:
        - "obligation.vendor.domain == data.sourceDomain"
      consume: obligation.line
      onNoMatch: hold                        # AP clerk sees "no matching PO", not a silent deny
      resolvers: role:ap-clerk
    requireApproval:
      when: "data.amount > 1000 and data.amount <= 10000"
      approvers: role:payments-manager
    dualAuthorization:
      when: "data.amount > 10000"
      approvers: role:treasury
    precondition:
      when: "exists data.newPayee"
      checks: [payeeCoolingOffElapsed]      # new-payee hold
  Invoice.markPaid:
    precondition: { from: [sent] }
```

Three decisions this example produces that no v0.5 gate could: an invoice that matches an open PO line passes limits **and** matching → allow, line consumed at settlement; the same invoice resubmitted → `no-match` (line consumed) → held; an invoice matching nothing → held for the AP clerk. A matched obligation never relaxes the tiers — the $50,000 matched invoice still takes dual authorization (§7.16 rule 5).

### 14.5 Legal matter assistant (data / business — ties to the repo demo)
*Reads scoped to the client; time entries and tasks are routine records; the `Engage` transition is legal only from `conflict_check` (the exact behaviour the repo already demonstrates); e-filing is allow-listed to approved courts, partner-approved, and confined to court hours; email is DLP-checked.*
```yaml
apiVersion: stele/v0.1
agent: legal-matter-assistant
defaults: { failureMode: closed, audit: full }
killable: true

allow:
  - observe:    [Matter, TimeEntry, Task, Staff]
  - record:     [TimeEntry, Task]
  - effect:     [sendEmail, eFile]
  - transition: { Matter: [engage, close] }
deny:
  - effect:     [deleteMatter]

scope:
  Matter:    clientOf(actor)
  TimeEntry: forMatterOfClient(actor)

gates:
  Matter.engage:
    precondition: { from: [conflict_check] }      # refuses from 'active', etc.
  eFile:
    allowlist:    { field: data.court, set: approved-court-systems }
    requireApproval: { approvers: role:supervising-partner }
    window:       { days: [Mon,Tue,Wed,Thu,Fri], hours: "08:00-17:00", tz: "America/New_York" }
  sendEmail:
    contentCheck: dlp.basic
    rate: 30/hour
```

### 14.6 Industrial vehicle controller (cyber-physical — bounded continuous effect)
*Setting target speed is inert; applying it is a safety-gated effect bounded by sensors and posted limits; the vehicle lifecycle is a transition; everything is killable.*
```yaml
apiVersion: stele/v0.1
agent: vehicle-controller
defaults: { failureMode: closed, audit: full }
killable: true

allow:
  - observe:    [Vehicle, Sensor]              # passive reads
  - record:     [Vehicle]                       # write target setpoint (inert)
  - effect:     [applySpeed]
  - transition: { Vehicle: [start, stop] }

gates:
  applySpeed:                                   # effect, continuous/safety-critical
    valueLimit:   { field: data.kph, max: 130, min: 0 }
    precondition: [surroundingsClear, withinPostedLimit, withinTractionLimits]
  Vehicle.start:
    precondition: { from: [stopped] }
```

---

## 15. Quick reference

**Kinds:** `observe` · `assess` · `record` · `effect` · `transition`
**Attributes:** `reversibility` · `emission` · `operativeForce` · `resultSensitivity` · `explainability`
**Gates:** `rate` · `quota` · `valueLimit` · `spendLimit` · `allowlist`/`denylist` · `precondition` · `contentCheck` · `requireApproval` · `dualAuthorization` · `window` · `quantityCap` · `disclosure` · `emissionControl` · `requireExplanation` · `requireMatch`
**Decisions:** `allow` · `hold` · `deny` · `halt`
**Top-level keys:** `apiVersion` · `agent` · `extends` · `defaults` · `allow` · `deny` · `scope` · `gates` · `standing` · `killable` · `audit`
**Precedence:** default deny → deny wins → most-specific allow → all matching gates AND → kill-switch → execute → record.
**Frozen:** the five kinds, the five attribute names, the fifteen gate types, and the condition operators/functions. The gate catalog's only amendment since v0.1 is v0.6's `requireMatch` (CS-032, argued in the change set); the grammar's only change remains CS-013 (the right side of `in` accepts a function) — the v0.6 tolerance comparison is a structured gate field (CS-033), not an operator. Growth is by adding resources, actions, scope predicates, named sets, hooks, and obligation registries — never new language constructs; the shape re-freezes at fifteen.
