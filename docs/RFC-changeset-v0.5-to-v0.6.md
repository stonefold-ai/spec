# RFC Change Set — v0.5 → v0.6 (ACCEPTED)

**Status: ACCEPTED 2026-07-09.** Items CS-026 through CS-039 are mirrored in the
`docs/01` changelog and the RFC body text; on any conflict with older wording, a Change
Set wins (same rule as prior sets). The acceptance conditions were met the same day the
set closed: the reference implements every item, and the TCK certifies the four new
profiles (`hold-precondition`, `feedback`, `match`, `consume`) in-process and over the
wire binding. The per-principal open-hold budget (CS-031's second half) is deferred to
v0.6.1.

**Scope of the change.** This set adds the third clause of the layering. SIF says *what
can be said*; Stele says *what is allowed*; v0.6 adds *what is owed*: a `requireMatch`
gate validates an intent against a **registered obligation** — a record created earlier,
by a different authority, in a system the agent cannot write to — and consumes it on
settlement. Supporting it: preconditions gain a `hold` verdict (the judgment-shaped
residue routes to a human instead of a bare deny), holds gain sound composition and real
expiry, and the agent's feedback channel becomes specified (reason codes with retry
classes, visibility control) so an iterating agent can converge without the deny channel
becoming an oracle.

**One deliberate amendment to the frozen shape, argued in CS-032:** the gate catalog
grows from fourteen to fifteen. Everything else is additive within the growth rule —
no new kinds, no new attribute names, and **no change to the condition grammar** (the
tolerance comparison deliberately takes a structured field form, CS-033, precisely to
keep §8 frozen). Existing `apiVersion: stele/v0.1` policy files remain valid as-is.

**Non-goals (stated here once, mirrored in §1):** Stonefold is not a system of record —
it persists match decisions and consumption receipts in the audit log, never obligations.
`requireMatch` verifies **entitlement, not judgment**: it establishes that an intent
corresponds to a registered obligation within tolerance; it cannot establish that the
obligation itself is legitimate — a fraudulent purchase order passes. That risk lives
upstream, at obligation creation, where existing organisational controls (separation of
duties, approval workflows) already sit — and CS-038 rule 15 exists precisely so the
governed agent cannot become its own upstream. Where no registry can exist, **the human
is the registry**: `requireApproval`/`dualAuthorization` remain the honest floor, and
`requireMatch` does not replace them.

---

## Part I — the hold substrate

## CS-026 — Preconditions may resolve `hold` (ADDED, §7.6, §12)

**What:** a named precondition check now resolves **pass / fail / hold** instead of
pass/fail. `hold` suspends the intent for human resolution, reusing the existing held
lifecycle (staging as pending-approval, §4.4) — no new lifecycle state. Three rules bound
it:

1. **Outages fail; only readable ambiguity holds.** A check that cannot reach its source
   MUST resolve `fail`, never `hold`; a check that raises is a dependency failure
   (`failureMode`, §10), never a hold. `hold` is legal only when the data was read
   successfully AND the result is judgment-shaped (multiple matching records, a
   near-tolerance value, a field that means "ask a human").
2. **A hold carries a machine-readable reason code** (CS-029) or it is invalid: the
   gateway MUST treat a hold without a reason code as a check implementation error —
   resolve `fail`, log loudly. The human resolving the hold must see *why* it paused
   without reading check code.
3. **Opt-in per check.** Two-valued checks remain valid indefinitely; a check declares
   hold capability in its registry declaration (CS-029, registry §5).

The gate-catalog row for `precondition` becomes pass/fail/hold. At dispatch-time
re-validation (CS-017) nothing changes: `precondition` is already volatile, and a
non-PASS at the claim — including a fresh `hold` — settles `CANCELLED`/`stale-guard:precondition`
(a claimed row is never re-suspended).

**Why:** preconditions are the seam where intents are checked against external records
(the obligation pattern, Part II). Two worked cases cannot be expressed with pass/fail:
two active prescriptions for the same drug (the check must refuse to pick one — but a
bare deny lands on a nurse at a bedside when the correct outcome is "paused, pharmacist
paged"), and an invoice matching no order (better held for the AP clerk's queue than
refused and resubmitted in a loop). Rule 1 exists because if outages hold, every registry
blip becomes a human interruption and fail-closed silently degrades into fail-to-a-queue
that gets rubber-stamped. Scarce holds get read; constant holds get ignored.

