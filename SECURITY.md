# Security policy

Stonefold is an enforcement layer and this repo is the canonical home of its
specifications. A flaw in the spec that would let a conformant implementation be bypassed
or exploited is a security issue, not an ordinary defect report — please report it
privately.

## How to report

**Do not open a public issue or PR.** Use GitHub's private vulnerability reporting on
this repository (Security tab → "Report a vulnerability"). If that is unavailable to you,
email **gallas.robert@gmail.com** with the subject line `STONEFOLD SECURITY`.

A useful report includes:

- The RFC section (`§n`), schema, or fixture affected.
- A reproduction. The ideal shape is an intent + policy + registry that demonstrates the
  hole — an implementation could follow the spec exactly and still allow the action.
- The stated guarantee you believe is broken.

Flaws in the reference gateway's code (as opposed to the spec text) are handled under the
same policy in the
[stonefold repo](https://github.com/stonefold-ai/stonefold/blob/main/SECURITY.md);
either entry point reaches the same maintainer.

Reports must be verified by a human before sending. Unverified AI-generated reports waste
the capacity this policy exists to protect and will get the sender banned.

## What to expect

This is a solo-maintained project, so the promises are modest but honest:

- Acknowledgment within a few days.
- An honest assessment of whether the report is valid, and why.
- Coordinated disclosure: a fix (normally a `CS-nnn` change-set item) or a documented
  limitation before the report is made public, on a timeline agreed with you.
- Credit in the changelog and advisory, if you want it.
- There is no bounty program.

## Scope

**In scope:** spec wording, schemas, or fixtures that would make a conformant
implementation exploitable — enforcement bypasses, scope or identity injection, kill or
audit guarantees that don't hold as written, under-specified corners with a demonstrated
attack.

**Out of scope:** implementation bugs where the spec is correct (those go to the
stonefold repo), and purely editorial issues (those are ordinary public issues, and
welcome).
