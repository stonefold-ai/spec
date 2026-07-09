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