**Implementation impact:** hook result type extended (a plain boolean stays accepted);
the gateway's check-invocation wrapper enforces rules 1–2 (a crash maps to fail, a
code-less hold maps to fail); pipeline maps the hold into the existing held path with
cause `precondition:<name>`. TCK: `hold-precondition` profile (Part IV note).

## CS-027 — Multi-hold release: every holding gate is satisfied (ADDED/FIXED, §12, §4.4)

**What:** when more than one gate resolves `hold` for the same intent, the staged row
carries the **release contract of every holding gate**, and promotion to dispatch
requires **each contract satisfied**. A release contract is what the holding gate
demands: for `requireApproval`, its approvers/quorum/timeout; for `dualAuthorization`,
two distinct non-actor identities; for a holding `precondition` or `requireMatch`, the
gate's **`resolvers:`** field (a `role:` name, resolving at the identity seam like
approvers, §7.8) — or, when absent, the deployment's **default resolver role**
(gateway configuration, like decision TTLs; never policy syntax). A hold whose contract
names no approver/resolver and has no deployment default is a configuration error: the
gateway MUST refuse it as fail-closed DENY at decision time, audited
(`hold-unresolvable`), rather than stage a row anyone could release.

Satisfying one contract never satisfies another; a resolver identity releasing a
precondition hold is recorded like an approver (who, when, which contract). Resolution
is not a bypass: the gates all evaluated at decision time, promotion requires every
contract, and dispatch re-validates the volatile gates as always (CS-017).

**Why:** v0.5 underspecified the composed case, and the reference derived the release
contract from the *first* holding gate only — with CS-026 that becomes an approval
bypass: a precondition hold (evaluated before approvals) would carry an empty contract,
and one release from anyone — including the acting principal — would promote a row that
`requireApproval` also held. This item closes that composition hole for the existing
approval gates too (a `requireApproval` and a `dualAuthorization` holding together now
each bind).

**Implementation impact:** the staged row's single approval contract becomes a list of
release contracts; `approve` satisfies the matching contract; promotion when all are
satisfied. Schema: optional `resolvers` on `precondition`, `emissionControl`, and
`requireMatch` gate objects. Linter: warns when a hold-capable check is gated with no
`resolvers` (the deployment default may cover it — the warning says which).

## CS-028 — Held-row expiry is enforced (ADDED/FIXED, §7.8, §12)

**What:** a held row expires by whichever comes first of (a) the holding gate's declared
`timeout` and (b) the row's staging TTL (CS-017) — and expiry is **acted on**, not merely
implied: the gateway MUST run an expiry sweep over held rows (the dispatch worker's loop
is sufficient) that settles an expired hold `CANCELLED`/`expired-hold:<gate>`, preserving
the original hold reason code in the audit record. Both deadlines are measured from
staging, **on the same clock that stamped the staging TTL** (the decision-time clock) —
anchoring them on different clocks makes the earlier-of comparison meaningless under skew. `onTimeout: deny` (the default)
cancels; `onTimeout: allow` promotes the row **iff every other release contract is
satisfied** (CS-027) — it satisfies its own contract only. CS-017's rule is unchanged and
remains the backstop: a late release cannot resurrect an expired row; the intent must be
re-submitted and re-decided.

**Why:** v0.5 declared `timeout`/`onTimeout` on approval gates but specified no
mechanism; the reference recorded them in the audit and enforced nothing — dead syntax.
Lazy expiry (cancel at claim, after a late approval) leaves held rows sitting in queues
indefinitely, which CS-031's hygiene and any operator inbox need not to happen.

**Implementation impact:** an expiry check in the worker loop; `expired-hold:<gate>` is a
normative settle reason (like `stale-decision`); audit record carries the original
reason code. Existing behaviour for non-held staged rows is unchanged.

## CS-029 — Reason codes carry a retry class (ADDED, hooks, §11, registry §5)

**What:** every deny/hold reason a gate or registered check produces is a
**machine-readable code** with a declared **retry class**:

- `retryable` — the defect is in the intent; fix it and resubmit (amount outside
  tolerance, missing field, rate window busy).
- `terminal` — nothing about this intent family can be fixed by the agent; do not
  resubmit (no obligation exists, denylisted destination, unknown action).
