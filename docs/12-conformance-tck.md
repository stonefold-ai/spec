# 12 — The Stonefold Conformance Test Kit (TCK)

*How a gateway — in **any language** — proves it conforms to the Stele specification.*

The TCK (`src/stonefold_tck/` in the [reference repo](https://github.com/stonefold-ai/stonefold)) is an implementation-independent, black-box test suite: the kit ships alongside the reference for convenience, but its core imports nothing from it. You do not port the reference gateway's tests; you implement ONE small adapter — the **driver** — and the kit runs every acceptance scenario against your gateway, then reports which **conformance profiles** you certify. The Python reference implementation is certified by the same kit (`tests/test_tck_reference.py`), both in-process and through the wire binding.

---

## 1. How to certify a new gateway (the short version)

**If your gateway is Python:** implement the `stonefold_tck.driver.ConformanceDriver` protocol (a few hundred lines of test-only glue — `stonefold_tck/adapters/reference.py` is the worked example, including its fixture-dialect converter and the required mocks) and run:

```python
from stonefold_tck import run_conformance
from my_gateway.tck_adapter import MyDriver

report = run_conformance(MyDriver(), implementation="my-gateway 0.1")
print(report.render())
```

**If your gateway is Java / Go / Rust / anything else:** expose the **TCK harness API** (§6) in a *test build* of your gateway — twenty-one small JSON endpoints — start it, and run:

```python
from stonefold_tck import run_conformance
from stonefold_tck.http_driver import HttpDriver

driver = HttpDriver("http://localhost:9099")
report = run_conformance(driver, implementation=driver.implementation_name())
print(report.render())
```

Either way the output is the same report:

```
Stonefold TCK conformance report -- implementation: my-gateway 0.1
[core]    CERTIFIED -- 13 pass, 0 fail, 0 skip
[lint]    CERTIFIED -- 7 pass, 0 fail, 0 skip
...
Certified profiles: core, lint, scope, staging, kill, audit, freshness, batch, digest, hold-precondition, feedback, match, consume
```

A profile is **certified** only when every one of its checks passed. A check skipped for a missing capability leaves the profile *incomplete* — a skip is never a pass, so a certification claim is always exactly as strong as what actually ran.

**The conformance claim format:** *"`<implementation>` certifies Stonefold TCK profiles `<list>` at specification `<version>`, kit version `<git ref>`."* Publish the rendered report alongside.

---

## 2. What the driver is (and is not)

The driver is a **test-only adapter** over your gateway: it loads a registry+policy, seeds rows into the store behind the connectors, submits intents *as an authenticated actor*, steps the dispatch worker, and exposes what happened (effects that left, audit records written). It is the *test harness's* hands — it is **not** part of your gateway, and the harness API must never exist in a production build (it can reset state and seed data by design).

Driver obligations (the contract is `stonefold_tck/driver.py`, one docstring per method):

| Method | Obligation |
|---|---|
| `load(registry_yaml, policy_yaml)` | (Re)configure with these fixtures; **reset all state**; refuse invalid policies (`ok=False`) |
| `set_clock(now)` | Pin the injected clock every time-based gate reads (the specification already mandates an injected clock) |
| `seed(resource, rows)` | Load rows into the store behind that entity's connector, **replacing** any rows previously seeded for that resource — re-seeding is how the TCK moves the world (B4's tenant reassignment, J3's resolved question); an appending driver fails those checks |
| `submit(actor, session_id, op)` | Submit one operation; **identity comes from this call, never the payload** (invariant 3) |
| `approve/reject(ticket, approver)` | Resolve a held action; `False` when refused (e.g. self-approval) |
| `dispatch_once()` | Run the staged-effect worker **synchronously to completion** |
| `effects()` | Every effect that actually left the gateway, in order |
| `kill(...)/lift(id)` | Issue/lift kill orders (global/agent/session/action_class) |
| `audit()` | The decision log since `load` (decision, resource, action, outcome, reason, reasonCode) — `reason` is the deciding rule/settle reason; the v0.2 reasons (`stale-decision`, `stale-guard:<gate>`, `scope-lost`) are normative and MUST be populated by drivers claiming the v0.2 capabilities, and `reasonCode` (v0.3) MUST be populated wherever a profile's codes are normative, so a check can assert that the *record* says why and not only the returned verdict |
| `inject_dispatch_failure(action)` | Make the next dispatch of that action fail at the connector |
| `update_named_set(name, values)` | Replace a registry named set's values at runtime — a sanctions update landing between decision and dispatch (`freshness` capability) |
| `submit_batch(actor, session_id, ops)` | Submit one multi-operation SIF batch, decided atomically (`batch` capability, v0.2) — reports the batch verdict, the failing operation's index, and the per-operation results |
| `connector_digest(name)` | The `sha256:<hex>` digest of the artifact currently implementing that connector, computed the way the gateway verifies a registry pin (`digest-pinning` capability, v0.2) |
| `tamper_connector(name)` | Swap the connector's implementation in place, without reloading policy — the supply-chain replacement the pin defends against (`digest-pinning` capability) |
| `resolve(ticket, resolver_id, gate)` | Credit a resolver identity against ONE named gate's release contract (v0.3; `hold-precondition` capability) — distinct from `approve`, which credits the approval-shaped contracts only |
| `sweep_holds()` | Run the held-row expiry sweep synchronously (v0.3; `hold-precondition` capability) — deadline arithmetic MUST run on the injected clock |
| `seed_obligations(registry, records)` | Load obligation records (ref → typed fields) into the mock adapter behind that declared registry (v0.3; `obligation` capability) — the obligation analogue of `seed` |
| `set_obligation_outage(registry, active)` | Make the obligation registry's adapter unreachable / restore it (v0.3; `obligation` capability) |
| `capabilities()` | Which optional obligations you support (missing ⇒ dependent checks SKIP) |

Two capabilities are the v0.3 opt-ins. **`per-item`** declares that an action may
declare `items: {field, independent}` and that the driver reports per-item verdicts —
`(item, decision, reasonCode, retryClass, ticket)` — plus the items that took effect. A
driver that returns `allow` for a partial application fails O2, which is the check that
matters: the envelope must not read as success when a quarter of the call was refused.
And **`closure`** declares the standard
`dispositionIsDeclared` check (Stele §7.6) and a registry that can declare an action's
`closure` (docs/06 §5c). A driver claiming it MUST supply the gateway's **own** decision
history for the run — what it refused earlier for the same actor and correlation id, read
from its own audit and from nothing else. The reason codes `DISPOSITION_REQUIRED` and
`CLOSED_WITHOUT_THE_WORK` are normative for drivers claiming it, in the returned verdict
and in the audit record.

Two capabilities are the v0.2 opt-ins: **`freshness`** declares that decision TTLs + volatile-gate re-validation are wired *with the REQUIRED TCK config* — default TTL **24 hours**, irreversible TTL **30 minutes** (the D5/D6 checks advance the clock against exactly these values, the same way §3 fixes the registered-function semantics); **`scope-reassert`** declares that the scope predicate is re-asserted at dispatch (either declared form — the TCK observes only the shared outcome).

The v0.2 opt-ins follow the same pattern: **`batch`** declares atomic batch decision semantics behind `submit_batch`; **`digest-pinning`** declares connector digest verification at load and at dispatch behind `connector_digest`/`tamper_connector` — the TCK authors the pin itself from the reported digest, so no fixture hard-codes an implementation artifact. For a driver claiming `digest-pinning`, the dispatch-time mismatch settle reason **`connector-digest-mismatch`** is normative, like the v0.2 reasons.

The v0.3 opt-ins (ACCEPTED with the reference certification of that set): **`hold-precondition`** declares three-valued checks with multi-hold release contracts, active held-row expiry, and hold dedupe — the driver gains a resolver-release call (`resolve`, contract-targeted), the expiry sweep is steppable like `dispatch_once`, and the REQUIRED TCK config (fixture semantics, like the freshness TTLs) is: dedupe window **one hour**, and **no deployment default resolver role** — so a gate naming no `resolvers:` has an unsatisfiable release contract, which is what J7 refuses fail-closed; **`feedback`** declares reason codes with retry classes and visibility redaction — the driver's submit result additionally carries `reason_code`, `retry_class`, and `agent_view` (the agent-facing payload rendered verbatim, post-redaction, so the kit can assert what did NOT leak); **`obligation`** declares `requireMatch` with the reservation lifecycle — the driver registers a **mock obligation-registry adapter** with the kit-specified semantics (§3) behind the fixture's declared registry, so certification stays black-box and no fixture depends on a real ERP/EMR. For drivers claiming these capabilities the settle/decision reasons `expired-hold:<gate>`, `hold-unresolvable`, `stale-guard:requireMatch`, and the `no-match` refusal are normative, like the v0.2 reasons.

Determinism is the design principle: `dispatch_once` steps the worker instead of racing a background thread, and `set_clock` removes wall-time — so every check is reproducible on any implementation.

## 3. Required registered-function semantics

The fixture pack (`stonefold_tck/fixtures.py`) references five registered names. Your driver must register implementations with **exactly** these semantics for the TCK run (they are deliberately trivial — the kit tests your *gateway*, not your DLP vendor):

| Name | Kind | Required behaviour |
|---|---|---|
| `tckOwnedBy` | scope predicate | row visible iff `row.owner_id == actor.id` |
| `tckTenantOf` | scope predicate | row visible iff `row.tenant == actor.claims["tenant"]` |
| `tck.rejectMarker` | content hook | BLOCK iff the payload contains the string `BLOCK-ME` |
| `tck.flagSet` | precondition check | pass iff the resolved target's `flag` is true |
| `tckSink` | disclosure sink | the only sink a `restricted` read may flow to |
| `tck.holdOnMarker` | precondition check (v0.3, hold-capable) | HOLD with code `tck-queue` iff the resolved target's `hold` field is truthy; RAISE iff its `crash` field is truthy; else pass. World-driven, like `tck.flagSet` — a resolved question stays resolved at the dispatch-time re-validation |
| `tck.codelessHold` | precondition check (v0.3, hold-capable) | a CODE-LESS hold iff the target's `badhold` field is truthy; else pass (the gateway must resolve that hold FAIL — §7.6 rule 2) |
| `tck.orders` | mock obligation-registry adapter (v0.3) | records seeded via `seed_obligations` (ref → typed fields per the fixture's declared schema); `query` filters by the gateway's typed selector; `reserve`/`consume`/`release` idempotent per (ref, intent id); reserving/consuming/releasing moves the record's `line.state` through `reserved`/`consumed`/`unconsumed`, so an `== 'unconsumed'` match clause refuses a spoken-for line at decision time |

The fixture registry ships in the **authoring format** (docs/06) — the spec's format — so every implementation adapts from the same artefact. (The reference driver converts it to its loader dialect in ~30 lines; see `authoring_to_compact`.)

Two fixture conventions the driver must map, both from docs/06 §4's implicit actions: the TCK submits the implicit observe under the name **`read`** (or with `action` omitted — the two are equivalent) and the implicit record under **`create`**; the fixture registry does not declare either, because declaring an entity is what makes it readable/writable.

## 4. Profiles and what they prove

| Profile | Checks | Proves |
|---|---|---|
| `core` | A1–A3, C1–C10 | default deny, deny-wins, gate AND-combination; valueLimit, rate, allow/denylist, from-states, quantityCap, disclosure (sinks + the declared classification order, fail-closed outside the declared order), contentCheck, fail-closed conditions, named preconditions |
| `lint` | A4–A9 | invalid policies refuse to load (open-on-irreversible, unknown names incl. `deny`, standing∩deny, dual-auth quorum, hold-capable checks declared without reason codes); warnings surfaced |
| `scope` | B1–B3 | scope injected below the model; effects on out-of-scope targets denied pre-dispatch; payloads cannot widen scope |
| `staging` | D1–D4 | effects staged by default and dispatched exactly once (idempotent); approvals hold/release/reject; dual-auth needs two distinct non-actor identities; failed irreversibles stage their declared compensation |
| `kill` | E1, E2s, E6 | session/agent/action-class kills → HALT; kill before the dispatch step cancels; a committed effect is never claimed reversed; lifting restores |
| `audit` | F1, F2c | every decision leaves a record; executed effects and success-audit records agree exactly |
| `freshness` | D5, D5b, D6, D6b, D6c, B4 | v0.2: an expired decision cancels at claim (`stale-decision`) and a late approval cannot resurrect it; a denylist update between decision and dispatch cancels (`stale-guard:denylist`); counters and approval grants are NOT re-run; a target reassigned after the decision never receives the effect (`scope-lost`) |
| `batch` | H1–H4 | v0.2: any DENY/HALT refuses the whole batch before anything commits or stages, naming the failing operation; a HOLD stages without refusing (record ops commit atomically with the staging); a later rejection does not roll committed ops back |
| `digest` | I1–I3 | v0.2: a pinned digest mismatch fails closed at policy load; a post-decision implementation swap cancels the staged effect at dispatch, audited `connector-digest-mismatch`; a matching pin dispatches normally |
| `hold-precondition` | J1–J7 | v0.3: a hold-capable check's hold stages the intent with its reason code and declared retry class; a code-less hold resolves fail; a precondition-hold composed with an approval requires **both** contracts (the resolver alone cannot release it — the approval-bypass regression); an expired hold settles `expired-hold:<gate>` on the injected clock that anchored the staging TTL, and a late release cannot resurrect it; outage ⇒ fail, never hold; duplicate holds collapse into one queue item (same ticket, attempt counted, each attempt audited) — and a DISTINCT question (a different target, hence different gate evidence) does **not** collapse; a hold with no resolvable release contract refuses fail-closed `hold-unresolvable`, never staged |
| `reads` | P1–P4 | v0.3: a stale source holds with `SOURCE_STALE` and says how stale; an unreachable one is distinguishable from it (`SOURCE_UNAVAILABLE`, denying by default); undated content is not treated as fresh; `onUnavailable: hold` queues for a human instead of denying permanently |
| `per-item` | O1–O4 | v0.3: a mixed call applies the clean items and refuses the rest, each named with its own reason code; a partial application is **never** reported as `allow`; every item clean is a plain allow and every item refused applies nothing; each refused item is audited on its own while the applied subset is one record |
| `closure` | N1–N4 | v0.3: a closure with no declared disposition holds `DISPOSITION_REQUIRED`; a closure claiming completion after this gateway refused the same actor in the same run holds `CLOSED_WITHOUT_THE_WORK` and names the refused actions; an honest non-completion disposition passes; `dispositionIsDeclared` on an action with no declared `closure` fails closed rather than passing |
| `feedback` | K1–K3 | v0.3: deny/hold results carry code + retry class; `code+fields` never leaks record-side values; the audit record is unaffected by redaction |
| `match` | L1–L5 | v0.3: exactly-one ⇒ pass; zero ⇒ `onNoMatch`; several ⇒ `onAmbiguous` hold, never a pick; a forged obligation copy in `data.*` changes nothing (pointer narrows, never substitutes); tolerance honoured; registry unreachable ⇒ fail-closed for irreversibles |
| `advisory` | A1–A12 | v0.4: a denied action executes and the record keeps the deny; the agent-facing result carries no trace of it; a kill order still halts; an advised hold queues no question; every record names its mode; an action the gateway could not judge is `unjudged`, never an allow; aggregates still accumulate; **advisory and enforcing runs of the same fixture reach the same verdicts** (A8 — the property the rest rests on); a batch enforcement would have refused commits whole; an item-bearing call applies every item and the record names the ones enforcement would have refused; a scoped read returns everything and the record counts what scope would have removed; an effect whose target leaves scope between decision and dispatch still dispatches, with the refusal recorded as advice |
| `consume` | M1–M5 | v0.3: reservation taken with staging (second intent against the line ⇒ `no-match`); consume lands with the settle; cancel/kill/expiry releases the line for resubmission; retries never double-consume; a reservation lost to another intent (adapter orphan-expiry, F5.2) cancels at claim with `stale-guard:requireMatch` — never a double payment |

## 5. What the TCK deliberately does NOT test (and why)

Honesty is the product's brand; it is also the kit's. Three specification guarantees are not black-box observable, and pretending otherwise would sell false certification:

1. **The true kill no-race (E2).** Whether the kill re-check and the `pending → dispatching` transition share one serialised transaction is a concurrency property *inside* your dispatcher. The TCK asserts the serialized contract at both interleaving boundaries (`E2s`); the concurrent race test remains an implementation-internal obligation (the reference keeps one over real Postgres row locks — `tests/test_m5_kill_race.py`).
2. **Transactional audit (F2).** Crash-consistency between the settle and the audit write needs fault injection inside your process. The TCK asserts the observable consequence (`F2c`: effects ⇔ success records, exactly); keep a crash-consistency test in your own suite.
3. **Multi-instance kill propagation (E3)** and **dependency-failure modes (C7/E5/F3)** need infrastructure control the kit doesn't assume. Capability hooks may add these later.
4. **The declared residual window in the audit record (B5's second clause).** The TCK's normalized audit shape doesn't carry `scopeApplied`, so which reassertion *form* ran is not asserted black-box; both forms are covered through their shared observable (the effect does not land, settle reason `scope-lost`). Keep a window-declaration test in your own suite (the reference does — `test_v04_scope_norace.py`).
5. **Transactional consumption atomicity.** Whether `consume` shares the effect's commit transaction on a `transactional` registry is, like F2, a crash-consistency property inside your process. The `consume` profile asserts the observable consequence (an injected dispatch failure leaves the line unconsumed — no consumed-without-effect); keep a fault-injection test for the shared transaction in your own suite.
6. **The preserved hold reason code on an expiry settle (v0.3).** The TCK's normalized audit shape carries the settle *reason* (`expired-hold:<gate>`, asserted by J4) but no reason-*code* field, so "original hold reason code preserved" is not asserted black-box. Keep a code-preservation test in your own suite (the reference does — `tests/test_v06_hold_substrate.py`).

7. **The advisory profile's deployment key (v0.4, §10a's second key).** A driver advertising `advisory` *is* a deployment that permits it, so the kit cannot show that a *different* deployment configuration would have refused the same policy. The check that matters — a policy declaring advisory where the deployment forbids it fails to load — needs two deployments, and the driver contract carries one. Keep it as a reference test (the reference does — `tests/test_advisory_keys.py`).
8. **The kill-store carve-out under advisory (v0.4).** That an unreadable kill store still fails closed while every other dependency failure is forwarded requires fault injection into the kill store, which the driver contract does not carry. The observable half — a *live* kill order still halts — is A3.
9. **Advisory forwarding on the interception surface (v0.4).** Whether a gateway forwards a call it could not map to the endpoint it was going to, rather than refusing it, depends on an upstream the driver contract has no way to place behind the gateway. A6 asserts the audit-side consequence (an unjudgeable action is recorded `unjudged`, never as an allow); the forwarding itself stays a reference test.

A certification claim therefore reads "certifies TCK profiles X, Y, Z" — never "proves the specification".

## 6. The wire binding (multi-language)

The harness API is the driver contract as twenty-one JSON endpoints — the full table with request/response shapes is in `stonefold_tck/http_driver.py`'s module docstring, and `stonefold_tck/adapters/http_harness.py` is the golden FastAPI example serving the reference. A non-Python gateway implements the same endpoints in its test build; `HttpDriver` does the rest. The whole suite runs through this path in CI (`test_wire_binding_certifies_end_to_end`), so the wire protocol itself is conformance-tested.

Rules: the harness is **test builds only**; every endpoint returns 200 with a JSON body; timestamps are ISO-8601; a capability you don't advertise may leave its endpoint unimplemented.

## 7. Versioning

The kit certifies against the specification version pinned in this repo (v0.4 today, shipped with the reference certification of the advisory profile in both bindings). This is also the worked example of how the kit absorbs a specification bump: v0.2's guarantees arrived as **new profiles** (`freshness` behind the `freshness` and `scope-reassert` capabilities, `batch` behind `batch`, `digest` behind `digest-pinning`), v0.3's follow the pattern (`hold-precondition`, `feedback`, `match` + `consume` behind `obligation`), and v0.4's `advisory` profile arrives behind the `advisory` capability the same way — an older gateway still certifies the original profiles unchanged (its missing capabilities SKIP the new checks, leaving the newer profiles honestly incomplete), while a gateway claiming the current version certifies them on top. Certifications stay meaningful because they name their profiles and kit version.
