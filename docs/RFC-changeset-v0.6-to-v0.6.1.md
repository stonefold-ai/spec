# RFC Change Set — v0.6 → v0.6.1 (OPEN)

**Status: OPEN — the current draft set.** Items are mirrored in the `docs/01` changelog
and the RFC body text; on any conflict with older wording, a Change Set wins (same rule
as prior sets). One item deferred from v0.6 is expected to join this set when deployment
evidence exists: the per-principal open-hold budget (`hold-budget-exhausted`, CS-031's
second half).

---

## CS-040 — Hold dedupe identity sharpened (FIXED, §12)

**What:** a hold's dedupe identity (CS-031) becomes, per holding gate:

> (agent, action, **reason code**, the gate's **evidence** — which carries the
> matched-candidate refs — and the **values of the intent fields the gate compared**,
> its CS-030 `fields` set)

Two holds collapse into one queue item only when every component matches within the
deployment's dedupe window. Comparing evidence canonically (order-insensitive) and
intent-field values by their `data.*` values keeps the two directions asymmetric on
purpose: **over-distinguishing degrades safely** (one extra queue item), while
over-collapsing loses a question.

**Why:** v0.6's key was (agent, action, reason code, matched-candidate refs) — and with
**zero** candidates the refs are empty, so every `no-match` hold on one action shared a
key: two *different* vendors' unmatched invoices collapsed into one queue item, and
resolving it addressed only the first. That over-collapses exactly where the operator
needs separate questions. The motivating case still holds under the sharper key: ten
resubmissions of the **same** unmatched invoice carry the same compared values and
collapse into one item with an attempt count. A check whose evidence contains volatile
content simply dedupes less — the safe direction.

**Implementation impact:** the dedupe-key function gains the evidence and compared-value
components; no schema change, no new syntax. Registered checks that want precise dedupe
SHOULD put what identifies their question (the target id, the candidate set) in the
hold's evidence. TCK: the J6 check gains the negative half — two holds over *distinct*
targets MUST NOT collapse.

---

## CS-041 — Closure accountability: the standard `dispositionIsDeclared` check (ADDED, §7.6, registry §5c)

**What:** two additions that work together, and neither adds a gate or a kind.

A registry MAY declare, on an action that **closes a unit of work**, what closing it
means:

```yaml
Document.markWorked:
  kind: record
  data:
    disposition: { values: [resolved, escalated, referred, duplicate], required: true }
  closure:
    dispositionField: disposition       # which data field carries the disposition
    claimsCompletion: [resolved]        # the values that assert the work was DONE
```

And the spec defines its first **standard check** — a named `precondition` whose
semantics are normative rather than integrator-defined:

```yaml
markWorked:
  precondition:
    checks: [dispositionIsDeclared]
    resolvers: role:supervisor
```

`dispositionIsDeclared` resolves, in order:

1. **The action declares no `closure`** → the policy is invalid; the linter refuses it at
   load (§13.20). At runtime, fail-closed under §10.
2. **`data[dispositionField]` is absent, empty, or not one of the declared values** →
   **hold**, reason code `DISPOSITION_REQUIRED`, retry class `retryable`. The actor may
   resubmit with a declared disposition; nothing has been closed.
3. **The disposition is in `claimsCompletion`, and this gateway refused (`deny`, `halt`
   or `hold`) an intent from this actor earlier in this run** → **hold**, reason code
   `CLOSED_WITHOUT_THE_WORK`, retry class `escalate`, carrying the refused actions as
   evidence. A human decides.
4. **Otherwise** → pass. Note what this leaves open on purpose: a disposition *outside*
   `claimsCompletion` — `escalated`, `referred` — always passes rule 3. The actor keeps a
   way to be honest about an item it could not complete.

**Why:** the gateway's control fires, the action does not happen, and the actor then
closes the item as done. The refusal is now invisible: the queue is empty, the managed
system's completeness check passes, and the next human sees nothing to look at. The
mirror failure is just as bad — an actor that correctly declines to close anything leaves
a row nobody touched, which is byte-identical to an actor that crashed. **Both are
illegible, and one control covers both**, because a disposition vocabulary gives the
actor a way to say *I looked at this and it is not mine to close* and gives the gateway a
way to refuse the claim that contradicts its own record.

This is a claim about the **record**, not about completeness. The gateway cannot know
whether work was done; it knows what passed through it. Stated exactly: *nothing is
closed as done while this gateway holds a refusal for the same actor in the same run.*

**The boundary, and it is deliberate.** The check reads **only what this gateway
witnessed** — its own audit of its own traffic. It needs no due dates, no ledger, and no
knowledge of what the item is. An item that never reached the gateway is invisible to it
and stays the managed system's responsibility.

**The scope limitation, stated rather than papered over.** The check is scoped to the
**run**, not to the item, because an intent does not cite the item it is acting for
(`Supplier.updateBankAccount` names a supplier, not the request that asked for it). So it
is exact where a run handles one item and over-broad where a run handles forty: a
truthful closure can be held because something *else* in the same run was refused. The
failure direction is a hold on an honest closure, never a silent false one, and an
honest disposition still passes. A deployment needing per-item precision must have the
actor cite the item it is closing — a change to the managed system's API, outside this
spec.

**Implementation impact:** `closure` is a new optional action-level declaration in
`registry.schema.json`; no Stele syntax changes, since the check is named where any
`precondition` check is named. A gateway MUST be able to answer "what did I refuse for
this actor in this run" from its own audit, and MUST NOT satisfy rule 3 from any other
source. Where it cannot answer, §10 fail-closed applies — the case a guard-unavailable
disposition would improve, and that disposition is not in this set. New lint rule
§13.20. TCK: profile `closure`, checks N1–N4 (one per rule).