- `escalate` — stop and surface to a human on the **agent's** side (distinct from a
  gateway hold, which queues on the operator's side).

The class is returned to the agent with the code (subject to CS-030 visibility — the
class itself is always visible). A code with no declared class defaults to `terminal`
(the safe direction: stop retrying). Registry declarations of precondition checks grow
from bare names to an optional object form:

```yaml
preconditionChecks:
  - payeeCoolingOffElapsed                    # bare form stays valid (two-valued, codes default terminal)
  - name: matchesOpenPurchaseOrder
    holdCapable: true
    reasonCodes:
      no-open-match:      terminal
      amount-outside-tolerance: retryable
      multiple-candidates: escalate           # hold-shaped: the check returns hold with this code
```

Built-in gate reasons get normative classes: `valueLimit`, `rate`, `quota`,
`quantityCap`, `spendLimit`, `window`, `contentCheck`, `requireExplanation` ⇒
`retryable`; `allowlist`, `denylist`, `disclosure`, unknown-action, scope
refusals ⇒ `terminal`; `stale-decision` and `stale-guard:<gate>` ⇒ `retryable`
(resubmit for a fresh decision); `expired-hold` ⇒ `escalate`;
`hold-unresolvable` ⇒ `escalate` (a configuration error a human must fix). An
approval-shaped hold with no check code carries no class — the agent's move
there is to wait, which none of the three classes means.

**Why:** the local agentic loop converges because a compiler error is specific and
in-principle fixable by editing. A gateway deny is not always: "amount outside
tolerance" is fixable; "no order exists" is not — the fix is a human conversation.
Without the distinction agents loop on unfixable denies until the rate gate throttles
them: a crude, uninformative stop. Three enum values make the deny channel a usable
convergence signal.

**Implementation impact:** a reason-code value type threaded through gate results and
check results (additive); registry schema gains the object form; linter rule 18
(CS-038). The classes for built-ins are a table in §7, not per-policy syntax.

## CS-030 — Agent feedback visibility (ADDED, §11)

**What:** what the *agent* receives on a deny or hold is now a declared choice, not an
accident. Per gate config or per check declaration, an optional `feedback:` key:

- `code` — reason code + retry class only.
- `code+fields` — **the default**: code, class, and *which* intent fields failed the
  comparison — never the record-side values they were compared against.
- `code+evidence` — the full comparison including record-side values; for trusted
  internal loops only.

The audit record always carries everything (§11 is unaffected); visibility governs the
agent-facing `EvalResult` only — redact on return, never on write. Reason codes remain
an oracle even at `code` (each probe maps one wall), so the countermeasure is
detection, not blunting: deny-rate and reason-code distribution per agent principal
SHOULD be surfaced by audit tooling — a converging loop and a mapping loop look
different in that data.

**Why:** the channel silently decides the loop's convergence *and* the policy's
leakage. Left unspecified, implementations default to one extreme: the reference
returned the full gate trace with prose reasons — maximally convergent, maximally
leaky. `code+fields` keeps the loop convergent (the agent learns *what to fix*) without
handing over the registry's data.

**Implementation impact:** redaction at the transport boundary; schema gains the
`feedback` key on gate sets; the reference default changes from full-trace to
`code+fields` (a breaking change for agents that parsed prose reasons — called out
here deliberately).

## CS-031 — Hold dedupe (ADDED, gateway behaviour)

**What:** holds that are the same question MUST NOT queue as separate items: a new hold
whose (agent, action, reason code, matched-candidate refs) equal an already-open held
row's within the deployment's dedupe window collapses into that row — attempt count
incremented, latest intent recorded, audit record written per attempt as always. Ten
variants of the same unmatched invoice are one question wearing ten disguises; the
operator sees one queue item with `attempts: 10`.

A per-principal ceiling on *open* holds (beyond which hold-shaped results resolve
terminal deny, audited `hold-budget-exhausted`) is **deferred to v0.6.1** — dedupe
removes the demo-visible failure mode; the budget waits for deployment evidence on
where the ceiling should sit.

**Why:** denies are free (pre-commit, no side effect); holds spend human attention —
the scarcest resource in the system, and the one whose depletion (rubber-stamping)
silently disables the safety property CS-026 adds.

**Implementation impact:** gateway-side matching at stage time; check code untouched;
dedupe window is deployment configuration.

---

## Part II — obligation matching

