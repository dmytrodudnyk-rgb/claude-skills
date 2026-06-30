---
name: adversarial-pr-review
description: Expensive, opt-in multi-agent adversarial code review of a PR, diff, or branch. Fans out isolated finder agents across distinct lenses (concurrency, security, correctness, architecture) plus a holistic design reviewer, then has an independent skeptic REFUTE every finding to kill false positives, then compiles one calibrated, severity-ranked review with file:line evidence and proposed fixes. Because it spawns many agents and costs a lot, trigger it ONLY when the user explicitly asks for an "adversarial review", "aggressive fan-out review", "fan-out review", or "multi-agent review" (or runs /adversarial-pr-review). Do NOT trigger on generic "review this", "thorough review", "audit", "find bugs", or "look this over" — those go to the lightweight inline reviewer (e.g. /code-review). This skill is opt-in by explicit keyword only; invoking it is the user's opt-in to expensive multi-agent orchestration. Handles remote PRs (GitHub gh, Bitbucket) and the local working diff.
---

# Adversarial PR Review

Reviews change-sets by **fanning out independent agents, then adversarially verifying every finding before it reaches the user.** The core insight: a single reviewer — or even a single fan-out of finders — produces plausible-but-wrong findings. Quality comes from a second, skeptical pass that tries to *refute* each finding against the real code. That refutation is what keeps the final review trustworthy and short.

It also enforces two habits that catch what naive reviews miss: review the **final code, not the PR's prose** (descriptions go stale and lie), and **calibrate severity** so the real issues aren't buried under nits.

## When to use this vs. a lightweight review

This is the **expensive, explicitly-invoked tier** — a multi-agent workflow that spawns dozens of agents. Fire it **only** when the user opts in by name: an **"adversarial review"**, **"aggressive fan-out review"**, **"fan-out review"**, or **"multi-agent review"**, or the `/adversarial-pr-review` command. For everything else — generic "review this," "thorough review," "audit," "find bugs," small/low-risk changes — use the **lightweight inline reviewer** (e.g. `/code-review`); this skill costs too much to fire on inference. The explicit keyword *is* the user's opt-in to that cost.

## The pipeline

1. **Resolve** the input and the exact diff range.
2. **Fan out** finders (one per lens) + a holistic design reviewer, in parallel.
3. **Refute** every finding with an independent skeptic.
4. **Compile** one calibrated, deduped, severity-ranked review.

## Step 1 — Resolve the input and diff range

Figure out *what* to review and pin the exact range. Review the **final state** of the source and diff against the merge-base so you see only what the change introduces.

- **Remote PR (GitHub):** `gh pr view <n>`, `gh pr diff <n>`; fetch the branch.
- **Remote PR (Bitbucket / other):** use the connected MCP (e.g. `bb_get` on `/pullrequests/{id}` and `/pullrequests/{id}/diff`) or fetch the source branch over git.
- **Local branch / working diff:** `git fetch` the branch if remote; otherwise review the current branch or the uncommitted working diff.
- **Always** compute `BASE=$(git merge-base <target> <source>)` and review `BASE..<source>`. Read changed files in full where context demands — don't review excerpts blind.

**Read the PR description, then distrust it.** Treat every claim (what it does, "no behavior change," "just a refactor") as a *hypothesis* to verify against the diff. Descriptions go stale routinely — late commits may have reversed what the summary describes. If the description and the code disagree, that mismatch is itself a finding worth surfacing first.

## Step 2 — Run the review (multi-agent)

This is a multi-agent fan-out, and invoking this skill is the user's opt-in — so **author and run a Workflow** (see `references/workflow-template.md` for a ready-to-adapt script). Pass the verified context to the Workflow via its **`args`** field — never paste the diff, code, or PR description into the script source, since backticks and `${...}` in that content break the script. Note: **`args` reaches the script as a string** (an object you pass arrives JSON-encoded, *not* as a live object), so the script must `JSON.parse` it before use — the template's `readContext()` helper handles this; a hand-rolled script must do it too, or `args.foo` will be `undefined`. If the Workflow tool isn't available, fan out with parallel `Agent` calls following the same find → verify → compile shape.

