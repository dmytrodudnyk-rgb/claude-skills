# Review output format

Produce **one** review, tiered by *disposition* so must-fixes aren't buried under nits. Lead with anything that changes the meaning of the PR (e.g. a stale description). For each finding: what it is, `file:line`, why it matters, a concrete fix, and a disposition.

## Disposition ≠ severity

These are two independent axes; don't conflate them.
- **Severity** (critical…nit) = how bad it is if it bites. Derived (see below).
- **Disposition** = which tier it goes in. **"By-design / no fix" is ONLY for things that are *not defects*** (correct in context). A *real* bug stays in a fix tier at its severity — a low/nit real bug goes in **Nits**, never under By-design. Filing a genuine bug as "by-design" just because its severity is low is the failure mode to avoid.

## Structure

```
# Review — <PR / diff>

<If the description is stale or contradicts the code, say so FIRST — it reframes everything below.>

## Must address
<Real defects, high/medium severity, that should block merge. Numbered. Each: 1-2 sentences +
 file:line + a before/after snippet or a concrete fix.>

## Flag for decision
<Real, but the right action depends on a project decision the owner must make (min OS, product
 intent, perf budget). State the decision needed.>

## By-design / no fix
<NOT a dumping ground for low-severity bugs. Only things that are correct-in-context — a naive
 review would flag them but they're fine. Say WHY. Often the only action is a clarifying comment.>

## Nits
<Real but low-stakes: small fixes, naming, dead code, diagnosability, low/nit real bugs. One line each.>

## False positives dropped
<What the verify pass killed — one line each with the refutation. This is the evidence that the
 kept findings are real and the review is calibrated.>
```

## Severity is DERIVED, not vibed

Severity drifts when each agent re-decides it by feel. Anchor it with a procedure that has both **floors** (so real defects aren't shaved away) and **caps** (so nothing is inflated):

1. **Base = impact × reachability.**
   - impact: **high** = data loss / memory corruption / auth bypass / crash on a common path · **med** = wrong behavior on a realistic path · **low** = cosmetic / no behavioral effect.
   - reachability: **common path** · **narrow precondition** · **latent/unreachable**.
2. **Class floors** — a *real* defect of these classes stays at least here **unless a cap in step 3 genuinely applies**:
   - memory-safety bug (UAF / double-free / OOB) → **≥ medium**.
   - silent security-relevant decision (wrong identity/cert/key, missing authz) on a realistic path → **≥ medium**.
   - common-path invariant / bound / cap violation, or injection / path-traversal from normal input → **high**.
3. **Caps** — may lower *below a floor*, but only when literally true:
   - mitigated by an existing in-code guard/lock/validation that **prevents** it → **low**.
   - latent / not reached on the target platform → **low**.
   - no runtime behavior change (dead code, naming, duplication, a missing-but-unreached guard) → **nit**.
   - requires a precondition the architecture prevents → **low** (or `false_positive` if fully prevented).
   - A **downstream backstop that only limits blast radius** (e.g. "the server rejects the wrong cert") is **not** a mitigation-cap — it informs *impact*, it does not cap severity.
4. **Match the nearest anchor; don't reflexively round down.**

**Anchors:** UAF with a narrow timeout-then-error trigger, single-BOOL write → **medium** · silent wrong-cert rebind on the common migration path (server-gated downstream) → **medium** · cap/bound broken on the concurrent common path → **high** · path traversal from unsanitized input → **high** · atomicity-qualifier conflict tolerated by a retry path → **nit/low** · dead/unused code → **nit** · missing platform guard on an unreached path → **low (latent)**.

Every finding records `triggerCondition` (what must happen to bite) and `mitigations` (an in-code guard that prevents it; "none" if there isn't one). Severity follows from those. The **verifier's `correctedSeverity` is authoritative**; the compile step uses it verbatim. Show **orig→corrected** when the verify pass changed it.

## Worked example (condensed)

```
# Review — PR #6304 (Feat/certificate)

Description is out of date — documents an app-side design later commits reversed. Review the final code.

## Must address
1. resolveCtkKeyid single-identity fallback silently rebinds to an unverified cert (pkcs11/index.ts) —
   medium (silent security-relevant rebind on the common migration path). [before/after] exact-match-or-clear.
2. Use-after-free in the sync XPC helpers (OVPNKeychainHelper.mm) — medium (memory-safety floor; narrow
   trigger, but no in-code guard prevents it). [before/after] return a value object.

## By-design / no fix
- Keychain PKCS#12 signs without a PIN — correct for an unlocked software keychain; bounded by TLS
  client-auth semantics. (Not a defect — hence here, not Nits.)

## Nits
- ctkTokenList() dead (zero callers). getCertPEM lacks the macOS guard its siblings have — low (latent).

## False positives dropped
- "Migration mutates frozen state → throws" — migrations run pre-hydration; no freeze applies.
```

Keep the prose tight; let the tiers and snippets carry the weight.
