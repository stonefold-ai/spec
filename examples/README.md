# Understanding a Policy — a worked example

This guide reads one policy end to end so you can read any of them, and — importantly — it marks **what the gateway provides for you** versus **what you must implement or register** to make a policy actually run.

Every fixture in this folder comes from a **tested estate** — a system that was built and driven by real language models before the controls were written: `payments-ops.stele.yaml` (with `payments.registry.yaml`), `platform-ops.stele.yaml`, and `worklist.stele.yaml` (with `worklist.registry.yaml`). There are no fictional domain examples here on purpose; an example whose estate was never built demonstrates syntax, and syntax is what the schemas are for. (`INVALID-open-on-irreversible.stele.yaml` is the one deliberate exception: a policy the linter must refuse, kept as the negative fixture.)

This walkthrough uses [`payments-ops.stele.yaml`](payments-ops.stele.yaml). The same structure applies to every example here.

## Legend — where each referenced name comes from

When a policy mentions a name, it is one of these. The tags appear throughout the walkthrough.

- **[ENGINE]** — built into the gateway; you do **not** implement it per domain. The five action *kinds*, the four *decisions*, and all *gate types* (`valueLimit`, `rate`, `requireApproval`, `dualAuthorization`, `requireMatch`, `precondition`, …) are engine-level.
- **[REGISTRY]** — declared in the **model registry** (the catalogue of your domain): entities, actions and their *kind*, lifecycle *states*, *governance attributes* (`reversibility`, `operativeForce`, …), and obligation registries. The policy reads these; it never sets them.
- **[REGISTER]** — a **named function, set, or adapter you implement and register in the gateway**: scope predicates, precondition checks, named sets, obligation-registry adapters. The policy refers to them by name.
- **[IDENTITY]** — supplied at runtime by your **identity provider / session**: the actor and their claims, and the roles that approvers hold. Never supplied by the AI.
- **[AGENT DATA]** — values the agent puts in its request (`data.*`), validated against the registry.

---

## The walkthrough

### Header & defaults
```yaml
apiVersion: stele/v0.1
agent: payments-ops-agent
defaults: { failureMode: closed, audit: full }
killable: true
```
- `failureMode: closed` **[ENGINE]** — if any dependency errors, refuse rather than allow.
- `audit: full` **[ENGINE]** — record every attempt in full.
- `killable: true` **[ENGINE]** — an operator can halt this agent live.

### `allow` — the whitelist (anything not listed is denied)
```yaml
allow:
  - observe:    [Account, Payment, Payee]
  - record:     [LedgerEntry]
  - effect:     [pay]
  - transition: { Invoice: [send, markPaid] }
```
Every entity and action here must exist in the **[REGISTRY]** (`Account`, `Payment`, `Payee`, `LedgerEntry`, `Invoice`; the actions `pay`, `send`, `markPaid`, with their kinds). The *kinds* themselves (`observe`/`record`/`effect`/`transition`) are **[ENGINE]**.

### `deny` — the hard "never" (always beats allow)
```yaml
deny:
  - effect:     [exportData]
```
`exportData` must be declared in the **[REGISTRY]** so it can be named here (you deny things that exist).

### `scope` — which records, not just which actions
```yaml
scope:
  Account: tenantOf(actor)
  Payment: tenantOf(actor)
```
- `tenantOf` **[REGISTER]** — you implement this scope-predicate function in the gateway; it returns a filter the connector applies below the model.
- `actor` **[IDENTITY]** — the authenticated principal from the session. The AI never sets it.

### `gates` — the checks on `pay`

```yaml
pay:
  denylist:   { field: data.destinationCountry, set: sanctioned-list }
  valueLimit: { field: data.amount, max: 1000000, currency: USD }
  rate:       { limit: 50/hour, per: resource.payeeId }
```
- `denylist`, `valueLimit`, `rate` **[ENGINE]** (gate types).
- `sanctioned-list` **[REGISTER]** — a named set you define and maintain.
- `data.destinationCountry`, `data.amount` **[AGENT DATA]** — typed-checked against the registry.
- `resource.payeeId` **[REGISTRY]**-typed — the counter is per payee, owned by the gateway.