**Finders — one per lens, in parallel.** Each gets the diff range, the changed-file list, the *verified* context (what the code actually does — not the description), and one lens to hunt through:

- **Concurrency** — races, deadlocks, lifetime/use-after-free, lock/queue misuse, async-boundary bugs.
- **Security** — trust boundaries, authz/peer validation, crypto misuse, injection, secret handling, privilege.
- **Correctness** — logic/branching bugs, edge cases, error handling, dead/unreachable code, broken invariants.
- **Architecture** — coupling, duplication, unnecessary complexity, naming drift, dead code, single-source-of-truth violations, and **sibling functions / branch-ladders that must stay in lock-step** (when two places encode the same decision independently, drift between them is a latent bug).

Each finder runs in its **own isolated context** — it sees only its lens and the diff, never the other finders' output. That non-pollution is the point: a single reviewer holding all lenses anchors on whatever it noticed first and blurs them together, whereas isolated reviewers each go deep on one axis. Findings converge only at the compile step (Step 3), and only after independent verification.

Plus a **holistic design reviewer** (separate agent): is the whole approach sound, and could it be simpler? This catches problems no line-level lens sees.

Each finder returns structured findings (`{title, severity, category, file, line, description, evidence, suggestedFix, confidence}`). Tell them to be aggressive but to **verify each claim against the real code before reporting** and to cite concrete `file:line`.

**Scale to the change.** A 5-file diff needs a few finders and single-vote verification. A "thorough audit" of a large PR warrants a bigger finder pool, multi-vote verification, and a loop-until-dry pass. Don't run a 48-agent workflow on a 3-file change, and don't single-pass a 200-file migration.

**Refute every finding — this is the part that matters.** For each finding, spawn an **independent skeptic** that re-inspects the actual code and tries to *refute* it. Default to skepticism: mark `false_positive` if the claim misreads the code, references removed/nonexistent code, ignores an existing guard/lock/validation, is a stylistic opinion dressed as a defect, or only fires under conditions the architecture prevents. Mark `partially_valid` (with a corrected severity) if real but over- or under-stated. For high-severity or security findings, use multiple skeptics with distinct angles and let a majority-refute kill it.

## Step 3 — Compile the review

Take the findings + verdicts and produce **one** review. Apply your own judgment over the skeptics' verdicts — you have the codebase context; spot-check anything borderline yourself before accepting or rejecting it.

- **Drop the false positives — and list them**, one line each with the refutation. Transparency about what you killed is what makes the survivors credible.
- **Dedup** — the same defect often surfaces under several lenses; merge them.
- **Calibrate severity** — most findings are not critical. Tier them so the must-fixes aren't buried. Be willing to conclude "by-design / non-issue" and say why — but **disposition ≠ severity**: "by-design / no fix" is only for things that aren't defects; a *real* bug at low/nit severity still belongs in a fix tier (Nits), never under by-design.
- **For each kept finding:** what it is, `file:line`, why it matters, a concrete fix, and a disposition.

See `references/review-format.md` for the exact tiered output structure.

## Principles

- **Verify before you assert.** A finding that hasn't been checked against the real code is a guess, and guesses are expensive when they reach the author. The refute pass exists for exactly this.
- **Isolate the reviewers.** Each lens is a cold, independent context that never sees the others' findings — that's what yields diverse coverage instead of one anchored take. They converge once, at compile time, after refutation.
- **The description is a hypothesis.** Review the code that ships, not the story about it.
- **Calibrate, don't catastrophize.** Ten "criticals" means none are. Honest severity is what makes a review actionable. **Derive** severity from impact × reachability and apply the downgrade caps in `references/review-format.md` — don't assign it by feel; deriving it is what keeps severity stable across runs and reviewers.
- **Report what you dropped.** The false positives you killed are the evidence that the survivors are real.
- **Be willing to say "no fix."** "By-design, and here's why" is a valid, valuable outcome — don't manufacture findings to look thorough.
- **Ground everything in `file:line`.** If you can't point at it, you haven't found it.