## CS-032 — The `requireMatch` gate (ADDED, §7.16 — frozen-shape amendment)

**What:** a fifteenth gate. An intent MUST correspond to **exactly one open obligation**
in a declared obligation registry (CS-034), within declared tolerance; the obligation is
reserved at staging and consumed at settlement (CS-035). Applies to `effect`,
`transition`, and `record` actions; resolves pass / fail / hold.

```yaml
gates:
  pay:
    requireMatch:
      registry: erp.purchase_orders            # declared obligation registry (CS-034)
      match:
        - "obligation.vendorId == data.vendorId"
        - "obligation.state == 'open'"
        - "obligation.line.state == 'unconsumed'"
        - { field: obligation.line.amount, matches: data.amount, within: "10%" }   # CS-033
      provenance:
        - "obligation.vendor.domain == data.sourceDomain"
      consume: obligation.line
      onNoMatch: deny              # deny (default) | hold
      onAmbiguous: hold            # hold (default; `allow` is not a legal value)
      resolvers: role:ap-clerk     # who releases a hold this gate raises (CS-027)
```

Fields: `registry` (MUST name a declared obligation registry; unknown ⇒ load error),
`match` (MUST; a conjunction of §8 condition-grammar comparisons between `obligation.*`
and the §8 namespaces, plus tolerance clauses per CS-033), `provenance` (MAY; additional
conjunction binding the matched obligation's counterparty fields to the intent's declared
source evidence — kills the valid-but-wrong-pointer class), `consume` (MUST for
`effect`/`transition`; MAY be `none` for pure verification on `record`), `onNoMatch`
(`deny` default | `hold`), `onAmbiguous` (`hold` default; the gateway MUST NOT
auto-select among candidates — `allow` is rejected at load), `resolvers` (MAY; CS-027).

Semantics:

1. **Deterministic evaluation at decision time** (§12 step 4). The gateway queries the
   registry with the typed selector derived from `match`. Candidate count 1 ⇒ the
   remaining clauses evaluate against that obligation; 0 ⇒ `onNoMatch`; >1 ⇒
   `onAmbiguous`. No model output participates.
2. **Agent-independent read path (CS-036).** Every `obligation.*` operand resolves from
   the registry's response, never from the intent.
3. **Volatile gate** — with a refinement: because the obligation is *reserved* at
   staging (CS-035), dispatch-time re-validation checks **reservation liveness** (still
   held, registry reachable) instead of re-running the query. Reservation lost ⇒
   `CANCELLED`/`stale-guard:requireMatch`.
4. **Runtime resolution (CS-005 applies).** An `obligation.*` path absent/null on the
   matched record fails the gate closed. Registry unreachable ⇒ `failureMode` (§10);
   for `irreversible` effects this MUST resolve closed.
5. **Composition unchanged** (§7): AND with all other matching gates. A matched
   obligation never relaxes a `valueLimit`, a list, or an approval.

**Why — the in-bounds wrong action.** v0.5's gates compare intent fields against
constants the policy carries. That bounds damage; it cannot establish that an action is
*owed*. The residual failure class: `pay` $800 to a known payee, under `valueLimit`,
inside `rate`, passing `denylist` — for an invoice that corresponds to no purchase, or
one already settled. Every gate passes; the action is wrong. Correctness here is a
**relation** between the intent and a pre-existing record: an invoice is payable only
against an open order; a dose is administrable only against an active prescription. Such
records already exist in domain systems and share three properties that make them usable
as ground truth: **temporal precedence** (created before the intent — the agent cannot
author its own justification), **independent authorship** (a different authority,
through a different channel), and **consumability** (each record entitles a bounded
number of executions, after which it is spent).

**Why a new gate type (the frozen-shape argument).** Three attempts to express this
inside the growth rule fail. As a `precondition` hook: even with CS-026's three verdicts
the gate's decision *inputs* (which registry, which match conjunction, which tolerance)
would live in check code — invisible to the reviewer, the linter, and the TCK, when the
whole point of Stele is that "payment requires an open order" is *in the policy*. As a
`contentCheck`: a content hook judges the payload; matching judges the payload against
external mutable state with a consumption side effect. As connector-internal logic: same
review-invisibility, plus the decision escapes the audit record. Beyond the verdict
shape, `requireMatch` has a property no existing gate has — **a stateful side effect
bound to the effect's own lifecycle** (reserve at staging, consume at settle, release on
cancel), composing the counter pattern (consumed at decision, CS-017) with the CS-018
in-transaction reassertion pattern while fitting neither gate's schema. Hence: one new
gate, fully specified, and the shape re-freezes at fifteen.

