# Workflow template — adversarial PR review

Adapt and run this with the `Workflow` tool. It implements the **find → refute** pipeline; you compile the final review in the main thread afterward.

**Before running:** pass your verified Step-1 context as the Workflow **`args`** field — a single string covering the real diff command, how to read the final code, and what the code *actually does* (not the PR description). The quality of every agent depends on this being accurate and grounded.

> ⚠️ **Pass context via `args`, never paste it into the script.** Diffs, code snippets, and PR descriptions routinely contain backticks (`` ` ``) and `${...}`. Inlining them into the script's template literals terminates the literal early and breaks the whole script. The `args` value is delivered to the script verbatim as structured JSON, so any content — backticks included — is safe.

**Shape:**
- `pipeline(DIMENSIONS, finder, verify)` — each lens's findings are refuted as soon as that lens finishes (no barrier).
- the design reviewer runs concurrently and is *not* per-finding refuted (it's holistic).
- the workflow returns `{reviewed, approach}`; **you** dedup, drop false positives, and tier in the main thread — **using the verifier's `correctedSeverity` verbatim** (do not re-rate).
- scale `DIMENSIONS`, the finder pool, and verifier votes to the diff size.

```js
export const meta = {
  name: 'adversarial-pr-review',
  description: 'Multi-lens adversarial review with per-finding refutation + holistic design review',
  phases: [{ title: 'Review' }, { title: 'Verify' }, { title: 'Approach' }],
}

// Verified Step-1 context, passed via the Workflow `args` field (NOT inlined) so that
// content containing backticks or ${...} can't break the script. Pass either a plain
// string, or an object { context: "..." }. It should cover:
//   - What is reviewed (PR/branch) and the exact diff command.
//   - How to read the FINAL code without checking out (git show <ref>:<path>, git grep ...).
//   - What the code ACTUALLY does — do NOT trust the PR description.
//   - Any known-stale or contradicted claims in the description.
const CONTEXT = typeof args === 'string' ? args : (args && args.context) || ''
if (!CONTEXT) throw new Error('Pass verified Step-1 context via the Workflow `args` field (a string, or { context: "..." }).')

// Shared severity rubric — floors (so real defects aren't shaved) + caps (so nothing is inflated).
const SEVERITY_RUBRIC = `
SEVERITY IS DERIVED, NOT VIBED — and it is NOT the disposition.
1) Base = impact × reachability. impact: high=data loss/memory corruption/auth bypass/crash on common path · med=wrong behavior on a realistic path · low=cosmetic/no behavior. reachability: common · narrow precondition · latent/unreachable.
2) Class FLOORS (a real defect of this class stays at least here UNLESS a step-3 cap genuinely applies): memory-safety bug (UAF/double-free/OOB) → >= medium; silent security-relevant decision (wrong identity/cert/key, missing authz) on a realistic path → >= medium; common-path invariant/bound/cap violation, or injection/path-traversal from normal input → high.
3) CAPS (may lower below a floor, but ONLY when literally true): mitigated by an in-code guard/lock/validation that PREVENTS it → low; latent / not reached on target platform → low; no runtime behavior (dead code, naming, duplication, missing-but-unreached guard) → nit; precondition the architecture prevents → low (or false_positive if fully prevented). A downstream backstop that only limits blast radius is NOT a mitigation-cap — it informs impact, not the cap.
4) Match the nearest anchor; don't reflexively round down.
Anchors: UAF w/ narrow timeout-then-error trigger, single-BOOL write → medium · silent wrong-cert rebind on the common migration path (server-gated) → medium · cap/bound broken on the concurrent common path → high · path traversal from unsanitized input → high · atomicity-qualifier conflict tolerated by a retry path → nit/low · dead/unused code → nit · missing platform guard on an unreached path → low (latent).
Record triggerCondition + mitigations; severity follows from them.`

const FINDINGS_SCHEMA = {
  type: 'object', additionalProperties: false, required: ['lens', 'findings'],
  properties: {
    lens: { type: 'string' },
    findings: { type: 'array', items: {
      type: 'object', additionalProperties: false,
      required: ['id','title','severity','category','file','line','description','evidence','triggerCondition','mitigations','suggestedFix','confidence'],
      properties: {
        id: { type: 'string' }, title: { type: 'string' },
        severity: { type: 'string', enum: ['critical','high','medium','low','nit'] },
        category: { type: 'string' }, file: { type: 'string' }, line: { type: 'string' },
        description: { type: 'string' }, evidence: { type: 'string' },
        triggerCondition: { type: 'string', description: 'what must happen for this to actually bite' },
        mitigations: { type: 'string', description: 'existing guards/locks/validation that already limit it; "none" if truly none' },
        suggestedFix: { type: 'string' }, confidence: { type: 'string', enum: ['high','medium','low'] },
      },
    } },
  },
}

const VERDICT_SCHEMA = {
  type: 'object', additionalProperties: false,
  required: ['verdict','reasoning','correctedSeverity','severityRationale'],
  properties: {
    verdict: { type: 'string', enum: ['confirmed','partially_valid','false_positive','uncertain'] },
    reasoning: { type: 'string' },
    correctedSeverity: { type: 'string', enum: ['critical','high','medium','low','nit','none'] },
    severityRationale: { type: 'string', description: 'which impact×reachability + which caps produced correctedSeverity' },
  },
}

const APPROACH_SCHEMA = {
  type: 'object', additionalProperties: false,
  required: ['summary','isSolid','strengths','weaknesses','simplifications','alternatives','verdict'],
  properties: {
    summary: { type: 'string' }, isSolid: { type: 'boolean' },
    strengths: { type: 'array', items: { type: 'string' } },
    weaknesses: { type: 'array', items: { type: 'string' } },
    simplifications: { type: 'array', items: {
      type: 'object', additionalProperties: false, required: ['title','description','tradeoff'],
      properties: { title: { type: 'string' }, description: { type: 'string' }, tradeoff: { type: 'string' } },
    } },
    alternatives: { type: 'array', items: { type: 'string' } },
    verdict: { type: 'string' },
  },
}

const FINDER_TAIL = `Be aggressive and adversarial, but VERIFY each claim against the actual code before reporting it, cite concrete file:line, and fill triggerCondition + mitigations honestly.\n${SEVERITY_RUBRIC}\nReturn via schema only.`

const DIMENSIONS = [
  { key: 'concurrency',  prompt: `${CONTEXT}\nLENS: concurrency/races/deadlocks/lifetime+use-after-free/lock+queue misuse/async-boundary (check-then-act across await) bugs.\n${FINDER_TAIL}` },
  { key: 'security',     prompt: `${CONTEXT}\nLENS: trust boundaries/authz+peer validation/crypto misuse/injection/path+input validation/secret handling/privilege.\n${FINDER_TAIL}` },
  { key: 'correctness',  prompt: `${CONTEXT}\nLENS: logic+branching bugs/edge cases/unit mismatches/error handling/dead+unreachable code/broken invariants.\n${FINDER_TAIL}` },
  { key: 'architecture', prompt: `${CONTEXT}\nLENS: coupling/duplication/unnecessary complexity/naming drift/dead code/SSOT violations, AND sibling functions or branch-ladders that must stay in lock-step (independent encodings of the same decision that can drift).\n${FINDER_TAIL}` },
]

function verifyPrompt(f, dim) {
  return `${CONTEXT}\nYou are an ADVERSARIAL VERIFIER. A "${dim}" reviewer made this claim:\n${JSON.stringify(f, null, 2)}\nIndependently inspect the ACTUAL code and TRY TO REFUTE IT. Default to skepticism: false_positive if it misreads code, references nonexistent/removed code, ignores an existing guard/validation/lock, is a style opinion dressed as a defect, or only fires under conditions the code prevents. partially_valid if real but mis-stated. confirmed only if you reproduced it.\nThen RECOMPUTE severity yourself via the rubric below and put it in correctedSeverity — this is the authoritative severity; explain it in severityRationale. Do not defer to the finder's label.\n${SEVERITY_RUBRIC}\nReturn via schema only.`
}

phase('Review')

const approachP = agent(
  `${CONTEXT}\nReview the ENTIRE approach at a high altitude: is it sound? could it be simpler? Trace the real call paths, weigh alternatives, cite code. Return via schema only.`,
  { label: 'approach', phase: 'Approach', schema: APPROACH_SCHEMA, effort: 'high' })
  .catch((e) => { log(`⚠ approach reviewer failed — design section omitted: ${e?.message ?? e}`); return null })
// ^ EVERY awaited agent MUST be catch-guarded. A terminal agent failure (StructuredOutput retry-cap
//   exceeded, or a terminal API error after retries) rejects its promise; an uncaught rejection that
//   reaches a top-level `await` aborts the ENTIRE run and discards all completed agents (recoverable
//   only via resumeFromRunId). The catch SURFACES the cause via log() — visible live in /workflows
//   and captured in result.logs — then degrades the one failed piece to null. Never silently
//   swallowed (don't `.catch(() => null)`), never a whole-run crash.

const reviewed = await pipeline(
  DIMENSIONS,
  (d) => agent(d.prompt, { label: `find:${d.key}`, phase: 'Review', schema: FINDINGS_SCHEMA, effort: 'high' })
    .catch((e) => { log(`⚠ finder lens "${d.key}" failed — lens skipped: ${e?.message ?? e}`); return null }),
  (res, d) => {
    if (!res || !res.findings || !res.findings.length) return { dimension: d.key, findings: [] }
    return parallel(res.findings.slice(0, 18).map((f) => () =>
      agent(verifyPrompt(f, d.key), { label: `verify:${d.key}:${f.id}`, phase: 'Verify', schema: VERDICT_SCHEMA, effort: 'high' })
        .then((v) => ({ ...f, dimension: d.key, verdict: v }))
        .catch((e) => { log(`⚠ verify ${d.key}:${f.id} failed — finding kept, unverified: ${e?.message ?? e}`); return { ...f, dimension: d.key, verdict: null } })
    )).then((vs) => ({ dimension: d.key, findings: vs.filter(Boolean) }))
  })

const approach = await approachP
return { reviewed, approach }
```

## After it returns

1. Flatten `reviewed[].findings` (a lens whose finder died is an empty `findings` array — skip it). Drop `verdict.verdict === 'false_positive'` — but spot-check the borderline ones yourself; you may have context the skeptic missed (e.g. a cross-module backstop).
2. Dedup findings that surface under multiple lenses.
3. **Use `verdict.correctedSeverity` as the final severity — do not re-rate.** Note `orig→corrected` when it changed.
4. Fold in the `approach` output as the design verdict + simplifications — but it may be `null` (that agent failed); if so, just omit the design section rather than blocking the review.
5. Compile per `references/review-format.md`.

## When an agent fails

The `.catch` guards above turn a single agent's terminal failure into a `null` instead of a crash, and each **surfaces** the cause via `log()` — it shows live in `/workflows` and lands in the returned `result.logs`, never swallowed. When you compile, disclose a skipped lens or unverified finding (per "Report what you dropped") rather than letting a silent gap pass for clean coverage. If a whole run still dies (script error, abort, kill), **don't re-run from scratch** — relaunch with `Workflow({scriptPath, resumeFromRunId})`: the journal returns every completed `agent()` call from cache instantly and only re-runs the failed/edited ones, so you recover the expensive work for free.

## Scaling knobs

- **Small diff (≤ ~10 files):** 2-4 lenses, single skeptic per finding. Consider a few direct parallel `Agent` calls instead of a Workflow.
- **Large / "exhaustive":** add lenses, raise verifier votes to 3 (majority-refute), wrap finders in a loop-until-dry — dedup against an accumulating `seen` set (not confirmed-only, or rejected findings reappear).
