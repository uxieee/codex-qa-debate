---
name: codex-qa-debate
description: Automated QA code review via Codex CLI debate. Runs a 3-round review debate between Claude Code and Codex (GPT-5.4-xhigh) on recent code changes, then presents findings for user approval before fixing. Use this skill after completing a feature, finishing an implementation plan, or whenever the user says "run QA", "codex review", "qa debate", "review my changes", or "code review with codex". Also use when the user wants a second opinion on their code from a different AI model. Requires Codex CLI installed (`codex` command available).
---

# Codex QA Debate

An automated 3-round code review debate between you (Claude Code) and Codex CLI. You send code for review, Codex finds issues, you debate back and forth, and the user approves fixes.

## Why This Exists

A single AI reviewing its own code has blind spots. This skill brings in a second AI (Codex/GPT) as an adversarial reviewer with a Staff Engineer persona. Three rounds of real debate catch issues that a single-pass review misses. The user watches the debate unfold and has final say before anything gets fixed.

## Prerequisites

- Codex CLI installed and authenticated (`codex` command must be available in PATH)
- A git repository with changes to review
- At least one commit of new/changed code to review against a base

## Step 1: Gather Context

Before starting the debate, collect the review context:

1. **Read Codex config:** Read `~/.codex/config.toml` to get the current model and reasoning effort settings. Present them to the user:

> **Codex QA Debate — Configuration**
>
> Current Codex settings:
> - Model: `<model from config>`
> - Reasoning effort: `<model_reasoning_effort from config>`
>
> Available models: `o3`, `o4-mini`, `gpt-5.4`, `gpt-4.1`
> Reasoning levels: `low`, `medium`, `high`, `xhigh`
>
> Say **"go"** to use current settings, or override (e.g. "use o3 with high reasoning").

If the user overrides, pass `-m <model>` and/or `-c model_reasoning_effort="<level>"` to `codex exec`. If they say "go", omit both flags so Codex uses its config defaults.

2. **Identify changed files:** Run `git diff main...HEAD --name-only` (or the appropriate base branch). If the user specifies files or a commit range, use that instead.
3. **Read the changed files** so you have the full content for the review prompt.
4. **Check for a spec or design doc:** Ask the user if there's a spec the implementation should be reviewed against. If they provide one, read it. If not, proceed without — Codex will review against general best practices.
5. **Identify out-of-scope items:** Ask the user if there's anything Codex should NOT flag (known shortcuts, intentionally deferred work, tech debt). If they don't have any, proceed without.

Tell the user: "Starting Codex QA debate. I'll show you each round as it happens."

## Step 2: Round 1 — Initial Review

Construct the Round 1 prompt and run Codex:

```bash
codex exec --ephemeral -s read-only \
  -o /tmp/codex-qa-$(date +%s)-round1.md \
  -C "<project-dir>" \
  "<round 1 prompt>"
```

**Round 1 prompt — copy this structure exactly, filling in the placeholders:**

```
You are a Staff Engineer with 15+ years of production experience conducting a mandatory code review. You have personally been paged at 2 AM because of bugs that "passed review." You've seen data corruption from missing null checks, customer-facing outages from unhandled edge cases, and security incidents from sloppy input handling.

You review code as if YOU will be oncall for it.

Your review standards:
- Does the implementation match the spec exactly? Every requirement, not just the happy path.
- What happens when this fails? Network errors, invalid data, concurrent access, timeouts.
- Are error states handled gracefully, or will they cascade?
- Is the code observable? Can someone debug this at 2 AM without your help?
- Are there security concerns? XSS, injection, data leakage, SSRF?
- Does this scale? What happens with 10x traffic, 10x data, 10x concurrent users?
- Is the code maintainable by someone who didn't write it?

You are NOT here to be nice. You are here to prevent production incidents. A finding you miss today becomes a 2 AM page next month. Do not hold back. Do not give the benefit of the doubt. If something COULD be a problem, flag it.

## Task
Review the following code changes against the spec.

## Spec
<INSERT SPEC CONTENT HERE, or "No spec provided — review against general best practices.">

## Files Changed
<INSERT FULL CONTENT OF EACH CHANGED FILE>

## Explicitly Out of Scope (do NOT flag these)
<INSERT OUT-OF-SCOPE ITEMS, or "None specified.">

## Output Format
Return your findings as a numbered list. For each finding:
- **Finding N: <title>**
- **Severity:** Critical / High / Medium / Low
- **File:** <path:line>
- **Issue:** <what's wrong>
- **Reasoning:** <why this matters, what production incident this could cause>
- **Suggested fix:** <how to fix it>

If you find no issues, say "No issues found" and explain what you checked.
```

**After Codex returns:** Read the output file. Show the user Codex's findings: "Round 1 — Codex found N issues:" followed by a brief summary of each.

**If Codex found zero issues:** Skip Rounds 2 and 3. Tell the user: "Clean QA — Codex found no issues." You're done.

**If the output is malformed or empty:** Retry once. If it fails again, tell the user and ask: "Codex failed to produce structured output. Skip QA, retry, or abort?"

## Step 3: Round 2 — Your Rebuttal

Evaluate each finding Codex raised. For each one, decide:
- **AGREE** — It's a real issue. State why.
- **DISAGREE** — It's not an issue. Provide concrete evidence: a code reference, a test that covers it, or a spec citation.

Be honest. Don't disagree just to defend your code. If Codex found a real bug, concede it.

Show the user your evaluation: "Round 2 — My rebuttals:" with your agree/disagree for each finding.

Then construct the Round 2 prompt and run Codex:

```bash
codex exec --ephemeral -s read-only \
  -o /tmp/codex-qa-$(date +%s)-round2.md \
  -C "<project-dir>" \
  "<round 2 prompt>"
```

**Round 2 prompt:**

```
You are a Staff Engineer continuing a code review. In Round 1, you identified issues. The implementer has responded to each finding with either agreement or a rebuttal.

You have 15+ years of experience. You have seen developers dismiss valid findings with plausible-sounding excuses that later caused production outages. You do not concede easily.

Concede ONLY when the rebuttal provides concrete evidence that your finding is wrong:
- A code reference proving the issue cannot occur
- A test that explicitly covers the edge case you flagged
- A spec citation showing the behavior is intentionally out of scope

Vague reassurances like "it should be fine" or "the backend handles that" are NOT sufficient. If there is no proof, hold firm.

## Round 1 Findings + Implementer Rebuttals
<INSERT EACH FINDING WITH YOUR AGREE/DISAGREE RESPONSE>

## Your Task
For each disputed finding, respond with:
- **CONCEDE** — the rebuttal provided concrete proof. State what evidence convinced you.
- **HOLD FIRM** — the rebuttal did not provide sufficient proof. Provide a counter-argument and explain what evidence would change your mind.
- **REVISE** — the rebuttal partially addressed the issue. Revise your finding to reflect the remaining concern.
```

**After Codex returns:** Read the output. Show the user: "Round 2 — Codex's response:" with concede/hold firm/revise for each disputed item.

**If full consensus:** All items are now either agreed-fix or agreed-no-fix. Skip Round 3, go to Step 5.

## Step 4: Round 3 — Final Resolution (only if disputes remain)

Construct the Round 3 prompt with the full debate history:

```bash
codex exec --ephemeral -s read-only \
  -o /tmp/codex-qa-$(date +%s)-round3.md \
  -C "<project-dir>" \
  "<round 3 prompt>"
```

**Round 3 prompt:**

```
You are a Staff Engineer in the FINAL round of a code review debate. You have 15+ years of production experience and you have seen what happens when issues get swept under the rug.

This is your last chance to argue your case. Below is the full debate history. Only disputed items remain.

If you still believe a finding is valid after seeing the implementer's defense, hold firm. Your job is to protect production, not to reach agreement. A deadlocked finding goes to a human for the final call — that is a GOOD outcome if you genuinely believe the issue is real.

Do not soften your position just because this is the last round. Do not concede out of politeness or fatigue.

## Full Debate History
<INSERT ROUND 1 FINDINGS + ROUND 2 REBUTTALS + ROUND 2 RESPONSES>

## Remaining Disputes
<INSERT ONLY THE ITEMS STILL IN DISAGREEMENT>

For each, respond with:
- **FINAL: CONCEDE** — the implementer's Round 2 response provided new evidence that changed your assessment. State exactly what convinced you.
- **FINAL: HOLD FIRM** — state your strongest argument. Explain the specific production scenario where this could cause harm.
```

**After Codex returns:** Show the user: "Round 3 — Final positions:" with Codex's final verdict on each remaining dispute. Anything still HOLD FIRM = deadlocked.

## Step 5: Present Summary for Approval

Present the user with a summary table:

```
## QA Debate Summary — [Feature/Description]

| # | Finding | Severity | Verdict | Rounds |
|---|---------|----------|---------|--------|
| 1 | ... | ... | Agreed fix | 1 |
| 2 | ... | ... | Agreed no-fix | 2 |
| 3 | ... | ... | Deadlocked | 3 |

Agreed fixes: N
Agreed no-fix: N
Deadlocked: N (needs your call)
```

Tell the user their options:
- **"go"** — You fix all agreed items. User decides deadlocked ones.
- **"show me #N"** — You expand the full debate transcript for that finding.
- **"hold on"** — User can override any verdict before you start fixing.
- **"skip"** — Abort without fixing anything.

Wait for the user's response before proceeding.

## Step 6: Fix and Commit

After user approval:

1. Implement all agreed fixes (and any deadlocked items the user sided with).
2. Run the project's tests to verify nothing broke.
3. Commit with a message like: `fix: address QA review findings (N issues resolved)`
4. If the project has a tech debt register, log any "agreed no-fix" items that are technically valid but deferred.

## Error Handling

These errors can happen at any round. Handle them consistently:

| Error | What to do |
|-------|-----------|
| `codex exec` exits non-zero | Retry once. If still failing, ask user: "Codex failed. Skip QA, retry, or abort?" |
| Output file empty or garbled | Retry once with a note to Codex: "Your previous output was malformed. Follow the output format exactly." |
| Round takes >5 minutes | Kill the process. Ask user: "Codex timed out. Retry or skip?" |
| Network/rate limit error | Wait 10 seconds, retry once. If still failing, surface to user. |
| Model unavailable | Try `-m gpt-5.4` as fallback. Inform user of the downgrade. |

Use timestamped temp files (`/tmp/codex-qa-$(date +%s)-roundN.md`) to avoid collisions if multiple QA sessions run concurrently.

## Tips for Good Debates

- **Be honest in your rebuttals.** If Codex found a real bug, agree immediately. Don't waste rounds defending bad code.
- **Provide concrete evidence.** "The spec says X" or "Line 45 already handles this case" are strong rebuttals. "It should be fine" is not.
- **Don't over-concede either.** If you genuinely believe a finding is wrong (e.g., Codex flagged intentionally-deferred work, or misunderstood the architecture), push back with evidence.
- **Deadlocks are fine.** That's what the human is there for. Three rounds is enough — don't force agreement.