**The gate is the ceiling, not the floor (adoption path).** The same protection runs on
any gateway as a plain registered `precondition` check — query the source, compare typed
fields, fail closed, let the record system mark things spent — and §7.16 says so in
normative text, with the five rules such a check follows and the specific reasons to
upgrade (match rule in the policy; ambiguous ⇒ hold with a named resolver;
gateway-managed consumption where the record system doesn't enforce spending;
standardized audit lineage). A deployment that starts with the check and never upgrades
is using the feature correctly. This is deliberate adoption engineering: the pattern
first, the syntax when the pattern's visibility or its double-spend window actually
bites.

**Implementation impact:** new gate function (volatile band); schema gains the
`requireMatch` key; linter rules 14–17 (CS-038); TCK `match` profile.

## CS-033 — Tolerance clauses are structured fields, not grammar (ADDED, §7.16)

**What:** approximate equality under declared tolerance is expressed as a **structured
match clause**, legal only inside `requireMatch.match`:

```yaml
- { field: obligation.line.amount, matches: data.amount, within: "10%" }   # relative
- { field: obligation.dose,        matches: data.dose,   within: 0 }       # exact
```

`field` is an `obligation.*` path; `matches` is a §8 namespace path; `within` is a
percentage string (`"N%"`, relative to the obligation-side value) or an absolute number
in the field's unit. `within: 0` means exact — useful where the policy reviewer wants
"exact" stated rather than implied by `==`. Tolerance applies to numeric/money fields
only (linter rule 14).