```yaml
pay:
  requireMatch:                            # no open order, no payment (spec §7.16)
    registry: erp.purchase_orders
    match:
      - "obligation.vendorId == data.vendorId"
      - "obligation.state == 'open'"
      - "obligation.line.state == 'unconsumed'"
      - { field: obligation.line.amount, matches: data.amount, within: "10%" }
    consume: obligation.line
    onNoMatch: hold
    resolvers: role:ap-clerk
```
- `requireMatch` **[ENGINE]** — the gate that checks the intent against a record another system already holds. Reserved at staging, consumed at settlement, so one order line can never pay two invoices.
- `erp.purchase_orders` **[REGISTRY]** — a declared obligation registry (docs/06 §5b); its adapter (`query`/`reserve`/`consume`/`release`) is **[REGISTER]** work.
- `obligation.*` operands resolve from the registry's response only — never from anything the agent wrote.
- `onNoMatch: hold` + `role:ap-clerk` **[IDENTITY]** — an unmatched invoice goes to a person's queue instead of out the door, or being silently refused.

```yaml
pay:
  requireApproval:
    when: "data.amount > 1000 and data.amount <= 10000"
    approvers: role:payments-manager
  dualAuthorization:
    when: "data.amount > 10000"
    approvers: role:treasury
  precondition:
    when: "exists data.newPayee"
    checks: [payeeCoolingOffElapsed]       # new-payee hold
```
- `requireApproval`, `dualAuthorization`, `precondition` **[ENGINE]**.
- `payeeCoolingOffElapsed` **[REGISTER]** — a precondition-check function you implement (deterministic; pass/fail).
- `role:payments-manager`, `role:treasury` **[IDENTITY]** — resolved to actual people by your identity system.
- A matched obligation never relaxes the tiers: a $50,000 matched invoice still takes dual authorization (§7.16 rule 5).

```yaml
Invoice.markPaid:
  precondition: { from: [sent] }
```
- The `from: [sent]` check is **[ENGINE]**, but `Invoice`'s lifecycle (including the `sent` state) must be declared in the **[REGISTRY]**.

---

## What you must implement / register to run this policy

If you deployed `payments-ops.stele.yaml`, this is your checklist. (The gateway and all gate types are assumed already built.)

**Declare in the model registry [REGISTRY]:**
- Entities: `Account`, `Payment`, `Payee`, `LedgerEntry`, `Invoice`.
- Actions + kinds: `observe` (Account/Payment/Payee), `record: LedgerEntry.create`, `effect: pay, exportData`, `transition: Invoice.send, Invoice.markPaid`.
- Lifecycle states: `Invoice` (incl. `draft`, `sent`, `paid`).
- The obligation registry `erp.purchase_orders` with its match surface (vendor, state, line amount/state).
- Attributes: `reversibility` on `pay` as your domain decides (the demo declares it `irreversible`; a domain with an in-system refund declares it `compensable` with the declared undo).

**Implement & register in the gateway [REGISTER]:**
- Scope predicate: `tenantOf`.
- Precondition check: `payeeCoolingOffElapsed`.
- Named set: `sanctioned-list`.
- Obligation-registry adapter for `erp.purchase_orders` (`query`/`reserve`/`consume`/`release`, idempotent per ref+intent).

**Provide from identity / session [IDENTITY]:**
- The actor's tenant; the roles `payments-manager`, `treasury`, `ap-clerk` (and who holds them).

**Provide connectors (effect bindings + data access):**
- `pay` → the payment execution binding (a bank/ledger API in production; the demo ships one).
- `observe` / `record` / `Invoice.*` → the data connector (ERP/ledger).

**Already provided by the gateway [ENGINE] — do not build per domain:**
- The five kinds, the four decisions, the pipeline, the outbox, kill, audit, and every gate type used above (plus the rest of the catalog).

---

## How to read any other example

Same method: skim `allow`/`deny`/`scope`/`gates`, and for every name ask "which tag is this?" — **[ENGINE]** (free), **[REGISTRY]** (declare it), **[REGISTER]** (implement it), **[IDENTITY]** (from your IdP), or **[AGENT DATA]** (the request). This exact policy runs in the shipped demo; its real bindings are listed the same way in [`docs/05-demo-spec.md`](https://github.com/stonefold-ai/stonefold/blob/main/docs/05-demo-spec.md). The full meaning of every key is in the specification, [`../docs/01-stele-policy-language.md`](../docs/01-stele-policy-language.md).