**Why:** the alternative — a `~=` operator in the condition language — would amend the
frozen grammar (§8 has been unchanged since v0.1 except CS-013's widening of `in`).
One frozen-shape amendment per change set is the budget this set spends on the gate
itself; the structured form has identical expressiveness, is easier to validate in
JSON Schema, and reads adequately in review. The condition grammar stays frozen.

**Implementation impact:** schema `oneOf` on match-list items (string | tolerance
clause); the match evaluator handles the clause form natively.

## CS-034 — Obligation registries (ADDED, registry doc §5b)

**What:** a new registry declaration class alongside entities and named functions:

```yaml
obligationRegistries:
  erp.purchase_orders:
    connector: erp-po-adapter
    digest: "sha256:…"                    # CS-020 pinning applies
    capability: transactional             # transactional | window (CS-018 semantics)
    schema:                               # typed; free-text fields are never match inputs
      vendorId: { type: string }
      state:    { values: [open, closed, cancelled] }
      vendor:   { type: object, properties: { domain: { type: string } } }
      line:     { type: object, properties:
                    { amount: { type: decimal },
                      state:  { values: [unconsumed, reserved, consumed] } } }
```

The adapter behind the connector implements four operations (connector contract —
gateway-code metadata like CS-018's capability, not policy syntax):

```
query(selector)                  -> [Obligation]        # typed records only
reserve(ref, intent_id)          -> Reserved | AlreadyReserved | AlreadyConsumed
consume(ref, intent_id)          -> ConsumeReceipt | AlreadyConsumed
release(ref, intent_id)          -> Released | NotHeld
```

All four are **idempotent per (obligation ref, intent id)**. The declared `capability`
states how `consume` composes with the effect's settlement (CS-035). The registry is
part of the trusted computing base (CS-019 wording applies); digest pinning and
deployment discipline are its integrity story. **The governed agent's principal MUST NOT
hold write access to any registry its policy matches against** — an agent that can
create orders and then pay against them approves itself. Enforced at deployment; linted
where statically visible (CS-038 rule 15).

**Why:** the gate needs a declared, typed, reviewable source — "which system of record,
which fields, which consistency guarantee" belongs in the registry with everything else
the linter validates names against, not in check code.

**Implementation impact:** registry schema + loader; an obligation-registry protocol
with an in-memory reference adapter; docs/06 gains the section.

## CS-035 — Reservation lifecycle (ADDED, §12)

**What:** obligation state tracks the staged effect's lifecycle exactly, or the gate
re-opens the races v0.4/v0.5 closed:

- **Decision (step 4):** query + match; on pass, the candidate's ref enters the decision
  record.
- **Staging (§4.4 commit):** the gateway calls `reserve(ref, intent_id)`, and the
  reservation MUST be in place before the staging commit returns *accepted/pending* —
  reservation is what prevents a second intent from matching the same line during the
  decide→dispatch gap (the double-spend window the decision TTL alone does not close).
  `AlreadyReserved`/`AlreadyConsumed` here ⇒ the intent settles refused `no-match`,
  audited.
- **Dispatch claim:** order per CS-017, extended — kill → TTL → volatile gates
  (including reservation liveness, CS-032 rule 3) → connector.
- **Settlement:** `consume(ref, intent_id)` executes with the settle, per the registry's
  declared capability: **transactional** ⇒ inside the same transaction as the effect's
  commit and the CS-006 audit write — no consumed-without-effect, no
  effect-without-consumed; **window** ⇒ immediately after connector confirmation, with
  the declared residual window surfaced in the audit record (the CS-018 pattern: priced,
  not hidden).
- **Release:** any terminal non-success — `CANCELLED` (kill, `stale-decision`,
  `stale-guard`, `expired-hold`, rejection) or `FAILED` — MUST release the reservation.
  A TTL expiry therefore frees the obligation for a re-submitted intent.
- **Orphan recovery:** a crash between `reserve` and the staging commit leaves a
  reservation with no row. Every reservation therefore carries a **TTL of its own**,
  agreed with the adapter and **at least** the staged row's decision TTL; the adapter
  MUST expire orphaned reservations, and the gateway's release path is idempotent so a
  recovery release of an already-expired reservation is a no-op (`NotHeld`). A
  deployment SHOULD run reconciliation (reservations ↔ staged rows) on gateway restart.
  The reservation TTL runs on the **adapter's** clock, not the gateway's; the gateway
  MUST tolerate the skew — a lost reservation is caught by the dispatch liveness check,
  an expired-but-unclaimed one MAY be re-acquired by the same intent at that check, and
  releasing an adapter-expired reservation is the `NotHeld` no-op.
- **Batches (CS-023):** reservations for all operations in a batch are taken with the
  batch's atomic staging; any refused operation refuses the batch and releases every
  reservation taken for it.
- **Idempotency:** `reserve`/`consume`/`release` are idempotent per (ref, intent id); a
  retry never double-consumes; a second **distinct** intent against a consumed line
  resolves `no-match`.

**Why:** consumability is the property that makes an obligation ground truth ("each
record entitles a bounded number of executions"); without reservation, one order line
validates as many payments as fit in the decide→dispatch gap. The orphan rule exists
because "reserve before the commit returns" plus a crash is otherwise a real PO line
locked forever by a gateway that no longer remembers why.

**Implementation impact:** staging/dispatch/settle paths in the outbox; fault-injection
tests (crash between reserve and commit; double-submit; retry-after-timeout; TTL expiry
frees the line); TCK `consume` profile.

## CS-036 — Agent-independent read path (ADDED, §7.16, §12)

**What:** obligation fields used for matching MUST be read by the gateway from the
registry. Agent-supplied copies of obligation data are never match inputs; the intent
contributes only its own fields and, optionally, an obligation **pointer** (e.g.
`data.poId`) — the gateway then queries by id and still evaluates the full `match`
conjunction against the re-read record. A pointer narrows the query; it never
substitutes for it.

**Why:** an agent that supplies the data it is checked against checks itself. This is
invariant 3 (scope below the model) applied to the obligation side.

**Implementation impact:** the match evaluator's namespace for `obligation.*` is
populated exclusively from the adapter response; enforced by construction, tested by the
TCK (`match` profile: a forged obligation copy in `data.*` changes nothing).

## CS-037 — Audit extension: entitlement lineage (ADDED, §11)

**What:** two fields at audit level `full`:

| Field | Description |
|---|---|
| `obligationRefs` | Registry + obligation/line id(s) matched, with the candidate count (`1`, `0`, or `n>1` for a held-ambiguous decision). The entitlement-side lineage key, complementing `resultRefs` (CS-009): `resultRefs` locate what the effect **produced**; `obligationRefs` locate what **entitled** it. |
| `consumption` | `reserved` \| `consumed` \| `released`, with the consume receipt id; for `window` registries, the declared residual window. |

Repeated `ambiguous` outcomes on one registry SHOULD be surfaced by audit tooling: they
are a signal of near-duplicate injection into the obligation store.

**Why:** a wrong-but-allowed payment is found through what entitled it as often as
through what it produced; reconciliation needs both ends of the relation.

**Implementation impact:** additive audit fields; populated by the pipeline (decision)
and the settle path (consumption).

## CS-038 — Linter rules 14–18; rule 4 amended (ADDED, §13)

14. `requireMatch.registry` names a declared obligation registry; every `obligation.*`
    path in `match`/`provenance`/`consume` exists in the registry's declared schema;
    a tolerance clause (`within`) applies to a numeric/money field ⇒ else **error**.
15. The policy grants the same agent `record`/`effect`/`transition` on the resource
    backing an obligation registry it matches against ⇒ **error** (creation/execution
    separation — the agent must not author its own obligations). Where the registry is
    external and the overlap is not statically visible, emit **info** pointing at the
    deployment check (CS-034).
16. `requireMatch` with `consume: none` on an `irreversible` effect ⇒ **warn**
    (verification without consumption leaves the double-spend window open).
17. `onAmbiguous: allow` ⇒ **error** (illegal value, CS-032).
18. A check declared `holdCapable: true` with no declared `reasonCodes` ⇒ **error**
    (CS-026 rule 2 would fail every hold it returns); a hold-capable check gated with no
    `resolvers` and no visible deployment default ⇒ **warn** (CS-027).

Rule 4 (irreversible + no `requireApproval`/`dualAuthorization`/`precondition` ⇒ warn)
is amended: `requireMatch` counts as a satisfying gate.

**Why:** each rule closes a hole another CS opens; 15 is the one that keeps the whole
construction honest.

**Implementation impact:** five rules + one amendment in the linter; TCK `lint` checks
extended.

## CS-039 — Worked examples and the honesty block (DOCS, §14, §1)

**What:** §14.4 (payments) gains `requireMatch` against `erp.purchase_orders` on `pay`
with `onNoMatch: hold` (the AP clerk sees "no matching PO", not a silent deny); §14.2
(clinical) gains `requireMatch` against `emr.prescriptions` on `administer` with
`onNoMatch: deny` and exact-dose tolerance — `quantityCap` remains as defence-in-depth
behind the prescription's own schedule (the policy stops proxying the obligation and
starts matching it). The demo beats the examples encode: an invoice that matches an open
PO line passes limits **and** matching → allow, line consumed; the same invoice
resubmitted → `no-match` (line consumed) → hold; an invoice matching nothing → hold.
The third refusal is the one no v0.5 gate could produce. The non-goals block (top of
this document) lands in §1.

**Why:** the worked examples are the RFC's fixtures; the honesty block is the boundary
statement every reviewer asks for (CS-019 precedent).

**Implementation impact:** `examples/payments.registry.yaml` +
`examples/payments-ops.stele.yaml` updated (they remain schema-valid fixtures); RFC text.

---

## Part III — compatibility

Additive throughout: no existing key changes meaning; existing policies, registries, and
`apiVersion` strings remain valid. Behavioural deltas an upgrading deployment sees:
(1) the agent-facing default feedback becomes `code+fields` (CS-030) — agents that
parsed prose reasons must switch to codes; (2) held rows now expire actively (CS-028) —
queues that relied on holds sitting forever will see `expired-hold` cancellations;
(3) duplicate holds collapse (CS-031). All three are called out as intended.

## Part IV — conformance

New TCK capabilities/profiles (specified in docs/12 §7 style, delivered with the
reference certification): `hold-precondition` (hold raised/released/rejected/expired;
multi-hold contracts each bind; code-less hold ⇒ fail), `feedback` (visibility levels;
retry classes returned; audit unaffected by redaction), `match` (match/no-match/
ambiguous; pointer-never-substitutes; forged obligation copy ignored; fail-closed on
unreachable registry for irreversibles), `consume` (reserve at staging; double-spend
refusal; release on cancel/expiry; idempotent retries). The kit ships a mock obligation
registry adapter so certification stays black-box. Not black-box observable (stated per
docs/12 §5 honesty rule): the transactional-consume atomicity itself — asserted through
its observable consequence (no consumed-without-effect under injected dispatch failure);
keep a fault-injection test in your own suite.
